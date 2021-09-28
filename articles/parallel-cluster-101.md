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

Slurm を使った必要最低限の設定ファイルは以下の通りになります。

```yaml
Region: ap-northeast-1
Image:
  Os: alinux2
HeadNode:
  InstanceType: t3.micro
  Networking:
    SubnetId: subnet-02265a025d9462be1
  Ssh:
    KeyName: pcluster-key
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: queue1
      ComputeResources:
        - Name: t3micro
          InstanceType: t3.micro
          MinCount: 0
          MaxCount: 10
      Networking:
        SubnetIds:
          - subnet-0bcea6a97db79b5ee
```

#### 詳細

- `Region`: AWS リージョン
- `Image`: EC2 インスタンスの AMI の情報
  - `Os`: OS の種類（`alinux2`, `centos7`, `ubuntu1804`, `ubuntu2004`）
- `HeadNode`: Head Node で使用する EC2 インスタンスの各種設定
  - `InstanceType`: EC2 インスタンスタイプ（一度立ち上げたら最後、更新は効かない）
  - `Networking`: ネットワーク構成
    - `SubnetId`: サブネットの ID
  - `Ssh`: EC2 インスタンスにアクセスするための SSH 情報
    - `KeyName`: EC2 キーペア名

### 参考文献

- [Cluster configuration file - AWS Parallel Cluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/cluster-configuration-file-v3.html)
- [example_configs - aws-parallelcluster](https://github.com/aws/aws-parallelcluster/tree/release-3.0/cli/tests/pcluster/example_configs)

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
