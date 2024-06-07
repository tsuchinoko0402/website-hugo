---
title: Rust の特徴、インストール、環境構築
publishedDate: "2021-05-05"
updatedDate: "2021-05-05"
description: Rust の基本的な事項
slug: /memo/rust/first-step
---

## 参考資料

- [Rust の公式サイト](https://www.rust-lang.org/ja)
    - [いわゆる The Book](https://doc.rust-lang.org/book/) と、 [Rust by Example](https://doc.rust-lang.org/stable/rust-by-example/) が聖書
- [Rustの日本語ドキュメント/Japanese Docs for Rust](https://doc.rust-jp.rs/)
    - ↑の日本語版も用意されている
- [κeen，河野 達也，小松礼人「実践Rust入門［言語仕様から開発手法まで］」技術評論社(2019)](https://gihyo.jp/book/2019/978-4-297-10559-4)

## Rust の特徴

- トップクラスのパフォーマンス
    - C++ ≒ C ≒ Rust > Java, Swift, Go（2~3倍遅い） > Node.js（6倍遅い）> Ruby, Python3（30倍遅い）
    - 早い理由
        1. マシンコードへのコンパイル
        2. 静的型付け
        3. ゼロコスト抽象化
            - プログラム言語が持つ抽象化の仕組みを実行時コストなしに動作する
                - 実行時コスト：実行速度やメモリ使用量
                - 抽象化：注目すべき要素を重点的に抜き出す。
                    - 例：オブジェクト指向におけるポリモーフィズム
            - Java 等は実行時に値の型を調べ、その型に対応するメソッドを呼び出す（**動的ディスパッチ**）
                - 同じ性質を満たす方を統一的に扱えるため、柔軟性が高い
                - 一方、実行時のコストがかかる
            - Rust はデフォルトでコンパイル事にわかる方によって呼び出すべきメソッドを決める（**静的ディスパッチ**）
                - 動的ディスパッチに比べ、柔軟性を欠く
                - 実行時のコストはかからない
        4. GC を行わない
            - 組み込みなど、リソースが限られている環境にも適している
- 安全なシステムプログラム言語
    - メモリ安全
        - C と C++ の問題：メモリ安全性に欠ける
            - データ転記の際のメモリ領域あふれ
            - ポインタによる誤ったメモリ領域へのアクセス
            - 初期化前のメモリ領域へのアクセス
            - 解放後のメモリ領域へのアクセス
            - データへのポインタと関数へのポインタの混同
        - 上記の問題を防ぐための安全なロジックを組むのは開発者の役目とされる
        - Java 等は解決法を用意している
            - ただし、リソースを必要としたりするため、組み込み系では向かなかったりする
        - Rust は開発者の責任だった問題をコンパイラが徹底的に検証する

            ```rust
            let a : i32;
            let b = a + 0; // 初期化されていない変数 a からの読み出しでコンパイルエラー

            let c = [0u8; 4];
            let c4a = c[4]; // 範囲外のインデックス（定数）にアクセスでコンパイルエラー

            let i = 4;
            let c4b = c[i]; // 範囲外のインデックス（変数）にアクセスでコンパイルエラー
            ```

    - 型安全
        - 正しく型付けされたプログラムが不正な動作（未定義動作）をしないように
        - シンタックス（文法）は正しくても、セマンティクス上問題があるプログラムをコンパイル時に見抜く

            ```rust
            let a = 1 + "hello"; // 数値と文字列の加算でコンパイルエラー

            if 1 { // bool が必要なところに数値、でコンパイルエラー
            println!("OK!")
            }

            1.unknown_method(); // 未定義メソッドの読み出しでコンパイルエラー
            ```

    - マルチスレッドプログラミングにおけるデータ競合の回避
        - スレッド間でデータを受け渡したり共有するために、いくつかの方法を用意
            1. チャネル
                - スレッド間でデータを送受信できる
                - データはある一時点で1つのスレッドから所有されることになり、データ競合が起こらないことが保証される
                - チャネルでは、値だけでなく、ポインタも送れる
            2. ロック
                - スレッドがデータにアクセスするときは、最初にロックを取得する必要がある
                - ある一時点で書き込みができるスレッドは1つだけとなる
            3. 配列などの範囲
                - **スライス**を使って配列のある要素範囲だけへのアクセスを実現できる
                    - 例えば：配列の前半と後半の2つのスライスを作成し、前半を１つの、後半をもう一つのスレッドに渡せる
            4. イミュータブルな参照
                - あるデータを指すポインタを作成する際、データを読み取り専用にできる（イミュータブルな参照）
                - コンパイラは、イミュータブルな参照が有効な間は、参照元のデータに対する変更を許さない
    - アンセーフなコードもサポート
        - ただし、 `unsafe` キーワードのついたブロックで囲む必要がある
- 生産性を高めるモダンな機能
    - 強力な型推論
    - 代数的データ型
    - パターンマッチ
    - トレイトによるポリモーフィズム
- シングルバイナリ、クロスコンパイル
    - アプリをビルドすると、シングルバイナリが生成される
    - クロスコンパイルにより、様々なプラットフォームに向けたバイナリを生成可能
- 多言語との連携が容易
    - FFI（多言語関数インタフェース）を通じて、他の言語と連携できる
        - FFI 関連のツール：bindgen, Neon, PyO3, Ruru, Rustler

## Rust のインストールと開発環境の構築

- 以下が必要
    - Rust ツールチェーン：`rustc` コマンド, `cargo` コマンド, `std`(標準ライブラリ)
        - `rustup` というツールでインストール
    - ターゲット環境向けのリンカ（Linker）
        - 正式には、リンケージエディタと呼ばれる
        - オブジェクトファイルやライブラリを結合して、ターゲット環境の ABI に準拠した実行可能ファイル（バイナリ）を生成
- `rustup` の機能
    - 複数バージョンの Rust ツールチェインのインストールと管理
    - クロスコンパイル用のターゲットのインストール
    - RLS などの開発支援ツールのインストール
    - stable 版だけでなく、nightly 版も管理可能
- 以下、 macOS へのインストール前提

### Rust ツールチェーンのインストール

- `rustup` をダウンロード

    ```shell
    $ curl <https://sh.rustup.rs> -sSf | sh
    ```

- コマンド検索パスの設定

    ```shell
    $ source $HOME/.cargo/env
    ```

    - fish の場合はこう

        ```shell
        $ set -U fish_user_paths $fish_user_paths $HOME/.cargo/bin
        ```

- 確認

    ```shell
    $ rustup --version
    rustup 1.22.1 (b01adbbc3 2020-07-08)
    $ rustc --version
    rustc 1.45.2 (d3fb005a3 2020-07-31)
    $ cargo --version
    cargo 1.45.1 (f242df6ed 2020-07-22)
    ```

### リンカのインストール

- `rustc` は `cc` コマンド経由で `ld` を呼び出す
- コマンドライン・デベロッパ・ツールのインストールが必要
- `cc -v` でバージョンを確認
    - なんか、エラーが出た
    - ので解決策　→ [git configコマンドを実行しようとしたらxcrun: errorになった](https://qiita.com/nya__str/items/f025fd2657e042e21e03)

    ```shell
    # 失敗
    $ cc -v 
    xcrun: error: active developer path ("/Applications/Xcode.app/Contents/Developer") does not exist
    Use `sudo xcode-select --switch path/to/Xcode.app` to specify the Xcode that you wish to use for command line developer tools, or use `xcode-select --install` to install the standalone command line developer tools.
    See `man xcode-select` for more details.

     # ↑の解決策を参考に
    $ xcode-select -p
    /Applications/Xcode.app/Contents/Developer

    $ sudo xcode-select -switch /Library/Developer/CommandLineTools

    $ xcode-select -p
    /Library/Developer/CommandLineTools

    # 再度
    $ cc -v
    Apple clang version 12.0.0 (clang-1200.0.31.1)
    Target: x86_64-apple-darwin19.6.0
    Thread model: posix
    InstalledDir: /Library/Developer/CommandLineTools/usr/bin
    ```

### 開発環境の構築

- VSCode を使うか、 IntelliJ のプラグインを使う
    - [Rust IDE に化ける VSCode](https://tech-blog.optim.co.jp/entry/2019/07/18/173000)
    - [IntelliJ の Rust Plugin](https://plugins.jetbrains.com/plugin/8182-rust/versions)

### パッケージの作成、ビルド、実行

- 以下のコマンドで

    ```shell
    $ cargo -new --bin hello # bin クレートの作成: バイナリパッケージが作られる
    $ cargo -new --lib hello # lib クレートの作成: ライブラリパッケージが作られる
    ```

- `Cargo.toml` ファイルにパッケージの情報を書く
- ビルドは `cargo build`
    - コードの検査
    - コンパイル
    - リンク
        - lib クレートでは実行されない
        - オブジェクトファイルと Rust 標準ライブラリなどのライブラリを結合
        - ABI に準拠した実行可能バイナリを作成
- 実行はバイナリファイルを実行するか、`cargo run` コマンドで

### 基本的なプログラムの内容

- ベタな "Hello world!" を出力するプログラム：

    ```rust
    fn main() {
        println!("Hello, world!");
    }
    ```

- `fn` は関数定義
    - 基本構文は以下
    - 引数と戻り値の型は省略可能

    ```rust
    fn 関数名(引数1: 型1, 引数2: 型2, ...) -> 戻り値の型 {
        関数の処理内容;
    }
    ```

- `println!` はマクロ
    - コンパイル初期段階で評価される
    - その定義に従って別のソースコードへと展開される
    - Rust の関数は（現時点では）可変個数の引数はサポートしていないが、マクロは可能

    ```rust
    println!(
    "Hello, {}!",
    "Hoge"
    ); //=>「Hello, Hoge!」
    println!(
    "半径 {:.1}, 円周率 {:.3}, 面積 {:.3}",
    3.2,
    std::f64::consts::PI,
    3.2f64.powi(2) * std::f64::consts::PI,
    ); //=> 「半径 3.2, 円周率 3.142, 面積 32.170」
    ```

### プログラムを作ってみる: 逆ポーランド記法の計算機例

- main 関数
    - `assert_debug!` はデバッグ時ビルド時のみ展開される
    - 要は評価、一致しない場合は、エラーでプログラム終了

    ```rust
    fn main() {
        let exp = "6.1 5.2 4.3 * + 3.4 2.5 / 1.6 * -";
        let ans = rpn(exp);
        debug_assert_eq!("26.2840", format!("{:.4}", ans));
        println!("{} = {:.4}", exp, ans);
    }
    ```

- rpm 関数
    - `split_whitespace()`: 空白を区切りとして、イテレータを返す
    - `parse`: `token` が f64 に変換できるか試す
    - `|x, y| x + y`: クロージャ（無名関数の一種）

    ```rust
    fn rpn(exp: &str) -> f64 {
        let mut stack = Vec::new();

        for token in exp.split_whitespace() {
            if let Ok(num) = token.parse::<f64>() {
                stack.push(num);
            } else {
                match token {
                    "+" => apply2(&mut stack, |x, y| x + y),
                    "-" => apply2(&mut stack, |x, y| x - y),
                    "*" => apply2(&mut stack, |x, y| x * y),
                    "/" => apply2(&mut stack, |x, y| x / y),
                    _ => panic!("Unknown operator: {}", token),
                }
            }
        }
        stack.pop().expect("Stack underflow")
    }
    ```

- apply2 関数
    - `<F>`: ジェネリクス
        - `F` は `where` 節で指定したトレイト境界を満たす型なら、どれにでもなれる

    ```rust
    // スタックから数値を2つ取り出して、F型のクロージャ fun で計算し、結果をスタックに積む
    fn apply2<F>(stack: &mut Vec<f64>, fun: F)
    where
        F: Fn(f64, f64) -> f64,
    {
        if let (Some(y), Some(x)) = (stack.pop(), stack.pop()) {
            let z = fun(x, y);
            stack.push(z);
        } else {
            panic!("Stack underflow");
        }
    }

    ```

## デバッガーの設定

- 参考：[https://nao_tuboyaki.gitlab.io/posts/2020/04/29/rust-vscode/](https://nao_tuboyaki.gitlab.io/posts/2020/04/29/rust-vscode/)
- VSCode での前提
    - 拡張機能「CodeLLDB」を入れる
    - `Cargo.toml` が直下にあるディレクトリで「構成の追加」
    - 自動で設定が以下のように追加される

    ```
    {
        // Use IntelliSense to learn about possible attributes.
        // Hover to view descriptions of existing attributes.
        // For more information, visit: <https://go.microsoft.com/fwlink/?linkid=830387>
        "version": "0.2.0",
        "configurations": [
            {
                "type": "lldb",
                "request": "launch",
                "name": "Debug executable 'rpn'",
                "cargo": {
                    "args": [
                        "build",
                        "--bin=rpn",
                        "--package=rpn"
                    ],
                    "filter": {
                        "name": "rpn",
                        "kind": "bin"
                    }
                },
                "args": [],
                "cwd": "${workspaceFolder}"
            },
            {
                "type": "lldb",
                "request": "launch",
                "name": "Debug unit tests in executable 'rpn'",
                "cargo": {
                    "args": [
                        "test",
                        "--no-run",
                        "--bin=rpn",
                        "--package=rpn"
                    ],
                    "filter": {
                        "name": "rpn",
                        "kind": "bin"
                    }
                },
                "args": [],
                "cwd": "${workspaceFolder}"
            }
        ]
    }
    ```

- ブレイクポイントを置いて、デバッグ実行（F5）で思ったところで止まったら OK

## ツールチェインの補足情報

### プラットフォーム・サポート・ティア

- プラットフォームごとのサポートレベルを Tier1~3 で分類

### リリースサイクル

- Nightly チャネル
    - 毎晩新しいリリース
    - unstable な機能も含む
- Beta チャネル
    - 6週間ごとに最新の rust コンパイラや標準ライブラリのソースコードから beta リリースが作られる
    - beta リリースから6週間経つと stable リリースに昇進する
- Stable チャネル
    - 広報互換性が保たれるようにしてる
- ポイントリリース
    - stable としてリリースした後に重大な問題が見つかった場合に行われる緊急リリース

### エディション

- Stable では、前のバージョンのコードが動かなくなるような新機能がリリースできない
- これを解決するために Rust 1.31.0 から2つのエディションが導入された
    - 2015 エディション：Rust 1.0.0 と後方互換性が保たれる仕様
    - 2018 エディション：新たな予約語などが取り入れられ、2015エディションとは一部が非互換の仕様
- `Cargo.toml` に下記のように書いて選択

    ```
    [package]
    edition = "2018"
    ```

- 上記を省略すると、 2015 エディションが指定されたと解釈される
- エディションは 2, 3年のペースで追加される見込み。古いエディションの廃止予定はない。
- `cargo fix`: 新しいエディションへの移行を支援するコマンド

### `rustup` の機能

- `rustup show` でインストール済みのツールチェインと現在アクティブになっているツールチェインを表示
- `rustup install` で追加のツールチェインをインストール
    - `nightly` とか
- `rustup update` でインストール済みのツールチェインと rustup 自体を最新版にアップデート
    - 定期的に実行すると良い
- 特定のパッケージのみ nightly を使用することも可能