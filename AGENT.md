# Agent Context: OKAZAKI Shogo's Website

## 概要 (Overview)
このリポジトリは、OKAZAKI Shogo の個人ウェブサイト（ブログ、技術メモ）のソースコードです。
日々の技術的な気づきや備忘録をまとめるためのプロジェクトです。

## コンテキスト (Context)
- ユーザー・グローバルルールの **System Engineering** の文脈に該当します。
- 高い技術水準（DDD、Rust、インフラ構築など）を扱うため、プロフェッショナルなエンジニアリングのトーンと用語を使用してください。
- 変更を加える際は、コードのシンプルさ（Simplicity First）を保ち、TDD（テスト駆動開発）が適用可能なカスタムロジック（テーマ拡張など）があれば適宜適用してください。

## 技術スタック (Tech Stack)
- **Framework**: [Hugo](https://gohugo.io/) (Static Site Generator)
- **Theme**: [Blowfish](https://blowfish.page/) (git submodule として `themes/blowfish` に配置)
- **Hosting**: Firebase Hosting (`okazaki-shogo-website`)
- **CI/CD**: GitHub Actions (Firebaseへのマージ/PR時自動デプロイ)

## ディレクトリ構造 (Directory Structure)
- `content/blog/`: ブログ記事。Page Bundle 形式 (`content/blog/[slug]/index.md`) で管理。
- `content/memo/`: 技術メモ。テーマごと（Rust, GCP, Terraform, Shellscriptなど）に階層化して管理。
- `config/_default/`: Hugo およびテーマの設定ファイル群 (`config.toml`, `params.toml` など)。
- `static/`: 静的ファイル。
- `themes/blowfish/`: テーマ本体（Submoduleのため直接編集は避ける）。

## 開発コマンド (Commands)
- **開発サーバー起動 (Draft含む)**: `hugo server -D`
- **ホットリロード**: `hugo server --navigateToChanged`
- **ビルド**: `hugo --minify`
- **記事作成**: `hugo new content/blog/[記事タイトル]/index.md`

## 運用ルール・ガイドライン (Guidelines)
1. **記事の作成**: 新規記事は `hugo new` コマンドを使用し、Page Bundle 形式（ディレクトリ内に `index.md` や関連画像を配置）で作成すること。
2. **テーマのカスタマイズ**: `themes/blowfish/` を直接編集せず、`config/_default/` の設定ファイルや、ルートの `layouts/`, `assets/`, `i18n/` ディレクトリを使用してオーバーライドすること。
3. **フロントマター**: 記事のフロントマターには、`title`, `date`, `slug`, `tags`, `categories` などを適切に設定すること。
4. **コミット**: 意味のある小さな単位でコミットし、コミット前にユーザーの承認を得るようにすること（User-Approved Commits）。
