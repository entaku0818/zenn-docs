# Zenn Contents

[Zenn](https://zenn.dev/)で公開する記事や本を管理するリポジトリです。

## セットアップ

このリポジトリをクローンした後、以下のコマンドを実行してください。

```bash
# 依存パッケージをインストール
npm install
```

## コマンド一覧

### 記事の作成

```bash
# 新しい記事を作成
npx zenn new:article

# slugを指定して記事を作成
npx zenn new:article --slug 記事のスラッグ

# タイトルを指定して記事を作成
npx zenn new:article --title タイトル

# タイプを指定して記事を作成（tech または idea）
npx zenn new:article --type idea

# 絵文字を指定して記事を作成
npx zenn new:article --emoji 🔥
```

### 本の作成

```bash
# 新しい本を作成
npx zenn new:book

# slugを指定して本を作成
npx zenn new:book --slug 本のスラッグ
```

### プレビュー

```bash
# プレビューを開始（デフォルト: localhost:8000）
npx zenn preview

# ポート番号を指定してプレビュー
npx zenn preview --port 3000
```

## 公開方法

1. GitHubのリポジトリとZennを連携する
2. 変更をコミットしてプッシュする
3. Zennのダッシュボードでデプロイ状況を確認する

## 参考リンク

- [Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [ZennとGitHubリポジトリを連携する](https://zenn.dev/zenn/articles/connect-to-github)