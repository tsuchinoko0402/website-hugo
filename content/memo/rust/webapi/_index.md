---
title: Rust で Web API を叩く
publishedDate: "2021-05-06"
updatedDate: "2021-05-06"
description: Rust で Web API を叩く方法に関するメモ。
slug: /memo/rust/webapi
---

## 環境

```bash
$ cargo version
cargo 1.45.0 (744bd1fbb 2020-06-15)

$ rustc --version
rustc 1.45.0 (5c1f21c3b 2020-07-13)
```

## 依存関係

- HTTP クライアントの `reqwest` の crate を使う

    [reqwest](https://docs.rs/reqwest/0.10.6/reqwest/)

    [crates.io: Rust Package Registry](https://crates.io/crates/reqwest)

- `Cargo.toml` には以下のように記載

    ```rust
    [dependencies]
    reqwest = { version = "0.10", features = ["json"] }
    tokio = { version = "0.2", features = ["full"] }
    ```

## 使い方

- とりあえず GET して Body を取得してみる

    ```rust
    #[tokio::main]
    async fn main() -> Result<(), Box<dyn std::error::Error>> {
        let res = reqwest::get("https://www.google.co.jp/").await?.text().await?;
        println!("{:?}", res);
        Ok(())
    }
    ```

- POST するときは以下のように：

    ```bash
    let client = reqwest::Client::new();
    let res = client.post("http://httpbin.org/post")
        .body("the exact body that is sent")
        .send()
        .await?;
    ```

## `reqwest` をモックする

[reqwest_mock - Rust](https://docs.rs/reqwest_mock/)

## こんなのもあるらしい

- `reqwest` の軽量クレート： `ureq`

    [[Rust] ureq - reqwest の軽量な代替クレート - Qiita](https://qiita.com/osanshouo/items/adadfe6bbcbb201cbaed)