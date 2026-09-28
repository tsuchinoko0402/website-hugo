---
title: Flask-Login によるユーザー認証
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Flask-Login の導入、UserMixin、user_loader、パスワードハッシュ化とログイン制御
slug: /memo/flask/authentication
---

## 概要

Flask-Login を用いたセッションベースの認証機能の実装。ユーザーモデルへの `UserMixin` 継承、セッション復元用の `user_loader`、Werkzeug を用いた安全なパスワードハッシュ化と照合についてのまとめ。

## 参考ブログ記事

- [Flask でユーザー認証の仕組みを入れる（前編）](/blog/flask-login-introduction)
- [Flask でユーザー認証の仕組みを入れる（後編）](/blog/flask-login-function-creation)

---

## 1. ユーザークラスとパスワードハッシュ化

`werkzeug.security` の関数を用いて平文パスワードを保存しない設計にする。

```python
from flask_login import UserMixin
from werkzeug.security import generate_password_hash, check_password_hash
from app import db

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password_hash = db.Column(db.String(256), nullable=False)

    def set_password(self, password):
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        return check_password_hash(self.password_hash, password)
```

---

## 2. LoginManager の初期化と user_loader

```python
from flask_login import LoginManager

login_manager = LoginManager()
login_manager.login_view = "auth.login"  # 未認証時にリダイレクトするエンドポイント

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))
```

---

## 3. ログイン・ログアウト処理とアクセス制御

```python
from flask import render_template, redirect, url_for, flash, request
from flask_login import login_user, logout_user, login_required, current_user

@bp.route("/login", methods=["GET", "POST"])
def login():
    if current_user.is_authenticated:
        return redirect(url_for("main.index"))

    if request.method == "POST":
        username = request.form.get("username")
        password = request.form.get("password")
        user = User.query.filter_by(username=username).first()

        if user and user.check_password(password):
            login_user(user)
            next_page = request.args.get("next")
            return redirect(next_page or url_for("main.index"))
        flash("ユーザー名またはパスワードが無効です。")

    return render_template("auth/login.html")

@bp.route("/logout")
@login_required
def logout():
    logout_user()
    return redirect(url_for("main.index"))
```
