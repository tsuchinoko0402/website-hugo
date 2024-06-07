---
title: Rust の静的変数
publishedDate: "2021-05-06"
updatedDate: "2021-05-06"
description: Rust で静的変数を扱う際のメモ
slug: /memo/rust/static
---

## 参考資料

- [`const` と `static`](https://doc.rust-jp.rs/the-rust-programming-language-ja/1.6/book/const-and-static.html)

- [Rustのstaticいろいろ - 簡潔なQ](https://qnighy.hatenablog.com/entry/2018/06/17/190000)

- [Rustのstatic変数とthread local - Qiita](https://qiita.com/tatsuya6502/items/bed3702517b36afbdbca)

- [lazy_static はもう古い！？ once_cell を使おう](https://zenn.dev/frozenlib/articles/lazy_static_to_once_cell)

## const と static の違い

- `const` はコンパイル時に決まる**値**そのものを定義する。
    - この値はコンパイル時定数が必要な他の場所で使える。
    - コンパイル時に決まる値そのものを定義したい場合はこっちを使う。
- `static` は（コンパイル時に決まる値が入っている）**場所**を定義する。
    - 値がコンパイル時に決まるのは初期化の都合上にすぎず、それ以上の意味はない。
    - 状態を管理する場所を作りたい場合はこっちを使う。

## 不変（イミュータブル）なグローバルデータを  `once_cell` で実現

- crate のドキュメントとか
    - [crates.io: Rust Package Registry](https://crates.io/crates/once_cell)
    - [once_cell - Rust](https://docs.rs/once_cell)
    - [matklad/once_cell](https://github.com/matklad/once_cell)

- 用意にグローバル変数を使うクレートとして `lazy_static` がデファクトスタンダードになっていたが、マクロを使うため、ちょっとわかりにくい。。。ということで同様の処理をマクロ無しで実現したのが `once_cell`
- `Cargo.toml` に記述する内容は以下：

    ```toml
    [dependencies]
    once_cell = "1.5.2"
    ```

- `main.rs` にグローバル変数を定義する：

    ```rust
    #[macro_use]
    use once_cell::sync::Lazy;

    mod app_context {
    	use std::collections::HashMap;
    		
    	static MAP: Lazy<HashMap<u32, &'static str>> = Lazy::new(|| {
    		let mut m = HashMap::new();
    		m.insert(0, "foo");
    		m
    	});
    }

    const NF: &'static str = "not found";

    fn main() {
        read_app_map();
        println!("Done");
    }

    fn read_app_map() {
        let ref m = app_context::MAP;
        assert_eq!("foo", *m.get(&0).unwrap_or(&NF));
        assert_eq!(NF,    *m.get(&1).unwrap_or(&NF));
    }
    ```