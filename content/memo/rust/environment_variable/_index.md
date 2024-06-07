---
title: Rust で環境変数
publishedDate: "2021-05-06"
updatedDate: "2021-05-06"
description: Rust で環境変数を扱う際のメモ
slug: /memo/rust/environment_variable
---

# envy

## 参考資料

- 主にこれ

    [Rustで環境変数を扱う | Developers.IO](https://dev.classmethod.jp/articles/rust-environment-variables/)

- envy を使う

    [crates.io: Rust Package Registry](https://crates.io/crates/envy)

    [envy - Rust](https://docs.rs/envy/)

## envy の利用

- 使用する crate（ `Cargo.toml` ）

    ```yaml
    [dependencies]
    envy = "0.4"
    ```

- 公式の使い方の例
    - 外部で `FOO=123, BAR=true, BAZ=example, BOOM=1,2,3` を設定しておく
    - 構造体のフィールドに `#[serde(default = "path")]` アトリビュートを適用することで、パラメータのデフォルト値を設定することが可能

    ```rust
    use serde::Deserialize;

    #[derive(Deserialize, Debug)]
    struct Config {
      foo: u16,
      bar: bool,
      baz: String,
    	boom: Vec<u64>,
    	#[serde(default="default_port")]
      port: u16,
    }

    fn default_port() -> u16 {
        8080
    }

    fn main() {
    		let config = match envy::from_env::<Config>() {
           Ok(val) => println!("{:#?}", val),
           Err(error) => panic!("{:#?}", error)
        }

    		println!("{:#?}", config);

    		assert_eq!(config.foo, 123);
        assert_eq!(config.bar, true);
    		assert_eq!(config.baz, example);
        assert_eq!(config.boom, vec![1,2,3]);
    		assert_eq!(config.port, 8080);
    }
    ```

# dotenv

- TypeScript でも同様のライブラリがある
- 環境変数を `.env` なる外部ファイルに書き出ししておいておける

## 参考資料

- 主にこれ

    [Rustで設定ファイルの内容をグローバル変数に保存する - Qiita](https://qiita.com/tiohsa/items/d694dfbfce52da09ea53)

- 公式ドキュメント

    [dotenv - Rust](https://docs.rs/dotenv/)

    [crates.io: Rust Package Registry](https://crates.io/crates/dotenv)

### 使い方

- 使用する crate

    ```yaml
    [dependencies]
    # .envファイルの内容を環境変数として読み込む
    dotenv = "0.15.0"
    # .envの内容をConfigに保存するcrate
    config = "0.10.1"
    # 実行時にstatic変数を初期化するcrate
    once_cell = "1.9.0"
    serde = {version = "1.0.116", features = ["derive"]}
    ```

- コードの内容はこんな感じ

    ```rust
    extern crate lazy_static;

    use config::ConfigError;
    use dotenv::dotenv;
	use once_cell::sync::Lazy;
    use serde::Deserialize;

    /// .evnの内容を保存するstruct
    #[derive(Deserialize, Debug)]
    pub struct Config {
        pub address: String,
        pub port: i32,
    }

    impl Config {
        /// 環境変数からデータを読み込む
        pub fn from_env() -> Result<Self, ConfigError> {
            let mut cfg = config::Config::new();
            cfg.merge(config::Environment::new())?;
            cfg.try_into()
        }
    }

    /// static変数の初期化
    static CONFIG: Lazy<Config> = Lazy::new(|| {
		dotenv().ok();
		Config::from_env().unwrap()
	});

    fn main() {
        println!("address: {}", CONFIG.address);
        println!("port: {}", CONFIG.port);
    }
    ```

- `.dotenv` は以下のように書いておけば良い

    ```
    ADDRESS=localhost
    PORT=6379
    ```
