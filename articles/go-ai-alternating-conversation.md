---
title: "【Go】AI同士の交互会話を実装する"
emoji: "🗣️"
type: "tech"
topics: ["Go", "AI", "ChatBot", "OpenAI"]
published: false
---

# この記事は？

複数のAIアバターが自然に会話を続ける機能の実装方法を解説します。AIが一方的に話し続けるのではなく、交互に発言することでより自然な会話を実現します。

## 要件

- 複数のAIアバターが交互に発言
- 連続発言を防止
- 会話のコンテキストを維持
- 自然な会話の流れを実現

## 実装

### 1. AIアバターの定義

```go
// model/ai_avatar.go
package model

type AIAvatar struct {
    ID          string `json:"id"`
    CharacterID string `json:"character_id"`
    Name        string `json:"name"`
    Personality string `json:"personality"`
}

var AvailableAvatars = []AIAvatar{
    {
        ID:          "ai_hinata",
        CharacterID: "hinata",
        Name:        "ひなた",
        Personality: "明るくて元気な女の子。絵文字をよく使う。",
    },
    {
        ID:          "ai_moco",
        CharacterID: "moco",
        Name:        "もこ",
        Personality: "おっとりした癒し系。ゆっくり話す。",
    },
}
```

### 2. 会話サービス

```go
// service/ai_conversation_service.go
package service

import (
    "context"
    "fmt"

    "myapp/model"
)

type AIConversationService struct {
    messageRepo MessageRepository
    aiClient    AIClient
}

func NewAIConversationService(
    messageRepo MessageRepository,
    aiClient AIClient,
) *AIConversationService {
    return &AIConversationService{
        messageRepo: messageRepo,
        aiClient:    aiClient,
    }
}

// GenerateAlternatingResponse は交互会話のレスポンスを生成
func (s *AIConversationService) GenerateAlternatingResponse(
    ctx context.Context,
    talkRoomID string,
) error {
    // 最新のメッセージを取得
    messages, err := s.messageRepo.GetRecentMessages(ctx, talkRoomID, 10)
    if err != nil {
        return err
    }

    // 次に発言すべきAIを決定
    nextAI := s.determineNextAI(messages)
    if nextAI == nil {
        return nil // 発言すべきAIがいない
    }

    // AIの応答を生成
    response, err := s.generateResponse(ctx, talkRoomID, nextAI, messages)
    if err != nil {
        return err
    }

    // メッセージを保存
    return s.messageRepo.Create(ctx, &model.Message{
        TalkRoomID: talkRoomID,
        UserID:     nextAI.ID,
        Content:    response,
    })
}

// determineNextAI は次に発言すべきAIを決定
func (s *AIConversationService) determineNextAI(
    messages []model.Message,
) *model.AIAvatar {
    if len(messages) == 0 {
        // メッセージがなければ最初のAI
        return &model.AvailableAvatars[0]
    }

    lastMessage := messages[0]

    // 最後のメッセージがAIでなければ、ランダムにAIを選択
    if !isAIUser(lastMessage.UserID) {
        return &model.AvailableAvatars[0]
    }

    // 最後に発言したAIとは別のAIを選択
    for _, avatar := range model.AvailableAvatars {
        if avatar.ID != lastMessage.UserID {
            return &avatar
        }
    }

    return nil // 他のAIがいない場合はスキップ
}
```

### 3. プロンプト構築

```go
// service/prompt_builder.go
package service

import (
    "fmt"
    "strings"

    "myapp/model"
)

func (s *AIConversationService) buildPrompt(
    roomTitle string,
    avatar *model.AIAvatar,
    messages []model.Message,
) string {
    var sb strings.Builder

    // システムプロンプト
    sb.WriteString(fmt.Sprintf(`あなたは「%s」というキャラクターです。
性格: %s

以下のトークルーム「%s」で会話しています。
`, avatar.Name, avatar.Personality, roomTitle))

    // 会話履歴
    sb.WriteString("\n## 会話履歴\n")
    for i := len(messages) - 1; i >= 0; i-- {
        msg := messages[i]
        name := getUserName(msg.UserID)
        sb.WriteString(fmt.Sprintf("%s: %s\n", name, msg.Content))
    }

    // 指示
    sb.WriteString(fmt.Sprintf(`
## 指示
- %sとして1回だけ返答してください
- 前の発言に自然に反応してください
- 絵文字は1-2個まで
- 30文字以内で返答
`, avatar.Name))

    return sb.String()
}

func getUserName(userID string) string {
    for _, avatar := range model.AvailableAvatars {
        if avatar.ID == userID {
            return avatar.Name
        }
    }
    return "ユーザー"
}
```

### 4. OpenAI連携

```go
// client/openai_client.go
package client

import (
    "context"
    "fmt"

    "github.com/sashabaranov/go-openai"
)

type OpenAIClient struct {
    client *openai.Client
}

func NewOpenAIClient(apiKey string) *OpenAIClient {
    return &OpenAIClient{
        client: openai.NewClient(apiKey),
    }
}

func (c *OpenAIClient) GenerateResponse(
    ctx context.Context,
    systemPrompt string,
) (string, error) {
    resp, err := c.client.CreateChatCompletion(
        ctx,
        openai.ChatCompletionRequest{
            Model: openai.GPT4,
            Messages: []openai.ChatCompletionMessage{
                {
                    Role:    openai.ChatMessageRoleSystem,
                    Content: systemPrompt,
                },
            },
            MaxTokens:   100,
            Temperature: 0.8,
        },
    )

    if err != nil {
        return "", fmt.Errorf("failed to create completion: %w", err)
    }

    if len(resp.Choices) == 0 {
        return "", fmt.Errorf("no choices returned")
    }

    return resp.Choices[0].Message.Content, nil
}
```

### 5. バッチ処理

定期的にAI会話を実行するバッチ：

```go
// batch/ai_conversation_batch.go
package batch

import (
    "context"
    "log"
    "time"

    "myapp/service"
)

type AIConversationBatch struct {
    conversationService *service.AIConversationService
    roomRepo           RoomRepository
}

func (b *AIConversationBatch) Run(ctx context.Context) error {
    // アクティブなルームを取得
    rooms, err := b.roomRepo.GetActiveRooms(ctx)
    if err != nil {
        return err
    }

    for _, room := range rooms {
        // 各ルームで1回ずつ会話を進める
        if err := b.processRoom(ctx, room.ID); err != nil {
            log.Printf("Failed to process room %s: %v", room.ID, err)
            continue
        }

        // レート制限対策
        time.Sleep(2 * time.Second)
    }

    return nil
}

func (b *AIConversationBatch) processRoom(
    ctx context.Context,
    roomID string,
) error {
    // 5回交互に会話
    for i := 0; i < 5; i++ {
        err := b.conversationService.GenerateAlternatingResponse(ctx, roomID)
        if err != nil {
            return err
        }
        time.Sleep(3 * time.Second) // 自然な会話の間隔
    }

    return nil
}
```

### 6. 並行処理対応

```go
import "golang.org/x/sync/errgroup"

func (b *AIConversationBatch) RunParallel(ctx context.Context) error {
    rooms, err := b.roomRepo.GetActiveRooms(ctx)
    if err != nil {
        return err
    }

    g, ctx := errgroup.WithContext(ctx)

    // 最大5並列
    semaphore := make(chan struct{}, 5)

    for _, room := range rooms {
        room := room
        g.Go(func() error {
            semaphore <- struct{}{}
            defer func() { <-semaphore }()

            return b.processRoom(ctx, room.ID)
        })
    }

    return g.Wait()
}
```

## 会話の自然さを向上させるテクニック

### 会話のバリエーション

```go
func (s *AIConversationService) buildPromptWithVariation(
    avatar *model.AIAvatar,
    messages []model.Message,
    variation string,
) string {
    variations := map[string]string{
        "question":   "質問を投げかけてください",
        "agreement":  "相手に同意しながら話を広げてください",
        "topic":      "新しい話題を提案してください",
        "reaction":   "リアクションを返してください",
    }

    instruction := variations[variation]
    // プロンプトに指示を追加
    return fmt.Sprintf("%s\n\n追加指示: %s", basePrompt, instruction)
}
```

### 感情分析による応答調整

```go
func (s *AIConversationService) adjustResponseBySentiment(
    lastMessage string,
) string {
    // 簡易的な感情分析
    if containsPositiveWords(lastMessage) {
        return "ポジティブに反応してください"
    }
    if containsNegativeWords(lastMessage) {
        return "共感しながら励ましてください"
    }
    return "自然に会話を続けてください"
}
```

## まとめ

- 最終発言者を確認して交互に発言
- キャラクター設定をプロンプトに含める
- 会話履歴でコンテキストを維持
- バッチ処理で定期的に会話を進行
- 並行処理でスケーラビリティを確保

## 参考資料

- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [go-openai](https://github.com/sashabaranov/go-openai)
