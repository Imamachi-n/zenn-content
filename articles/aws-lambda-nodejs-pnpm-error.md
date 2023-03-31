---
title: "AWS CDK(aws-lambda-nodejs)のデプロイ時にNode.js v14非対応のpnpm v8が使われエラーになる問題を解消"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWSCDK", "lambda", "AwsLambdaNodejs"]
published: true
publication_name: "cureapp"
---

:::message
こちらの記事は 2023/03/29 14 時時点で発生している問題です。ここで紹介する方法は、AWS CDK 側のコード修正が入るまでのワークアラウンドになります。

2023/03/30 追記: AWS CDK **>=2.71.0** でこの問題は解消されています
:::

## 発端

2023/03/28、何もコードを変更していないのに、aws-lambda-nodejs を使った Lambda 関数へのコードのデプロイができなくなってしまいました…。エラーは以下。

```
ERROR: This version of pnpm requires at least Node.js v16.4
The current version of Node.js is v14.21.3
Visit https://r.pnpm.io/comp to see the list of past pnpm versions with respective Node.js version support.
```

以下は、AWS CDK に上がっている Issue になります。

[CDK Deployment crashes: Dockerfile aws-lambda-nodejs/lib/Dockerfile has unsupported version of NodeJs 14 for new pnpm version · Issue #24820 · aws/aws-cdk](https://github.com/aws/aws-cdk/issues/24820)

### aws-lambda-nodejs とは？

aws-lambda-nodejs は TypeScript で書かれたコードを自動的にトランスパイル & バンドルしてくれるパッケージです（以下のリンクは AWS CDK v1 のやつ）
[@aws-cdk/aws-lambda-nodejs - npm](https://www.npmjs.com/package/@aws-cdk/aws-lambda-nodejs)

コードとしては、以下のように記述します。これだけで、`cdk deploy` 時にトランスパイル & バンドルまで自動的にやってくれます。

```javascript
// AWS CDK v2 の場合
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";
// AWS CDK v1 の場合
import { NodejsFunction } from "@aws-cdk/aws-lambda-nodejs";

new NodejsFunction(this, "MyFunction", {
  entry: "/path/to/my/file.ts", // accepts .js, .jsx, .ts, .tsx and .mjs files
  handler: "myExportedFunc", // defaults to 'handler'
});
```

内部的には、esbuild でバンドルしており、esbuild がローカルにインストールされている場合はローカルでバンドルされます。ローカルに esbuild がない場合は、AWS CDK 側で用意されている Docker ファイルをもとに Docker イメージを作り、Docker コンテナ上でバンドルします。

今回問題になったのは、Docker コンテナのイメージ作成時に使われる Docker ファイルの記述です。

### 何が原因だったのか？

前述の通り、aws-lambda-nodejs は Docker コンテナ上で TypeScript のコードからバンドルを行っています。このとき使用される Docker ファイルの中身を見てみると `pnpm` がインストールされています。そして、少なくとも、**2023/03/29 14 時時点**ではバージョンが固定されていませんでした。

```docker
# Install pnpm
RUN npm install --global pnpm
```

以下のコードから抜粋。
https://github.com/aws/aws-cdk/blob/07d3aa74e6c1a7b3b7ddf298cf3cc4b7ff180b48/packages/%40aws-cdk/aws-lambda-nodejs/lib/Dockerfile#L10

コレの何が問題かと言うと、latest（**2023/03/29 14 時時点**）の pnpm v8.x 以降がインストールされてしまう点です。

pnpm は v8.x 系から **Node.js v14 以下のバージョンに対応しなくなりました**。上述の Docker ファイルの記述ではバージョン固定されていないため、Node.js v14 を使っている場合でも v8.x 系の pnpm がインストールされるようになってしまいました。

結果として、AWS CDK のコードは何もいじってないのに、Node.js v14 以下のコードをデプロイしようとすると突然、バンドル時にエラーが発生するようになった…というわけです。

## 影響範囲

- AWS CDK
  - <= v1.198.0 以下
  - <= v2.70.0 以下
- Node.js v14 以下

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
