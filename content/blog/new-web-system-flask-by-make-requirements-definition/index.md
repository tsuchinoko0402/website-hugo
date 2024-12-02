---
title: ボーイスカウト阪神さくら地区加盟員向けWebシステムリニューアル（ひとり）プロジェクト：要件を整理してみる
slug: new-web-system-flask-by-make-requirements-definition
date: 2024-12-02
---

ちょっと思い立ったので、今年は「ひとりアドベントカレンダー」に挑戦してみます。
ということで、「OKAZAKI Shogo のひとりアドベントカレンダー」の1日目です。

まずは、長いことやろうとしてやれてなかったことで妄想してることを吐き出してみます。

## はじめに
ボーイスカウト阪神さくら地区の内部向け HP に [文書館](https://member.bs-hanshin-sakura.org/Document/main.cgi) というものがあり、今でも現役で動いている。
これを私ひとりでリニューアルしようという試み。

## 概要

- 元は Perl で書かれた cgi 
    - とある人が1人で作った
- アップロードされたファイルを地区内の一般の指導者が閲覧することができる
- アップロードすることができる指導者は登録されたもののみ
- ファイルは地区のアカウントのGoogleドライブにアップし、文書館ページからはGoogleドライブのリンクを掲載することとする
- 文書にはタグ付けを行い、ジャンルを分けて表示することができる
    - 1つの文書にタグは複数つけることが可能
- ファイルタイプに応じたアイコンを表示する

## As-Is の確認

### 一般利用者向け画面

![一般利用者向け画面](./001.png)

### 管理者画面

![管理者画面1](./002.png)
![管理者画面2](./003.png)
![管理者画面3](./004.png)
![管理者画面4](./005.png)

- 管理者認証はBASIC認証のみ
- 文書データはサーバーに直置き
    - 文書データは `item.dat` として DAT ファイルで管理
    - レコード例： `001096		令和6年度　宗教章（仏教章第一教程）講習会 参加申込書		xls`
        - ID　文書名　ファイル種別
- 年度方針の機能は使われていない

### インフラ

- さくらのレンタルサーバースタンダードプラン
    - Python 3.8.12 がデフォルトでインストールされているので、Python ベースの Web アプリを動かすことは可能
    - MySQL が使える
        - ただし、ローカルホスト接続は無理。インターネット経由で。
- ボーイスカウトで Google WorkSpace のアカウントを持っている
    - ファイル自体は Google ドライブに置くのが良さそう
- ファイルのアップロードは FTP 経由
- ソースの git 管理はされてない

##  要件検討

- 処理の詳しい中身はわからない（Perlは読めない）ので、必要そうな機能を洗い出しながら作っていってみる
- 実装は Python で行う
    - Flask ベースのWebシステムにす
- ファイルと管理ユーザーの管理は MySQL の DB を使う

### ER 図

```mermaid
erDiagram
File {
  int file_id PK
  text file_name
  text display_name
  text url
  enum file_type
  set tag
  boolean is_standard
  datetime created_at
  varchar created_by
  datetime updated_at
  varchar updated_by
}

ManagerUser {
  int user_id PK
  varchar user_name
  text password
  varchar service
}
```

### ユースケース図
```plantuml
@startuml
left to right direction
actor 加盟員
actor 管理者

rectangle {
   加盟員 --> (文書館からダウンロードする)
   加盟員 --> (定型文書館からダウンロードする)

   管理者 --> (文書を登録する)
   管理者 --> (文書ファイル名を変更する)
   管理者 --> (文書ファイルを一覧から削除する)
   管理者 --> (一覧から消えている文書ファイルを復活させる)
   管理者 --> (文書ファイルを定型文書館にも掲載する)
   管理者 --> (文書ファイルを定型文書間から消す)
   
}
@enduml
```

### システム構成

とってもざっくりだが。
![システム構成](./006.png)

