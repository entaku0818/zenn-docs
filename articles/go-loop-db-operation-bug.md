---
title: "【Go】ループ内DB操作のバグを防ぐ"
emoji: "🐛"
type: "tech"
topics: ["Go", "バグ", "データベース", "並行処理"]
published: false
---

# この記事は？

Goでよくあるループ内でのDB操作に関するバグと、その修正方法を解説します。特に、rangeループでの変数キャプチャ問題は見落としやすいので注意が必要です。

## 問題の状況

ユーザーがトークルームに参加する際、過去に参加していた全てのルームから退出する処理で、最初の1つしか退出できていないバグが発生しました。

### バグのあるコード

```go
// ❌ バグのあるコード
func (uc *TalkRoomUseCase) LeaveAllRooms(
    ctx context.Context,
    userID string,
) error {
    // ユーザーが参加している全ルームを取得
    rooms, err := uc.repo.GetJoinedRooms(ctx, userID)
    if err != nil {
        return err
    }

    for _, room := range rooms {
        // ❌ 問題: goroutine内でループ変数を参照
        go func() {
            uc.repo.LeaveMember(ctx, room.ID, userID)
        }()
    }

    return nil
}
```

このコードでは、goroutineがループ変数`room`を参照しているため、全てのgoroutineが最後の`room`を参照してしまいます。

## 修正方法

### 修正1: ループ変数を引数で渡す

```go
// ✅ 修正版: 引数で渡す
for _, room := range rooms {
    go func(r model.TalkRoom) {
        uc.repo.LeaveMember(ctx, r.ID, userID)
    }(room) // 引数として渡す
}
```

### 修正2: ループ内で変数をコピー

```go
// ✅ 修正版: ローカル変数にコピー
for _, room := range rooms {
    room := room // シャドーイング
    go func() {
        uc.repo.LeaveMember(ctx, room.ID, userID)
    }()
}
```

### 修正3: Go 1.22以降ではループ変数が自動コピー

Go 1.22以降では、ループ変数のセマンティクスが変更され、この問題は発生しなくなりました。

```go
// Go 1.22以降では問題なし
for _, room := range rooms {
    go func() {
        uc.repo.LeaveMember(ctx, room.ID, userID)
    }()
}
```

## 同期的な処理の場合も注意

goroutineを使わない場合でも、クロージャを使う場合は同様の問題が発生します。

```go
// ❌ バグのあるコード
var operations []func()

for _, room := range rooms {
    operations = append(operations, func() {
        uc.repo.LeaveMember(ctx, room.ID, userID)
    })
}

// 全ての operations が最後の room を参照
for _, op := range operations {
    op()
}
```

修正：

```go
// ✅ 修正版
var operations []func()

for _, room := range rooms {
    room := room
    operations = append(operations, func() {
        uc.repo.LeaveMember(ctx, room.ID, userID)
    })
}
```

## より安全な実装パターン

### パターン1: errgroup + インデックスアクセス

```go
import "golang.org/x/sync/errgroup"

func (uc *TalkRoomUseCase) LeaveAllRooms(
    ctx context.Context,
    userID string,
) error {
    rooms, err := uc.repo.GetJoinedRooms(ctx, userID)
    if err != nil {
        return err
    }

    g, ctx := errgroup.WithContext(ctx)

    for i := range rooms {
        i := i // Go 1.21以前用
        g.Go(func() error {
            return uc.repo.LeaveMember(ctx, rooms[i].ID, userID)
        })
    }

    return g.Wait()
}
```

### パターン2: 同期的に順次処理

並行処理が不要な場合は、シンプルに同期処理：

```go
func (uc *TalkRoomUseCase) LeaveAllRooms(
    ctx context.Context,
    userID string,
) error {
    rooms, err := uc.repo.GetJoinedRooms(ctx, userID)
    if err != nil {
        return err
    }

    for _, room := range rooms {
        if err := uc.repo.LeaveMember(ctx, room.ID, userID); err != nil {
            return fmt.Errorf("failed to leave room %s: %w", room.ID, err)
        }
    }

    return nil
}
```

### パターン3: バッチ処理

一度のクエリで全て処理：

```go
func (r *TalkRoomMemberRepository) LeaveAllRooms(
    ctx context.Context,
    userID string,
) error {
    _, err := r.db.NewDelete().
        Model((*model.TalkRoomMember)(nil)).
        Where("user_id = ?", userID).
        Exec(ctx)

    return err
}
```

## テストで検出する

このようなバグはテストで検出できます：

```go
func TestLeaveAllRooms(t *testing.T) {
    // 3つのルームに参加
    rooms := []string{"room1", "room2", "room3"}
    for _, roomID := range rooms {
        repo.AddMember(ctx, roomID, "user1")
    }

    // 全ルームから退出
    err := useCase.LeaveAllRooms(ctx, "user1")
    require.NoError(t, err)

    // 全てのルームから退出していることを確認
    for _, roomID := range rooms {
        exists, _ := repo.ExistsMember(ctx, roomID, "user1")
        assert.False(t, exists, "Should have left room %s", roomID)
    }
}
```

## go vetでの検出

`go vet`はこの問題を検出できます：

```bash
go vet ./...
# loop variable room captured by func literal
```

## まとめ

- ループ変数をgoroutineやクロージャで参照する際は注意
- Go 1.21以前: 引数で渡すかローカル変数にコピー
- Go 1.22以降: 自動的にループごとに新しい変数が作成
- `go vet`で検出可能
- バッチ処理で解決できる場合はそちらを検討

## 参考資料

- [Go Wiki - CommonMistakes](https://go.dev/wiki/CommonMistakes)
- [Go 1.22 Release Notes - LoopVar](https://go.dev/doc/go1.22#language)
