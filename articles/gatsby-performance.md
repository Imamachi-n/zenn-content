---
title: "Gatsby製のランディングページ（LP）のパフォーマンス改善"
emoji: "🏇"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gatsby", "performance"]
published: false
publication_name: "aldagram_tech"
---

みなさんこんにちは！[アルダグラム](https://aldagram.com/about/)でエンジニアをしている今町です。

弊社では、サービスサイトの LP ページ・コーポレートサイトのような静的サイトで Gatsby を利用しています（Gatsby とは React ベースの SSG フレームワークです）

私がアルダグラムに入社して初めて行った仕事が、Gatsby 製の LP ページのパフォーマンス改善でした。入社から 3 ヶ月が経過し、当時を思い出しながらどうやってパフォーマンス改善を進めていったか共有したいと思います。また、Gatsby 製のサイトを運用している方にとって、少しでも参考になったら嬉しいです 🥰

# 現状を知る

まずは、Gatsby 製のページのパフォーマンスがどうなっているか現状を理解するところから始めてみましょう。

Google が提供している PageSpeed Insights を利用してパフォーマンスを含めた各種指標を測定できます。公開されている Web ページの URL を入力して分析を開始します。

https://pagespeed.web.dev/

最初は、以下のようにパフォーマンスが真っ赤になっていました… 😭

![gatsby_1.png](/images/gatsby-performance/gatsby_1.png)

パフォーマンスのセクションの各種指標の意味は、以下のとおりです。

| 指標                            | 内容                                                                                                                                                                                                                                                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| First Contentful Paint（FCP）   | ページの最初のコンテンツが表示されるまでの時間。<br><https://developer.chrome.com/docs/lighthouse/performance/first-contentful-paint?hl=ja><br><br>CP を **1.8 秒以下**にすることが推奨されています。<br><https://web.dev/articles/fcp?hl=ja#what-is-a-good-fcp-score>                                       |
| Total Blocking Time（TBT）      | ユーザーがページにアクセスして、実際に入力可能になるまでの時間。<br><https://developer.chrome.com/docs/lighthouse/performance/lighthouse-total-blocking-time?hl=ja><br><br>TBT を **200 ミリ秒以下**にすることが推奨されています。<br><https://web.dev/articles/tbt?hl=ja#good-score>                        |
| Speed Index（SI）               | ページの読み込み中にコンテンツが視覚的に表示される速度。<br><https://developer.mozilla.org/ja/docs/Glossary/Speed_index><br><br>SI を 3.4 秒以下にすることが推奨されています。<br><https://developer.chrome.com/docs/lighthouse/performance/speed-index?hl=ja>                                               |
| Largest Contentful Paint（LCP） | メインコンテンツ（ビューポート内の最大のコンテンツ要素）の読み込み時間。<br><https://developer.chrome.com/docs/lighthouse/performance/lighthouse-largest-contentful-paint?hl=ja><br><br>LCP を **2.5 秒以下**にすることが推奨されています。<br><https://web.dev/articles/lcp?hl=ja#what-is-a-good-lcp-score> |
| Cumulative Layout Shift（CLS）  | レイアウトシフトの度合いを表す指標。<br><https://web.dev/articles/cls?hl=ja><br><br>サイトのページ訪問の 75% 以上で CLS が **0.1 以下**であることが推奨されています。<br><https://web.dev/articles/optimize-cls?hl=ja>                                                                                       |

詳細は診断の欄に出ているので、主だった改善ポイントを以下で確認していきます 💪

![gatsby_2.png](/images/gatsby-performance/gatsby_2.png)

# 診断結果から改善ポイントを洗い出す

以下では、診断結果から改善できるポイントをピックアップした上で、解決策を例示したいと思います。

## 1. 次世代フォーマットでの画像の配信（高圧縮な画像フォーマットへの置換）

Gatsby で画像データを扱う場合、 `gatsby-plugin-image` プラグインを使うことで複数サイズ・フォーマットの画像をビルド時に自動生成し、ユーザー画面（PC・スマホ）に応じて最適なサイズの画像を表示できます。これにより、開発者側は特に意識することなく、画像の最適化を行うことができます。

`<StaticImage>` という専用のコンポーネントが用意されているので、画像を表示したい場合は `<img>` の代わりとして使用します（詳しくは公式ドキュメントを参照のこと）

https://www.gatsbyjs.com/plugins/gatsby-plugin-image/

しかし、別の React 関連のパッケージを利用しており、そのパッケージ内部で素の `<img>` タグが使われてしまっていると、Gatsby の画像最適化の恩恵を受けられません。これが原因で、スコア悪化しているという結果が出ていました。

### 解決策 1: 該当の React パッケージの置き換え

根本解決としては、該当の React パッケージを置き換えて `<StaticImage>` を使う形に改修するという方法がまず挙げられます。ただし、これは実装コストがかかるので、工数と相談になります。

### 解決策 2: 画像ファイルを高圧縮フォーマット WebP に置換

暫定対応になってしまうのですが、高圧縮の WebP の画像に差し替えることで、描画する画像サイズを圧縮できます。今回は時間をかけられなかったので、この解決策を取りました。

## 2. オフスクリーンの遅延読み込み（画像の読み込み遅延）

多くの Web サイトでは、ページを開いた際に表示されるのは**ページ上部**で、ユーザーは画面をスクロールして、下に隠れているコンテンツ（オフスクリーン）を見ていくことになると思います。

だとすると、ページ内の画像を一気に読み込まず、ブラウザに表示されている画像だけ先に読み込むことでパフォーマンス改善できそうですよね（このアラートはそれを指しています）

`gatsby-plugin-image` プラグインを使って画像を描画している場合は、遅延読み込みを自動でやってくれます。つまり、開発者側は特に意識する必要はありません。

問題になってくるのは先程と同様、別の React 関連のパッケージを利用しており、そのパッケージ内で素の `<img>` タグが使われてしまっているケースです。この場合、開発者側で lazy loading するように設定してあげる必要があります。

### 解決策: lazy loading の設定を追加

対応としては簡単で、`<img>` タグに `loading=lazy` の設定を追加するだけです。

## 3. レンダリングを妨げるリソースの除外（Google フォントの読み込みを最適化）

Gatsby では `gatsby-omni-font-loader` プラグインを使って、Google フォントなどの Web フォントの読み込みを最適化しています。このプラグインが使われているにも関わらず、blocked rendering になってしまっていてアラートが出ていました。

調べてみると、このプラグインの現存するバグであると報告されています。

https://www.gatsbyjs.com/plugins/@nathanpate/gatsby-omni-font-loader/

### 解決策: 該当パッケージにパッチを適応する

以下の `generators/getFontConfig.tsx` と `components/AsyncFonts.tsx` に修正が必要です。

- Original
  - <https://github.com/codeAdrian/gatsby-omni-font-loader/blob/main/generators/getFontConfig.tsx#L24-L30>
  - <https://github.com/codeAdrian/gatsby-omni-font-loader/blob/main/components/AsyncFonts.tsx>
- 修正版
  - <https://github.com/np36/gatsby-omni-font-loader/blob/main/generators/getFontConfig.tsx>
  - <https://github.com/np36/gatsby-omni-font-loader/blob/main/components/AsyncFonts.tsx>

オリジナルの npm パッケージにパッチを当てたい場合は、以下のパッケージをインストールし、

```tsx
npm install --dev patch-package postinstall-postinstall
```

package.json ファイルに以下のカスタムコマンドを追加します。

```tsx
{
  "scripts": {
    "postinstall": "patch-package"
    ...
  }
}
```

その上で、修正版のコードの通りに node_modules 配下にあるファイルを修正します。

最後に、以下のコマンドを実行して、パッチファイルを作成します。

```tsx
npm run patch-package gatsby-omni-font-loader
```

こうすることで、npm install が実行された後に、必ずパッチファイルが適応されるようになります。

## 4. コードサイズの圧縮（不要なパッケージの削減）

コードサイズを圧縮することで、ページを開いたときの読み込み速度を改善できます。Gatsby 向けに webpack のバンドル解析ツールが提供されています。

https://www.gatsbyjs.com/plugins/gatsby-plugin-webpack-bundle-analyser-v2/

以下は解析結果の例になります。モジュールサイズが大きなものから中身を確認し、コードサイズを削減できないか検討します。React のパッケージなど必須のものは削減できないとして、やたら大きなサイズのモジュールが存在することがわかります。今回のケースでは、axios を使っていることでモジュールサイズが肥大化していました（これはあくまで一例です）

![gatsby_3.png](/images/gatsby-performance/gatsby_3.png)

### 解決策: fetch API もしくは redaxios を採用する

axios は HTTP クライアントのパッケージです。Web サイトの場合、ブラウザ標準の fetch API が用意されているので、npm パッケージである axois を導入する必要はありません。なので、シンプルに fetch API に置き換え、axios をコードから排除することで、コードサイズを圧縮できます。

もしくは、redaxios というパッケージを使う選択肢もあります。redaxios は fetch API を内部的に使用してインターフェイスは axios に合わせてあるパッケージです。こちらのほうがコードの変更点を減らせるので、移行はしやすいでしょう。

https://github.com/developit/redaxios

## 5. gzip 圧縮して配信

gzip 圧縮したうえで Web サイトの配信を行います。ブラウザからリクエストされた JavaScript や CSS ファイルサイズを小さくできるので、結果としてダウンロード時間を短縮できます（以下は AWS の CloudFront の場合）

https://docs.aws.amazon.com/ja_jp/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html

Web サイトのホスティングに何を使用しているのかに依存するため、解決策の詳細は割愛します。

## まとめ

いかがでしたでしょうか？今回紹介したのは、以下の 5 点になります。Gatsby に限らず汎用的に Web サイト構築時に意識したい内容だと思います。

- 高圧縮な画像フォーマットへの置換
- オフスクリーンの遅延読み込み
- レンダリングを妨げるリソースの除外
- コードサイズの圧縮
- gzip 圧縮して配信

気になる結果ですが、真っ赤だったパフォーマンスのスコアが 10 ポイント以上改善できました。劇的に改善したかというとそうではないのですが、地道な積み重ねで Web ページをより良い状態にしていきたいですね。

![gatsby_4.png](/images/gatsby-performance/gatsby_4.png)

もっとアルダグラムエンジニア組織を知りたい人、ぜひ下記の情報をチェックしてみてください！
