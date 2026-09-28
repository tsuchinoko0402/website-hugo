---
title: ルーティングと Blueprint によるモジュール分割
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Blueprint を用いた機能分割、リクエストパラメータ受け取り、Jinja2 テンプレート連携
slug: /memo/flask/routing-and-blueprint
---

## 概要

アプリケーションの規模拡大に伴うコード肥大化を防ぐための Blueprint によるモジュール分割、GET/POST/JSON 等のリクエストデータ取得方法、Jinja2 テンプレートの継承手法についてのまとめ。

## 参考ブログ記事

- [Flask の Blueprint を使う](/blog/flask-blue-print)
- [Flask でリクエスト経由でデータを受け取る](/blog/flask-receive-request)
- [Flask のテンプレート（Jinja2）を活用する](/blog/flask-template-html)

---

## 1. Blueprint による機能分割

機能ごと（例: 認証、ブログ、管理画面）に Blueprint を定義し、独立したルーティングとビューを管理する。

### Blueprint の定義例 (`app/views/auth.py`)
```python
from flask import Blueprint, render_template, request, redirect, url_for

bp = Blueprint("auth", __name__, url_prefix="/auth")

@bp.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        # ログイン処理
        pass
    return render_template("auth/login.html")
```

### アプリケーションへの登録 (`app/__init__.py`)
```python
from app.views import auth

def create_app():
    app = Flask(__name__)
    app.register_blueprint(auth.bp)
    return app
```

---

## 2. リクエストパラメータの受け取り

| データ種別 | 取得方法 | 例 |
| :--- | :--- | :--- |
| **URL パスパラメータ** | ルーティングパラメータ | `@bp.route("/users/<int:user_id>")` |
| **クエリパラメータ (GET)** | `request.args.get("key")` | `/?page=1` |
| **フォームデータ (POST)** | `request.form.get("key")` | `<form method="POST">` |
| **JSON ボディ** | `request.get_json()` | API リクエスト |

---

## 3. Jinja2 テンプレートとレイアウト継承

`base.html` を作成し、各テンプレートで `{% extends "base.html" %}` を使用して共通レイアウトを継承する。
`url_for('auth.login')` のように `[blueprint_name].[endpoint]` 形式で URL を逆引きする。
