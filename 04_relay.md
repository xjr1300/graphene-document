# リレー

- [リレー](#リレー)
  - [役に立つリンク](#役に立つリンク)
  - [ノード](#ノード)
    - [簡単な例](#簡単な例)
    - [独自のノード](#独自のノード)
    - [ノード型へのアクセス](#ノード型へのアクセス)
    - [ノードルートフィールド](#ノードルートフィールド)
  - [コネクション](#コネクション)
    - [簡単な例](#簡単な例-1)
    - [コネクションフィールド](#コネクションフィールド)
  - [ミューテーション](#ミューテーション)
    - [ファイルの受付](#ファイルの受付)

Grapheneは[Relay](https://relay.dev/docs/guides/graphql-server-specification/)を完全にサポートしており、簡単にPythonと統合するユーティリティをいくつか提供しています。

## 役に立つリンク

- [Getting started with Relay: Relayを始める](https://relay.dev/docs/getting-started/step-by-step-guide/)
- [Relay Global Identification Specification: Relayグローバル識別仕様](https://relay.dev/graphql/objectidentification.htm)
- [Relay Cursor Connection Specification: Relayカーソルコネクション仕様](https://relay.dev/graphql/connections.htm)

## ノード

`Node`は、`ID!`である1つの`id`フィールドを含む、`graphene.relay`によって提供されるインターフェイスです。
それから派生した任意のオブジェクトは、`id`で`Node`を取得するための`get_node`メソッドを実装しなければなりません。

### 簡単な例

[Starwarsのリレーの例](https://github.com/graphql-python/graphene/blob/master/examples/starwars_relay/schema.py)から持ってきた使用例です。

```python
class Ship(graphene.ObjectType):
    """スターウォーズの物語の船"""
    class Meta:
        interfaces = (relay.Node,)

    name = graphene.String(description="The name of the ship.")

    @classmethod
    def get_node(cls, info, id):
        return get_ship(id)
```

クエリしたときに`Ship`型によって返された`id`は、サーバーがその方とそのIDを知るために十分な情報を含んでいるスカラーです。

例えば、`Ship: 1`をBASE64でエンコードしたクエリを実行したとき、インスタンス`Ship(id=1)`は`U2hpcDocx`を返します。
そして、それは後でIDでノードをクエリしたい場合に便利です。

### 独自のノード

事前に定義された`relay.Node`を使用したり、それをサブクラス化して、クラスの`to_global_id`メソッドを使用してノードIDをエンコードする方法や、`get_node_from_global_id`メソッドを使用してエンコードされたIDからノードを取得する独自な方法を定義できます。

独自なノードの例を次に示します。

```python
class CustomNode(Node):
    class Meta:
        name = "Node"

    @staticmethod
    def to_global_id(_type, id):
        return f"{_type}:{id}"

    @staticmethod
    def get_node_from_global_id(info, global_id, only_type=None):
        _type, id = global_id.split(":")
        if only_type:
            # 取得したいノード型がフィールド型で示されている型と同じであると想定しています。
            assert _type == only_type._meta.name, "Received not compatible node"
        if _type == "User":
            return get_user(id)
        elif _type == "Photo":
            return get_photo(id)
        else:
            return None
```

### ノード型へのアクセス

もし、型名とIDによってインスタンスを識別するスカラーである`global_id`からノードインスタンスを取得したい場合、単に`Node.get_node_from_global_id(info, global_id)`を呼び出すことでできます。

この場合、特定の型に取得するインスタンスを制限したい場合、`Node.get_node_from_global_id(info, global_id, only_type=Ship)`を呼び出すことでできます。
これは、`global_id`が`Ship`型に対応していない場合、エラーを引き起こします。

### ノードルートフィールド

[Relay仕様](https://facebook.github.io/relay/docs/graphql-relay-specification.html)に要求されるように、サーバーは`Node`インターフェイスを返す`node`と呼ばれるルートフィールドを実装しなければなりません。

この理由で、`graphene`は`relay.Node.Field`を提供しており、それは`Node`を実装したスキーマ内の任意の方にリンクします。
使用方法を次に示します。

```python
class Query(graphene.ObjectType):
    # 独自のノードを使用したい場合は、`CustomNode.Field()`にしなくてはなりません。
    node = relay.Node.Field()
```

## コネクション

コネクションは、リストをスライスしてページネーションする方法を提供するビタミンを追加した`List`バージョンです。
`graphene`でコネクション型を作成する方法は、`relay.Connection`と`relay.ConnectionField`を使用することです。

### 簡単な例

与えられたノードの独自なコネクションを作成したい場合、`Connection`クラスをサブクラス化しなければなりません。

次の例において、`extra`はコネクションの追加的なフィールドで、`other`はコネクションの端の追加的なフィールドです。

```python
class ShipConnection(Connection):
    extra = String()

    class Meta:
        node = Ship

    class Edge:
        other = STring()
```

`ShipConnection`コネクションクラスは、自動的に`pageInfo`フィールドと、`ShipConnection.Edge`のリストである`edges`フィールドを持ちます。
この`Edge`は、`ShipConnection.Meta`で指定した特定のノードにリンクする`node`フィールドと、クラス内で定義した`other`フィールドを持ちます。

### コネクションフィールド

任意のコネクションに対してコネクションフィールドを作成でき、`Node`を実装した任意の`ObjectType`の場合、デフォルトコネクションを持ちます。

```python
class Faction(graphene.ObjectType):
    name = graphene.String()
    ships = relay.ConnectionField(ShipConnection)

    @staticmethod
    def resolve_ships(parent, info):
        return []
```

## ミューテーション

ほとんどのAPIはデータの読み込みを許可するだけでなく、書き込みも許可します。

GraphQLにおいて、これはミューテーションを使用して行われます。
ちょうどクエリのように、`Relay`はミューテーションに追加的な要件をいくつか追加しますが、Grapheneはそれらをうまく管理します。
しなければならないすべてのことは、`relay.ClientIDMutation`のサブクラスのミューテーションを作成することです。

```python
class IntroduceShip(relay.ClientIDMutation):
    class Input:
        ship_name = graphene.String(required=True)
        faction_id = graphene.String(required=True)

    ship = graphene.Field(Ship)
    faction = graphene.Field(Faction)

    @classmethod
    def mutate_and_get_Payload(cls, parent, info, **input):
        ship_name = input.ship_name
        faction_id = input.faction_id
        ship = create_ship(ship_name, faction_id)
        faction = get_faction(faction_id)
        return IntroduceShip(ship=ship, faction=faction)
```

### ファイルの受付

ミューテーションはファイルも受付でき、次は、さまざまな統合で機能するようにする方法です。
> Mutations can also accept files, that’s how it will work with different integrations:

```python
class UploadFile(graphene.ClientIDMutation):
    class Input:
        pass
        # ファイルのアップロードでは何も必要ありません。

    # 返却するフィールドです。
    # success = graphene.String()  <- graphene.Boolean()の間違い?
    success = graphene.Boolean()

    @classmethod
    def mutate_and_get_payload(cls, parent, info, **input):
        # Djangoで使用する場合、contextはrequestオブジェクトです。
        files = info.context.FILES
        # または、Flaskで使用する場合、contextはFlaskグローバルリクエストになります。
        # files = context.files

        # ファイルに対して何らかの処理をします。

        return UploadFile(success=True)
```
