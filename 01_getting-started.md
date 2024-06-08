# はじめる

- [はじめる](#はじめる)
  - [イントロダクション](#イントロダクション)
    - [GraphQLとは](#graphqlとは)
    - [Grapheneとは](#grapheneとは)
  - [Grapheneの例](#grapheneの例)
    - [要求事項](#要求事項)
    - [プロジェクトの準備](#プロジェクトの準備)
    - [基本的なスキーマの作成](#基本的なスキーマの作成)
    - [スキーマ定義言語(SDL)](#スキーマ定義言語sdl)
    - [問い合わせ](#問い合わせ)
    - [次のステップ](#次のステップ)

## イントロダクション

### GraphQLとは

GraphQLはAPIのためのクエリ言語です。

それは標準的な方法で次を提供します。

- サーバーによって提供されたデータを静的に型付けられた`Schema`で記述します。
- データ要件を正確に説明する`Query`でデータをリクエストして、
- 要求したデータのみを含む`Response`でデータを受け取ります。

GraphQLの紹介とその概念の概要については、[オフィシャルのGraphQLドキュメント](http://graphql.org/learn/)を参照してください。

### Grapheneとは

Grapheneは、*コードファースト*な手法でPythonでGraphQL APIを実装するためのツールを提供するライブラリです。

GraphQL APIを構築するために Grapheneの*コードファースト*な手法と、[Apollo Server](https://www.apollographql.com/docs/apollo-server/)(JavaScript)または[Aradne](https://ariadnegraphql.org/)(Python)のような*スキーマファースト*な手法を比較してください。
GraphQLの`スキーマ定義言語(SDL)`を記述する代わりに、あなたのサーバーによって提供されるデータを説明するためのPythonコードを記述します。

Grapheneは、ほどんどの人気のあるWebフレームワークとORMと統合する完全な機能があります。
Grapheneは、GraphQL仕様に完全に準拠したスキーマを提供して、同様に Relay に準拠した APIを構築するツールとパターンを提供します。

## Grapheneの例

Grapheneで"hello"と"goodbye"と言う基本的なGraphQLスキーマを構築しましょう。

`hello`というたった 1 つの`Field`と、`firstName`という`Argument`の値を指定したリクエストする`Query`を送信したとき、

```graphql
{
  hello(firstName: "friend")
}
```

要求したデータのみを含む次のレスポンスを予期できます(`goodbye`フィールドは解決されていません)。

```json
{
  "data": {
    "hello": "Hello friend!"
  }
}
```

### 要求事項

- Python(3.6, 3.7, 3.8, 3.9, 3.10, pypy)
- Graphene(3.0)

### プロジェクトの準備

```sh
pip install "graphene>=3.0"
```

### 基本的なスキーマの作成

Grapheneにおいて、次のコードを使用して単純なスキーマを定義できます。

```python
from graphene import ObjectType, String, Schema

class Query(ObjectType):
    # これは単一の引数`first_name`を持つ`hello`というフィールドを持つスキーマを定義します。
    # デフォルトでは、引数名は自動的にキャメルケースにない、生成されたスキーマでは`firstName`になります。
    hello = String(first_name=String(default_value="stranger"))
    goodbye = String()

    # リゾルバのメソッドは、GraphQLコナンテキスト(root、info)と同様に
    # フィールドの引数（first_name）を取り、クエリのレスポンスのデータを返します。
    def resolve_hello(root, info, first_name):
        return f'Hello {first_name}!'

    def resolve_goodbye(root, info):
        return 'See ya!'

schema = Schema(query=Query)
```

GraphQLの`Schema`は、*String、Int*そして*Enum*のようなスカラー型と、*List*と*Object*のような複合型を使用して、サーバーによって提供されるデータモデルのそれぞれの`Field`を説明します。
詳細は、Grapheneの[型リファレンス](https://docs.graphene-python.org/en/latest/types/#typesreference)を参照してください。

また、スキーマは`Field`のために任意の数の`Argument`を定義できます。
これは、`Query`でそれぞれの`Field`の正確なデータの要求を記述する強力な方法です。

`Schema`のそれぞれのフィールドのために、現在のコンテキストと`Argument`を使用して、クライアントの`Query`によってリクエストされたデータを取得する`Resolver`メソッドを記述します。
詳細は、このセクションの[Resolver](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolvers)を参照してください。

### スキーマ定義言語(SDL)

[GraphQLスキーマ定義言語](https://graphql.org/learn/schema/)において、下に示す通りコード例によって定義されたフィールドを説明できます。

```graphql
type Query {
  hello(firstName: String = "stranger"): String
  goodbye: String
}
```

このドキュメントの追加的な例は、オブジェクトタイプと他のフィールドをによって作成されたスキーマを説明するためにSDLを使用します。

### 問い合わせ

次に、`execute`にGraphQLのクエリ文字列を渡すことによって、`Schema`の問い合わせを開始できます。

```python
# デフォルトの引数でフィールドを問い合わせできます。
query_string = "{ hello }"
result = schema.execute(query_string)
print(result.data["hello"])
# => "Hello stranger!"

# または、クエリに引数を渡します。
query_with_argument = "{ hello(firstName: "GraphQL") }"
result = schema.execute(query_with_argument)
print(result.data["hello"])
# => "Hello GraphQL!"
```

### 次のステップ

おめでとうございます。
機能する最初のGrapheneのスキーマを得ました。

GrapheneはFlaskやDjangoなどの人気のあるWebフレームワークと統合する多くの便利な機能を提供しているため、通常、スキーマに対してクエリ文字列を直接実行する必要はありません。
GraphQL APIを提供することを開始する方法についてより情報が必要な場合は[統合](https://docs.graphene-python.org/en/latest/#integrations)を確認してください。
