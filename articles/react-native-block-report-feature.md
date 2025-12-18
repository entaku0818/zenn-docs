---
title: "【React Native】ブロック・通報機能を実装する"
emoji: "🚫"
type: "tech"
topics: ["ReactNative", "UGC", "モデレーション", "セキュリティ"]
published: false
---

# この記事は？

UGC（User Generated Content）アプリでは、ブロック・通報機能が必須です。この記事では、React Nativeでのブロック・通報機能の実装パターンを解説します。

## 機能要件

1. **ブロック機能**: 特定ユーザーのコンテンツを非表示にする
2. **通報機能**: 不適切なコンテンツをモデレーターに報告する

## UI実装

### 1. ActionSheetでメニュー表示

```typescript
// components/UserActionSheet.tsx
import { ActionSheetIOS, Platform, Alert } from 'react-native'

interface UserActionSheetProps {
  userId: string
  userName: string
  onBlock: (userId: string) => void
  onReport: (userId: string, reason: string) => void
}

export const showUserActionSheet = ({
  userId,
  userName,
  onBlock,
  onReport,
}: UserActionSheetProps) => {
  const options = ['キャンセル', 'ブロック', '通報する']
  const destructiveButtonIndex = 1
  const cancelButtonIndex = 0

  if (Platform.OS === 'ios') {
    ActionSheetIOS.showActionSheetWithOptions(
      {
        options,
        destructiveButtonIndex,
        cancelButtonIndex,
        title: userName,
        message: 'このユーザーに対するアクションを選択',
      },
      (buttonIndex) => {
        if (buttonIndex === 1) {
          confirmBlock(userId, userName, onBlock)
        } else if (buttonIndex === 2) {
          showReportReasonPicker(userId, onReport)
        }
      }
    )
  }
}

const confirmBlock = (
  userId: string,
  userName: string,
  onBlock: (userId: string) => void
) => {
  Alert.alert(
    'ブロックしますか？',
    `${userName}をブロックすると、このユーザーのメッセージが表示されなくなります。`,
    [
      { text: 'キャンセル', style: 'cancel' },
      {
        text: 'ブロック',
        style: 'destructive',
        onPress: () => onBlock(userId),
      },
    ]
  )
}
```

### 2. 通報理由の選択

```typescript
// components/ReportReasonPicker.tsx
import { Alert } from 'react-native'

const REPORT_REASONS = [
  { id: 'spam', label: 'スパム・宣伝' },
  { id: 'harassment', label: '嫌がらせ・いじめ' },
  { id: 'inappropriate', label: '不適切なコンテンツ' },
  { id: 'violence', label: '暴力的な表現' },
  { id: 'other', label: 'その他' },
]

export const showReportReasonPicker = (
  userId: string,
  onReport: (userId: string, reason: string) => void
) => {
  Alert.alert(
    '通報理由を選択',
    'このユーザーを通報する理由を選んでください',
    [
      ...REPORT_REASONS.map((reason) => ({
        text: reason.label,
        onPress: () => onReport(userId, reason.id),
      })),
      { text: 'キャンセル', style: 'cancel' },
    ]
  )
}
```

## 状態管理（Jotai）

```typescript
// atoms/blockAtom.ts
import { atom } from 'jotai'
import { atomWithStorage, createJSONStorage } from 'jotai/utils'
import AsyncStorage from '@react-native-async-storage/async-storage'

const storage = createJSONStorage<string[]>(() => AsyncStorage)

// ブロックしたユーザーIDのリスト
export const blockedUserIdsAtom = atomWithStorage<string[]>(
  'blockedUserIds',
  [],
  storage
)

// ブロック追加
export const addBlockedUserAtom = atom(
  null,
  (get, set, userId: string) => {
    const current = get(blockedUserIdsAtom)
    if (!current.includes(userId)) {
      set(blockedUserIdsAtom, [...current, userId])
    }
  }
)

// ブロック解除
export const removeBlockedUserAtom = atom(
  null,
  (get, set, userId: string) => {
    const current = get(blockedUserIdsAtom)
    set(blockedUserIdsAtom, current.filter((id) => id !== userId))
  }
)

// ブロック判定
export const isBlockedAtom = atom((get) => {
  const blockedIds = get(blockedUserIdsAtom)
  return (userId: string) => blockedIds.includes(userId)
})
```

## メッセージリストでのフィルタリング

```typescript
// hooks/useFilteredMessages.ts
import { useMemo } from 'react'
import { useAtomValue } from 'jotai'
import { blockedUserIdsAtom } from '../atoms/blockAtom'

interface Message {
  id: string
  userId: string
  content: string
}

export const useFilteredMessages = (messages: Message[]) => {
  const blockedUserIds = useAtomValue(blockedUserIdsAtom)

  const filteredMessages = useMemo(() => {
    return messages.filter(
      (message) => !blockedUserIds.includes(message.userId)
    )
  }, [messages, blockedUserIds])

  return filteredMessages
}
```

使用例：

```typescript
const MessageList = () => {
  const { data: messages } = useMessages()
  const filteredMessages = useFilteredMessages(messages ?? [])

  return (
    <FlatList
      data={filteredMessages}
      renderItem={({ item }) => <MessageItem message={item} />}
    />
  )
}
```

## API連携

### ブロックAPI

```typescript
// api/block.ts
export const blockUser = async (targetUserId: string): Promise<void> => {
  await fetch(`${API_BASE_URL}/users/${targetUserId}/block`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${await getToken()}`,
    },
  })
}

export const unblockUser = async (targetUserId: string): Promise<void> => {
  await fetch(`${API_BASE_URL}/users/${targetUserId}/block`, {
    method: 'DELETE',
    headers: {
      Authorization: `Bearer ${await getToken()}`,
    },
  })
}
```

### 通報API

```typescript
// api/report.ts
interface ReportPayload {
  targetUserId: string
  reason: string
  contentId?: string  // 特定のメッセージを通報する場合
  description?: string
}

export const reportUser = async (payload: ReportPayload): Promise<void> => {
  await fetch(`${API_BASE_URL}/reports`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${await getToken()}`,
    },
    body: JSON.stringify(payload),
  })
}
```

## カスタムフック

```typescript
// hooks/useUserActions.ts
import { useSetAtom } from 'jotai'
import { addBlockedUserAtom } from '../atoms/blockAtom'
import { blockUser, reportUser } from '../api'
import { Alert } from 'react-native'

export const useUserActions = () => {
  const addBlockedUser = useSetAtom(addBlockedUserAtom)

  const handleBlock = async (userId: string) => {
    try {
      await blockUser(userId)
      addBlockedUser(userId)
      Alert.alert('ブロックしました', 'このユーザーは表示されなくなります')
    } catch (error) {
      Alert.alert('エラー', 'ブロックに失敗しました')
    }
  }

  const handleReport = async (userId: string, reason: string) => {
    try {
      await reportUser({ targetUserId: userId, reason })
      Alert.alert(
        '通報を受け付けました',
        'ご報告ありがとうございます。内容を確認いたします。'
      )
    } catch (error) {
      Alert.alert('エラー', '通報に失敗しました')
    }
  }

  return { handleBlock, handleReport }
}
```

## App Store審査対策

Apple/Googleはブロック・通報機能を必須としています。

### 必要な要素

1. ブロック機能が容易にアクセス可能
2. 通報機能でカテゴリ選択ができる
3. 通報後のフィードバック表示
4. ブロック解除ができる設定画面

### 設定画面でのブロックリスト管理

```typescript
const BlockedUsersScreen = () => {
  const blockedUserIds = useAtomValue(blockedUserIdsAtom)
  const removeBlockedUser = useSetAtom(removeBlockedUserAtom)

  return (
    <FlatList
      data={blockedUserIds}
      renderItem={({ item }) => (
        <View style={styles.item}>
          <Text>ユーザーID: {item}</Text>
          <Button
            title="ブロック解除"
            onPress={() => removeBlockedUser(item)}
          />
        </View>
      )}
      ListEmptyComponent={
        <Text>ブロックしているユーザーはいません</Text>
      }
    />
  )
}
```

## まとめ

- ActionSheetでブロック・通報メニューを表示
- ブロックリストはローカル（AsyncStorage）とサーバー両方で管理
- メッセージ表示時にフィルタリング
- App Store審査のため、設定画面にブロック解除機能を用意

## 参考資料

- [Apple - App Review Guidelines 1.2](https://developer.apple.com/app-store/review/guidelines/#safety)
- [Google Play - User Generated Content Policy](https://support.google.com/googleplay/android-developer/answer/9876937)
