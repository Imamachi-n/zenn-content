---
title: "普段使っている Visual Studio Code の便利な拡張機能を紹介するよ！"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode"]
published: true
published_at: 2024-10-10 09:00
publication_name: "aldagram_tech"
---

こんにちは！[アルダグラム](https://aldagram.com/about/)でエンジニアをしている今町です。

皆さん、コーディングをする際、どのようなエディタもしくは統合開発環境（IDE）を使っていますか？Vim、JetBrains が出している各種 IDE、それとも Visual Studio Code（VSCode）でしょうか？そんな私は、ずっと **VSCode** を愛用し続けています！

VSCode を素晴らしいエディタたらしめているのは、なんといっても豊富な「拡張機能」の存在でしょう。VSCode の開発元である Microsoft 純正の拡張機能から、サードパーティ製の拡張機能まで、様々な便利ツールが拡張機能として提供されています。

とはいえ、星の数ほど拡張機能が存在しており（それは言い過ぎか）、最初はどれを使ったら良いのかわからなかったりします（なんなら、私が他人の使っている拡張機能を知りたいくらいです 👀）

普段は、以下のような技術スタックで開発を行っているので、これらの開発に使っている拡張機能をメインで紹介したいと思います！（私の独断と偏見で）

- フロントエンド: TypeScript / Next.js
- バックエンド: Ruby on Rails

## フロントエンド: TypeScript / Next.js 向けのオススメ拡張機能

有名な拡張機能ばかりかもしれませんが、普段使っている拡張機能を紹介します。

### [ESLint](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)

もはや必須レベル。
VSCode 上で、ESLint のルールに違反しているコード箇所にエラーなら赤色、警告なら黄色の波線がつきます。
`Editor: Format On Save` の設定を ON にしておけば、ESLint のルールで auto fix できるものに関しては、ファイル保存時に自動的にフォーマットが効きます。
大概のプロジェクトでは CI/CD で lint チェックしているため、ファイル保存時に linter によるフォーマットを効かせることで、CI/CD での lint チェックでコケるのを未然に防げます。

![image.png](/images/vscode-extention-recommend/1.png =600x)

### [Prettier - Code formatter](https://marketplace.visualstudio.com/items?itemName=esbenp.prettier-vscode)

もはや必須レベル。
ESLint は基本的には JavaScript / TypeScript の Linter であるのに対して、Prettier は GraphQL や JSON などの他のファイルに対してもフォーマット可能です。
こちらも、`Editor: Format On Save` の設定を ON にしておけば、ファイル保存時に自動的にフォーマットが効きます。

### [npm Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.npm-intellisense)

import statement を記述したときに、npm モジュールの autocomplete をしてくれます。

![image.png](/images/vscode-extention-recommend/2.png)

### [Path Intellisense](https://marketplace.visualstudio.com/items?itemName=christian-kohler.path-intellisense)

ファイルパスの autocomplete をしてくれます。

![image.png](/images/vscode-extention-recommend/3.png =600x)

### [Pretty TypeScript Errors](https://marketplace.visualstudio.com/items?itemName=yoavbls.pretty-ts-errors)

VSCode で TypeScript の型エラーの文言が表示されるのですが、単なる文字列になっているので、非常に読みにくいです（個人的な感想）

この拡張機能は、そんな読みにくい TypeScript の型エラーの文言を読みやすいカタチにフォーマットして表示してくれます。エラー文言がめちゃくちゃ長くて読みにくいとき、結構助かります。
フォーマットされたエラー文言がコピペできない点が、玉にキズ…。

![image.png](/images/vscode-extention-recommend/4.png =600x)

### [Import Cost](https://marketplace.visualstudio.com/items?itemName=wix.vscode-import-cost)

import のコスト計算をしてくれます。コストが高い import に関しては、コストが赤文字で表示されます。内部的には、webpack を利用してインポートされたサイズを計算しています。また、Webpack の Tree Shaking を利用して、不要なコードを含まない状態で計算しています。
この拡張機能の詳細を知りたい人は、[Keep Your Bundle Size Under Control](https://citw.medium.com/keep-your-bundle-size-under-control-with-import-cost-vscode-extension-5d476b3c5a76) の記事も参照ください。

![image.png](/images/vscode-extention-recommend/10.png =600x)

### [Color Highlight](https://marketplace.visualstudio.com/items?itemName=naumovs.color-highlight)

RGB のカラーコードが CSS や TypeScript のコード内に記述されていると、該当のカラーコードに対応する色で表示されます。
地味に便利です。

![image.png](/images/vscode-extention-recommend/5.png =300x)

### [JavaScript (ES6) code snippets](https://marketplace.visualstudio.com/items?itemName=xabikos.JavaScriptSnippets)

便利スニペット集。
`clg` → `console.log(object);` だったり、`imd` → `import { } from 'module';` だったり、JavaScript で使うスニペットがまとまった拡張機能です。

![image.png](/images/vscode-extention-recommend/6.png =500x)

![image.png](/images/vscode-extention-recommend/7.png =500x)

### [Jest Snippets](https://marketplace.visualstudio.com/items?itemName=andys8.jest-snippets)

Jest に特化したスニペット集。
個人的には、こちらのほうが恩恵を得ています。
`describe` や `test` の記述がめっちゃ楽になります。

![image.png](/images/vscode-extention-recommend/8.png =500x)

![image.png](/images/vscode-extention-recommend/9.png =500x)

## バックエンド: Ruby on Rails 向けのオススメ拡張機能

えっ？私は RubyMine 派？そんなこと言わないでください。
Ruby のサポートが比較的弱めな VSCode で頑張って Rails のコードを書くために使っている拡張機能を以下にまとめます。

### [Ruby LSP](https://marketplace.visualstudio.com/items?itemName=Shopify.ruby-lsp)

Shopify が開発した Ruby 向けの Language Server Protocol 実装です（LSP とは、コード補完や変数参照、スタイル修正といった機能実装をあらゆるエディタ・IDE で提供するためのプロトコルです）
シンタックスハイライトをつけてくれたり、Rubocop のエラーや警告を表示してくれたり、ファイル保存時に Rubocop の auto fix を効かせたり、変数や関数、クラスの検索などが可能になります。

### [Ruby Solargraph](https://marketplace.visualstudio.com/items?itemName=castwide.solargraph)

Solargraph も Ruby LSP と同様に Language Server の一種です。
クラスやモジュールの定義ジャンプの精度としては、Solargraph のほうが優れているという意見もあるため、Ruby LSP と併用して私は使っています。

### [Rails](https://marketplace.visualstudio.com/items?itemName=bung87.rails)

Ruby on Rails 向けの拡張機能。クラスやモジュールの定義ジャンプや、パスのサジェストを出してくれたりします。

### [Rails DB Schema](https://marketplace.visualstudio.com/items?itemName=aki77.rails-db-schema)

Rails の DB スキーマに基づいて、autocomplete をしてくれます。

### [vscode-gemfile](https://marketplace.visualstudio.com/items?itemName=bung87.vscode-gemfile)

Gem ライブラリの docs にジャンプできます。

![image.png](/images/vscode-extention-recommend/11.png =400x)

## 開発体験向上: その他オススメの拡張機能

### [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot)

言わずとしれた GitHub Copliot。脳死で入れておきましょう。

### [GitHub Pull Requests](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)

GitHub のプルリクエストにまつわるサポートをしてくれます。脳死で入れておきましょう。

### [Live Share](https://marketplace.visualstudio.com/items?itemName=MS-vsliveshare.vsliveshare)

ペアプロ・モブプロをするうえで便利なツール。
同じ VSCode のエディタをオンラインで共有できます。1 つの Google Docs をみんなで編集するようなイメージです。

### [**Code Spell Checker**](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker)

文字通り、スペルミスをチェックしてくれます。

### [Open in GitHub, Bitbucket, Gitlab, VisualStudio.com !](https://marketplace.visualstudio.com/items?itemName=ziyasal.vscode-open-in-github)

VSCode 上で見ているファイルを GitHub 上で開いてくれます。
該当コードを GitHub の URL で共有したいときにめちゃ役に立ちます。

![image.png](/images/vscode-extention-recommend/12.png =600x)

### [Material Icon Theme](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme)

Material Design の良い感じのアイコン。見た目が良くなるだけで、テンションが上がるかも。

![image.png](/images/vscode-extention-recommend/13.png =200x)

### [Material Theme — Free](https://marketplace.visualstudio.com/items?itemName=Equinusocio.vsc-material-theme)

VSCode 向けの Material Theme。
私は `Material Theme High Contrast` を使っています。自分の気に入ったテーマがあればどうぞ。

![image.png](/images/vscode-extention-recommend/14.png =600x)

### [Rainbow CSV](https://marketplace.visualstudio.com/items?itemName=mechatroner.rainbow-csv)

CSV の各カラムが色分けされます。
CSV ファイルを見る機会がある場合はとても見やすくて便利です。

![image.png](/images/vscode-extention-recommend/15.png =300x)

### [TODO Highlight v2](https://marketplace.visualstudio.com/items?itemName=jgclark.vscode-todo-highlight)

TODO や FIXME などが色分けして表示されます。
実装途中で TODO を書きがちなので、個人的に地味に役立つ拡張機能です。

![image.png](/images/vscode-extention-recommend/16.png =300x)

## 最後に

いかがだったでしょうか？

どの拡張機能を使うかは、個人的な好みによることも多いと思います。例えば、[Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens) という有名な拡張機能がありますが、コードの横にエラー原因が表示されるのが個人的に気に入らなくて、使わなくなった拡張機能もありました。

自分の手にあった拡張機能を見つけて、うまく使いこなすことで、開発生産性を向上できる可能性も秘めています。
「自分はこんな拡張機能を使っているよー」など、他にも便利な拡張機能があったらコメントを下さい！

もっとアルダグラムエンジニア組織を知りたい人、ぜひ下記の情報をチェックしてみてください！
