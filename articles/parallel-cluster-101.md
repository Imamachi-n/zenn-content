---
title: "AWS Parallel Cluster 101"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HPC", "ParallelCluster"]
published: false
---

## Parallel Cluster コトハジメ

### CLI をインストールする

Parallel Cluster の CLI は Python の `virtualenv` を利用しているため、用意する。

```bash
python3 -m pip install --upgrade pip
python3 -m pip install --user --upgrade virtualenv
python3 -m virtualenv ~/hpc-ve
source ~/hpc-ve/bin/activate
```

先ほど作成した仮想環境に入っていることを確認する。

```bash
which python3
# ~/hpc-ve/bin/python3
# 抜ける場合は、`deactivate`
```

続いて、AWS CLI がインストールされていることを確認。入っていなければ、インストールしておく。

```bash
pip3 install awscli
```

さらに、AWS CDK にも依存しているため、Node.js と CDK のインストールも忘れずに。
[nodenv](https://github.com/nodenv/nodenv#installation) などを経由してインストールしておくと、バージョン管理もできる。

最後に、Parallel Cluster の CLI をインストールする。

```bash
pip3 install aws-parallelcluster
```

AWS の設定も済ませておきましょう。

```bash
aws configure
# AWS Access Key ID [None]: YOUR_KEY
# AWS Secret Access Key [None]: YOUR_SECRET
# Default region name [ap-northeast-1]:
# Default output format [JSON]:
```

### 設定ファイルを自動でつくる

```bash
aws ec2 create-key-pair --key-name pcluster-key --query KeyMaterial --output text > ~/.ssh/pcluster-key
```

### Parallel Cluster の設定ファイルを書く

まず、設定ファイルを配置するディレクトリを用意する。

```bash
mkdir -p ~/.parallelcluster
```

続いて、以下の設定ファイルを作成する。

```bash
cat > ~/.parallelcluster/config << EOF
[aws]
aws_region_name = ap-northeast-1

[cluster default]
key_name = lab-3-your-key
vpc_settings = public
base_os = alinux2
scheduler = slurm
EOF
```

### 参考文献

- [Configuration - AWS Parallel Cluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/configuration.html)

## カスタムの設定をインスタンスに追加したい

### カスタム AWS ParallelCluster AMI の構築

[AWS のドキュメント](https://docs.aws.amazon.com/ja_jp/parallelcluster/latest/ug/tutorials_02_ami_customization.html)にも書いてあるとおり、カスタマイズのためのアプローチとしてカスタム AMI を構築することは推奨していない。

理由としては、今後のリリースでアップデートやバグ修正をカスタム AMI に適応できなくなるためとされている（おそらく、アップデート版の AMI をベースとしてカスタム AMI を作り直さなければならないと思われる）

### カスタムブートストラップアクション (Custom Bootstrap Actions)

[AWS のドキュメント](https://docs.aws.amazon.com/parallelcluster/latest/ug/pre_post_install.html)によると、カスタムブートストラップアクションを使うことが推奨される方法のようだ。インスタンス（クラスタ）がブートされる前と後に処理が挟み込める（pre-install と post-install の 2 種類）

#### pre-install と post-install の具体例

pre-install とは、NAT、Amazon Elastic Block Store (Amazon EBS)、スケジューラーの設定などがなされる前のタイミングを指す。

- pre-install アクション例
  - ストレージの変更
  - ユーザの追加
  - 各種パッケージ・ライブラリの追加

post-install とは、インスタンスの設定が完了した後のタイミングを指す。

- post-install アクション例
  - ストレージの変更
  - スケジューラーの設定
  - 各種パッケージ・ライブラリの更新

#### 対応するスクリプト

`Bash` と `Python` に対応している。

## その他

### 設定ファイルは `TOML` から `YAML` 形式のファイルに移行

AWS Parallel Cluster v2.x 系では、`TOML` の設定ファイルを使っていましたが、v3.x 系では `YAML` ファイルに変更されています。まだ、古い情報がネット上には残っているため、`TOML` で書かれたサンプルの設定ファイルが存在しますが、これらは v3.x 系では実行できないはずです。

### スケジューラーとして `SGE` と `Torque` がサポートされなくなる（2021/12/31 までサポート）

現行の AWS Parallel Cluster v3.x 系では、`SGE` と `Torque` のサポートがなくなりました。結果として、スケジューラーとして利用できるのは、`Slurm` と `AWS Batch` の 2 種類となっています。

[Configuring AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/getting-started-configuring-parallelcluster.html)

### Parallel Cluster の実行に必要な IAM ロール

強力な権限を付与して実行しても良いですが、できるだけ IAM ロールに割り振るポリシーは絞り込みたいです。AWS がミニマムな権限を示してくれているので、それらを参考にできます。

[AWS Identity and Access Management roles in AWS ParallelCluster 3.x](https://docs.aws.amazon.com/parallelcluster/latest/ug/iam-roles-in-parallelcluster-v3.html)

### Parallel Cluster のネットワーク構成

すべてパブリックサブネットで構築するケースと、Head Node をパブリックサブネット、Compute Node をプライベートサブネットに分けるケースに大別される。

このあたりは、要件やセキュリティレベルに応じて設定を考える必要がある。

[Network configurations](https://docs.aws.amazon.com/parallelcluster/latest/ug/network-configuration-v3.html)

## 参考文献

- [awsdocs - aws-parallelcluster-user-guide](https://github.com/awsdocs/aws-parallelcluster-user-guide)
- [AWS Black Belt Online Seminar - AWS ParallelCluster ではじめるクラウド HPC](https://d1.awsstatic.com/webinars/jp/pdf/services/20200408_BlackBelt_ParallelCluster.pdf)
- [AWS Black Belt Online Seminar - HPC on AWS](https://d1.awsstatic.com/webinars/jp/pdf/services/20201209_BlackBelt_HPC_on_AWS.pdf)
- [Using cost allocation tags with AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/using-cost-allocation-tags-with-aws-parallelcluster/)
- [Monitoring dashboard for AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/monitoring-dashboard-for-aws-parallelcluster/)
