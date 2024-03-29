---
title: "AWS Cost Anomaly Detectionを使ってコスト異常を検知する"
emoji: "💰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "CostAnomalyDetection"]
published: true
publication_name: "cureapp"
---

## AWS Cost Anomaly Detection とは？

AWS の請求で想定外のコストが発生していないか、心配になったことはこれまでにないでしょうか？ そういったコスト異常を検知し、通知してくれるのが Cost Anomaly Detection というサービスになります。
https://aws.amazon.com/jp/aws-cost-management/aws-cost-anomaly-detection/

Cost Anomaly Detection を事前に設定しておくことで、例えば、インフラ構成を変更後、コストが異常に増加していないかどうか、ユーザ数の増加に伴い、想定以上のコスト増になっていないかなどのさまざまなコスト異常に気づきやすくなり、対処できるようになります。

## AWS Cost Anomaly Detection の設定

AWS Organizations を使って、社内の AWS アカウント全体を一元管理している場合は、Organizations のアカウント（管理アカウント）に設定するだけで、管理下にあるすべての AWS アカウントに対してコスト異常検知ができるようになります。

設定には、**コストモニター**（モニタリング対象のタイプ）と**アラートサブスクリプション**（異常検知のしきい値と通知先）の設定が必要になります。

### コストモニター（モニタリング対象のタイプ）

コストモニターは以下の 4 つに分かれます。まず最初は、**AWS のサービス**のコストモニターを作っておけば OK だと思います。それ以外のコストモニターは、チームやプロダクトが分かれていて、それぞれで細かく設定したくなったときに検討してみるといいでしょう。

- **AWS のサービス**
  - AWS アカウント・使用するサービスごとに評価
- **連結アカウント**
  - 複数の AWS アカウントの総支出を評価（チームが管理するリソースが複数のアカウントにまたがって展開されているケースなどに有用）
- **コストカテゴリ**
  - AWS Cost Categories の機能でコストをグルーピングしたときの総支出を評価（事前にコストカテゴリの作成が必要になります）
- **コスト配分タグ**
  - コスト配分タグのキーと値のペアに合致するリソースの総支出を評価（事前にコスト配分タグの設定が必要になります）

AWS Organizations を使っており、Organizations のアカウント（管理アカウント）に設定を行った場合では、AWS のサービスのコストモニターでは**新規 AWS アカウントを追加したときの設定は特に不要**です。

![](/images/cost-anomaly-detection/cost-monitor.png)

### アラートサブスクリプション（異常検知のしきい値と通知先）

異常検知のしきい値と通知先を設定していきます。

#### 異常検知のしきい値

金額（例. $10）もしくは割合（例. 40%）でしきい値を設定できます。

- 予想支出を上回る金額
- 予想支出を上回る割合

#### 通知先の設定

アラートの通知内容に応じて、通知先が限定されています。

- **個々のアラート**
  - AWS アカウント・サービスごとにアラートが発砲されます。
  - SNS トピックの設定を行うことで、AWS Chatbot 経由で **Slack 通知させることが可能**です。
- **日次の要約 / 週次の要約**
  - 個々のアラートの内容をまとめて 1 つの**メールで通知**してくれます。個々のアラートだと件数が多すぎる、サマリがほしいなどのケースで有用です。
  - E メールアドレスの設定しかできません。

![](/images/cost-anomaly-detection/alart-subscription.jpg)
_👆 設定例（イメージ）_

![](/images/cost-anomaly-detection/cost-anomaly-individual.jpg =650x)
_👆 個々のアラートの例_

![](/images/cost-anomaly-detection/cost-anomaly-summary.jpg)
_👆 日次の要約の例_

### コスト異常検知のしきい値の決め方（参考）

最初はどのくらいのしきい値を設定していいかどうか判断がつきません。そのため、まずはコストモニターの詳細から確認できる検出履歴を参考にしてしきい値を徐々に調整していきましょう。

以下は検出履歴の例になりますが、**コストへの影響度の合計**や**影響の割合の値**は前述の異常検知のしきい値と対応しています。値の状況を確認しながら、狼アラート化しない程度にしきい値を定めましょう。

- **コストへの影響度の合計**: 「予想支出を上回る**金額**」のしきい値と比較される値
- **影響の割合**: 「予想支出を上回る**割合**」のしきい値と比較される値

![](/images/cost-anomaly-detection/cost-anomaly-detection-log.png)

### AWS CDK による設定

一応、L1 Constract の実装（Cfn 相当）は用意されていますが、CDK で記述するメリットは薄いと個人的には感じています。管理コンソールからポチポチ設定していいレベルだと思います。このあたりは好みでどうぞ。
https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ce.CfnAnomalyMonitor.html

## アラートの確認方法

コスト異常を検知した AWS アカウントもしくは Organizations のアカウント（管理アカウント）から、Cost Explorer でコストの推移をチェックし原因分析を行いましょう。

https://us-east-1.console.aws.amazon.com/cost-management/home#/cost-explorer

デフォルトでは月次で表示されますが、**日次**に切り替えて日々のコストの変化を見ることをオススメします（Cost Anomaly Detection では日次でのコスト変化を検知しているため）

以下では、いくつかのコスト変化のパターン別に解説していきます（あまりいいサンプルがなかったので、分かりにくかったらすいません…）

### コストが急激に増加したケース

CloudFront の例を挙げます。今まで、ほぼコストがかかっていなかったのに**ある地点を境に急激にコストが増加**しています。こういったケースは、その月の**無料枠**を食いつぶしてしまった場合によく見られるコスト変動です。

![](/images/cost-anomaly-detection/cost-suddenly-increase.png)

CloudFront の場合、1TB のデータ転送が無料枠として与えられています。この場合、その無料枠を 10/21 時点で食いつぶしてしまい、そこから通常の CloudFront のコストが計上されるようになったのではないか？と仮説を立てることができます。
https://aws.amazon.com/jp/cloudfront/pricing/

詳しく調査したい場合は、CloudWatch Metrics からデータ転送量のデータを取得し分析してみると良いでしょう。以下はメトリクス（`BytesDownload`）から取得したデータ転送量から、月別に累積したデータ転送量を可視化したグラフになります。

見ての通り、10/21 で累積のデータ転送が 1TB を超えていることがわかります（後から、9/30 でもコストが発生していることが発覚）以上から、無料枠を食いつぶしてしまった結果、10/21 から CloudFront の本来のコストが計上されるようになったと理解できます。

![](/images/cost-anomaly-detection/cost-suddenly-increase-2.png)

### コストが単調増加しているケース

検証目的で作ったリソースの**消し忘れたり**、**ユーザ数の増加**に伴いコストが増加した場合、コストが**単調増加**していることがわかります（もっとわかりやすい例があると良かったのですが…）

![](/images/cost-anomaly-detection/cost-monotonically-increase.jpg)

リソースの消し忘れの場合、もったいないので不要になったリソースは消しましょう。ユーザ数の増加など、コストが単調増加している場合は必要コストなので、問題ないと判断できるでしょう。

ただし、今のインフラ・アーキテクチャだとコストが増加しすぎるとわかったら、インフラ構成の見直しも検討してみるといいでしょう。

### 一過的にコストが上昇しただけのケース

Amazon Connect の例を挙げます。Amazon Connect のコストは架電・受電した時間に応じて**従量課金**されます。つまり、使った分だけコストがかかります。

以下の例では、10/26 にコストへの影響が +$7 (153%) となっていますが、一過的にコストが上昇しただけだと判断できます。
このケースは変化幅が小さいですが、変化幅が大きかったとしても、運用サイドに確認してそれだけ架電数が多かったことがわかれば、妥当なコストと判断できます。

![](/images/cost-anomaly-detection/cost-increase-once.png)
![](/images/cost-anomaly-detection/cost-increase-once-2.jpg)

## まとめ

AWS Cost Anomaly Detection を設定しておくことで、コストへの意識を高め、コスト異常のポイントを検知・対処できるようになります。

どうしてコストが増加してしまったか、その詳細を調べる上では CloudWatch Metrics などでメトリクスが取れていることも重要です（コストが急激に増加した例ではメトリクスから原因分析できたように）

しきい値の設定など、まだまだ初心者丸出しな部分もありますが、誰かの参考になったのなら幸いです 🙇‍♂
