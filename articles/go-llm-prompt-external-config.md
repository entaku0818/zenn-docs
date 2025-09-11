---
title: "【Go】LLMのプロンプトとモデル設定を外部ファイル化してデプロイなしで調整可能にする"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "openai", "llm", "gcs"]
published: false
---

## この記事は？

OpenAI APIを使ったLLM機能で、プロンプトやモデル設定を外部ファイル（GCS）で管理し、デプロイなしで調整できるようにした実装をまとめます。

## 背景

- プロンプトの調整はトライ＆エラーが必要
- モデルの変更（GPT-4からGPT-4-turboへなど）を迅速に行いたい
- temperatureなどのパラメータを本番環境で微調整したい
- 毎回デプロイするのは時間がかかりすぎる

## 実装方法

### 1. 設定ファイルの形式

GCSに配置する設定ファイル（YAML形式）:

```yaml
# config/llm_config.yaml
version: "1.0"
updated_at: "2024-01-15T10:00:00Z"

default_model: "gpt-4-turbo"
default_temperature: 0.7
default_max_tokens: 2000

prompts:
  summarize:
    system: |
      あなたは優秀な要約アシスタントです。
      与えられたテキストを簡潔に要約してください。
      要点を箇条書きで3-5個にまとめてください。
    model: "gpt-4-turbo"
    temperature: 0.3
    max_tokens: 1000

  translate:
    system: |
      あなたは翻訳者です。
      与えられた日本語を自然な英語に翻訳してください。
    model: "gpt-4"
    temperature: 0.2
    max_tokens: 2000

  chat:
    system: |
      あなたはフレンドリーなアシスタントです。
      ユーザーの質問に丁寧に回答してください。
    model: "gpt-4-turbo"
    temperature: 0.8
    max_tokens: 1500
```

### 2. 設定構造体の定義

```go
import (
    "gopkg.in/yaml.v3"
)

type LLMConfig struct {
    Version           string                   `yaml:"version"`
    UpdatedAt         time.Time                `yaml:"updated_at"`
    DefaultModel      string                   `yaml:"default_model"`
    DefaultTemp       float32                  `yaml:"default_temperature"`
    DefaultMaxTokens  int                      `yaml:"default_max_tokens"`
    Prompts           map[string]*PromptConfig `yaml:"prompts"`
}

type PromptConfig struct {
    System      string  `yaml:"system"`
    Model       string  `yaml:"model"`
    Temperature float32 `yaml:"temperature"`
    MaxTokens   int     `yaml:"max_tokens"`
}

// GetPrompt はプロンプト設定を取得する（デフォルト値をマージ）
func (c *LLMConfig) GetPrompt(name string) (*PromptConfig, error) {
    prompt, ok := c.Prompts[name]
    if !ok {
        return nil, fmt.Errorf("prompt not found: %s", name)
    }

    // デフォルト値をマージ
    result := &PromptConfig{
        System:      prompt.System,
        Model:       prompt.Model,
        Temperature: prompt.Temperature,
        MaxTokens:   prompt.MaxTokens,
    }

    if result.Model == "" {
        result.Model = c.DefaultModel
    }
    if result.Temperature == 0 {
        result.Temperature = c.DefaultTemp
    }
    if result.MaxTokens == 0 {
        result.MaxTokens = c.DefaultMaxTokens
    }

    return result, nil
}
```

### 3. 設定ローダーの実装

```go
type LLMConfigLoader struct {
    gcsClient  *storage.Client
    bucketName string
    objectPath string

    mu       sync.RWMutex
    config   *LLMConfig
    loadedAt time.Time
}

func NewLLMConfigLoader(ctx context.Context, bucketName, objectPath string) (*LLMConfigLoader, error) {
    client, err := storage.NewClient(ctx)
    if err != nil {
        return nil, fmt.Errorf("failed to create gcs client: %w", err)
    }

    loader := &LLMConfigLoader{
        gcsClient:  client,
        bucketName: bucketName,
        objectPath: objectPath,
    }

    // 初回ロード
    if err := loader.reload(ctx); err != nil {
        return nil, fmt.Errorf("failed to load config: %w", err)
    }

    return loader, nil
}

// reload はGCSから設定を読み込む
func (l *LLMConfigLoader) reload(ctx context.Context) error {
    bucket := l.gcsClient.Bucket(l.bucketName)
    obj := bucket.Object(l.objectPath)

    reader, err := obj.NewReader(ctx)
    if err != nil {
        return fmt.Errorf("failed to read object: %w", err)
    }
    defer reader.Close()

    var config LLMConfig
    if err := yaml.NewDecoder(reader).Decode(&config); err != nil {
        return fmt.Errorf("failed to decode config: %w", err)
    }

    l.mu.Lock()
    l.config = &config
    l.loadedAt = time.Now()
    l.mu.Unlock()

    slog.Info("reloaded LLM config",
        "version", config.Version,
        "prompt_count", len(config.Prompts),
    )
    return nil
}

// GetConfig は現在の設定を取得する
func (l *LLMConfigLoader) GetConfig() *LLMConfig {
    l.mu.RLock()
    defer l.mu.RUnlock()
    return l.config
}

// StartAutoReload は定期的に設定をリロードする
func (l *LLMConfigLoader) StartAutoReload(ctx context.Context, interval time.Duration) {
    ticker := time.NewTicker(interval)
    go func() {
        for {
            select {
            case <-ctx.Done():
                ticker.Stop()
                return
            case <-ticker.C:
                if err := l.reload(ctx); err != nil {
                    slog.Error("failed to reload LLM config", "error", err)
                }
            }
        }
    }()
}
```

### 4. LLMクライアントとの統合

```go
import (
    "github.com/sashabaranov/go-openai"
)

type LLMService struct {
    openaiClient *openai.Client
    configLoader *LLMConfigLoader
}

func NewLLMService(ctx context.Context, openaiKey, bucket, configPath string) (*LLMService, error) {
    loader, err := NewLLMConfigLoader(ctx, bucket, configPath)
    if err != nil {
        return nil, err
    }

    // 5分ごとに設定をリロード
    loader.StartAutoReload(ctx, 5*time.Minute)

    return &LLMService{
        openaiClient: openai.NewClient(openaiKey),
        configLoader: loader,
    }, nil
}

// Chat は指定したプロンプトでLLMを呼び出す
func (s *LLMService) Chat(ctx context.Context, promptName, userMessage string) (string, error) {
    config := s.configLoader.GetConfig()
    prompt, err := config.GetPrompt(promptName)
    if err != nil {
        return "", err
    }

    resp, err := s.openaiClient.CreateChatCompletion(
        ctx,
        openai.ChatCompletionRequest{
            Model: prompt.Model,
            Messages: []openai.ChatCompletionMessage{
                {
                    Role:    openai.ChatMessageRoleSystem,
                    Content: prompt.System,
                },
                {
                    Role:    openai.ChatMessageRoleUser,
                    Content: userMessage,
                },
            },
            Temperature: prompt.Temperature,
            MaxTokens:   prompt.MaxTokens,
        },
    )
    if err != nil {
        return "", fmt.Errorf("failed to create chat completion: %w", err)
    }

    if len(resp.Choices) == 0 {
        return "", errors.New("no response from LLM")
    }

    return resp.Choices[0].Message.Content, nil
}
```

### 5. 使用例

```go
func main() {
    ctx := context.Background()

    service, err := NewLLMService(
        ctx,
        os.Getenv("OPENAI_API_KEY"),
        os.Getenv("GCS_BUCKET"),
        "config/llm_config.yaml",
    )
    if err != nil {
        log.Fatal(err)
    }

    // 要約を実行
    summary, err := service.Chat(ctx, "summarize", "長いテキスト...")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Summary:", summary)

    // 翻訳を実行
    translation, err := service.Chat(ctx, "translate", "こんにちは")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("Translation:", translation)
}
```

### 6. 管理画面からの更新

```go
// UpdateLLMConfig はLLM設定を更新する
func (h *AdminHandler) UpdateLLMConfig(w http.ResponseWriter, r *http.Request) {
    var config LLMConfig
    if err := yaml.NewDecoder(r.Body).Decode(&config); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    config.UpdatedAt = time.Now()

    // バリデーション
    for name, prompt := range config.Prompts {
        if prompt.System == "" {
            http.Error(w, fmt.Sprintf("prompt %s: system is required", name), http.StatusBadRequest)
            return
        }
    }

    // GCSに保存
    ctx := r.Context()
    bucket := h.gcsClient.Bucket(h.bucketName)
    obj := bucket.Object("config/llm_config.yaml")
    writer := obj.NewWriter(ctx)
    if err := yaml.NewEncoder(writer).Encode(config); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    if err := writer.Close(); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // 即座にリロード
    if err := h.llmConfigLoader.reload(ctx); err != nil {
        slog.Error("failed to reload config", "error", err)
    }

    w.WriteHeader(http.StatusOK)
}
```

## ポイント

1. **YAML形式**: プロンプトは複数行になりがちなのでYAMLが扱いやすい
2. **デフォルト値のマージ**: 個別設定がない場合はデフォルト値を使用
3. **プロンプト名による呼び出し**: 用途ごとにプロンプトを分離
4. **バリデーション**: 更新時に必須項目をチェック

## まとめ

LLMの設定を外部ファイル化することで、以下のメリットがあります：

- プロンプトの調整がデプロイなしで可能
- モデルの切り替えが即座に反映
- temperatureなどのパラメータを本番で微調整可能
- 用途ごとにプロンプトを管理しやすい

LLMを使った機能開発では、プロンプトの調整が頻繁に発生するため、この仕組みは開発効率を大きく向上させます。

## 参考資料

- [OpenAI Go Client](https://github.com/sashabaranov/go-openai)
- [Cloud Storage Go Client](https://pkg.go.dev/cloud.google.com/go/storage)
