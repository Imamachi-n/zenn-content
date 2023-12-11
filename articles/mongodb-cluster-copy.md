---
title: "mongoDB Atlas でクラスタをまるごとコピーする（Live Migration）"
emoji: "💾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mongoDBAtlas"]
published: true
publication_name: "cureapp"
---

:::message
こちらは [CureApp Advent Calendar 2023](https://qiita.com/advent-calendar/2023/cureapp) 11 日目の記事です。
:::

# はじめに

mongoDB Atlas 上で、DB を含むクラスタを丸ごと別環境にコピーしたいとき、どうしていますか？今回は、mongoDB Atlas の機能を使ってお手軽にクラスタをコピーする方法を紹介します。

# 内容

mongoDB のライブマイグレーション機能（Live Migration）を使って、ダウンタイムが発生しない方法でクラスタをまるごとコピーする方法を紹介します。

## Live Migration のやり方

あらかじめ、コピー先のクラスタを作っておきましょう（**コピー先の mongoDB Atlas のプロジェクトがないとコピーできません！**）
また、VPC Peering で mongoDB project の CIDR をデフォルト値から変更したい場合は、**クラスタを作成する前に**事前に設定しておきましょう。

コピー先のクラスタを作成したら、**コピー先**のクラスタの設定から、Migrate Data to this Cluster > General Live Migration をクリックします。

![](/images/mongodb-cluster-copy/mongo-1.png)
![](/images/mongodb-cluster-copy/mongo-2.png)

すると、以下の設定を**コピー元**のクラスタで行うように指示されます。

1. ネットワーク設定から 2 つの IP アドレスをホワイトリストに追加
2. コピー元のクラスタのホスト名の指定
3. read 権限を付与した mongoDB アカウントの指定

![](/images/mongodb-cluster-copy/mongo-4.jpg =500x)

まず、**コピー元**の mongoDB Atlas プロジェクト側でネットワーク設定を行っていきます。

- Network Access
  - Live Migration 画面に記載の **IP address たち**を登録

![](/images/mongodb-cluster-copy/mongo-5.jpg)

続いて、**コピー元**の mongoDB Atlas プロジェクト側でホスト名を調べます。

- Hostname の取得
  - primary の hostname を特定

![](/images/mongodb-cluster-copy/mongo-6.png)
![](/images/mongodb-cluster-copy/mongo-7.png)

以下が primary のホスト名になります。
![](/images/mongodb-cluster-copy/mongo-8.jpg)

最後に、**コピー元**の mongoDB Atlas プロジェクト側で mongoDB アカウントの作成を行います。

- Database Access（user アカウント作成）
  - Password Authentication
  - Database User Privileges: **Atlas admin**

![](/images/mongodb-cluster-copy/mongo-3.png)

**コピー先**の live migration の画面に戻り、上記で取得した情報をすべて記載していきます。

![](/images/mongodb-cluster-copy/mongo-4.jpg =500x)

Validate ボタンをクリックしすべての条件をパスすると、Start Migration がクリックできるようになるので、Start Migration をクリックします。

![](/images/mongodb-cluster-copy/mongo-9.png)

Cutover をクリックすると mongoDB の同期がストップし、migration 完了となります！（サーバの DB 接続情報等を切り替えた後、cutover してください）

![](/images/mongodb-cluster-copy/mongo-10.png =500x)

![](/images/mongodb-cluster-copy/mongo-11.png)

## トラブルシューティング

### Live Migration encountered error: could not initialize destination connection: could not connect to server: server selection timeout, current topology:

コピー元とコピー先とで、mongoDB のバージョンが異なるとこのエラーが発生しました。
コピー先のクラスタの mongoDB バージョンを確認し、作り直してみてください。

![](/images/mongodb-cluster-copy/mongo-12.png)

## [参考] データベース名を変更して restore したい場合

:::message
💡 DB へのアクセスがある状態で実施するとデータが不完全になるので、本番環境での実施は推奨しません ❗
:::

いったん、上記の方法で restore した後、同じクラスタ上でデータベース名を変更した DB を dump & restore して作ります（いったん、ローカルを経由してダウンロード & アップロードをするので、ネットワーク回線次第で速度が出ない・不安定であるかも）

Database Access > Database Users でユーザを作成（一時的）
Network Access > IP Addresses リストに、自分の public IP を登録（一時的）

以下のコマンドを実行（output stream を mongorestore コマンドに流すことで、ローカルに dump ファイルを作らずにダイレクトに新データベースを構築します）

```bash
mongodump --uri="mongodb+srv://${username}:${password}@${hostname}/${old_dbname}" --archive \
| mongorestore --uri="mongodb+srv://${username}:${password}@${hostname}/${old_dbname}" --archive --nsFrom='${old_dbname}.*' --nsTo='${new_dbname}.*'
```

# 最後に

クラスタをコピーするケースはあまりないかもしれませんが、mongoDB Atlas の Live Migration 機能はデータベースが載っているクラスタを別環境に移動させたいときに本当に便利です。ちょいネタでした。
