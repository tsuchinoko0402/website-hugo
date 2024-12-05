---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
categories: [""]
tags: ["","",""]
Authors: "OKAZAKI Shogo"
toc: true
autoCollapseToc: false
draft: true
---

### 1. はじめに
xxxx

### x. Markdown CheetSheet

#### Text Format

_Italic（斜体）_
*Italic（斜体）*

__Emphasis（強調）__
**Emphasis（強調）**

~~Strikethrough（取り消し線）~~

<details><summary>これは詳細表示の例です。</summary>詳細をこっちに書きます。</details>

This is `inline`.

### List
* text
    * test
    * test

- text
    - test
    - test

1. text
1. test
    1. test

#### Horizontal rules
* * *
***
*****
- - -
---------------------------------------

#### Blockquotes（引用）
> This is Blockquotes

#### Links（参照）
[yonehub blog](https://yonehub.y10e.com/)

#### Images（画像）
![sample](/img/sample/sample.png)

#### Tables（表）
| id     | name    | date       |
| ------ | ------- | ---------- |
| 1      | test    | 2019-01-01 |
| 2      | test    | 2019-01-02 |
| 3      | test    | 2019-01-03 |
