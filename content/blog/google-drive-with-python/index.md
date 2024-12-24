---
title: 'Google Drive の操作関連の処理を追加する'
date: 2024-12-24
tags: ["ひとりアドベントカレンダー2024","Python","Google"]
author: "OKAZAKI Shogo"
---

## はじめに

「OKAZAKI Shogo のひとりアドベントカレンダー2024」の17日目です。 
ファイルのアップロード機能に、 Google Drive へのアップロード機能をつけていきます。

## サービスアカウント経由で Google ドライブにファイルをアップロードするための準備

ボーイスカウトでは、 Google Workspace のアカウントが配布されているので、そこに割り当てられている Google Drive を利用する。

普通のユーザーではなく、アプリケーションなどの人以外のシステムが使う IAM ユーザーである「サービスアカウント」を作成して利用する。

### Google Cloud Project を作成する

[Google 公式のドキュメント](https://developers.google.com/workspace/guides/create-project?hl=ja)に沿って、今回のアプリで利用するプロジェクトを作成する。

今回は、プロジェクト ID を `bshssa-member-system` としている。

### Google Drive の API を有効化

プロジェクトが作成できたら、左メニューの「すべてのプロダクト」を選択

![](./001.png)

「API とサービス」を選択する

![](./002.png)

検索欄から「Google Drive」と検索し、「Google Drive API」を選択

![](./003.png)

「有効にする」を押して、 API を有効化する。

![](./004.png)

### サービスアカウントを作成する

再度、「API とサービス」の画面に戻り、上の「認証情報を作成」>「サービスアカウント」を選択。

![](./005.png)

内容を適当に埋めていく。

![](./006.png)

「ロールを選択」から「基本」>「編集者」を選択

![](./007.png)

最後に「完了」を押してサービスアカウントの作成を完了する。

![](./008.png)

### 秘密鍵生成

再度、「API とサービス」の画面に戻り、「認証情報」を選択。先ほど作成したサービスアカウントを選択する。

![](./009.png)

「鍵」のタブを選択し、「キーを追加」を選択。

![](./010.png)

秘密鍵は JSON のものを選択して作成する。

![](./011.png)

JSON がダウンロードされたら、`client_email:` の後に記述されているメールアドレスをコピーしておく。

Google Drive の画面に行き、サービスアカウントで利用したいフォルダを選択し、「共有」を押す。

先ほど作成したメールアドレスを編集者として追加して共有する。

![](./012.png)
![](./013.png)
![](./014.png)

これで、 JSON に書かれた秘密鍵をリヨすいて、先ほど共有設定をしたフォルダを操作することができるようになる。

## Google Drive を Python で操作する

必要なライブラリを追加する。

```shell
poetry add google-api-python-client google-auth-httplib2 google-auth-oauthlib
```

### Google Drive を操作するためのコード

今回はアップロードするためのコードを実装する。

`app/util/google_drive.py`

```python
import logging
import os
from dataclasses import dataclass
from typing import List, Optional

from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from instance.config.dev import CREDENTIAL_SERVICE_ACCOUNT_PATH

error_logger = logging.getLogger("error")
app_logger = logging.getLogger("app")


@dataclass(frozen=True)
class MimeMapping:
    """ファイルの拡張子と MINE タイプを対応づけるデータクラス"""

    extension: str
    mine_type: str


# 対応可能なファイルタイプ一覧
# 参考資料: https://developer.mozilla.org/ja/docs/Web/HTTP/MIME_types/Common_types
MIME_MAPPINGS = [
    MimeMapping(".pdf", "application/pdf"),
    MimeMapping(".doc", "application/msword"),
    MimeMapping(".docx", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"),
    MimeMapping(".xls", "application/vnd.ms-excel"),
    MimeMapping(".xlsx", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"),
]

# Google Drive スコープ定数
GOOGLE_DRIVE_SCOPE_EDIT = ["https://www.googleapis.com/auth/drive"]
GOOGLE_DRIVE_SCOPE_READONLY = ["https://www.googleapis.com/auth/drive.readonly"]

# Google Drive のフォルダ ID
GOOGLE_DRIVE_FOLDER_ID = "1tmbiC2eiHxenEVF6X5dtfSDIQ8lxIGJ6"


def __get_mime_type(extension: str) -> Optional[str]:
    """特定の拡張子の MIME Type を取得する関数"""
    for mapping in MIME_MAPPINGS:
        if mapping.extension == extension:
            return mapping.mine_type
    # 未知の拡張子の場合のデフォルト
    return None


def __get_document_path(file_name: str) -> Optional[str]:
    """documentsフォルダ内の指定されたファイルの絶対パスを返す。

    :param file_name: 参照したいファイル名
    :return: ファイルの絶対パス
    """
    # このスクリプトの位置からプロジェクトのルートディレクトリを計算
    project_root = os.path.abspath(os.path.join(os.path.dirname(__file__), "../../"))
    documents_path = os.path.join(project_root, "documents", file_name)
    app_logger.debug(documents_path)

    # ファイルの存在を確認
    if not os.path.exists(documents_path):
        error_logger.error(f"'{file_name}' does not exist in 'documents'.")
        return None

    return documents_path


def __get_credentials(scopes: List[str]):
    """スコープを受け取り、Google のクレデンシャル情報を返却する"""
    return service_account.Credentials.from_service_account_file(CREDENTIAL_SERVICE_ACCOUNT_PATH, scopes=scopes)


def upload_file_to_gdrive(file_name: str) -> bool:
    """指定されたファイル名のファイルを Google Drive にアップロードする"""
    try:
        app_logger.debug(f"file_name:{file_name}")
        file_path = __get_document_path(file_name)
        app_logger.debug(f"file_path:{file_path}")

        _, extension = os.path.splitext(file_path)
        app_logger.debug(f"ext: {extension}")

        mine_type = __get_mime_type(extension)
        app_logger.debug(f"mine_type:{mine_type}")

        if file_path and mine_type:
            credentials = __get_credentials(GOOGLE_DRIVE_SCOPE_EDIT)
            drive_service = build("drive", "v3", credentials=credentials)
            media = MediaFileUpload(file_path, mimetype=mine_type, resumable=True)
            file_metadata = {"name": file_name, "mimeType": mine_type, "parents": [GOOGLE_DRIVE_FOLDER_ID]}
            # Google Driveへのアップロード処理
            file = drive_service.files().create(body=file_metadata, media_body=media).execute()
            app_logger.info(file)
            return True
        return False
    except Exception as e:
        error_logger.error(f"Error uploading file: {e}")
        return False
```

`upload_file_to_gdrive()` を Flask のコード内で用いれば、指定したファイルを Google Drive にアップすることができる。

## 参考資料

- [Google Cloud プロジェクトを作成する  |  Google Workspace  |  Google for Developers](https://developers.google.com/workspace/guides/create-project?hl=ja)
- [Python のクイックスタート  |  Google Drive  |  Google for Developers](https://developers.google.com/drive/api/quickstart/python?hl=ja)
- [GoogleDrive のファイル操作（アップロード／ダウンロード）を自動化する #Python - Qiita](https://qiita.com/saurus12/items/b4c851211d768a0f1212)
- [\[Python\] Googleドライブ上のファイルをダウンロードする｜こはた](https://note.com/kohaku935/n/nd7e984e8676c)
- [\[Python\] Googleドライブにファイルをアップロードする｜こはた](https://note.com/kohaku935/n/n99779e59561b)
- [PythonからGoogleDriveにファイルをアップロード #PyDrive2 - Qiita](https://qiita.com/sey323/items/875c0ab1585044772ab2)
