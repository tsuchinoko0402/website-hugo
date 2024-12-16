---
title: 'Flask でユーザー認証の仕組みを入れる（前編）'
date: 2024-12-16
tags: ["ひとりアドベントカレンダー2024","Python","Flask", "flask-Login"]
author: "OKAZAKI Shogo"
---

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の11日目です。 
今回実装するページで、ユーザー認証が必要な箇所があるので、ユーザー認証の仕組みを実装していきます。まずは、Flask-Login の導入から行います。


### flask-login を使うための準備
[Flask-Login 0.7.0 documentation](https://flask-login.readthedocs.io/en/latest/)に沿って導入する。

ライブラリを追加する：
```shell
poetry add flask-login
```

`LoginManager` クラスをアプリに追加する。今回作成のアプリでは、flask-login の初期化として以下のようにする：

```python
import os
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
import flask_login # 追加


db = SQLAlchemy()


def create_app():
    # appの設定
    app = Flask(__name__, instance_relative_config=True)

    # configファイルを読み込む
    config_path = os.path.join("config", "dev.py")
    app.config.from_pyfile(config_path)

    # DB の設定
    db.init_app(app)

    # flask-login の初期化（追加）
    login_manager = flask_login.LoginManager()
    login_manager.init_app(app)

    # Blueprint の登録
    from app.views.index import index_bp
    from app.views.files import files_bp
    from app.views.regist_file_form import regist_file_form_bp
    from app.views.regist_file import regist_file_bp
    from app.views.manage_file import manage_file_bp
    from app.views.manage_file_list import manage_file_list_bp
    from app.views.update_file import update_file_bp
    from app.views.manage_menu import manage_menu_bp

    app.register_blueprint(index_bp)
    app.register_blueprint(files_bp)
    app.register_blueprint(regist_file_form_bp)
    app.register_blueprint(regist_file_bp)
    app.register_blueprint(manage_file_bp)
    app.register_blueprint(manage_file_list_bp)
        app.register_blueprint(update_file_bp)
    app.register_blueprint(manage_menu_bp)

    return app
```

まだ、ログインやログアウトに関するエンドポイントは実装してない。

今回は、管理者ユーザーのみ認証を行う。管理者ユーザーの Model は以下の通り：

#### `models/manager_user.py`

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

    def set_password(self, password):
        self.password = generate_password_hash(password)
        return self.password

    def check_password(self, password):
        print(password)
        print(self.password)
        return check_password_hash(self.password, password)


@login_manager.user_loader
def load_user(user_id: str) -> ManagerUser:
    with db.session() as session:
        return session.get(ManagerUser, int(user_id))
```

ユーザーモデルについては、以下をのこと留意する：
- ユーザークラスを簡単に実装するには、`flask_login.UserMixin` を継承することができる。
    - これを継承することで、flask-login においてユーザーを表すクラスで求められる、以下のプロパティとメソッドが実装される：
        - `is_authenticated`：ユーザーが認証されている場合に True を返す。
        - `is_active`：ユーザーがアクティブな場合に True を返す。認証済みであることに加えて、以下の条件も満たしている必要があります:
            - カウントが有効化されている。
            - 停止されていない。
            - アプリケーションが定めるその他の拒否条件に該当していない。
        - `is_anonymous`：このプロパティは、匿名ユーザーの場合に True を返す。（実際のユーザーは代わりに False を返す。）
        - `get_id()`：ユーザーを一意に識別する str を返す。この値は、`user_loader` コールバックからユーザーをロードする際に使用される。
            - ID が元々 int 型や他の型の場合、それを str に変換する必要がある。
            - ユーザー ID に当たるプロパティの名前が `id` ではない場合、独自に実装する必要がある（今回はそれに相当）
- flask-login が呼び出すコールバック関数 `user_loader` を定義しておく必要がある（未定義だとエラー）。
    - このコールバック関数は、セッションに保存されているユーザー ID からユーザーオブジェクトを再ロードするために呼び出される。この関数が `None` を返すとユーザ ID に該当するユーザが存在しないものと判断される。
    - flask-login は `@login_manager.user_loader` のデコレータがついた関数を呼ぶだけなので、情報がどこにあって、どう読み取るかについてはプログラムする必要がある。
    - 今回の例では、引数に str 型の ID を受け取り、対応するユーザーオブジェクトを返す。
- Flask-Loginは`user_loader()` を呼ぶだけなので、情報がどこにあって、どう読み取るかについては実装する必要がある。
    - `@login_manager.user_loader` のデコレータをつけた関数がそれに相当。
    - この関数は、引数にID（`str`）を受け取り、指定したユーザーを返すもの。

パスワードはハッシュ化されたものを格納するようにする。Flaskの依存モジュールとして一緒にインストールされる Werkzeug に、パスワード暗号化に使える関数があるので、これを使う。`set_password()` と `check_password()` では、ハッシュ化されたパスワードを格納したり、ハッシュ化されたパスワードで比較をするようにしている。

### パスワードハッシュを生成するツールを作成する

今回、ユーザー登録をする機能は（とりあえずは）設けていないので、パスワードを生成するためのツールを用意する（Werkzeug が入っていれば使える）

`tools/generate_password_hash.py`
```python
from werkzeug.security import generate_password_hash

def hash_password():
    # ユーザーにパスワードを入力させる
    password = input("Enter a password to hash: ")

    # パスワードをハッシュ化 (デフォルトのメソッドは 'pbkdf2:sha256')
    hashed_password = generate_password_hash(password)

    # ハッシュ化したパスワードを出力
    print("Hashed password:", hashed_password)

if __name__ == "__main__":
    hash_password() 
```

```shell
$ poetry run python tools/generate_password_hash.py
The currently activated Python version 3.13.0 is not supported by the project (3.8.12).
Trying to find and use a compatible version.
Using python3 (3.8.12)
Enter a password to hash: python
Hashed password: scrypt:32768:8:1$JGZGtMoMS8C3Kv4G$571fb0326cd920b0b59f73ac56c1a64dfefa22ecd3f57e06c5966487963aa54700c936b521bf756f5cf72934870b90587d3b3e812249d8019258cc73ee23a4ba
```

ここで得た値を ManagerUser テーブルの password に書き込んでおく。

実際のログイン・ログアウトの処理とログインが必要なページの作成は明日の記事にて。

## 参考資料

- [Flask-Login 0.7.0 documentation](https://flask-login.readthedocs.io/en/latest/)
- [とほほのFlask入門 - とほほのWWW入門](https://www.tohoho-web.com/ex/flask.html#login)
- [Flaskでユーザ認証をさせてみよう - PythonOsaka](https://scrapbox.io/PythonOsaka/Flask%E3%81%A7%E3%83%A6%E3%83%BC%E3%82%B6%E8%AA%8D%E8%A8%BC%E3%82%92%E3%81%95%E3%81%9B%E3%81%A6%E3%81%BF%E3%82%88%E3%81%86)
- [Flaskのセッション管理とセキュリティについて](https://zenn.dev/saiki_toshiki/articles/946e4a3c2eb4c5)
