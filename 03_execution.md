# 実行

## クエリの実行

スキーマに対してクエリを実行するために、スキーマの`execute`メソッドを直接呼び出せます。

```python
from graphene import Schema


schema = Schema(...)
result = schema.execute("{name}")
```

`result`は、実行結果を表現します。
`result.data`はクエリの実行結果で、もしエラーが発生していない場合、`result.errors`は`None`で、エラーが発生した場合はからのリストです。

### コンテキスト

`context`を経由してクエリにコンテキストを渡せます。

```python
from graphene import ObjectType, String, Schema


class Query(ObjectType):
    name = String()

    @staticmethod
    def resolve_name(parent, info):
        return info.context.get("name")


schema = Schema(query=Query)
result = schema.execute("{name}", context={"name": "Syrus"})
assert result.data["name"] == "Syrus"
```

### 変数

`variables`を経由してクエリに変数を渡せます。

```python
from graphene import ObjectType, Field, ID, Schema


class Query(ObjectType):
    user = Field(User, id=ID(required=True))

    @staticmethod
    def resolve_user(parent, info, id):
        return get_user_by_id(id)


schema = Schema(query=Query)
result = schema.execute("""
  query getUser($id: ID) {
    user(id: $id) {
      id
      firstName
      lastName
    }
  }
  """,
  variables={"id": 12},
)
```

### ルート値

ルートクエリまたはミューテーションで[Parent Value Object (parent)](https://docs.graphene-python.org/en/latest/types/objecttypes/#resolverparamparent)に使用される値は、`root`パラメーターを使用して上書きされます。

```python
from graphene import ObjectType, Field, Schema


class Query(ObjectType):
    me = Field(User)

    @staticmethod
    def resolve_me(parent, info):
        return {"id": root.id, "firstName": root.name}


schema = Schema(query=Query)
user_root = User(id=12, name="Bob")
result = schema.execute("""
    query getUser {
      user {
        id
        firstName
        lastName
      }
    }
    """,
    root=user_root
)
assert result.data["user"]["id"] == user_root.id
```

### 操作名

クエリ文字列に停止された複数の操作がある場合、`operation_name`はどちらが実行されるべきかを示すために使用されるべきです。

```python
from graphene import ObjectType, Field, Schema


class Query(ObjectType):
    user = Field(User)

    @staticmethod
    def resolve_user(parent, info):
        return get_user_by_id(12)


schema = Schema(query=Query)
query_string = """
    query getUserWithFirstName {
        user {
          id
          firstName
          lastName
        }
    }
    query getUserWithFullName {
      user {
        id
        fullName
      }
    }
"""
result = schema.execute(query_string,operation_name="getUserWithFullName")
assert result.data["user"]["fullName"]
```

## ミドルウェア

スキーマのフィールドの評価に影響を与えるために`middleware`を使用できます。

ミドルウェアは、`resolve(next_middleware, *args)`に反応する任意のオブジェクトまたは関数です。

メソッドの内部で、ミドルウェアは次のどちらかを実行します。

- 評価を続けるために次のミドルウェアに`resolve`を送信するか・・・
- 早期に評価を終了するために値を返します。

### 引数を解決する

ミドルウェアの`resolve`は、いくつかの引数で呼び出されます。

- `next`は実行チェインを表現します。評価を続けるために`next`を呼び出します。
- `root`はクエリを通じて渡されるルート値オブジェクトです。
- `info`はリゾルバーの情報です。
- `args`はフィールドに渡される引数の辞書です。

### 例

このミドルウェアは、もし`field_name`が`user`でない場合、単に評価を継続します。

```python
class AuthorizationMiddleware(object):
    def resolve(self, next, parent, info, **args):
        if info.field_name == "user:
            return None
        return next(parent, info, **args)
```

そして次にそれを実行します。

```python
result = schema.execute("THE QUERY", middleware=[AuthorizationMiddleware()])
```

もし、`middleware`引数が複数のミドルウェアを含む場合、それらのミドルウェアは、例えば最後から最初までなど、下から上に実行されます。

### 関数の例

またミドルウェアは、関数としても定義できます。
ここに、それぞれのフィールドを解決するためにかかった時間をログに記録するミドルウェアを定義します。

```python
from time import time as timer


def timing_middleware(next, parent, info, **args):
    start = timer()
    return_value = next(parent, inf, **args)
    duration = round((timer() - start) * 1000, 2)
    parent_type_name = parent._meta.name if root and hasattr(parent, "_meta") else ""
    logger.debug(f"{parent_type_name}.{info.field_name}: {duration} ms")
    return return_value
```

## データローダー

データローダーは、バッチやキャッシュを介してデータベースまたはWebサービスのような様々なリモートデータソースに対して単純で一貫性のあるAPIを提供するために、アプリケーションのデータ取得分として使用される汎用的なユーティリティです。
それは分離されたパッケージである[Asyncio DataLoader](https://pypi.org/project/aiodataloader/)によって提供されます。

### バッチ

バッチは高度な機能ではなく、それはデータローダの主要な機能です。
ローダーは、提供されたバッチロード関数によって作成されます。

```python
from aiodataloader import DataLoader


class UserLoader(DataLoader):
    async def batch_load_fn(self, keys):
        # ここで、キー内のそれぞれのキーに対応するユーザーを返す関数を呼び出します。
        return [get_user(id_key) for key in keys]
```

バッチロード非同期関数はキーのリストを受け付けて、`values`のリストを返します。

`DataLoader`は、実行の単一フレーム内で発生するすべての個々のロードを合成して（ラップしたイベントループが解決されると実行されます）、その後、すべての要求されたキーでバッチ関数を呼び出します。

```python
user_loader = UserLoader()

user1 = await user_loader.load(1)
user1_best_friend = await user_loader.load(user1.best_friend_id)

user2 = await user_loader.load(2)
user2_best_friend = await user_loader.load(user2.best_friend_id)
```

単純なアプリケーションは、要求された情報のために4回のラウンドトリップを発行するかもしれませんが、`DataLoader`を使用するアプリケーションは、最大2回しかしません。

- 単純なアプリケーション: `user1`, `user1_best_friend`, `user2`, `user2_best_friend`の取得
- データローダーを使用するアプリケーション: `user1`, `user2`の取得（`user1_best_friend`, `user2_best_friend`は`user1`または`user2`の取得時に取得）

ロードされた値はキーと1対1で、同じ順番でもたなければなりません。
これは、もし単一のクエリからすべての値をロードした場合、次にクエリ結果がキーと一致するように順序を付けなくてはならないことを意味します。

```python
class UserLoader(DataLoader):
    async def batch_load_fn(self, keys):
        users = {user.id: user for user in User.objects.filter(id__in=keys)}
        return [users.get(user_id) for user_id in keys]
```

`DataLoader`は、バッチデータロードの性能に犠牲を払うっ子とナシで、アプリケーションの関連のない部分と切り離します。
ローダーは個々の値をロードするAPIを提供しますが、すべての同時並行リクエストは合成され、バッチロード関数に提供されます。
これは、アプリケーションと維持された最小の外に向かうデータリクエストを介して、アプリケーションが安全にデータ取得要求を分散できるようにします。

### Grapheneで使用する

`DataLoader`は、Graphene/GraphQLとうまく組み合います。
GraphQLフィールドは、単独の関数になるように設計されています。
キャッシュまたはバッチメカニズムなしで、そのままのGraphQLサーバーがフィールドが解決されるたびに新しいデータベースリクエストを発行することを簡単にします。

次のGraphQLリクエストを考えてください。

```graphql
{
  me {
    name
    bestFriend {
      name
    }
    friends(first: 5){
      name
      bestFriend {
        name
      }
    }
  }
}
```

もし、`me`, `bestFriend`と`friends`それぞれが、バックエンドにリクエストを送信する必要があり、最大13回のデータベースリクエストになる可能性があります。

`DataLoader`を使用したとき、無駄のないコードで前の例を使用して`User`型を定置でき、最大4回のデータベースリクエストになり、キャッシュがヒットする場合、可能な限り少なくします。

```python
class User(graphene.ObjectType):
    name = graphene.String()
    best_friend = graphene.Field(lambda: User)
    friends = graphene.List(lambda: User)

    @staticmethod
    async def resolve_best_friend(parent, info):
        return await user_loader.load(parent.best_friend_id)

    @staticmethod
    async def resolve_friends(parent, info):
        return await user_loader.load_many(root.friend_ids)
```

## ファイルのアップロード

ファイルのアップロードは、まだ公式のGraphQLの仕様の一部となっておらず、Grapheneで実装されていません。

もし、サーバーがファイルのアップロードをサポートする必要がある場合、ファイルのアップロードを追加することでGrapheneを強化する[Graphene-file-upload](https://github.com/lmcgartland/graphene-file-upload)、非公式GraphQLの[マルチパートリクエスト仕様](https://github.com/jaydenseric/graphql-multipart-request-spec)に準拠するライブラリを使用できます。

## サブスクリプション

サブスクリプションを作成するために、スキーマの`subscribe`メソッドを直接呼び出してください。
このメソッドは非同期で待機しなければなりません(`must be awaited`)。

```python
import asyncio
from datetime import datetime
from graphene import ObjectType, String, Schema, Field


# すべてのスキーマはクエリを要求します。
class Query(ObjectType):
    hello = String()

    def resolve_hello(parent, info):
        return "Hello, world!"


class Subscription(ObjectType):
    time_of_day = String()

    @staticmethod
    async def subscribe_time_of_day(parent, info):
        while True:
            yield datetime.now().isoformat()
            await asyncio.sleep(1)


schema = Schema(query=Query, subscription= Subscription)


async def main(schema):
    subscription = "subscription { timeOfDay }"
    result = await schema.subscription(subscription)
    async for item in result:
        print(item.data["timeOfDay"])


asyncio.run(main(schema))
```

`result`は、クエリと同じ方法でアイテムを生み出す非同期イテレーターです。

## クエリの検証

GraphQLは、クエリのASTが妥当で実行可能か検証するために、クエリバリデーターを使用します。
すべてのGraphQLサーバーは、標準のクエリバリデーターを実装しています。
例えば、クエリしたフィールドがクエリした方に存在するか確認するバリデーターがあり、もしそれが存在しない場合、それは「Cannot query field on type」エラーでクエリを失敗させます。

一般的なユースケースを支援するために、Grapheneは標準でいくつかの検証ルールを提供しています。

### 深さ制限バリデーター

深さ制限バリデーターは、悪意のあるクエリの実行を防ぐために役立ちます。
それは次の引数を受け取ります。

- `max_depth`は　GraphQLドキュメント内の操作に許可される最大の深さです。
- `ignore`は、フィールド名に基づく再帰的な深さ確認を停止します。文字列または正規表現にマッチする名前、またはブール値を返す関数です。
- `callback`は、バリデーションが動作するたびに呼び出されます。それぞれの操作の深さのマップを持つオブジェクトを受け取ります。

#### 使用方法

スキーマに深さ制限を實相する方法をここに示します。

```python
from graphql import validate, parse
from graphene import ObjectType, Schema, String
from graphene.validation import depth_limit_validator


class MyQuery(ObjectType):
    name = String()


schema = Schema(query=MyQuery)


# 20以上の深さを持つクエリを実行させない。
validation_errors = validate(
    schema=schema.graphql_schema,
    document_ast=parse("THE QUERY"),
    rules=(
        depth_limit_validator(max_depth=20)
    )
)
```

### イントロスペクションの無効化

イントロスペクション無効化検証ルールは、スキーマがイントロスペクションされないように保証します。

> イントロスペクション(introspection): 内観、反省、自らを振り返る（自己反省）、ITでは自分自身の構造を調べて、さらに変更すること

これは、プロダクション環境でセキュリティの測定に役立ちます。

#### 使用方法

スキーマのイントロスペクションを無効化する方法をここに示します。

```python
from graphql import validate, parse
from graphene import ObjectType, Schema, String
from graphene.validation import DisableIntrospection


class MyQuery(ObjectType):
    name = String()


schema = Schema(query=MyQuery)


# イントロスペクションクエリを実行しない
validation_errors = validate(
    schema=schema.graphql_schema,
    document_ast=parse("THE QUERY"),
    rules=(
        DisableIntrospection()
    )
)
```

### 独自のバリデーターの実装

すべての独自なクエリバリデーターは、`graphql.validation.rules`モジュールからインポートできる[ValidationRule](https://github.com/graphql-python/graphql-core/blob/v3.0.5/src/graphql/validation/rules/__init__.py#L37)基本クラスを拡張しなければなりません。
クエリバリデーターはビジタークラスです。
それらは、クエリを検証するとき、`ASTValidationContext`の1つの必須な引数を使用してインスタンス化されます。
検証を実行するために、バリデータークラスは1つ以上の`enter_*`と`leave_*`メソッドを実装しなければなりません。
可能な`enter/leave`アイテムと関数の詳細のドキュメントは、訪問者モジュールのコンテンツを参照してください。
検証を失敗させるために、失敗した理由を説明した`GraphQLError`インスタンスで`report_error`メソッドを呼び出さなくてはなりません。
GraphQLクエリのフィールド定義を訪問するクエリバリデーターと、それらのフィールドがブラックリストにある場合にクエリの検証を失敗させる例をここに示します。

```python
from graphql import GraphQLError
from graphql.language import FieldNode
from graphql.validation import ValidationRule


my_blacklist = ("disallowed_field",)


def is_blacklisted_field(field_name: str):
    return field_name.lower() in my_blacklist


class BlacklistRule(ValidationRule):
    def enter_field(self, node: FieldNode, *_args):
        field_name = node.name.value
        if not is_blacklisted_field(field_name):
            return
        self.report_error(
            GraphQLError(
                f"Cannot query '{field_name}': field is blacklisted", node
            )
        )
```

#### ビジタークラス（Visitorパターン）

ビジターパターンは、オブジェクトの構造とアルゴリズムを分離するためのデザインパターンであｒ．
このパターンでは、操作をオブジェクトの構造から分離し、操作を別の訪問者クラス（`Visitor`）に定義します。
これにより、オブジェクトの構造を変更せずに新しい操作を追加することができます。

次は、Pythonでビジターパターンを実装した例である。
この例では、図形（`Circle`と`Rectangle`）のリストに対して、面積を計算する訪問者（`AreaVisitor`）を使用している。

```python
from abc import ABC, abstractmethod
import math

# Shapeインターフェース（abc.ABCを継承した抽象基本クラス）
class Shape(ABC):
    @abstractmethod
    def accept(self, visitor):
        pass


# ConcreteShapeクラス
class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius

    def accept(self, visitor):
        """訪問者を受け入れる"""
        visitor.visit_circle(self)


# ConcreteShapeクラス
class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def accept(self, visitor):
        """訪問者を受け入れる"""
        visitor.visit_rectangle(self)


# Visitorインターフェース
class Visitor(ABC):
    @abstractmethod
    def visit_circle(self, circle):
        pass

    @abstractmethod
    def visit_rectangle(self, rectangle):
        pass


# ConcreteVisitorクラス
class AreaVisitor(Visitor):
    def visit_circle(self, circle):
        area = math.pi * (circle.radius ** 2)
        print(f'Circle area: {area}')

    def visit_rectangle(self, rectangle):
        area = rectangle.width * rectangle.height
        print(f'Rectangle area: {area}')


shapes = [
    Circle(5),
    Rectangle(3, 4)
]
visitor = AreaVisitor()
for shape in shapes:
    shape.accept(visitor)
```
