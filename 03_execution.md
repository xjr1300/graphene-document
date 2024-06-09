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
