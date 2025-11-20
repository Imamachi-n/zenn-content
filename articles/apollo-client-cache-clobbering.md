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

これは Apollo Client が **「キャッシュデータの整合性が保てないため、古いデータを破棄して新しいデータで上書きした」** ことを警告しています。
この「上書き」が引き金となり、アプリケーションは DDoS 攻撃さながらの大量リクエストをサーバーに送り続けていました。

## なぜ無限ループするのか？ メカニズムの解説

この現象を理解するには、Apollo Client の **「正規化（Normalization）」** と **「監視（WatchQuery）」** の仕組みを知る必要があります。
この「データの消失」が、Apollo の監視機能（WatchQuery）と組み合わさることで、無限ループを引き起こします。

### 1. ID がないと「正規化」されない

通常、Apollo Client はデータに含まれる `id` (または `_id`) をキーにして、データをフラットな構造（正規化された状態）でキャッシュします。

https://www.apollographql.com/blog/demystifying-cache-normalization#assigning-a-unique-identifier-to-each-object

しかし、**`id` を取得し忘れると、Apollo はそのデータを「独立したエンティティ」として認識できません。**
結果として、データは正規化されず、親である `ROOT_QUERY` の直下に「クエリ名と引数」をキーとしてベタ書きされます。

### 2. 上書き合戦の発生

上述の通り、同じ `document(uuid: $uuid)` というパスを持つが、取得するフィールドが異なる 2 つのクエリ（A と B）があるとします。

1. **クエリ A (`hogeX`を取得) を実行:**
   キャッシュには以下のように保存されます。

```json
{
  "ROOT_QUERY": {
    "__typename": "Query",
    // キーは「クエリ名 + 引数」で生成される
    "document({\"uuid\":\"123-abc\"})": {
      "__typename": "Document",
      "hogeX": "x"
    }
  }
}
```

2. **クエリ B (`hogeY`を取得) を実行:**
   パス（引数）が同じであるため、Apollo は**同じ場所**に結果を保存しようとします。しかし、正規化されていないため「フィールドのマージ」ができず、オブジェクトごと置き換えてしまいます。

```json
{
  "ROOT_QUERY": {
    "__typename": "Query",
    // キーが全く同じであるため、値が新しいオブジェクトで上書きされる
    "document({\"uuid\":\"123-abc\"})": {
      "__typename": "Document",
      "hogeY": "y"
      // 😱 ここにあったはずの "hogeX": "x" が消えている！
    }
  }
}
```

### 3. `useQuery` の検知によるループ

Apollo の `useQuery` は、自分が監視しているデータが変更されると、コンポーネントを再レンダリングしようとします。

1. クエリ A が完了し、hogeX がキャッシュされる。
2. クエリ B が完了し、キャッシュを hogeY で上書きする（hogeX が消える）。
3. Apollo Client が「クエリ A で監視していた `hogeX` がキャッシュから消えた」と検知し、自動的にクエリ A を再実行（Refetch）する。
4. **クエリ A** が再びキャッシュを `hogeX` で上書きする（今度は `hogeY` が消える）。
5. Apollo Client が「クエリ B で監視していた `hogeY` がキャッシュから消えた」と検知し、自動的にクエリ B を再実行する。
6. （以下、無限ループ 🔄）

これが、「謎の無限ループ」の正体です。

## 解決策：5 つのアプローチ

この問題に対処するための、5 つのアプローチを紹介します。

### 1. id を必ず取得する（推奨・ベストプラクティス）

最も推奨される解決策は、クエリに `id` を含めることです（当たり前かもしれませんが…）
`id` があれば、Apollo はそのデータを `ROOT_QUERY` 直下ではなく、`Document:123` のような独立した参照（Ref）として管理します。

```graphql
query GetDocumentX($uuid: ID!) {
  document(uuid: $uuid) {
    id # これを追加！
    hogeX
  }
}
```

`id` がある場合、クエリ A とクエリ B は、それぞれのフィールド (`hogeX`, `hogeY`) を、同じ `Document:123` オブジェクトに対して**マージ**します。競合が発生しないため、ループも止まります。

Apollo Client を使用する場合、基本的にすべてのクエリで id を取得するように癖をつけておくのがベストプラクティスです。ESLint などの Linter を用いて、id が欠落しないようにチェックするとよいでしょう。

### 2. 複数のクエリを 1 つにまとめる

最もシンプルな解決策は、分割されている GraphQL クエリを統合することです。

```graphql
query GetDocumentX($uuid: ID!) {
  document(uuid: $uuid) {
    id
    hogeX
    hogeY # 一緒に取ってくる
  }
}
```

意図的にクエリを分けている場合でなければ、まずはこれを検討しましょう。

### 3. typePolicies で `merge: true` を設定する

クエリの構造上どうしても `id` が取得できない、あるいは API の仕様で `id` がない場合は、キャッシュポリシー設定で「上書きではなくマージ」を強制します。

```tsx
// merge: true を設定するパターン
const client = new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      // Query型（ルート）のフィールドに対する設定
      Query: {
        fields: {
          document: {
            merge: true,
          },
        },
      },
    },
  }),
});

// マージ関数を記述するパターン
const client = new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      // Query型（ルート）のフィールドに対する設定
      Query: {
        fields: {
          document: {
            // incoming: 新しいデータ, existing: 既存のデータ
            merge(existing, incoming, { mergeObjects }) {
              // 既存データと新データをマージして保存する
              return mergeObjects(existing, incoming);
            },
            // 省略記法として単に merge: true と書いても同様の動作をする場合が多いですが
            // 複雑なオブジェクトの場合は関数を書く方が確実です。
          },
        },
      },
    },
  }),
});
```

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

Relay-Style Pagination はリスト操作によく使われますが、単一オブジェクトの取得でも「引数が違うのに同じキャッシュとして扱いたい」あるいはその逆のケースで `keyArgs` が役立ちます。
デフォルトでは全ての引数がキャッシュキーになりますが、これをカスタマイズすることで意図しない上書きを防ぐことができます。

```tsx
new ApolloClient({
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          // keyArgs に　StatusUuid を指定し、別々のキャッシュとして扱う
          projectBoard: relayStylePagination(["StatusUuid"]),
        },
      },
    },
  }),
});
```

### まとめ

Apollo Client のキャッシュ競合は、**「正規化」** の仕組みを理解していれば怖くありません。

- 開発者コンソールで `Cache data may be lost...` の警告が出ていないか確認する
- ネットワークタブで同じクエリの無限リクエストが起きていないか監視する

もしこれらの兆候が見られたら、今回紹介した「ID の取得漏れ」や「マージ設定」を確認してみてください。

また、開発中は `Network` タブだけでなく、**Apollo Client DevTools** も活用して、「Cache」タブでデータがどのように保存されているか（正規化されているか、Root Query 直下にないか）を確認する習慣をつけると良いでしょう。

https://chromewebstore.google.com/detail/apollo-client-devtools/jdkknkkbebbapilgoeccciglkfbmbnfm?hl=ja

もっとアルダグラムエンジニア組織を知りたい人、ぜひ下記の情報をチェックしてみてください！
