---
title: "【Go】Cloud Run Jobで適切なexit処理を実装してジョブの成功/失敗を正しく判定させる"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "cloudrun", "gcp", "batch"]
published: false
---

## この記事は？

Cloud Run Jobでバッチ処理を実行する際に、適切なexit処理を実装してジョブの成功/失敗をCloud Runに正しく認識させる方法をまとめます。

## 背景

- Cloud Run Jobはexit codeでジョブの成功/失敗を判定する
- exit code 0で成功、それ以外で失敗
- Goのlog.Fatalはexit code 1で終了するが、deferが実行されない
- グレースフルシャットダウンを実装したい

## 問題のあるコード

```go
func main() {
    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        log.Fatal(err) // deferが実行されない！
    }
    defer db.Close()

    if err := runBatch(db); err != nil {
        log.Fatal(err) // deferが実行されない！
    }
}
```

`log.Fatal`は内部で`os.Exit(1)`を呼ぶため、`defer`が実行されません。DBコネクションが正しくクローズされない可能性があります。

## 実装方法

### 1. 基本的なパターン

```go
func main() {
    exitCode := run()
    os.Exit(exitCode)
}

func run() int {
    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        slog.Error("failed to open database", "error", err)
        return 1
    }
    defer db.Close()

    if err := runBatch(context.Background(), db); err != nil {
        slog.Error("batch failed", "error", err)
        return 1
    }

    slog.Info("batch completed successfully")
    return 0
}
```

### 2. シグナルハンドリングを含むパターン

```go
func main() {
    exitCode := run()
    os.Exit(exitCode)
}

func run() int {
    // コンテキストとキャンセル関数を作成
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // シグナルハンドリング
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        sig := <-sigCh
        slog.Info("received signal", "signal", sig)
        cancel()
    }()

    // DBコネクション
    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        slog.Error("failed to open database", "error", err)
        return 1
    }
    defer db.Close()

    // バッチ実行
    if err := runBatch(ctx, db); err != nil {
        if errors.Is(err, context.Canceled) {
            slog.Info("batch was cancelled")
            return 0 // キャンセルは正常終了扱い
        }
        slog.Error("batch failed", "error", err)
        return 1
    }

    slog.Info("batch completed successfully")
    return 0
}
```

### 3. タイムアウト付きのパターン

Cloud Run Jobにはタスクタイムアウトがありますが、アプリケーション側でもタイムアウトを設定することで、グレースフルにシャットダウンできます。

```go
func run() int {
    // 環境変数からタイムアウトを取得（デフォルト10分）
    timeout := 10 * time.Minute
    if t := os.Getenv("JOB_TIMEOUT"); t != "" {
        if d, err := time.ParseDuration(t); err == nil {
            timeout = d
        }
    }

    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    // シグナルハンドリング
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        select {
        case sig := <-sigCh:
            slog.Info("received signal", "signal", sig)
            cancel()
        case <-ctx.Done():
        }
    }()

    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        slog.Error("failed to open database", "error", err)
        return 1
    }
    defer db.Close()

    if err := runBatch(ctx, db); err != nil {
        if errors.Is(err, context.Canceled) || errors.Is(err, context.DeadlineExceeded) {
            slog.Warn("batch was cancelled or timed out", "error", err)
            return 1 // タイムアウトは失敗扱い
        }
        slog.Error("batch failed", "error", err)
        return 1
    }

    slog.Info("batch completed successfully")
    return 0
}
```

### 4. クリーンアップ処理付きのパターン

```go
func run() int {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // シグナルハンドリング
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        sig := <-sigCh
        slog.Info("received signal, starting graceful shutdown", "signal", sig)
        cancel()
    }()

    // リソース初期化
    db, err := sql.Open("mysql", os.Getenv("DATABASE_URL"))
    if err != nil {
        slog.Error("failed to open database", "error", err)
        return 1
    }

    // クリーンアップ関数を登録
    var cleanups []func() error
    cleanups = append(cleanups, db.Close)

    defer func() {
        slog.Info("running cleanup...")
        for _, cleanup := range cleanups {
            if err := cleanup(); err != nil {
                slog.Error("cleanup failed", "error", err)
            }
        }
        slog.Info("cleanup completed")
    }()

    // 一時ファイルのクリーンアップも登録
    tempDir, err := os.MkdirTemp("", "batch-*")
    if err != nil {
        slog.Error("failed to create temp dir", "error", err)
        return 1
    }
    cleanups = append(cleanups, func() error {
        return os.RemoveAll(tempDir)
    })

    // バッチ実行
    if err := runBatch(ctx, db, tempDir); err != nil {
        slog.Error("batch failed", "error", err)
        return 1
    }

    return 0
}
```

### 5. 実際のバッチ処理例

```go
func runBatch(ctx context.Context, db *sql.DB) error {
    // 処理対象を取得
    items, err := fetchPendingItems(ctx, db)
    if err != nil {
        return fmt.Errorf("failed to fetch items: %w", err)
    }

    slog.Info("processing items", "count", len(items))

    var processed, failed int
    for _, item := range items {
        // コンテキストのキャンセルをチェック
        select {
        case <-ctx.Done():
            slog.Info("batch interrupted",
                "processed", processed,
                "failed", failed,
                "remaining", len(items)-processed-failed,
            )
            return ctx.Err()
        default:
        }

        if err := processItem(ctx, db, item); err != nil {
            slog.Error("failed to process item",
                "item_id", item.ID,
                "error", err,
            )
            failed++
            continue
        }
        processed++
    }

    slog.Info("batch completed",
        "processed", processed,
        "failed", failed,
    )

    // 失敗が多すぎる場合はエラーを返す
    if failed > 0 && float64(failed)/float64(len(items)) > 0.1 {
        return fmt.Errorf("too many failures: %d/%d", failed, len(items))
    }

    return nil
}
```

### 6. Cloud Run Job用のDockerfile

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /batch ./cmd/batch

FROM gcr.io/distroless/static-debian12
COPY --from=builder /batch /batch
ENTRYPOINT ["/batch"]
```

## ポイント

1. **log.Fatalを避ける**: deferが実行されないため、os.Exitを明示的に使う
2. **シグナルハンドリング**: SIGTERMを受けたらグレースフルにシャットダウン
3. **コンテキストの伝播**: キャンセルを各処理に伝える
4. **exit code**: 0が成功、1以上が失敗

## Cloud Run Jobの設定例

```yaml
# job.yaml
apiVersion: run.googleapis.com/v1
kind: Job
metadata:
  name: my-batch-job
spec:
  template:
    spec:
      template:
        spec:
          containers:
            - image: gcr.io/my-project/my-batch:latest
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: database-url
                      key: latest
                - name: JOB_TIMEOUT
                  value: "30m"
          timeoutSeconds: 3600  # 1時間
```

## まとめ

Cloud Run Jobでは適切なexit処理を実装することで：

- ジョブの成功/失敗が正しく判定される
- リソースが確実にクリーンアップされる
- シグナルを受けてもグレースフルにシャットダウンできる

`log.Fatal`の代わりにexit codeを返すパターンを使うことで、安全で信頼性の高いバッチ処理を実装できます。

## 参考資料

- [Cloud Run Jobs](https://cloud.google.com/run/docs/create-jobs)
- [Go signal handling](https://pkg.go.dev/os/signal)
