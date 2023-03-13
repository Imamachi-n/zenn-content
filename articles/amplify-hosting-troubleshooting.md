---
title: "Amplify Hosting に Next.js アプリをデプロイする際のトラブルシューティング"
emoji: "😭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AmplifyHosting", "Nextjs"]
published: true
publication_name: "cureapp"
---

かなり初歩的なエラーから Amplify Hosting 固有のエラーまで取り揃えてみました（実際にすべてハマりました…😭）
いずれも 2023/03/08 時点で発生しうるエラーとなります。

## モノレポ（yarn workspace）構成の Next.js アプリをデプロイしたら、子パッケージが not found というエラーが出た

デフォルトのアプリケーション構築の設定ファイル `amplify.yml` は以下の通りになっています。

```yaml
version: 1
applications:
  - frontend:
      phases:
        preBuild:
          commands:
            - npm install
        build:
          commands:
            - npm run build
（以下省略）
```

yarn workspaces を使ったモノレポ（monorepo）構成にしていた場合、このままだと当然ながら npm に子パッケージを探しに行ってしまうため、以下のエラーが出ます。

```bash
ERR! 404 Not Found - GET https://registry.npmjs.org/your-app-sub-package - Not found
```

### 解決策：`npm` を `yarn` に変更

`npm` ではなく `yarn` に変更する必要があります（以下）

```yaml
version: 1
applications:
  - frontend:
      phases:
        preBuild:
          commands:
            - yarn install
        build:
          commands:
            - yarn run build
（以下省略）
```

アプリの設定 > ビルドの設定 > アプリケーション構築の仕様 から変更するか、Next.js アプリのディレクトリに配置した `amplify.yml` ファイルを書き換えてください。

![](/images/amplify-hosting-troubleshooting/build_image_settings_2.png)

## Node.js v18 にしたら `'GLIBC_2.27' not found` というエラーが出た

2023/03/08 時点では、Amplify Hosting のデフォルトで使用されるデフォルトイメージ（Amazon Linux2）に `GLIBC_2.26` までしかインストールされていません。

![](/images/amplify-hosting-troubleshooting/build_image_settings_1.png =500x)

結果として、Node.js v18 は `GLIBC_2.27` と `GLIBC_2.28` の両方が必要なため、デフォルトイメージを使うと以下のエラーが発生します。

```bash
node: /lib64/libm.so.6: version `GLIBC_2.27' not found (required by node)
node: /lib64/libc.so.6: version `GLIBC_2.28' not found (required by node)
```

公式のデフォルトイメージの対応が待たれます… 🥺
[Fix support for node 18 · Issue #3109 · aws-amplify/amplify-hosting](https://github.com/aws-amplify/amplify-hosting/issues/3109)

### 解決策：カスタムの Docker イメージを使う（Node.js 公式のイメージ）

ECR public ギャラリーから、Node.js 公式の Docker イメージから該当の Node.js のバージョンのイメージを探しましょう。例えば、Node.js v18.13.0 を使っている場合は `public.ecr.aws/docker/library/node:18.13.0` となります（以下、設定例）
[ECR Public Gallery - Docker/library/node](https://gallery.ecr.aws/docker/library/node)

アプリの設定 > ビルドの設定 > Build image settings から変更できます。
具体例では、以下の設定をしています。

- 構築イメージ
  - 構築イメージ: `public.ecr.aws/docker/library/node:18.13.0` を指定
- ライブパッケージの更新
  - Node.js version: `18.13.0`

![](/images/amplify-hosting-troubleshooting/build_image_settings_3.png =500x)

## `@sentry/nextjs` をインストールしたら、`Modules not found` というエラーが出た

stack overflow にも同様の報告が上がっています。
[next.js - Error building NextJS with @sentry/nextjs on AWS Amplify - Stack Overflow](https://stackoverflow.com/questions/72048510/error-building-nextjs-with-sentry-nextjs-on-aws-amplify)

Amplify Hosting 側の Issue はこちら。
[NextJS: Build fails with Module not found (webpack) · Issue #2427 · aws-amplify/amplify-hosting](https://github.com/aws-amplify/amplify-hosting/issues/2427)

どうやら、内部で利用される Lambda 関数を Next.js の SSR に対応させるために、このフラグが必要みたいです。
[amplify-hosting/FAQ.md at main · aws-amplify/amplify-hosting](https://github.com/aws-amplify/amplify-hosting/blob/main/FAQ.md#webpack-modulenotfound-errors)

> Your app may need to be built using the experimental-serverless-trace target. To opt-in into this behavior, you need to set the environment variable AMPLIFY_NEXTJS_EXPERIMENTAL_TRACE=true in your App settings.

### 解決策：環境変数に `AMPLIFY_NEXTJS_EXPERIMENTAL_TRACE: true` という設定を追加する

アプリの設定 > 環境変数 から、`AMPLIFY_NEXTJS_EXPERIMENTAL_TRACE` が `true` という変数を設定します。

![](/images/amplify-hosting-troubleshooting/build_image_settings_4.png)

## 本番稼働ブランチ以外のブランチを連携させてデプロイしたら、`Framework Web not supported` というエラーが出た

例えば、master ブランチを本番稼働ブランチとして設定した後、develop ブランチを開発環境として追加連携させたとします。このとき、Amplify Hosting 側で Next.js アプリではなく Web アプリとして誤って認識された場合に発生するエラーです（なぜ発生するのかは謎です）
[node.js - How to solve AWS Amplify error: CustomerError Framework Web not supported - Stack Overflow](https://stackoverflow.com/questions/74595024/how-to-solve-aws-amplify-error-customererror-framework-web-not-supported)

### 解決策：Amplify CLI で該当ブランチのフレームワークに Next.js を指定する

以下のコマンドを実行して、該当ブランチのフレームワークに Next.js を指定します。

```bash
aws amplify update-branch --app-id ${対象のappID} --branch-name ${該当のブランチ} --framework 'Next.js - SSR'
```

## Next.js v13 に上げたら `Invalid next.config.js options detected` というエラーが出た

Next.js v13 に上げたとき、以下のエラーが発生しました。

```bash
Error: Command failed with exit code 1: node_modules/.bin/next build
warn  - Invalid next.config.js options detected:
- The root value has an unexpected property, target, which is not in the list of allowed properties (amp, analyticsId, assetPrefix, basePath, cleanDistDir, compiler, compress, crossOrigin, devIndicators, distDir, env, eslint, excludeDefaultMomentLocales, experimental, exportPathMap, generateBuildId, generateEtags, headers, httpAgentOptions, i18n, images, onDemandEntries, optimizeFonts, output, outputFileTracing, pageExtensions, poweredByHeader, productionBrowserSourceMaps, publicRuntimeConfig, reactStrictMode, redirects, rewrites, sassOptions, serverRuntimeConfig, staticPageGenerationTimeout, swcMinify, trailingSlash, typescript, useFileSystemPublicRoutes, webpack).
See more info here: https://nextjs.org/docs/messages/invalid-next-config

> Build error occurred
Error: The "target" property is no longer supported in next.config.js.
See more info here https://nextjs.org/docs/messages/deprecated-target-config
```

プラットフォームが `ウェブコンピューティング` ではなく、`ウェブ動的`になっていないか確かめてみましょう。ウェブ動的は Next.js v13 に対応していない古いプラットフォームになります。
[Next.js 11 SSR アプリを Amplify Hosting コンピューティングに移行する - AWS Amplify ホスティング](https://docs.aws.amazon.com/ja_jp/amplify/latest/userguide/update-app-nextjs-version.html)

![](/images/amplify-hosting-troubleshooting/build_image_settings_5.png)

### 解決策：プラットフォームを `ウェブ動的` から `ウェブコンピューティング` に変更する（マイグレーション）

アプリの設定 > 全般 に「プラットフォームをウェブコンピューティングに移行」というボタンがあるので、これをクリックします。

![](/images/amplify-hosting-troubleshooting/build_image_settings_6.png =600x)

チェック内容を確認の上、全てにチェックを付け、万が一の際の切り戻し用のコマンドをコピーして手元に保存しておき、最後に「移行」ボタンをクリックします。

![](/images/amplify-hosting-troubleshooting/build_image_settings_7.png =600x)

と、移行の流れを書いておきながら、この方法だと原因不明のトラブルに見舞われる事（後述）があるので、１から Amplify Hosting の環境を作り直すことをおすすめします。

## プラットフォームを `ウェブ動的` から `ウェブコンピューティング` に変更したら、Cognito の認証ができなくなった（`cognito User does not exist`）

バックエンドを API Gateway と Lambda 関数によるサーバレス構成にしており、API Gateway の認証に Cognito を使っていた場合になります。

ウェブコンピューティングにマイグレーションした後、Next.js アプリにアクセスしてログインしようとすると、以下のエラーが出ました（原因不明）

```bash
cognito User does not exist
```

### 解決策：１からプラットフォームに `ウェブコンピューティング` を指定した環境構築を行う

`ウェブ動的` から `ウェブコンピューティング` へのマイグレーションを諦め、１から環境構築を行うと解決します（泣）
