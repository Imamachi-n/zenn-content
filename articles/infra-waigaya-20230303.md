---
title: "CureApp インフラわいがや会 2023/03/03 議事録"
emoji: "😎"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "WAF", "AppRunner", "NestJS", "JWT"]
published: true
publication_name: "cureapp"
---

:::message
こちらは CureApp のインフラ勉強会で話題に上がった内容をまとめた記事になります。
:::

# プロダクトのバージョン番号の管理（bump）をコマンド 1 発で処理しちゃう

## monorepo 構成になっている場合の子パッケージのバージョン上げ

- lerna をタスクランナーとして使用しています。

```bash
# パッチバージョンを上げる
lerna version patch --no-git-tag-version

# マイナーバージョンを上げる
lerna version miner --no-git-tag-version

# メジャーバージョンを上げる
lerna version major --no-git-tag-version
```

## react-native 製の Android・iOS のバージョン上げ

- `android/`, `ios/` 配下にある Android・iOS アプリのバージョン上げは react-native-version というコマンドツールを使っています。
  - https://www.npmjs.com/package/react-native-version

実行するコマンドは以下（`packages.json` の scirpts に追加して実行）

```json
{
  "scripts": {
    "version-bump": "react-native-version --never-amend"
  }
}
```

## パターンマッチで特定のファイル内に埋め込まれているバージョン番号を繰り上げる

- Deno のツールになっちゃうんですが、 bmp というコマンドで特定のファイルのバージョンを書き換えられます。
- larna version と bmp の組み合わせで version を繰り上げてます（v.6.1.2 みたいなバージョン）
  - bmp コマンド
    - https://deno.land/x/bmp@v0.2.0

Deno をインストール

```bash
brew install deno
```

zsh の場合、`.zshrc` に以下を追加。

```
export DENO_INSTALL="/Users/taro.yamada/.deno"
export PATH="$DENO_INSTALL/bin:$PATH"
```

設定ファイルの例はこんな感じ。

```
version: 6.3.4
commit: 'chore: bump to v%.%.%'
files:
README.md: v%.%.%
packages/hoge-mobile/.env: VERSION=%.%.%
packages/hoge-mobile/.env.production: VERSION=%.%.%
packages/hoge-mobile/.env.staging: VERSION=%.%.%
packages/hoge-mobile/android/app/build.gradle: versionName "%.%.%"
packages/hoge-mobile/ios/HogeMobile/Info.plist: <string>%.%.%</string>
packages/hoge-mobile/ios/HogeMobile-tvOS/Info.plist: <string>%.%.%</string>
packages/hoge-mobile/ios/HogeMobile-tvOSTests/Info.plist: <string>%.%.%</string>
packages/hoge-mobile/ios/HogeMobileTests/Info.plist: <string>%.%.%</string>
```

実行するコマンドは以下。

```bash
# パッチバージョンを上げる
bmp -p

# マイナーバージョンを上げる
bmp -m

# メジャーバージョンを上げる
bmp -j
```

## まとめ

- larna version と bmp と react-native-version の三種の神器を使えば、version の bump はコマンド化できる！（やったぜ！）

# IP アドレス直叩きの攻撃を WAF でブロックしたい

- IP アドレスの正規表現でチェックし、IP アドレス直叩きのアクセスをすべてブロック
- IP アドレスの正規表現のパターンは以下。

```
^((25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9]?[0-9])$
```

- 参考にしたのは、こちらの記事。
  - [ALB への IP アドレス直指定によるアクセスをブロックしたい | DevelopersIO](https://dev.classmethod.jp/articles/tsnote-alb-waf-ip-request-block/#toc-6)

以下の例では、259 件のアクセス（`ip_regex_pattern_block_rule_BlockedRequests`）をブロックできている。
![](/images/infra-waigaya-20230303/waf_ip_block.png)

# App Runner ってどうなんですか？

## 概要

- 何か知ってる人がいたら教えてほしい！（雑かよ！）
- ECS よりも App Runner を採用したほうが楽なのか気になっている。今日このごろ。

## わかっていること

- [DB 接続] mongoDB や RDS に VPC 内で接続可能（VPC Connector を使って）
  - [AWS App Runner の VPC ネットワーキングに Dive Deep する | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/deep-dive-on-aws-app-runner-vpc-networking/)
- [IaC] CDK はまだ L2 constructs がない。cloudformation から自動生成された L1 のみ（CDK の記述が増える）
  - [aws-cdk-lib.aws_apprunner module · AWS CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_apprunner-readme.html)
- [ルーティング] Route 53 エイリアスレコードをサポート（IP アドレス直指定しなくて良い）
  - [AWS App Runner で Route 53 エイリアスレコードがサポートされました | DevelopersIO](https://dev.classmethod.jp/articles/app-runner-using-route-53-alias-record/)
- [セキュリティ] WAF が使える
  - VPN 経由のみ許可、Managed rules の適応、IP アドレス直叩きの防止などに使える。
  - [[アップデート] AWS App Runner でついに AWS WAF がサポートされました | DevelopersIO](https://dev.classmethod.jp/articles/aws-app-runner-waf/)DevelopersIO
- [セキュリティ] 環境変数ソースで Secrets Manager と SSM パラメータストアをサポート
  - [[アップデート] AWS App Runner の環境変数ソースで Secrets Manager と SSM パラメータストアがサポートされました | DevelopersIO](https://dev.classmethod.jp/articles/app-runner-secrets-ssm-parameter/)
- [監視] サービスレベルの同時実行数、CPU およびメモリ使用率の各メトリクスが追加
  - [AWS App Runner に、サービスレベルの同時実行数、CPU およびメモリ使用率の各メトリクスが追加](https://aws.amazon.com/jp/about-aws/whats-new/2023/02/aws-app-runner-concurrency-cpu-memory-utilization-metrics/)

# NestJS ってどうなんですか？

- 何か知ってる人がいたら教えてほしい！（雑かよ！）
- NestJS は重厚長大なバックエンドフレームワークって感じで、型にはまってやるなら良さそうだが、学習コストもかかりそうで…。
- バックエンドを作るならコレ！ってライブラリ、あるのかなぁ（フロントエンドでいう Next.js みたいな）
- GraphQL との相性が良い
  - スキーマ自動生成

# JWT (JSON Web トークン) の検証ってみんなちゃんとやってます？

- [JSON web トークンの検証 - Amazon Cognito](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html)
  - token が改ざんされておらず、期限切れでもなく、有効な署名が付いているかなどを検証するためのもの

## Cognito を使っている場合

- 検証用のライブラリはこれが推奨らしい
  - [awslabs/aws-jwt-verify: JS library for verifying JWTs signed by Amazon Cognito, and any OIDC-compatible IDP that signs JWTs with RS256, RS384, and RS512](https://github.com/awslabs/aws-jwt-verify)
- 例
  - [Using Cognito groups to control access to API endpoints - DEV Community](https://dev.to/aws-builders/using-cognito-groups-to-control-access-to-api-endpoints-346g)

## Auth0 を使っている場合

- [auth0/node-jsonwebtoken: JsonWebToken implementation for node.js http://self-issued.info/docs/draft-ietf-oauth-json-web-token.html](https://github.com/auth0/node-jsonwebtoken#jwtverifytoken-secretorpublickey-options-callback)

# CloudWatch Logs もデータ保護ポリシーなるものが追加されました

## 概要

- ログの中に credential 情報や個人情報が載っていても、権限がないとすべてマスクされる仕組みです。
  - [[アップデート]Amazon CloudWatch Logs でログデータに含まれた機密情報を保護出来るようになりました #reinvent | DevelopersIO](https://dev.classmethod.jp/articles/cloudwatch-logs-mask-protected-data/)
  - [[小ネタ]CloudWatch Logs の機密データ保護機能でシークレットアクセスキーがマスキングされるパターンを調べてみた | DevelopersIO](https://dev.classmethod.jp/articles/cloudwatch-logs-aws-secret-mask-pattern/)

## 仕組み（公式ドキュメントから抜粋）

1. クレジットカード番号や IAM アクセスキーなど、監査の対象となるデータ識別子を選択します。
2. 選択したデータ識別子と一致するデータが送信される際の検出結果の送信先を選択します。
3. マスキングするデータ識別子を設定します。

## ロール（公式ドキュメントから抜粋）

- マスキングされたデータには、昇格した IAM 特権を使用してマスキングなしでアクセスできます。
