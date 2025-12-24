---
title: "【Go】環境変数でフィーチャーフラグを制御する"
emoji: "🚩"
type: "tech"
topics: ["Go", "環境変数", "フィーチャーフラグ", "設定"]
published: false
---

# この記事は？

新機能を段階的にリリースしたり、環境ごとに機能のON/OFFを切り替えたい場合、環境変数ベースのフィーチャーフラグが便利です。この記事では、Goでの実装パターンを解説します。

## 基本的な実装

### 1. 設定構造体の定義

```go
// config/config.go
package config

import (
    "os"
    "strconv"
)

type Config struct {
    // 機能フラグ
    Features FeatureFlags
}

type FeatureFlags struct {
    AIAutoJoinEnabled     bool
    AIAutoResponseEnabled bool
    NewUIEnabled          bool
    DebugModeEnabled      bool
}

func Load() *Config {
    return &Config{
        Features: FeatureFlags{
            AIAutoJoinEnabled:     getBoolEnv("AI_AUTO_JOIN_ENABLED", false),
            AIAutoResponseEnabled: getBoolEnv("AI_AUTO_RESPONSE_ENABLED", false),
            NewUIEnabled:          getBoolEnv("NEW_UI_ENABLED", false),
            DebugModeEnabled:      getBoolEnv("DEBUG_MODE_ENABLED", false),
        },
    }
}

func getBoolEnv(key string, defaultValue bool) bool {
    val := os.Getenv(key)
    if val == "" {
        return defaultValue
    }

    boolVal, err := strconv.ParseBool(val)
    if err != nil {
        return defaultValue
    }

    return boolVal
}
```

### 2. 機能フラグの使用

```go
// usecase/ai_avatar_usecase.go

type AIAvatarUseCase struct {
    config *config.Config
    repo   TalkRoomRepository
}

func (uc *AIAvatarUseCase) OnUserJoinRoom(
    ctx context.Context,
    roomID string,
    userID string,
) error {
    // フィーチャーフラグで制御
    if !uc.config.Features.AIAutoJoinEnabled {
        return nil
    }

    // AI自動参加処理
    return uc.addAIToRoom(ctx, roomID)
}

func (uc *AIAvatarUseCase) OnNewMessage(
    ctx context.Context,
    roomID string,
    message string,
) error {
    if !uc.config.Features.AIAutoResponseEnabled {
        return nil
    }

    // AI自動応答処理
    return uc.triggerAIResponse(ctx, roomID, message)
}
```

## より堅牢な実装

### envconfigを使用

```go
// config/config.go
package config

import (
    "github.com/kelseyhightower/envconfig"
)

type Config struct {
    Environment string       `envconfig:"ENVIRONMENT" default:"development"`
    Features    FeatureFlags `envconfig:"FEATURE"`
}

type FeatureFlags struct {
    AIAutoJoin     bool `envconfig:"AI_AUTO_JOIN" default:"false"`
    AIAutoResponse bool `envconfig:"AI_AUTO_RESPONSE" default:"false"`
    NewUI          bool `envconfig:"NEW_UI" default:"false"`
    DebugMode      bool `envconfig:"DEBUG_MODE" default:"false"`
}

func Load() (*Config, error) {
    var cfg Config
    if err := envconfig.Process("", &cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}
```

環境変数の設定例：

```bash
export FEATURE_AI_AUTO_JOIN=true
export FEATURE_AI_AUTO_RESPONSE=false
export FEATURE_NEW_UI=true
```

### 環境別デフォルト値

```go
type FeatureFlags struct {
    AIAutoJoin     bool
    AIAutoResponse bool
    DebugMode      bool
}

func LoadFeatureFlags(environment string) FeatureFlags {
    // 環境別のデフォルト値
    defaults := map[string]FeatureFlags{
        "development": {
            AIAutoJoin:     true,
            AIAutoResponse: true,
            DebugMode:      true,
        },
        "staging": {
            AIAutoJoin:     true,
            AIAutoResponse: true,
            DebugMode:      false,
        },
        "production": {
            AIAutoJoin:     false,
            AIAutoResponse: false,
            DebugMode:      false,
        },
    }

    base := defaults[environment]

    // 環境変数で上書き
    return FeatureFlags{
        AIAutoJoin:     getBoolEnvWithDefault("AI_AUTO_JOIN_ENABLED", base.AIAutoJoin),
        AIAutoResponse: getBoolEnvWithDefault("AI_AUTO_RESPONSE_ENABLED", base.AIAutoResponse),
        DebugMode:      getBoolEnvWithDefault("DEBUG_MODE_ENABLED", base.DebugMode),
    }
}

func getBoolEnvWithDefault(key string, defaultValue bool) bool {
    val := os.Getenv(key)
    if val == "" {
        return defaultValue
    }
    boolVal, _ := strconv.ParseBool(val)
    return boolVal
}
```

## フィーチャーフラグサービス

```go
// service/feature_service.go
package service

type FeatureService struct {
    flags FeatureFlags
}

func NewFeatureService(flags FeatureFlags) *FeatureService {
    return &FeatureService{flags: flags}
}

func (s *FeatureService) IsEnabled(feature string) bool {
    switch feature {
    case "ai_auto_join":
        return s.flags.AIAutoJoin
    case "ai_auto_response":
        return s.flags.AIAutoResponse
    case "new_ui":
        return s.flags.NewUI
    case "debug_mode":
        return s.flags.DebugMode
    default:
        return false
    }
}

// フラグ一覧を取得（デバッグ用）
func (s *FeatureService) GetAllFlags() map[string]bool {
    return map[string]bool{
        "ai_auto_join":     s.flags.AIAutoJoin,
        "ai_auto_response": s.flags.AIAutoResponse,
        "new_ui":           s.flags.NewUI,
        "debug_mode":       s.flags.DebugMode,
    }
}
```

### APIエンドポイントでフラグを確認

```go
// handler/feature_handler.go

func (h *FeatureHandler) GetFeatureFlags(c echo.Context) error {
    // 開発/ステージング環境のみ
    if h.config.Environment == "production" {
        return c.JSON(http.StatusForbidden, nil)
    }

    flags := h.featureService.GetAllFlags()
    return c.JSON(http.StatusOK, flags)
}
```

## ミドルウェアでの使用

```go
// middleware/feature_middleware.go

func RequireFeature(featureService *service.FeatureService, feature string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            if !featureService.IsEnabled(feature) {
                return c.JSON(http.StatusNotFound, map[string]string{
                    "error": "Feature not available",
                })
            }
            return next(c)
        }
    }
}
```

使用例：

```go
// router/router.go

aiGroup := e.Group("/api/ai")
aiGroup.Use(middleware.RequireFeature(featureService, "ai_auto_response"))
aiGroup.POST("/trigger", aiHandler.TriggerResponse)
```

## テストでのフラグ制御

```go
func TestAIAutoJoin_WhenEnabled(t *testing.T) {
    cfg := &config.Config{
        Features: config.FeatureFlags{
            AIAutoJoin: true,
        },
    }

    uc := NewAIAvatarUseCase(cfg, mockRepo)
    err := uc.OnUserJoinRoom(ctx, "room1", "user1")

    assert.NoError(t, err)
    assert.True(t, mockRepo.AIAdded)
}

func TestAIAutoJoin_WhenDisabled(t *testing.T) {
    cfg := &config.Config{
        Features: config.FeatureFlags{
            AIAutoJoin: false,
        },
    }

    uc := NewAIAvatarUseCase(cfg, mockRepo)
    err := uc.OnUserJoinRoom(ctx, "room1", "user1")

    assert.NoError(t, err)
    assert.False(t, mockRepo.AIAdded)
}
```

## Docker Composeでの設定

```yaml
# docker-compose.yml
services:
  api:
    environment:
      - ENVIRONMENT=development
      - FEATURE_AI_AUTO_JOIN=true
      - FEATURE_AI_AUTO_RESPONSE=true
      - FEATURE_DEBUG_MODE=true
```

## Cloud Runでの設定

```yaml
# cloudbuild.yaml
steps:
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-service'
      - '--set-env-vars'
      - 'FEATURE_AI_AUTO_JOIN=false,FEATURE_AI_AUTO_RESPONSE=true'
```

## まとめ

- 環境変数でフィーチャーフラグを制御
- envconfigで簡潔に設定を読み込み
- 環境別のデフォルト値を設定
- ミドルウェアでエンドポイント単位の制御も可能
- テストではフラグを直接設定して検証

## 参考資料

- [kelseyhightower/envconfig](https://github.com/kelseyhightower/envconfig)
- [Feature Flags - Martin Fowler](https://martinfowler.com/articles/feature-toggles.html)
