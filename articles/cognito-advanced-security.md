---
title: "Amazon Cognito アドバンスドセキュリティで不正ログインと思しきアクセスを検知する & トラブルシューティング"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AmazonCognito", "AWSCDK"]
published: true
publication_name: "cureapp"
---

## はじめに

Amazon Cognito のユーザー認証機能を利用していて、運用上、特定のユーザーのログインが成功したのか・失敗したのかログを確認したい、不正ログインと思しきアクセスをブロック・検知したい、と思ったことはありませんか？

Cognito にはアドバンスドセキュリティという追加機能があり、その機能を ON にすることで上記を実現可能です（ただし、[追加料金](https://aws.amazon.com/jp/cognito/pricing/)がかかります）

そのあたりの内容は、以下の記事に詳しくまとまっています。
[Amazon Cognito ユーザー認証に成功・失敗したログの確認方法 | DevelopersIO](https://dev.classmethod.jp/articles/how-to-check-the-cognito-authentication-log/)

Cognito の監視周りは、以下の記事があったりします。
[Amazon Cognito の監視のベストプラクティス - QG Tech Blog](https://tech.quickguard.jp/posts/monitoring-aws-cognito/)

「以上、終わり！」だと記事にならないので、ここでは AWS CDK での設定方法や、アドバンスドセキュリティを ON にすることで生成される CloudWatch Metrics に対してアラートを設定し、不正ログインと思しきアクセスが発生を検知 & 調査する方法について説明したいと思います。

また、アドバンスドセキュリティを導入した際に直面したトラブル & 対処法についても合わせて紹介したいと思います…。

## もう少し詳しく！

まずは、アドバンスドセキュリティを ON にすると、Cognito 側にはどのような設定が反映されるのか見てみましょう。

### ユーザーイベント履歴

AWS 管理コンソールから、ユーザー詳細を見ると、ユーザーごとにログイン等のイベント履歴を確認できるようになります。
サインインやパスワードの変更に成功したのか、不正アクセスと思しき挙動が見れるかどうか（リスク判定）、デバイス、IP アドレス等がわかります。　
![user-event-log](/images/cognito-advanced-security/user-event-log.jpg)

### 検出した脅威（不正アクセス等）の対する保護

主に、**漏えいした可能性のある認証情報**を使用したユーザーアカウントによるサインインを検出・ブロックできます。一般的なリスク要因（新しいデバイスからのサインイン・通常とは異なる場所や IP アドレスからのサインインなど）から自動で判断してくれます。

アドバンスドセキュリティは、以下の 2 種類のモードがあります。

- 監査のみ（`AUDIT`）
- フル機能（`ENFORCED`）

#### 監査のみ

「監査のみ」は文字通りメトリクスの収集（脅威の検出）のみとなります。まずは、こちらのモードで脅威をカウントして、CloudWatch Alarm を設定して高リスクのものに関しては適宜調査すると良いでしょう。

#### フル機能

「フル機能」はサインアップ・サインイン・パスワード変更のイベントごとに脅威判定するかどうか、検出されたリスクレベルに応じてサインインを許可・ブロックするか、といった詳細な設定を行えます。

### 検出精度を上げるためのフィードバック

脅威の検出は、AWS が Cognito のユーザーアプリの使用パターンを学習して、怪しい挙動が発生していないかチェックしています（と公式に書いてありました）

使った感じだと、AWS の AI が今までのユーザ動向をもとに判定しているので、以下のように誤検知されることもあります。その場合は、AWS 側にフィードバックを行うことができます（これをやることで精度が上がるはず？）

![feedback](/images/cognito-advanced-security/feedback.jpg =500x)

また、Cognito の使用パターンから十分に学習するのに一定の期間が必要とされています。AWS の公式ドキュメントには、最初は「監査のみ」モードに設定して、ブロックする前に **2 週間**、アドバンストセキュリティ機能を「フル機能」モードにしておくように勧めています（誤検知もあるので、ユースケースによっては自動ブロックをしないほうがいいと思いますが）
[ユーザープールにアドバンストセキュリティを追加する - Amazon Cognito](https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security.html#cognito-user-pool-configure-advanced-security)

## AWS CDK によるアドバンスドセキュリティの設定方法

CDK 側のアドバンスドセキュリティの設定としてはこれだけです。
`AUDIT` もしくは `ENFORCED` を指定することで、アドバンスドセキュリティを ON にできます（デフォルトは `OFF`）

```javascript
new cognito.UserPool(this, "myuserpool", {
  // ...
  advancedSecurityMode: cognito.AdvancedSecurityMode.ENFORCED,
});
```

### `AUDIT` と `ENFORCED` モード

この 2 つのモードは、前述した「監査のみ」「フル機能」の設定に該当します。

| 名前     | 説明                                                                                 |
| -------- | ------------------------------------------------------------------------------------ |
| AUDIT    | 検出した脅威（不正アクセス等）に関するメトリクスのみを収集                           |
| ENFORCED | 検出した脅威（不正アクセス等）に対して、リスクレベルに応じた自動予防アクションを実行 |

👇 詳しくは、公式ドキュメントを参照ください。
[enum AdvancedSecurityMode · AWS CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_cognito.AdvancedSecurityMode.html#example)

じつは、CDK によるモード設定だけでは 2 つの設定に違いはありません。CDK 側で `ENFORCED` モードに設定した場合、すべてのリスクレベルで「**サインインを許可**」するように設定されます。
つまり、`AUDIT` モードでメトリクスを取得している状態と変わりません。

CDK 側からさらに詳細設定する方法も用意されており、`CfnUserPoolRiskConfigurationAttachment` を使ってリスクレベルに応じたブロックの設定をできるはずなのですが、私が手元で確認した限りでは設定が反映されませんでした…（有識者求む）
https://github.com/Imamachi-n/aws-cdk-samples/blob/67d75e68e5967817d3af4eefc429ea54d9877bf5/packages/cognito-advanced-security/lib/cognito-advanced-security-stack.ts#L162-L199

## 検出された脅威を通知させる

Cognito のアドバンスドセキュリティで脅威を検知したら、Slack 等に通知させたいですよね。以下では、CDK による設定方法を説明します。

具体例では、CloudWatch Metrics をもとに高リスクの驚異が 1 件でも検知されたらアラートを発砲して、SNS・Chatbot 経由で Slack 通知させる仕組みを作ろうと思います。

![workflow](/images/cognito-advanced-security/workflow.png =450x)

### Slack ワークスペースの登録（to AWS）

AWS 管理コンソールから AWS Chatbot を開き、自分の Slack ワークスペースとの連携をやっておきます。

![](/images/cognito-advanced-security/chatbot-setup.png)

### SNS トピック・アクションの設定

Chatbot に Slack 連携させた後、Slack ワークスペース ID を取得できるのでそれを使います。また、通知先の Slack チャンネルは任意のものを指定してください。

```javascript
import { SlackChannelConfiguration } from "aws-cdk-lib/aws-chatbot";
import { Topic } from "aws-cdk-lib/aws-sns";
import { SnsAction } from "aws-cdk-lib/aws-cloudwatch-actions";

// SNS -> ChatBot 経由での Slack 通知設定
const slackChannel = new SlackChannelConfiguration(
  this,
  `cognito-as-slack-bot`,
  {
    slackChannelConfigurationName: `cognito-as-slack-bot`,
    slackWorkspaceId: "ABCD1234" // Chatbot に連携済みの Slack ワークスペース ID を指定,
    slackChannelId: `${yourSlackChannelId}`, // 任意のチャンネル ID を指定する
  }
);
const snsTopic = new Topic(this, `cognito-as-sns-topic`);
slackChannel.addNotificationTopic(snsTopic);
const snsAction = new SnsAction(snsTopic);
```

### アラートの設定

`metricName` には、Cognito アドバンスドセキュリティが用意するメトリクス名を指定します。

| 名前                      | 説明                                                 |
| ------------------------- | ---------------------------------------------------- |
| AccountTakeoverRisk       | アカウントの乗っ取りリスクの検知をカウント           |
| CompromisedCredentialRisk | 漏えいした認証情報を使ったサインインの検知をカウント |

`operation` には、脅威検出した該当イベントを指定します。

| 名前           | 説明             |
| -------------- | ---------------- |
| PasswordChange | パスワード変更時 |
| SignIn         | サインイン時     |
| SignUp         | サインアップ時   |

`riskLevel` には、検知した脅威のリスクレベルを指定します。

| 名前   | 説明     |
| ------ | -------- |
| High   | 高リスク |
| Medium | 中リスク |
| Low    | 低リスク |

```javascript
import { Alarm, ComparisonOperator, Metric } from "aws-cdk-lib/aws-cloudwatch";

// Cognito アドバンスドセキュリティのメトリクスに対してアラート設定
// URL: https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-settings-viewing-advanced-security-metrics.html
const metricName = "AccountTakeoverRisk"; // "AccountTakeoverRisk" | "CompromisedCredentialRisk"
const operation = "SignIn"; // "PasswordChange" | "SignIn" | "SignUp"
const riskLevel = "High"; // "High" | "Medium" | "Low"

const alarm = new Alarm(this, `cognito-as-high-risk-alarm`, {
  metric: new Metric({
    namespace: "AWS/Cognito",
    metricName,
    statistic: "SUM",
    dimensionsMap: {
      UserPoolId: `${yourUserPoolId}`, // メトリクスをチェックしたいユーザープールIDを指定
      Operation: operation,
      RiskLevel: riskLevel,
    },
    period: Duration.minutes(5), // 5分おきにチェック
  }),
  evaluationPeriods: 1,
  threshold: threshold || 1, // 1件以上でアラートを発砲
  comparisonOperator: ComparisonOperator.GREATER_THAN_OR_EQUAL_TO_THRESHOLD,
  alarmDescription: `${metricName} (${operation}) is too high`,
});
alarm.addAlarmAction(snsAction); // SNS アクションは先ほど用意したものを指定
```

### サンプルコード

サンプルコードを用意したので、実際に確認してみたい人はこちらを参照ください。
[aws-cdk-samples/packages/cognito-advanced-security at main · Imamachi-n/aws-cdk-samples](https://github.com/Imamachi-n/aws-cdk-samples/tree/main/packages/cognito-advanced-security)

## アラート発砲時の調査方法（該当ユーザーの特定）

せっかくアラートを設定しても、該当ユーザーを特定できなければ意味がありません。簡単ではありますが、以下が CloudTrail のログから調査する方法の流れです。

1. CloudTrail に Cognito のログが残るので、アラートが発砲された時間帯を調べます。
   a. イベントソース: `cognito-idp.amazonaws.com` で絞り込みます。
2. イベントレコードの `additionalEventData.sub` が Cognito のユーザー ID (Sub) に該当するので、それをキーにしてユーザプールから検索をかけます。
3. 該当ユーザのユーザーイベント履歴を確認することで、サインインの状況を確認できます。

こちらのブログには画像つきで詳細も載っているので、合わせて参考にすると良いかもしれません。
[Amazon Cognito ユーザー認証に成功・失敗したログの確認方法 | DevelopersIO](https://dev.classmethod.jp/articles/how-to-check-the-cognito-authentication-log/)

## トラブルシューティング

### `ENFORCED` モード & サインインをブロックする設定にしたら誤検知でブロックされてしまった

パスワードを連続して間違えたり、普段とは異なる IP アドレスからアクセスがあった場合、脅威として判定されてしまい（誤検知）、そのアカウントがブロックされてしまうというトラブルが実際に起こりました。

サービスによるのかもしれませんが、個人的な感覚では誤検知が多いので、状況に応じてブロックせずにカウントのみにして高リスクのものについてはアラートを発砲するようにし、アラート毎に状況確認してブロックするか否かを判断しても良いかもしれません。

## さいごに

Cognito のアドバンスドセキュリティは脳死で ON にして、収集したメトリクスからアラートを設定しておくのが良いと考えています。

アドバンスドセキュリティを ON にすることで取得できるログインログは、エンドユーザから「ログイン出来ないんだけど…」等の問い合わせが来たときの調査に使えます。

また、アカウントの乗っ取り等の脅威をこちらで早急に検知して、エンドユーザへの被害を食い止めるといった対処も可能となります。
