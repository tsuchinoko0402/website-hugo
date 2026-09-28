---
title: Pytest による単体テストと結合テスト
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Pytest フィクスチャを活用したテスト用アプリ起動、Model 単体テスト、View 結合テスト
slug: /memo/flask/testing
---

## 概要

Flask アプリケーションに対する Pytest を用いたテスト設計。テスト用データベース（インメモリ SQLite）のセットアップ、フィクスチャ設計、ORM モデルのテスト、Test Client を用いたルーティング・View の結合テストについてのまとめ。

## 参考ブログ記事

- [Flask でユニットテストを作成する（Model編）](/blog/flask-unit-test-model)
- [Flask でユニットテストを作成する（View編）](/blog/flask-test-view)

---

## 1. Pytest フィクスチャ設計 (`tests/conftest.py`)

```python
import pytest
from app import create_app, db

@pytest.fixture
def app():
    test_config = {
        "TESTING": True,
        "SQLALCHEMY_DATABASE_URI": "sqlite:///:memory:",
        "WTF_CSRF_ENABLED": False,
    }
    app = create_app(test_config)

    with app.app_context():
        db.create_all()
        yield app
        db.session.remove()
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()

@pytest.fixture
def runner(app):
    return app.test_cli_runner()
```

---

## 2. Model の単体テスト (`tests/test_models.py`)

```python
from app.models import User

def test_user_password_hashing(app):
    user = User(username="testuser")
    user.set_password("mypassword")

    assert user.password_hash != "mypassword"
    assert user.check_password("mypassword") is True
    assert user.check_password("wrongpassword") is False
```

---

## 3. View / ルーティングのテスト (`tests/test_views.py`)

```python
def test_login_page_get(client):
    response = client.get("/auth/login")
    assert response.status_code == 200
    assert b"Login" in response.data

def test_login_post_success(client, app):
    # テストユーザー作成
    with app.app_context():
        user = User(username="admin")
        user.set_password("secret123")
        db.session.add(user)
        db.session.commit()

    response = client.post(
        "/auth/login",
        data={"username": "admin", "password": "secret123"},
        follow_redirects=True,
    )
    assert response.status_code == 200
    assert b"Welcome, admin" in response.data
```
