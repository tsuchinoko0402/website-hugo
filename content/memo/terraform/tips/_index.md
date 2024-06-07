---
title: Terraform の Tips
publishedDate: "2021-05-12"
updatedDate: "2021-05-12"
description: Terraform を利用する上で何回か使った Tips をまとめておく。
slug: /memo/terraform/tips
---

## 変数 `var` の上書き

- tffile 内で `var` で指定した値は外部から上書き可能
- コマンド実行時に上書き

    ```shell
    $ terraform plan -var 'foo=test-bucket'
    ```

- `vars.tfvars` ファイルを作成して以下のように作成

    ```
    access_key = "XXX"
    region = "xxx"
    secret_key = "XXX"
    ```

- 環境変数による指定
    - `TF_VAR_` のプレフィックスを付けて環境変数を設定すると、その値が変数にロードされる。`TF_VAR_foo='env-test' terraform plan` を実行すると同様の結果が得られる。

- コマンド実行時に以下のようにファイルを指定

    ```shell
    $ terraform apply -var-file=vars.tfvars
    ```

## 文字列操作

- `replace` 関数：[https://www.terraform.io/docs/language/functions/replace.html](https://www.terraform.io/docs/language/functions/replace.html)
    - `replace(string, substring, replacement)`
        - `substring` には正規表現も利用可能
        - [https://www.terraform.io/docs/language/functions/regex.html](https://www.terraform.io/docs/language/functions/regex.html)

    ```
    > replace("1 + 2 + 3", "+", "-")
    1 - 2 - 3

    > replace("hello world", "/w.*d/", "everybody")
    hello everybody
    ```