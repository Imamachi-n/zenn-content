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

Slurm を使った必要最低限の設定ファイルは以下の通りになります（`Region`, `Image`, `HeadNode`, `Scheduling` の項目）

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
      CapacityType: SPOT
      ComputeResources:
        - Name: t3micro
          InstanceType: t3.micro
          MinCount: 0
          MaxCount: 10
      Networking:
        SubnetIds:
          - subnet-0bcea6a97db79b5ee
SharedStorage:
  - MountDir: workspaces
    Name: shared-ngs-resources
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 1200
      DeploymentType: SCRATCH_1
      ExportPath: s3://ngs-data-bucket
      ImportPath: s3://ngs-data-bucket/workspaces
```

#### 詳細 (必要最小限の設定)

- `Region`: AWS リージョン
- `Image`: EC2 インスタンスの AMI の情報
  - `Os`: OS の種類（`alinux2`, `centos7`, `ubuntu1804`, `ubuntu2004`）
- `HeadNode`: Head Node で使用する EC2 インスタンスの各種設定
  - `InstanceType`: EC2 インスタンスタイプ（一度立ち上げたら最後、更新は効かない）
  - `Networking`: ネットワーク構成
    - `SubnetId`: Head Node を配置するサブネットの ID
  - `Ssh`: EC2 インスタンスにアクセスするための SSH 情報
    - `KeyName`: EC2 キーペア名
- `Scheduling`: スケジューラの設定
  - `Scheduler`: 使用するスケジューラ（`slurm`, `awsbatch`）
  - `SlurmQueues`: slurm のキュー設定（slurm をスケジューラとして使用している場合のみ）
    - `Name`: キューの任意の名前
    - `CapacityType`: EC2 インスタンスのキャパシティ（`ONDEMAND`, `SPOT`）
    - `ComputeResources`: コンピューティングリソース設定
      - `name`: コンピューティング環境の任意の名前
      - `InstanceType`: EC2 インスタンスタイプ
      - `MinCount`: リソースの最小値
      - `MaxCount`: リソースの最大値
    - `Networking`: ネットワーク設定
      - `SubnetIds`: キューを配置するサブネットの ID

#### 共有ストレージの設定 (Amazon FSx for Lustre のケース)

- `SharedStorage`: 共有ストレージの設定
  - `MountDir`: 共有ストレージをマウントするパス
  - `Name`: 共有ストレージの名前
  - `StorageType`: 共有ストレージのタイプ（サポートされる値は `Ebs`, `Efs`, `FsxLustre`）
  - `FsxLustreSettings`: FSx for Lustre の各種設定
    - `StorageCapacity`: Lustre ファイルシステムの FSx のストレージ容量 (`GiB` 単位) を設定 (1,200 GiB ~)
    - `DeploymentType`: デプロイタイプ（`SCRATCH_1`、`SCRATCH_2`、`PERSISTENT_1`のいずれか。詳細は後述）
    - `ExportPath`: Amazon FSx for Lustre ファイルシステムのルート（エクスポート先）となる S3 のパス
    - `ImportPath`: FSx for Lustre ファイルシステムのデータリポジトリとして使用している S3 バケットへのパス(`ExportPath` と同一の S3 バケットである必要がある)

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

### Elastic Fabric Adapter

同じサブネット上の他のインスタンスとの**低レイテンシー**のネットワーク通信を実現するためのネットワークデバイスです。

Compute Node とストレージがネットワーク通信してデータのやり取りを行っている以上、ネットワークのレイテンシーが処理上のボトルネックになる可能性があります。この問題を解消するために、AWS が用意しているネットワークデバイスが Elastic Fabric Adapter です。

すべてのインスタンスで利用可能なわけではなく、以下のインスタンスでのみ使用可能です。

- c5n.18xlarge
- c5n.metal
- g4dn.metal
- i3en.24xlarge
- i3en.metal
- m5dn.24xlarge
- m5n.24xlarge
- m5zn.12xlarge
- m5zn.metal
- r5dn.24xlarge
- r5n.24xlarge
- p3dn.24xlarge
- c6gn.16xlarge

#### 参考文献

- [Elastic Fabric Adapter - AWS ParallelCluster](https://docs.aws.amazon.com/ja_jp/parallelcluster/latest/ug/efa.html)
- [AWS Elastic Fabric Adapter の通信速度評価](https://tech.preferred.jp/ja/blog/aws-elastic-fabric-adapter-evaluation/)

## Amazon FSx for Lustre について

### ファイルシステム

#### スクラッチ (Scratch) ファイルシステム

スクラッチファイルシステムでは、以下のような目的に使われます。

- 一時的なストレージとして使いたい
- 短期間のデータ処理に使いたい

ファイルシステムに障害が発生しても、データは複製されず保持されません（つまり、可用性は低いということ）短期間のワークロード向けなので、障害が発生してデータが消えてしまっても、データ処理をやり直せば良いケースに適しています。

#### 永続 (Persistent) ファイルシステム

永続ファイルシステムでは、以下のような目的で使われます。

- 長期的に保持可能なストレージとして使いたい
- 長期間のデータ処理に使いたい

データはアベイラビリティーゾーン (AZ) 内で自動複製されます（つまり、高い可用性が担保されている）長期的・無期限に実行され、データが途中で消えてしまうと困る処理に適しています。

#### 参考文献

- [Amazon FSx for Lustre ファイルシステムのデプロイオプションの使用 - FSx for Lustre](https://docs.aws.amazon.com/ja_jp/fsx/latest/LustreGuide/using-fsx-lustre.html)

### Lustre におけるデータ圧縮

Lustre のデータ圧縮機能を利用することで、ファイルサーバとストレージ間ファイルシステムやバックアップストレージでコストを削減できます。また、Lustre ファイルサーバとストレージ（S3 など）間でのデータ転送量が小さくなるため、結果としてネットワークスループットの改善にも繋がります。

データ圧縮を有効にすると、Amazon FSX for Lustre では、新しく書き込まれたファイルがディスクに書き込まれる前に自動的に圧縮され、読み込まれるときに自動的に解凍されます。

データ圧縮では、LZ4 アルゴリズムを使用しており、ファイルシステムのパフォーマンスへの悪影響がなく、高圧縮であると謳われている（パフォーマンス指向のアルゴリズムで、圧縮速度が速く、高圧縮である…らしい）

#### 参考文献

- [Lustre data compression - FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/data-compression.html)

### ExportPath と ImportPath について

TODO: https://docs.aws.amazon.com/ja_jp/fsx/latest/LustreGuide/create-fs-linked-data-repo.html

## 参考文献

- [awsdocs - aws-parallelcluster-user-guide](https://github.com/awsdocs/aws-parallelcluster-user-guide)
- [AWS Black Belt Online Seminar - AWS ParallelCluster ではじめるクラウド HPC](https://d1.awsstatic.com/webinars/jp/pdf/services/20200408_BlackBelt_ParallelCluster.pdf)
- [AWS Black Belt Online Seminar - HPC on AWS](https://d1.awsstatic.com/webinars/jp/pdf/services/20201209_BlackBelt_HPC_on_AWS.pdf)
- [Using cost allocation tags with AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/using-cost-allocation-tags-with-aws-parallelcluster/)
- [Monitoring dashboard for AWS ParallelCluster](https://aws.amazon.com/jp/blogs/compute/monitoring-dashboard-for-aws-parallelcluster/)
