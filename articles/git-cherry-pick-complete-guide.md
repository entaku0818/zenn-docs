---
title: "Git cherry-pick完全ガイド：コンフリクト解決から実践的な運用まで"
emoji: "🍒"
type: "tech"
topics: ["Git", "cherry-pick", "コンフリクト解決", "バージョン管理", "開発効率化"]
published: false
---

# この記事は？

Git cherry-pickを使う際、コンフリクトが発生して困った経験はありませんか？特にマージコミットをcherry-pickする時の「どちらの親から取り込むか」の選択や、コンフリクト解決中の効率的な作業方法について、実践的な手順とともに解説します。

この記事では、基本的なcherry-pickから高度なテクニック、そしてトラブル時の対処法まで、実際の開発現場で使える知識を体系的にまとめています。

## cherry-pickとは？

cherry-pickは、**特定のコミットだけを別のブランチに適用する**Git機能です。ブランチ全体をマージするのではなく、必要なコミットだけを「つまみ食い」できます。

### よくある使用シナリオ

```bash
# 典型的な開発フロー
main ──●──●──●──●
        \     /
feature  ●──●──●
```

- **ホットフィックスの適用**: 本番で発見されたバグ修正を複数ブランチに適用
- **機能の部分的な取り込み**: 大きなフィーチャーブランチから特定の機能のみを先行リリース
- **リリースブランチへの選択的反映**: 開発ブランチから安定した変更のみをリリースブランチに適用

## 基本的なcherry-pick手順

### 単一コミットのcherry-pick

```bash
# 基本的な使い方
git cherry-pick <commit-hash>

# 例：特定のバグ修正を現在のブランチに適用
git cherry-pick a1b2c3d
```

### 複数コミットの連続cherry-pick

```bash
# 範囲指定でのチェリーピック
git cherry-pick <start-commit>..<end-commit>

# 例：3つのコミットを連続で適用
git cherry-pick abc123..def456

# 個別に指定する場合
git cherry-pick abc123 def456 ghi789
```

### cherry-pick前の準備

```bash
# 1. 作業ディレクトリをクリーンにする
git status
git stash  # 未コミットの変更がある場合

# 2. 対象ブランチに移動
git checkout target-branch

# 3. 最新の状態に更新
git pull origin target-branch
```

## マージコミットのcherry-pick

マージコミットをcherry-pickする際は、**どちらの親から変更を取り込むか**を指定する必要があります。

### マージコミットの構造理解

```bash
# マージコミットの親を確認
git show --pretty=format:"%h %s" <merge-commit>^1  # 1番目の親
git show --pretty=format:"%h %s" <merge-commit>^2  # 2番目の親

# グラフィカルに確認
git log --oneline --graph --parents <merge-commit>
```

通常、マージコミットには2つの親があります：
- **1番目の親**: メインライン（マージ先のブランチ）
- **2番目の親**: マージされたブランチ

### どちらの親から取り込むかの選択

```bash
# 1番目の親（メインライン）から取り込む
git cherry-pick -m 1 <merge-commit-hash>

# 2番目の親（マージされたブランチ）から取り込む
git cherry-pick -m 2 <merge-commit-hash>
```

### 実践例：フィーチャーブランチのマージコミットを適用

```bash
# 状況：developブランチにマージされた機能をreleaseブランチに適用したい
# develop: A──B──M (Mはマージコミット)
#              \ /
# feature:      C──D

# releaseブランチに移動
git checkout release

# フィーチャーブランチの変更（2番目の親）を取り込む
git cherry-pick -m 2 <merge-commit-M>
```

**判断基準**:
- **フィーチャーブランチの変更を取り込みたい場合**: `-m 2`
- **メインラインの変更を取り込みたい場合**: `-m 1`

## cherry-pick中のコンフリクト解決

### コンフリクトが起きる典型的なパターン

1. **同じファイルの同じ箇所を変更**
2. **ファイルの移動・削除との競合**
3. **依存関係のあるコミットの順序問題**

### 基本的なコンフリクト解決手順

```bash
# cherry-pick実行
git cherry-pick abc123

# コンフリクト発生時の表示例
# error: could not apply abc123... Fix user authentication
# hint: after resolving the conflicts, mark the corrected paths
# hint: with 'git add <paths>' and run 'git cherry-pick --continue'

# 1. コンフリクト状況を確認
git status

# 2. コンフリクトファイルを編集
# <<<<<<< HEAD
# 現在のブランチの内容
# =======
# cherry-pickしようとしている内容
# >>>>>>> abc123 (Fix user authentication)

# 3. 解決後にステージング
git add <resolved-files>

# 4. cherry-pickを継続
git cherry-pick --continue
```

### cherry-pick中のcheckout活用

コンフリクト解決中に、他のブランチやコミットのファイル内容を参照・取り込むことができます：

```bash
# コンフリクト解決中に他ブランチのファイルを参照
git checkout <source-branch> -- <file-path>

# 元のHEADの状態に戻す
git checkout HEAD -- <file-path>

# cherry-pick元のファイルを確認
git checkout <cherry-pick-commit> -- <file-path>

# 例：認証ファイルのコンフリクト解決
git cherry-pick abc123
# コンフリクト発生
git checkout feature-auth -- src/auth.js  # フィーチャーブランチの内容を取り込み
git add src/auth.js
git cherry-pick --continue
```

### cherry-pick時のマージ戦略オプション

cherry-pick実行時に、コンフリクト解決の戦略を事前に指定できます：

```bash
# ours戦略：現在のブランチの変更を優先
git cherry-pick -X ours <commit-hash>

# theirs戦略：cherry-pick元のコミットの変更を優先
git cherry-pick -X theirs <commit-hash>

# union戦略：両方の変更をマージ（テキストファイルの場合）
git cherry-pick -X union <commit-hash>
```

**使い分けの目安**：
- **ours**: 設定ファイルなど、現在の環境設定を保持したい場合
- **theirs**: バグ修正など、cherry-pick元の変更を確実に適用したい場合  
- **union**: ドキュメントなど、両方の変更を統合したい場合

### 部分的な取り込みでの調整

```bash
# チェリーピック中断後、特定ファイルのみ手動適用
git cherry-pick --abort

# 必要なファイルのみを選択的に取り込み
git checkout <commit-hash> -- <specific-file>
git add <specific-file>
git commit -m "Partial cherry-pick: <description>"
```

## cherry-pick作業中の高度なテクニック

### 段階的なcherry-pick

```bash
# コミットせずにcherry-pick（-n オプション）
git cherry-pick -n <commit-hash>
git status  # 変更内容を確認
git reset HEAD <unwanted-file>  # 不要なファイルを除外
git commit -m "Cherry-pick: <description>"
```

### 状態確認とログ

```bash
# cherry-pick状況確認
git status
git log --oneline -5

# コミット内容確認
git show <commit-hash>
```

## 実践的なcherry-pick戦略

### コミット選択の指針

```bash
# 依存関係を確認
git log --oneline --graph <target-branch>

# コミット内容確認
git show <commit-hash>
```

**選択の指針**:
1. **自己完結したコミット**を優先
2. **依存関係の少ないコミット**から適用
3. **テストが通るコミット**を選択

## トラブルシューティング

### よくあるエラーと対処法

#### 1. マージコミットのエラー

```bash
# エラー例
error: commit abc123 is a merge but no -m option was given

# 対処法
git cherry-pick -m 1 abc123  # または -m 2
```

#### 2. 空のコミットエラー

```bash
# エラー例
The previous cherry-pick is now empty, possibly due to conflict resolution.

# 対処法：空のコミットを許可
git cherry-pick --allow-empty --continue

# または、スキップ
git cherry-pick --skip
```

#### 3. ファイル削除コンフリクト

```bash
# 削除されたファイルのコンフリクト
# CONFLICT (modify/delete): file.txt deleted in HEAD and modified in abc123

# ファイルを保持する場合
git add file.txt

# ファイルを削除する場合
git rm file.txt

git cherry-pick --continue
```

#### 4. マージ戦略が効かない場合の対処

```bash
# 戦略を指定してもコンフリクトが発生した場合
git cherry-pick -X ours abc123
# CONFLICT が発生

# 手動解決後、戦略の確認
git status
# 必要に応じて追加の調整
git checkout --ours <file-path>  # 現在のブランチの内容を採用
git checkout --theirs <file-path>  # チェリーピック元の内容を採用

git add <file-path>
git cherry-pick --continue
```

#### 5. 複数ファイルでの戦略適用

```bash
# 特定のファイルタイプに対してのみ戦略を適用
git cherry-pick -X ours abc123
# コンフリクト発生時、ファイル別に対処

# 設定ファイルは ours
git checkout --ours config/database.yml
git checkout --ours config/application.yml

# ソースコードは theirs
git checkout --theirs src/main.js
git checkout --theirs src/utils.js

git add .
git cherry-pick --continue
```

### checkout を使った復旧方法

```bash
# チェリーピック失敗時の特定ファイル復旧
git checkout HEAD~1 -- <file-path>

# 完全に元の状態に戻す
git cherry-pick --abort

# 特定のコミット状態に戻す
git checkout <safe-commit-hash> -- .
```

### 作業の中断・再開・やり直し

```bash
# 作業を中断
git cherry-pick --abort

# 現在の状態を保存して中断
git stash
git cherry-pick --abort
git stash pop

# やり直し
git reset --hard HEAD~1  # 最後のコミットを取り消し
git cherry-pick <commit-hash>  # 再実行
```

## 実際のワークフロー例

### ホットフィックスのチェリーピック

```bash
# シナリオ：本番環境でバグ発見、mainブランチで修正済み
# main: A──B──C──D (Dがバグ修正)
# release: A──B──E──F

# 1. リリースブランチに移動
git checkout release

# 2. ホットフィックスを適用
git cherry-pick D

# 3. コンフリクト発生時の対処
# CONFLICT in config/settings.js
git checkout main -- config/settings.js  # mainの設定を取り込み
git checkout release -- config/database.js  # リリース固有の設定は保持
git add .
git cherry-pick --continue

# 4. 検証
npm test
git push origin release
```

### 機能の部分的チェリーピック

```bash
# シナリオ：大きなフィーチャーブランチから特定機能のみを先行リリース
# feature: A──B──C──D──E (CとDが必要な機能)

# 1. 対象ブランチに移動
git checkout develop

# 2. 段階的にチェリーピック
git cherry-pick -n C  # コミットせずに適用
git status
# 不要なファイルを除外
git reset HEAD tests/integration_test.js
git checkout HEAD -- tests/integration_test.js

git commit -m "Feature: Add user profile API (partial)"

# 3. 次のコミットを適用
git cherry-pick D

# 4. 依存関係の調整
git checkout feature -- src/utils/helper.js  # 必要な依存ファイルを追加
git add src/utils/helper.js
git commit -m "Add missing dependencies for user profile"
```

## チェリーピック vs その他の手法

### マージとの使い分け

| 手法 | 適用場面 | メリット | デメリット |
|------|----------|----------|------------|
| チェリーピック | 特定のコミットのみ必要 | 選択的な適用が可能 | 履歴が複雑になる可能性 |
| マージ | ブランチ全体を統合 | 履歴が明確 | 不要な変更も含まれる |

```bash
# チェリーピックが適している場面
git cherry-pick abc123  # 特定のバグ修正のみ

# マージが適している場面
git merge feature-branch  # 機能全体の統合
```

## まとめ

チェリーピックを安全かつ効率的に行うためのチェックリスト：

### 事前準備
- [ ] 作業ディレクトリがクリーンな状態
- [ ] 対象ブランチが最新の状態
- [ ] チェリーピック対象のコミットを特定
- [ ] 依存関係を確認

### 実行中
- [ ] マージコミットの場合は適切な親を選択（`-m 1` or `-m 2`）
- [ ] コンフリクト発生時は`git checkout`を活用して効率的に解決
- [ ] 段階的な適用で安全性を確保
- [ ] 各段階でテストを実行

### 事後確認
- [ ] 期待通りの変更が適用されているか確認
- [ ] テストが通ることを確認
- [ ] 不要な変更が含まれていないか確認
- [ ] コミットメッセージが適切か確認

チェリーピックは強力な機能ですが、適切な手順と理解があってこそ真価を発揮します。特にコンフリクト解決時の`git checkout`の活用や、マージコミットの親選択は、実際の開発現場で頻繁に遭遇する場面です。

この記事の内容を参考に、安全で効率的なチェリーピック作業を実践してください。

## 参考リンク

- [Git公式ドキュメント - git-cherry-pick](https://git-scm.com/docs/git-cherry-pick)
- [Git公式ドキュメント - git-checkout](https://git-scm.com/docs/git-checkout)
- [Atlassian Git Tutorial - Cherry Pick](https://www.atlassian.com/git/tutorials/cherry-pick)
- [GitHub Docs - About merge conflicts](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/about-merge-conflicts) 