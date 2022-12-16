---
title: "AWS Lambda でのトラブル事例・思い出話をまとめて見ました"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lambda"]
published: false
---

AWS Lambda といえば、API Gateway + Lambda を組み合わせたサーバレスなバックエンドの構築に使われたり、サクッとバッチ処理を作るときに使われたりと、インフラをあまり意識せずにコードの実行環境として使える便利なサービスです（と思っています）

今回は、実際の業務やプライベートを含めて、システム運用してたらトラブルに見舞われた AWS Lambda での苦い思い出話をまとめようと思います（ケースをある程度網羅するためにフィクションも含みます）。初歩的なミスも多分に含まれると思うので、温かいで目で見てもらえるとー。

また、ここに書いてある解決策についても最善とは言えないものもあるかもしれません。その際は、コメントで指摘してもらえるとうれしいです。

# 最大実行時間を超えて処理がタイムアウトした

Lambda 関数をバッチ処理として使っている場合、Lambda 関数の最大実行時間が 15 分であることに注意しましょう（デフォルトでは Node.js だと 3 秒に設定されている点も注意）

初歩的なミスとして、DB のテーブルにインデックスを貼り忘れた結果、ユーザ数の増加に伴いパフォーマンスが急激に悪化、Lambda 関数がタイムアウトしかけたことがありました（Lambda 関数の実行時間（Duration）のメトリクスがすごいことになってました…）

![](/images/lambda-problems/lambda_duration_max.png =500x)

誰にでも間違いはあります。問題になる前にできること、根本的な解決策として何が考えられるでしょうか？

## まずは観測。問題になる前に検知しよう！

Lambda 関数のデフォルト設定で、実行時間（Duration）の CloudWatch メトリクスが見れます。上図のようなメトリクスを、AWS 管理コンソールの Lambda 関数の画面からも確認できます。

目視でチェックするのも良いですが、CloudWatch Alarm を設定しておくとより安心です。例えば、実行時間が上限の 90% を超えた場合にアラートを発火させる設定を行い、CloudWatch Alarm → SNS → AWS ChatBot 経由で Slack 通知してみる。  
また、PagerDuty や Opsgenie などを使って、一箇所に通知をまとめても良いでしょう。

![](/images/lambda-problems/cloudwatch_alarm.png =600x)

実際、このアラートを設定しておいたおかげで、トラブルになる前に対策を打つことできました。

## 問題を解決するのはどうすればいいのか？

上記の例では、DB 側のインデックスの問題なので解決は比較的簡単です。または、処理を並列化させて 1 つの Lambda 関数にかける処理時間を短縮することでも問題は解消可能です。

ただ、処理時間が長くなりすぎて、どうしても Lambda 関数では処理しきれないケースも出てくるでしょう。そういった場合は、どんなサービスが使えるでしょうか？

![](/images/lambda-problems/ecs_vs_batch.png =250x)

### ECS Task

ECS でサーバを運用しているケースでは、ECS task としてバッチ処理を実行するのが比較的簡単に設定できて、第一選択肢になるのではと思います。複雑なワークフローを組みたい場合は Step Functions と組み合わせることも可能です。  
[Step Functions で Amazon ECS または Fargate タスクを管理する
](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/connect-ecs.html)

### AWS batch

ECS の運用環境がない場合の選択肢として、AWS Batch があります。AWS Batch はフルマネージドなバッチサービスで、内部的には ECS + キューを組み合わせた構成になっています。こちらも、Step Functions と組み合わせて複雑なワークフローを組むこともできます。  
[AWS Batch](https://aws.amazon.com/jp/batch/)

# メモリ不足（out of memory）になってしまった

こちらも文字通り、メモリ不足の問題です。Lambda 関数に割り当てられるメモリは 128 MB〜10,240 MB（10 GB）までです。メモリを消費する処理を Lambda 関数を使って処理している場合は要注意です。

## まずは観測。問題になる前に検知しよう！

困ったことに、Lambda 関数のデフォルト設定では、消費メモリ量を CloudWatch メトリクスから取得できません。拡張メトリクスである Lambda Insights を別途導入する必要があります。

### Lambda Insights

![](/images/lambda-problems/lambda_memory.png =600x)

AWS CDK から、`aws-cdk-lib.aws_lambda` や `aws-cdk-lib.aws_lambda_nodejs` モジュールで Lambda Insights を ON にできます。

コード例としては以下になります。詳しくは、公式ドキュメントを参照のこと。  
[aws-cdk-lib.aws_lambda module](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda-readme.html#lambda-insights)

```js
new lambda.Function(this, "MyFunction", {
  runtime: lambda.Runtime.NODEJS_18_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset(path.join(__dirname, "lambda-handler")),
  insightsVersion: lambda.LambdaInsightsVersion.VERSION_1_0_98_0,
});
```

# Lambda の初回起動が遅い（コールドスタートのレイテンシが気になる）

# 同時実行数のデフォルト 1000 個を超えてしまった

# Lambda 関数にデプロイするコードがでかすぎてアップロードできない

コンテナイメージのコードパッケージサイズ

# Lambda 関数の tmp ストレージが足りなくなった

# Lambda 関数の呼び出しペイロード（request / response）の最大値を超過した

# ファイルディスクリプタの上限を超えてしまった

# 親 Lambda から子 Lambda を（繰り返し）呼び出す

まずはじめに、Lambda 関数内から別の Lambda 関数を呼び出さないほうが良いという教訓です。

ある 1000 件くらいのデータ処理を行う Lambda 関数で書かれたバッチ処理がありました。この Lambda 関数では、親 Lambda が 1000 件のデータを 50 件のデータに分割し、内部で子 Lambda を呼び出してその 50 件のデータを渡して処理させる、ということをやっていました。

擬似的なコードを書くと以下になります。

```js
import { Lambda } from "aws-sdk";
const lambda = new Lambda({ region: "ap-northeast-1" });

while (data.length > 0) {
  const chunk = data.slice(0, 10);
  await lambda
    .invoke({
      FunctionName: FUNC_NAME,
      Payload: JSON.stringify(chunk),
      InvocationType: "Event", // 非同期呼び出し
    })
    .promise();
}
```

一見すると問題ないかのように見えます。しかし、ある時事件は起こりました。東京リージョンのある AZ で Lambda の障害が発生したのです（この障害は完全に Lambda が動かなくなるわけではなく、**Lambda 関数の実行中に処理落ちする頻度が増える**というたぐいの障害だったと記憶しています）

一部の子 Lambda 関数は正常に実行され処理が終了していましたが、親 Lambda 関数が処理途中でエラーになっていました。Lambda 関数は非同期呼び出ししていたため、エラーが発生するとデフォルトで 2 回までリトライが自動的に走ります。結果として、子 Lambda 関数の呼び出しが再度発生し、同じ処理が繰り返し実行されてしまうというトラブルに見舞われました。

例えば、アプリへの PUSH 通知で上記のような仕組みを使っていた場合、同一のユーザに同一のメッセージが繰り返し通知されるという嫌がらせが発生してしまいます。当然、ユーザ体験の毀損となり、クレームに繋がります…。

## どうすればよかったのか？

Lambda 関数内で Lambda 関数を呼び出す設計はアンチパターンとして AWS の公式ブログで紹介されています。  
[Operating Lambda: イベント駆動型アーキテクチャにおけるアンチパターン – Part 3](https://aws.amazon.com/jp/blogs/news/compute-operating-lambda-anti-patterns-in-event-driven-architectures-part-3/)

Lambda 関数内で Lambda 関数を呼び出す上記の設計では、エラーハンドリングが難しくなり、予期せぬトラブルに見舞われる恐れがあります。そこで以下のような設計パターンを考える必要があります。

### Lambda → SQS → Lambda という構成に変更する

1 つ目は、2 つの Lambda 関数の処理の間に SQS をはさみ、1 つ目の Lambda 関数が処理時間のかかる処理を複数の分割し Queue に登録し、Queue 毎に 2 つ目の別の Lambda 関数で処理させるという設計です。こちらはよくあるパターン。
[AWS Solutions - aws-lambda-sqs-lambda](https://docs.aws.amazon.com/solutions/latest/constructs/aws-lambda-sqs-lambda.html)

### Step Functions で処理ワークフローを作る

2 つ目は、Step Functions を使う方法です。こちらは元のコードに近い仕組みを Step Functions のワークフローとして記述するカタチになります。  
[Lambda を使用してループを反復する](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/tutorial-create-iterate-pattern-section.html)

今回のユースケースとしては複雑すぎるかもしれませんが、3 つの Lambda 関数の処理を逐次実行するなどのより複雑なワークフローを設計したい場合では、Step Functions は有用です。

また、ワークフロー図が可視化され、エラーが発生した場合はどのステップでエラーになったのか視覚的に把握できる点も Step Functions ならではの良さです。

![](/images/lambda-problems/step_functions_img.png =250x)

# まとめ

Lambda 関数のクォータ（制限）の情報はしっかり確認しておきましょう。Lambda 関数は年々アップデートを繰り返してより便利になり、活用の幅が広がっていますが、魔法の杖というわけではありません。

機能的な制約もあるため、Lambda について正しく理解した上で、正しい使い方をしていきたいですね。最後に、AWS の公式ドキュメントを貼っておきます。  
[Lambda クォータ](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html)
