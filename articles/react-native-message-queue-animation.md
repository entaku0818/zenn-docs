---
title: "【React Native】メッセージ表示キューシステムとアニメーション完了通知の実装"
emoji: "💬"
type: "tech"
topics: ["ReactNative", "Animation", "Queue", "jotai"]
published: false
---

# この記事は？

チャットアプリやチュートリアルで、複数のメッセージを順番にアニメーション付きで表示したい場合があります。この記事では、メッセージをキューで管理し、アニメーション完了を検知して次のメッセージを表示するシステムの実装方法を解説します。

## 実現したいこと

- メッセージを順番に表示する
- 各メッセージにフェードインアニメーションを付ける
- アニメーションが完了したら次のメッセージを表示
- 途中でメッセージが追加されてもキューで管理

## アーキテクチャ

```
新規メッセージ → キュー → 表示 → アニメーション完了 → 次のメッセージ
                  ↑                              |
                  └──────────────────────────────┘
```

## 実装

### 1. メッセージキューの状態管理（jotai）

```typescript
// stores/messageQueue.ts
import { atom } from 'jotai'

export interface QueuedMessage {
  id: string
  content: string
  type: 'user' | 'system' | 'ai'
}

// メッセージキュー
export const messageQueueAtom = atom<QueuedMessage[]>([])

// 現在表示中のメッセージ
export const currentMessageAtom = atom<QueuedMessage | null>(null)

// アニメーション完了フラグ
export const animationFinishedAtom = atom(false)
```

### 2. キュー処理のカスタムフック

```typescript
// hooks/useMessageQueue.ts
import { useAtom } from 'jotai'
import { useCallback, useEffect } from 'react'
import {
  messageQueueAtom,
  currentMessageAtom,
  animationFinishedAtom,
} from '@/stores/messageQueue'

export const useMessageQueue = () => {
  const [queue, setQueue] = useAtom(messageQueueAtom)
  const [currentMessage, setCurrentMessage] = useAtom(currentMessageAtom)
  const [animationFinished, setAnimationFinished] = useAtom(animationFinishedAtom)

  // メッセージをキューに追加
  const enqueueMessage = useCallback((message: QueuedMessage) => {
    setQueue(prev => {
      // 重複チェック
      if (prev.some(m => m.id === message.id)) {
        return prev
      }
      return [...prev, message]
    })
  }, [setQueue])

  // 次のメッセージを表示
  const processNextMessage = useCallback(() => {
    if (queue.length === 0) {
      setCurrentMessage(null)
      return
    }

    const [nextMessage, ...remainingQueue] = queue
    setCurrentMessage(nextMessage)
    setQueue(remainingQueue)
    setAnimationFinished(false)
  }, [queue, setQueue, setCurrentMessage, setAnimationFinished])

  // アニメーション完了を通知
  const onAnimationComplete = useCallback(() => {
    setAnimationFinished(true)
  }, [setAnimationFinished])

  // アニメーション完了後に次のメッセージを処理
  useEffect(() => {
    if (animationFinished && currentMessage) {
      const timer = setTimeout(() => {
        processNextMessage()
      }, 300)
      return () => clearTimeout(timer)
    }
  }, [animationFinished, currentMessage, processNextMessage])

  // キューにメッセージがあり、表示中がなければ処理開始
  useEffect(() => {
    if (!currentMessage && queue.length > 0) {
      processNextMessage()
    }
  }, [currentMessage, queue.length, processNextMessage])

  return {
    currentMessage,
    enqueueMessage,
    onAnimationComplete,
    queueLength: queue.length,
  }
}
```

### 3. アニメーション付きメッセージコンポーネント

```typescript
// components/AnimatedMessage.tsx
import { useEffect, useRef } from 'react'
import { Animated, StyleSheet, Text, View } from 'react-native'

interface AnimatedMessageProps {
  message: {
    id: string
    content: string
    type: 'user' | 'system' | 'ai'
  }
  onAnimationComplete: () => void
}

export const AnimatedMessage = ({
  message,
  onAnimationComplete,
}: AnimatedMessageProps) => {
  const fadeAnim = useRef(new Animated.Value(0)).current
  const slideAnim = useRef(new Animated.Value(20)).current

  useEffect(() => {
    // フェードイン + スライドアップアニメーション
    Animated.parallel([
      Animated.timing(fadeAnim, {
        toValue: 1,
        duration: 300,
        useNativeDriver: true,
      }),
      Animated.timing(slideAnim, {
        toValue: 0,
        duration: 300,
        useNativeDriver: true,
      }),
    ]).start(() => {
      // アニメーション完了を通知
      onAnimationComplete()
    })
  }, [message.id, fadeAnim, slideAnim, onAnimationComplete])

  return (
    <Animated.View
      style={[
        styles.messageContainer,
        styles[message.type],
        {
          opacity: fadeAnim,
          transform: [{ translateY: slideAnim }],
        },
      ]}
    >
      <Text style={styles.messageText}>{message.content}</Text>
    </Animated.View>
  )
}

const styles = StyleSheet.create({
  messageContainer: {
    padding: 12,
    borderRadius: 16,
    marginVertical: 4,
    maxWidth: '80%',
  },
  user: {
    backgroundColor: '#007AFF',
    alignSelf: 'flex-end',
  },
  system: {
    backgroundColor: '#E5E5EA',
    alignSelf: 'center',
  },
  ai: {
    backgroundColor: '#34C759',
    alignSelf: 'flex-start',
  },
  messageText: {
    fontSize: 16,
    color: '#fff',
  },
})
```

### 4. 使用例：チュートリアルメッセージ

```typescript
// screens/TutorialScreen.tsx
import { useEffect } from 'react'
import { View } from 'react-native'
import { useMessageQueue } from '@/hooks/useMessageQueue'
import { AnimatedMessage } from '@/components/AnimatedMessage'

export const TutorialScreen = () => {
  const { currentMessage, enqueueMessage, onAnimationComplete } = useMessageQueue()

  useEffect(() => {
    const messages = [
      { id: '1', content: 'ようこそ！', type: 'system' as const },
      { id: '2', content: 'メッセージを送ってみましょう', type: 'ai' as const },
      { id: '3', content: '下のボタンをタップしてください', type: 'ai' as const },
    ]

    messages.forEach((msg, i) => {
      setTimeout(() => enqueueMessage(msg), i * 100)
    })
  }, [enqueueMessage])

  return (
    <View style={{ flex: 1, padding: 16 }}>
      {currentMessage && (
        <AnimatedMessage
          message={currentMessage}
          onAnimationComplete={onAnimationComplete}
        />
      )}
    </View>
  )
}
```

## 重複エンキューの防止

APIからメッセージを受信する際、同じメッセージが複数回追加されることがあります：

```typescript
const enqueueMessage = useCallback((message: QueuedMessage) => {
  setQueue(prev => {
    // IDで重複チェック
    if (prev.some(m => m.id === message.id)) {
      if (__DEV__) {
        console.warn(`Message ${message.id} is already in queue`)
      }
      return prev  // 変更しない
    }
    return [...prev, message]
  })
}, [setQueue])
```

## 自動スクロールとの連携

```typescript
const flatListRef = useRef<FlatList>(null)

const onAnimationComplete = useCallback(() => {
  setAnimationFinished(true)
  // 最下部にスクロール
  flatListRef.current?.scrollToEnd({ animated: true })
}, [setAnimationFinished])
```

## まとめ

- jotaiでメッセージキューと表示状態を管理
- `Animated.timing`でフェードイン/スライドアニメーション
- アニメーション完了コールバックで次のメッセージを処理
- 重複チェックで同じメッセージの二重追加を防止

## 参考資料

- [React Native - Animated](https://reactnative.dev/docs/animated)
- [jotai](https://jotai.org/)
