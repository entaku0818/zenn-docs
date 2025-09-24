---
title: "【Go】8桁のショートIDを自動生成してURLを短縮する"
emoji: "🔗"
type: "tech"
topics: ["Go", "UUID", "URL短縮", "crypto"]
published: false
---

# この記事は？

UUIDは一意性は保証されますが、URLに含めると長くなりすぎて共有しにくいという問題があります。この記事では、8桁のランダムなショートIDを生成し、共有しやすいURLを実現する方法を解説します。

## 問題の背景

トークルームのURLを共有する際、UUIDを使用すると非常に長くなります：

```
# UUIDを使用した場合
https://example.com/talkrooms/550e8400-e29b-41d4-a716-446655440000

# ショートIDを使用した場合
https://example.com/t/a1b2c3d4
```

## 実装

### 1. ショートID生成関数

```go
// domain/talkroom/shortid/generator.go
package shortid

import (
    "crypto/rand"
    "fmt"
    "math/big"
)

const (
    // 使用する文字セット（小文字英数字）
    charset = "abcdefghijklmnopqrstuvwxyz0123456789"
    // ショートIDの長さ
    length = 8
)

// Generate は8桁のランダムなショートIDを生成する
func Generate() (string, error) {
    result := make([]byte, length)
    charsetLen := big.NewInt(int64(len(charset)))

    for i := 0; i < length; i++ {
        // crypto/randで暗号学的に安全な乱数を生成
        n, err := rand.Int(rand.Reader, charsetLen)
        if err != nil {
            return "", fmt.Errorf("failed to generate random number: %w", err)
        }
        result[i] = charset[n.Int64()]
    }

    return string(result), nil
}
```

### 2. ショートIDサービス（衝突チェック付き）

```go
// domain/talkroom/shortid/service.go
package shortid

import (
    "context"
    "fmt"

    "github.com/volatiletech/sqlboiler/v4/boil"
)

const maxRetries = 3

type Repository interface {
    Exists(ctx context.Context, exec boil.ContextExecutor, shortID string) (bool, error)
    Create(ctx context.Context, exec boil.ContextExecutor, talkRoomID, shortID string) error
}

type Service struct {
    repo Repository
}

func NewService(repo Repository) *Service {
    return &Service{repo: repo}
}

// GenerateUnique は一意のショートIDを生成する
func (s *Service) GenerateUnique(
    ctx context.Context,
    exec boil.ContextExecutor,
    talkRoomID string,
) (string, error) {
    for i := 0; i < maxRetries; i++ {
        shortID, err := Generate()
        if err != nil {
            return "", err
        }

        // 衝突チェック
        exists, err := s.repo.Exists(ctx, exec, shortID)
        if err != nil {
            return "", err
        }

        if !exists {
            // DBに保存
            if err := s.repo.Create(ctx, exec, talkRoomID, shortID); err != nil {
                return "", err
            }
            return shortID, nil
        }
        // 衝突した場合はリトライ
    }

    return "", fmt.Errorf("failed to generate unique short ID after %d retries", maxRetries)
}
```

### 3. データベーススキーマ

```sql
CREATE TABLE talk_room_short_ids (
    id VARCHAR(100) PRIMARY KEY,
    talk_room_id VARCHAR(100) NOT NULL,
    short_id VARCHAR(8) NOT NULL UNIQUE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    FOREIGN KEY (talk_room_id) REFERENCES talk_rooms(id) ON DELETE CASCADE,
    INDEX idx_short_id (short_id)
);
```

### 4. リポジトリ実装

```go
// repository/talk_room_short_id_repository.go
package repository

import (
    "context"

    "github.com/google/uuid"
    "github.com/volatiletech/sqlboiler/v4/boil"
    "github.com/example/models" // SQLBoiler生成モデル
)

type TalkRoomShortIDRepository struct{}

func NewRepository() *TalkRoomShortIDRepository {
    return &TalkRoomShortIDRepository{}
}

func (r *TalkRoomShortIDRepository) Exists(
    ctx context.Context,
    exec boil.ContextExecutor,
    shortID string,
) (bool, error) {
    return models.TalkRoomShortIds(
        models.TalkRoomShortIdWhere.ShortID.EQ(shortID),
    ).Exists(ctx, exec)
}

func (r *TalkRoomShortIDRepository) Create(
    ctx context.Context,
    exec boil.ContextExecutor,
    talkRoomID, shortID string,
) error {
    record := &models.TalkRoomShortId{
        ID:         uuid.New().String(),
        TalkRoomID: talkRoomID,
        ShortID:    shortID,
    }
    return record.Insert(ctx, exec, boil.Infer())
}

func (r *TalkRoomShortIDRepository) FindByShortID(
    ctx context.Context,
    exec boil.ContextExecutor,
    shortID string,
) (*models.TalkRoomShortId, error) {
    return models.TalkRoomShortIds(
        models.TalkRoomShortIdWhere.ShortID.EQ(shortID),
    ).One(ctx, exec)
}
```

### 5. トークルーム作成時に使用

```go
// usecase/talkroom_usecase.go

func (uc *UseCase) CreateTalkRoom(
    ctx context.Context,
    title string,
    createdBy string,
) (*TalkRoom, error) {
    tx, err := boil.BeginTx(ctx, nil)
    if err != nil {
        return nil, err
    }
    defer tx.Rollback()

    // トークルーム作成
    talkRoom := &models.TalkRoom{
        ID:        uuid.New().String(),
        Title:     title,
        CreatedBy: createdBy,
    }
    if err := talkRoom.Insert(ctx, tx, boil.Infer()); err != nil {
        return nil, err
    }

    // ショートID生成
    shortID, err := uc.shortIDService.GenerateUnique(ctx, tx, talkRoom.ID)
    if err != nil {
        return nil, err
    }

    if err := tx.Commit(); err != nil {
        return nil, err
    }

    return &TalkRoom{
        ID:      talkRoom.ID,
        ShortID: shortID,
        Title:   title,
    }, nil
}
```

## テスト

```go
// domain/talkroom/shortid/generator_test.go
func TestGenerate(t *testing.T) {
    shortID, err := Generate()
    require.NoError(t, err)

    // 長さの確認
    assert.Len(t, shortID, 8)

    // 文字セットの確認
    for _, c := range shortID {
        assert.Contains(t, charset, string(c))
    }
}

func TestGenerate_Uniqueness(t *testing.T) {
    generated := make(map[string]bool)

    for i := 0; i < 10000; i++ {
        shortID, err := Generate()
        require.NoError(t, err)

        if generated[shortID] {
            t.Fatalf("duplicate short ID generated: %s", shortID)
        }
        generated[shortID] = true
    }
}
```

## 衝突確率

8桁の英数字（36文字）の組み合わせ数は約2.8兆通り：

```
36^8 = 2,821,109,907,456
```

100万件のレコードがある場合の衝突確率は約0.00004%で、実用上問題ありません。

## まとめ

- `crypto/rand`で暗号学的に安全な乱数を生成
- 衝突時は最大3回リトライ
- CASCADE DELETEでトークルーム削除時に自動削除
- 8桁で約2.8兆通りの組み合わせ

## 参考資料

- [crypto/rand](https://pkg.go.dev/crypto/rand)
- [SQLBoiler](https://github.com/volatiletech/sqlboiler)
