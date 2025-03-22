---
title: "【Go】人気コンテンツを自動で別テーブルに複製する仕組みを実装する"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "mysql", "batch"]
published: false
---

## この記事は？

コンテンツの人気度（いいね数やコメント数）に応じて、自動的に別テーブルに複製する機能を実装した際の知見をまとめます。「おすすめ」や「人気」として表示するコンテンツを効率的に管理する仕組みです。

## 背景

- コンテンツの人気度に応じて「おすすめ」として表示したい
- 毎回人気度を計算してソートするのはパフォーマンスが悪い
- 人気コンテンツを別テーブルで管理し、高速に取得したい

## 実装方法

### 1. 人気判定のしきい値を定義

```go
const (
    // 人気コンテンツと判定するしきい値
    PopularLikeThreshold    = 10 // いいね数
    PopularCommentThreshold = 5  // コメント数
)

// IsPopular は人気コンテンツかどうかを判定する
func (c *Content) IsPopular() bool {
    return c.LikeCount >= PopularLikeThreshold ||
           c.CommentCount >= PopularCommentThreshold
}
```

### 2. 複製先テーブルの定義

```sql
CREATE TABLE popular_contents (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    content_id BIGINT UNSIGNED NOT NULL,
    category_id BIGINT UNSIGNED NOT NULL,
    popularity_score INT NOT NULL DEFAULT 0,
    copied_at DATETIME NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_content_id (content_id),
    INDEX idx_category_score (category_id, popularity_score DESC)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 3. 自動複製バッチの実装

```go
type PopularContentCopier struct {
    db *sql.DB
}

// CopyPopularContents は人気コンテンツを複製する
func (c *PopularContentCopier) CopyPopularContents(ctx context.Context) error {
    // 人気コンテンツを取得
    contents, err := c.fetchPopularContents(ctx)
    if err != nil {
        return fmt.Errorf("failed to fetch popular contents: %w", err)
    }

    // バッチで複製
    for _, content := range contents {
        if err := c.upsertPopularContent(ctx, content); err != nil {
            // エラーログを出力して継続
            slog.Error("failed to upsert popular content",
                "content_id", content.ID,
                "error", err,
            )
            continue
        }
    }

    slog.Info("copied popular contents", "count", len(contents))
    return nil
}

// fetchPopularContents は人気コンテンツを取得する
func (c *PopularContentCopier) fetchPopularContents(ctx context.Context) ([]*Content, error) {
    query := `
        SELECT id, category_id, like_count, comment_count
        FROM contents
        WHERE like_count >= ? OR comment_count >= ?
        AND deleted_at IS NULL
    `
    rows, err := c.db.QueryContext(ctx, query,
        PopularLikeThreshold,
        PopularCommentThreshold,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var contents []*Content
    for rows.Next() {
        var content Content
        if err := rows.Scan(
            &content.ID,
            &content.CategoryID,
            &content.LikeCount,
            &content.CommentCount,
        ); err != nil {
            return nil, err
        }
        contents = append(contents, &content)
    }
    return contents, rows.Err()
}

// upsertPopularContent は人気コンテンツをUPSERTする
func (c *PopularContentCopier) upsertPopularContent(ctx context.Context, content *Content) error {
    // 人気度スコアを計算（いいね×2 + コメント×3）
    score := content.LikeCount*2 + content.CommentCount*3

    query := `
        INSERT INTO popular_contents (content_id, category_id, popularity_score, copied_at)
        VALUES (?, ?, ?, NOW())
        ON DUPLICATE KEY UPDATE
            popularity_score = VALUES(popularity_score),
            copied_at = NOW()
    `
    _, err := c.db.ExecContext(ctx, query, content.ID, content.CategoryID, score)
    return err
}
```

### 4. 人気でなくなったコンテンツの削除

```go
// RemoveUnpopularContents は人気でなくなったコンテンツを削除する
func (c *PopularContentCopier) RemoveUnpopularContents(ctx context.Context) error {
    query := `
        DELETE pc FROM popular_contents pc
        LEFT JOIN contents c ON pc.content_id = c.id
        WHERE c.id IS NULL
           OR c.deleted_at IS NOT NULL
           OR (c.like_count < ? AND c.comment_count < ?)
    `
    result, err := c.db.ExecContext(ctx, query,
        PopularLikeThreshold,
        PopularCommentThreshold,
    )
    if err != nil {
        return fmt.Errorf("failed to remove unpopular contents: %w", err)
    }

    affected, _ := result.RowsAffected()
    slog.Info("removed unpopular contents", "count", affected)
    return nil
}
```

### 5. バッチジョブとして実行

```go
func main() {
    ctx := context.Background()

    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    copier := &PopularContentCopier{db: db}

    // 人気コンテンツを複製
    if err := copier.CopyPopularContents(ctx); err != nil {
        slog.Error("failed to copy popular contents", "error", err)
    }

    // 人気でなくなったコンテンツを削除
    if err := copier.RemoveUnpopularContents(ctx); err != nil {
        slog.Error("failed to remove unpopular contents", "error", err)
    }
}
```

## ポイント

1. **ON DUPLICATE KEY UPDATE**: 既存レコードがあれば更新、なければ挿入
2. **人気度スコア**: いいねとコメントに重みをつけてスコア化
3. **削除処理**: 元データが削除されたり人気でなくなった場合は自動削除
4. **エラーハンドリング**: 1件の失敗で全体を止めず、ログを出して継続

## まとめ

人気コンテンツを別テーブルに複製することで、以下のメリットがあります：

- 人気コンテンツの取得が高速
- インデックスを最適化しやすい
- 元テーブルに影響を与えない

バッチで定期実行することで、常に最新の人気コンテンツを管理できます。

## 参考資料

- [MySQL ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html)
