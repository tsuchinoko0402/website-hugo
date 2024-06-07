---
title: docker-compose で DB とアプリサーバを立ち上げ、接続する
date: 2021-12-17
slug: docker-compose-db-for-app
tags:
  - Docker
  - docker-compose
  - Rust
  - PostgreSQL
---

`docker-compose` で DB とアプリサーバのコンテナを管理する際に、両者を接続するための方法に関しての備忘録です。  

## 構成

必要最低限の構成のみ記します。

```
.
|-- docker-compose.yml
`-- server
    |-- Dockerfile
    |-- Cargo.toml
    `-- src
         |-- *.rs
         ...      
```

## ファイルの中身

Rust 関連のファイルの内容に関しては省略します。  
DB は Postgres を使用し、アプリサーバには Rust で [diesel](https://github.com/diesel-rs/diesel) を用いて DB に接続するとします。

### `Dockerfile`

ポート 8088 でサーバを立ち上げます。  
Docker コンテナ立ち上げ時に diesel をインストールします。

```
# 開発環境
FROM rust:1.57.0 as develop-stage
WORKDIR /app
RUN cargo install cargo-watch
RUN apt install -y libpq-dev
RUN cargo install diesel_cli

# ビルド環境
FROM develop-stage as build-stage
RUN cargo build --release
```

### `docker-composed.yml`

diesel 起動時に環境変数 `DATABASE_URL` が必要なので指定します。

```yaml
version: '3.7'

services:
  server:
    build:
      context: ./server
      target: 'develop-stage'
    ports:
      - '8088:8088'
    depends_on:
      db:
        condition: service_healthy
    environment:
      - DATABASE_URL=postgres://admin:password@db:5432/postgres
    command: /bin/sh -c "diesel setup && cargo watch -x run
    tty: true
  db:
    image: postgres:14-alpine 
    ports: 
      - '5432:5432'
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      POSTGRES_DB: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
```

`docker-compose.yml` の中身に関してのポイントは以下の3点です：

-   DB のホスト名は DB コンテナのサービス名となる。
    -   今回の場合は `db`
    -   よって、`DATABASE_URL` に渡す URL は上記の通りとなる。
-   アプリサーバのコンテナは起動させっぱなしにする： `tty: true` を指定する。
-   アプリサーバのコンテナは DB サーバのコンテナが正常起動し終わった後に起動させる。
    -   `server` の `depends_on` にサービス `db` を指定し、条件 `service_healthy` を指定する。
    -   `db` の `healthcheck` に DB が正常に起動していることを確かめるためのコマンドを指定する。
    -   `depends_on` のみの指定は単に起動順を制御するのみで、正常起動したかどうかを確かめた上で起動するわけではないことに注意。

上記のファイルを作成した上で、 `docker-compose up -d` を実行すると、（初回はビルドが走った後に）無事にコンテナが起動します。

```
$ docker-compose ps                                                                                                                       (git)-[main]
               Name                              Command                State                   Ports         
------------------------------------------------------------------------------------------------------------
rust-web-api-server-sample_db_1       docker-entrypoint.sh postgres   Up(healthy)      0.0.0.0:5432->5432/tcp
rust-web-api-server-sample_server_1   bash                            Up               0.0.0.0:8088->8088/tcp
```

## まとめ

-   docker-compose で DB とアプリサーバを管理する際に引っかかった点をまとめました。
-   diesel を使用していると、 DB が正常起動していなければアプリが立ち上がらなかったため、少し苦労しました。

## 参考資料

-   [Docker Compose の depends_on の使い方まとめ](https://gotohayato.com/content/533/)
-   [peter-evans/docker-compose-healthcheck](https://github.com/peter-evans/docker-compose-healthcheck)
-   [Connecting to Postgres with Rust](https://viblo.asia/p/connecting-to-postgres-with-rust-GrLZDReB5k0)