---
title: "AWS Parallel Cluster 101"
emoji: "💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HPC", "ParallelCluster"]
published: false
---

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

## 参考文献

- [awsdocs - aws-parallelcluster-user-guide](https://github.com/awsdocs/aws-parallelcluster-user-guide)
