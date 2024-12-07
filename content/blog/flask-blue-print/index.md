---
title: 'Flask の Blueprint を使う'
date: 2024-12-07T00:40:20+09:00
tags: ["ひとりアドベントカレンダー2024","Python","Flask"]
author: "OKAZAKI Shogo"
draft: true
---

## はじめに
「OKAZAKI Shogo のひとりアドベントカレンダー2024」の5日目です。
平日限定で続けようとしてましたが、昨日できなかったので・・・ 
ここまで作った Flask のアプリですが、今後のことも考えてディレクトリ構造を少し変えてみます。

## MVT モデル

Flask では MVT モデルを採用している。

MVC モデルでは、
- Model：ビジネスロジックを記述。データベースアクセスを伴うことが多い。
- View：画面描画を担当。ユーザからの入力も受け付ける。
- Controller：URL を振り分ける。 View からのデータを Model に渡したり、逆に Model から View にデータを渡す役割を担う。
と言った具合だが、MVT モデルでは、
- Model : MVC モデルの Model と同じ役割
- View : MVC モデルの Controller と同じ役割
- Template : MVC モデルの View と同じ役割
と少し役割が異なる。

## アプリ構造の変更

`app.py` にコードが集中しすぎないように、上記の MVT モデルの役割でコードを分割する。以下のような構造に変更する。

```
.
|-- Makefile
|-- app <-- 名前変更
|   |-- __init__.py <-- 処理を追加
|   |-- app.py
|   |-- index.cgi
|   |-- static
|   |   `-- css
|   |       `-- default.css
|   |-- templates
|   |   |-- index.html
|   |   `-- layout.html
|   `-- views
|       `-- index.py <-- 新規作成
|-- instance
|   `-- config
|       `-- dev.py　<-- 新規作成
|-- poetry.lock
`-- pyproject.toml
```

## Flask の Blueprint（青写真）

分割するにあたって、Flask の Blueprint 機能を使う。以下、公式マニュアルの説明：

> Blueprintは、関連するviewおよびその他のコードをグループへと編成する方法です。viewおよびその他のコードを直接Flaskアプリケーションに登録するよりも、代わりにそれらをblueprintに登録します。それから、factory関数の中でFlaskアプリケーションが利用可能になったときに、blueprintをFlaskアプリケーションに登録します。

まずは、 `__init__.py` で config ファイルを読み込み、 Blueprint の登録を行った上で、アプリケーションを作成するように変更する。

`__init__.py`
```python
import os
from flask import Flask


def create_app():
    # appの設定
    app = Flask(__name__, instance_relative_config=True)
    # configファイルを読み込む
    config_path = os.path.join('config', 'dev.py')
    app.config.from_pyfile(config_path)
    
    # Blueprint の登録
    from app.views.index import index_bp

    app.register_blueprint(index_bp)

    return app
```

次に、 `app.py` では、アプリを起動するように変更する。

`app.py`
```python
from app import create_app

app = create_app()

if __name__ == '__main__':
    app.run()
```

`index.py` では、 `/` にアクセスがあった場合の処理を書く。また、「index」という名前の Blueprint も作成する。Blueprintは自分がどこで定義されているか知る必要があるため、`__name__` が2番目の引数として渡されている。`url_prefix` は、 Blueprint と関連付けられる全ての URL の（パス部分の）先頭に付けられる。

`index.py`
```python
from flask import Blueprint, render_template

index_bp = Blueprint('index', __name__, url_prefix='/')


@index_bp.route("/", methods=["GET", "POST"])
def index():
    return render_template('index.html')
```

最後に、環境ごとの設定を記述するファイルを作成する。とりあえず、開発環境のものだけ。

`dev.py`
```python
DEBUG = True
```

これで起動して、前と同じ動作になっていれば OK。

## 参考資料

- [青写真とビュー（Blueprints and Views） — Flask Documentation (2.2.x)](https://msiz07-flask-docs-ja.readthedocs.io/ja/latest/tutorial/views.html)
- [【Flask】中規模な開発のディレクトリ構成を考える #Python - Qiita](https://qiita.com/Koichi73/items/9d73f062f0ad56d6f953)
- [【Flask】ディレクトリ構成を考える - AI can fly !!](https://ai-can-fly.hateblo.jp/entry/flask-directory-structure)
