---
title: 'Flask でユニットテストを作成する（Model編）'
date: 2024-12-19
tags: ["ひとりアドベントカレンダー2024","Python","Flask", "Pytest"]
author: "OKAZAKI Shogo"
draft: true
---

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の14日目です。 
今更ながら、Flask で作成したアプリに Pytest を用いて自動テストを作成していきます。
色々手こずってしまったので、今日はひとまずモデルのテストのみです。

## Pytest のフィクスチャ

Pytest を用いて Flask のテストを作成するにあたって、 Pytest の「フィクスチャ」を多用することになるので、概要を確認する。

ソフトウェアテストにおける「フィクスチャ」とは、テスト対象のソフトウェアのコンポーネントの実際の使い方を模倣できるように、固定された環境設定をシミュレートするものである。Pytest においては、テスト関数の依存として提供できる、 setup/teardown 内の再利用可能なパーツがフィクスチャである。

Pytest を使うには、いつもの通り、パッケージを導入すれば良い：

```shell
poetry add pytest --group dev
```

### Pytest のフィクスチャの作成

以下のように関数に `@pytest.fixture` デコレータをつけるとフィクスチャを作成できる。

```python
import pytest


@pytest.fixture
def dependency():
    return "fixture value"


def test_one(dependency):
    assert dependency == "fixture value"
```

フィクスチャ関数の戻り値は、テスト関数に入力引数として渡すことができる。その際、フィクスチャ関数名をそのまま記述すればいよい。

以下のようなジェネレータ構文を使うと、同じフィクスチャ関数内で setup と teardown の両方を提供できる。

```python
import pytest


@pytest.fixture
def dependency():
    # ここに setup のための処理を書く
    yield "fixture value"
    # ここに teardown のための処理を書く
```

### フィクスチャのスコープ

フィクスチャの値のライフタイムを決定するスコープとして以下の5つが利用可能。

- "function" スコープ：デフォルトのスコープで、各テストの実行ごとに一度だけ実行され、そのあと破棄される。
- "class" スコープ：xUnit スタイルで書かれたテストメソッドに使用できる。テストクラス内の最後のテストの後に破棄される。
- "module" スコープ：テストモジュール内の最後のテストの後に破棄される。
- "package" スコープ：テストパッケージの最後のテストの後に破棄される。
- "session" スコープ：一種のグローバルスコープで、ランナーの実行全体を通じて生き続け、最後のテストの後に破棄される。

### Flask におけるフィクスチャの利用

Flask においては、テスト用の `app` のフィクスチャを用意し、各テストケースに渡すことができる。

今回作成しているアプリのように `create_app()` 関数を呼び出して `app` を生成する場合は以下のような感じでテスト用のクライアントを作成する。今の実装のままだと、元のアプリケーションが DB 接続を持っている場合、テスト用の設定が上書きされずに元の設定が利用されてしまう可能性があるため、テスト時にテスト用の設定を上書きするようにする。`app/__init__.py` を以下のように書きあけ、引数でテスト用設定を受け取れるようにする。


```python
def create_app(test_config=None):
    # appの設定
    app = Flask(__name__, instance_relative_config=True)

    # configファイルを読み込む
    config_path = os.path.join("config", "dev.py")
    app.config.from_pyfile(config_path)

    # dev.py からログ設定を読み込んで適用
    dictConfig(app.config["LOGGING_CONFIG"])

    # テスト用設定が渡された場合に上書き
    if test_config:
        app.config.update(test_config)
    
    # 以下省略
```

テストコードでは以下のようにする。

```python
import pytest

from app import create_app


@pytest.fixture
def app():
    """テスト用のFlaskアプリケーションを設定"""
    test_config = {
        'TESTING': True
    }
    app = create_app(test_config)
    # setup 処理
    yield app
    # teardown 処理
```

あとは、テストで app を利用すれば良い。

## Flask におけるテストの配置

以下のようにすれば良い。

- テスト用プログラムは `tests` ディレクトリに入れる
- テストは `tests_` から始める関数。
- `test_` から始めるモジュールの中に入れる。
- テストクラスは `Test` で始まる。

以下は構成例：

```
.
|-- Makefile
|-- app
|   |-- __init__.py
|   |-- app.py
|   |-- index.cgi
|   |-- models
|   |   |-- file.py
|   |   `-- manager_user.py
|   |-- templates
|   `-- views
|       |-- files.py
|       |-- index.py
|       `-- login.py
|-- instance
|   |-- config
|       `-- dev.py
|-- poetry.lock
|-- pyproject.toml
|-- pytest.ini
`-- tests
    |-- models
    |   |-- test_file.py
    |   `-- test_manager_user.py
    `-- views
        |-- test_files.py
        |-- test_index.py
        `-- test_login.py
```

## モデルのテスト：DB 接続を伴うテストを行う

テスト用のデータを挿入してテストを実施したい場合があるが、本番と同じDBを用いたりするのは手間なので、 [SQLite のインメモリー](https://www.sqlite.org/inmemorydb.html) を利用する。

テスト用の app を作成する際に以下のようにする。

```python
import pytest

from app import create_app, db


@pytest.fixture
def app():
    """テスト用のFlaskアプリケーションを設定"""
    # テスト用の設定を渡してアプリケーションを作成
    test_config = {
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "SQLALCHEMY_TRACK_MODIFICATIONS": False,
    }
    app = create_app(test_config)
    with app.app_context():
        # テスト用のデータベーステーブルを作成
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()


@pytest.fixture
def db_session(app):
    """テスト用のデータベースセッション"""
    with app.app_context():
        yield db.session
```

`app.app_context()` は Flask アプリケーションの アプリケーションコンテキスト を明示的に作成し、その範囲内で動作するための仕組み。通常、Flask アプリケーションはリクエストが処理されるときに自動的にアプリケーションコンテキストを作成する。しかし、リクエストの外側でアプリケーション関連の操作を行いたい場合（例えば、データベース操作やテスト実行）には、`app.app_context()` を明示的に使用してコンテキストを作成する必要がある。

上記の例では、アプリケーションコンテキスト内で、テスト実行前に DB を初期化し、テスト後に DB とのセッションを切り、 DB 内のデータを消すという動作を行っている。


### `app/models/file.py` のテスト例

簡単な例として、以下のような File モデルのテストを書く：

```python
from app import db


# File テーブル
class File(db.Model):
    __tablename__ = "File"
    file_id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    file_name = db.Column(db.String(255))
    display_name = db.Column(db.String(255))
    url = db.Column(db.String(255))
    file_type = db.Column(db.String(255))
    size = db.Column(db.String(255))
    description = db.Column(db.String(255))
    tag = db.Column(db.String(255))
    is_standard = db.Column(db.Integer)
    created_at = db.Column(db.String(255))
    created_by = db.Column(db.String(255))
    updated_at = db.Column(db.String(255))
    updated_by = db.Column(db.String(255))
```

以下のようにすれば良い。

```python
import pytest

from app import create_app, db
from app.models.file import File


@pytest.fixture
def app():
    """テスト用のFlaskアプリケーションを設定"""
    # テスト用の設定を渡してアプリケーションを作成
    test_config = {
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "SQLALCHEMY_TRACK_MODIFICATIONS": False,
    }
    app = create_app(test_config)
    with app.app_context():
        # テスト用のデータベーステーブルを作成
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()


@pytest.fixture
def db_session(app):
    """テスト用のデータベースセッション"""
    with app.app_context():
        yield db.session


def test_FileモデルがDBに作成されること(db_session):
    file = File(
        file_name="test.pdf",
        display_name="テスト用ファイル",
        url="https://hoge.com/test.pdf",
        file_type="pdf",
        size="1 MB",
        description="テスト用のファイルです",
        tag="BS, CS",
        is_standard=1,
        created_at="2024-12-16 12:00:00",
        created_by="okazaki",
        updated_at="2024-12-16 12:00:00",
        updated_by="okazaki",
    )
    db_session.add(file)
    db_session.commit()

    # データベースに挿入されたことを確認
    saved_file = db_session.query(File).filter_by(file_name="test.pdf").first()
    assert saved_file is not None
    assert saved_file.display_name == "テスト用ファイル"
    assert saved_file.size == "1 MB"


def test_FileモデルがDBで更新されること(db_session):
    # 初期データを挿入
    file = File(file_name="update.pdf", display_name="更新前の名前")
    db_session.add(file)
    db_session.commit()

    # 更新処理
    file_to_update = db_session.query(File).filter_by(file_name="update.pdf").first()
    file_to_update.display_name = "更新後の名前"
    db_session.commit()

    # 更新されたことを確認
    updated_file = db_session.query(File).filter_by(file_name="update.pdf").first()
    assert updated_file.display_name == "更新後の名前"


def test_FileモデルがDBから削除されること(db_session):
    # 初期データを挿入
    file = File(file_name="delete.pdf", display_name="削除用ファイル")
    db_session.add(file)
    db_session.commit()

    # 削除処理
    file_to_delete = db_session.query(File).filter_by(file_name="delete.pdf").first()
    db_session.delete(file_to_delete)
    db_session.commit()

    # 削除されたことを確認
    deleted_file = db_session.query(File).filter_by(file_name="delete.pdf").first()
    assert deleted_file is None
```

### `app/models/manager_user.py` のテスト例

ログインユーザーを管理する `manager_user.py` では、セッションからユーザー情報を取り出す load_user() 関数があるので、これをテストするコードを以下のように作成する。

`app/models/manager_user.py` の実装：

```python
from flask_login import UserMixin
from werkzeug.security import check_password_hash, generate_password_hash

from app import db, login_manager


# ManagerUser テーブル
class ManagerUser(UserMixin, db.Model):
    __tablename__ = "ManagerUser"
    user_id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    user_name = db.Column(db.String(255))
    password = db.Column(db.String(255))
    service = db.Column(db.String(255))
    logined_at = db.Column(db.String(255))

    def get_id(self):
        return str(self.user_id)  # IDをstr型で返す
    # 中略


@login_manager.user_loader
def load_user(user_id: str) -> ManagerUser:
    with db.session() as session:
        return session.get(ManagerUser, int(user_id))
```

`tests/models/test_manager_user.py` の一部：

```python
import pytest
from werkzeug.security import generate_password_hash

from app import create_app, db
from app.models.manager_user import ManagerUser


@pytest.fixture
def app():
    """テスト用のFlaskアプリケーションを設定"""
    test_config = {
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "SQLALCHEMY_TRACK_MODIFICATIONS": False,
    }
    app = create_app(test_config)
    with app.app_context():
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()


@pytest.fixture
def db_session(app):
    """テスト用のデータベースセッション"""
    with app.app_context():
        yield db.session
    # 中略

def test_load_user(db_session):
    """load_user関数のテスト"""
    # テスト用ユーザーを作成
    user = ManagerUser(user_id=1, user_name="loader_test_user")
    db_session.add(user)
    db_session.commit()

    # load_user 関数でユーザーをロード
    from app.models.manager_user import load_user

    loaded_user = load_user("1")

    assert loaded_user is not None
    assert loaded_user.user_name == "loader_test_user"
```

## 参考資料

- [Flaskアプリケーションのテスト — Flask Documentation (2.2.x)](https://msiz07-flask-docs-ja.readthedocs.io/ja/latest/testing.html)
- [【Flask】TODOアプリ作成その５：単体テスト│ぼんの備忘録](https://techorgana.com/python/web_framework/flask/2958/)
- [Flaskの単体テスト - 合同会社inctore](https://www.inctore.com/blog/unit-test-for-flask/)
- [エキスパートPythonプログラミング 改訂4版](https://www.kadokawa.co.jp/product/302304004673/)
-  [In-Memory Databases](https://www.sqlite.org/inmemorydb.html)
