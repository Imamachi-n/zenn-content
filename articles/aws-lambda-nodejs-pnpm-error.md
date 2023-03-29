---
title: "aws-lambda-nodejs を使ってデプロイする際に pnpm が Node.js v14 に非対応となりエラーになる問題を解消する"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWSCDK", "lambda", "AwsLambdaNodejs"]
published: true
publication_name: "cureapp"
---

:::message
こちらの記事は 2023/03/29 14 時時点で発生している問題です。ここで紹介する方法は、AWS CDK 側のコード修正が入るまでのワークアラウンドになります。
:::

## 発端

2023/03/28、何もコードを変更していないのに、aws-lambda-nodejs を使った Lambda 関数へのコードのデプロイができなくなった。エラーは以下。

```
ERROR: This version of pnpm requires at least Node.js v16.4
The current version of Node.js is v14.21.3
Visit https://r.pnpm.io/comp to see the list of past pnpm versions with respective Node.js version support.
```

以下は、AWS CDK に上がっている Issue。

[CDK Deployment crashes: Dockerfile aws-lambda-nodejs/lib/Dockerfile has unsupported version of NodeJs 14 for new pnpm version · Issue #24820 · aws/aws-cdk](https://github.com/aws/aws-cdk/issues/24820)

### 何が原因なのか？

aws-lambda-nodejs は Docker コンテナ上で TypeScript のコードからバンドルを行っており、このとき使用される Docker イメージの `pnpm` のバージョンが固定されていません（あくまで、**2023/03/29 14 時時点** の話です）

```docker
# Install pnpm
RUN npm install --global pnpm
```

以下のコードから抜粋。
https://github.com/aws/aws-cdk/blob/07d3aa74e6c1a7b3b7ddf298cf3cc4b7ff180b48/packages/%40aws-cdk/aws-lambda-nodejs/lib/Dockerfile#L10

最近、pnpm のメジャーバージョンである v8.x 系リリースされ、上記の記述だと pnpm v8.x 系がインストールされるようになってしまいました。pnpm v8.x 系では Node.js v14 以下のバージョンに対応しなくなり、結果として Node.js v14 以下のコードをデプロイしようとするとバンドル時にエラーとなった…というわけです。

## 解決策（ワークアラウンド）

### patch-package をインストール

`patch-package` という npm パッケージを使って、パッチを作成します。
[ds300/patch-package: Fix broken node modules instantly 🏃🏽‍♀️💨](https://github.com/ds300/patch-package#set-up)

yarn を使っている場合は、以下の通りインストールします。

```bash
yarn add patch-package postinstall-postinstall --dev
```

### パッチを作成する

該当のコードを `node_modules` から探します。AWS CDK v2 を使っている場合は、以下のようなパスになります。

```
node_modules/aws-cdk-lib/aws-lambda-nodejs/lib/Dockerfile
```

上記の Dockerfile を開き、`pnpm` を `pnpm@7.30.5` に変更します。

```
 # Install pnpm
- RUN npm install --global pnpm
+ RUN npm install --global pnpm@7.30.5
```

node_modules のファイルを変更したら、以下のコマンドを実行します。

```bash
yarn patch-package aws-cdk-lib
```

すると、ディレクトリに `patches/aws-cdk-lib+2.62.1.patch` みたいな名前のパッチファイルが作成されます。

![](/images/aws-lambda-nodejs-pnpm-error/patch_img.png)

最後に、デプロイを実行して、ログで以下のように pnpm がバージョン固定されてインストールされたら成功です！

```
Step 4/13 : RUN npm install --global pnpm@7.30.5
 ---> Running in 12abc0012345

added 1 package in 602ms
```
