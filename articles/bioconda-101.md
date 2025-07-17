---
title: "NGS 解析を始めたくて Bioconda をセットアップしてみた"
emoji: "🧬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bioconda", "mamba", "anaconda", "NGS", "PARCLIP"]
published: true
---

次世代シーケンサー（NGS: Next-Generation Sequencing）の解析を始めたくて、Bioconda をセットアップしてみることにしました。また、FASTQ ファイルの QC に関しても試してみたので、合わせて忘備録的にやり方をまとめておこうと思います。

Bioconda のインストールで挫折した人もいるんじゃないでしょうか？
公式ドキュメントなどを確認してみたのですが、結構混乱します。

結論、Miniconda をインストールした環境に、Mamda をインストールするをオススメします。
詳細は以下に書きますが、**とにかく Bioconda をインストールできればいい人は読み飛ばしてください。**

## ⚠️ Miniconda / Anaconda / Mamba / Micromamba どれをインストールすればいいのか問題

Bioconda の使い方を確認すると、`conda` をインストールするように書いてあります。
@[card](https://bioconda.github.io/)

一方で、Bioconda に登録されているバイオインフォマティクスツールのページを開くと、mamba コマンドでのインストール方法が紹介されています。これを見ると、Bioconda は mamba を推奨しているようにも見えます。
@[card](https://bioconda.github.io/recipes/sra-tools/README.html#package-sra-tools)

conda のインストールページでは、Miniconda と Anaconda の両方が紹介されていて、マジで意味がわかりません。
@[card](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html)

それぞれのツールの概要をまとめると、以下になります。

| ツール     | 説明                                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| Anaconda   | Python + Jupyter + NumPy, pandas, matplotlib, scikit-learn,など全部入り。GUI（Anaconda Navigator）も付属。 |
| Miniconda  | Conda 本体と Python だけが入った最小構成の Anaconda。                                                      |
| Mamba      | Conda の代替。C++で実装されていて数十倍高速。Anaconda / Miniconda に追加インストールして使う。             |
| Micromamba | mamba の超軽量版。conda 自体も不要。完全にスタンドアロンで conda 環境を作れる。                            |

まず、**Anaconda** ですが、不要かもしれない・最初は使わないツール類がまとめてインストールされてしまいます。なので、今回の選択肢から外れます。
次に、**Micromamba** ですが、Anaconda（Anaconda / Miniconda）に依存せずに動作します。万が一、トラブルになったときのことを考えると、Anaconda のほうが利用者が多いですし信頼性も高いです。なので、こちらも選択肢から外れます。

結論、Miniconda で最小構成の Anaconda をインストールし、Anaconda 経由で Mamba をインストール。ライブラリ・ツールのインストール時のみ、Mamba を活用するという運用にするのが一番良いのではないかと思いました。この構成であれば、Mamba でトラブったときに Anaconda で直接ライブラリ・ツールをインストールすることも可能です。

## Miniconda + Mamda 構成に Bioconda をインストールする

### Miniconda のインストール

以下の公式ドキュメントの手順に従って、まずは Miniconda をインストールします。

@[card](https://www.anaconda.com/docs/getting-started/miniconda/install#macos)

```bash
# Intel版のMacのインストール手順
mkdir -p ~/miniconda3
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -o ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh

# Apple Silicon(ARM)版のMacのインストール手順
mkdir -p ~/miniconda3
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh -o ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```

ターミナルを再起動するか、以下のコマンドを実行します。

```bash
source ~/miniconda3/bin/activate
```

次のコマンドを実行して、利用可能なすべてのシェルで conda を初期化します。

```bash
conda init --all
```

### Mamba のインストール

`conda` コマンド経由で mamba をインストール可能なので、以下を実行します。

@[card](https://anaconda.org/conda-forge/mamba)

```bash
conda install conda-forge::mamba
```

accept が必要なので、`a` と打ち Enter します。

```bash
Do you accept the Terms of Service (ToS) for https://repo.anaconda.com/pkgs/main?
[(a)ccept/(r)eject/(v)iew]: a
Do you accept the Terms of Service (ToS) for https://repo.anaconda.com/pkgs/r?
[(a)ccept/(r)eject/(v)iew]: a
2 channel Terms of Service accepted
Channels:
 - conda-forge
 - bioconda
 - defaults
Platform: osx-64
Collecting package metadata (repodata.json): |
```

### Bioconda のインストール

以下の通りに、コマンドを実行します。

@[card](https://bioconda.github.io/)

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

ここまでで、Miniconda + Mamba 構成に Bioconda をインストールできました。

### Bioconda に登録されているツールのインストール（Bioconda の動作確認）

Bioconda のページで、インストールできるツールを検索できます。
@[card](https://bioconda.github.io/conda-package_index.html>)

とりあえず、sra-tools をインストールしてみましょう。`-y` オプションを指定することで、confirm をスキップできます（ターミナルで `y` と打つのも面倒なので）

```bash
mamba install sra-tools -y
```

以下のコマンドでインストールできたか確認します。Usage が出れば成功。

```bash
$ fasterq-dump --help

Usage:
  fasterq-dump <path> [options]
  fasterq-dump <accession> [options]

Options:
  -F|--format                      format (special, fastq, default=fastq)
  -o|--outfile                     output-file
  -O|--outdir                      output-dir
  ...
```

### ⚠️ python・依存ライブラリのバージョン不整合によるインストールエラー

Anaconda も万能ではありません。python や依存ライブラリのバージョン不整合により、インストールエラーになることがあります。
以下の例だと、cutadapt をインストールしようとして、`python 3.12` 以下でないとインストールというエラーが出ました（現環境だと `python 3.13`）

```bash
$ mamba install cutadapt

error    libmamba Could not solve for environment specs
    The following packages are incompatible
    ├─ cutadapt =* * is installable with the potential options
    │  └─ cutadapt [4.8|4.9|5.0|5.1] would require
    │     └─ python >=3.12,<3.13.0a0 *, which can be installed;
    └─ pin on python =3.13 * is not installable because it requires
       └─ python =3.13 *, which conflicts with any installable versions previously reported.
critical libmamba Could not solve for environment specs
```

上記の場合だと、以下のように、python 3.12 環境で `cutadapt-env` という名前の専用の環境を用意し、cutadapt をインストールできます。

```bash
mamba create -n cutadapt-env python=3.12 cutadapt -y
```

インストールした cutadapt は以下のコマンドで使用できます（`cutadapt` コマンドの代わりに `mamba run -n cutadapt-env` コマンドを使う）

```bash
mamba run -n cutadapt-env cutadapt --help
```

## mamba でバイオインフォマティクスツールをインストールして使ってみよう（FASTQ ファイルの QC をやってみる）

### 概要

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

### 1. 必要なツールのインストール

必要なツールをインストールします。

```bash
mamba install sra-tools -y
mamba install fastqc -y
mamba install multiqc -y

# cutadapt だけ python 3.12 をインストールした独自環境をセットアップ
# MEMO: python 3.13 の現環境だとインストールできなかったため
mamba create -n cutadapt-env python=3.12 cutadapt -y
```

### 2. NGS データのダウンロード & 解凍

NCBI GEO から NGS データ（FASTQ ファイル）をダウンロードします。
sra-tools のコマンドである `prefetch` を使います。

```bash
$ prefetch SRR488740.sra

2025-07-17T12:19:05 prefetch.3.2.1: 1) Resolving 'SRR488740.sra'...
2025-07-17T12:19:07 prefetch.3.2.1: Current preference is set to retrieve SRA Normalized Format files with full base quality scores
2025-07-17T12:19:08 prefetch.3.2.1: 1) Downloading 'SRR488740'...
2025-07-17T12:19:08 prefetch.3.2.1:  SRA Normalized Format file is being retrieved
2025-07-17T12:19:08 prefetch.3.2.1:  Downloading via HTTPS...
2025-07-17T12:24:13 prefetch.3.2.1:  HTTPS download succeed
2025-07-17T12:24:14 prefetch.3.2.1:  'SRR488740' is valid: 578878621 bytes were streamed from 578870841
2025-07-17T12:24:14 prefetch.3.2.1: 1) 'SRR488740' was downloaded successfully
```

SRA ファイルから FASTQ ファイルに解凍します。
sra-tools のコマンドである `fasterq-dump` を使います。
ちなみに、fastq-dump を高速化したのが、fasterq-dump になります（名前が紛らわしい…）

```bash
$ fasterq-dump SRR488740

spots read      : 21,240,858
reads read      : 21,240,858
reads written   : 21,240,858
```

### ⚠️ fasterq-dump トラブルシューティング

ダウンロードしたファイルは `sra` 拡張子が取れているので注意。勘違いして、以下を実行したらエラーになりました…。

```bash
$ fasterq-dump SRR488740.sra

2025-07-17T12:26:36 fasterq-dump.3.2.1 err: the input data is missing the QUALITY-column
fasterq-dump quit with error code 3
```

### 3. FastQC のレポート出力

FastQC で FASTQ ファイルの詳細をレポーティングしてもらいます。

```bash
$ fastqc SRR488740.fastq

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

### ⚠️ FastQC 実行時のトラブルシューティング（Java Runtime がない問題）

以下のようなエラーが出たら、Java がインストールされていないので、Java を brew 経由でインストールする必要があります（mamba / conda では Java までは管理してくれない）

```bash
$ fastqc SRR488740.fastq

The operation couldn’t be completed. Unable to locate a Java Runtime.
Please visit http://www.java.com for information on installing Java.
```

brew 経由で Java をインストールしてパスを通しておきます。

```bash
brew install openjdk

# brew install したときに「以下のパスを通してね」という指示が出てくるので、それに従う。
# 以下は例。
echo 'export PATH="/usr/local/opt/openjdk/bin:$PATH"' >> ~/.zshrc
```

ターミナルを再起動するか、以下のコマンドを実行します。
これで、`fastqc` コマンドが使えるようになっているはずです。

```bash
source ~/.zshrc
```

### 4. Cutadapt を使ってアダプター配列をトリムした FASTQ ファイルを生成

cutadapt のインストール方法に応じて、いずれかのコマンドを実行してみます。パラメータチューニングや細かなオプションの説明は割愛します（ChatGPT とか、AI に聞けば教えてくれます）

```bash
# 通常のコマンド
$ cutadapt --match-read-wildcards \
  --times 1 -e 0.1 -O 5 --quality-cutoff 6 -m 18 \
  -a TCGTATGCCGTCTTCTGCTTGT \
  --json SRR488740_PAR-CLIP_CSTF-64_cutadapt_result.json \
  SRR488740.fastq > SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq

# 独自環境を構築した場合のコマンド
$ mamba run -n cutadapt-env cutadapt --match-read-wildcards \
  --times 1 -e 0.1 -O 5 --quality-cutoff 6 -m 18 \
  -a TCGTATGCCGTCTTCTGCTTGT \
  --json SRR488740_PAR-CLIP_CSTF-64_cutadapt_result.json \
  SRR488740.fastq > SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq
```

### 5. アダプター配列をトリムした FASTQ ファイルの FastQC レポートを出力

```bash
fastqc SRR488740_PAR-CLIP_CSTF-64_adapter_trimmed.fastq
```

### 6. MultiQC のレポート出力

最後に、MultiQC で FastQC と Cutadapt のレポートをまとめて 1 つのレポートにしてもらいます。

```bash
$ multiqc .

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

## さいごに

Bioconda を使っている人に怒られそうですが、コンテナ技術（Docker・Singularity・Apptainer 等）を使おうぜ…と思いました。
今回、Bioconda をインストールして軽く触ってみたわけですが、Bioconda も万能じゃなくて、いろんな依存ライブラリのバージョンの影響をもろに受けるので、管理がかなり面倒になる未来が見えました。

とはいえ、ローカルで試す分にはこれで十分だし、便利であることは変わりありません。
Bioconda のインストールに苦しんでいる人がいたら、参考になれば幸いです。
