---
title: Hugo で当ブログを作り直しました（Gatsbyから移行）
date: 2024-06-12
slug: create-site-with-hugo
tags:
  - Hugo
---

しばらくサイトの更新ができてませんでした。。。が、心機一転新しいツールを使って作り直しました。

前まで、Gatsby を使ってたり、 JamStack な構成にしてたりしたのですが、以下の点でちょっと辛くなってきました：
- Gatsby やその周辺テーマのライブラリ更新を何も考えずにやるとサイトが動かなくなってしまう
- テーマを色々更新していたのですが、維持するのが面倒になってきた
    - 自由度が高かったのですが、色々やろうと思うと結構手間だった
- JamStack な構成もオーバースペックになってきた
    - もう Markdown でサクッと記事を書ける方が良い
ということで、大体のツールを色々探していたのですが、 Go 製の静的サイトジェネレータ [Hugo](https://gohugo.io/) を使うのが無理なく維持管理できそうと思ったので、作り直してみました。

## サイトの作成から設定まで

### Hugo のインストール

[Hugo の公式ドキュメントの Installation](https://gohugo.io/installation/) を参考に各環境に応じてインストールできます。
当方の環境は mac なので、 brew 経由でインストールできました：

```shell
$ brew install hugo
$ hugo version
hugo v0.127.0+extended darwin/arm64 BuildDate=2024-06-05T10:27:59Z VendorInfo=brew
```

### サイトの作成

hugo コマンドですぐに作成できます：

```shell
$ hugo new site hugo_website_example
```

すると、次のような構成のファイルが自動作成されます：

```
hugo_website_example
|-- archetypes
|   `-- default.md
|-- assets
|-- content
|-- data
|-- hugo.toml
|-- i18n
|-- layouts
|-- static
`-- themes
```

#### hugo コマンドの使い方

基本的に使うのは以下の3つです：
- `hugo server -D`: ローカルでサイト立ち上げ
- `hugo --minify`: サイトをビルド
- `hugo new [ディレクトリ名]/[ファイル名].md`: 記事のファイルの新規作成

### テーマの導入

Hugo 向けのさまざまなテーマが公開されてます。今回は [Blowfish](https://blowfish.page/ja/) を使用しました。こちらも[ドキュメントの Installation](https://blowfish.page/ja/docs/installation/) を参考に作成できます。推奨されているのは `blowfish-tools` をインストールして Hugo のサイトを作るところからテーマのツールにやってもらう方法ですが、今回は先に Hugo のサイトを作成したので、 git のサブモジュールを用いる方法でテーマを導入します：

```shell
$ cd hugo_website_example
$ git init
$ git submodule add -b main https://github.com/nunocoracao/blowfish.git themes/blowfish
```

### 設定ファイルの作成

カスタマイズをしようと思うと、色々な設定はありますが、テーマを導入して日本語のサイトを作るための最低限の設定は以下の通りです:
（この辺りの設定は、 Blowfish のテーマの中身の設定ファイルの内容などをそのまま持ってきてよしなに変更すれば適用されます。テーマの中身の設定ファイルを直接編集することはしなくて OK です）

#### `config/_default/config.toml`

```toml
theme = "blowfish"
baseURL = "https://hogehoge.com/"
languageCode = "ja"
defaultContentLanguage = "ja"
```

#### `config/_default/language.ja.toml`

```toml
languageCode = "ja"
languageName = "日本語"
weight = 1
title = "OKAZAKI Shogo's Website"

[params]
  displayName = "JA"
  isoCode = "ja"
  rtl = false
  dateFormat = "2006年1月2日"
  logo = "img/logo.png"

[author]
  name = "OKAZAKI Shogo"
  image = "img/profile.png"
  links = [
    { email = "mailto:[メールアドレス]" },
    { github = "https://github.com/[GitHub のアカウント名]" },
```

ここでロゴやプロファイルの画像を指定していますが、画像ファイルなどは `assets` のディレクトリの下に置いておくと読んでくれます。

### CSS の設定

`assets` ディレクトリの配下に配置すれば OK です。テーマが読まれたあと、上書いて適用してくれます。ここで、フォントを指定したい場合は、 ファビコンと同じく `static` 配下に置いておきます。

## firebase の設定

CLI ツールで設定します。今回は、既存のサイトの置き換えだったので、以下のように設定していきました：

```shell
$ firebase init
# 以下、質問項目と回答
? Which Firebase features do you want to set up for this directory? Press Space to select features, then Enter to confirm your choices. Hosting: Configure files for Firebase Hosting and 
(optionally) set up GitHub Action deploys, Hosting: Set up GitHub Action deploys

=== Project Setup

? Please select an option: Use an existing project
? Select a default Firebase project for this directory: [すでにあるプロジェクト名]
i  Using project [すでにあるプロジェクト名]

=== Hosting Setup

? What do you want to use as your public directory? public
? Configure as a single-page app (rewrite all urls to /index.html)? No
? Set up automatic builds and deploys with GitHub? Yes
? File public/404.html already exists. Overwrite? No
i  Skipping write of public/404.html
? File public/index.html already exists. Overwrite? No
i  Skipping write of public/index.html

Visit this URL on this device to log in:
https://github.com/login/oauth/authorize?client_id=xxx&state=xxx&redirect_uri=xxx&scope=read%3Auser%20repo%20public_repo

Waiting for authentication...

✔  Success! Logged into GitHub as hogehoge

? For which GitHub repository would you like to set up a GitHub workflow? (format: user/repository) [リポジトリ名]

? Set up the workflow to run a build script before every deploy? No

? Set up automatic deployment to your site's live channel when a PR is merged? Yes
? What is the name of the GitHub branch associated with your site's live channel? main

✔  Firebase initialization complete!
```

Firebase の設定のみならず、 GitHub Actions でデプロイするワークフローの設定も入れてしまいました。GitHub Actions から Firebase にデプロイするために必要なトークンなどの情報は、自動的にシークレットに設定してくれます。楽ちんですね。

## GitHub Actions の設定

自動で作成されるワークフローですが、 Hugo で生成されたサイトをビルドするためのステップは入っていないので、以下のように修正しました：

```yaml
# This file was auto-generated by the Firebase CLI
# https://github.com/firebase/firebase-tools

name: Deploy to Firebase Hosting on merge
on:
  push:
    branches:
      - main
jobs:
  build_and_deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repo
        uses: actions/checkout@v4
        with:
          submodules: recursive
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
      - name: Build
        run: hugo --minify
      - name: Deploy
        uses: FirebaseExtended/action-hosting-deploy@v0
        with:
          repoToken: ${{ secrets.GITHUB_TOKEN }}
          firebaseServiceAccount: ${{ secrets.FIREBASE_SERVICE_ACCOUNT_XXX }}
          channelId: live
          projectId: XXX
```

ポイントは、
- チェックアウトのステップで `submodules: recursive` を指定
    - そうしないと、ビルドした `public` のディレクトリをうまく見つけてくれません
- [`peaceiris/actions-hugo`](https://github.com/peaceiris/actions-hugo) のアクションを利用して、 Hugo のセッティング
- ビルドのコマンドは `hugo --minify`
といったところです。

これで、 `main` に push すれば Hugo のサイトをビルドし、 Firebase にデプロイしてくれるようになりました。

## まとめ

過去の記事の移行など、作業としては少し手間な部分はありましたが、比較的簡単にサイトを作ることができました。また、維持管理もしやすそうです。
テーマなど、自分でバリバリデザインするのでなければ、これくらいの方が扱いやすいなとも思いました。
