---
title: Rust のトレイト
publishedDate: "2021-05-06"
updatedDate: "2021-05-06"
description: Rust のトレイトに関すること。
slug: /memo/rust/trait
---

## 基本概要

- 型に対して実装すべきメソッドを定義したのがトレイト
    - Java で言うところのインタフェース

    ```rust
    // デカルト座標
    #[derive(Debug, Clone, Copy, PartialEq, PartialOrd, Default)]
    pub struct CartesianCoord {
        pub x: f64,
        pub y: f64,
    }

    // 極座標
    pub struct PolarCoord {
        pub r: f64,
        pub theta: f64,
    }

    // 座標
    pub trait Coordinates {
        // 関数の本体は書かない
        fn to_cartesian(self) -> CartesianCoord;
        fn from_cartesian(cart: CartesianCoord) -> Self;
    }

    // デカルト座標系はそのまま
    impl Coordinates for CartesianCoord {
        fn to_cartesian(self) -> CartesianCoord {
            self
        }
        fn from_cartesian(cart: CartesianCoord) -> Self {
            cart
        }
    }

    // 極座標系は変換が必要
    impl Coordinates for PolarCoord {
        fn to_cartesian(self) -> CartesianCoord {
            CartesianCoord {
                x: self.r * self.theta.cos(),
                y: self.r * self.theta.sin(),
            }
        }
        fn from_cartesian(cart: CartesianCoord) -> Self {
            PolarCoord {
                r: (cart.x * cart.x + cart.y * cart.y).sqrt(),
                theta: (cart.y / cart.x).atan(),
            }
        }
    }

    // タプルにもトレイトを実装できる
    impl Coordinates for (f64, f64) {
        fn to_cartesian(self) -> CartesianCoord {
            CartesianCoord {
                x: self.0,
                y: self.1,
            }
        }
        fn from_cartesian(cart: CartesianCoord) -> Self {
            (cart.x, cart.y)
        }
    }
    ```

- ジェネリクスで受け取る型に境界（「トレイトを実装している型」というような境界）を**トレイト境界**という
- 下の例だと、パラメータ `P` の後にトレイト名を付けることで、`to_cartesian()` を実装している型しか受け取れない、と強制できる

    ```rust
    fn print_point<P: Coordinates>(point: P) {
        let p = point.to_cartesian();
        println!("({}, {})", p.x, p.y)
    }
    ```

- 関数の型の後に `where<トレイト境界>` という書き方も OK

    ```rust
    fn print_point<P>(point: P)
    where
        P: Coordinates,
    {
        let p = point.to_cartesian();
        println!("({}, {})", p.x, p.y)
    }
    ```

- `impl Trait` 構文で書く方法もある
    - 関数の引数の型の位置に `impl トレイト名` を書くことで、トレイト境界を指定する。
    - この構文を使うと、型パラメータや具体的な型名に言及せずにトレイト境界を書くことができる

    ```rust
    fn print_point(point: impl Coordinates) {
        let p = point.to_cartesian();
        println!("({}, {})", p.x, p.y)
    }
    ```

- トレイトを継承することも可能
- トレイトのメソッドにはデフォルト実装をもたせることができる
- トレイトで定義した関数は、トレイトとそれを実装する型が可視であれば、他のモジュールからアクセスできる
    - 関数ごとに `pub` を付ける必要はない
- トレイト実装のルール：あるトレイト `Trait` をある型 `Type` に実装するためには、トレイト `Trait` または型 `Type` の少なくともどちらか一方の定義のあるクレートで実装しなければならない

    ||型：自クレート|型：他クレート|
    |---|---|---|
    |**トレイト：自クレート**|○|○|
    |**トレイト：他クレート**|○|×|

- いくつかの標準ライブラリのトレイトは `#[derive(XXX)]` アトリビュートを使うことで型定義時に自動で実装できる

## トレイトのジェネリクス

- `trait トレイト名<型パラメータ>` で宣言できる

    ```rust
    trait Init<T> {
        fn init(t: T) -> Self;
    }
    ```

- 実装も今まで通り
    - 型パラメータを導入する箇所は、トレイト名より前であることに注意

    ```rust
    impl<T> Init<T> for Box<T> {
        // 内部では`T`でパラメータの型を参照する
        fn init(t: T) -> Self {
            Box::new(t)
        }
    }
    ```

## トレイトを実装する際に、列挙されてないメソッドも定義する

- クラスベースの言語から来た身としては、変に悩んでしまったので…
- [rust-jp.rs](https://rust-jp.rs/) の 日本語コミュニティ（Slack）の方々に回答いただいた。

### やりたいこと：Java での実装例

- タイトルだけでは何がしたいのかさっぱりわからないと思うので、実現したいことを Java で書くと以下のような感じ：

    ```java
    import java.util.*;

    interface Service {
        int ID = 123;
        void displayId();
    }

    class Module implements Service {
        public void displayId() {
            System.out.println(getId());
        }
        
        private int getId() {
            return this.ID;
        }
    }

    public class Main {
        public static void main(String[] args) {
            Module module = new Module();
            module.displayId();
        }
    }
    ```

- 要は、以下の2点：
    - インタフェース（Rust ではトレイト）で public なメソッドを1つ列挙しておきたい
    - 実装クラス（Rust では impl）の中で、 private なメソッドを定義して利用したい

### Rust で実現：失敗例

- これを Rust で実現しようとこのように書くとコンパイルエラーが出て怒られる：
    - [Rust Playground 上での実装例](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=38680dcb1db0e8180bb0d63a762f46f0)



    ```rust
    trait Service {
        fn display(&self);
    }

    struct Module {
        id: u64,
    }

    impl Service for Module {
        fn display(&self) {
            print!("{}", get_id());
        }

        fn get_id(&self) -> u64 {
            self.id
        }
    }

    fn main() {
        let module = Module{
            id: 12,
        };
        module.display();
    }
    ```


- エラーメッセージは、`error[E0407]: method get_id is not a member of trait Service` と `error[E0425]: cannot find function get_id in this scope` というもの。 
    - **`get_id` というメソッドは `Service` トレイトで列挙していないので、書けないよ** と怒られてしまう。

###  Rust で実現：成功例

- ということで、以下のようにするとコンパイラに怒られずに済む：

    ```rust
    trait Service {
        fn display(&self);
    }

    struct Module {
        id: u64,
    }

    impl Service for Module {
        fn display(&self) {
            print!("{}", self.get_id());
        }
    }

    impl Module {
        fn get_id(&self) -> u64 {
            self.id
        }
    }

    fn main() {
        let module = Module{
            id: 12,
        };
        module.display();
    }
    ```

- 直したのは、 **`Service` トレイトの実装 `Module` をトレイトの実装部分（ `impl Service for Module` の箇所）と Module 独自のメソッドを定義する部分（ `impl Module` の箇所）に分けて書く**、という点。