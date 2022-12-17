---
title: "AWS Lambda でのトラブル事例とその解決策（案）についてまとめて見ました"
emoji: "😱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["lambda"]
published: true
---

:::message
こちらは [CureApp Advent Calendar 2022](https://qiita.com/advent-calendar/2022/cureapp) 17 日目の記事です。
:::

AWS Lambda といえば、API Gateway + Lambda を組み合わせたサーバレスなバックエンドの構築に使ったり、サクッとバッチ処理を作るときに使ったりと、インフラをあまり意識せずにコードの実行環境として使える便利なサービスです（と思っています）

今回は、実際の業務やプライベートを含めて、システム運用してたらトラブルに見舞われた AWS Lambda での苦い経験とその対策をまとめようと思います（ケースをある程度網羅するためにフィクションも含みます）。初歩的なミスも多分に含まれると思うので、温かいで目で見てもらえるとー。

また、ここに書いてある解決策についても最善とは言えないものもあるかもしれません。その際は、コメントで指摘してもらえるとうれしいです。

それでは早速見ていきましょう！

# 最大実行時間を超えて処理がタイムアウトした

Lambda 関数をバッチ処理として使っている場合、Lambda 関数の最大実行時間が 15 分であることに注意が必要です（2022/12 月現在。デフォルトでは Node.js だと 3 秒に設定されている点も注意）  
[Lambda クォータ - 関数の設定、デプロイ、実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#function-configuration-deployment-and-execution)

過去のやらかしとして、初歩的なミスで、DB のテーブルにインデックスを貼り忘れた結果、ユーザ数の増加に伴いパフォーマンスが急激に悪化、Lambda 関数がタイムアウトしかけたことがありました（Lambda 関数の実行時間（Duration）のメトリクスがすごいことになってました…）

![](/images/lambda-problems/lambda_duration_max.png =500x)

誰にでもミスはあります。問題になる前にできること、根本的な解決策としてそれぞれ何が考えられるでしょうか？

## まずは観測。問題になる前に検知しよう！

Lambda 関数のデフォルト設定で、実行時間（Duration）の CloudWatch メトリクスが見れます。上図のようなメトリクスを、AWS 管理コンソールの Lambda 関数の画面からも確認できます。

目視でチェックするのも良いですが、CloudWatch Alarm を設定しておくとより安心です。例えば、実行時間が上限の 90% を超えた場合にアラートを発火させる設定を行い、CloudWatch Alarm → SNS → AWS ChatBot 経由で Slack 通知してみる。  
また、PagerDuty や Opsgenie などを使って、一箇所に通知をまとめても良いでしょう。

![](/images/lambda-problems/cloudwatch_alarm.png =600x)

実際、このアラートを設定しておいたおかげで、トラブルになる前に対策を打つことできました。

## 問題を解決するにはどうすればいいのか？

上記の例では、DB 側のインデックスの問題なので解決は比較的簡単です。または、処理を並列化させて 1 つの Lambda 関数にかける処理時間を短縮することでも問題は解消可能です。

### TCP keepAlive を有効化する（Node.js 限定）

Node.js の Lambda 関数の場合、HTTP/HTTPS エージェントは `keepAlive` 接続がデフォルトで無効になっています。`AWS_NODEJS_CONNECTION_REUSE_ENABLED` 環境変数を `1` に設定すると、 keepAlive 接続が有効化され、接続の再利用による接続コストの軽減ができます。
[Node.js で Keep-alive を使用して接続を再利用する - AWS SDK for JavaScript](https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/node-reusing-connections.html)  
[AWS Lambda の Node.js ランタイムで TCP 接続を使いまわそう!(AWS_NODEJS_CONNECTION_REUSE_ENABLED) | DevelopersIO](https://dev.classmethod.jp/articles/aws-lambda-node-reusing-connections-keepalive/)

AWS CDK で Lambda 関数を作成しており、`aws-cdk-lib.aws_lambda_nodejs` を使っている場合は、デフォルトの設定で `AWS_NODEJS_CONNECTION_REUSE_ENABLED = 1` になっているので気にする必要はありません。  
[class NodejsFunction (construct) · AWS CDK](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs.NodejsFunction.html#awssdkconnectionreuse)

### Lambda 関数のパフォーマンス最適化（割り当てメモリ量を引き上げる）

その他に改善できることとしては、シンプルに割り当てメモリ量を増やしてみることです。Lambda 関数はメモリ量に応じて関数がしようできる仮想 CPU 量が決まります。つまり、メモリ量を増やすだけで処理パフォーマンスが上がります（CPU パフォーマンスに依存しない処理は早くならないですが）  
[Operating Lambda: パフォーマンスの最適化 – Part 2 | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/operating-lambda-performance-optimization-part-2/)

コスト的にも、メモリ量 × 実行時間で課金されるので、メモリ量を増やしてその分だけ実行時間が短縮されれば、そこまでコスト増にならないはずです。例えば、以下のようなイメージです（コストの値は見やすいように適当な値にしてあります）

| メモリ量 | 実行時間 | コスト |
| -------- | -------- | ------ |
| 128 MB   | 12 sec   | $10    |
| 256 MB   | 6 sec    | $10    |
| 512 MB   | 3 sec    | $10    |
| 1024 MB  | 1.5 sec  | $10    |

最適なコストを叩き出す、適切なメモリ量を厳密に決めたい場合は、AWS Lambda Power Tuning というツールがあります。これは、Step Functions を利用して、異なるメモリ量で該当の Lambda 関数を実行したときの実行時間からコストバランスが取れた最適なメモリ量を割り出してくれます。  
[alexcasalboni/aws-lambda-power-tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)

ツールを用意するのがめんどくさい人は、[AWS Compute Optimizer](https://aws.amazon.com/jp/compute-optimizer/) というサービスもあります。AWS 管理コンソールから Compute Optimizer をオプトインすると、自動的に Lambda 関数を解析してリソース不足・過剰を検知してくれます。

例えば、以下のように最適なメモリ量をサジェストしてくれます。
![](/images/lambda-problems/compute_opt.png)

他にも、Lambda 関数のパフォーマンスチューニングの方法はあります。以下に様々なユースケースが載っていますので参考までにペタリ。
[AWS Lambda Performance Tuning Deep Dive - Speaker Deck](https://speakerdeck.com/_kensh/aws-lambda-performance-tuning-deep-dive?slide=45)  
[AWS Lambda Performance Tuning Deep Dive〜本当に知りたいのは”ここ”だった 〜(AWS-46) - YouTube](https://www.youtube.com/watch?v=SUNbX89fuRw)

とはいえ、処理時間が長くなりすぎて、どうしても Lambda 関数では処理しきれないケースも出てくるでしょう。そういった場合は、どんなサービスが使えるでしょうか？以下で、2 パターンを考えてみました。

![](/images/lambda-problems/ecs_vs_batch.png =250x)

### ECS Task

ECS でサーバを運用しているケースでは、ECS task としてバッチ処理を実行するのが比較的簡単に設定できて、第一選択肢になるのではと思います。複雑なワークフローを組みたい場合は Step Functions と組み合わせることも可能です。  
[Step Functions で Amazon ECS または Fargate タスクを管理する
](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/connect-ecs.html)

### AWS batch

ECS の運用環境がない場合の選択肢として、AWS Batch があります。AWS Batch はフルマネージドなバッチサービスで、内部的には ECS + キューを組み合わせた構成になっています。こちらも、Step Functions と組み合わせて複雑なワークフローを組むこともできます。  
[AWS Batch](https://aws.amazon.com/jp/batch/)

# メモリ不足（out of memory）になってしまった

こちらも文字通り、メモリ不足の問題です。Lambda 関数に割り当てられるメモリは 128 MB〜10,240 MB（10 GB）までです（2022/12 月現在）。メモリを消費する処理を Lambda 関数を使って処理している場合は要注意です。  
[Lambda クォータ - 関数の設定、デプロイ、実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#function-configuration-deployment-and-execution)

## まずは観測。問題になる前に検知しよう！

困ったことに、Lambda 関数のデフォルト設定では、消費メモリ量を CloudWatch メトリクスから取得できません。拡張メトリクスである Lambda Insights を別途導入する必要があります。

![](/images/lambda-problems/lambda_memory.png =600x)

### Lambda Insights

気になるお値段ですが、Lambda Insights のコストは、主に CloudWatch 分の料金になります。内訳は「8 つのメトリクス分」と「関数呼び出しごとに約 1 KB のログデータ分」となります。
[Amazon CloudWatch での Lambda Insights の使用](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/monitoring-insights.html)

概算ですが、以下の条件だと $3/月 くらいです（CloudWatch のコスト）

- Lambda: 1 個
- リクエスト数: 1,000/時間

詳細な計算は AWS Pricing Calculator でコスト見積もりすると良いです。
[AWS Pricing Calculator - Configure Amazon CloudWatch](https://calculator.aws/#/addService/CloudWatch)

正直、めちゃ高いわけじゃないので、必要に応じて気軽に導入して良いと思っています。

### AWS CDK から Lambda Insights を導入

AWS CDK から、`aws-cdk-lib.aws_lambda` や `aws-cdk-lib.aws_lambda_nodejs` モジュールで Lambda Insights を ON にできます。もちろん、AWS 管理コンソールから画面ポチポチで設定も可能です。

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

## 問題を解決するにはどうすればいいのか？

まずは、メモリ消費を減らすようにコードを工夫したり、処理を分割して別々の Lambda 関数に処理させられないか検討してみて、それでも無理な場合は、こちらも最大実行時間の超過のケースと同様に、ECS task もしくは AWS Batch が選択肢に入ってきそうです。

# Lambda の初回起動が遅い（コールドスタートのレイテンシが気になる）

API Gateway + Lambda を組み合わせたサーバレスなバックエンドを構築した場合、Lambda の初回起動（コールドスタート）が遅いことが気がかりになるケースがあります。多少の遅延が許されるシステムなら無視することもできますが、そうでないユースケースも多くあるでしょう。

Lambda 関数は Firecracker と呼ばれる Rust で書かれたサーバレス特化の MicroVM を使って実行環境を提供しています。起動速度は 125 ms（2018 年時点）とされており、現在はさらに高速化されています。ですが、Lambda 上で実行する言語の違いによっても起動速度が変わってきそうです。  
[Firecracker – サーバーレスコンピューティングのための軽量な仮想化機能](https://aws.amazon.com/jp/blogs/news/firecracker-lightweight-virtualization-for-serverless-computing/)  
[Lambda の裏側を知りたい人にオススメ Firecracker に関する論文「Firecracker: Lightweight Virtualization for Serverless Applications」の紹介](https://dev.classmethod.jp/articles/firecracker-lightweight-virtualization-for-serverless-applications/)

## Lambda 関数のコールドスタートのレイテンシーを測定してみる

まずは、Lambda 関数の response が遅い理由が、コールドスタートのレイテンシーに起因するものなのかどうか測定してみましょう。測定ツールは何でもいいと思いますが、ここでは以下の記事で使われていた [hey](https://github.com/rakyll/hey) というツールを使ってみました。  
[VPC Lambda のコールドスタートにお悩みの方へ捧ぐコールドスタート予防のハック Lambda を定期実行するならメモリの割り当ては 1600M がオススメ？！](https://dev.classmethod.jp/articles/lambda-cold-start-avoid-hack/#toc-7)

では、並列で大量のリクエストを飛ばすことで、Lambda のコールドスタートが意図的に発生させましょう。以下のコマンドは、API Gateway に対して、合計 500 リクエストを 50 並列で実行するという意味です。Lambda 関数は Node.js の実行環境を使っています。

```
hey -n 500 -c 50 -H "Authorization: xxx" https://yyy.execute-api.ap-northeast-1.amazonaws.com/api/zzzz
```

結果を見てみると、ほとんどのリクエストが 0.3 sec 程度で終了しています。一方で、**1〜2 sec 付近**に外れ値になっているリクエストがあります。これらがコールドスタートが発生した Lambda 関数のリクエストになります。

```
Summary:
  Total:	3.1652 secs
  Slowest:	2.1503 secs
  Fastest:	0.0698 secs
  Average:	0.2702 secs
  Requests/sec:	157.9660

  Total data:	1000 bytes
  Size/request:	2 bytes

Response time histogram:
  0.070 [1]	|
  0.278 [446]	|■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.486 [3]	|
  0.694 [0]	|
  0.902 [0]	|
  1.110 [0]	|
  1.318 [15]	|■
  1.526 [14]	|■
  1.734 [0]	|
  1.942 [9]	|■
  2.150 [12]	|■

Status code distribution:
  [200]	500 responses
```

つまり、この結果から、Node.js の実行環境における Lambda 関数のコールドスタートのレイテンシーは、1〜2 sec 程度であることがわかります（これが許容範囲と見るかどうかはユースケースによる）

## 問題を解決するにはどうすればいいのか？

Lambda 関数のパフォーマンス最適化については、公式ブログでも言及されているので、こちらも参照ください。  
[Operating Lambda: パフォーマンスの最適化 – Part 1 | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/operating-lambda-performance-optimization-part-1/)

### Provisioned Concurrency によるコールドスタートの短縮化

Provisioned Concurrency は事前に立ち上がった状態（ウォームアップ（暖機）済み）の Lambda 関数を用意する機能です。事前起動する Lambda 関数の数を予約しておくことができます。

ここでいう Concurrency（同時実行数）とは、Lambda 関数のメトリクスのうち、Concurrent Executions のカウント数に該当します。なので、現在のメトリクスを参照しつつ、予約する個数を決めれば良いと思います。

![](/images/lambda-problems/concurrent_executions.png =500x)

気になるお値段ですが、以下で考えると **$70.42** となります（東京リージョンの場合）Lambda 自体が比較的お安いせいか、同時実行数次第で比較的高いコストに見えます…。  
[AWS Pricing Calculator - Configure AWS Lambda](https://calculator.aws/#/addService/Lambda)

- 同時実行数: **10 個**
- プロビジョニングされた同時実行が有効になっている時間: **720 時間/月**（1 ヶ月まるまる有効）
- リクエスト数: **10 万**
- リクエスト処理時間: **1 sec**
- メモリ量: **512 MB**

### Lambda SnapStart を使う

Lambda SnapStart は関数のスナップショット（キャッシュ）を保存しておき、Lambda 関数を呼び出す前にそのスナップショットを復元し起動速度を高速化する機能です。

2022/12 月現在では、Corretto (java11) ランタイムを利用する Java でのみ使用可能です。Lambda SnapStart を有効にすると、追加コストなしで最大で 10 倍の速さで起動できると書かれています。  
[New – Lambda SnapStart で Lambda 関数を高速化 | Amazon Web Services ブログ](https://aws.amazon.com/jp/blogs/news/new-accelerate-your-lambda-functions-with-lambda-snapstart/)

難しい設定は不要で、ただ SnapStart の機能を ON にするだけです。

# Lambda 関数の呼び出しペイロード（request / response）の上限を超過した

こちらも API Gateway + Lambda でサーバレスなバックエンドを構築した際のトラブルです。ある API でクソデカ JSON データを response として返しており、ある時以下のエラーが出ました。

[413 Payload Too Large - HTTP | MDN](https://developer.mozilla.org/ja/docs/Web/HTTP/Status/413)

```
[ERROR] [1661317406359] LAMBDA_RUNTIME Failed to post handler success response. Http response code: 413.
```

Lambda の呼び出し Payload（リクエスト・レスポンス）の上限は 6 MB と決まっており、これを超過した場合、上記のような 413 エラーが発生します（2022/12 月現在。そもそも、そんな大きなデータを返すなという話かもしれませんが…）  
[Lambda クォータ - 関数の設定、デプロイ、実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#function-configuration-deployment-and-execution)

## 問題を解決するにはどうすればいいのか？

JSON を response として返している場合は、ページングして 1 response あたりのデータ量を削減するという方法や、そもそも不要なデータを含んでいる場合は response データを削減してみるなどの方法が考えられます。

一方で、画像データのように、1 ファイルあたりのデータ量が 6 MB を超えてしまう場合は、以下の方法が考えられます。

### S3 の Pre-Signed URL を使う

lambda 関数から直接 JSON データを返すのやめて、いったん S3 に保存します。lambda 関数から S3 のデータにアクセスするための Pre-Signed URL を発行し、データの代わりに URL を返すようにします。
![](/images/lambda-problems/pre_signed_url.png =600x)
[Lambda で 6MB を超えるデータを Return できなかったので、S3 の Pre-Signed URL を使った話](https://dev.classmethod.jp/articles/lambda-over-6mb-response-use-s3-pre-signed-url/)  
[Presigned URL を利用した S3 へのファイルアップロード - KAKEHASHI Tech Blog](https://kakehashi-dev.hatenablog.com/entry/2022/03/15/101500)

# Lambda 関数にデプロイするコードサイズがでかすぎてアップロードできない

ある時、Puppeteer を Lambda 関数上で動かしたいと思い、コードを書いてデプロイしたところ、コードサイズがでかすぎて Lambda にアップロードできませんでした。Lambda 関数のコードサイズには上限があり、解凍後 **250 MB** が上限となっています（2022/12 月現在）  
[Lambda クォータ - 関数の設定、デプロイ、実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#function-configuration-deployment-and-execution)

## 問題を解決するにはどうすればいいのか？

まずは、コードサイズを削減する方法を検討してみます。例えば、Puppeteer の場合は [alixaxel/chrome-aws-lambda](https://github.com/alixaxel/chrome-aws-lambda) という Lambda 向けの Chromium のバイナリを配布してくれているので、それを利用できます。  
[ヘッドレス Chrome を AWS Lambda 上の Puppeteer から操作してみた | DevelopersIO](https://dev.classmethod.jp/articles/run-headless-chrome-puppeteer-on-aws-lambda/)

### lambda コンテナイメージの作成

250 MB のコードサイズの上限をどうしても超えてしまう場合の別の選択肢として、lambda コンテナイメージを自作して使うという手もあります。この仕組を使うと、コードサイズの上限は 10 GB まで許容できます。lambda 関数の手軽さはどこへいった…という気持ちになりますが…。  
[Lambda コンテナイメージの作成 - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/images-create.html)

# ファイルディスクリプタの上限を超えてしまった

ある時、以下のエラーが出ている事に気づきました。よくよく調べてみると、`Promise.all` で API を叩きまくっている箇所があり、結果としてファイルディスクリプタの上限を超えてしまい、処理落ちしているとわかりました。

```js
{
  "errorType": "Runtime.UnhandledPromiseRejection",
  "errorMessage": "FetchError: request to https://hoge.com/api/hoge failed, reason: connect EMFILE 10.111.111.111:443 - Local (undefined:undefined)",
  "trace": [
    "Runtime.UnhandledPromiseRejection: FetchError: request to https://hoge.com/api/hoge failed, reason: connect EMFILE 10.111.111.111:443 - Local (undefined:undefined)",
    "    at process.<anonymous> (/var/runtime/index.js:100:10)",
    "    at process.emit (events.js:400:28)",
    "    at process.emit (domain.js:475:12)",
    "    at processPromiseRejections (internal/process/promises.js:245:33)",
    "    at processTicksAndRejections (internal/process/task_queues.js:96:32)"
  ]
}
```

## 問題を解決するにはどうすればいいのか？

`Promise.all` で無制限に API を叩きまくるのは避けて、並行処理させる数を制限しましょう…。

# Lambda 関数の tmp ストレージが足りなくなった

Lambda 関数に一時的にファイルを保存したいとき、`/tmp` ディレクトリが使えます。ただ、デフォルトでは 512 MB までしか使えないので、一時ファイルを作りまくるとストレージ不足で Lambda 関数の処理がエラーになってしまうことがあります。  
[Lambda クォータ - 関数の設定、デプロイ、実行](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#function-configuration-deployment-and-execution)

## 問題を解決するにはどうすればいいのか？

一時ファイルを最後に削除して、ゴミファイルが残らないようにする。より大きなストレージが必要な場合は、10,240 MB（10 GB）までストレージ量を増やすこともできます。

# 同時実行数のデフォルト 1000 個を超えてしまった

バッチ処理でガンガン lambda 関数を使っていて、いつの間にか同時実行数が 1000 個を超過。他の lambda 関数にも影響が出てしまった…なんて話を聞いたことがあります。

1 AWS アカウントあたりの lambda 関数のデフォルトの同時実行数は 1000 と決まっています。これを超えると lambda 関数をこれ以上起動させることができなくなります。  
[Lambda クォータ - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/gettingstarted-limits.html#compute-and-storage)

## まずは観測。問題になる前に検知しよう！

AWS アカウント単位での Lambda 関数の同時実行数（Concurrent Executions）は、CloudWatch メトリクスで確認可能です。なので、CloudWatch Alarm で同時実行数が上限の 90% を超えたらアラームを発砲するといったような通知を整備しておきましょう。
![](/images/lambda-problems/lambda_cloudwatch.png)

## 問題を解決するにはどうすればいいのか？

デフォルトの同時実行数を超えそうになった場合でも、AWS に問い合わせると、同時実行数は引き上げることができます（公式ドキュメントによると数万まで引き上げられそう？）

# 親 Lambda から子 Lambda を（繰り返し）呼び出してた

最後に、Lambda 関数内から別の Lambda 関数を呼び出さないほうが良いという教訓です。

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

![](/images/lambda-problems/lambda-sqs-lambda.png =350x)

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
