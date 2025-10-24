---
title: "【React Native】BottomSheetModalのstackBehaviorでモーダルオンモーダルを実現する"
emoji: "📚"
type: "tech"
topics: ["ReactNative", "BottomSheet", "Modal", "UI"]
published: false
---

# この記事は？

@gorhom/bottom-sheetで、BottomSheetの上にさらにBottomSheetを重ねて表示したいことはありませんか？例えば、設定画面のBottomSheetから確認ダイアログを表示するケースです。この記事では、`stackBehavior`プロパティを使ってモーダルオンモーダルを実現する方法を解説します。

## 問題の状況

デフォルトでは、新しいBottomSheetModalを開くと、既存のBottomSheetModalが閉じられてしまいます：

```
期待する動作:
  [設定BottomSheet]
        ↓ 確認ボタンをタップ
  [設定BottomSheet] + [確認BottomSheet（上に重なる）]

実際の動作:
  [設定BottomSheet]
        ↓ 確認ボタンをタップ
  [確認BottomSheet]（設定BottomSheetが閉じてしまう）
```

## 解決方法

`BottomSheetModalProvider`と`stackBehavior="push"`を使用します。

### 1. BottomSheetModalProviderの設定

```typescript
// app/_layout.tsx
import { BottomSheetModalProvider } from '@gorhom/bottom-sheet'
import { GestureHandlerRootView } from 'react-native-gesture-handler'

export default function RootLayout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <BottomSheetModalProvider>
        <Stack />
      </BottomSheetModalProvider>
    </GestureHandlerRootView>
  )
}
```

### 2. stackBehaviorの設定

各BottomSheetModalに`stackBehavior="push"`を設定：

```typescript
import { BottomSheetModal } from '@gorhom/bottom-sheet'

const SettingsBottomSheet = forwardRef<BottomSheetModal>((props, ref) => {
  return (
    <BottomSheetModal
      ref={ref}
      stackBehavior="push"  // ← これが重要
      snapPoints={['50%', '90%']}
    >
      <SettingsContent />
    </BottomSheetModal>
  )
})

const ConfirmBottomSheet = forwardRef<BottomSheetModal>((props, ref) => {
  return (
    <BottomSheetModal
      ref={ref}
      stackBehavior="push"  // ← これが重要
      snapPoints={['30%']}
    >
      <ConfirmContent />
    </BottomSheetModal>
  )
})
```

### 3. 実装例

```typescript
export const SettingsScreen = () => {
  const settingsRef = useRef<BottomSheetModal>(null)
  const confirmRef = useRef<BottomSheetModal>(null)

  const handleDeleteAccount = () => {
    // 設定BottomSheetを閉じずに確認BottomSheetを開く
    confirmRef.current?.present()
  }

  const handleConfirmDelete = () => {
    confirmRef.current?.dismiss()
    settingsRef.current?.dismiss()
    // 削除処理...
  }

  return (
    <View>
      <Button title="設定" onPress={() => settingsRef.current?.present()} />

      <SettingsBottomSheet ref={settingsRef}>
        <Button title="アカウント削除" onPress={handleDeleteAccount} />
      </SettingsBottomSheet>

      <ConfirmBottomSheet ref={confirmRef}>
        <Text>本当に削除しますか？</Text>
        <Button title="削除" onPress={handleConfirmDelete} />
        <Button title="キャンセル" onPress={() => confirmRef.current?.dismiss()} />
      </ConfirmBottomSheet>
    </View>
  )
}
```

## stackBehaviorの種類

| 値 | 動作 |
|------|------|
| `push` | 新しいモーダルを上に重ねる |
| `replace` | 既存を閉じて新しいモーダルを表示（デフォルト） |

## まとめ

- `BottomSheetModalProvider`でラップ
- `stackBehavior="push"`を設定
- `present()`で上に重ねて表示
- `dismiss()`で上から順番に閉じる

## 参考資料

- [@gorhom/bottom-sheet](https://gorhom.github.io/react-native-bottom-sheet/modal)
