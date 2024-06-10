# Grapheneにおけるテスト

- [Grapheneにおけるテスト](#grapheneにおけるテスト)
  - [テストツール](#テストツール)
    - [テストクライアント](#テストクライアント)
    - [概要と簡単な例](#概要と簡単な例)
    - [パラメーターの実行](#パラメーターの実行)
    - [スナップショットテスト](#スナップショットテスト)

自動化されたテストは、現代の開発者にとって非常に役に立つバグを倒すツールです。
いくつかの問題を解決する、または避けるために、テストの集合であるテストスイートを使用できます。

- 新しいコードを記述したとき、予期したようにコードが機能するか検証するためにテストを使用できます。
- 古いコードをリファクタリングまたは修正したとき、予期せず変更がアプリケーションの振る舞いに影響を与えていないか保証するためにテストを使用できます。

GraphQLアプリケーションは、論理スキーマ定義、スキーマ検証、権限そしてフィールド解決など、いくつかのレイヤで構成されているため、GraphQLアプリケーションをテストすることは複雑な作業になります。

Grapheneのテスト実行フレームワークとユーティリティの盛り合わせを使用して、GraphQLのリクエスト、ミューテーションの実行をシミューレートして、アプリケーションの出力の検査、そして、コードが実行するべきことを実行しているかを確認できます。

## テストツール

Grapheneは、テストを記述するとき便利にする小さなツールセットを提供します。

### テストクライアント

テストクライアントは、ダミーなGraphQLクライアントそして動作するPythonクラスで、プログラム的にビューとGrapheneによって強化されたアプリケーションとの相互作用をテストできるようにします。

テストクライアントを使用してできることがいくつかあります。

- クエリとミューテーションをシミュレートしてレスポンスを観測
- ある値を含むテンプレートコンテキストを、Djangoテンプレートに与えることでレンダリングされたクエリのリクエストのテスト(Test that a given query request is rendered by a given Django template, with a template context that contains certain values.)

### 概要と簡単な例

テストクライアントを使用するために、`graphene.test.Client`をインスタンス化して、GraphQLレスポンスを取得します。

```python
from graphene.test import Client


def test_hey():
    client = Client(my_schema)
    executed = client.execute("{ hey }")
    assert executed == {
        "data": {
          "hey": "hello"
        }
    }
```

### パラメーターの実行

また、`execute`メソッドに、`context`、`root`、`variables`などのような追加のキーワード引数を追加できます。

```python
from graphene.test import Client


def test_hey():
    client = Client(my_schema)
    executed = client.execute("{ hey }", context={"user": "Peter"})
    assert executed == {
        "data": {
          "hey": "hello Peter!"
        }
    }
```

### スナップショットテスト

APIが進化するように、変更がGraphQLアプリのクライアントのいくつかを破壊するかもしれない破壊的な変更を導入したか知る必要があります。

しかし、テストを記述してGraphQLアプリケーションからの予期するいくつかのレスポンスを複製することは、退屈で繰り返しの作業になる可能性があり、場合によっては、このプロセスをスキップするほうが簡単になります。

よって、[スナップショットテスト](https://github.com/syrusakbary/snapshottest/)の使用を推奨します。

スナップショットテストは、最初にテストが実行されたときの`スナップショット`を自動的に作成するため、そよ風の中で(in a breeze)これらすべてのテストを記述できるようにします。

ここに、`pytest`を使用した場合にテストする方法を示した簡単な例を示します。

```python
def test_hey(snapshot):
    client = Client(my_schema)
    # これは、最初にテストが実行されたときに、その実行のレスポンスの
    # スナップショットディレクトリとスナップショットファイルを作成します。
    snapshot.assert_match(client.execute("{ hey }"))
```

もし、`unittest`を使用する場合は次のとおりです。

```python
import snapshottest


class APITestCase(snapshottest.TestCase):
    def test_api_me(self):
        """`/me`APIをテスト"""
        client = Client(my_schema)
        self.assertMathSnapshot(client.execute("{ hey }"))
```
