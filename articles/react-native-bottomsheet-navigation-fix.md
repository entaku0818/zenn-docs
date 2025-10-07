---
title: "【React Native】ページ遷移時にBottomSheetが開いたまま残る問題の修正"
emoji: "📱"
type: "tech"
topics: ["ReactNative", "BottomSheet", "ExpoRouter", "Navigation"]
published: false
---

# この記事は？

React Nativeで@gorhom/bottom-sheetを使用していて、画面遷移時にBottomSheetが閉じずに表示され続けてしまう問題に遭遇したことはありませんか？この記事では、`useIsFocused`フックを使ってこの問題を解決する方法を解説します。

## 問題の状況

@gorhom/bottom-sheetは非常に人気のあるBottomSheetライブラリですが、以下のような状況で問題が発生します：

- ホーム画面でBottomSheetを開いた状態で、通知やディープリンクから別の画面に遷移
- BottomSheetが開いたまま新しい画面が表示される
- 見た目がおかしく、操作に支障が出る

```
遷移前: ホーム画面 + BottomSheet（開）
         ↓ ディープリンクで遷移
遷移後: 詳細画面 + BottomSheet（開いたまま！）
```

## 原因

React NavigationやExpo Routerでは、画面遷移時に前の画面のコンポーネントが即座にアンマウントされるわけではありません。そのため、BottomSheetの状態が保持されたまま新しい画面がマウントされてしまいます。

## 解決方法

`@react-navigation/native`の`useIsFocused`フックを使用して、画面がフォーカスを失った際にBottomSheetを閉じます。

### 実装コード

```typescript
import { useIsFocused } from '@react-navigation/native'
import { useEffect, useRef, useState } from 'react'
import { BottomSheetModal } from '@gorhom/bottom-sheet'

const HomePage = () => {
  const isFocused = useIsFocused()

  // BottomSheetの参照
  const settingsBottomSheetRef = useRef<BottomSheetModal>(null)
  const talkBottomSheetRef = useRef<BottomSheetModal>(null)

  // BottomSheetの表示状態
  const [isSettingsVisible, setIsSettingsVisible] = useState(false)
  const [isTalkVisible, setIsTalkVisible] = useState(false)

  // フォーカスを失った時にBottomSheetを閉じる
  useEffect(() => {
    if (!isFocused) {
      // 状態をリセット
      setIsSettingsVisible(false)
      setIsTalkVisible(false)

      // BottomSheetを閉じる
      settingsBottomSheetRef.current?.dismiss()
      talkBottomSheetRef.current?.dismiss()
    }
  }, [isFocused])

  return (
    <>
      {/* 画面のコンテンツ */}

      <SettingsBottomSheet
        ref={settingsBottomSheetRef}
        isVisible={isSettingsVisible}
        onClose={() => setIsSettingsVisible(false)}
      />

      <TalkBottomSheet
        ref={talkBottomSheetRef}
        isVisible={isTalkVisible}
        onClose={() => setIsTalkVisible(false)}
      />
    </>
  )
}
```

### トークルーム画面での適用例

同様に、トークルーム画面でも複数のBottomSheetを管理する場合があります：

```typescript
const TalkRoom = () => {
  const isFocused = useIsFocused()

  const userActionRef = useRef<BottomSheetModal>(null)
  const aiAvatarRef = useRef<BottomSheetModal>(null)
  const reportRef = useRef<BottomSheetModal>(null)
  const blockConfirmRef = useRef<BottomSheetModal>(null)

  const [isUserActionVisible, setIsUserActionVisible] = useState(false)
  const [isAiAvatarVisible, setIsAiAvatarVisible] = useState(false)
  const [isReportVisible, setIsReportVisible] = useState(false)
  const [isBlockConfirmVisible, setIsBlockConfirmVisible] = useState(false)

  useEffect(() => {
    if (!isFocused) {
      // すべてのBottomSheetを閉じる
      setIsUserActionVisible(false)
      setIsAiAvatarVisible(false)
      setIsReportVisible(false)
      setIsBlockConfirmVisible(false)

      userActionRef.current?.dismiss()
      aiAvatarRef.current?.dismiss()
      reportRef.current?.dismiss()
      blockConfirmRef.current?.dismiss()
    }
  }, [isFocused])

  // ...
}
```

## なぜuseIsFocusedを使うのか？

### 他のアプローチとの比較

| アプローチ | メリット | デメリット |
|------------|----------|------------|
| `useIsFocused` | シンプル、確実に動作 | React Navigationに依存 |
| `useFocusEffect` | クリーンアップが明示的 | コードが複雑になる |
| `navigation.addListener` | 細かい制御が可能 | ボイラープレートが多い |
| コンポーネントのアンマウント | 根本的解決 | 状態がすべてリセットされる |

`useIsFocused`は最もシンプルで、画面がフォーカスを失ったタイミングで確実にBottomSheetを閉じることができます。

### useFocusEffectを使う場合

クリーンアップ処理を明示的に書きたい場合は`useFocusEffect`も有効です：

```typescript
import { useFocusEffect } from '@react-navigation/native'
import { useCallback } from 'react'

const HomePage = () => {
  const bottomSheetRef = useRef<BottomSheetModal>(null)

  useFocusEffect(
    useCallback(() => {
      // フォーカス時の処理（必要であれば）

      return () => {
        // フォーカスを失った時のクリーンアップ
        bottomSheetRef.current?.dismiss()
      }
    }, [])
  )

  // ...
}
```

## 実装のポイント

### 1. 状態とrefの両方をリセットする

```typescript
// 状態をリセット（UIの更新）
setIsSettingsVisible(false)

// refを通じてBottomSheetを閉じる（実際の閉じる動作）
settingsBottomSheetRef.current?.dismiss()
```

状態だけをリセットしても、BottomSheetのアニメーションが完了する前に画面が遷移してしまうと、表示が残ることがあります。`dismiss()`メソッドを呼び出すことで、確実にBottomSheetを閉じます。

### 2. 複数のBottomSheetを一括で閉じる

```typescript
const bottomSheetRefs = [
  settingsBottomSheetRef,
  talkBottomSheetRef,
  userActionRef,
]

useEffect(() => {
  if (!isFocused) {
    bottomSheetRefs.forEach(ref => ref.current?.dismiss())
  }
}, [isFocused])
```

## 対応するケース

この修正により、以下のすべてのケースでBottomSheetが正しく閉じられます：

- 通常のナビゲーション（戻るボタン、タブ切り替え）
- ディープリンクからの遷移
- プッシュ通知タップからの遷移
- プログラム的なナビゲーション（`router.push()`など）

## まとめ

- `useIsFocused`フックで画面のフォーカス状態を監視
- フォーカスを失った時にBottomSheetを`dismiss()`で閉じる
- 状態（useState）とref（dismiss）の両方をリセット
- 複数のBottomSheetがある場合はすべてを閉じる

この実装により、どのような遷移方法でも確実にBottomSheetが閉じられ、ユーザー体験が向上します。

## 参考資料

- [@gorhom/bottom-sheet](https://github.com/gorhom/react-native-bottom-sheet)
- [React Navigation - useIsFocused](https://reactnavigation.org/docs/use-is-focused/)
- [React Navigation - useFocusEffect](https://reactnavigation.org/docs/use-focus-effect/)
