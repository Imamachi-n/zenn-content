---
title: "AWS Lambda でのトラブル事例・思い出話をまとめて見ました"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lambda"]
published: false
---

AWS Lambda といえば、API Gateway + Lambda を組み合わせたサーバレスなバックエンドの構築に使われたり、サクッとバッチ処理を作るときに使われたりと、インフラをあまり意識せずにコードの実行環境として使える便利なサービスです（と思っています）

今回は、実際の業務やプライベートを含めて、システム運用してたらトラブルに見舞われた AWS Lambda での苦い思い出話をまとめようと思います。初歩的なミスも多分に含まれると思うので、温かいで目で見てもらえるとー。

また、ここに書いてある解決策についても最善とは言えないものもあるかもしれません。その際は、コメントで指摘してもらえるとうれしいです。

# 最大実行時間を超えてタイムアウトになった

Lambda 関数をバッチ処理として使っているケースで、Lambda 関数の最大実行時間である 15 分を超えてしまったケースです。

初歩的なミスとして、DB のテーブルにインデックスを貼り忘れた結果、ユーザ数の増加に伴いパフォーマンスが急激に悪化、Lambda 関数がタイムアウトしかけたことがありました（Lambda 関数の Duration のメトリクスがすごいことになってました…）

![](/images/lambda-problems/lambda_duration_max.png)

誰にも間違いはあります。問題になる前にできること、根本的な解決策として何が考えられるでしょうか？

## まずは観測。問題になる前に検知しよう！

Lambda 関数のデフォルト設定で、実行時間（Duration）の CloudWatch メトリクスが設定されています。上図のようなメトリクスを、AWS 管理コンソールの Lambda 関数の画面から確認できます。

目視でチェックするのも良いですが、CloudWatch Alarm を設定しておくと安心です。例えば、実行時間が上限の 90% を超えた場合にアラートを発火させる設定を行い、CloudWatch Alarm → SNS → AWS ChatBot 経由で Slack 通知することもできます。

実際、このアラートを設定しておいたおかげで、トラブルになる前に対策を打つことできました。

##

# Lambda 関数にデプロイするコードがでかすぎてアップロードできない

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

![](/images/lambda-problems/step_functions_img.png)

# まとめ

Lambda 関数のクォータ（制限）の情報はしっかり確認しておきましょう。Lambda 関数は年々アップデートを繰り返してより便利になり、活用の幅が広がっていますが、魔法の杖というわけではありません。

機能的な制約もあるため、Lambda について正しく理解した上で、正しい使い方をしていきたいですね。最後に、AWS の公式ドキュメントを貼っておきます。  
[Lambda クォータ](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html)
