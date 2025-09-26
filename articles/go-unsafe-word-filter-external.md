---
title: "【Go】禁止ワードリストを外部ファイルで管理してデプロイなしで更新可能にする"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "gcs", "moderation"]
published: false
---

## この記事は？

ユーザー投稿コンテンツの禁止ワードフィルタリングで、禁止ワードリストを外部ファイル（GCS）で管理し、デプロイなしで更新できるようにした実装をまとめます。

## 背景

- 禁止ワードをコードにハードコーディングするとデプロイが必要
- 新しい禁止ワードを迅速に追加したい
- 運用チームがエンジニアに依頼せずに更新できるようにしたい

## 実装方法

### 1. 禁止ワードファイルの形式

GCSに配置する禁止ワードファイル（JSON形式）:

```json
{
  "words": [
    "禁止ワード1",
    "禁止ワード2",
    "NGワード"
  ],
  "patterns": [
    "正規表現パターン.*",
    "\\d{3}-\\d{4}-\\d{4}"
  ],
  "updated_at": "2024-01-15T10:00:00Z"
}
```

### 2. 禁止ワードフィルターの実装

```go
import (
    "context"
    "encoding/json"
    "regexp"
    "sync"
    "time"

    "cloud.google.com/go/storage"
)

type UnsafeWordFilter struct {
    gcsClient  *storage.Client
    bucketName string
    objectPath string

    mu       sync.RWMutex
    words    map[string]struct{}
    patterns []*regexp.Regexp
    loadedAt time.Time
}

type UnsafeWordConfig struct {
    Words     []string  `json:"words"`
    Patterns  []string  `json:"patterns"`
    UpdatedAt time.Time `json:"updated_at"`
}

func NewUnsafeWordFilter(ctx context.Context, bucketName, objectPath string) (*UnsafeWordFilter, error) {
    client, err := storage.NewClient(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to create gcs client: %w", err)
    }

    f := &UnsafeWordFilter{
        gcsClient:  client,
        bucketName: bucketName,
        objectPath: objectPath,
        words:      make(map[string]struct{}),
    }

    // 初回ロード
    if err := f.reload(ctx); err != nil {
        return nil, fmt.Errorf("failed to load unsafe words: %w", err)
    }

    return f, nil
}
```

### 3. 禁止ワードの読み込み

```go
// reload はGCSから禁止ワードリストを読み込む
func (f *UnsafeWordFilter) reload(ctx context.Context) error {
    bucket := f.gcsClient.Bucket(f.bucketName)
    obj := bucket.Object(f.objectPath)

    reader, err := obj.NewReader(ctx)
    if err != nil {
        return fmt.Errorf("failed to read object: %w", err)
    }
    defer reader.Close()

    var config UnsafeWordConfig
    if err := json.NewDecoder(reader).Decode(&config); err != nil {
        return fmt.Errorf("failed to decode config: %w", err)
    }

    // 禁止ワードをmapに変換
    words := make(map[string]struct{}, len(config.Words))
    for _, word := range config.Words {
        words[strings.ToLower(word)] = struct{}{}
    }

    // 正規表現パターンをコンパイル
    patterns := make([]*regexp.Regexp, 0, len(config.Patterns))
    for _, pattern := range config.Patterns {
        re, err := regexp.Compile(pattern)
        if err != nil {
            slog.Warn("failed to compile pattern", "pattern", pattern, "error", err)
            continue
        }
        patterns = append(patterns, re)
    }

    // 排他制御してデータを更新
    f.mu.Lock()
    f.words = words
    f.patterns = patterns
    f.loadedAt = time.Now()
    f.mu.Unlock()

    slog.Info("reloaded unsafe words",
        "word_count", len(words),
        "pattern_count", len(patterns),
    )
    return nil
}
```

### 4. 禁止ワードのチェック

```go
// Contains は文字列に禁止ワードが含まれているかチェックする
func (f *UnsafeWordFilter) Contains(text string) bool {
    f.mu.RLock()
    defer f.mu.RUnlock()

    lowerText := strings.ToLower(text)

    // 単語マッチ
    for word := range f.words {
        if strings.Contains(lowerText, word) {
            return true
        }
    }

    // 正規表現マッチ
    for _, pattern := range f.patterns {
        if pattern.MatchString(text) {
            return true
        }
    }

    return false
}

// Filter は禁止ワードを指定文字で置換する
func (f *UnsafeWordFilter) Filter(text string, replacement string) string {
    f.mu.RLock()
    defer f.mu.RUnlock()

    result := text

    // 単語を置換
    for word := range f.words {
        // 大文字小文字を区別しない置換
        re := regexp.MustCompile("(?i)" + regexp.QuoteMeta(word))
        result = re.ReplaceAllString(result, replacement)
    }

    // パターンを置換
    for _, pattern := range f.patterns {
        result = pattern.ReplaceAllString(result, replacement)
    }

    return result
}
```

### 5. 定期的なリロード

```go
// StartAutoReload は定期的に禁止ワードリストをリロードする
func (f *UnsafeWordFilter) StartAutoReload(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    go func() {
        for {
            select {
            case <-ctx.Done():
                ticker.Stop()
                return
            case <-ticker.C:
                if err := f.reload(ctx); err != nil {
                    slog.Error("failed to reload unsafe words", "error", err)
                }
            }
        }
    }()
}

// ForceReload は手動でリロードを実行する
func (f *UnsafeWordFilter) ForceReload(ctx context.Context) error {
    return f.reload(ctx)
}
```

### 6. 使用例

```go
func main() {
    ctx := context.Background()

    filter, err := NewUnsafeWordFilter(
        ctx,
        os.Getenv("GCS_BUCKET"),
        "config/unsafe_words.json",
    )
    if err != nil {
        log.Fatal(err)
    }

    // 5分ごとに自動リロード
    filter.StartAutoReload(ctx, 5*time.Minute)

    // APIハンドラ内で使用
    text := "ユーザーの投稿内容"
    if filter.Contains(text) {
        // 禁止ワードが含まれている
        slog.Warn("unsafe word detected", "text", text)
        return errors.New("inappropriate content")
    }

    // または禁止ワードを伏せ字にする
    filtered := filter.Filter(text, "***")
}
```

### 7. 管理画面からの更新用API

```go
// UpdateUnsafeWords は禁止ワードリストを更新する
func (h *AdminHandler) UpdateUnsafeWords(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Words    []string `json:"words"`
        Patterns []string `json:"patterns"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    config := UnsafeWordConfig{
        Words:     req.Words,
        Patterns:  req.Patterns,
        UpdatedAt: time.Now(),
    }

    // GCSに保存
    ctx := r.Context()
    bucket := h.gcsClient.Bucket(h.bucketName)
    obj := bucket.Object("config/unsafe_words.json")
    writer := obj.NewWriter(ctx)
    if err := json.NewEncoder(writer).Encode(config); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if err := writer.Close(); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // 即座にリロード
    if err := h.filter.ForceReload(ctx); err != nil {
        slog.Error("failed to reload filter", "error", err)
    }

    w.WriteHeader(http.StatusOK)
}
```

## ポイント

1. **RWMutex**: 読み込みは並行可能、書き込み時のみ排他制御
2. **定期リロード**: バックグラウンドで自動更新
3. **強制リロード**: 管理画面から即座に反映可能
4. **パターンマッチ**: 単純な単語だけでなく正規表現も対応

## まとめ

禁止ワードリストを外部ファイル化することで、以下のメリットがあります：

- デプロイなしで禁止ワードを追加・削除可能
- 運用チームが自律的に更新できる
- 正規表現パターンで柔軟なフィルタリングが可能

GCSを使用することで、バージョン管理や権限管理も容易になります。

## 参考資料

- [Cloud Storage Go Client](https://pkg.go.dev/cloud.google.com/go/storage)
- [Goの正規表現パッケージ](https://pkg.go.dev/regexp)
