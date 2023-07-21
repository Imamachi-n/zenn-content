---
title: "CureApp インフラわいがや会 2023/06/02 議事録（AWS CDK の設定切り戻しでハマった話・AWS コスト異常検出、他）"
emoji: "🏗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWSCDK"]
published: true
publication_name: "cureapp"
---

:::message
こちらは CureApp のインフラ勉強会で話題に上がった内容をまとめた記事になります
:::

## AWS CDK の設定切り戻しでハマった話

Cognito の ユーザープール設定に KeepOriginalAttrs という設定があります。この設定を以下のように true にした後、

```javascript
const userPool = new UserPool(this, "UserPool", {
  keepOriginal: { email: true },
  ...
});
```

以下のように設定を消しました。

```javascript
const userPool = new UserPool(this, "UserPool", {
  // keepOriginal: { email: true },
  ...
});
```

false がデフォルト設定になるので、設定を消せば false になると思い込んでいましたが、true のままになってしまった…という話です。
AWS CDK でインフラ設定をする場合は、能動的に設定を書くようにしたほうが良い…という教訓になりました。

## AWS コスト異常検出 AWS Chatbot と統合された

リリースされたのは結構前になりますが、Slack 等に通知できるので便利です。

[AWS Chatbot と統合された AWS コスト異常検出 (Cost Anomaly Detection)を試してみた | DevelopersIO](https://dev.classmethod.jp/articles/aws-cost-anomaly-detection-integration-chatbot/)

## User Notifications を使えば、アラーム設定を一元管理できる話

最近 GA になった User Notifications。様々な通知系を一元管理できるので便利ですよね。

[CloudWatch アラームの状態変更を AWS User Notifications で通知してみた | DevelopersIO](https://dev.classmethod.jp/articles/cloudwatch-alarm-aws-user-notifications/)

## Security Lake が GA になりました

Security Lake は CloudTrail ログ、Route 53 Resolver クエリログ、Security Hub ログ、VPC フローログなどを一元管理できるサービスです。監査対応や GuardDuty で検知した問題を調査する場合などで便利そうです。

[[アップデート]組織内のセキュリティデータを AWS 上で一元管理できる！Amazon Security Lake が GA(一般利用開始)されました | DevelopersIO](https://dev.classmethod.jp/articles/amazon-security-lake-ga/)

## Trusted Advisor を Slack 通知させる

[Trusted Advisor レコメンデーションを EventBridge Scheduler を使って定期的に通知してみた | DevelopersIO](https://dev.classmethod.jp/articles/trusted-advisor-eventbridge-scheduler/)
