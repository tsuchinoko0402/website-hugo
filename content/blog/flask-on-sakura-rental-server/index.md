---
title: 'Flask アプリをさくらのレンタルサーバで動かす'
date: 2024-12-05T23:09:15+09:00
tags: ["ひとりアドベントカレンダー2024","Python","Flask", "cgi"]
author: "OKAZAKI Shogo"
toc: true
---

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の4日目です。 
今回のインフラの条件である、[さくらのレンタルサーバ](https://rs.sakura.ad.jp/)に Flask のアプリをおいて動かすことを試みます。

## 今回の契約形態

- さくらのレンタルサーバ　スタンダートプラン
    - sudo とかが使えなかったりして一部制限がある
    - Python 3.8.12 が標準でインストールされている（2024年12月現在）
        - pyenv 入れて別のバージョンを動かす、みたいなことは試みたけど出来なかった
        - pip install はできる
    - デプロイ方法は FTP クライアントを使って直接アプリを所定の場所に置く

## Flask アプリを動かすまでの準備

レンタルサーバーに SSH で接続。デフォルトで csh なので bash に変更する。

```shell
% chsh -s /usr/local/bin/bash
Password:
chsh: user information updated
```

一旦接続をやめて再度接続すると bash になっている。為念 Python のバージョンを確認する。

```shell
$ python -V
Python 3.8.12
$ which python
/usr/local/bin/python
```

pip を使うためのインストーラをダウンロードして、実行する。

```shell
$ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
$ python get-pip.py
Defaulting to user installation because normal site-packages is not writeable
Collecting pip
  Using cached pip-24.3.1-py3-none-any.whl.metadata (3.7 kB)
Using cached pip-24.3.1-py3-none-any.whl (1.8 MB)
Installing collected packages: pip
  Attempting uninstall: pip
    Found existing installation: pip 24.3.1
    Uninstalling pip-24.3.1:
      Successfully uninstalled pip-24.3.1
  WARNING: The scripts pip, pip3 and pip3.8 are installed in '/home/<アカウント名>/.local/bin' which is not on PATH.
  Consider adding this directory to PATH or, if you prefer to suppress this warning, use --no-warn-script-location.
Successfully installed pip-24.3.1
```

`/home/<アカウント名>/.local/bin` にパスが通ってないので通す。

```shell
$ echo 'export PATH=~/.local/bin/:$PATH' >> .bash_profile
```

Flask をインストールする。

```shell
$ pip install flask
$ flask --version
Python 3.8.12
Flask 3.0.3
Werkzeug 3.0.6
```

## Flask アプリを cgi で動かす

src ディレクトリ配下に以下のものを作成する

`index.cgi`
```python
#!/usr/local/bin/python
# -*- coding: utf-8 -*-

import cgitb
cgitb.enable()


from wsgiref.handlers import CGIHandler
from app import app


class ProxyFix(object):
  def __init__(self, app):
      self.app = app
      
  def __call__(self, environ, start_response):
      return self.app(environ, start_response)


if __name__ == '__main__':
   app.wsgi_app = ProxyFix(app.wsgi_app)
   CGIHandler().run(app)
```

`.htaccess`
```
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ /index.cgi/$1 [QSA,L]
<Files ~ "\.py$">
  deny from all
</Files>
```

src ディレクトリ以下のものをサーバーのルートにアップし、`index.cgi` と `.htaccess` のパーミッションを 755 にする。

```shell
$ ls -la
total 28
drwx---r-x  4 <アカウント名>  users  512 Dec  4 23:35 .
drwx---r-x  5 <アカウント名>  users  512 Dec  4 20:48 ..
-rw----r--  1 <アカウント名>  users  136 Dec  4 23:35 .htaccess
-rw----r--  1 <アカウント名>  users  188 Dec  4 23:34 app.py
-rw----r--  1 <アカウント名>  users  419 Dec  4 23:34 index.cgi
drwx---r-x  4 <アカウント名>  users  512 Dec  4 23:34 static
drwx---r-x  2 <アカウント名>  users  512 Dec  4 23:34 templates
$ chmod 755 .htaccess index.cgi
$ ls -la
total 28
drwx---r-x  4 <アカウント名>  users  512 Dec  4 23:35 .
drwx---r-x  5 <アカウント名>  users  512 Dec  4 20:48 ..
-rwxr-xr-x  1 <アカウント名>  users  136 Dec  4 23:35 .htaccess
-rw----r--  1 <アカウント名>  users  188 Dec  4 23:34 app.py
-rwxr-xr-x  1 <アカウント名>  users  419 Dec  4 23:34 index.cgi
drwx---r-x  4 <アカウント名>  users  512 Dec  4 23:34 static
drwx---r-x  2 <アカウント名>  users  512 Dec  4 23:34 templates
```

これでルートにアクセスして、ローカルで表示されたものと同じものが表示されれば成功。

## 参考資料

- [【Python】さくらインターネットの共有サーバーでFlaskを使ったWEBアプリケーションデプロイ #さくらのレンタルサーバ - Qiita](https://qiita.com/renren5977/items/b2dd5c8f89579d8efe3a)
- [Flaskで作ったサービスをさくらのWebサーバーで公開 | おさしみわかめ](https://peraimaru.work/flask%E3%81%A7%E4%BD%9C%E3%81%A3%E3%81%9F%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%82%92%E3%81%95%E3%81%8F%E3%82%89%E3%81%AEweb%E3%82%B5%E3%83%BC%E3%83%90%E3%83%BC%E3%81%A7%E5%85%AC%E9%96%8B/)
- [Flask + Flask-SQLAlchemy + さくらレンタルサーバーでnoteと連携させたブログを作った方法｜東京大学柔道部](https://note.com/todaijudobu/n/nc3ecdb4c1af8)
- [dayjournal | Try #019 – さくらのレンタルサーバでFlaskを利用した住所検索APIを構築してみた](https://www.memo.dayjournal.dev/memo/try-019/)
