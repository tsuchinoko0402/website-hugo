---
title: API サーバを Rust で実装する　〜ローカル開発からデプロイまで〜
date: 2022-01-19
slug: api-server-on-rust-develop-deploy
tags:
  - Rust
  - Heroku
  - GitHubActions
---

[前回の記事](https://www.zakioka.net/blog/api-server-on-rust-architecture/)で作成した Rust で実装した API サーバーをローカルで動作確認できる環境を整えます。また Heroku にもスムーズにデプロイできるように設定します。いずれも、Docker を利用します。  
GitHub のリポジトリは[こちら](https://github.com/tsuchinoko0402/rust-web-api-server-sample)。

## ローカル開発環境を整える

基本的には以前の記事：[docker-compose で DB とアプリサーバを立ち上げ、接続する](https://www.zakioka.net/blog/docker-compose-db-for-app/) で行ったことと同じです。

### DB 設定のための Dockerfile

`/db/Dockerfile` に DB のための設定を書きます：

```dockerfile
ENV LANG ja_JP.utf8
FROM postgres:14-alpine AS db
```

`/db/docker-entrypoint-initdb.d` のディレクトリの下に DB 立ち上げ直後に実行する SQL を置いておきます。

-   今回は、DB 立ち上げ時にロール `root` とデータベース `root` がないよ、というエラーメッセージが出るのに対応するための SQL を置いてます。
    -   なぜこのエラーが出るのかはよく分かってません。

```
CREATE ROLE root WITH LOGIN;
CREATE DATABASE root;
```

##Q# API サーバのための Dockerfile

`/server/Dockerfile` に API サーバを立ち上げるための設定を書きます：

-   マルチステージビルドを利用します。
-   開発環境（develop-stage）ではローカル環境でサーバーを稼働させるためのコマンド cargo-watch 、Diesel で PostgreSQL を扱う際に必要な libpg-dev 、Diesel でマイグレーションなどを行うのに必要なコマンド `diesel_cli` をインストールします。また、本番環境のために成果物をコピーしておきます。
-   ビルド環境（build-stage）では、開発環境を用いて本番環境向けのビルドを実行します。
-   本番環境（production-stage） では、ビルド環境の成果物をコピーしてきて実行します。

```dockerfile
# 開発環境
FROM rust:1.57.0 as develop-stage
WORKDIR /app
RUN cargo install cargo-watch
RUN apt install -y libpq-dev
RUN cargo install diesel_cli
COPY . .

# ビルド環境
FROM develop-stage as build-stage
RUN update-ca-certificates
RUN cargo build --release

# 本番環境
FROM rust:1.57.0-slim-buster as production-stage
RUN apt-get update
RUN apt-get install libpq-dev -y
COPY --from=build-stage /app/target/release/actix_web_sample .
CMD ["./actix_web_sample"]
```

### **`docker-compose.yml`** **の記述**

DB とサーバーの両方をローカル環境で立ち上げるために、 `docker-composed.yml` ﻿を用意します。

-   DB は `pg_isready` コマンドでヘルスチェックを行い、正常に起動していることを確認します。
-   DB の環境変数は後に紹介する `.env` ファイルの内容に合わせて設定します。
-   サーバーは DB が正常に起動したことを確認してから起動します。
    -   起動の際、diesel によるマイグレーション（テーブル作成）と [`cargo watch`](https://crates.io/crates/cargo-watch) によるアプリ起動を行います。
    -   `cargo watch` により、ソースの保存を自動で検知して再ビルドしてくれるため、開発が楽になります。

```
version: '3.7'
services:
  server:
    build:
      context: ./server
    target: 'develop-stage'
    ports:
      - "8080:8080"
    depends_on:
      db:
    condition: service_healthy
    volumes:
      - ./server:/app
      - cargo-cache:/usr/local/cargo/registry
      - target-cache:/app/target
    command: /bin/sh -c "diesel setup && cargo watch -x run"
    tty: true
  
   db:
     build:
       context: ./db
     ports:
       - '5432:5432'
     environment:
       POSTGRES_USER: admin
       POSTGRES_PASSWORD: password
       POSTGRES_DB: postgres
     volumes:
       - ./db/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d
     healthcheck:
       test: ["CMD-SHELL", "pg_isready", "-U", "${POSTGRES_USER}", "-d", "postgres"]
       interval: 10s
       timeout: 5s
       retries: 5
       
  volumes:
    cargo-cache:
    target-cache:
```

### `.env` の用意

ローカルで立ち上げる際に利用する環境変数の設定を記述します。

-   Rust の [dotenv](https://crates.io/crates/dotenv) で環境変数を設定するようにしています。

```
SERVER_PORT=8080
DATABASE_URL=postgres://admin:password@db:5432/postgres
SERVER_ADDRESS=0.0.0.0
```

### ローカル開発環境でのサーバ＆DB立ち上げ

docker-compose を用いてローカル環境を立ち上げます。  
前述の通り、開発中は立ち上げっぱなしにしておけば、Rust のコードを変更した場合でも自動的にビルドして反映してくれます。

```
docker-compose up -d
```

## Heroku の設定

コマンドラインから設定します。アプリの追加と、 PostgreSQL のアドオンを追加します。

-   PostgreSQL のアドオンを追加すると自動的にアプリの環境変数 `DATABASE_URL` が追加されます。

```
# Heroku に CLI でログイン（ブラウザが立ち上がるのでログインを行う）
$ heroku login
heroku: Press any key to open up the browser to login or q to exit:
Opening browser to https://cli-auth.heroku.com/auth/cli/browser/XXXXX
Logging in... done
Logged in as hoge@mail.com
heroku create rust-web-api-server-sample

# アプリの作成
$ heroku create rust-web-api-sample
Creating ⬢ rust-web-api-sample... done
https://rust-web-api-sample.herokuapp.com/ | https://git.heroku.com/rust-web-api-sample.git

# アプリに PostgreSQL のアドオン（Hobby プラン）を追加
$ heroku addons:create heroku-postgresql:hobby-dev --app rust-web-api-sample
Creating heroku-postgresql:hobby-dev on ⬢ rust-web-api-sample... free
Database has been created and is available
 ! This database is empty. If upgrading, you can transfer
 ! data from another database with pg:copy
Created postgresql-contoured-25420 as DATABASE_URL
Use heroku addons:docs heroku-postgresql to view documentation
```

## GitHub Actions の設定

ここから、手でコマンドをポチポチ打ってデプロイをしても良いのですが、せっかくなので、 GitHub Action でデプロイを自動化します。  
以下のように `.github/workflows/deploy-to-heroku.yml` を作成します：

-   [AkhileshNS/heroku-deploy](https://github.com/AkhileshNS/heroku-deploy) を利用してデプロイします。
-   Heroku へのデプロイは Docker を利用する設定にしています。
-   あらかじめ、リポジトリのシークレットに以下を設定しておきます：
    -   `HEROKU_API_KEY`: Heroku の Account Settings 画面 -> API Key で取得できる API キーを指定
    -   `HEROKU_EMAIL`: Heroku のアカウントで使用しているメールアドレスを指定
-   起動したサーバーの IP アドレスは環境変数で設定します。
    -   環境変数の設定は `heroku config:set` コマンドで行います。
    -   Github Actions 内で `heroku` コマンドを利用するためには Heroku へのログイン処理を行う必要があるのですが、AkhileshNS/heroku-deploy を一度でも使うことで Heroku へのログインを行なってくれます。
        -   このツールを使ってログイン処理だけ行うということもできます（`justlogin` オプション）
-   ヘルスチェックも入れています。

```yaml
name: Deploy to heroku

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: akhileshns/heroku-deploy@v3.12.12
        with:
          heroku_api_key: ${{secrets.HEROKU_API_KEY}}
          heroku_app_name: "rust-web-api-server-sample"
          heroku_email: ${{secrets.HEROKU_EMAIL}}
          buildpack: "https://github.com/emk/heroku-buildpack-rust.git"
          usedocker: true
          appdir: server
          healthcheck: "https://rust-web-api-server-sample.herokuapp.com/health"
          checkstring: "Ok"
          rollbackonhealthcheckfailed: true
      - run: |
          heroku config:set SERVER_ADDRESS=0.0.0.0
```

## Heroku へのデプロイにあたっての補足：環境変数について

Heroku でのサーバー起動の際のポートの指定は、環境変数 `PORT` の値を使用するようにすればOKですので、以下のように書けばOKです。

```
 let port = std::env::var("PORT")
        .ok()
        .and_then(|val| val.parse::<u16>().ok())
        .unwrap_or(CONFIG.server_port)
```

今回は、ローカルでの起動も考慮に入れているので、環境変数 `PORT` の指定がなければ、 `.env` ファイルの `SERVER_PORT` の値を利用するという設定にしています。

## まとめ

-   Docker を用いて、ローカルでの開発と Heroku へのデプロイを簡単にすることができました。
    -   今回の成果物をテンプレとすると、 Heroku に載せる Rust で実装した API サーバーを簡単に構築できる気がします。
-   他のサーバー環境へのデプロイ方法も試してみたいです。
-   GitHub Actions のワークフローをもう少し工夫して、テストなどを走らせるようにもしておきたいです。