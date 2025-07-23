---
title: "コンテナ（Docker・Singularity/Apptainer）上で NGS 解析のツールを動作させてみた"
emoji: "🧬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NGS", "PARCLIP", "docker", "singularity", "apptainer"]
published: true
---

## はじめに

次世代シーケンサー（NGS: Next-Generation Sequencing）の解析を始めたくて、ツール類をインストールする際にぶつかる壁として、**そもそもツールを自分の環境にインストールできない**という問題があります。

Bioconda というパッケージマネージャ（ライブラリ・ツール管理の仕組み）ができたことにより、今までよりも確かにインストールやツールの管理が楽になりました。ただ、それでもやはり、自分の環境依存でインストールできたり、できなかったりします。

これを解決する手段として、コンテナ技術（Docker・Singularity/Apptainer）が登場しました。コンテナとは、**「アプリケーション」とその「実行に必要な環境」（OS・ライブラリ・ツールなど）をひとまとめ**にして、どこでも同じように動かすための仕組みです。これにより、再現性、移植性、依存関係の管理が容易になります。

たとえば「研究室の PC では動いたのに、スパコンでは動かない」。そんな問題を解決するのがコンテナ技術です。

### Docker・Singularity/Apptainer の比較

コンテナ技術として、Docker・Singularity/Apptainer の大きく分けて 2 種類存在しています（Apptainer は Singularity の後継）

| 特徴             | Docker                            | Singularity / Apptainer               |
| ---------------- | --------------------------------- | ------------------------------------- |
| 実行ユーザー     | 基本 root（root 権限が必要）      | 非 root（ユーザー空間）で実行可能     |
| 主な用途         | クラウド、開発環境                | HPC、バイオ系研究、高セキュリティ環境 |
| ファイルシステム | 隔離されている（要 volume mount） | デフォルトでホストにアクセス可        |
| 互換性           | 広く使われている                  | Docker イメージを取り込める           |
| 利用例           | CI/CD、Web サービス               | NGS 解析、スパコン上のツール実行      |

NGS 分野において、Docker があまり採用されない背景として、以下のような問題があります。そのため、NGS 解析の現場では、Singularity/Apptainer が使われる傾向にあります。

| 問題点                    | 説明                                                                   |
| ------------------------- | ---------------------------------------------------------------------- |
| root 権限が必要           | 多くのクラスタでは許可されていないため運用できない                     |
| デフォルトでの I/O 制限   | ボリュームの明示的なマウントが必要、NGS のような大量データ処理には面倒 |
| GPU や MPI の対応が難しい | 特に MPI（並列分散処理）を使うツールで不安定な場合がある               |
| 高い学習コスト            | 初心者にとってネットワークやボリューム設定がわかりづらい               |

Docker・Singularity/Apptainer の使い分けの例としては、Docker を**開発時に使って**、本番環境では **Singularity/Apptainer に変換して使う**というパターンが考えられます。また、Nextflow や Snakemake などのパイプラインツールも、Singularity/Apptainer 対応が手厚くなっています。Apptainer（Singularity の後継）は積極的に HPC 向けに進化しており、今後もこの流れは強まると見られます。

と言っておきながら、私はソフトウェアエンジニアなので、今回は馴染みのある Docker を使って NGS 解析ツールを実行してみようと思います（正直、使い方はあまり変わりません）

## ツール検証の概要

今回使うデータは、CSTF-64 タンパク質の PAR-CLIP データになります（データサイズも小さいので、取り回しの良いデータだと思い選びました）
CSTF-64 は Alternative Polyadenylation に関与する RNA 結合タンパク質（RBP）で、PAR-CLIP は CSTF-64 結合サイトの RNA 配列をサンプリングしたデータになります。
@[card](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSM917676)

過去に私が解析したデータで、以下の解析フローの一部をなぞってみたいと思います。
@[card](https://imamachi-n.hatenablog.com/entry/2017/03/29/211905)

主な流れは以下になります。

1. 必要なツールのインストール
2. NGS データのダウンロード & 解凍
3. FASTQ ファイルの FastQC レポートを出力
4. Cutadapt を使ってアダプター配列をトリムした FASTQ ファイルを生成
5. アダプター配列をトリムした FASTQ ファイルの FastQC レポートを出力
6. MultiQC のレポート出力
7. alias を作成して、省略コマンドでツールを呼び出す

### 1. 必要なツールのインストール

まずは、Docker をインストールしましょう。Docker Desktop のデスクトップアプリをインストールします。

@[card](https://www.docker.com/ja-jp/)

![1](/images/ngs-analysis-with-container/1.png)

### 🛠️ BioContainer: バイオインフォマティクス分野でよく使われるコンテナイメージのレジストリ

NGS 解析で使用するツールのコンテナイメージの情報は、BioContainer というサイトで調べることができます。今回使用するコンテナイメージは以下になります。

@[card](https://biocontainers.pro/)

| ツール名  | リンク                                      |
| --------- | ------------------------------------------- |
| sra-tools | <https://biocontainers.pro/tools/sra-tools> |
| fastqc    | <https://biocontainers.pro/tools/fastqc>    |
| cutadapt  | <https://biocontainers.pro/tools/cutadapt>  |
| multiqc   | <https://biocontainers.pro/tools/multiqc>   |

### 2. NGS データのダウンロード & 解凍

NCBI GEO から NGS データ（FASTQ ファイル）をダウンロードします。
sra-tools のコマンドである `prefetch` を使います。

Docker コマンドを使って、sra-tools のコンテナを立ち上げ、`prefetch` コマンドをコンテナ上で実行します。コンテナイメージがダウンロードできていない場合は、初回だけダウンロードが走ります。

```bash
$ docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/sra-tools:3.2.1--h4304569_1 \
  prefetch SRR488740

Unable to find image 'quay.io/biocontainers/sra-tools:3.2.1--h4304569_1' locally
3.2.1--h4304569_1: Pulling from biocontainers/sra-tools
0cacab098358: Pull complete
bd9ddc54bea9: Pull complete
6b448fe1988c: Pull complete
Digest: sha256:d388f70b57293aaaab21275ebfc7e6bde9f51103561cda9a36cb410375c12179
Status: Downloaded newer image for quay.io/biocontainers/sra-tools:3.2.1--h4304569_1
```

オプションの詳細は以下。

| オプション | 説明                                                                                                                                                 |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--rm`     | 実行後に Docker コンテナを閉じる                                                                                                                     |
| `-v`       | カレントディレクトリを Docker 上では `/data` にマウントする                                                                                          |
| `-w`       | Docker 上のワーキングディレクトリを `/data` とする（この設定により、`/data/SRR488740` のフルパスではなく、`SRR488740` のファイル名だけで OK になる） |

SRA ファイルから FASTQ ファイルに解凍します。
sra-tools のコマンドである `fasterq-dump` を使います。
ちなみに、fastq-dump を高速化したのが、fasterq-dump になります（名前が紛らわしい…）

```bash
$ docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/sra-tools:3.2.1--h4304569_1 \
  fasterq-dump SRR488740

spots read      : 21,240,858
reads read      : 21,240,858
reads written   : 21,240,858
```

### 3. FastQC のレポート出力

FastQC で FASTQ ファイルの詳細をレポーティングしてもらいます。

```bash
$ docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0 \
  fastqc SRR488740.fastq

null
Started analysis of SRR488740.fastq
Approx 5% complete for SRR488740.fastq
Approx 10% complete for SRR488740.fastq
Approx 15% complete for SRR488740.fastq
Approx 20% complete for SRR488740.fastq
Approx 25% complete for SRR488740.fastq
Approx 30% complete for SRR488740.fastq
Approx 35% complete for SRR488740.fastq
Approx 40% complete for SRR488740.fastq
Approx 45% complete for SRR488740.fastq
Approx 50% complete for SRR488740.fastq
Approx 55% complete for SRR488740.fastq
Approx 60% complete for SRR488740.fastq
Approx 65% complete for SRR488740.fastq
Approx 70% complete for SRR488740.fastq
Approx 75% complete for SRR488740.fastq
Approx 80% complete for SRR488740.fastq
Approx 85% complete for SRR488740.fastq
Approx 90% complete for SRR488740.fastq
Approx 95% complete for SRR488740.fastq
Analysis complete for SRR488740.fastq
```

以下で HTML ファイルをブラウザで開いて確認できます（後で、MultiQC で全データをまとめたレポートを作ります）

```bash
open SRR488740_fastqc.html
```

### 4. Cutadapt を使ってアダプター配列をトリムした FASTQ ファイルを生成

cutadapt のインストール方法に応じて、いずれかのコマンドを実行してみます。パラメータチューニングや細かなオプションの説明は割愛します（ChatGPT とか、AI に聞けば教えてくれます）

```bash
$ docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/cutadapt:5.1--py39hbcbf7aa_0 \
  cutadapt --match-read-wildcards \
  --times 1 -e 0.1 -O 5 --quality-cutoff 6 -m 18 \
  -a TCGTATGCCGTCTTCTGCTTGT \
  --json SRR488740_PAR-CLIP_CSTF-64_cutadapt_result.json \
  SRR488740.fastq > SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq
```

### 5. アダプター配列をトリムした FASTQ ファイルの FastQC レポートを出力

```bash
docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0 \
  fastqc SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq
```

### 6. MultiQC のレポート出力

最後に、MultiQC で FastQC と Cutadapt のレポートをまとめて 1 つのレポートにしてもらいます。

```bash
$ docker run --rm -v $(pwd):/data -w /data \
  quay.io/biocontainers/multiqc:1.30--pyhdfd78af_0 \
  multiqc .


/// MultiQC v1.30

       file_search | Search path: /Users/imamachi/work/ngs-test
        searching | ████████████████████████████████████████ 100% 8/8
          cutadapt | Found 1 reports
            fastqc | Found 2 reports
     write_results | Data        : multiqc_data
     write_results | Report      : multiqc_report.html
           multiqc | MultiQC complete
```

できあがった MultiQC のファイルを開きます。

```bash
open multiqc_report.html
```

MultiQC は、多種多様なバイオインフォマティクスツールが出力するレポートに対応しており、それを 1 つのレポートにまとめてくれます。一覧性に優れているのでありがたいですね（私が研究してた頃にほしかった）

![1](/images/bioconda-101/1.png)

![2](/images/bioconda-101/2.png)

### 7. alias を作成して、省略コマンドでツールを呼び出す

ここまでで、コンテナ上でツールを実行してきて、コマンドを打つのが面倒になってきませんか？
なので、コマンドのエイリアスを設定してみようと思います。
まず、`.zsh_aliases` というファイルを作ります（以下では、zsh を使った例を書きます）

```bash
touch ~/.zsh_aliases
```

`.zsh_aliases` ファイルに以下を記述します。

```bash
# 共通ラッパー関数
run_in_docker() {
  image="$1"
  shift
  docker run --rm -v "$(pwd)":/data -w /data "$image" "$@"
}

# 個別のツール用関数
prefetch-d() {
  run_in_docker quay.io/biocontainers/sra-tools:3.2.1--h4304569_1 prefetch "$@"
}

fasterq-dump-d() {
  run_in_docker quay.io/biocontainers/sra-tools:3.2.1--h4304569_1 fasterq-dump "$@"
}

fastqc-d() {
  run_in_docker quay.io/biocontainers/fastqc:0.12.1--hdfd78af_0 fastqc "$@"
}

cutadapt-d() {
  run_in_docker quay.io/biocontainers/cutadapt:5.1--py39hbcbf7aa_0 cutadapt "$@"
}

multiqc-d() {
  run_in_docker quay.io/biocontainers/multiqc:1.30--pyhdfd78af_0 multiqc "$@"
}

```

`.zshrc` ファイル内で、`.zsh_aliases` を読み込む処理を記述します。

```bash
echo 'if [ -f ~/.zsh_aliases ]; then source ~/.zsh_aliases; fi' >> ~/.zshrc
```

ターミナルを再起動するか、設定ファイルを読み込みます。

```bash
source ~/.zshrc
```

例えば、コンテナ上で動作する FastQC を以下のように実行できるようになります。

```bash
fastqc-d SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq
```

### Apptainer (Singularity) を利用した場合

コマンドとイメージファイルが変わるだけで、ほとんど一緒です（コマンドなどは割愛します）
例えば、遺伝研のスパコンの場合、Docker は使えないので、Apptainer を使う必要があります。

@[card](https://sc.ddbj.nig.ac.jp/guides/software/Container/BioContainers/)

## さいごに

自分の環境に直接ツールをインストールを行う場合と比べて、「インストールを行う」という手間がなくなります。シンプルに「コンテナ上でツールを動かす」というカタチに変わります。

これまで見てきた通り、NGS 解析はたくさんのツールを組み合わせて、データを処理していかなくてはいけません。そういったデータ解析のワークフローをコマンド 1 つ 1 つ叩いて実行していくのは現実的ではありません（もちろん、トライ&エラーを行っている時点では、コマンドを個別に叩くことを否定しませんが）
次の課題として、Nextflow や Snakemake でワークフローを記述し、再現性の高い解析を考える必要があります。
