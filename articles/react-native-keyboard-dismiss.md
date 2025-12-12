---
title: "【React Native】キーボードを適切に閉じる実装パターン"
emoji: "⌨️"
type: "tech"
topics: ["ReactNative", "Keyboard", "UX", "UI"]
published: false
---

# この記事は？

React Nativeでキーボードが開いている状態で別の要素をタップした際、キーボードを閉じる実装パターンを解説します。適切なタイミングでキーボードを閉じることで、UXが大幅に向上します。

## 問題の状況

テキスト入力中に別のボタンやUI要素をタップしても、キーボードが閉じずに残ってしまう問題がよく発生します。

## 基本的な実装

### Keyboard.dismiss()の使用

```typescript
import { Keyboard } from 'react-native'

const handleButtonPress = () => {
  Keyboard.dismiss()
  // その他の処理...
}
```

### TouchableWithoutFeedbackでラップ

画面全体をタップでキーボードを閉じるパターン：

```typescript
import { TouchableWithoutFeedback, Keyboard, View } from 'react-native'

const Screen = () => {
  return (
    <TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
      <View style={{ flex: 1 }}>
        <TextInput placeholder="入力..." />
        {/* その他のコンテンツ */}
      </View>
    </TouchableWithoutFeedback>
  )
}
```

### KeyboardAvoidingViewとの組み合わせ

```typescript
import {
  KeyboardAvoidingView,
  TouchableWithoutFeedback,
  Keyboard,
  Platform,
} from 'react-native'

const Screen = () => {
  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      style={{ flex: 1 }}
    >
      <TouchableWithoutFeedback onPress={Keyboard.dismiss}>
        <View style={{ flex: 1 }}>
          {/* コンテンツ */}
        </View>
      </TouchableWithoutFeedback>
    </KeyboardAvoidingView>
  )
}
```

## 実践的なパターン

### パターン1: 特定のボタンタップ時に閉じる

```typescript
const AvatarSelector = () => {
  const handleAvatarSelect = (avatarId: string) => {
    // まずキーボードを閉じる
    Keyboard.dismiss()

    // その後セレクターを表示
    showAvatarPicker(avatarId)
  }

  return (
    <TouchableOpacity onPress={() => handleAvatarSelect('avatar1')}>
      <AvatarIcon />
    </TouchableOpacity>
  )
}
```

### パターン2: BottomSheet表示時に閉じる

```typescript
const openBottomSheet = () => {
  // BottomSheetを開く前にキーボードを閉じる
  Keyboard.dismiss()

  // 少し遅延させてからBottomSheetを開く
  setTimeout(() => {
    bottomSheetRef.current?.present()
  }, 100)
}
```

### パターン3: ScrollViewでのスクロール時に閉じる

```typescript
<ScrollView
  keyboardShouldPersistTaps="handled"
  onScrollBeginDrag={Keyboard.dismiss}
>
  {/* コンテンツ */}
</ScrollView>
```

### パターン4: FlatListでの実装

```typescript
<FlatList
  data={items}
  renderItem={renderItem}
  keyboardShouldPersistTaps="handled"
  onScrollBeginDrag={Keyboard.dismiss}
  keyboardDismissMode="on-drag"
/>
```

## keyboardShouldPersistTapsの使い分け

| 値 | 動作 |
|------|------|
| `never` | タップでキーボードを閉じる。ボタンは2回タップ必要 |
| `always` | キーボードは閉じない。ボタンは1回で反応 |
| `handled` | ボタンタップは1回で反応、他の場所はキーボードを閉じる |

ほとんどの場合、`handled`が最適です。

## カスタムフックの作成

```typescript
// hooks/useDismissKeyboard.ts
import { useCallback } from 'react'
import { Keyboard } from 'react-native'

export const useDismissKeyboard = () => {
  const dismissKeyboard = useCallback(() => {
    Keyboard.dismiss()
  }, [])

  const dismissKeyboardAndThen = useCallback(
    (callback: () => void, delay = 100) => {
      Keyboard.dismiss()
      setTimeout(callback, delay)
    },
    []
  )

  return {
    dismissKeyboard,
    dismissKeyboardAndThen,
  }
}
```

使用例：

```typescript
const MyComponent = () => {
  const { dismissKeyboardAndThen } = useDismissKeyboard()

  const handleOpenModal = () => {
    dismissKeyboardAndThen(() => {
      setModalVisible(true)
    })
  }

  return (
    <Button title="モーダルを開く" onPress={handleOpenModal} />
  )
}
```

## キーボードの状態を監視

```typescript
import { useEffect, useState } from 'react'
import { Keyboard, KeyboardEvent } from 'react-native'

export const useKeyboardVisible = () => {
  const [isVisible, setIsVisible] = useState(false)

  useEffect(() => {
    const showSubscription = Keyboard.addListener('keyboardDidShow', () => {
      setIsVisible(true)
    })
    const hideSubscription = Keyboard.addListener('keyboardDidHide', () => {
      setIsVisible(false)
    })

    return () => {
      showSubscription.remove()
      hideSubscription.remove()
    }
  }, [])

  return isVisible
}
```

## まとめ

- `Keyboard.dismiss()`で明示的にキーボードを閉じる
- BottomSheetやモーダル表示前にはキーボードを閉じる
- `keyboardShouldPersistTaps="handled"`を基本設定に
- スクロール時は`keyboardDismissMode="on-drag"`が便利

## 参考資料

- [React Native - Keyboard](https://reactnative.dev/docs/keyboard)
- [React Native - KeyboardAvoidingView](https://reactnative.dev/docs/keyboardavoidingview)
