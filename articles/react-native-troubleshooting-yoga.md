---
title: "古い react-native (<0.69) の iOS アプリを Xcode 14.3 でビルドした際に発生するエラーの解消方法"
emoji: "😭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ReactNative", "yoga", "トラブルシューティング"]
published: true
publication_name: "cureapp"
---

## 発端

古い react-native の iOS アプリを Xcode 14.3 でビルドしたところ、`use of bitwise '|' with boolean operands` というエラーが出ました（詳細は以下。Bitrise 上でビルドした場合のエラーログ）

```
/Users/[REDACTED]/git/node_modules/react-native/ReactCommon/yoga/yoga/Yoga.cpp:2289:9: use of bitwise '|' with boolean operands [-Werror,-Wbitwise-instead-of-logical]
        node->getLayout().hadOverflow() |
         ^
```

stack overflow を確認したところ、以下のような報告がありました。
[ios - Use of bitwise '|' with boolean operands | XCode 14.3 fails builds using react-native Yoga - Stack Overflow](https://stackoverflow.com/questions/75897834/use-of-bitwise-with-boolean-operands-xcode-14-3-fails-builds-using-react-n)

> Starting **April 2023**, all iOS apps submitted to the App Store must be built with the **iOS 16.1 SDK or later**, included in **Xcode 14.1 or later**.

とはいえ、2023/4 月から iOS のビルドは Xcode 14.1 以上に含まれる iOS 16.1 SDK 以上でビルドしなければならなくなったので困りました（この条件を満たさないと App Store にアプリをアップロードできなくなった）

## 何が原因だったのか？

react-native 内部で使われている Yoga というパッケージのバージョンが古い（yoga <= v1.18.0）と、operator の使い方間違っているようです（以下が修正 commit）
[Use logical operator instead of bit operation · facebook/yoga@f174de7](https://github.com/facebook/yoga/commit/f174de70afdde2492e8677bd0e716eb41bf64469)

余談：ちなみに、Yoga とは react-native 内で使われているレイアウトエンジンのことです。
[Yoga Layout | A cross-platform layout engine](https://yogalayout.com/)

## 影響のあるバージョン

- react-native < 0.69
  [Use logical operator instead of bit operation · facebook/react-native@52d8a79](https://github.com/facebook/react-native/commit/52d8a797e7a6be3fa472f323ceca4814a28ef596)

## 解決策（パッチを当てる方法）

シンプルに react-native のバージョンを上げるという方法もありますが、プロダクトの状況によってはパッチを当てて、すぐにリリースしたいケースもあるでしょう。

ここでは、`patch-package` という npm パッケージを使って、パッチを作成する方法を紹介します。
[ds300/patch-package: Fix broken node modules instantly 🏃🏽‍♀️💨](https://github.com/ds300/patch-package#set-up)

### patch-package をインストール

yarn を使っている場合は、以下の通りインストールします。

```bash
yarn add patch-package postinstall-postinstall --dev
```

### パッチを作成する

該当のコードを `node_modules` から探します。以下のパスに該当のコードがあります。

```
react-native/ReactCommon/yoga/yoga/Yoga.cpp
```

上記の `Yoga.cpp` を開き、以下の通りに修正します。

```diff
- node->getLayout().hadOverflow() |
+ node->getLayout().hadOverflow() ||
```

以下のコマンドを実行します。

```bash
yarn patch-package react-native
```

すると、ディレクトリに `patches/react-native+0.64.2.patch` みたいな名前のパッチファイルが作成されます。

![](/images/react-native-troubleshooting-yoga/patch_img.png)

最後に、node_modules のファイルを変更したら、`package.json` に以下のコマンドを追加します（`yarn install` 後にパッチを当てる処理を記述します）

```json
"scripts": {
  ...
  "postinstall": "patch-package",
  ...
},
```

### 最終確認

いったん、`node_modules` を削除して、インストールし直し、`Yoga.cpp` が修正した通りになっていれば OK です。

最後に、Bitrise などで iOS アプリをビルドしてみて、ビルドが通れば成功です！
