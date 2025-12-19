---
title: "【Go】AIの連続発言を防止するロジックの実装"
emoji: "🤖"
type: "tech"
topics: ["Go", "AI", "ChatBot", "バックエンド"]
published: false
---

# この記事は？

チャットアプリにAIアバターを導入した際、AIが連続で発言してしまい不自然になる問題が発生しました。この記事では、最終メッセージを確認してAI応答を制御するロジックを解説します。

## 問題の状況

AIトリガーAPIが連続で呼ばれると、AIが何度も連続で発言してしまいます。

```
ユーザーA: こんにちは
AI(ひなた): やっほー！
AI(ひなた): 元気？  ← AIが連続発言（不自然）
AI(ひなた): 何してるの？ ← さらに連続
```

## 解決方法

最終メッセージがAIからの発言かどうかをチェックし、AIからの場合はスキップします。

### 実装

```go
// usecase/ai_trigger.go

type AITriggerUseCase struct {
    talkMessageRepo TalkMessageRepository
    aiService       AIService
}

func (uc *AITriggerUseCase) TriggerAIResponse(
    ctx context.Context,
    talkRoomID string,
) error {
    // 最新のメッセージを取得
    lastMessage, err := uc.talkMessageRepo.GetLatestMessage(ctx, talkRoomID)
    if err != nil {
        return fmt.Errorf("failed to get latest message: %w", err)
    }

    // 最終メッセージがAIからの場合はスキップ
    if lastMessage != nil && lastMessage.IsFromAI() {
        log.Printf("Skipping AI response: last message was from AI (user_id: %s)",
            lastMessage.UserID)
        return nil
    }

    // AI応答を生成
    return uc.aiService.GenerateResponse(ctx, talkRoomID)
}
```

### リポジトリ層

```go
// repository/talk_message_repository.go

func (r *TalkMessageRepository) GetLatestMessage(
    ctx context.Context,
    talkRoomID string,
) (*model.TalkMessage, error) {
    var message model.TalkMessage

    err := r.db.NewSelect().
        Model(&message).
        Where("talk_room_id = ?", talkRoomID).
        Order("created_at DESC").
        Limit(1).
        Scan(ctx)

    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil // メッセージがない場合
        }
        return nil, err
    }

    return &message, nil
}
```

### モデル層

```go
// model/talk_message.go

type TalkMessage struct {
    ID         string    `bun:"id,pk"`
    TalkRoomID string    `bun:"talk_room_id"`
    UserID     string    `bun:"user_id"`
    Content    string    `bun:"content"`
    CreatedAt  time.Time `bun:"created_at"`
}

// IsFromAI はこのメッセージがAIユーザーからのものかを判定
func (m *TalkMessage) IsFromAI() bool {
    return isAIUserID(m.UserID)
}

// AIユーザーIDのプレフィックスで判定
func isAIUserID(userID string) bool {
    return strings.HasPrefix(userID, "ai_")
}
```

## より堅牢な実装

### 複数AIがいる場合の対応

複数のAIアバターがいる場合、どのAIが発言したかも考慮します。

```go
func (uc *AITriggerUseCase) TriggerAIResponse(
    ctx context.Context,
    talkRoomID string,
    triggerAIUserID string,
) error {
    lastMessage, err := uc.talkMessageRepo.GetLatestMessage(ctx, talkRoomID)
    if err != nil {
        return err
    }

    // 最終メッセージが同じAIからの場合のみスキップ
    if lastMessage != nil &&
        lastMessage.UserID == triggerAIUserID {
        return nil
    }

    return uc.aiService.GenerateResponse(ctx, talkRoomID, triggerAIUserID)
}
```

### 時間ベースの制限を追加

短時間での連続発言も防止します。

```go
const minIntervalBetweenAIMessages = 3 * time.Second

func (uc *AITriggerUseCase) TriggerAIResponse(
    ctx context.Context,
    talkRoomID string,
) error {
    lastMessage, err := uc.talkMessageRepo.GetLatestMessage(ctx, talkRoomID)
    if err != nil {
        return err
    }

    if lastMessage != nil {
        // AIからの発言かつ一定時間内の場合はスキップ
        if lastMessage.IsFromAI() {
            elapsed := time.Since(lastMessage.CreatedAt)
            if elapsed < minIntervalBetweenAIMessages {
                log.Printf("Skipping: AI message too recent (%v ago)", elapsed)
                return nil
            }
        }
    }

    return uc.aiService.GenerateResponse(ctx, talkRoomID)
}
```

### 最後のN件をチェック

より自然な会話のため、最後の数件をチェックして連続発言を防止します。

```go
func (uc *AITriggerUseCase) ShouldAIRespond(
    ctx context.Context,
    talkRoomID string,
) (bool, error) {
    // 最後の3件を取得
    messages, err := uc.talkMessageRepo.GetRecentMessages(ctx, talkRoomID, 3)
    if err != nil {
        return false, err
    }

    if len(messages) == 0 {
        return true, nil // メッセージがなければ応答OK
    }

    // 最後のメッセージがAIからなら応答しない
    if messages[0].IsFromAI() {
        return false, nil
    }

    // 連続でAI発言が続いていたらスキップ
    aiMessageCount := 0
    for _, msg := range messages {
        if msg.IsFromAI() {
            aiMessageCount++
        }
    }

    // 直近3件のうち2件以上がAIならスキップ
    if aiMessageCount >= 2 {
        return false, nil
    }

    return true, nil
}
```

## APIハンドラー

```go
// handler/ai_trigger_handler.go

func (h *AITriggerHandler) HandleTrigger(c echo.Context) error {
    talkRoomID := c.Param("talkRoomId")

    err := h.useCase.TriggerAIResponse(c.Request().Context(), talkRoomID)
    if err != nil {
        return c.JSON(http.StatusInternalServerError, map[string]string{
            "error": err.Error(),
        })
    }

    return c.JSON(http.StatusOK, map[string]string{
        "status": "ok",
    })
}
```

## テスト

```go
func TestAITriggerUseCase_TriggerAIResponse(t *testing.T) {
    tests := []struct {
        name           string
        lastMessage    *model.TalkMessage
        expectGenerate bool
    }{
        {
            name:           "メッセージなし→AI応答する",
            lastMessage:    nil,
            expectGenerate: true,
        },
        {
            name: "人間のメッセージ→AI応答する",
            lastMessage: &model.TalkMessage{
                UserID: "user_123",
            },
            expectGenerate: true,
        },
        {
            name: "AIのメッセージ→スキップ",
            lastMessage: &model.TalkMessage{
                UserID: "ai_hinata",
            },
            expectGenerate: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            mockRepo := &MockTalkMessageRepo{
                lastMessage: tt.lastMessage,
            }
            mockAI := &MockAIService{}

            uc := &AITriggerUseCase{
                talkMessageRepo: mockRepo,
                aiService:       mockAI,
            }

            _ = uc.TriggerAIResponse(context.Background(), "room_1")

            if tt.expectGenerate && !mockAI.generateCalled {
                t.Error("Expected AI to generate response")
            }
            if !tt.expectGenerate && mockAI.generateCalled {
                t.Error("Expected AI NOT to generate response")
            }
        })
    }
}
```

## まとめ

- 最終メッセージを確認してAI連続発言を防止
- AIユーザーIDはプレフィックスで判定
- 時間ベースの制限も併用するとより自然に
- 最後のN件をチェックすることでより精度を向上

## 参考資料

- [OpenAI - Chat Completions API](https://platform.openai.com/docs/guides/chat)
