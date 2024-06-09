# 型のリファレンス

- [型のリファレンス](#型のリファレンス)
  - [スキーマ](#スキーマ)
    - [問い合わせ](#問い合わせ)
    - [型](#型)
    - [フィールド名の自動キャメルケース化](#フィールド名の自動キャメルケース化)
  - [スカラー](#スカラー)
    - [ビルドインされているスカラー型](#ビルドインされているスカラー型)
    - [独自のスカラー型](#独自のスカラー型)
    - [スカラーをマウントする](#スカラーをマウントする)
  - [リストと非Null](#リストと非null)
    - [NonNull](#nonnull)
    - [List](#list)
    - [非Nullリスト](#非nullリスト)
  - [ObjectType](#objecttype)
    - [簡単な例](#簡単な例)
  - [リゾルバー](#リゾルバー)
    - [リゾルバーパラメーター](#リゾルバーパラメーター)
      - [Parent Value Object (parent)](#parent-value-object-parent)
        - [リゾルバーの例](#リゾルバーの例)
        - [名前の慣習](#名前の慣習)
      - [GraphQL Execution Info (info)](#graphql-execution-info-info)
      - [GraphQL Arguments (\*\*Kwargs)](#graphql-arguments-kwargs)
    - [Grapheneのリゾルバーの便利な機能](#grapheneのリゾルバーの便利な機能)
      - [暗黙的な静的メソッド](#暗黙的な静的メソッド)
      - [デフォルトリゾルバー](#デフォルトリゾルバー)
      - [高度なリゾルバー](#高度なリゾルバー)
        - [GraphQLのデフォルトの引数](#graphqlのデフォルトの引数)
        - [クラスの外にあるリゾルバー](#クラスの外にあるリゾルバー)
        - [値オブジェクトとしてインスタンス化する](#値オブジェクトとしてインスタンス化する)
        - [フィールドのキャメルケース化](#フィールドのキャメルケース化)
  - [ObjectTypeの設定 - Metaクラス](#objecttypeの設定---metaクラス)
    - [GraphQLタイプ名](#graphqlタイプ名)
    - [GraphQLの説明](#graphqlの説明)
    - [インターフェイスと可能な型](#インターフェイスと可能な型)
  - [列挙型](#列挙型)
    - [定義](#定義)
    - [値の説明](#値の説明)
    - [PythonのEnumと使用する](#pythonのenumと使用する)
    - [注意事項](#注意事項)
  - [インターフェイス](#インターフェイス)
    - [データオブジェクトを型へ解決する](#データオブジェクトを型へ解決する)
  - [ユニオン](#ユニオン)
    - [簡単な例](#簡単な例-1)

## スキーマ

GraphQLの`Schema`は、APIの型と`Field`間の関連を定義します。

`Schema`は、操作、必須のクエリ、ミューテーションとサブスクリプションそれぞれのルート[ObjectType](https://docs.graphene-python.org/en/latest/types/objecttypes/#objecttype)を提供することにより作成されます。

`Schema`は、ルートの操作と関連する型の定義を収集して、バリデーターとエグゼキューターにそれらを提供します。

```python
my_schema = Schema(
    query=MyRootQuery,
    mutation=MyRootMutation,
    subscription=MyRootSubscription
)
```

ルート`Query`は、APIのエントリポイントとなるフィールドを定義する特別な[ObjectType](https://docs.graphene-python.org/en/latest/types/objecttypes/#objecttype)です。

ルート`Mutation`とルート`Subscription`は、ルート`Query`と同様ですが、異なる操作の型です。

- `Query`はデータを取得します。
- `Mutation`はデータを変更して変更を取得します。
- `Subscription`はリアルタイムで変更をクライアントに送信します。

フィールド、スキーマそして操作の簡潔な概要を確認するために[GraphQL ドキュメントの Schema](https://graphql.org/learn/schema/)を復習してください。

### 問い合わせ

スキーマを問い合わせるために、それの`execute`メソッドを呼び出してください。
詳細は[クエリの実行](https://docs.graphene-python.org/en/latest/execution/execute/#schemaexecute)を参照してください。

```python
query_string = "query whoIsMyBestFriend { myBestFriend { lastName } }"
my_schema.execute(query_string)
```

### 型

スキーマが、予定しているすべての型にアクセスできない場合がいくつかあります。
例えば、フィールドが`Interface`を返したとき、スキーマはその実装について理解できません。

この場合、スキーマを作成するときに`types`引数を使用する必要があります。

```python
my_schema = Schema(
    query=MyRootQuery,
    types=[SomeExtraObjectType,]
)
```

### フィールド名の自動キャメルケース化

デフォルトで、明示的に`name`引数で設定されていないすべてのフィールドと引数の名前は、通常、APIはJavaScriptやモバイルクライアントによって使用されるため、`snake_case`から`camelCase`に変換されます。

例えば、`ObjectType`を使用すると、`last_name`フィールド名は`lastName`に変換されます。

```python
class Person(graphene.ObjectType):
    last_name = graphene.String()
    other_name = graphene.String(name="_other_Name")
```

この変換を適用したくない場合、フィールドのコンストラクタに`name`引数を提供してください。
`other_name`は、さらなる変換なしで`_other_Name`に変換します。

クエリは次のようになります。

```graphql
{
  lastName
  _other_Name
}
```

この振る舞いを無効にするために、スキーマのインスタンスを作成するときに、`auto_camelcase`を`False`に設定してください。

## スカラー

スカラー型は、クエリのリーフにある具体的な値を表現します。
Pythonで一般的な型を表現するGrapheneが標準機能として提供するいくつかのビルトイン型があります。
また、データモデル内の値をより良く表現するために独自のスカラー型を作成できます。

すべてのスカラー型は、次の引数を受け付けます。
それらすべてはオプションです。

- `name`: string

  `Field`の名前を上書きします。

- `description`: string

  GraphQLブラウザー内で表示する型の説明です。

- `required`: boolean

  もし`True`の場合、サーバーはこのフィールドの値を強制します。
  [NonNull](https://docs.graphene-python.org/en/latest/types/list-and-nonnull.html#nonnull)を参照してください。
  デフォルトは`False`です。

- `deprecation_reason`: string

  `Field`の非推奨理由を提供します。

- `default_value`: any

  `Field`のデフォルト値を提供します。

### ビルドインされているスカラー型

Grapheneは、デフォルトの[GraphQLの型](https://graphql.org/learn/schema/#scalar-types)と適合する、次の基礎的なスカラー型を定義しています。

- `graphene.String`

  テキストエリアを表現して、UTF-8文字列として表現されます。
  `String`型は、非構造形式で、人間が読みやすいテキストを表現するために、GraphQLで最もよく使用されます。

- `graphene.Int`

  少数のない符号付き数値を表現します。
  `Int`は、[GraphQL仕様](https://facebook.github.io/graphql/June2018/#sec-Int)に基づく、符号付き32ビット整数です。

- `graphene.Float`

  [IEEE754](http://en.wikipedia.org/wiki/IEEE_floating_point)によって規定された符号付き倍精度浮動小数点数を表現します。

- `graphene.Boolean`

  *true*または*false*を表現します。

- `graphene.ID`

  一意な識別子を表現し、オブジェクトを再取得するとき、またはキャッシュのキーとしてよく使用されます。
  `ID`型は`String`としてJSON表現に現れます。
  しかし、人間が読みやすくすることを意図されていません。
  入力する型として予期されているとき、任意の"4"のような文字列、または4のような整数の入力値は`ID`として受け入れられます。

---

また、Grapheneは、一般的な値のために独自のスカラー型を提供しています。

- `graphene.Date`

  [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)によって規定された日付の値を表現します。

  ```python
  import datetime
  from graphene import Schema, ObjectType, Date


  class Query(ObjectType):
      one_week_from = Date(required=True, date_input=Date(required=True))

      def resolve_one_week_from(root, info, date_input):
          assert date_input == datetime.date(2006, 1, 2)
          return date_input + datetime.timedelta(weeks=1)

  schema = Schema(query=Query)

  results = schema.execute("""
    query {
        oneWeekFrom(dateInput: "2006-01-02")
    }
  """)

  assert results.date == {"oneWeekFrom": "2006-01-09"}
  ```

- `graphene.DateTime`

  [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)によって規定された日時の値を表現します。

  ```python
  import datetime
  from graphene import Schema, ObjectType, DateTime

  class Query(ObjectType):
      one_hour_from = DateTime(required=True, datetime_input=DateTime(required=True))

      def resolve_one_hour_from(root, info, datetime_input):
          assert datetime_input == datetime.datetime(2006, 1, 2, 15, 4, 5)
          return datetime_input + datetime.timedelta(hours=1)

  schema = Schema(query=Query)

  results = schema.execute("""
      query {
          oneHourFrom(datetimeInput: "2006-01-02T15:04:05")
      }
  """)
  ```

- `graphene.Time`

  [ISO8601](https://en.wikipedia.org/wiki/ISO_8601)によって規定された時刻の値を表現します。

  ```python
  from datetime import date, datetime, time, timedelta
  from graphene import Schema, ObjectType, Time


  class Query(ObjectType):
      one_hour_from = Time(required=True, time_input=Time(required=True))

      def resolve_one_hour_from(root, info, time_input):
          assert time_input == time(15, 4, 5)
          tmp_time_input = datetime.combine(date(1, 1, 1), time_input)
          return (tmp_time_input + timedelta(hours=1)).time()

  schema = Schema(query=Query)

  results = schema.execute("""
      query {
          oneHourFrom(time_input: "15:04:05")
      }
  """)

  assert results.data == {"oneHourFrom": "16:04:05"}
  ```

- `graphene.Decimal`

  Pythonの`Decimal`値を表現します。

  ```python
  import decimal
  from graphene import Schema, ObjectType, Decimal


  class Query(ObjectType):
      add_one_to = Decimal(required=True, decimal_input=Decimal(required=True))

      def resolve_add_one_to(root, info, decimal_input):
          assert decimal_input == decimal.Decimal("10.50")
          return decimal_input + decimal.Decimal("1")


  schema = Schema(query=Query)

  results = schema.execute("""
      query: {
          addOneTo(decimalInput: "10.50")
      }
  """)

  assert results.data == {"addOneTo": "11.50"}
  ```

- `graphene.JSONString`

  JSON文字列を表現します。

  ```python
  from graphene import Schema, ObjectType, JSONString, String


  class Query(ObjectType):
      update_json_key = JSONString(
          required=True,
          json_input=JSONString(required=True),
          key=String(required=True),
          value=String(required=True)
      )

      def resolve_update_json_key(root, info, json_input, key, value):
          assert json_input == {"name": "Jane"}
          json_input[key] = value
          return json_input


  schema = Schema(query=Query)

  results = schema.execute("""
      query {
          updateJsonKey(jsonInput: "{\\"name\\": \\"Jane\\"}", key: "name", value: "Beth")
      }
  """)

  assert results.data == {"updateJsonKey": "{\"name\": \"Beth\"}"}
  ```

- `graphene.Base64`

  Base64エンコードされた文字列を表現します。

  ```python
  from graphene import Schema, ObjectType, Base64


  class Query(ObjectType):
      increment_encoded_id = Base64(
          required=True,
          base64_input=Base64(required=True),
      )

      def resolve_increment_encoded_id(root, info, base64_input):
          assert base64_input == "4"
          return int(base64_input) + 1


  schema = Schema(query=Query)

  results = schema.execute("""
      query {
          incrementEncodedId(base64Input: "NA==")
      }
  """)

  assert results.data == {"incrementEncodedId": "NQ=="}
  ```

### 独自のスカラー型

スキーマ用に独自のスカラー型を作成できます。
次は、`DataTime`スカラー型を作成する例です。

```python
from datetime import datetime

from graphene.types import Scalar
from graphql.language import ast


class DateTime(Scalar):
    """DateTimeスカラー型の説明"""

    @staticmethod
    def serialize(dt):
        return dt.isoformat()

    @staticmethod
    def parse_literal(node, _variables=None):
        if isinstance(node, ast.StringValueNode):
            return datetime.strftime(node.value, "%Y-%m-%dT%H:%M:%S.%f")

    @staticmethod
    def parse_value(value):
        return datetime.strptime(value, "%Y-%m-%dT%H:%M:%S.%f")
```

### スカラーをマウントする

> マウント: 利用可能な状態にする

`ObjectType`、`interface`または`Mutation`でマウントされたスカラー型は、`File`として機能します。

```python
class Person(graphene.ObjectType):
    name = graphene.String()

# 上記と等価です。
class Person(graphene.ObjectType):
    name = graphene.Field(graphene.String)
```

注意事項: 直接フィールドのコンストラクタを使用するとき、型を渡して、インスタンスを渡さないようにしてください。

`Field`でマウントされた型は、`Argument`として機能します。

```python
graphene.Field(graphene.String, to_graphene.String())

# 上記と等価です。
graphene.Field(graphene.String, to=graphene.Argument(graphene.String()))
```

## リストと非Null

`ObjectType`、`Scaler`と`Enum`は、単にGrapheneで定義された型の種類です。
しかし、スキーマの他の部分またはクエリ変数の定義でその型を使用したとき、それらの値の検証に影響を与える追加的な型の修飾子を適用できます。

### NonNull

```python
import graphene


class Character(graphene.ObjectType):
    name = graphene.NonNull(graphene.String)
```

ここで、`String`型を使用して、`NonNull`クラスを使用してそれをラップすることによって非Nullにしています。
これは、常にサーバーがこのフィールドを非Null値で返すことを予期することを意味して、もし、最終的にNull値を種t区した場合、GraphQLの実行エラーが発せられ、何か問題が発生したことをクライアントに知らせます。

前の`NonNull`コードの断片は、次とも等価です。

```python
import graphene


class Character(graphene.ObjectType):
    name = graphene.String(required=True)
```

### List

```python
import graphene


class Character(graphene.ObjectType):
    appears_in = graphene.List(graphene.String)
```

同様に`List`も機能します。
フィールドがその型のリストを返すことを示す`List`として型をマークするために型修飾子を利用できます。
それは引数の場合も同様に機能して、検証ステップではその値のリストが予期されます

### 非Nullリスト

デフォルトでは、リスト内のアイテムはNull可能として考えられます。
Null許可アイテム以外のリストを定義するために、そお型は`NonNull`としてマークされる必要があります。

```python
import graphene


class Character(graphene.ObjectType):
    appears_in = graphene.List(graphene.NonNull(graphene.String))
```

上記の結果は型定義に現れます。

```graphql
type Character {
  appearsIn: [String!]
}
```

## ObjectType

Grapheneの`ObjectType`は、`Schema`内の`Field`間の関連と、それらのデータが取得される方法を定義するために使用される構成要素です。

その基本は次です。

- それぞれの`ObjectType`は、`graphene.ObjectType`から派生したPythonのクラスです。
- `ObjectType`のそれぞれの属性は、`Field`を表現します。
- それぞれの`Field`は、データを取得する[リゾルバーメソッド](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolvers)または[デフォルトリゾルバ](https://docs.graphene-python.org/en/latest/types/objecttypes/#defaultresolver)を持ちます。

### 簡単な例

この例のモデルは、名前と苗字を持つ`Person`を定義します。

```python
from graphene import ObjectType, String


class Person(ObjectType):
    first_name = String()
    last_name = String()
    full_Name = String()

    def resolve_full_name(parent, info):
        return f"{parent.first_name} {parent.last_name}"
```

この`ObjectType`は、`first_name`、`last_name`と`full_name`フィールドを定義します。
それぞれのフィールドはクラス属性として記述され、それぞれの属性は`Field`にマップされます。
データは、`full_name`フィールドのための`resolve_full_name`[リゾルバーメソッド](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolvers)で、その他のフィールドは[デフォルトリゾルバ](https://docs.graphene-python.org/en/latest/types/objecttypes/#defaultresolver)により取得されます。

上記`Person ObjectType`は、次のスキーマ表現を持ちます。

```graphql
type Person {
  firstName: String
  lastName: String
  fullName: String
}
```

## リゾルバー

`Resolver`は、`Schema`内の`Field`のデータの取得により`Query`に回答することを支援するメソッドです。

もし、クエリ内にフィールドが含まれていない場合、そのリゾルバーは実行できないため、リゾルバーは遅延実行されます。

Grapheneの`ObjectType`にあるそれぞれのフィールドは、データを取得するリゾルバーメソッドと対応するべきです。
このリゾルバーメソッドはフィールド名とマッチするべきです。
例えば、上記の`Person`型の場合、`full_name`フィールド`resolve_full_name`メソッドによって解決されます。

それぞれのリゾルバーメソッドはパラメータを持ちます。

- 値オブジェクトの[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)は、ほとんどのフィールドを解決するために使用します。
- クエリ、スキーマのメタ情報、そしてリクエストごとのコンテキストの[GraphQL Execution Info (info)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparaminfo)
- `Field`に定義された[GraphQL Arguments (**kwargs)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamgraphqlarguments)

### リゾルバーパラメーター

#### Parent Value Object (parent)

このパラメーターは、典型的に`ObjectType`のほとんどのフィールドの値を導出するために使用されます。

リゾルバーメソッド(*parent*)の最初のパラメーターは親フィールドのリゾルバーから返された値オブジェクトです。
もし、ルート`Query`フィールドのように親フィールドがない場合、*parent*の値はクエリを実行している間に構成されたデフォルトが`None`の`root_value`に設定されます。
クエリを実行する詳細は[クエリの実行](https://docs.graphene-python.org/en/latest/execution/execute/#schemaexecute)を参照してください。

##### リゾルバーの例

ルートクエリに`Person`型と1つのフィールドを持つスキーマを持っています。

```python
from graphene import ObjectType, String, Field


class Person(ObjectType):
    full_name = String()

    def resolve_full_name(parent, info):
        return f"{parent.first_name} {parent.last_name}"


class Query(ObjectType):
    me = Field(Person)

    def resolve_me(parent, info):
        # Personを表現するオブジェクトが返されます。
        return get_human(name="Luke Skywalker")
```

このスキーマに対してクエリを実行します。

```python
schema = Schema(query=Query)

query_string = "{ me { fullName } }"
result = schema.execute(query_string)

assert result.data["me"] == {"fullName": "Luke Skywalker"}
```

そして、このクエリを解決するために次のステップを順番に実施します。

- `parent`は、クエリの実行から`root_value`の`None`で設定されます。
- `parent`を`None`に設定して、値オブジェクト`Person("Luke Skywalker")`を返す`Query.resolve_me`が呼び出されます。
- そして、その値オブジェクトは、スカラー文字列値"Luke Skywalker"を解決するために、`Person.resolve_full_name`を呼び出している間、`parent`として利用されます。
- そのスカラー値は、シリアライズされて、クエリレスポンスで戻されます。

それぞれのリゾルバーは、チェイン内の後続するリゾルバーの実行で使用されるために、次の[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)を返します。
もし、`Field`が`Scalar`型の場合、その値はシリアライズされ`Response`として送信されます。
そうでなければ、`ObjectType`のような複合型を解決する間、その値は次の[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)として渡されます。

##### 名前の慣習

この[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)は、他のGraphQLドキュメント内で`obj`、`parent`または`source`と名前を付けられています。
また、値オブジェクトが解決されたあとに名前を付けることもできます。
例えば、ルート`Query`または`Mutation`の`root`や、`Person`値オブジェクトの`person`です。
時々、この引数はGrapheneのコードで`self`と名前を付けられますが、Grapheneでクエリを実行する際の[暗黙的な静的メソッド](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverimplicitstaticmethod)なため、これは誤解を招く可能性があります。

> `@staticmethod`デコレーターでリゾルバーメソッドを修飾して、リゾルバーメソッドが静的メソッドであることを明示すること。
> `@staticmethod`デコレーターでリゾルバーメソッドを修飾しない場合、リンターから警告が発生する。
>
> ```python
> import graphene
>
>
> class Person(graphene.ObjectType):
>     full_name = graphene.String()
>
>     @staticmethod
>     def resolve_full_name(parent, info):
>         return f"{parent.first_name} {parent.last_name}"
> ```

#### GraphQL Execution Info (info)

2番目のパラメーターは2つのものを提供します。

- フィールド、スキーマ、解析されたクエリなど、現在のGraphQLクエリの実行に関するメタ情報の参照
- ユーザーの認証、データローダーインスタンス、またはクエリを解決するために便利なほかのなにかを保存するために使用できる、リクエストごとの`context`へのアクセス

ほとんどのアプリケーションにおいて、`context`飲みが要求されます。
`context`の設定については[Context](https://docs.graphene-python.org/en/latest/execution/execute/#schemaexecutecontext)を参照してください。

#### GraphQL Arguments (**Kwargs)

フィールドの任意の引数は、キーワード引数としてリゾルバー関数に渡されることで定義します。
例を次に示します。

```python
from graphene import ObjectType, Field, String


class Query(ObjectType):
    human_by_name = Field(Human, name=String(required=True))

    def resolve_human_by_name(parent, info, name):
        return get_human(name=name)
```

そして、次のクエリを実行できます。

```graphql
query {
  humanByName(name: "Luke Skywalker") {
    firstName
    lastName
  }
}
```

注意事項: Grapheneによって「予約されている」フィールドの引数がいくつかあります。
[Fields (Mounted Types)](https://docs.graphene-python.org/en/latest/api/#fields-mounted-types)を参照してください。
`arg`パラメーターを使用することで、これらのフィールドの1つと衝突する次のような引数も定義できます。

```python
from graphene import ObjectType, Field, String


class Query(ObjectType):
    answer = String(args={"description": String()})

    def resolve_answer(parent, info, description):
        return description
```

> `description`引数は、Grapheneのフィールドの引数として「予約されて」いる。

### Grapheneのリゾルバーの便利な機能

#### 暗黙的な静的メソッド

Grapheneの1つの驚くべき機能は、すべてのリゾルバーメソッドが暗黙的な静的メソッドとして扱われることです。
これは、Pythonの他のメソッドと異なり、Grapheneによって実行されている間、リソルバーの最初の引数が決して`self`にならないことを意味します。
代わりに、最初の引数は常に[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)です。
実際に、GraphQLにおいて、Pythonオブジェクト自身の属性よりも、ほとんどの場合、親の値オブジェクトを使用してクエリを解決することに関心があるため、これはとても便利です。

次の例の2つのリゾルバーは同じように効果的です。

```python
from graphene import ObjectType, String

class Person(ObjectType):
    first_name = String()
    last_name = String()

    @staticmethod
    def resolve_first_name(parent, info):
        """
        `staticmethod`でPythonのメソッドをデコレートすることは、`self`が引数として
        提供されないことを保証します。しかし、Grapheneはこの振る舞いのために、このデコレーター
        を必要としません。
        """
        return parent.first_name

    def resolve_last_name(parent, info):
        """
        通常、このメソッドの最初の引数は`self`になりますが、Grapheneは暗黙的な静的メソッドとして
        これを実行します。
        """
        return parent.last_name
```

もし、コードをより明示的にしたい場合、自由に`@staticmethod`デコレーターを使用してください。
あるいは、それらを使用しないほうがコードがきれいになるかもしれません。

> リンターによってメソッドの最初の引数は`self`であることを強制されるた。
> よって、`@staticmethod`デコレーターでリゾルバーメソッドを修飾することが推奨される。

#### デフォルトリゾルバー

リゾルバーメソッドが`ObjectType`の`Field`属性に定義されていない場合、Grapheneはデフォルトリゾルバーを提供します。

[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)が辞書の場合、リゾルバーはフィールド名にマッチする辞書のキーを探します。
辞書でない場合、リゾルバーは親値オブジェクトからフィールド名とマッチする属性を取得します。

```python
from collections import namedtuple

from graphene import ObjectType, String, Field, Schema

PersonValueObject = namedtuple("Person", ["first_name", "last_name"])


class Person(ObjectType):
    first_name = String()
    last_name = String()


class Query(ObjectType):
    me = Field(Person)
    my_best_friend = Field(Person)

    @staticmethod
    def resolve_me(parent, info):
        # `me`フィールドのためにオブジェクトが常に渡されます。
        return PersonValueObject(first_name="Luke", last_name="Skywalker")

    @staticmethod
    def resolve_my_best_friend(parent, info):
        # `my_best_friend`フィールドのために辞書が常に渡される。
        return {"first_Name": "R2", "last_name": "D2"}


schema = Schema(query=Query)
result = schema.execute("""
    query {
        me {
            firstName
            lastName
        }
        myBestFriend {
            firstName
            lastName
        }
    }
""")

# デフォルトリゾルバーで、オブジェクトから属性を解決できます。
# リゾルバーが返したデータがオブジェクトの場合、Grapheneはクエリのフィールド名と、一致する名前を持つ
# オブジェクトの属性の値を取得する。
assert result.data["me"] == {"firstName": "Luke", "lastName": "Skywalker"}

# デフォルトリゾルバーで、辞書からキーを解決することもできます。
# リゾルバーが返したデータが辞書の場合、Grapheneはクエリのフィールド名と、辞書からフィールド名と一致する
# キーの値を取得する。
assert result.data["myBestFriend"] == {"firstName": "R2", "lastName": "D2"}
```

#### 高度なリゾルバー

##### GraphQLのデフォルトの引数

必須でない引数を定義して、クエリの実行で引数として提供されない場合、全くそれはリゾルバー関数に渡されません。
これは、開発者に引数の`undefined`値と、明示的な`null`値を区別できるようにするためです。

例えば、次のスキーマが与えられたとします。

```python
from graphene import ObjectType, String


class Query(ObjectType):
    hello = String(required=True, name=String())

    def resolve_hello(parent, info, name):
        return name if name else "World"
```

そして、このクエリを実行します。

```graphql
query {
  hello
}
```

するとエラーがスローされます。

```test
TypeError: resolve_hello() missing 1 required positional argument: 'name'
```

様々な方法でこのエラーを修正できます。
すべてのキーワード引数を辞書に結合することによって。

```python
from graphene import QueryType, String


class Query(ObjectType):
    hello = String(required=True, name=String())

    def resolve_hello(parent, info, **kwargs):
        name = kwargs.get("name", "World")
        return f"Hello, {name}!"
```

または、キーワード引数にデフォルト値を設定することによって。

```python
from graphene import QueryType, String


class Query(ObjectType):
    hello = String(required=True, name=String())

    def resolve_hello(parent, info, name="World"):
        return f"Hello, {name}!"
```

もう一つは、Grapheneを使用して、GraphQLスキーマ自身に、引数のデフォルト値を設定することもできます。

```python
from graphene import QueryType, String


class Query(ObjectType):
    hello = String(required=True, name=String(default_value="World"))

    def resolve_hello(parent, info, name):
        return f"Hello, {name}!"
```

> スキーマに設定することでクエリの仕様が明確になるため、GraphQLスキーマ自身に、引数のデフォルト値を設定することを推奨する。

##### クラスの外にあるリゾルバー

フィールドは、クラスの外にある独自のリゾルバーを使用できます。

```python
from graphene import ObjectType, String


def resolve_full_name(person, info):
    return f"{person.first_name} {person.last_name}"


class Person(ObjectType):
    first_name = String()
    last_name = String()
    full_name = String(resolver=resolve_full_name)
```

>　クラスの外にリゾルバーを定義することは、複数クラス間でリゾルバーを共有するときに利用できる。

##### 値オブジェクトとしてインスタンス化する

Grapheneの`ObjectType`は、値オブジェクトとしても動作できます。
前の例で、それぞれの`ObjectType`のフィールドのデータをキャプチャーするために、`Person`を使用できました。

```python
peter = Person(first_Name="Peter", last_name="Griffin")

peter.first_name  # "Peter"を出力
peter.last_name # "Griffin"を出力
```

##### フィールドのキャメルケース化

Grapheneは、GraphQL標準に準拠するために、自動的に`ObjectType`のフィールドを`field_name`から`fieldName`のようにキャメルケースにします。
詳細は[フィールド名の自動キャメルケース化](https://docs.graphene-python.org/en/latest/types/schema/#schemaautocamelcase)を参照してください。

## ObjectTypeの設定 - Metaクラス

Grapheneは、異なるオプションを設定するために、`ObjectType`の`Meta`内部クラスを使用します。

### GraphQLタイプ名

デフォルトでGraphQLスキーマの型名は、`ObjectType`を定義したクラス名と同じです。
これは、`Meta`クラスの`name`属性を設定することで変更できます。

```python
from graphene import ObjectType


class MyGraphQlSong(ObjectType):
    class Meta:
        name = "Song"
```

### GraphQLの説明

`ObjectType`のスキーマの説明は、Pythonオブジェクトまたは`Meta`内部クラスのdocstringで設定できます。

```python
from graphene import ObjectType


class MyGraphQlSong(ObjectType):
    """docstringでObjectTypeのスキーマの説明を設定できます"""

    class Meta:
        description = (
          "しかし、Meta内で説明を設定した場合、代わりにこの値が設定されます。"
        )
```

### インターフェイスと可能な型

`Meta`内部クラスに`interface`を設定することは、この`ObjectType`が実装するGraphQLインターフェイスが指定されます。

`possible_types`の提供は、インターフェイスやユニオンのような不明瞭な型をGrapheneが解決するために役に立ちます。

詳細は[インターフェイス](https://docs.graphene-python.org/en/latest/types/interfaces/#interfaces)を参照してください。

```python
from graphene import ObjectType, Node


Song = namedtuple("Song", ["title", "artist"])


class MyGraphQlSong(ObjectType):
    class Meta:
        interfaces = (Node,)
        possible_types = (Song,)
```

## 列挙型

`Enum`は、一意な定数値に束縛する象徴的な名前（メンバー）の集合を表現する、特別なGraphQLの型です。

### 定義

クラスを使用することで`Enum`を作成できます。

```python
import graphene


class Episode(graphene.Enum):
    NEWHOPE = 4
    EMPIRE = 5
    JEDI = 6
```

`Enum`のインスタンスを使用して作成することもできます。

```python
import graphene

Episode = graphene.Enum("Episode", [("NEWHOPE", 4), ("EMPIRE", 5), ("JEDI", 6)])
```

### 値の説明

`Enum`の値に説明を追加でき、`Enum`の値は`description`プロパティを持つ必要があります。

```python
class Episode(graphene.Enum):
    NEWHOPE = 4
    EMPIRE = 5
    JEDI = 6

    @property
    def description(self):
        if self == Episode.NEWHOPE:
            return "New Hope Episode"
        return "Other Episode"
```

### PythonのEnumと使用する

`Enum`がすでに定義されている場合、`Enum.from_enum`関数を使用してそれらを再利用できます。

```python
graphene.Enum.from_enum(AlreadyExistingPyEnum)
```

`Enum.from_enum`は、オリジナルを変更することなしで、説明などを`Enum`に追加するために、入力として`description`と`description_reason`ラムダをサポートしています。

```python
graphene.Enum.from_enum(
    AlreadyExistingPyEnum,
    description=lambda v: return "foo" if v == AlreadyExistingPyEnum.Foo else "bar"
)
```

### 注意事項

`graphene.Enum`は、内部で`enum.Enum`を使用しており（またそれが利用できない場合はバックポート）、メンバーのゲッターを除いて同様な方法で使用できます。

> バックポートとは、新しいバージョンのソフトウェアの機能を、古いバージョンのソフトウェアに移植すること。

Pythonの`Enum`の実装において、`Enum`を初期化することでメンバーにアクセスできます。

```python
from enum import Enum


class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3


assert Color(1) == Color.RAD
```

しかし、Grapheneの`Enum`において、ある効果を得るために`.get`を呼び出す必要があります。

```python
from graphene import Enum


class Color(Enum):
    RED = 1
    GREEN = 2
    BLUE = 3


assert Color.get(1) == Color.RED
```

## インターフェイス

`Interface`は、インターフェイスを実装するために型が含まなければならない特定のフィールドの集合を定義する抽象的な型です。

例えば、スターウォーズ3部作の任意のキャラクターを表現する`Character`インターフェイスを定義できます。

```python
import graphene


class Character(graphene.Interface):
    id = graphene.ID(required=True)
    name = graphene.String(required=True)
    friends = graphene.List(lambda: Character)
```

`Character`を実装する任意の`ObjectType`は、これらの引数と戻り値の型を持つ、これらの正確なフィールドがあります。

例えば、ここに`Character`を実装するいくつかの型を示します。

```python
class Human(graphene.ObjectType):
    class Meta:
        interfaces = (Character,)

    starships = graphene.List(Starship)
    home_planet = graphene.String()


class Droid(graphene.ObjectType):
    class Meta:
        interfaces = (Character,)

    primary_function = graphene.String()
```

これらの型の両方は、`Character`インターフェイスのフィールドのすべてを持ちますが、その特定の型の`Character`に固有の追加フィールドである`home_planet`、`starships`と`primary_function`を持ちます。

その完全なGraphQLスキーマの定義は次のようになります。

```graphql
interface Character {
  id: ID!
  name: String!
  friends: [Character]
}

type Human implements Character {
  id: ID!
  name: String!
  friends: [Character]
  starships: [Starships]
  homePlanet: String
}

type Droid implements Character {
  id: ID!
  name: String!
  friends: [Character]
  primaryFunction: String
}
```

いくつかの異なる型のオブジェクトまたはオブジェクトの集合を返したいとき、`Interface`は便利です。

例えば、そのエピソードに応じて任意の`Character`に解決する`hero`フィールドを定義できます。

```python
class Query(graphene.ObjectType):
    hero = graphene.Field(Character, required=True, episode=graphene.Int(required=True))

    @staticmethod
    def resolve_hero(parent, info, episode):
        # ルークはエピソード5のヒーローです。
        if episode == 5:
            return get_human(name="Luke Skywalker")
        return get_droid(name="R2-D2")


schema = graphene.Schema(query=Query)
```

これは、`Character`インターフェイスに存在するフィールドを直接クエリしたり、[インラインフラグメント](https://graphql.org/learn/queries/#inline-fragments)を使用してインターフェイスを實相する任意の型の特定のフィールドを選択することができます。

例えば、次のクエリは・・・

```graphql
query HeroForEpisode($episode: Int!) {
  hero(episode: $episode) {
    __typename
    name
    ... on Droid {
      primaryFunction
    }
    ... on Human {
      homePlanet
    }
  }
}
```

`{"episode": 4}`で次のデータを返します。

```json
{
  "data": {
    "hero": {
      "__typename": "Droid",
      "name": "R2-D2",
      "primaryFunction": "Astromech"
    }
  }
}
```

そして、`{"episode": 5}`で異なるデータを返します。

```json
{
  "data": {
    "hero": {
      "__typename": "Human",
      "name": "Luke Skywalker",
      "homePlanet": "Tatooine"
    }
  }
}
```

### データオブジェクトを型へ解決する

Grapheneでスキーマを構築すると、リゾルバーは、Grapheneの型のインスタンスではなく、DjangoまたはSQLAlchemyモデルのような、GraphQLの型を裏付けるデータを表すオブジェクトを返すことが一般的です。
これは、`ObjectType`と`Scalar`フィールドでも機能しますが、インターフェイスを使用し始めたとき、次のエラーに出会うかもしれません。

```text
"Abstract Type Character must resolve to an Object type at runtime for field Query.hero ..."
```

Grapheneは、`Interface`を解決するために必要なデータオブジェクトをGrapheneの型に変換する十分な情報を持っていないことが理由で発生します。
これを解決するために、データオブジェクトとGrapheneの型をマップする`resolve_type`クラスメソッドを`Interface`に定義できます。

```python
class Character(graphene.Interface):
    id = graphene.ID(required=True)
    name = graphene.String(required=True)

    @classmethod
    def resolve_type(cls, instance, info):
        if instance.type == "DROID":
            return Droid
        return Human
```

## ユニオン

ユニオン型はインターフェイスととても似ていますが、それらは型の間で任意の共通なフィールドを指定できません。

基本は次のとおりです。

- それそれのユニオンは、`graphene.Union`から派生したPythonのクラスです。
- ユニオンは任意のフィールドを持たず、単に可能性のある`ObjectType`へのリンクです。

### 簡単な例

このモデルの例は、`独自のフィールドを持ついくつかの`ObjectType`を定義しています。
`SearchResult`は、この`ObjectType`の`Union`の実装です。

```python
import graphene


class Human(graphene.ObjectType):
    name = graphene.String()
    born_in = graphene.String()


class Droid(graphene.ObjectType):
    name = graphene.String()
    primary_function = graphene.String()


class Starship(graphene.ObjectType):
    name = graphene.String()
    length = graphene.Int()


class SearchResult(graphene.Union):
    class Meta:
        types = (Human, Droid, Starship)
```

スキーマの`SearchResult`型を返す場合、`Human`、`Droid`または`Starship`を取得する可能性があります。
ユニオン型のメンバーは、具体的なオブジェクトの型でなければならないことに注意してください。
インターフェイスまたは他のユニオン型から、ユニオン型を作成することはできません。

上記の型は、スキーマ内で次の表現を持ちます。

```graphql
type Droid {
  name: String
  primaryFunction: String
}

type Human {
  name: String
  bornIn: String
}

type Starship {
  name: String
  length: Int
}

union SearchResult = Human | Droid | Starship
```
