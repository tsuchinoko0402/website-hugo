---
title: Rust の ORM：Diesel を使ってみる
date: 2021-12-25
slug: rust-diesel-sample
tags:
  - Rust
  - Diesel
  - PostgreSQL
---

Rust での DB を扱いを楽にする [Diesel](https://diesel.rs/) を利用してみます。他にも ORM はありそうなのですが、これが一番メジャーっぽいです。  
  
今回は、Postgres の DB と連携させることを目標とします。  

## サンプルプログラム

DB の立ち上げを Docker でできるようにしておきました：  
[GitHub のリポジトリ](https://github.com/tsuchinoko0402/diesel_sample)  

## 準備

### 依存関係

`cargo new` で適当なプロジェクトを作成し、 `Cargo.toml` には以下のように記述しておきます：

```
[dependencies]
diesel = { version = "1.4.4", features = ["postgres", "chrono"] }
dotenv = "0.15.0"
chrono = "0.4"
```

  
`diesel` の `features` には2つ指定しています：

-   `postgres` : PostgreSQL の DB を扱うため
-   `chrono` : 日付型のデータを扱うため

  

### Diesel CLI のインストール

Diesel には [CLI ツール](https://github.com/diesel-rs/diesel/tree/master/diesel_cli) が用意されているため、インストールします。 Rust のプロジェクトルートで以下のコマンドを実行します：

```
cargo install diesel_cli --no-default-features --features postgres
```

ここで、もし以下のようなエラーが出た場合：

```
note: ld: library not found for -libpq

clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

[libpq](https://www.postgresql.org/docs/current/libpq.html) のライブラリがインストールされていませんので、各 OS の環境に合わせてインストールします。  

### Diesel のセットアップ

Diesel のセットアップをする前に DB を立ち上げておきます。今回のプロジェクトの場合、プロジェクトルートで

```
docker-compose up -d db
```

を実行すると PostgreSQL の DB が立ち上がります。[DBeaver](https://dbeaver.io/) などのツールで DB を閲覧できるようになります。

![](20230212104543.png)

環境変数 `DATABASE_URL` は `postgres://admin:password@localhost:5432/postgres` です。これを Rust のプロジェクトルートの `.env` に書き込んでおきます：

```
echo DATABASE_URL=postgres://admin:password@localhost:5432/postgres > .env
```

次に、以下のコマンドでセットアップができます：

```
diesel setup
```

  

### マイグレーション

以下のコマンドを実行します：

```
diesel migration generate create_table
```

すると、以下のようにマイグレーション用の SQL ファイルが作成されます：

```
$ diesel migration generate create_table
Creating migrations/2021-12-24-150710_create_table/up.sql
Creating migrations/2021-12-24-150710_create_table/down.sql
```

`up.sql` を以下の内容にします：

-   `id` は `SERIAL` を指定し、自動で採番されるようにします。

```sql
CREATE TABLE posts (
   id SERIAL PRIMARY KEY,
   title TEXT NOT NULL,
   body TEXT NOT NULL,
   is_published BOOLEAN NOT NULL DEFAULT false,
   create_time TIMESTAMP NOT NULL DEFAULT current_timestamp,
   update_time TIMESTAMP NOT NULL DEFAULT current_timestamp
);
```

また、`down.sql` を以下のようにします：

```
DROP TABLE posts;
```

この状態で以下のコマンド

```
diesel migration run
```

を実行すると、`up.sql` の内容が実行され、DB にテーブルが作成されます。  

![](20230212104609.png)

また、`src/schema.rs` が以下の内容で作成されます。

```rust
table! {
   posts (id) {
       id -> Int4,
       title -> Text,
       body -> Text,
       is_published -> Bool,
       create_time -> Timestamp,
       update_time -> Timestamp,
   }
}
```

マイグレーションをロールバックしたい場合は以下のコマンドを実行します：

```
diesel mifration revert
```

このとき、 `down.sql` が実行され、テーブルも無事に消え、`schema.rs` の内容も消えます。  

## CRUD 操作

Rust で CRUD 操作を記述します。

### DB の接続

まずは、 DB とのコネクションを確立するためのメソッドを以下の `src/utils.rs` のように記述します：

```rust
use diesel::prelude::*;
use diesel::pg::PgConnection;
use dotenv::dotenv;
use std::env;

pub fn establish_connection() -> PgConnection {
   dotenv().ok();
   let database_url = env::var("DATABASE_URL")
       .expect("DATABASE_URL must be set");
   PgConnection::establish(&database_url)
       .expect(&format!("Error connecting to {}", database_url))
}
```

### モデルの定義

モデルの構造体を以下のように定義します（`models.rs` ）。

-   `SELECT` 文で取得してきたものを入れる構造体には `Queryable` を付与します。
-   `UPDATE` 文でテーブルに要素を追加するものには `Insertable` を付与します。
    -   `table_name="posts"` も指定します。

```
use chrono::{DateTime, Utc};
use crate::schema::posts;

#[derive(Debug, Queryable)]
pub struct Post {
   pub id: i32,
   pub title: String,
   pub body: String,
   pub is_published: bool,
   pub create_time: DateTime<Utc>,
   pub update_time: DateTime<Utc>,
}

#[derive(Insertable)]
#[table_name="posts"]
pub struct NewPost<'a> {
   pub title: &'a str,
   pub body: &'a str,
}
```

### 単一の要素を登録する

登録するときは以下のように書きます。

```rust
pub fn create_post<'a>(conn: &PgConnection, id: i32, title: &'a str, body: &'a str) -> Post {
   use crate::schema::posts;
   let new_post = NewPost {
       id,
       title,
       body,
   };

   diesel::insert_into(posts::table)
       .values(&new_post)
       .get_result(conn)
       .expect("Error saving new post")
}
```

![](20230212104631.png)

### 要素を取得する

こんな感じで書けます。`filter` を使用して条件の絞り込み、 `limit` で取得上限数を指定します。

```rust
pub fn show_posts(conn: &PgConnection) -> Vec<Post> {
   use crate::schema::posts::dsl::*;
   posts.filter(is_published.eq(true))
       .limit(5)
       .load::<Post>(conn)
       .expect("Error loading posts")
}
```

### 要素を更新する

`update` を用いて更新します。引数に更新対象を指定し、`set` で更新内容を記述します。

```rust
pub fn publish_post(conn: &PgConnection, id: i32) -> Post {
   use crate::schema::posts::dsl::{posts, is_published};
   diesel::update(posts.find(id))
       .set(is_published.eq(true))
       .get_result::<Post>(conn)
       .expect(&format!("Unable to find post {}", id))
}
```

### 要素を削除する

ここでは `LIKE` 句でパターンを指定して削除を行う処理を記述します。

```rust
pub fn delete_post(conn: &PgConnection, target: &str) -> usize {
   use crate::schema::posts::dsl::*;
   let pattern = format!("%{}%", target);
   diesel::delete(posts.filter(title.like(pattern)))
       .execute(conn)
       .expect("Error deleting posts")
}
```

## 参考

-   [Diesel の API ドキュメント](https://docs.diesel.rs/master/diesel/index.html)
-   [RustのDieselでPostgresに接続する with Docker](https://qiita.com/Yukimura127/items/6615f650aec3f579180b)
-   [diesel小ネタ集](https://keens.github.io/blog/2017/12/16/dieselshounetashuu/)