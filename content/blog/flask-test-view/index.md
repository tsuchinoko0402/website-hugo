---
title: 'Flask でユニットテストを作成する（View編）'
date: 2024-12-20
tags: ["ひとりアドベントカレンダー2024","Python","Flask", "Pytest"]
author: "OKAZAKI Shogo"
draft: true
---
## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の15日目です。 
昨日に引き続いて、Flask で作成したアプリに Pytest を用いて自動テストを作成していきます。
今日はビューのテストです。

## テスト用クライアントを使ったリクエストの送信

まずはとても単純な index の view のテストを書く。

`app/views/index.py`

```python
from flask import Blueprint, render_template

index_bp = Blueprint("index", __name__, url_prefix="/")

logger = logging.getLogger(__name__)


@index_bp.route("/", methods=["GET", "POST"])
def index():
    return render_template("index.html", page_title="TOP")
```

テスト用クライアントは実際のサーバを走らせずにアプリケーションへのリクエストを作成する。Flask のクライアントは Werkzeug のクライアントを拡張している。

テスト用のクライアントは `app.test_client()` を呼び出すことで作成可能。この戻り値をテスト関数に渡すことで、リクエストの送受信のテストが可能になる。

テスト用アプリケーションには `SERVER_NAME` を設定しておく必要がある（設定がないとリクエスト送信時にエラーになる）。

昨日のフィクスチャ作成の流れに則ってテストを以下のように書くことができる：

`tests/views/test_index.py`

```python
import pytest
from flask import url_for

from app import create_app


# Flask アプリケーションのセットアップ
@pytest.fixture
def app():
    test_config = {"TESTING": True, "SERVER_NAME": "localhost"}
    app = create_app(test_config)
    return app


# テスト用のクライアントの設定
@pytest.fixture
def client(app):
    return app.test_client()


# index ページのテスト
def test_indexにアクセスしてトップページが返されること(client):
    # GET リクエストを "/" に送信
    response = client.get(url_for("index.index"))

    # ステータスコード 200 を確認
    assert response.status_code == 200

    # レンダリングされた HTML に "TOP" が含まれていることを確認
    assert b"TOP" in response.data
    assert "<h2>ボーイスカウト阪神さくら地区 加盟員向けページ</h2>".encode() in response.data
```

## DB への参照を伴うページのテスト（テスト用 DB の参照なし）

例として以下のようなビューのテストを書く：

`app/views/files.py`

```python
from flask import Blueprint, render_template

from app.models.file import File

files_bp = Blueprint("files", __name__, url_prefix="/files")


@files_bp.route("/", methods=["GET", "POST"])
def index():
    files = File.query.all()
    return render_template("files.html", files=files, page_title="文書館")
```

DB に対して SELECT 文を発行しているので、`File.query.all()` の処理をモックして、テスト用のデータを返却するようにする。このテストでは、テスト用の DB は必要ない。

`tests/views/test_files.py`

```python
from unittest.mock import patch

import pytest
from flask import url_for

from app import create_app
from app.models.file import File


@pytest.fixture
def app():
    test_config = {"TESTING": True, "SERVER_NAME": "localhost"}
    app = create_app(test_config)
    return app


@pytest.fixture
def client(app):
    # テストクライアントを提供
    return app.test_client()


def test_ファイル一覧に登録されたファイルが表示されること(app, client):
    # モックデータを設定
    mock_files = [
        File(file_id=1, display_name="テスト用ファイル1"),
        File(file_id=1, display_name="テスト用ファイル2"),
    ]

    with app.app_context():
        # File.query.all() をモック
        with patch("app.models.file.File.query") as mock_query:
            mock_query.all.return_value = mock_files

            # テスト対象のエンドポイントにリクエスト
            response = client.get(url_for("files.index"))

            # ステータスコードを検証
            assert response.status_code == 200

            # レスポンスデータを検証
            data = response.data.decode("utf-8")
            assert "文書館" in data
            assert "テスト用ファイル1" in data
            assert "テスト用ファイル2" in data
```

## DB にデータが入っていることを前提とするテスト&フォームのデータ送信を伴うテスト

例として `app/views/login.py` のテストを実装する：

```python
from flask import Blueprint, flash, redirect, render_template, request, url_for
from flask_login import login_user

from app.models.manager_user import ManagerUser

login_bp = Blueprint("login", __name__, url_prefix="/login")


@login_bp.route("/", methods=["GET", "POST"])
def index():
    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        # ManagerUserテーブルからusernameに一致するユーザを取得
        user = ManagerUser.query.filter_by(user_name=username).first()
        if user is None or not user.check_password(password):
            flash("無効なユーザー ID かパスワードです。")
            return redirect(url_for("login.index"))
        login_user(user)
        return redirect(url_for("manage_menu.index"))
    return render_template("login.html", page_title="管理者向けログイン")
```

テスト用の DB を設定し、テスト前にデータを投入する。今回の場合は、ManagerUser オブジェクトを作成し、 `db.session.add()` でデータを追加し、 `db.session.add()` でデータを投入する。

フォームのデータを渡すには、`client.post()` の data 引数に dict を渡す。Content-Type ヘッダーはmultipart/form-data または application/x-www-form-urlencoded へ自動的に設定される。

```python
import pytest
from flask import url_for

from app import create_app, db
from app.models.manager_user import ManagerUser


@pytest.fixture
def app():
    # テスト用の Flask アプリケーションを作成
    test_config = {"TESTING": True, "SERVER_NAME": "localhost", "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:"}
    app = create_app(test_config)

    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()


@pytest.fixture
def client(app):
    return app.test_client()


def test_GETリクエストを送るとログイン画面が返却されること(client, app):
    with app.app_context():
        response = client.get(url_for("login.index"))
        assert response.status_code == 200
        assert "管理者向けログイン" in response.data.decode("utf-8")


def test_POSTリクエストでユーザー名とパスワードを受け取りログインが成功すること(client, app):
    with app.app_context():
        # ユーザーデータを作成とDBへの投入
        user = ManagerUser(user_name="testuser")
        user.set_password("password123")
        db.session.add(user)
        db.session.commit()

    # ログインのPOSTリクエストを送信
    response = client.post(
        url_for("login.index"),
        data={"username": "testuser", "password": "password123"}
        # リダイレクトを追跡しない
        follow_redirects=False, 
    )

    # リダイレクト先を確認
    assert response.status_code == 302
    # 相対パスで比較
    assert response.location == url_for("manage_menu.index", _external=False)


def test_POSTリクエストで無効なパスワードを受け取るとログインが失敗すること(client, app):
    with app.app_context():
        # ユーザーデータを作成
        user = ManagerUser(user_name="testuser")
        user.set_password("password123")
        db.session.add(user)
        db.session.commit()

    response = client.post(
        url_for("login.index"),
        data={"username": "testuser", "password": "wrongpassword"},
        follow_redirects=True,
    )

    assert response.status_code == 200
    assert "無効なユーザー ID かパスワードです。" in response.data.decode("utf-8")
```


もし、data 引数に渡された dict に含まれる値が "rb" モード file オブジェクトの場合、アップロードされたファイルとして扱われる。検知されたファイル名とcontent typeを変更するには、`file: (open(filename, "rb"), filename, content_type)` のように、dict 内のアイテムで該当する tuple を値に設定する。file オブジェクトは、通常の `with open() as f:` のパターンを使う必要が無いように、リクエストを作成した後に閉じられる。

ファイルオブジェクトを POST で送信するテストを行う例：

```python
from pathlib import Path

# get the resources folder in the tests folder
resources = Path(__file__).parent / "resources"

def test_edit_user(client):
    response = client.post("/user/2/edit", data={
        "name": "Flask",
        "theme": "dark",
        "picture": (resources / "picture.png").open("rb"),
    })
    assert response.status_code == 200
```

## ログインが必要な画面の実装例：

以下のように `@login_required` がついているエンドポイントのテストを実装する。

`app/views/manage_menu.py`

```python
from flask import Blueprint, render_template
from flask_login import login_required

manage_menu_bp = Blueprint("manage_menu", __name__, url_prefix="/manage-menu")


@manage_menu_bp.route("/", methods=["GET"])
@login_required
def index():
    return render_template("manage-menu.html", page_title="管理者向けメニュー")
```

ログイン画面ではユーザーID（`username`）とパスワード（`password`）が POST で送信される。テストコード内では、ログインのエンドポイントにアクセスするためのログインヘルパーを利用する。

Flask のコンテキストの変数（主に `session`）へアクセスするには `with` 文の中でクライアントを使う。app と request のコンテキストは、`with` ブロックを終了するまで、リクエストを作成した後も残る。

`session` には、ログイン処理のところで説明した通り、ユーザー ID にあたるものが str 型で格納されている。

```python
import pytest
from werkzeug.security import generate_password_hash
from flask import session

from app import create_app, db
from app.models.manager_user import ManagerUser


@pytest.fixture
def app():
    """テスト用の Flask アプリケーションを作成"""
    test_config = {
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "SQLALCHEMY_TRACK_MODIFICATIONS": False,
        "SECRET_KEY": "test_secret",
    }
    app = create_app(test_config)

    with app.app_context():
        db.create_all()
        # テスト用のユーザーを追加
        test_user = ManagerUser(
            user_name="test_user",
            password=generate_password_hash("password123"),
            service="test_service",
        )
        db.session.add(test_user)
        db.session.commit()
        yield app
        db.drop_all()


@pytest.fixture
def client(app):
    """テスト用クライアントを作成"""
    return app.test_client()


@pytest.fixture
def login(client):
    """テスト用ログインヘルパー"""

    def do_login(username, password):
        return client.post(
            "/login",
            data={"username": username, "password": password},
            follow_redirects=True,
        )

    return do_login


def test_ログインせずに保護されたエンドポイントにアクセスすると401エラー(client):
    response = client.get("/manage-menu/")
    assert response.status_code == 401


def test_ログイン後に保護されたエンドポイントにアクセスすると管理者メニューに遷移すること(client, login):
    with client:
        response = login("test_user", "password123")
        assert response.status_code == 200  # ログイン成功
        assert session["_user_id"] == '1'

        response = client.get("/manage-menu/")
        assert response.status_code == 200  # 正常にアクセス可能
        assert "管理者向けメニュー".encode() in response.data  # ページの内容を検証
        assert session["_user_id"] == '1'
```

## 参考資料

- [Flaskアプリケーションのテスト — Flask Documentation (2.2.x)](https://msiz07-flask-docs-ja.readthedocs.io/ja/latest/testing.html)
- [【Flask】TODOアプリ作成その５：単体テスト│ぼんの備忘録](https://techorgana.com/python/web_framework/flask/2958/)
