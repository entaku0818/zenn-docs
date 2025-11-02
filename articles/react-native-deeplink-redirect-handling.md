---
title: "【React Native】ディープリンクで無効な画面にアクセスした際のリダイレクト処理"
emoji: "🔗"
type: "tech"
topics: ["ReactNative", "ExpoRouter", "DeepLink", "Navigation"]
published: false
---

# この記事は？

ディープリンクで削除済みや非アクティブなコンテンツにアクセスした際、白い画面が表示されてしまう問題に遭遇したことはありませんか？この記事では、Expo Routerを使用したアプリで、無効なリソースへのアクセス時に適切にホーム画面へリダイレクトする方法を解説します。

## 問題の状況

ディープリンク（例: `myapp://talkrooms/abc123`）でアプリを開いた際、対象のトークルームが以下の状態だと問題が発生します：

- 削除済み
- 終了済み（closed）
- 非アクティブ

```
期待する動作: ホーム画面にリダイレクト
実際の動作:   白い画面が表示される（null returnのため）
```

## 修正前のコード（問題あり）

```typescript
// app/talkrooms/[talkRoomId]/index.tsx
export default function TalkRoomPage() {
  const { talkRoomId } = useLocalSearchParams<{ talkRoomId: string }>()
  const { data: talkRoom, isLoading } = useTalkRoom({ talkRoomId })

  if (isLoading) {
    return <LoadingScreen />
  }

  // 問題: nullを返すと白い画面になる
  if (!talkRoom || talkRoom.status !== 'active') {
    return null
  }

  return <TalkRoomContent talkRoom={talkRoom} />
}
```

## 修正後のコード

```typescript
// app/talkrooms/[talkRoomId]/index.tsx
import { useRouter } from 'expo-router'
import { useEffect } from 'react'

export default function TalkRoomPage() {
  const router = useRouter()
  const { talkRoomId } = useLocalSearchParams<{ talkRoomId: string }>()
  const { data: talkRoom, isLoading } = useTalkRoom({ talkRoomId })

  // 無効なトークルームの場合はホームにリダイレクト
  useEffect(() => {
    if (!isLoading && (!talkRoom || talkRoom.status !== 'active')) {
      router.replace('/')
    }
  }, [isLoading, talkRoom, router])

  if (isLoading) {
    return <LoadingScreen />
  }

  // リダイレクト中はローディングを表示
  if (!talkRoom || talkRoom.status !== 'active') {
    return <LoadingScreen />
  }

  return <TalkRoomContent talkRoom={talkRoom} />
}
```

## 実装のポイント

### 1. `router.replace` vs `router.push`

```typescript
// replace: 履歴を置き換える（戻るボタンで無効な画面に戻らない）
router.replace('/')

// push: 履歴に追加する（戻るボタンで無効な画面に戻ってしまう）
router.push('/')
```

`replace`を使用することで、ユーザーが戻るボタンを押しても無効な画面に戻ることを防ぎます。

### 2. ローディング中はリダイレクトしない

```typescript
if (!isLoading && (!talkRoom || talkRoom.status !== 'active')) {
  router.replace('/')
}
```

`isLoading`をチェックすることで、データ取得が完了する前にリダイレクトしてしまうことを防ぎます。

### 3. リダイレクト中もUIを表示する

```typescript
// リダイレクト処理中はローディング画面を表示
if (!talkRoom || talkRoom.status !== 'active') {
  return <LoadingScreen />
}
```

`useEffect`でのリダイレクトは非同期で実行されるため、その間は白い画面ではなくローディング画面を表示します。

## より堅牢な実装

複数の条件をチェックする必要がある場合の実装例：

```typescript
export default function TalkRoomPage() {
  const router = useRouter()
  const { talkRoomId } = useLocalSearchParams<{ talkRoomId: string }>()
  const { data: talkRoom, isLoading, error } = useTalkRoom({ talkRoomId })

  const shouldRedirect = useMemo(() => {
    if (isLoading) return false
    if (error) return true
    if (!talkRoom) return true
    if (talkRoom.status === 'closed') return true
    if (talkRoom.status === 'deleted') return true
    if (!talkRoom.isActive) return true
    return false
  }, [isLoading, error, talkRoom])

  useEffect(() => {
    if (shouldRedirect) {
      router.replace('/')
    }
  }, [shouldRedirect, router])

  if (isLoading || shouldRedirect) {
    return <LoadingScreen />
  }

  return <TalkRoomContent talkRoom={talkRoom} />
}
```

## カスタムフックに切り出す

複数の画面で同じロジックを使用する場合は、カスタムフックに切り出します：

```typescript
// hooks/useRedirectOnInvalid.ts
import { useRouter } from 'expo-router'
import { useEffect } from 'react'

interface UseRedirectOnInvalidOptions {
  isLoading: boolean
  isValid: boolean
  redirectTo?: string
}

export const useRedirectOnInvalid = ({
  isLoading,
  isValid,
  redirectTo = '/',
}: UseRedirectOnInvalidOptions) => {
  const router = useRouter()

  useEffect(() => {
    if (!isLoading && !isValid) {
      router.replace(redirectTo)
    }
  }, [isLoading, isValid, redirectTo, router])

  return { shouldShowLoading: isLoading || !isValid }
}
```

使用例：

```typescript
export default function TalkRoomPage() {
  const { talkRoomId } = useLocalSearchParams<{ talkRoomId: string }>()
  const { data: talkRoom, isLoading } = useTalkRoom({ talkRoomId })

  const { shouldShowLoading } = useRedirectOnInvalid({
    isLoading,
    isValid: !!talkRoom && talkRoom.status === 'active',
  })

  if (shouldShowLoading) {
    return <LoadingScreen />
  }

  return <TalkRoomContent talkRoom={talkRoom!} />
}
```

## まとめ

- ディープリンクで無効なリソースにアクセスした場合は`router.replace`でリダイレクト
- データ取得完了前にリダイレクトしないよう`isLoading`をチェック
- リダイレクト中は白い画面ではなくローディング画面を表示
- 複数画面で使用する場合はカスタムフックに切り出す

この実装により、ユーザーが古いディープリンクやシェアされたリンクからアクセスしても、適切にホーム画面に誘導できます。

## 参考資料

- [Expo Router - Routing](https://docs.expo.dev/router/navigating-pages/)
- [Expo Router - useRouter](https://docs.expo.dev/router/navigating-pages/#userouter)
