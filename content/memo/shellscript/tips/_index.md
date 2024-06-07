---
title: シェルスクリプトの Tips
publishedDate: "2021-05-28"
updatedDate: "2021-05-28"
description: よく使うけど、使うたびに忘れてしまうので、備忘録のためにメモ。
slug: /memo/shellscript/tips
---

## jq コマンド

- JSON をいじる際に便利なコマンド
- 参考にしたページ：
    - [jqコマンドで覚えておきたい使い方17個](https://orebibou.com/ja/home/201605/20160510_001/)
- 以下、例として以下のようなデータ（`example.json`）を扱うとする

```json
[
  {
    "id": "0001",
    "name": "test001",
    "value": 112,
    "group1": {
      "subg01": [
        {
          "id": "1001",
          "type": "type001"
        },
        {
          "id": "1004",
          "type": "type002"
        }
      ]
    },
    "group2": [
      {
        "id": "5001",
        "type": "None"
      },
      {
        "id": "5003",
        "type": "type053"
      },
      {
        "id": "5004",
        "type": "type054"
      }
    ]
  },
  {
    "id": "0002",
    "name": "test002",
    "value": 954,
    "group1": {
      "subg01": [
        {
          "id": "1021",
          "type": "type101"
        },
        {
          "id": "1054",
          "type": "type052"
        }
      ]
    },
    "group2": [
      {
        "id": "5001",
        "type": "None"
      },
      {
        "id": "5004",
        "type": "type054"
      }
    ]
  }
]
```

### select でフィルタリング

```shell
$ cat example.json | jq '.[] | select(.id == "0001")'
{
  "id": "0001",
  "name": "test001",
  "value": 112,
  "group1": {
    "subg01": [
      {
        "id": "1001",
        "type": "type001"
      },
      {
        "id": "1004",
        "type": "type002"
      }
    ]
  },
  "group2": [
    {
      "id": "5001",
      "type": "None"
    },
    {
      "id": "5003",
      "type": "type053"
    },
    {
      "id": "5004",
      "type": "type054"
    }
  ]
}
```

```shell
$ cat example.json | jq '.[].group1.subg01[] | select(.id == "1004")'
{
  "id": "1004",
  "type": "type002"
}
```

## シェルで変数の空文字判定

- 空じゃないときのみ処理を行う場合の条件の書き方

```shell
if [ -n "$STRING" ]; then
  #処理
fi
```

- 空のときのみ処理を行う場合の条件の書き方
    - ↓以外では `else` を使う手もある。

```shell
if [ -z "$STRING" ]; then
  #処理
fi
```

## 空行判定

- とあるものを動かしているときに `[: ']' expected` とか出てエラーになった。
- 行き当たった記事が [これ](https://karoten512.hatenablog.com/entry/2018/01/19/002558)

> 変数に格納されている文字列比較において、 
> 文字列の比較は文字列と行う必要があるところを、 
> 文字列と何かわからない型の変数を比較してしまっているので、
> このようなエラーが発生するらしい。

- 以下ではダメ。

```shell
#!/bin/bash
while read line
do
  echo $line # ここで出力する
  if [ $line = '' ]; then
    echo '空行があるよ'
  fi  
done < line.txt
```

- 変数を `" "` で囲うと良い。

```shell
#!/bin/bash
while read line
do
  if [ "$line" = '' ]; then
    echo '空行があるよ'
  fi  
done < line.txt
```