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
