---
title: "【Go】N+1問題を解消してホーム画面APIを高速化する"
emoji: "🚀"
type: "tech"
topics: ["Go", "GORM", "MySQL", "パフォーマンス", "N+1問題"]
published: false
---

# この記事は？

Go + GORMでAPIを実装していて、レスポンスが遅いと感じたことはありませんか？その原因の一つがN+1問題です。この記事では、ホーム画面APIで発生していたN+1問題を解消し、レスポンスタイムを大幅に改善した実例を紹介します。

## 問題の状況

ホーム画面でトークルーム一覧を取得するAPIで、トークルーム数が増えるとレスポンスが極端に遅くなる問題が発生していました。

### 修正前のパフォーマンス

- トークルーム10件で20-30クエリ
- レスポンスタイム: 500ms〜2秒

### 修正後のパフォーマンス

- トークルーム10件で3-5クエリ
- レスポンスタイム: 50-200ms

## N+1問題とは？

N+1問題は、1回のメインクエリに対して、結果の各行ごとに追加のクエリが実行されてしまうパターンです。

```go
// ❌ N+1問題のあるコード
func GetTalkRooms(ctx context.Context) ([]TalkRoom, error) {
    var talkRooms []TalkRoom
    db.Find(&talkRooms)  // 1回のクエリ

    for i := range talkRooms {
        var messages []Message
        // トークルームごとにクエリ（N回）
        db.Where("talk_room_id = ?", talkRooms[i].ID).Find(&messages)
        talkRooms[i].MessageCount = len(messages)
    }

    return talkRooms, nil
}
```

## 解決方法：バッチ取得メソッドの実装

### 1. リポジトリにバッチ取得メソッドを追加

```go
// repository/talk_message_repository.go

// CountByTalkRoomIDs は複数のトークルームのメッセージ数を一括取得する
func (r *TalkMessageRepository) CountByTalkRoomIDs(
    ctx context.Context,
    exec boil.ContextExecutor,
    talkRoomIDs []string,
) (map[string]int64, error) {
    if len(talkRoomIDs) == 0 {
        return make(map[string]int64), nil
    }

    type result struct {
        TalkRoomID string `boil:"talk_room_id"`
        Count      int64  `boil:"count"`
    }

    var results []result

    query := `
        SELECT talk_room_id, COUNT(*) as count
        FROM talk_messages
        WHERE talk_room_id IN (?)
        GROUP BY talk_room_id
    `

    // IN句の展開
    query, args, err := sqlx.In(query, talkRoomIDs)
    if err != nil {
        return nil, err
    }

    rows, err := exec.QueryContext(ctx, query, args...)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    counts := make(map[string]int64)
    for rows.Next() {
        var r result
        if err := rows.Scan(&r.TalkRoomID, &r.Count); err != nil {
            return nil, err
        }
        counts[r.TalkRoomID] = r.Count
    }

    return counts, nil
}
```

### 2. ユースケースでバッチ取得を使用

```go
// usecase/talkroom_usecase.go

func (uc *UseCase) GetTalkRoomRecommendations(
    ctx context.Context,
) ([]RecommendedTalkRoom, error) {
    // トークルーム一覧を取得
    talkRooms, err := uc.talkRoomRepo.GetRecommended(ctx, boil.GetContextDB())
    if err != nil {
        return nil, err
    }

    // 全トークルームIDを収集
    talkRoomIDs := make([]string, len(talkRooms))
    for i, tr := range talkRooms {
        talkRoomIDs[i] = tr.ID
    }

    // メッセージ数を一括取得（N+1問題を解消）
    messageCounts, err := uc.talkMessageRepo.CountByTalkRoomIDs(
        ctx, boil.GetContextDB(), talkRoomIDs,
    )
    if err != nil {
        return nil, err
    }

    // 結果を組み立て
    result := make([]RecommendedTalkRoom, len(talkRooms))
    for i, tr := range talkRooms {
        result[i] = RecommendedTalkRoom{
            TalkRoom:     tr,
            MessageCount: int(messageCounts[tr.ID]),  // O(1)アクセス
        }
    }

    return result, nil
}
```

## 実装のポイント

### 1. GROUP BYでカウントを集計

```sql
SELECT talk_room_id, COUNT(*) as count
FROM talk_messages
WHERE talk_room_id IN ('id1', 'id2', 'id3')
GROUP BY talk_room_id
```

1回のクエリで全トークルームのメッセージ数を取得できます。

### 2. Map構造でO(1)アクセス

```go
counts := make(map[string]int64)
// ...

// ループ内でO(1)でアクセス
messageCount := counts[talkRoom.ID]
```

### 3. 空の入力への対応

```go
if len(talkRoomIDs) == 0 {
    return make(map[string]int64), nil
}
```

空のスライスが渡された場合、不要なクエリを実行しないようにします。

## GORMのPreloadを使う方法

関連テーブルの読み込みには、GORMの`Preload`も有効です：

```go
// メンバー情報をプリロード
db.Preload("Members").Find(&talkRooms)

// 条件付きプリロード
db.Preload("Members", "is_active = ?", true).Find(&talkRooms)

// ネストしたプリロード
db.Preload("Members.User").Find(&talkRooms)
```

ただし、カウントのみが必要な場合は、GROUP BYを使ったバッチ取得の方が効率的です。

## クエリ数の比較

| 操作 | 修正前 | 修正後 |
|------|--------|--------|
| トークルーム取得 | 1クエリ | 1クエリ |
| メッセージ数取得 | N回（10件なら10回） | 1回（GROUP BY） |
| 合計 | 11クエリ | 2クエリ |

## まとめ

- N+1問題はループ内でのクエリ発行が原因
- GROUP BY + IN句で一括取得するバッチメソッドを実装
- Map構造でO(1)アクセスを実現
- GORMのPreloadも活用可能

この最適化により、レスポンスタイムが5〜10倍改善しました。

## 参考資料

- [GORM - Preload](https://gorm.io/docs/preload.html)
- [SQLBoiler](https://github.com/volatiletech/sqlboiler)
