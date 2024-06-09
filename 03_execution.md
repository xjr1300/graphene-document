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
