---
title: "CureApp インフラわいがや会 2023/05/19 議事録"
emoji: "🏗"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["VerifiedAccess", "UserNotifications", "GuardDuty", "SecurityHub"]
published: true
publication_name: "cureapp"
---

:::message
こちらは CureApp のインフラ勉強会で話題に上がった内容をまとめた記事になります（すいません、最近書くのをサボっていました 💦）
:::

## AWS Verified Access で VPN なしで Web アプリケーションに安全にアクセス

最近、[AWS Verified Access](https://aws.amazon.com/jp/verified-access/) というサービスが GA となり、これを使うと VPN なしで Web アプリケーションに安全にアクセスできるというものです。

特に、VPN 以外の方法で、社内向け Web アプリに安全にアクセスさせる手段として使えそうです。VPN の管理が必要なくなり、社内で使われている Azure AD などの認証基盤を経由してユーザがアクセスできるので、ユーザ体験としても良いと思いました。

すでに、いくつか試してみた記事があったので紹介します。

- Azure AD を ID プロバイダーとする実装例
  - [Azure AD を Verified Access 信頼プロバイダーに指定してみた | DevelopersIO](https://dev.classmethod.jp/articles/verified-access-azure-ad/#toc-1)
- Verified Access エンドポイントの作成等がまとまっている
  - [AWS Verified Access で VPN-less な世界を体験してみた - Qiita](https://qiita.com/hayao_k/items/5f3b8a88a0ea75828f95)

懸念点としては、アクセス対象のサーバ・ALB 等をすべて private subnet に配置しないといけないという点です。既存のリソースに対して適応したいときは、インフラ構成の変更が必要となり手間かもしれません。

## AWS User Notifications で GuardDuty や AWS Health イベントなどの通知設定を一元的に設定・管理可能に

[AWS User Notifications](https://docs.aws.amazon.com/notifications/latest/userguide/what-is-service.html) というサービスが GA となりました。このサービスを使うことで、GuardDuty や AWS Health イベントなどの通知設定を一元的に設定・管理が可能となります。

今までバラバラに設定・管理していたのを、User Notifications を使って一元管理できるとスッキリしそうですね。

- [[新サービス] AWS User Notifications で通知設定を試してみた | DevelopersIO](https://dev.classmethod.jp/articles/aws-user-notifications-release/)
- [AWS User Notifications の高度なフィルターを使って通知するイベントを絞ってみた | DevelopersIO](https://dev.classmethod.jp/articles/aws-user-notifications-advanced-filter-event-notification/)

通知先としては、メール、AWS Chatbot、AWS モバイルアプリケーションに対応しています（Chatbot 経由で Slack 通知させるときにメンションを付けたいのですが、Chatbot の制約によりそこまではカスタムできないようです。）

## セキュリティインシデント疑似体験 GameDay に参加してみた

ゲームを通していくつかのケーススタディを学んだのですが、いずれも GuardDuty のアラートが起点となっていて、改めて GuardDuty のアラートの重要性を痛感しました（GuardDuty は CloudTrail や VPC フローログなどから、不審な操作・アクセスを検知してくれるサービスです。）

また、そもそもログが取れていないと GuardDuty による検知ができない、アラートが発砲されても調査する材料が足りない、ということに陥りかねません。今一度、必要なログをちゃんと収集できているか確認してみることも大切です。

Security Hub と GuardDuty で予防的統制と発見的統制にしっかり取り組んでいきたいですね。

- 発見的統制
  - AWS のサービスで言うと、[GuardDuty](https://aws.amazon.com/jp/guardduty/) による悪意のあるアクティビティや異常な動作をモニタリング & 検知がそれに該当します（怪しい動きを発見する）
- 予防的統制
  - AWS のサービスで言うと、[Security Hub](https://aws.amazon.com/jp/security-hub/) による自動セキュリティチェックがそれに該当します（問題が起こる前に予防する）

[「Security Hub と GuardDuty で予防的統制と発見的統制を運用する」というタイトルで JAWS-UG 名古屋に登壇しました #jawsug #jawsug_nagoya | DevelopersIO](https://dev.classmethod.jp/articles/22020128_jawsug_nagoya_security_control/)

## AWS 関連のカンファレンス紹介

- [AWS CDK Conference Japan 2023 - connpass](https://jawsug-cdk.connpass.com/event/278205/)
- [AWS Dev Day 2023 Tokyo | AWS](https://aws.amazon.com/jp/events/devday/japan/)
