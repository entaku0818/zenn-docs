---
title: "【React Native】App Tracking Transparency (ATT)を実装する"
emoji: "🔒"
type: "tech"
topics: ["ReactNative", "Expo", "iOS", "ATT", "プライバシー"]
published: false
---

# この記事は？

iOS 14.5以降、アプリがユーザーのトラッキングを行う前にATT（App Tracking Transparency）の許可を求めることが必須になりました。この記事では、React Native（Expo）でATTを実装する方法を解説します。

## 前提条件

- Expo SDK 48以上
- expo-tracking-transparency

## インストール

```bash
npx expo install expo-tracking-transparency
```

## 実装

### 1. app.jsonの設定

```json
{
  "expo": {
    "plugins": [
      [
        "expo-tracking-transparency",
        {
          "userTrackingPermission": "このアプリでは、パーソナライズされた広告を表示するためにトラッキングを使用します。"
        }
      ]
    ]
  }
}
```

### 2. カスタムフックの作成

```typescript
// hooks/useAppTrackingTransparency.ts
import { useCallback, useEffect, useState } from 'react'
import {
  getTrackingPermissionsAsync,
  requestTrackingPermissionsAsync,
  PermissionStatus,
} from 'expo-tracking-transparency'
import { Platform } from 'react-native'

type TrackingStatus = 'undetermined' | 'granted' | 'denied' | 'restricted'

export const useAppTrackingTransparency = () => {
  const [status, setStatus] = useState<TrackingStatus>('undetermined')
  const [isLoading, setIsLoading] = useState(true)

  const checkPermission = useCallback(async () => {
    // Androidでは常に許可扱い
    if (Platform.OS !== 'ios') {
      setStatus('granted')
      setIsLoading(false)
      return
    }

    try {
      const { status: currentStatus } = await getTrackingPermissionsAsync()
      setStatus(mapPermissionStatus(currentStatus))
    } catch (error) {
      console.error('Failed to check tracking permission:', error)
    } finally {
      setIsLoading(false)
    }
  }, [])

  const requestPermission = useCallback(async (): Promise<boolean> => {
    if (Platform.OS !== 'ios') {
      return true
    }

    try {
      const { status: newStatus } = await requestTrackingPermissionsAsync()
      const mappedStatus = mapPermissionStatus(newStatus)
      setStatus(mappedStatus)
      return mappedStatus === 'granted'
    } catch (error) {
      console.error('Failed to request tracking permission:', error)
      return false
    }
  }, [])

  useEffect(() => {
    checkPermission()
  }, [checkPermission])

  return {
    status,
    isLoading,
    isGranted: status === 'granted',
    requestPermission,
  }
}

const mapPermissionStatus = (status: PermissionStatus): TrackingStatus => {
  switch (status) {
    case PermissionStatus.GRANTED:
      return 'granted'
    case PermissionStatus.DENIED:
      return 'denied'
    case PermissionStatus.UNDETERMINED:
      return 'undetermined'
    default:
      return 'restricted'
  }
}
```

### 3. アプリ起動時に許可を求める

```typescript
// App.tsx または起動時に実行されるコンポーネント
import { useEffect } from 'react'
import { useAppTrackingTransparency } from './hooks/useAppTrackingTransparency'

export const AppInitializer = ({ children }: { children: React.ReactNode }) => {
  const { status, isLoading, requestPermission } = useAppTrackingTransparency()

  useEffect(() => {
    const initTracking = async () => {
      if (status === 'undetermined') {
        // 少し遅延させてアプリの起動を優先
        await new Promise(resolve => setTimeout(resolve, 1000))
        await requestPermission()
      }
    }

    if (!isLoading) {
      initTracking()
    }
  }, [status, isLoading, requestPermission])

  return <>{children}</>
}
```

### 4. 広告SDKとの連携

ATTの結果に基づいて広告SDKを初期化します。

```typescript
// services/analytics.ts
import analytics from '@react-native-firebase/analytics'

export const initializeAnalytics = async (isTrackingGranted: boolean) => {
  // ATTが許可されていない場合、広告IDの収集を無効化
  await analytics().setAnalyticsCollectionEnabled(isTrackingGranted)

  if (isTrackingGranted) {
    // パーソナライズ広告を有効化
    console.log('Personalized ads enabled')
  } else {
    // コンテキスト広告のみ
    console.log('Contextual ads only')
  }
}
```

## 注意点

### ダイアログ表示のタイミング

ATTダイアログは適切なタイミングで表示することが重要です。

```typescript
// ❌ アプリ起動直後に表示
useEffect(() => {
  requestPermission() // ユーザーに唐突感を与える
}, [])

// ✅ 少し遅延させて表示
useEffect(() => {
  const timer = setTimeout(() => {
    if (status === 'undetermined') {
      requestPermission()
    }
  }, 1500) // アプリのロードが完了してから

  return () => clearTimeout(timer)
}, [status])
```

### 事前説明の表示

ATTダイアログの前に、なぜトラッキングが必要かを説明するカスタムモーダルを表示することを推奨します。

```typescript
const [showPreATTModal, setShowPreATTModal] = useState(false)

const handlePreATTConfirm = async () => {
  setShowPreATTModal(false)
  await requestPermission()
}

return (
  <>
    <Modal visible={showPreATTModal}>
      <Text>
        より良い体験のために、パーソナライズされたコンテンツを
        お届けしたいと考えています。
      </Text>
      <Button title="続ける" onPress={handlePreATTConfirm} />
    </Modal>
    {children}
  </>
)
```

### App Store審査対策

- `NSUserTrackingUsageDescription`は必ず設定する
- 説明文は具体的に記載（「広告のため」ではなく「パーソナライズされた広告を表示するため」）
- ATTを求める前にアプリの価値を提示する

## トラッキングステータスの確認

```typescript
import { getTrackingPermissionsAsync } from 'expo-tracking-transparency'

const checkStatus = async () => {
  const { status, canAskAgain } = await getTrackingPermissionsAsync()

  console.log('Current status:', status)
  console.log('Can ask again:', canAskAgain) // 一度拒否されると false
}
```

## まとめ

- `expo-tracking-transparency`で簡単に実装可能
- ダイアログ表示のタイミングはUXに大きく影響
- 事前説明モーダルで許可率を向上
- ATT未許可でもアプリは正常に動作させる設計が重要

## 参考資料

- [Apple - App Tracking Transparency](https://developer.apple.com/documentation/apptrackingtransparency)
- [Expo - expo-tracking-transparency](https://docs.expo.dev/versions/latest/sdk/tracking-transparency/)
