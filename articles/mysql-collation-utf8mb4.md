---
title: "【MySQL】照合順序をutf8mb4_0900_ai_ciに統一する移行手順"
emoji: "🗄️"
type: "tech"
topics: ["MySQL", "データベース", "照合順序", "マイグレーション"]
published: false
---

# この記事は？

MySQLで日本語を扱う際、テーブルやカラムによって照合順序がバラバラだと、検索結果が不安定になったり、JOINでエラーが発生することがあります。この記事では、全テーブルの照合順序を`utf8mb4_0900_ai_ci`に統一する方法を解説します。

## 照合順序とは？

照合順序（Collation）は、文字列の比較・ソート方法を定義するものです。

### 主な照合順序

| 照合順序 | 特徴 |
|----------|------|
| `utf8mb4_general_ci` | 古い標準、高速だが精度低い |
| `utf8mb4_unicode_ci` | Unicode準拠、やや遅い |
| `utf8mb4_0900_ai_ci` | MySQL 8.0のデフォルト、高速かつ高精度 |
| `utf8mb4_bin` | バイナリ比較、大文字小文字を区別 |

### 命名規則

- `ai`: Accent Insensitive（アクセント記号を区別しない）
- `ci`: Case Insensitive（大文字小文字を区別しない）
- `0900`: Unicode 9.0準拠

## なぜutf8mb4_0900_ai_ciなのか？

1. **MySQL 8.0のデフォルト**: 最新の推奨設定
2. **パフォーマンス**: `utf8mb4_general_ci`より高速
3. **Unicode準拠**: 正確な文字比較
4. **日本語対応**: ひらがな・カタカナの正規化

## 移行手順

### 1. 現状の確認

```sql
-- データベースの照合順序
SELECT @@character_set_database, @@collation_database;

-- 各テーブルの照合順序
SELECT TABLE_NAME, TABLE_COLLATION
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database';

-- 各カラムの照合順序
SELECT TABLE_NAME, COLUMN_NAME, CHARACTER_SET_NAME, COLLATION_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'your_database'
  AND COLLATION_NAME IS NOT NULL;
```

### 2. マイグレーションファイルの作成

```sql
-- migrations/20231001_unify_collation.sql

-- データベースの照合順序を変更
ALTER DATABASE your_database
CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

-- テーブルごとに照合順序を変更
ALTER TABLE users
CONVERT TO CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

ALTER TABLE talk_rooms
CONVERT TO CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

ALTER TABLE talk_messages
CONVERT TO CHARACTER SET utf8mb4
COLLATE utf8mb4_0900_ai_ci;

-- 以下、すべてのテーブルに対して実行
```

### 3. 自動生成スクリプト

テーブルが多い場合は、SQLでマイグレーションを生成：

```sql
SELECT CONCAT(
    'ALTER TABLE `', TABLE_NAME, '` ',
    'CONVERT TO CHARACTER SET utf8mb4 ',
    'COLLATE utf8mb4_0900_ai_ci;'
) AS migration_sql
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_TYPE = 'BASE TABLE';
```

### 4. Goでのマイグレーション実行

```go
// migrations/unify_collation.go
package migrations

import (
    "context"
    "database/sql"
)

var tables = []string{
    "users",
    "talk_rooms",
    "talk_messages",
    "talk_room_members",
    // ... すべてのテーブル
}

func UnifyCollation(ctx context.Context, db *sql.DB) error {
    // データベースの照合順序を変更
    _, err := db.ExecContext(ctx, `
        ALTER DATABASE your_database
        CHARACTER SET utf8mb4
        COLLATE utf8mb4_0900_ai_ci
    `)
    if err != nil {
        return err
    }

    // 各テーブルの照合順序を変更
    for _, table := range tables {
        query := fmt.Sprintf(`
            ALTER TABLE %s
            CONVERT TO CHARACTER SET utf8mb4
            COLLATE utf8mb4_0900_ai_ci
        `, table)

        if _, err := db.ExecContext(ctx, query); err != nil {
            return fmt.Errorf("failed to convert table %s: %w", table, err)
        }
    }

    return nil
}
```

## 注意点

### 1. ロック時間

`ALTER TABLE ... CONVERT TO`はテーブルロックが発生します。大きなテーブルでは数分〜数十分かかることがあります。

```sql
-- レコード数の確認
SELECT TABLE_NAME, TABLE_ROWS
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
ORDER BY TABLE_ROWS DESC;
```

### 2. インデックスの再構築

照合順序を変更すると、文字列カラムのインデックスが再構築されます。ディスク容量に注意してください。

### 3. 外部キー制約

外部キーで参照しているカラムは、参照元と参照先の照合順序が一致している必要があります：

```sql
-- 外部キーの確認
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    CONSTRAINT_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'your_database'
  AND REFERENCED_TABLE_NAME IS NOT NULL;
```

### 4. 本番環境での実行

```bash
# メンテナンス時間を設けて実行
# 1. レプリカで先にテスト
# 2. バックアップを取得
# 3. 本番で実行
```

## 変更の確認

```sql
-- 変更後の確認
SELECT TABLE_NAME, TABLE_COLLATION
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'your_database'
  AND TABLE_COLLATION != 'utf8mb4_0900_ai_ci';

-- 結果が0件なら成功
```

## まとめ

- `utf8mb4_0900_ai_ci`はMySQL 8.0の推奨照合順序
- `ALTER TABLE ... CONVERT TO`で変換
- 大きなテーブルはロック時間に注意
- 外部キー制約がある場合は依存関係に注意

## 参考資料

- [MySQL - Character Sets and Collations](https://dev.mysql.com/doc/refman/8.0/en/charset.html)
- [MySQL - utf8mb4_0900_ai_ci](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-sets.html)
