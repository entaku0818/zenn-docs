---
title: "【Go】Slack通知を実装する（Incoming Webhooks）"
emoji: "🔔"
type: "tech"
topics: ["Go", "Slack", "Webhook", "通知"]
published: false
---

# この記事は？

Goアプリケーションから特定のイベント発生時にSlackへ通知を送る実装を解説します。Incoming Webhooksを使ったシンプルな方法です。

## 前提条件

- Slack Appの作成権限
- Incoming WebhooksのWebhook URL

## Slack Appの設定

1. [Slack API](https://api.slack.com/apps)でアプリを作成
2. 「Incoming Webhooks」を有効化
3. 「Add New Webhook to Workspace」でチャンネルを選択
4. Webhook URLを取得

## 実装

### 1. Slackクライアントの作成

```go
// pkg/slack/client.go
package slack

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type Client struct {
    webhookURL string
    httpClient *http.Client
}

func NewClient(webhookURL string) *Client {
    return &Client{
        webhookURL: webhookURL,
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}

type Message struct {
    Text        string       `json:"text,omitempty"`
    Blocks      []Block      `json:"blocks,omitempty"`
    Attachments []Attachment `json:"attachments,omitempty"`
}

type Block struct {
    Type string `json:"type"`
    Text *Text  `json:"text,omitempty"`
}

type Text struct {
    Type string `json:"type"`
    Text string `json:"text"`
}

type Attachment struct {
    Color  string `json:"color,omitempty"`
    Title  string `json:"title,omitempty"`
    Text   string `json:"text,omitempty"`
    Fields []Field `json:"fields,omitempty"`
}

type Field struct {
    Title string `json:"title"`
    Value string `json:"value"`
    Short bool   `json:"short"`
}

func (c *Client) Send(ctx context.Context, msg *Message) error {
    body, err := json.Marshal(msg)
    if err != nil {
        return fmt.Errorf("failed to marshal message: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPost, c.webhookURL, bytes.NewReader(body))
    if err != nil {
        return fmt.Errorf("failed to create request: %w", err)
    }

    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return fmt.Errorf("failed to send request: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("slack returned non-200 status: %d", resp.StatusCode)
    }

    return nil
}
```

### 2. 通知サービスの作成

```go
// service/notification_service.go
package service

import (
    "context"
    "fmt"

    "myapp/pkg/slack"
)

type NotificationService struct {
    slackClient *slack.Client
}

func NewNotificationService(webhookURL string) *NotificationService {
    return &NotificationService{
        slackClient: slack.NewClient(webhookURL),
    }
}

// NotifyFirstParticipant はトークルームに最初の参加者が入った時に通知
func (s *NotificationService) NotifyFirstParticipant(
    ctx context.Context,
    roomTitle string,
    userName string,
) error {
    msg := &slack.Message{
        Blocks: []slack.Block{
            {
                Type: "header",
                Text: &slack.Text{
                    Type: "plain_text",
                    Text: "🎉 新規トークルーム参加",
                },
            },
            {
                Type: "section",
                Text: &slack.Text{
                    Type: "mrkdwn",
                    Text: fmt.Sprintf("*%s* に最初のユーザーが参加しました！", roomTitle),
                },
            },
        },
        Attachments: []slack.Attachment{
            {
                Color: "#36a64f",
                Fields: []slack.Field{
                    {
                        Title: "ルーム名",
                        Value: roomTitle,
                        Short: true,
                    },
                    {
                        Title: "参加者",
                        Value: userName,
                        Short: true,
                    },
                },
            },
        },
    }

    return s.slackClient.Send(ctx, msg)
}

// NotifyError はエラー発生時に通知
func (s *NotificationService) NotifyError(
    ctx context.Context,
    errorType string,
    errorMessage string,
) error {
    msg := &slack.Message{
        Blocks: []slack.Block{
            {
                Type: "header",
                Text: &slack.Text{
                    Type: "plain_text",
                    Text: "⚠️ エラー発生",
                },
            },
        },
        Attachments: []slack.Attachment{
            {
                Color: "#ff0000",
                Fields: []slack.Field{
                    {
                        Title: "エラータイプ",
                        Value: errorType,
                        Short: true,
                    },
                    {
                        Title: "詳細",
                        Value: errorMessage,
                        Short: false,
                    },
                },
            },
        },
    }

    return s.slackClient.Send(ctx, msg)
}
```

### 3. ユースケースでの使用

```go
// usecase/talkroom_usecase.go

type TalkRoomUseCase struct {
    talkRoomRepo        TalkRoomRepository
    notificationService *service.NotificationService
}

func (uc *TalkRoomUseCase) JoinTalkRoom(
    ctx context.Context,
    roomID string,
    userID string,
) error {
    // 参加前のメンバー数を確認
    memberCount, err := uc.talkRoomRepo.GetMemberCount(ctx, roomID)
    if err != nil {
        return err
    }

    // 参加処理
    if err := uc.talkRoomRepo.AddMember(ctx, roomID, userID); err != nil {
        return err
    }

    // 最初の参加者の場合、Slack通知
    if memberCount == 0 {
        room, _ := uc.talkRoomRepo.GetByID(ctx, roomID)
        user, _ := uc.userRepo.GetByID(ctx, userID)

        // 非同期で通知（エラーは無視）
        go func() {
            _ = uc.notificationService.NotifyFirstParticipant(
                context.Background(),
                room.Title,
                user.Name,
            )
        }()
    }

    return nil
}
```

## メンション付き通知

特定のユーザーにメンションを送る場合：

```go
func (s *NotificationService) NotifyWithMention(
    ctx context.Context,
    message string,
    mentionUserIDs []string, // Slack User IDs
) error {
    // メンションを構築
    mentions := ""
    for _, id := range mentionUserIDs {
        mentions += fmt.Sprintf("<@%s> ", id)
    }

    msg := &slack.Message{
        Text: mentions + message,
    }

    return s.slackClient.Send(ctx, msg)
}

// チャンネル全体へのメンション
func (s *NotificationService) NotifyChannel(ctx context.Context, message string) error {
    msg := &slack.Message{
        Text: "<!channel> " + message,
    }
    return s.slackClient.Send(ctx, msg)
}
```

## 環境別の設定

```go
// config/config.go

type Config struct {
    SlackWebhookURL string `env:"SLACK_WEBHOOK_URL"`
    Environment     string `env:"ENVIRONMENT" envDefault:"development"`
}

// 本番環境のみ通知を送る
func (c *Config) ShouldNotify() bool {
    return c.Environment == "production"
}
```

使用例：

```go
if config.ShouldNotify() {
    _ = notificationService.NotifyFirstParticipant(ctx, room.Title, user.Name)
}
```

## リトライ機能の追加

```go
func (c *Client) SendWithRetry(ctx context.Context, msg *Message, maxRetries int) error {
    var lastErr error

    for i := 0; i < maxRetries; i++ {
        err := c.Send(ctx, msg)
        if err == nil {
            return nil
        }

        lastErr = err
        time.Sleep(time.Duration(i+1) * time.Second) // 指数バックオフ
    }

    return fmt.Errorf("failed after %d retries: %w", maxRetries, lastErr)
}
```

## まとめ

- Incoming Webhooksで簡単にSlack通知を実装
- Block Kitでリッチなメッセージを構築
- 非同期で送信してメイン処理をブロックしない
- 環境別に通知の有効/無効を制御

## 参考資料

- [Slack - Incoming Webhooks](https://api.slack.com/messaging/webhooks)
- [Slack - Block Kit Builder](https://app.slack.com/block-kit-builder)
