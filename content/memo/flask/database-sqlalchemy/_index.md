---
title: Flask-SQLAlchemy によるデータベース操作
publishedDate: "2026-09-29"
updatedDate: "2026-09-29"
description: モデル定義、SELECT・INSERT・UPDATE・DELETE 処理、トランザクション管理
slug: /memo/flask/database-sqlalchemy
---

## 概要

Flask-SQLAlchemy を用いたリレーショナルデータベースへの接続設定、ORM モデルの定義、CRUD 操作、トランザクションコミット・ロールバックの基本的な実装パターンについてのまとめ。

## 参考ブログ記事

- [Flask から DB に接続する（とりあえず、SELECT まで）](/blog/flask-db-select)
- [Flask で DB のレコードを追加・編集する機能を実装する](/blog/flask-insert-update)

---

## 1. セットアップとモデル定義

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    created_at = db.Column(db.DateTime, default=db.func.current_timestamp())
```

---

## 2. CRUD 操作

### SELECT (参照)
```python
# 全件取得
users = User.query.all()

# 主キー検索
user = User.query.get(user_id)

# 条件絞り込み
user = User.query.filter_by(username="admin").first()
```

### INSERT (追加)
```python
new_user = User(username="alice", email="alice@example.com")
db.session.add(new_user)
db.session.commit()
```

### UPDATE (更新)
```python
user = User.query.get(user_id)
user.email = "new_email@example.com"
db.session.commit()
```

### DELETE (削除)
```python
user = User.query.get(user_id)
db.session.delete(user)
db.session.commit()
```

---

## 3. トランザクション管理とエラーハンドリング

例外発生時には必ず `db.session.rollback()` を行い、セッションの不整合を防ぐ。

```python
try:
    # 複数テーブルへの更新処理など
    db.session.commit()
except Exception as e:
    db.session.rollback()
    raise e
```
