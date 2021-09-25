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

### スケジューラーとして `SGE` と `Torque` がサポートされなくなる（2021/12/31 までサポート）

[Configuring AWS ParallelCluster](https://docs.aws.amazon.com/parallelcluster/latest/ug/getting-started-configuring-parallelcluster.html)

## 参考文献

- [awsdocs - aws-parallelcluster-user-guide](https://github.com/awsdocs/aws-parallelcluster-user-guide)
