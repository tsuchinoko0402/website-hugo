---
title: Flask の環境構築とプロジェクト設計
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Poetry によるパッケージ管理、Application Factory パターン、ディレクトリ構成
slug: /memo/flask/environment-and-structure
---

## 概要

Flask アプリケーションを構築する際の環境構築（Poetry を用いた依存関係管理）、Application Factory パターンによる拡張性の高いプロジェクト初期化、および推奨ディレクトリ構成についてのまとめ。

## 参考ブログ記事

- [Poetry を使って Python のプロジェクトを作成する](/blog/first-step-flask-by-poetry)
- [ここまで作成したものの画面遷移と機能を整理する](/blog/flask-deliverable-organizing)

---

## 1. Poetry による環境構築

### 初期セットアップ
```bash
poetry init
poetry add flask
poetry run flask --version
```

### 仮想環境とパッケージ管理のポイント
- `pyproject.toml` による依存関係とバージョンの厳密管理
- 開発用依存パッケージ（pytest, ruff 等）のグループ分割管理 (`--group dev`)

---

## 2. ディレクトリ構成（推奨パターン）

```text
my_flask_app/
├── app/
│   ├── __init__.py          # create_app() (Application Factory)
│   ├── models/              # SQLAlchemy モデル定義
│   ├── views/               # Blueprint (ルーティング/コントローラ)
│   ├── templates/           # Jinja2 テンプレート
│   └── static/              # CSS / JS / 画像ファイル
├── config/                  # 環境別設定ファイル
├── tests/                   # Pytest テストスイート
├── pyproject.toml
└── wsgi.py                  # アプリケーションエントリポイント
```

---

## 3. Application Factory パターン

アプリケーションの初期化を `create_app()` 関数内にカプセル化することで、テスト時や本番環境で異なる設定を柔軟に適用できるようにする。

```python
from flask import Flask

def create_app(test_config=None):
    app = Flask(__name__, instance_relative_config=True)

    if test_config is None:
        app.config.from_pyfile("config.py", silent=True)
    else:
        app.config.from_mapping(test_config)

    # 拡張機能の初期化（DB, LoginManagerなど）
    # Blueprint の登録

    return app
```
