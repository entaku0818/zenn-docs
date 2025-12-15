---
title: "【React Native】Pusherから受信するデータのバリデーション実装"
emoji: "🛡️"
type: "tech"
topics: ["ReactNative", "Pusher", "WebSocket", "バリデーション", "TypeScript"]
published: false
---

# この記事は？

Pusher（WebSocket）経由で受信するデータは、サーバーサイドのバグや不正なクライアントにより、想定外の形式で届くことがあります。この記事では、受信データのバリデーションを実装してアプリをクラッシュから守る方法を解説します。

## 問題の状況

リアルタイムでリアクション機能を実装していたところ、不正なリアクションタイプがPusherから届き、アプリがクラッシュする問題が発生しました。

```typescript
// 想定していたデータ
{ type: 'heart', userId: 'user123' }

// 実際に届いたデータ（サーバーのバグ）
{ type: 'invalid_type', userId: 'user123' }
// → アプリがクラッシュ
```

## 解決方法

### 1. 型定義とバリデーション関数

```typescript
// types/reaction.ts
export const REACTION_TYPES = ['heart', 'smile', 'thumbsUp', 'clap'] as const
export type ReactionType = (typeof REACTION_TYPES)[number]

export interface ReactionEvent {
  type: ReactionType
  userId: string
  messageId: string
  createdAt: string
}

// utils/validation.ts
export const isValidReactionType = (type: unknown): type is ReactionType => {
  return (
    typeof type === 'string' &&
    REACTION_TYPES.includes(type as ReactionType)
  )
}

export const isValidReactionEvent = (data: unknown): data is ReactionEvent => {
  if (typeof data !== 'object' || data === null) {
    return false
  }

  const event = data as Record<string, unknown>

  return (
    isValidReactionType(event.type) &&
    typeof event.userId === 'string' &&
    typeof event.messageId === 'string' &&
    typeof event.createdAt === 'string'
  )
}
```

### 2. Pusherイベントハンドラーでのバリデーション

```typescript
// hooks/usePusherReactions.ts
import { useEffect } from 'react'
import { pusherClient } from '../lib/pusher'
import { isValidReactionEvent } from '../utils/validation'

export const usePusherReactions = (
  channelName: string,
  onReaction: (event: ReactionEvent) => void
) => {
  useEffect(() => {
    const channel = pusherClient.subscribe(channelName)

    channel.bind('reaction', (data: unknown) => {
      // バリデーション
      if (!isValidReactionEvent(data)) {
        console.warn('Invalid reaction event received:', data)
        return // 不正なデータは無視
      }

      // 有効なデータのみ処理
      onReaction(data)
    })

    return () => {
      channel.unbind('reaction')
      pusherClient.unsubscribe(channelName)
    }
  }, [channelName, onReaction])
}
```

### 3. zodを使ったバリデーション（推奨）

より堅牢なバリデーションにはzodを使用します。

```typescript
// schemas/reaction.ts
import { z } from 'zod'

export const ReactionTypeSchema = z.enum(['heart', 'smile', 'thumbsUp', 'clap'])

export const ReactionEventSchema = z.object({
  type: ReactionTypeSchema,
  userId: z.string().min(1),
  messageId: z.string().min(1),
  createdAt: z.string().datetime(),
})

export type ReactionType = z.infer<typeof ReactionTypeSchema>
export type ReactionEvent = z.infer<typeof ReactionEventSchema>
```

使用例：

```typescript
channel.bind('reaction', (data: unknown) => {
  const result = ReactionEventSchema.safeParse(data)

  if (!result.success) {
    console.warn('Invalid reaction event:', result.error.errors)
    return
  }

  onReaction(result.data)
})
```

### 4. エラーレポートの追加

不正なデータを受信した場合、ログを残して調査できるようにします。

```typescript
import * as Sentry from '@sentry/react-native'

channel.bind('reaction', (data: unknown) => {
  const result = ReactionEventSchema.safeParse(data)

  if (!result.success) {
    // Sentryにエラーを報告
    Sentry.captureMessage('Invalid Pusher event received', {
      level: 'warning',
      extra: {
        eventName: 'reaction',
        receivedData: JSON.stringify(data),
        validationErrors: result.error.errors,
      },
    })
    return
  }

  onReaction(result.data)
})
```

## 汎用的なPusherイベントラッパー

```typescript
// lib/pusherEventHandler.ts
import { z, ZodSchema } from 'zod'

export const createSafeEventHandler = <T>(
  schema: ZodSchema<T>,
  handler: (data: T) => void,
  options?: {
    onError?: (error: z.ZodError, rawData: unknown) => void
  }
) => {
  return (data: unknown) => {
    const result = schema.safeParse(data)

    if (!result.success) {
      options?.onError?.(result.error, data)
      console.warn('Invalid event data:', result.error.errors)
      return
    }

    handler(result.data)
  }
}
```

使用例：

```typescript
const handleReaction = createSafeEventHandler(
  ReactionEventSchema,
  (event) => {
    // 型安全にイベントを処理
    addReaction(event)
  },
  {
    onError: (error, rawData) => {
      Sentry.captureMessage('Invalid reaction event', {
        extra: { error, rawData },
      })
    },
  }
)

channel.bind('reaction', handleReaction)
```

## フォールバック値の使用

一部のフィールドが不正でも処理を続行したい場合：

```typescript
const ReactionEventWithDefaultsSchema = z.object({
  type: ReactionTypeSchema.catch('heart'), // 不正な場合はheartに
  userId: z.string().min(1),
  messageId: z.string().min(1),
  createdAt: z.string().catch(new Date().toISOString()),
})
```

## まとめ

- Pusherからのデータは必ずバリデーション
- zodを使うと型安全で堅牢なバリデーションが可能
- 不正データはログに残して調査可能に
- 必要に応じてフォールバック値を設定

## 参考資料

- [Pusher - Handling events](https://pusher.com/docs/channels/using_channels/events/)
- [Zod - TypeScript-first schema validation](https://zod.dev/)
