---
title: "Next.jsのビルド速度を改善したい〜Next.jsのTrace情報を分析してボトルネックとなっている処理を特定してみる"
emoji: "🚄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "docker", "githubactions"]
published: true
published_at: 2025-08-07 09:00
publication_name: "aldagram_tech"
---

こんにちは！[アルダグラム](https://aldagram.com/about/)でエンジニアをしている今町です。

ある日、Next.js で作られた Web アプリケーションをデプロイしていたときの話。
「なんか、Next.js のビルド速度、遅くない…？」
ということに気づきました（全体のデプロイ時間のうち、7 割がビルド時間になっていました）

そんなわけで、Next.js のビルド速度を改善することにしました。

## Next.js のビルド時間が遅い原因（ボトルネック）の調査

まずは、Next.js のビルド時間のうち、どこの処理がボトルネックになっているのか特定する必要があります。Next.js は `.next/trace` にトレース情報を自動で出力してくれます。このトレース情報を分析することで、ビルド時の処理毎の処理時間を割り出すことができます。

### Next.js のビルドにおけるトレース情報の用意

ローカル環境で Next.js のビルドを実行してみて、ビルド処理における Next.js のトレース情報を取得してみましょう（`.next/trace` に trace 情報が出力されます）

```bash
npm run build
```

`.next/trace` に出力されたデータを見てみると、以下のような JSON で記述されたデータになっています。正直、人間が見てわかるデータになっていないですね…。見にくいデータなので、データ加工を行う必要があります。

```json:.next/trace
// 見にくいデータ
[{"name":"generate-buildid","duration":96,"timestamp":356037304708,"id":4,"parentId":1,"tags":{},"startTime":1748005352098,"traceId":"6783aff358590a83"},{"name":"load-custom-routes","duration":127,"timestamp":356037304839,"id":5,"parentId":1,"tags":{},"startTime":1748005352098,"traceId":"6783aff358590a83"},...]
```

Next.js の GitHub リポジトリに、このトレース情報を tree 構造のデータ（人間が見れるデータ）に変換するスクリプトがあるので、それを活用します。
[https://github.com/vercel/next.js/blob/canary/scripts/trace-to-tree.mjs](https://github.com/vercel/next.js/blob/canary/scripts/trace-to-tree.mjs)

必要なパッケージを手元にインストールします。

```bash
npm install picocolors -D
npm install event-stream -D
```

元のソースコードのままだと手元で動かないので、一部ソースコードを修正します（`picocolors` のインポート方法を修正。なぜか、next.js のリポジトリ内にある picocolors.js ファイルを参照するカタチになっているので、その部分を修正します）

```diff:trace-to-tree.mjs
- import {
-   bold,
-   blue,
-   cyan,
-   green,
-   magenta,
-   red,
-   yellow,
- } from '../packages/next/dist/lib/picocolors.js'
+ import pkg from "picocolors";
+
+ const { bold, blue, cyan, green, magenta, red, yellow } = pkg;
```

スクリプトを用意できたら、以下のコマンドを実行します。

```bash
node trace-to-tree.mjs .next/trace
```

以下のように trace ツリーが標準出力されるので、ファイルに保存しておきましょう。

![image.png](/images/nextjs-build-perf/1.png =700x)

### AI にトレース情報を解釈させる（オプション）

tree データの状態でも十分、人間が読めるカタチになっていますが、どこが処理上のボトルネックになっているのか AI に整理してもらうと楽です。
実際に AI にデータを読み込ませると、

1. 型チェック
2. Lint チェック
3. バンドル処理（Webpack コンパイル）

の３つが処理上のボトルネックであることがわかりました（今回扱った Next.js アプリケーションでの話です）
※ Next.js のビルド（`npm run build`）では、型チェックと Lint チェックが自動で実行されるようになっている。

## Next.js のビルドパフォーマンスの改善

それでは、ここからはトレース情報の分析に基づいて、Next.js のビルドパフォーマンスを改善してみましょう。
以下はあくまで具体例なので、Next.js アプリケーションによって判断してください。

### 1. 型チェック

CI で型チェックは実施済みのため、検証環境へのデプロイ時に限り、型チェックを Skip するようにしました（一方で、本番環境へのデプロイでは、型チェックをスキップしないようにしました）
`next.config.ts` の設定ファイルに `typescript.ignoreBuildErrors: true` の設定を追加することで、型チェックをスキップできます。
[https://nextjs-ja-translation-docs.vercel.app/docs/api-reference/next.config.js/ignoring-typescript-errors](https://nextjs-ja-translation-docs.vercel.app/docs/api-reference/next.config.js/ignoring-typescript-errors)

以下は、設定例になります。

```tsx:next.config.ts
module.exports = {
  typescript: {
    // 本番環境へのデプロイ以外では、型チェックをスキップする
    ignoreBuildErrors: process.env.NODE_ENV !== "production",
  },
  // その他の設定
};
```

### 2. Lint チェック

CI で Lint チェックは実施済みのため、Lint チェックを Skip するようにしました。
`next.config.ts` の設定ファイルに `eslint.ignoreDuringBuilds: true` の設定を追加することで、Lint チェックをスキップできます。
[https://nextjs.org/docs/app/api-reference/config/next-config-js/eslint](https://nextjs.org/docs/app/api-reference/config/next-config-js/eslint)

```tsx:next.config.ts
module.exports = {
  eslint: {
    ignoreDuringBuilds: true,
  },
  // その他の設定
};
```

### 3-a. バンドル処理（Rust 製のバンドラーに置き換える）

結論、Vercel が開発中の Turbopack が `next build` に完全対応するまで待つことにしました（背景は以下）

今回対象である Next.js のコードのバンドルには Webpack を使っていました。なので、Turbopack や Rspack のような Rust 製のバンドラーに変更することで速度改善が見込める可能性があります。

@[card](https://buildersbox.corp-sansan.com/entry/2025/04/14/110000)

Rspack は Webpack の互換性を意識した後継バンドラーなんですが、Next.js との integration はまだ experimental のステータスとなっており、本番投入するには二の足を踏んでしまいます。

@[card](https://nextjs.org/docs/community/rspack)

一方で、Vercel が開発中の Turbopack ですが、Next.js 15.4 でようやく `next build` に対する α 版がリリースされました。Next.js 16 で β 版に到達する想定となっています。こちらも、本番投入するには時期尚早といった状況です。

@[card](https://nextjs.org/blog/next-15-4#turbopack-builds)

個人的な主観としては、Next.js の開発元である Vercel が開発中の Turbopack を選択したほうが良さそうだと思っています。思っていますが、Turbopack は Webpack との互換性がないのが懸念点としてあります（マイグレーションで苦労しそう…）

### 3-b. バンドル処理（各種パッケージの見直し）

Next.js の Bundle Analyzer プラグインを導入することで、バンドルの解析ができます。
@[card](https://nextjs.org/docs/app/guides/package-bundling)

例えば、`next.config.ts` のように設定を書きます。

```ts:next.config.ts
import withBundleAnalyzer from '@next/bundle-analyzer'
import type { NextConfig } from 'next'

const bundleAnalyzerConfig = {
    enabled: process.env.ANALYZE === 'true',
}

const nextConfig: NextConfig = {
  // 既存の設定
}

module.exports = withBundleAnalyzer(bundleAnalyzerConfig)(nextConfig)
```

そのうえで、以下のようにコマンドを実行すると、バンドル情報が Web ブラウザ上で開きます。

```bash
ANALYZE=true npm run build
```

バンドルサイズを確認できたら、サイズの大きいものから順に最適化を検討します。また、トレース情報と対比しながら、バンドルに時間がかかっている箇所を特定するのもよいです。こちらも、扱っているパッケージに依存するため、詳細は割愛します。

## ビルドパフォーマンス結果

上記のもろもろの改善を行った結果、ビルド時間を以前よりも **4.5 分**短縮できました 🎉

| Before              | After              |
| ------------------- | ------------------ |
| **12.2 分（729s）** | **7.5 分（447s）** |

![image.png](/images/nextjs-build-perf/2.png =700x)

## （補足）GitHub Actions & Docker コンテナ上で Next.js のビルドを行っている場合

### Docker Buildx の利用

Docker コンテナ上で Next.js のビルドを実行している場合、キャッシュが効いていないと処理が遅くなる要因となります。
そこで、`docker buildx` を使い、キャッシュの保存先を指定することで、繰り返し Next.js のビルドをしたときにキャッシュを効かせて、ビルド時間を短縮することができます。

GitHub Actions を利用している場合、Buildx をセットアップするカスタム action が提供されているので、これを利用します。
@[card](https://github.com/docker/setup-buildx-action)

また、Docker コマンドの具体例は、以下のとおりです。

```bash
docker buildx build \
  --platform linux/arm64 \
  --cache-from type=registry,ref=<registry>/<cache-image>[,parameters...] \
  --cache-to type=registry,ref=<registry>/<cache-image>[,parameters...] \
  --push \
  -t <registry>/<image> \
  -f dockerfile \
  .
```

- [--cache-from](https://docs.docker.com/reference/cli/docker/buildx/build/#cache-from) オプションでイメージレジストリから現在のビルドにキャッシュをインポートするように指定します。
- [--cache-to](https://docs.docker.com/reference/cli/docker/buildx/build/#cache-to) オプションでキャッシュ保存先であるイメージレジストリを指定します。

使っているサービス（ECR 等）で指定内容が変わってくるので、詳細は割愛します。

## 最後に

型チェック・Lint チェックのスキップは、本質的な問題解決というわけではないですが、Rust 製の Linter を使っていないと、Lint チェックの処理時間も無視できないレベルで時間がかかることがあります。このあたりの処理のスキップは、割とビルド時間の短縮に効果的です。
また、Docker コンテナ上でビルドしている場合は、Docker Buildx でのキャッシュ指定はビルド時間短縮に効果的なので、設定しておきたいところです。

地味な小技が多いのですが、この記事が Next.js のビルド時間が長くて困っている人の助けになったら幸いです。
