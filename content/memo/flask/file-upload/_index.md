---
title: ファイルアップロードとバリデーション
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: Flask でのファイル受け取り、secure_filename、拡張子制限、メッセージ表示
slug: /memo/flask/file-upload
---

## 概要

Web フォームから送信されたファイルの安全な受信・保存処理。ファイル名のサニタイズ（`secure_filename`）、拡張子バリデーション、アップロード容量制限についてのまとめ。

## 参考ブログ記事

- [Flask でファイルアップロード機能を実装&フォームにメッセージを表示する](/blog/flask-file-upload)

---

## 1. HTML フォーム側の設定

ファイルをアップロードする場合、form タグに `enctype="multipart/form-data"` を必ず指定する。

```html
<form method="POST" enctype="multipart/form-data">
    <input type="file" name="upload_file">
    <button type="submit">アップロード</button>
</form>
```

---

## 2. サーバー側の安全な保存処理

### 拡張子チェックとファイル名サニタイズ
```python
import os
from flask import request, flash, redirect, current_app
from werkzeug.utils import secure_filename

ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg", "gif", "pdf"}

def allowed_file(filename):
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS

@bp.route("/upload", methods=["POST"])
def upload():
    if "upload_file" not in request.files:
        flash("ファイルが選択されていません。")
        return redirect(request.url)

    file = request.files["upload_file"]

    if file.filename == "":
        flash("ファイル名が空です。")
        return redirect(request.url)

    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        save_path = os.path.join(current_app.config["UPLOAD_FOLDER"], filename)
        file.save(save_path)
        flash("ファイルのアップロードに成功しました。")
        return redirect(request.url)

    flash("許可されていないファイル形式です。")
    return redirect(request.url)
```

---

## 3. セキュリティ上の考慮点
- **ファイル名の上書き・ディレクトリトラバーサル防止**: 必ず `werkzeug.utils.secure_filename` を通す。
- **最大容量の制限**: `MAX_CONTENT_LENGTH` を設定して巨大ファイルの POST による DoS を防ぐ（例: `app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024`）。
