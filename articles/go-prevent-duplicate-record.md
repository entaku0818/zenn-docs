---
title: "【Go】重複レコード作成を防ぐ実装パターン"
emoji: "🔐"
type: "tech"
topics: ["Go", "MySQL", "データベース", "並行処理"]
published: false
---

# この記事は？

並行リクエストが発生する環境で、同じレコードが重複して作成されてしまう問題に対処する方法を解説します。

## 問題の状況

トークルームにAIユーザーを自動参加させる機能で、同じAIユーザーが重複して参加してしまう問題が発生しました。

```
リクエストA: AIユーザーを追加 → 成功
リクエストB: AIユーザーを追加 → 成功（重複！）
```

## 解決方法

### 方法1: ユニーク制約 + INSERT IGNORE

データベースレベルで重複を防ぎます。

```sql
-- マイグレーション
ALTER TABLE talk_room_members
ADD UNIQUE KEY uk_room_user (talk_room_id, user_id);
```

```go
// repository/talk_room_member_repository.go

func (r *TalkRoomMemberRepository) AddMemberIfNotExists(
    ctx context.Context,
    talkRoomID string,
    userID string,
) error {
    member := &model.TalkRoomMember{
        ID:         uuid.New().String(),
        TalkRoomID: talkRoomID,
        UserID:     userID,
        JoinedAt:   time.Now(),
    }

    _, err := r.db.NewInsert().
        Model(member).
        Ignore(). // INSERT IGNORE
        Exec(ctx)

    return err
}
```

### 方法2: ON DUPLICATE KEY UPDATE（Upsert）

重複時に更新を行うパターン：

```go
func (r *TalkRoomMemberRepository) UpsertMember(
    ctx context.Context,
    talkRoomID string,
    userID string,
) error {
    member := &model.TalkRoomMember{
        ID:         uuid.New().String(),
        TalkRoomID: talkRoomID,
        UserID:     userID,
        JoinedAt:   time.Now(),
    }

    _, err := r.db.NewInsert().
        Model(member).
        On("DUPLICATE KEY UPDATE").
        Set("updated_at = ?", time.Now()).
        Exec(ctx)

    return err
}
```

### 方法3: 存在チェック + 挿入（トランザクション付き）

```go
func (r *TalkRoomMemberRepository) AddMemberSafely(
    ctx context.Context,
    talkRoomID string,
    userID string,
) error {
    return r.db.RunInTx(ctx, nil, func(ctx context.Context, tx bun.Tx) error {
        // 存在チェック（FOR UPDATE でロック）
        exists, err := tx.NewSelect().
            Model((*model.TalkRoomMember)(nil)).
            Where("talk_room_id = ? AND user_id = ?", talkRoomID, userID).
            For("UPDATE").
            Exists(ctx)

        if err != nil {
            return err
        }

        if exists {
            // 既に存在する場合は何もしない
            return nil
        }

        // 挿入
        member := &model.TalkRoomMember{
            ID:         uuid.New().String(),
            TalkRoomID: talkRoomID,
            UserID:     userID,
            JoinedAt:   time.Now(),
        }

        _, err = tx.NewInsert().Model(member).Exec(ctx)
        return err
    })
}
```

### 方法4: アプリケーションレベルのロック

Redis等を使ったロック：

```go
func (uc *TalkRoomUseCase) AddAIUserWithLock(
    ctx context.Context,
    talkRoomID string,
    aiUserID string,
) error {
    lockKey := fmt.Sprintf("lock:add_ai:%s:%s", talkRoomID, aiUserID)

    // ロック取得（10秒間）
    acquired, err := uc.redis.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
    if err != nil {
        return err
    }

    if !acquired {
        // 他のプロセスが処理中
        return nil
    }

    defer uc.redis.Del(ctx, lockKey)

    // 重複チェック
    exists, err := uc.memberRepo.Exists(ctx, talkRoomID, aiUserID)
    if err != nil {
        return err
    }
    if exists {
        return nil
    }

    // 追加
    return uc.memberRepo.Add(ctx, talkRoomID, aiUserID)
}
```

## 実践的な実装例

AIユーザーの重複作成を防ぐ完全な例：

```go
// usecase/ai_avatar_usecase.go

type AIAvatarUseCase struct {
    talkRoomMemberRepo TalkRoomMemberRepository
    aiUserRepo         AIUserRepository
}

func (uc *AIAvatarUseCase) JoinAIToRoom(
    ctx context.Context,
    talkRoomID string,
    aiCharacterID string,
) error {
    // AIユーザーを取得（存在しなければ作成）
    aiUser, err := uc.getOrCreateAIUser(ctx, aiCharacterID)
    if err != nil {
        return fmt.Errorf("failed to get/create AI user: %w", err)
    }

    // 既に参加しているかチェック
    exists, err := uc.talkRoomMemberRepo.ExistsMember(ctx, talkRoomID, aiUser.ID)
    if err != nil {
        return fmt.Errorf("failed to check membership: %w", err)
    }

    if exists {
        log.Printf("AI user %s already in room %s, skipping", aiUser.ID, talkRoomID)
        return nil
    }

    // INSERT IGNOREで安全に追加
    err = uc.talkRoomMemberRepo.AddMemberIfNotExists(ctx, talkRoomID, aiUser.ID)
    if err != nil {
        return fmt.Errorf("failed to add AI member: %w", err)
    }

    log.Printf("AI user %s joined room %s", aiUser.ID, talkRoomID)
    return nil
}

func (uc *AIAvatarUseCase) getOrCreateAIUser(
    ctx context.Context,
    aiCharacterID string,
) (*model.User, error) {
    // 既存のAIユーザーを検索
    user, err := uc.aiUserRepo.FindByCharacterID(ctx, aiCharacterID)
    if err == nil {
        return user, nil
    }

    if !errors.Is(err, sql.ErrNoRows) {
        return nil, err
    }

    // 存在しなければ作成（INSERT IGNORE）
    newUser := &model.User{
        ID:          fmt.Sprintf("ai_%s", aiCharacterID),
        Name:        getAICharacterName(aiCharacterID),
        IsAI:        true,
        CharacterID: aiCharacterID,
    }

    err = uc.aiUserRepo.CreateIfNotExists(ctx, newUser)
    if err != nil {
        return nil, err
    }

    return newUser, nil
}
```

## 方法の比較

| 方法 | メリット | デメリット |
|------|---------|-----------|
| INSERT IGNORE | シンプル、高速 | 重複時のエラーが隠れる |
| ON DUPLICATE KEY UPDATE | 更新も同時に行える | 更新が不要な場合はオーバーヘッド |
| トランザクション + FOR UPDATE | 確実、柔軟 | パフォーマンスに影響 |
| Redisロック | 高速、分散対応 | Redis依存、複雑性増 |

## 推奨パターン

1. まずユニーク制約を設定
2. INSERT IGNOREまたはON DUPLICATE KEY UPDATEを使用
3. アプリケーション側でも事前チェック（ログ出力用）

```go
// 推奨実装
func (r *Repository) AddMemberSafely(ctx context.Context, roomID, userID string) error {
    // 1. 事前チェック（任意、ログ目的）
    exists, _ := r.Exists(ctx, roomID, userID)
    if exists {
        log.Printf("Member already exists: room=%s, user=%s", roomID, userID)
        return nil
    }

    // 2. INSERT IGNORE で安全に挿入
    return r.InsertIgnore(ctx, roomID, userID)
}
```

## まとめ

- データベースにユニーク制約を設定するのが基本
- INSERT IGNOREまたはON DUPLICATE KEY UPDATEで重複を防止
- アプリケーション側での事前チェックはログ・デバッグ目的
- 高負荷環境ではRedisロックも検討

## 参考資料

- [MySQL - INSERT IGNORE](https://dev.mysql.com/doc/refman/8.0/en/insert.html)
- [Bun - Insert](https://bun.uptrace.dev/guide/query-insert.html)
