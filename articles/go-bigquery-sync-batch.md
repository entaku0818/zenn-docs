---
title: "【Go】MySQLのデータをBigQueryに定期同期するバッチを実装する"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "bigquery", "gcp", "batch"]
published: false
---

## この記事は？

MySQLに蓄積されたデータをBigQueryに定期同期するバッチ処理を実装した際の知見をまとめます。分析基盤としてBigQueryを活用するための第一歩です。

## 背景

- 本番DBに直接分析クエリを投げるとパフォーマンスに影響
- BigQueryで分析用のデータウェアハウスを構築したい
- 定期的にデータを同期するバッチ処理が必要

## 実装方法

### 1. BigQueryクライアントの初期化

```go
import (
    "context"
    "cloud.google.com/go/bigquery"
)

type BigQuerySyncer struct {
    mysqlDB *sql.DB
    bqClient *bigquery.Client
    projectID string
    datasetID string
}

func NewBigQuerySyncer(ctx context.Context, mysqlDB *sql.DB, projectID, datasetID string) (*BigQuerySyncer, error) {
    client, err := bigquery.NewClient(ctx, projectID)
    if err != nil {
        return nil, fmt.Errorf("failed to create bigquery client: %w", err)
    }

    return &BigQuerySyncer{
        mysqlDB:   mysqlDB,
        bqClient:  client,
        projectID: projectID,
        datasetID: datasetID,
    }, nil
}
```

### 2. 同期対象のデータ構造を定義

```go
// BigQueryに保存するユーザーデータ
type UserRecord struct {
    ID        int64     `bigquery:"id"`
    Name      string    `bigquery:"name"`
    Email     string    `bigquery:"email"`
    CreatedAt time.Time `bigquery:"created_at"`
    UpdatedAt time.Time `bigquery:"updated_at"`
}

// BigQueryに保存するアクティビティデータ
type ActivityRecord struct {
    ID        int64     `bigquery:"id"`
    UserID    int64     `bigquery:"user_id"`
    Action    string    `bigquery:"action"`
    CreatedAt time.Time `bigquery:"created_at"`
}
```

### 3. MySQLからデータを取得

```go
// fetchUsersUpdatedSince は指定時刻以降に更新されたユーザーを取得する
func (s *BigQuerySyncer) fetchUsersUpdatedSince(ctx context.Context, since time.Time) ([]*UserRecord, error) {
    query := `
        SELECT id, name, email, created_at, updated_at
        FROM users
        WHERE updated_at > ?
        ORDER BY id
    `
    rows, err := s.mysqlDB.QueryContext(ctx, query, since)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var records []*UserRecord
    for rows.Next() {
        var r UserRecord
        if err := rows.Scan(&r.ID, &r.Name, &r.Email, &r.CreatedAt, &r.UpdatedAt); err != nil {
            return nil, err
        }
        records = append(records, &r)
    }
    return records, rows.Err()
}
```

### 4. BigQueryにデータを挿入

```go
// syncUsers はユーザーデータをBigQueryに同期する
func (s *BigQuerySyncer) syncUsers(ctx context.Context, records []*UserRecord) error {
    if len(records) == 0 {
        slog.Info("no users to sync")
        return nil
    }

    table := s.bqClient.Dataset(s.datasetID).Table("users")
    inserter := table.Inserter()

    // バッチサイズで分割して挿入
    const batchSize = 1000
    for i := 0; i < len(records); i += batchSize {
        end := i + batchSize
        if end > len(records) {
            end = len(records)
        }
        batch := records[i:end]

        if err := inserter.Put(ctx, batch); err != nil {
            return fmt.Errorf("failed to insert batch: %w", err)
        }
        slog.Info("inserted batch", "start", i, "end", end)
    }

    return nil
}
```

### 5. 増分同期の実装

```go
// SyncState は同期状態を管理する
type SyncState struct {
    TableName     string    `json:"table_name"`
    LastSyncedAt  time.Time `json:"last_synced_at"`
}

// loadSyncState は最後の同期時刻を取得する
func (s *BigQuerySyncer) loadSyncState(ctx context.Context, tableName string) (*SyncState, error) {
    query := `SELECT last_synced_at FROM sync_states WHERE table_name = ?`
    var lastSyncedAt time.Time
    err := s.mysqlDB.QueryRowContext(ctx, query, tableName).Scan(&lastSyncedAt)
    if err == sql.ErrNoRows {
        // 初回同期の場合は1年前から
        return &SyncState{
            TableName:    tableName,
            LastSyncedAt: time.Now().AddDate(-1, 0, 0),
        }, nil
    }
    if err != nil {
        return nil, err
    }
    return &SyncState{
        TableName:    tableName,
        LastSyncedAt: lastSyncedAt,
    }, nil
}

// saveSyncState は同期状態を保存する
func (s *BigQuerySyncer) saveSyncState(ctx context.Context, state *SyncState) error {
    query := `
        INSERT INTO sync_states (table_name, last_synced_at)
        VALUES (?, ?)
        ON DUPLICATE KEY UPDATE last_synced_at = VALUES(last_synced_at)
    `
    _, err := s.mysqlDB.ExecContext(ctx, query, state.TableName, state.LastSyncedAt)
    return err
}
```

### 6. 同期バッチの実行

```go
// Run は同期処理を実行する
func (s *BigQuerySyncer) Run(ctx context.Context) error {
    // ユーザーテーブルの同期
    if err := s.syncTable(ctx, "users", s.syncUsersTable); err != nil {
        return fmt.Errorf("failed to sync users: %w", err)
    }

    // アクティビティテーブルの同期
    if err := s.syncTable(ctx, "activities", s.syncActivitiesTable); err != nil {
        return fmt.Errorf("failed to sync activities: %w", err)
    }

    return nil
}

func (s *BigQuerySyncer) syncTable(ctx context.Context, tableName string, syncFunc func(context.Context, time.Time) error) error {
    // 同期状態を取得
    state, err := s.loadSyncState(ctx, tableName)
    if err != nil {
        return err
    }

    slog.Info("starting sync", "table", tableName, "since", state.LastSyncedAt)

    // 同期実行
    syncStartTime := time.Now()
    if err := syncFunc(ctx, state.LastSyncedAt); err != nil {
        return err
    }

    // 同期状態を更新
    state.LastSyncedAt = syncStartTime
    if err := s.saveSyncState(ctx, state); err != nil {
        return fmt.Errorf("failed to save sync state: %w", err)
    }

    slog.Info("completed sync", "table", tableName)
    return nil
}

func (s *BigQuerySyncer) syncUsersTable(ctx context.Context, since time.Time) error {
    records, err := s.fetchUsersUpdatedSince(ctx, since)
    if err != nil {
        return err
    }
    return s.syncUsers(ctx, records)
}
```

### 7. main関数

```go
func main() {
    ctx := context.Background()

    mysqlDB, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err)
    }
    defer mysqlDB.Close()

    syncer, err := NewBigQuerySyncer(
        ctx,
        mysqlDB,
        os.Getenv("GCP_PROJECT_ID"),
        os.Getenv("BIGQUERY_DATASET_ID"),
    )
    if err != nil {
        log.Fatal(err)
    }
    defer syncer.bqClient.Close()

    if err := syncer.Run(ctx); err != nil {
        slog.Error("sync failed", "error", err)
        os.Exit(1)
    }

    slog.Info("sync completed successfully")
}
```

## ポイント

1. **増分同期**: `updated_at`を使って差分のみ同期することでパフォーマンス向上
2. **バッチ挿入**: 1000件ずつ分割して挿入することでメモリ使用量を抑制
3. **同期状態の管理**: 最後の同期時刻を保存して再開可能に
4. **構造体タグ**: `bigquery:"field_name"`でフィールド名をマッピング

## まとめ

GoとBigQueryのクライアントライブラリを使うことで、シンプルにデータ同期バッチを実装できます。増分同期の仕組みを入れることで、効率的かつ安定した同期が可能です。

## 参考資料

- [BigQuery Go Client Library](https://pkg.go.dev/cloud.google.com/go/bigquery)
- [Streaming inserts into BigQuery](https://cloud.google.com/bigquery/docs/streaming-data-into-bigquery)
