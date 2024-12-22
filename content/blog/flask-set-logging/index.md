---
title: 'Flask ログの設定を行う'
date: 2024-12-23
tags: ["ひとりアドベントカレンダー2024","Python","Flask", "Logging"]
author: "OKAZAKI Shogo"
---

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の16日目です。 
Flask アプリでログの設定を入れていきます。

## ログの設定

`instance/config/dev.py` にログの設定を記述する。`dictConfig()` を使用する。

今回は以下のような設定をする：
- ログはアプリ、アクセス、エラー用でそれぞれファイル出力する
- ファイルは日付ごとに分ける
- ログファイルは `logs/YYYY/MM` と月ごとに分ける
- コンソールにはデバッグレベルの全てのログを出力する

```python
import os
from datetime import datetime, timedelta, timezone

# 中略

# ログの設定
# ログファイルを月ごとに分ける
# JST のタイムゾーンを考慮して、月ごとのタイムスタンプを取得
JST = timezone(timedelta(hours=9))
current_time_jst = datetime.now(JST)
year_month = current_time_jst.strftime("%Y/%m")

# 年ごとのログディレクトリ
LOG_FOLDER = os.path.join(BASE_DIR, "logs", year_month)
if not os.path.exists(LOG_FOLDER):
    os.makedirs(LOG_FOLDER)

# ログファイルのパス
ACCESS_LOG_FILE = os.path.join(LOG_FOLDER, f'access_{current_time_jst.strftime("%Y-%m-%d")}.log')
ERROR_LOG_FILE = os.path.join(LOG_FOLDER, f'error_{current_time_jst.strftime("%Y-%m-%d")}.log')
APP_LOG_FILE = os.path.join(LOG_FOLDER, f'app_{current_time_jst.strftime("%Y-%m-%d")}.log')
LOGGING_CONFIG = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "default": {
            "format": "[%(asctime)s] %(levelname)-7s | %(name)s | %(funcName)s.%(lineno)d | %(message)s",
        },
        "app": {
            "format": "[%(asctime)s] %(levelname)-7s | %(name)s | %(filename)s:%(lineno)d | %(message)s",
        },
    },
    "handlers": {
        # アクセスログ（Werkzeugのアクセスログ）
        "access_log_handler": {
            "level": "INFO",
            "class": "logging.handlers.TimedRotatingFileHandler",
            "filename": ACCESS_LOG_FILE,
            "when": "midnight",  # 毎日0時に新しいファイルにローテーション
            "interval": 1,  # 1日ごと
            "backupCount": 30,  # 過去30日分のログを保持
            "formatter": "default",
        },
        # エラーログ（エラー専用のログファイル）
        "error_log_handler": {
            "level": "ERROR",
            "class": "logging.handlers.TimedRotatingFileHandler",
            "filename": ERROR_LOG_FILE,
            "when": "midnight",
            "interval": 1,
            "backupCount": 30,
            "formatter": "default",
        },
        # アプリのログ（アプリケーションの動作を記録）
        "app_log_handler": {
            "level": "INFO",
            "class": "logging.handlers.TimedRotatingFileHandler",
            "filename": APP_LOG_FILE,
            "when": "midnight",
            "interval": 1,
            "backupCount": 30,
            "formatter": "app",
        },
        # コンソールログ（デバッグ用）
        "console": {
            "level": "DEBUG",
            "class": "logging.StreamHandler",
            "formatter": "default",
        },
    },
    "loggers": {
        # アクセスログ
        "access": {
            "handlers": ["access_log_handler"],
            "level": "INFO",
            "propagate": True,
        },
        # エラーログ
        "error": {
            "handlers": ["error_log_handler"],
            "level": "ERROR",
            "propagate": True,
        },
        # アプリケーションのログ
        "app": {
            "handlers": ["app_log_handler"],
            "level": "INFO",
            "propagate": True,
        },
        # すべてのログ
        "": {
            "handlers": ["console"],
            "level": "DEBUG",
            "propagate": False,
        },
    },
}
```

## Flask アプリへのログの設定

上記で設定したログの設定を `app.config["LOGGING_CONFIG"]` で取り出したものを `dictConfig()` に渡す。

`app/__init__.py`

```python
import os
from logging.config import dictConfig

# 中略

def create_app(test_config=None):
    # appの設定
    app = Flask(__name__, instance_relative_config=True)

    # configファイルを読み込む
    config_path = os.path.join("config", "dev.py")
    app.config.from_pyfile(config_path)

    # dev.py からログ設定を読み込んで適用
    dictConfig(app.config["LOGGING_CONFIG"])

    # 以下略
```

## アクセスログとエラーログの出力設定

Flask アプリの設定で、アクセスログとエラーログを出力するように設定する。
- `@app.before_request` で受け取ったリクエストを処理する前に実行する関数を指定する
    - 今回は、受け取ったリクエストの内容をアクセスログに出力する関数に付与する
- `@app.after_request` でレスポンスをクライアントを処理する前に実行する関数を指定する
    - 今回は、返却するレスポンスの内容をアクセスログに出力する関数に付与する
- `@app.errorhandler(Exception)` で全ての Exeption が発生した際に実行する関数を指定する
    - 今回は、 Exception が発生した場合に内容をエラーログに出力する関数を指定する

`app/app.py` に以下のように設定する：

```python
import logging

from flask import flash, redirect, request, url_for
from werkzeug.exceptions import RequestEntityTooLarge

from app import create_app

app = create_app()


if __name__ == "__main__":
    app.run()


@app.errorhandler(RequestEntityTooLarge)
def handle_over_max_file_size(error):
    flash("ファイルのサイズが大きすぎます。 50 MB 以下のファイルを選択してください。")
    return redirect(url_for("regist_file_form.index"))


def headers_to_string(headers):
    return ", ".join(f"{key}: {value}" for key, value in headers.items())


access_logger = logging.getLogger("access")
error_logger = logging.getLogger("error")


# 各リクエストの前後でログを記録
@app.before_request
def log_request_info():
    request_info = (
        f"Request: method={request.method}, url={request.url}, "
        f"headers=[{headers_to_string(request.headers)}], "
        f"body={request.get_data(as_text=True)}"
    )
    access_logger.info(request_info)


@app.after_request
def log_response_info(response):
    response_info = f"Response: status={response.status}, headers=[{headers_to_string(response.headers)}]"
    access_logger.info(response_info)
    return response


# 例外が発生したときのエラーログ
@app.errorhandler(Exception)
def handle_exception(e):
    error_logger.error(f"Exception occurred: {e}", exc_info=True)
    return "An error occurred", 500
```

## アプリログの出力

次のようにして使うとアプリログが出力される：

```python
import logging

from flask import Blueprint, render_template

index_bp = Blueprint("index", __name__, url_prefix="/")

logger = logging.getLogger("app")


@index_bp.route("/", methods=["GET", "POST"])
def index():
    logger.debug("インデックスページを表示")
    return render_template("index.html", page_title="TOP")
```

## ログの出力例

上記の設定では以下のようなログが出力される

### アプリログ

`logs/2024/12/app_2024-12-19.log`

```
[2024-12-19 00:49:56,350] INFO    | app | index.py:12 | インデックスページを表示
```

### アクセスログ

`logs/2024/12/access_2024-12-19.log`

```
[2024-12-19 00:49:56,350] INFO    | access | log_request_info.36 | Request: method=GET, url=http://127.0.0.1:5000/, headers=[Host: 127.0.0.1:5000, User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:133.0) Gecko/20100101 Firefox/133.0, Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8, Accept-Language: ja,en-US;q=0.7,en;q=0.3, Accept-Encoding: gzip, deflate, br, zstd, Dnt: 1, Sec-Gpc: 1, Connection: keep-alive, Referer: http://127.0.0.1:5000/, Cookie: session=.eJwlzsENwzAIAMBdePcBBtuQZSJjg9pv0ryq7t5IXeB0H9jziPMJ2_u44gH7a8EGyk59SYRItqlOZs4SXBRXTNY6Wqc2q0nDtZhqRqpwiLXJY2YG3wQGSWPD0uvg0mwUJcuCZJU6usoKRx26HM1HWO0FRw3yDnfkOuP4bwi-P4qfLoc.Z2OxPA.dR8i3lpkOULIJvGYUZua5h85epw, Upgrade-Insecure-Requests: 1, Sec-Fetch-Dest: document, Sec-Fetch-Mode: navigate, Sec-Fetch-Site: same-origin, Sec-Fetch-User: ?1, Priority: u=0, i], body=
[2024-12-19 00:49:56,355] INFO    | access | log_response_info.42 | Response: status=200 OK, headers=[Content-Type: text/html; charset=utf-8, Content-Length: 1088]
[2024-12-19 00:49:56,376] INFO    | access | log_request_info.36 | Request: method=GET, url=http://127.0.0.1:5000/static/css/default.css, headers=[Host: 127.0.0.1:5000, User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:133.0) Gecko/20100101 Firefox/133.0, Accept: text/css,*/*;q=0.1, Accept-Language: ja,en-US;q=0.7,en;q=0.3, Accept-Encoding: gzip, deflate, br, zstd, Dnt: 1, Sec-Gpc: 1, Connection: keep-alive, Referer: http://127.0.0.1:5000/, Cookie: session=.eJwlzsENwzAIAMBdePcBBtuQZSJjg9pv0ryq7t5IXeB0H9jziPMJ2_u44gH7a8EGyk59SYRItqlOZs4SXBRXTNY6Wqc2q0nDtZhqRqpwiLXJY2YG3wQGSWPD0uvg0mwUJcuCZJU6usoKRx26HM1HWO0FRw3yDnfkOuP4bwi-P4qfLoc.Z2OxPA.dR8i3lpkOULIJvGYUZua5h85epw, Sec-Fetch-Dest: style, Sec-Fetch-Mode: no-cors, Sec-Fetch-Site: same-origin, If-Modified-Since: Wed, 04 Dec 2024 12:48:52 GMT, If-None-Match: "1733316532.6492927-644-2100238851", Priority: u=2], body=
[2024-12-19 00:49:56,376] INFO    | access | log_response_info.42 | Response: status=304 NOT MODIFIED, headers=[Content-Disposition: inline; filename=default.css, Content-Type: text/css; charset=utf-8, Content-Length: 644, Last-Modified: Wed, 04 Dec 2024 12:48:52 GMT, Cache-Control: no-cache, ETag: "1733316532.6492927-644-2100238851", Date: Thu, 18 Dec 2024 15:49:56 GMT]
```

### エラーログ

わざとエラーを発生させて出力。

`logs/2024/12/error_2024-12-19.log`

```
[2024-12-19 00:49:54,291] ERROR   | error | handle_exception.49 | Exception occurred: 404 Not Found: The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.
Traceback (most recent call last):
  File "/Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/lib/python3.8/site-packages/flask/app.py", line 880, in full_dispatch_request
    rv = self.dispatch_request()
  File "/Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/lib/python3.8/site-packages/flask/app.py", line 854, in dispatch_request
    self.raise_routing_exception(req)
  File "/Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/lib/python3.8/site-packages/flask/app.py", line 463, in raise_routing_exception
    raise request.routing_exception  # type: ignore[misc]
  File "/Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/lib/python3.8/site-packages/flask/ctx.py", line 362, in match_request
    result = self.url_adapter.match(return_rule=True)  # type: ignore
  File "/Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/lib/python3.8/site-packages/werkzeug/routing/map.py", line 629, in match
    raise NotFound() from None
werkzeug.exceptions.NotFound: 404 Not Found: The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.
```

## 参考資料

- [ログ処理（Logging） — Flask Documentation (2.2.x)](https://msiz07-flask-docs-ja.readthedocs.io/ja/latest/logging.html)
- [エキスパートPythonプログラミング 改訂4版](https://www.kadokawa.co.jp/product/302304004673/)
