---
title: Rust のエラーハンドリング
publishedDate: "2021-05-06"
updatedDate: "2021-05-06"
description: Rust のエラーハンドリングに関すること。
slug: /memo/rust/error_handring
---

- [The Book の該当箇所](https://doc.rust-jp.rs/book-ja/ch09-00-error-handling.html)

## 分類

- 大きく分けて2つ
    - 回復不可能なエラー： `panic!` マクロを用いる
        - 配列の境界を超えた箇所にアクセスしようとすることなど
        - プログラムを強制的に終わらせる
    - 回復可能なエラー： `Result<T, E>` を用いる
        - ファイルが見つからないなどの回復可能なエラーには、問題をユーザに報告し、処理を再試行する

## panic! で回復不可能なエラーを起こす

- `panic!` マクロが実行されると、プログラムは失敗のメッセージを表示し、終了する。
    - 標準では、プログラムはスタックの巻き戻しを行う。
        - スタックを遡り、 遭遇した各関数のデータを片付ける。
    - しかし、これではやることが多くなる場合があるので、オプションとして、データの片付けをせずにプログラムを異常終了させる方法も用意されている。
        - 結果として、プロジェクトの実行可能ファイルは小さくなる。
        - ただし、この場合、スタックに残ったデータの処理は OS で行う必要がある。
        - このオプションを選択する場合は、 `Cargo.toml` の `[profile]` 欄に `panic = 'abort'` を追記する
            - 下記の例は、リリースモードにて異常終了を選択する場合の記述

                ```toml
                [profile.release]
                panic = 'abort'
                ```

- これが起こる場合の大半は、 何らかのバグが検出された時であり、プログラマ対して対処方法が明確ではない場合。
- 実際に自分で起こす場合は以下のようにする：

    ```rust
    fn main() {
        panic!("crash and burn");  //クラッシュして炎上
    }
    ```

    - これを実行すると、以下のようなエラーメッセージが発生：

        ```shell
        $ cargo run
           Compiling panic v0.1.0 (file:///projects/panic)
            Finished dev [unoptimized + debuginfo] target(s) in 0.25 secs
             Running `target/debug/panic`
        thread 'main' panicked at 'crash and burn', src/main.rs:2:4
        ('main'スレッドはsrc/main.rs:2:4の「クラッシュして炎上」でパニックしました)
        note: Run with `RUST_BACKTRACE=1` for a backtrace.
        ```

- ライブラリで実装されている `panic!` を起こしたのが以下の例：ベクタの境界を超えた要素へのアクセスを試みた

    ```rust
    fn main() {
        let v = vec![1, 2, 3];

        v[99];
    }
    ```

    - これを実行すると以下のメッセージが出る：

        ```shell
        $ cargo run
           Compiling panic v0.1.0 (file:///projects/panic)
            Finished dev [unoptimized + debuginfo] target(s) in 0.27 secs
             Running `target/debug/panic`
        thread 'main' panicked at 'index out of bounds: the len is 3 but the index is
        99', /checkout/src/liballoc/vec.rs:1555:10
        ('main'スレッドは、/checkout/src/liballoc/vec.rs:1555:10の
        「境界外番号: 長さは3なのに、添え字は99です」でパニックしました)
        note: Run with `RUST_BACKTRACE=1` for a backtrace.
        ```

        - このエラーは、自分のファイルではない `vec.rs` （標準ライブラリの `Vec<T>` の実装）ファイルを指している。
            - ベクタ `v` に対して `[]` を使った時に走るコードは、 `vec.rs` に存在し、ここで実際に `panic!` が発生している。
    - エラー発生に至るまでに呼び出された全関数を見る**バックトレース**を得るには、環境変数 `RUST_BACKTRACE` を 0 以外に設定する
        - 実際に吐き出されるバックトレースは以下のような感じ

        ```shell
        $ RUST_BACKTRACE=1 cargo run
            Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
             Running `target/debug/panic`
        thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', /checkout/src/liballoc/vec.rs:1555:10
        stack backtrace:
           0: std::sys::imp::backtrace::tracing::imp::unwind_backtrace
                     at /checkout/src/libstd/sys/unix/backtrace/tracing/gcc_s.rs:49
           1: std::sys_common::backtrace::_print
                     at /checkout/src/libstd/sys_common/backtrace.rs:71
           2: std::panicking::default_hook::{{closure}}
                     at /checkout/src/libstd/sys_common/backtrace.rs:60
                     at /checkout/src/libstd/panicking.rs:381
           3: std::panicking::default_hook
                     at /checkout/src/libstd/panicking.rs:397
           4: std::panicking::rust_panic_with_hook
                     at /checkout/src/libstd/panicking.rs:611
           5: std::panicking::begin_panic
                     at /checkout/src/libstd/panicking.rs:572
           6: std::panicking::begin_panic_fmt
                     at /checkout/src/libstd/panicking.rs:522
           7: rust_begin_unwind
                     at /checkout/src/libstd/panicking.rs:498
           8: core::panicking::panic_fmt
                     at /checkout/src/libcore/panicking.rs:71
           9: core::panicking::panic_bounds_check
                     at /checkout/src/libcore/panicking.rs:58
          10: <alloc::vec::Vec<T> as core::ops::index::Index<usize>>::index
                     at /checkout/src/liballoc/vec.rs:1555
          11: panic::main
                     at src/main.rs:4
          12: __rust_maybe_catch_panic
                     at /checkout/src/libpanic_unwind/lib.rs:99
          13: std::rt::lang_start
                     at /checkout/src/libstd/panicking.rs:459
                     at /checkout/src/libstd/panic.rs:361
                     at /checkout/src/libstd/rt.rs:61
          14: main
          15: __libc_start_main
          16: <unknown> 
        ```

## `Result` で回復可能なエラーを起こす

- `Result` enum は `Ok` と `Err` の2つから成る

    ```rust
    enum Result<T, E> {
        Ok(T),
        Err(E),
    }
    ```

    - `T` と `E` はジェネリクス
- 使い方は以下のような感じ

    ```rust
    use std::fs::File;

    fn main() {
        // let f: u32 = File::open("hello.txt"); と書くとコンパイルエラー
    		// 帰ってくる型は std::result::Result
        let f = File::open("hello.txt");

        let f = match f {
            Ok(file) => file,
            Err(error) => {
                // ファイルを開く際に問題がありました
                panic!("There was a problem opening the file: {:?}", error)
            },
        };
    }
    ```

### 色々な種類のエラーを異なる方法で扱う

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(ref error) if error.kind() == ErrorKind::NotFound => {
            match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => {
                    panic!(
                        //ファイルを作成しようとしましたが、問題がありました
                        "Tried to create file but there was a problem: {:?}",
                        e
                    )
                },
            }
        },
        Err(error) => {
            panic!(
                "There was a problem opening the file: {:?}",
                error
            )
        },
    };
}
```

- `if error.kind() == ErrorKind::Notfound` という条件式は、**マッチガード**と呼ばれる
    - アームのパターンをさらに洗練する `match` アーム上のおまけの条件式

### エラー時にパニックするショートカット

- `unwrap` :  `Result` が `Ok` の場合は中身の型を返す、 `Err` の場合は `panic!` を起こす

    ```rust
    use std::fs::File;

    fn main() {
        let f = File::open("hello.txt").unwrap();
    }
    ```

    - ファイルが見つからない場合は以下のようなメッセージが出力される

        ```plain
        thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: Error {
        repr: Os { code: 2, message: "No such file or directory" } }',
        src/libcore/result.rs:906:4
        ('main'スレッドは、src/libcore/result.rs:906:4の
        「`Err`値に対して`Result::unwrap()`が呼び出されました: Error{
        repr: Os { code: 2, message: "そのようなファイルまたはディレクトリはありません" } }」でパニックしました)
        ```

- `expect` : `unwrap` と基本的に同じだが、 `panic!` のメッセージ内容も記述可能

    ```rust
    use std::fs::File;

    fn main() {
        // hello.txtを開くのに失敗しました
        let f = File::open("hello.txt").expect("Failed to open hello.txt");
    }
    ```

    - エラーメッセージは以下のように成る

        ```plain
        thread 'main' panicked at 'Failed to open hello.txt: Error { repr: Os { code:
        2, message: "No such file or directory" } }', src/libcore/result.rs:906:4
        ```

### エラーを委譲する

- 関数内ではなく、関数の呼び出し元にエラーをどうするかを決めてもらう（ **委譲** ）

    ```rust
    use std::io;
    use std::io::Read;
    use std::fs::File;

    fn read_username_from_file() -> Result<String, io::Error> {
        let f = File::open("hello.txt");

        let mut f = match f {
            Ok(file) => file,
            Err(e) => return Err(e),
        };

        let mut s = String::new();

        match f.read_to_string(&mut s) {
            Ok(_) => Ok(s),
            Err(e) => Err(e),
        }
    }
    ```

    - 1つめのファイル呼び出しが失敗しても、2つめの文字読み込みが失敗しても、呼び出し元の関数にエラーを委譲する

#### エラー委譲のショートカット： `?` 演算子

- 上記の処理は以下のように書ける

    ```rust
    use std::io;
    use std::io::Read;
    use std::fs::File;

    fn read_username_from_file() -> Result<String, io::Error> {
        let mut f = File::open("hello.txt")?;
        let mut s = String::new();
        f.read_to_string(&mut s)?;
        Ok(s)
    }
    ```

- もっと省略するとこう。 `?` 演算子の連結：

    ```rust
    use std::io;
    use std::io::Read;
    use std::fs::File;

    fn read_username_from_file() -> Result<String, io::Error> {
        let mut s = String::new();

        File::open("hello.txt")?.read_to_string(&mut s)?;

        Ok(s)
    }
    ```

- `?` を利用することで、定型コードの多くを排除できる
- ただし、 `?` 演算子が使えるのは、 `return` する型が `Result` の関数のみ
    - 以下はエラーとなる：

        ```rust
        use std::fs::File;

        fn main() {
            let f = File::open("hello.txt")?;
        }
        ```

        - エラーメッセージはこんな感じ：

            ```plain
            error[E0277]: the trait bound `(): std::ops::Try` is not satisfied
            (エラー: `(): std::ops::Try`というトレイト境界が満たされていません)
             --> src/main.rs:4:13
              |
            4 |     let f = File::open("hello.txt")?;
              |             ------------------------
              |             |
              |             the `?` operator can only be used in a function that returns
              `Result` (or another type that implements `std::ops::Try`)
              |             in this macro invocation
              |             (このマクロ呼び出しの`Result`(かまたは`std::ops::Try`を実装する他の型)を返す関数でしか`?`演算子は使用できません)
              |
              = help: the trait `std::ops::Try` is not implemented for `()`
              (助言: `std::ops::Try`トレイトは`()`には実装されていません)
              = note: required by `std::ops::Try::from_error`
              (注釈: `std::ops::Try::from_error`で要求されています)
            ```