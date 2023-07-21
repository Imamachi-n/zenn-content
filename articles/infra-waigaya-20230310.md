---
title: "CureApp インフラわいがや会 2023/03/10 議事録（CDK・Node.js・mongoDB EOL、Cognito のログ）"
emoji: "😎"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["EOL", "Cognito"]
published: true
publication_name: "cureapp"
---

:::message
こちらは CureApp のインフラ勉強会で話題に上がった内容をまとめた記事になります。
:::

# 各種 EOL ニュース

## CDK v1 ついにメンテナンスモードに

今後、CDK v1 は L2 Construct に対する新規アップデートがストップします。一方で、L1 Construct や重大なバグ修復、セキュリティアップデートのみのアップデートは継続して行われるそうです。

サポート終了日は 2023/06/01 となっています。

[Version 1 of the AWS Cloud Development Kit (AWS CDK) is now in maintenance mode | AWS Developer Tools Blog](https://aws.amazon.com/jp/blogs/developer/version-1-of-the-aws-cloud-development-kit-aws-cdk-is-now-in-maintenance-mode/)

## Node.js v14 EOL 間近

Node.js v14 のセキュリティアップデートは 2023/04/30 までとなっています。AWS 関連のサービス（例えば、Lambda Node.js ランタイム）に関しては、+1 年位の猶予があると見ています。

また、Node.js 16 は OpenSSL 1.1.1 の EOL に引っ張られて EOL の時期が早くなってしまいました。

[LTS のはずの「Node.js 16」のサポート期間が 7 カ月短縮 ～ 2023 年 9 月 11 日までに - 窓の杜](https://forest.watch.impress.co.jp/docs/news/1417053.html)

なので、Node.js v14 → v18 へ一気にアップデートするのが良いかもしれません…（ツライ）

[Node.js | endoflife.date](https://endoflife.date/nodejs)

## mongoDB v4.2 EOL 間近

mongoDB Atlas で v4.2 のサポートが 2023/04/30 までとなっています。それ以降でバージョンを上げていなかった場合、強制アップデートがなされます（[こちらの記述によると](https://www.mongodb.com/docs/atlas/reference/faq/database/#what-happens-to-service-clusters-using-a-mongodb-version-nearing-end-of-life-)）

- リリースノート
  - https://www.mongodb.com/docs/manual/release-notes/
  - https://www.mongodb.com/docs/manual/release-notes/4.2/
  - https://www.mongodb.com/docs/manual/release-notes/4.4/
- 変更内容
  - https://www.mongodb.com/docs/manual/release-notes/4.4-compatibility/
- EOL 情報
  - https://www.mongodb.com/support-policy/lifecycles

## Cognito 周りのログ取りについて

### SES でメール送信している場合のログ

以下のような仕組みで、送信履歴を S3 に保存しておくことができる。後は、Athena でクエリを書いて良しなに。

[SES の送信履歴を確認したい](https://zenn.dev/isseeeeey55/articles/61b350c27e1040)

### Cognito アドバンスドセキュリティ

ユーザのログイン状況など、ユーザ動向を見れます。CDK も最近対応されたので、CDK で ON にすることもできます。

[Amazon Cognito ユーザー認証に成功・失敗したログの確認方法 | DevelopersIO](https://dev.classmethod.jp/articles/how-to-check-the-cognito-authentication-log/)
