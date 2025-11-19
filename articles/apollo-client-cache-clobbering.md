---
title: "Apollo Clientのキャッシュ競合で無限ループ！？その原因と5つの対処法"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "apolloclient", "react"]
published: false
publication_name: "aldagram_tech"
---

Apollo Client を使用した開発中に、「取得したはずのデータが消える」「なぜか GraphQL クエリが無限にリクエストされ続ける」といった不可解な現象に遭遇したことはありませんか？

それは、Apollo Client における **「キャッシュ競合（Cache Clobbering）」** が原因かもしれません。

本記事では、実際に遭遇したキャッシュ競合によるバグ（無限ループ）の仕組みと、その具体的な 5 つの対処法について解説します。

## 発生したトラブル：謎の無限ループ

ある日、特定の画面を開くと、**2 種類の GraphQL クエリが交互に、短時間で無限に実行され続ける** という現象が発生しました。

開発者ツールの Network タブはリクエストで埋め尽くされ、コンソールには以下のような警告が出ていました。

```text
Cache data may be lost when replacing the document field of a Query object.
```

これは、Apollo Client 内で「キャッシュ競合」が発生している典型的なサインでした。

この現象が発生すると、クライアント側のパフォーマンス低下だけでなく、サーバー側に大量のリクエストを送ることになり、DDoS 攻撃に近い状態になってしまう恐れがあります。早急な対処が必要です。

## キャッシュ競合（Cache Clobbering）とは？

キャッシュ競合とは、あるクエリの結果が、別のクエリによって意図せず「上書き（置換）」されてしまう現象 のことです。

具体例で見てみましょう。（ここではデフォルト設定である `merge: false` の挙動を前提とします）

例えば、あるドキュメントに対して以下の 2 種類のクエリがあるとします（GraphQL API のクエリとしては同一。取得する値が異なる。）

1. GetDocumentX
2. GetDocumentY

```graphql
# クエリA
query GetDocumentX($uuid: ID!) {
  document(uuid: $uuid) {
    hogeX
  }
}

# クエリB
query GetDocumentY($uuid: ID!) {
  document(uuid: $uuid) {
    hogeY
  }
}
```

何が起きているのか？

1. GetDocumentDownloadUrl を実行 キャッシュには以下のように保存されます。

```json
{
  "document": {
    "__typename": "Document",
    "hogeX": "x"
  }
}
```

2. 次に GetDocumentFileUrl を実行 すると、Apollo は既存のキャッシュデータを完全に置き換えてしまいます。

```json
{
  "document": {
    "__typename": "Document",
    "hogeY": "y"
    // ⚠️ hogeX フィールドが消えてしまった！
  }
}
```

本来であれば両方のフィールドが保持されてほしいところですが、設定やクエリの書き方によっては、このように後勝ちでデータが消失してしまいます。

## なぜ無限ループするのか？

この「データの消失」が、Apollo の監視機能（WatchQuery）と組み合わさることで、無限ループを引き起こします。

無限ループ発生のメカニズム

1. クエリ A を実行 → `hogeX` がキャッシュされる。
2. クエリ B を実行 → `hogeX` が消え、`hogeY` だけになる。
3. Apollo Client が「クエリ A で監視していた `hogeX` がキャッシュから消えた」と検知し、自動的にクエリ A を再実行（Refetch）する。
4. クエリ A の結果でキャッシュが置き換わる → 今度は `hogeY` が消える。
5. Apollo Client が「クエリ B で監視していた `hogeY` がキャッシュから消えた」と検知し、自動的にクエリ B を再実行する。

3〜5 が繰り返され、無限ループへ…… 😭

## 解決策：5 つのアプローチ

この問題に対処するための、5 つのアプローチを紹介します。

### 1. id を必ず取得する（推奨）

Apollo Client は id を基準にデータを正規化してキャッシュします。 クエリに id を含めることで、Apollo は「これはデータの置き換えではなく、同じ ID を持つデータへのフィールド追加だ」と認識できるようになります（上述のトラブルは、これが原因でした）

```graphql
query GetDocumentX($uuid: ID!) {
  document(uuid: $uuid) {
    id # これを追加！
    fileUrl
  }
}
```

Apollo Client を使用する場合、基本的にすべてのクエリで id を取得するように癖をつけておくのがベストプラクティスです。ESLint などの Linter を用いて、id が欠落しないようにチェックするとよいでしょう。

### 2. 複数のクエリを 1 つにまとめる

最もシンプルな解決策は、分割されている GraphQL クエリを統合することです。

```graphql
query GetDocumentX($uuid: ID!) {
  document(uuid: $uuid) {
    fileUrl
    fileDownloadUrl # 一緒に取ってくる
  }
}
```

意図的にクエリを分けている場合でなければ、まずはこれを検討しましょう。

### 3. typePolicies で merge: true を設定する

クエリの構造上どうしても競合してしまう場合は、InMemoryCache の設定で明示的にマージを許可します。

```graphql
new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          // documentフィールドのキャッシュは上書きではなくマージする
          document: {
            merge: true
          }
        }
      }
    }
  })
})
```

これを設定すると、fileDownloadUrl を保持したまま fileUrl を追加してくれるようになります。

### 4. エイリアスを使用する

同じフィールドを異なる条件で取得し、別々に管理したい場合はエイリアスが有効です。

```graphql
query GetUsersByA($uuid: ID!) {
  usersForA: users(uuid: $uuid) {
    # エイリアス usersForA
    nodes {
      id
      uuid
      hoge
    }
  }
}

query GetUsersByB($uuid: ID!) {
  usersForB: users(uuid: $uuid) {
    # エイリアス usersForB
    nodes {
      id
      uuid
      huga
    }
  }
}
```

### 5. キャッシュキー引数を指定する

Relay-Style Pagination を使っている場合などで、特定の引数（key）ごとに別々のデータとしてキャッシュさせたい場合は、typePolicies でキー引数を指定します。

```graphql
new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          // StatusUuidという引数が違う場合は別のキャッシュとして扱う
          projectBoard: relayStylePagination(['StatusUuid'])
        }
      }
    }
  })
})
```

### まとめ

Apollo Client のキャッシュ競合は、一見すると不可解な挙動に見えますが、仕組みを理解すれば対処は難しくありません。

- 開発者コンソールで `Cache data may be lost...` の警告が出ていないか確認する
- ネットワークタブで同じクエリの無限リクエストが起きていないか監視する

もしこれらの兆候が見られたら、今回紹介した「ID の取得漏れ」や「マージ設定」を確認してみてください。
