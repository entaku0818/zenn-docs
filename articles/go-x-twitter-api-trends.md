---
title: "【Go】X (Twitter) APIからトレンドを取得する"
emoji: "𝕏"
type: "tech"
topics: ["Go", "Twitter", "X", "API"]
published: false
---

# この記事は？

GoでX (旧Twitter) API v2を使用してトレンドデータを取得するクライアントの実装方法を解説します。

## 前提条件

- X Developer Portalでアプリを作成
- Bearer Tokenを取得

## 実装

### 1. X APIクライアントの作成

```go
// pkg/xapi/client.go
package xapi

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

const baseURL = "https://api.twitter.com/2"

type Client struct {
    bearerToken string
    httpClient  *http.Client
}

func NewClient(bearerToken string) *Client {
    return &Client{
        bearerToken: bearerToken,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

func (c *Client) doRequest(ctx context.Context, method, path string) ([]byte, error) {
    url := baseURL + path

    req, err := http.NewRequestWithContext(ctx, method, url, nil)
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    req.Header.Set("Authorization", "Bearer "+c.bearerToken)
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to do request: %w", err)
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("API error: status=%d, body=%s", resp.StatusCode, string(body))
    }

    return body, nil
}
```

### 2. トレンド取得機能

```go
// pkg/xapi/trends.go
package xapi

import (
    "context"
    "encoding/json"
    "fmt"
)

type TrendsResponse struct {
    Data []Trend `json:"data"`
}

type Trend struct {
    TrendName    string `json:"trend_name"`
    TweetCount   int    `json:"tweet_count,omitempty"`
    DomainID     string `json:"domain_id,omitempty"`
    EntityID     string `json:"entity_id,omitempty"`
}

// GetTrendsForWoeid は指定されたWOEIDのトレンドを取得
// 日本: 23424856
func (c *Client) GetTrendsForWoeid(ctx context.Context, woeid int) ([]Trend, error) {
    // Note: X API v2のトレンドエンドポイントは制限あり
    // 代替としてv1.1を使用
    path := fmt.Sprintf("/1.1/trends/place.json?id=%d", woeid)

    body, err := c.doRequest(ctx, "GET", path)
    if err != nil {
        return nil, err
    }

    var response []struct {
        Trends []struct {
            Name       string `json:"name"`
            TweetVolume *int  `json:"tweet_volume"`
        } `json:"trends"`
    }

    if err := json.Unmarshal(body, &response); err != nil {
        return nil, fmt.Errorf("failed to unmarshal: %w", err)
    }

    if len(response) == 0 {
        return nil, nil
    }

    trends := make([]Trend, 0, len(response[0].Trends))
    for _, t := range response[0].Trends {
        trend := Trend{
            TrendName: t.Name,
        }
        if t.TweetVolume != nil {
            trend.TweetCount = *t.TweetVolume
        }
        trends = append(trends, trend)
    }

    return trends, nil
}

// GetJapanTrends は日本のトレンドを取得
func (c *Client) GetJapanTrends(ctx context.Context) ([]Trend, error) {
    const japanWoeid = 23424856
    return c.GetTrendsForWoeid(ctx, japanWoeid)
}
```

### 3. ツイート検索機能

```go
// pkg/xapi/search.go
package xapi

import (
    "context"
    "encoding/json"
    "fmt"
    "net/url"
)

type SearchResponse struct {
    Data []Tweet `json:"data"`
    Meta Meta    `json:"meta"`
}

type Tweet struct {
    ID        string `json:"id"`
    Text      string `json:"text"`
    AuthorID  string `json:"author_id"`
    CreatedAt string `json:"created_at"`
}

type Meta struct {
    ResultCount int    `json:"result_count"`
    NextToken   string `json:"next_token,omitempty"`
}

func (c *Client) SearchRecentTweets(
    ctx context.Context,
    query string,
    maxResults int,
) (*SearchResponse, error) {
    params := url.Values{}
    params.Set("query", query)
    params.Set("max_results", fmt.Sprintf("%d", maxResults))
    params.Set("tweet.fields", "created_at,author_id")

    path := "/tweets/search/recent?" + params.Encode()

    body, err := c.doRequest(ctx, "GET", path)
    if err != nil {
        return nil, err
    }

    var response SearchResponse
    if err := json.Unmarshal(body, &response); err != nil {
        return nil, fmt.Errorf("failed to unmarshal: %w", err)
    }

    return &response, nil
}
```

### 4. トレンドを使ったサービス

```go
// service/trend_service.go
package service

import (
    "context"
    "log"

    "myapp/pkg/xapi"
)

type TrendService struct {
    xClient *xapi.Client
}

func NewTrendService(bearerToken string) *TrendService {
    return &TrendService{
        xClient: xapi.NewClient(bearerToken),
    }
}

// GetTopTrends は上位N件のトレンドを取得
func (s *TrendService) GetTopTrends(ctx context.Context, limit int) ([]string, error) {
    trends, err := s.xClient.GetJapanTrends(ctx)
    if err != nil {
        return nil, err
    }

    if len(trends) > limit {
        trends = trends[:limit]
    }

    result := make([]string, len(trends))
    for i, t := range trends {
        result[i] = t.TrendName
    }

    return result, nil
}

// GetTrendingTopics はトレンドからトピックを抽出
func (s *TrendService) GetTrendingTopics(ctx context.Context) ([]string, error) {
    trends, err := s.xClient.GetJapanTrends(ctx)
    if err != nil {
        return nil, err
    }

    topics := make([]string, 0)
    for _, trend := range trends {
        // ハッシュタグを除外して純粋なトピックのみ
        if len(trend.TrendName) > 0 && trend.TrendName[0] != '#' {
            topics = append(topics, trend.TrendName)
        }
    }

    return topics, nil
}
```

## 使用例

### トレンドからトークルームを自動生成

```go
// usecase/room_generator.go

type RoomGeneratorUseCase struct {
    trendService *service.TrendService
    roomRepo     TalkRoomRepository
    aiService    AIService
}

func (uc *RoomGeneratorUseCase) GenerateRoomsFromTrends(ctx context.Context) error {
    // トレンドを取得
    topics, err := uc.trendService.GetTrendingTopics(ctx)
    if err != nil {
        return err
    }

    // 上位5件のトピックでルームを生成
    for _, topic := range topics[:5] {
        // AIでルームタイトルを生成
        title, err := uc.aiService.GenerateRoomTitle(ctx, topic)
        if err != nil {
            log.Printf("Failed to generate title for %s: %v", topic, err)
            continue
        }

        // ルームを作成
        room := &model.TalkRoom{
            ID:        uuid.New().String(),
            Title:     title,
            Category:  "trending",
            CreatedAt: time.Now(),
        }

        if err := uc.roomRepo.Create(ctx, room); err != nil {
            log.Printf("Failed to create room: %v", err)
            continue
        }

        log.Printf("Created room: %s", title)
    }

    return nil
}
```

## レート制限への対応

```go
type RateLimitedClient struct {
    client      *xapi.Client
    limiter     *rate.Limiter
}

func NewRateLimitedClient(bearerToken string) *RateLimitedClient {
    // 15分あたり15リクエスト → 1分あたり1リクエスト
    limiter := rate.NewLimiter(rate.Every(time.Minute), 1)

    return &RateLimitedClient{
        client:  xapi.NewClient(bearerToken),
        limiter: limiter,
    }
}

func (c *RateLimitedClient) GetJapanTrends(ctx context.Context) ([]xapi.Trend, error) {
    if err := c.limiter.Wait(ctx); err != nil {
        return nil, err
    }

    return c.client.GetJapanTrends(ctx)
}
```

## エラーハンドリング

```go
type APIError struct {
    StatusCode int
    Message    string
}

func (e *APIError) Error() string {
    return fmt.Sprintf("X API error: status=%d, message=%s", e.StatusCode, e.Message)
}

func (e *APIError) IsRateLimit() bool {
    return e.StatusCode == 429
}

func (e *APIError) IsUnauthorized() bool {
    return e.StatusCode == 401
}
```

## まとめ

- X API v2でトレンドを取得
- Bearer Tokenを使った認証
- レート制限に注意（15分あたり15リクエスト）
- トレンドからコンテンツを自動生成する応用例

## 参考資料

- [X API Documentation](https://developer.twitter.com/en/docs/twitter-api)
- [X API - Trends lookup](https://developer.twitter.com/en/docs/twitter-api/trends/api-reference)
