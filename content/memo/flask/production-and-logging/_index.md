---
title: ロギング設定と本番デプロイ
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Flask の logging 設定、RotatingFileHandler、レンタルサーバ/WSGI環境へのデプロイ
slug: /memo/flask/production-and-logging
---

## 概要

Flask アプリケーションにおけるロギング基盤の構築（ログローテーション、出力フォーマット）、およびレンタルサーバ等の本番環境（WSGI / CGI）へのデプロイ設定についてのまとめ。

## 参考ブログ記事

- [Flask ログの設定を行う](/blog/flask-set-logging)
- [Flask アプリをさくらのレンタルサーバで動かす](/blog/flask-on-sakura-rental-server)

---

## 1. ロギング設定 (RotatingFileHandler)

アプリケーションの動作ログをファイル出力し、容量オーバーを防ぐためにサイズローテーションを設定する。

```python
import logging
from logging.handlers import RotatingFileHandler
import os

def setup_logging(app):
    if not os.path.exists("logs"):
        os.mkdir("logs")

    file_handler = RotatingFileHandler(
        "logs/app.log", maxBytes=10240, backupCount=10, encoding="utf-8"
    )
    file_handler.setFormatter(
        logging.Formatter(
            "%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]"
        )
    )
    file_handler.setLevel(logging.INFO)

    app.logger.addHandler(file_handler)
    app.logger.setLevel(logging.INFO)
    app.logger.info("Flask App startup")
```

---

## 2. 本番環境へのデプロイと WSGI 設定

### エントリポイント (`wsgi.py`)
```python
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run()
```

### CGI / レンタルサーバ（さくらのレンタルサーバ等）での注意点
- `.htaccess` によるリライトルールの設定
- 実行ユーザーの権限とパーミッション（`chmod 755 index.cgi` 等）
- 仮想環境（Poetry / venv）の Python インタプリタパスの明示的指定（Shebang 行）
- 本番環境での `FLASK_DEBUG=0` （デバッグモード無効化）の徹底
