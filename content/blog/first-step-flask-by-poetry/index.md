+++
title = 'Poetry を使って Python のプロジェクトを作成する'
date = 2024-12-03T21:56:40+09:00
tags = ['Python', 'Poetry','ひとりアドベントカレンダー2024']
draft = true
+++

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の2日目です。
昨日に引き続いて、Flask で Web アプリを作っていくのを進めます。まずは、モダンなパッケージ依存関係管理ツールの Poetry の使い方を確認します。

## Poetry とは

[Poetry](https://python-poetry.org/) は Python の依存パッケージと仮想環境管理を行うためのツール。
mac の場合は、 brew 経由でインストール可能：

```shell
brew install poetry
```

## Poetry を使ってプロジェクトを作成する

以下のコマンドをプロジェクト作成前に実行し、 Poetry プロジェクト内に `.venv` がつくられるようにする。

```shell
poetry config virtualenvs.in-project true --local
```

すでにプロジェクトのフォルダを作成している場合は以下のコマンドを実行し、 Poetry プロジェクトを開始する。

```shell
poetry init
```

以下のように質問されるが大体デフォルトでOK。終わると、 `pyproject.toml` だけが作成される。

```shell
$ poetry init

This command will guide you through creating your pyproject.toml config.

Package name [flask-sample]:
Version [0.1.0]:
Description []:
Author [None, n to skip]: OKAZAKI Shogo
License []:
Compatible Python versions [^3.13]: 

Would you like to define your main dependencies interactively? (yes/no) [yes]
You can specify a package in the following forms:
  - A single name (requests): this will search for matches on PyPI
  - A name and a constraint (requests@^2.23.0)
  - A git url (git+https://github.com/python-poetry/poetry.git)
  - A git url with a revision (git+https://github.com/python-poetry/poetry.git#develop)
  - A file path (../my-package/my-package.whl)
  - A directory (../my-package/)
  - A url (https://example.com/packages/my-package-0.1.0.tar.gz)

Package to add or search for (leave blank to skip):

Would you like to define your development dependencies interactively? (yes/no) [yes]
Package to add or search for (leave blank to skip):

Generated file

[tool.poetry]
name = "sample-project"
version = "0.1.0"
description = ""
authors = ["OKAZAKI Shogo"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.13"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"


Do you confirm generation? (yes/no) [yes]
```

フォルダごと作成したい場合は以下のコマンド。

```shell
poetry new <project-name>
```

`poetry new` を実行すると、以下のようなフォルダ構造が作られる。

```
$ poetry new sample-project
$ cd sample-project/
$ tree
.
├── README.rst
├── pyproject.toml
├── sample_project
│   └── __init__.py
└── tests
    ├── __init__.py
    └── test_sample_project.py
```

`pyproject.toml` は Python 標準の設定ファイル（[PEP518](https://peps.python.org/pep-0518/)）
中身はこんな感じ：

```toml
[tool.poetry]
name = "sample-project"
version = "0.1.0"
description = ""
authors = ["OKAZAKI Shogo"]
readme = "README.md"

[tool.poetry.dependencies]
python = "^3.13"


[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

- `[tool.poetry]`: プロジェクトの基本的なメタデータの設定
- `[tool.poetry.dependencies]`: プロジェクトの依存パッケージのリスト
- `[build-system]`: Poetry をプロジェクト管理に使うビルドシステムとして指定

## Poetry にライブラリを追加

`poetry add` コマンドで必要なライブラリを追加できる[^1]：

```shell
poetry add flask
```

## Poetry 環境を構築

以下のコマンドで、実際に環境を構築する。

- プロジェクトルート直下に `poetry.lock` が作成される[^2]
    - すでに存在する場合は、`poetry.lock` に書かれたバージョンを利用してパッケージがインストールされる
- `virtualenvs.in-project true` が設定できている場合はプロジェクトルート直下に `.venv` フォルダが作成される
    - `.venv` フォルダが作られていない場合は既に virtualenv が作られている場合があるので、`poetry env remove <env-name>` で一度削除して再度環境を作り直す

```shell
poetry install
```

## Poetry 環境での実行

以下のコマンドで Poetry 環境でコマンドを実行できる[^3]。

```shell
poetry run <コマンド>
```

## Poetry 環境に入る

以下のコマンドで poetry 環境に入ることができる[^4]。

```shell
poetry shell
```

## 参考資料

- [エキスパートPythonプログラミング 改訂4版](https://www.kadokawa.co.jp/product/302304004673/)
- [poetry を使って python の project を作る](https://info.drobe.co.jp/blog/engineering/poetry-python-project)

[^1]: pipenv だったら `pipenv install` コマンドに相当
[^2]: ロックファイル自体は `poetry lock` コマンドでも作成可能
[^3]: pipenv だったら `pipenv run` コマンドに相当
[^4]: pipenv だったら `pipenv shell` コマンドに相当