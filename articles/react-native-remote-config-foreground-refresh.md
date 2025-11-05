---
title: "【React Native】フォアグラウンド復帰時にFirebase Remote Configをリフレッシュする"
emoji: "🔄"
type: "tech"
topics: ["ReactNative", "Firebase", "RemoteConfig", "Expo"]
published: false
---

# この記事は？

React Nativeアプリでバックグラウンドから復帰した際に、Firebase Remote Configの値が古いままになっていて困った経験はありませんか？この記事では、AppStateを使ってフォアグラウンド復帰を検知し、Remote Configを自動的にリフレッシュする方法を解説します。

## 問題の背景

Firebase Remote Configは、アプリの動作をサーバー側から動的に変更できる便利な機能です。しかし、以下のような課題があります：

- アプリ起動時にのみfetchするため、長時間バックグラウンドにいると値が古くなる
- ユーザーがアプリを再起動しないと新しい設定が反映されない
- A/Bテストやフィーチャーフラグの即時反映が難しい

## 実装方法

### useRemoteConfigRefreshフックの作成

```typescript
import { useEffect, useRef } from 'react'
import { AppState, type AppStateStatus } from 'react-native'

/**
 * アプリがフォアグラウンドに戻った時にRemote Configをリフレッシュするhook
 */
export const useRemoteConfigRefresh = (
  refreshConfig: () => Promise<void>
): void => {
  const appStateRef = useRef<AppStateStatus>(AppState.currentState)

  useEffect(() => {
    const subscription = AppState.addEventListener('change', nextAppState => {
      // バックグラウンド/非アクティブからアクティブに変化した場合
      if (
        appStateRef.current.match(/inactive|background/) &&
        nextAppState === 'active'
      ) {
        if (__DEV__) {
          console.log('🔄 [RemoteConfig] Refreshing config on app foreground')
        }
        refreshConfig()
      }
      appStateRef.current = nextAppState
    })

    return () => subscription.remove()
  }, [refreshConfig])
}
```

### 使い方

ルートレイアウトなど、アプリ全体で一度だけ呼び出す場所でフックを使用します：

```typescript
// app/_layout.tsx
import { useRemoteConfigRefresh } from '@/hooks/useRemoteConfigRefresh'
import { useRemoteConfig } from '@/lib/remoteConfig'

export default function RootLayout() {
  const { refreshConfig } = useRemoteConfig()

  // フォアグラウンド復帰時にRemote Configをリフレッシュ
  useRemoteConfigRefresh(refreshConfig)

  return (
    // ...
  )
}
```

## 実装のポイント

### 1. AppStateの状態遷移を正確に検知する

AppStateには3つの状態があります：

| 状態 | 説明 |
|------|------|
| `active` | アプリがフォアグラウンドで動作中 |
| `inactive` | 遷移中の状態（電話着信など） |
| `background` | アプリがバックグラウンドにある |

`inactive`または`background`から`active`への遷移を検知することで、確実にフォアグラウンド復帰を捕捉できます。

### 2. useRefで前の状態を保持する

```typescript
const appStateRef = useRef<AppStateStatus>(AppState.currentState)
```

前の状態を`useRef`で保持することで、状態の変化（遷移）を正確に判定できます。`useState`を使うと再レンダリングが発生してしまうため、`useRef`が適切です。

### 3. クリーンアップを忘れない

```typescript
return () => subscription.remove()
```

イベントリスナーの解除を忘れると、メモリリークやコンポーネントのアンマウント後にエラーが発生する原因になります。

## テストコード

```typescript
import { renderHook } from '@testing-library/react-native'
import { AppState } from 'react-native'
import { useRemoteConfigRefresh } from './useRemoteConfigRefresh'

describe('useRemoteConfigRefresh', () => {
  let appStateCallback: (state: string) => void

  beforeEach(() => {
    jest.spyOn(AppState, 'addEventListener').mockImplementation(
      (type, callback) => {
        appStateCallback = callback
        return { remove: jest.fn() }
      }
    )
  })

  it('バックグラウンドからアクティブになった時にrefreshConfigを呼ぶ', () => {
    const refreshConfig = jest.fn()
    renderHook(() => useRemoteConfigRefresh(refreshConfig))

    // バックグラウンド状態をシミュレート
    appStateCallback('background')
    // アクティブに復帰
    appStateCallback('active')

    expect(refreshConfig).toHaveBeenCalledTimes(1)
  })

  it('アクティブのままの場合はrefreshConfigを呼ばない', () => {
    const refreshConfig = jest.fn()
    renderHook(() => useRemoteConfigRefresh(refreshConfig))

    // アクティブのまま
    appStateCallback('active')
    appStateCallback('active')

    expect(refreshConfig).not.toHaveBeenCalled()
  })
})
```

## 注意点

### リフレッシュ頻度の制御

頻繁にフォアグラウンド復帰が発生する場合、Remote Configへのリクエストが増えすぎる可能性があります。必要に応じて、最後のフェッチからの経過時間をチェックするロジックを追加しましょう：

```typescript
const lastFetchRef = useRef<number>(0)
const MIN_FETCH_INTERVAL = 60 * 1000 // 1分

useEffect(() => {
  const subscription = AppState.addEventListener('change', nextAppState => {
    if (
      appStateRef.current.match(/inactive|background/) &&
      nextAppState === 'active'
    ) {
      const now = Date.now()
      if (now - lastFetchRef.current > MIN_FETCH_INTERVAL) {
        refreshConfig()
        lastFetchRef.current = now
      }
    }
    appStateRef.current = nextAppState
  })

  return () => subscription.remove()
}, [refreshConfig])
```

## まとめ

- `AppState`のイベントリスナーでフォアグラウンド復帰を検知
- `useRef`で前の状態を保持し、状態遷移を正確に判定
- クリーンアップ関数でメモリリークを防止
- 必要に応じてリフレッシュ頻度を制御

この実装により、ユーザーがアプリを再起動しなくても、常に最新のRemote Config設定が適用されるようになります。

## 参考資料

- [React Native - AppState](https://reactnative.dev/docs/appstate)
- [Firebase Remote Config](https://firebase.google.com/docs/remote-config)
