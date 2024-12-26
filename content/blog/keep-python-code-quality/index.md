---
title: 'Python コードの品質を保つためのツールの導入（Ruff, pytest-cov, lizard）'
date: 2024-12-25T07:00:13+09:00
tags: ["ひとりアドベントカレンダー2024","Python", "Ruff", "lizard"]
author: "OKAZAKI Shogo"
---
## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の18日目です。ついに最終日です。
作ろうとしていたアプリの完成はまだですが、コードが肥大化して読めなくなってしまう前に、コードの品質を保つための仕組みを入れていきます。

## 静的コード解析ツール Ruff を導入する。

Ruff は 2022 年 8 月にリリースされた Python のリンター兼フォーマッターである。これまでよく利用されてきた
- リンター：Flake8
- フォーマッター：Black
- import のソート：isort
の 3 つのツールの機能をすべて実行することができる。

今回も poetry 経由で導入する：

```shell
poetry add --group=dev ruff
```

### 基本的な使い方

フォーマッターを実行するには `ruff format [ディレクトリ名]` を実行する。

```shell
poetry run ruff format .
```

リンターの実行には `ruff check [ディレクトリ名]` を実行する。以下は例：

```shell
poetry run ruff check --extend-select I --fix .
```

- `--extend-select I`：Ruff が検出する問題の種類を拡張するオプション。`I` は不要なインポートの削除や、インポート順序の整理を行ってくれる。
- `--fix`：検出された問題を自動的に修正する。

リンターを実行すると以下のようにルール違反している箇所とその理由が出力される：

```
app/__init__.py:12:5: D103 Missing docstring in public function
   |
12 | def create_app(test_config=None):
   |     ^^^^^^^^^^ D103
13 |     # appの設定
14 |     app = Flask(__name__, instance_relative_config=True)
   |
```

GitHub Action で CI を走らせてチェックする場合は以下のように設定すれば良い。

```yaml
- name: Run Ruff (lint)
  run: ruff check --output-format=github .
- name: Run Ruff (format)
  run: ruff format . --check --diff
```

- `--output-format=github`: 出力を GitHub Actions に適した形式で表示する。この形式にすることで、解析結果が GitHub のプルリクエストやチェックページにわかりやすく表示される。
- `--check`: 実際にフォーマットを適用せず、フォーマットが必要かどうかだけを確認する。
- `--diff`: フォーマットが必要な箇所がある場合、その差分を表示する。


### Ruff の設定

`pyproject.toml` に以下のように記載する。リンターの設定とフォーマッターの設定は分けて記述する：

```toml
[tool.ruff]
# Ruff の全般的な設定。Ruff の動作全体に影響を与える基本的なオプションを指定する。

[tool.ruff.lint]
# Ruff を静的解析（Lint）ツールとして利用する際の設定。静的解析のルールやチェックの対象範囲を詳細に指定する。

[tool.ruff.format]
# Ruff をコードフォーマッターとして利用する際の設定。コード整形に関連する設定を指定する。

[tool.ruff.lint.isort]
# Ruff のインポート整理機能（isort互換）に関する設定。インポートの並び順やグループ化に関連する設定を指定する。
```

[公式ドキュメントの説明](https://docs.astral.sh/ruff/configuration/) も参考にしながら、以下のような設定が可能である。


```toml
[tool.ruff]
# Ruff の設定
# 参考: https://docs.astral.sh/ruff/configuration/
src = ["src", "test"]
# よく無視されるディレクトリを除外する。
exclude = [
    ".bzr",
    ".direnv",
    ".eggs",
    ".git",
    ".git-rewrite",
    ".hg",
    ".ipynb_checkpoints",
    ".mypy_cache",
    ".nox",
    ".pants.d",
    ".pyenv",
    ".pytest_cache",
    ".pytype",
    ".ruff_cache",
    ".svn",
    ".tox",
    ".venv",
    ".vscode",
    "__pypackages__",
    "_build",
    "buck-out",
    "build",
    "dist",
    "node_modules",
    "site-packages",
    "venv",
]

# Python 3.8 を想定。
target-version = "py38"
# Black と同じ設定。
line-length = 88
indent-width = 4

[tool.ruff.lint]
# アンダースコアで始まる未使用変数を許可する。
dummy-variable-rgx = "^(_+|(_+[a-zA-Z0-9_]*[a-zA-Z0-9]+?))$"

# デフォルトで Pyflakes (`F`) と pycodestyle (`E`) の一部のコードを有効にする。
# Ruff は Flake8 と異なり、デフォルトでは pycodestyle の警告 (`W`) や
# McCabe の複雑性チェック (`C901`) を有効にしない。
select = ["E4", "E7", "E9", "F"]
ignore = []

# すべての有効なルールに対して修正を許可する（`--fix` が指定された場合）。
fixable = ["ALL"]
unfixable = []

[tool.ruff.format]
# Black のように、文字列に二重引用符を使用する。
quote-style = "double"

# Black のように、タブではなくスペースでインデントする。
indent-style = "space"

# Black のように、マジックトレーリングカンマを尊重する。
skip-magic-trailing-comma = false

# Black のように、適切な改行コードを自動検出する。
line-ending = "auto"

# Docstring 内のコード例を自動フォーマットする。Markdown、reStructuredText のコード/リテラルブロック、および doctest に対応している。
#
# 現在はデフォルトで無効になっているが、将来的にはオプトアウト形式になる予定である。
docstring-code-format = true

# Docstring 内のコードスニペットをフォーマットする際に使用される行の長さの制限を設定する。
#
# この設定は `docstring-code-format` が有効になっている場合にのみ影響を与える。
docstring-code-line-length = "dynamic"
```

## テストのカバレッジを集計する

Pytest で実行したテストのカバレッジを測定できるようにする：

```shell
poetry add --group dev pytest-cov
```

以下のようなコマンドを実行すると下記の通り出力が得られる：

```
$ poetry run pytest -v --cov=app --cov-report=term-missing
The currently activated Python version 3.13.1 is not supported by the project (3.8.12).
Trying to find and use a compatible version.
Using python3 (3.8.12)
=============================================================================== test session starts ================================================================================
platform darwin -- Python 3.8.12, pytest-8.3.4, pluggy-1.5.0 -- /Users/shogo/Library/Caches/pypoetry/virtualenvs/bshssa-member-system-GhAlEoqL-py3.8/bin/python
cachedir: .pytest_cache
rootdir: /Users/shogo/develop/bshssa-member-system
configfile: pytest.ini
plugins: cov-5.0.0
collected 16 items

tests/models/test_file.py::test_FileモデルがDBに作成されること PASSED                                                                                                        [  6%]
tests/models/test_file.py::test_FileモデルがDBで更新されること PASSED                                                                                                        [ 12%]
tests/models/test_file.py::test_FileモデルがDBから削除されること PASSED                                                                                                      [ 18%]
tests/models/test_manager_user.py::test_ManagerUserがDBに登録されること PASSED                                                                                               [ 25%]
tests/models/test_manager_user.py::test_ManagerUserがのパスワードの設定と検証が PASSED                                                                                       [ 31%]
tests/models/test_manager_user.py::test_manager_user_update PASSED                                                                                                           [ 37%]
tests/models/test_manager_user.py::test_manager_user_delete PASSED                                                                                                           [ 43%]
tests/models/test_manager_user.py::test_load_user PASSED                                                                                                                     [ 50%]
tests/views/test_files.py::test_ファイル一覧に登録されたファイルが表示されること PASSED                                                                                      [ 56%]
tests/views/test_index.py::test_indexにアクセスしてトップページが返されること PASSED                                                                                         [ 62%]
tests/views/test_login.py::test_GETリクエストを送るとログイン画面が返却されること PASSED                                                                                     [ 68%]
tests/views/test_login.py::test_POSTリクエストでユーザー名とパスワードを受け取りログインが成功すること PASSED                                                                [ 75%]
tests/views/test_login.py::test_POSTリクエストで無効なユーザー名を受け取るとログインが失敗すること PASSED                                                                    [ 81%]
tests/views/test_login.py::test_POSTリクエストで無効なパスワードを受け取るとログインが失敗すること PASSED                                                                    [ 87%]
tests/views/test_manage_menu.py::test_ログインせずに保護されたエンドポイントにアクセスすると401エラー PASSED                                                                 [ 93%]
tests/views/test_manage_menu.py::test_ログイン後に保護されたエンドポイントにアクセスすると管理者メニューに遷移すること PASSED                                                [100%]

---------- coverage: platform darwin, python 3.8.12-final-0 ----------
Name                            Stmts   Miss  Cover   Missing
-------------------------------------------------------------
app/__init__.py                    35      0   100%
app/app.py                         28     28     0%   1-51
app/models/__init__.py              0      0   100%
app/models/file.py                 16      0   100%
app/models/manager_user.py         23      0   100%
app/util/google_drive.py           54     32    41%   48-52, 62-71, 76, 83-114
app/views/__init__.py               0      0   100%
app/views/files.py                  7      0   100%
app/views/index.py                  8      0   100%
app/views/login.py                 16      0   100%
app/views/logout.py                 7      2    71%   9-10
app/views/manage_file.py           10      3    70%   12-14
app/views/manage_file_list.py       9      2    78%   14-15
app/views/manage_menu.py            7      0   100%
app/views/regist_file.py           59     41    31%   22-28, 32-44, 50-99
app/views/update_file.py           23     13    43%   19-39
-------------------------------------------------------------
TOTAL                             302    121    60%


================================================================================ 16 passed in 1.50s ================================================================================
```

- `-v`: テスト実行時に詳細な出力を表示する（verbose モード）
- `--cov=app`: `pytest-cov` プラグインを使用し、`app` ディレクトリのコードカバレッジを測定する
- `--cov-report=term-missing`: テストがカバーしていない行（未実行行）を表示する
    - 上記の `Missing` の列がそれに当たる
    - `--cov-report=html` とすると、カバレッジレポートを HTML ファイルとして生成し、`htmlcov` ディレクトリに保存することができる


## コードの複雑度を測る

コードの品質と複雑性を測定し、以下を特定するために、[lizard](https://pypi.org/project/lizard/) を入れる。複雑度を測定すると、以下のことに役立つ：
- 問題のある関数やファイル: 高いサイクロマティック複雑性（CCN）やコメントを除いたコードの行数（NLOC） を持つコードはリファクタリングの対象になる可能性がある。
    - サイクロマティック複雑性については [Microsoft のドキュメント](https://learn.microsoft.com/ja-jp/visualstudio/code-quality/code-metrics-cyclomatic-complexity?view=vs-2022) の説明がわかりやすい
- テストやレビューの優先度: 特に複雑な関数に重点を置くことで、バグのリスクを減らせる。
- コード全体の健康状態: 平均的な複雑性や行数の指標から、プロジェクト全体のコード品質を判断できる。

```shell
poetry add --group=dev lizard
```

`lizard` を実行すると以下のような出力が得られる：

```
$ poetry run lizard app
The currently activated Python version 3.13.1 is not supported by the project (3.8.12).
Trying to find and use a compatible version.
Using python3 (3.8.12)
================================================
  NLOC    CCN   token  PARAM  length  location
------------------------------------------------
       6      3     31      1       7 __get_mime_type@46-52@app/util/google_drive.py
      13      2     81      1      17 __get_document_path@55-71@app/util/google_drive.py
       4      1     24      1       5 __get_credentials@74-78@app/util/google_drive.py
      30      4    167      1      34 upload_file_to_gdrive@81-114@app/util/google_drive.py
       2      1     12      1       2 get_id@16-17@app/models/manager_user.py
       3      1     19      2       3 set_password@19-21@app/models/manager_user.py
       4      1     26      2       4 check_password@23-26@app/models/manager_user.py
       3      1     30      1       3 load_user@30-32@app/models/manager_user.py
       3      1     26      0       3 index@13-15@app/views/manage_file_list.py
       3      1     26      0       3 index@9-11@app/views/files.py
       3      1     15      0       3 index@8-10@app/views/logout.py
       3      1     19      0       3 index@11-13@app/views/index.py
      14      2     99      0      23 index@17-39@app/views/update_file.py
       2      1     13      0       2 index@9-10@app/views/manage_menu.py
       6      1     52      0       6 index@11-16@app/views/manage_file.py
       6      3     45      1       9 __get_file_extension_if_allowed@20-28@app/views/regist_file.py
      10      3     53      1      14 __get_file_size@31-44@app/views/regist_file.py
      42      6    231      0      51 index@49-99@app/views/regist_file.py
      11      4     93      0      12 index@10-21@app/views/login.py
      28      2    197      1      43 create_app@12-54@app/__init__.py
       3      1     17      1       3 handle_over_max_file_size@16-18@app/app.py
       2      2     23      1       2 headers_to_string@21-22@app/app.py
       7      1     20      0       7 log_request_info@31-37@app/app.py
       4      1     17      1       4 log_response_info@41-44@app/app.py
       3      1     20      1       3 handle_exception@49-51@app/app.py
15 file analyzed.
==============================================================
NLOC    Avg.NLOC  AvgCCN  Avg.token  function_cnt    file
--------------------------------------------------------------
     83      13.2     2.5       75.8         4     app/util/google_drive.py
      0       0.0     0.0        0.0         0     app/models/__init__.py
     23       3.0     1.0       21.8         4     app/models/manager_user.py
     16       0.0     0.0        0.0         0     app/models/file.py
     11       3.0     1.0       26.0         1     app/views/manage_file_list.py
      7       3.0     1.0       26.0         1     app/views/files.py
      7       3.0     1.0       15.0         1     app/views/logout.py
      8       3.0     1.0       19.0         1     app/views/index.py
     23      14.0     2.0       99.0         1     app/views/update_file.py
      7       2.0     1.0       13.0         1     app/views/manage_menu.py
     12       6.0     1.0       52.0         1     app/views/manage_file.py
     73      19.3     4.0      109.7         3     app/views/regist_file.py
     16      11.0     4.0       93.0         1     app/views/login.py
     35      28.0     2.0      197.0         1     app/__init__.py
     32       3.8     1.2       19.4         5     app/app.py

===============================================================================================================
No thresholds exceeded (cyclomatic_complexity > 15 or length > 1000 or nloc > 1000000 or parameter_count > 100)
==========================================================================================
Total nloc   Avg.NLOC  AvgCCN  Avg.token   Fun Cnt  Warning cnt   Fun Rt   nloc Rt
------------------------------------------------------------------------------------------
       353       8.6     1.8       54.2       25            0      0.00    0.00
```

前半部分：関数ごとのメトリクス
- `NLOC` (Non-Commenting Lines of Code): コメントを除いたコードの行数。
- `CCN` (Cyclomatic Complexity Number): サイクロマティック複雑性。コードの分岐（条件分岐やループ）の数を示し、値が高いほど複雑な関数である。
- `PARAM`: 関数が受け取るパラメータの数。
- `Length`: 関数全体の行数（空行やコメントを含む）。
- `Location`: 関数の名前と、その位置情報（行番号とファイルパス）。

後半部分：ファイルごとのメトリクス
- `NLOC`: ファイル全体のコメントを除いたコードの行数。
- `Avg.NLOC`: 関数ごとの平均 NLOC。
- `AvgCCN`: 関数ごとの平均 CCN。
- `Avg.token`: 関数ごとの平均トークン数。
    - トークンとは、プログラムコードを構成する最小単位であり、以下のような要素を含む：
        - キーワード（例: `if`, `for`, `def` など）
        - 識別子（変数名や関数名）
        - リテラル（例: 数値や文字列リテラル）
        - 演算子（例: `+`, `-`, `=` など）
        - 区切り記号（例: `,`, `;`, `()` など）
    - 平均トークン数は、（ファイル内の全トークン数）÷（ファイル内の関数の数） で定義される。
    - 値が大きい場合、関数が多くの処理を詰め込みすぎている可能性があり、リファクタリングの候補となる。
- `function_cnt`: ファイル内の関数の数。
- `file`: ファイル名。

上記の出力結果では、特に `app/views/regist_file.py` の `index` 関数（CCN 6, 長さ 51 行）がやや複雑であり、リファクタリングの候補となる可能性が示唆されている。実際、この関数が含まれている `app/views/regist_file.py` 自体も `AvgCCN` や `Avg.token` が高めであることがわかる。

## 参考資料

- [Ruff](https://docs.astral.sh/ruff/)
- [新しい静的コード解析ツール「Ruff」をご紹介 | gihyo.jp](https://gihyo.jp/article/2023/03/monthly-python-2303)
- [Pythonの Ruff (linter) でコード整形もできるようになりました #flake8 - Qiita](https://qiita.com/ciscorn/items/bf78b7ad8e0e332f891b)
- [pytestのすぐに使えるカバレッジ計測 #Python - Qiita](https://qiita.com/kg1/items/e2fc65e4189faf50bfe6)
- [サイクロマティック複雑度の計測ツール「lizard」のセットアップ&使い方 #Python - Qiita](https://qiita.com/uhooi/items/a1a96a2d7f5e081e2049)
- [lizardを使ってCCNというコード品質の指標を学ぶ](https://zenn.dev/koya6565/articles/20230330_lizard-ccn)
