---
title: "【React Native】本番環境で発生した無限再レンダリングを検出・修正する"
emoji: "🔄"
type: "tech"
topics: ["ReactNative", "React", "useEffect", "パフォーマンス", "デバッグ"]
published: false
---

# この記事は？

React Nativeアプリで本番環境のみ無限再レンダリングが発生し、アプリがフリーズする問題に遭遇しました。この記事では、無限ループの検出方法と、よくある原因・修正パターンを解説します。

## 問題の状況

開発環境では問題なく動作するのに、本番ビルドでのみアプリがフリーズ。調査の結果、特定のコンポーネントが無限に再レンダリングされていました。

```
開発環境: 正常動作
本番環境: フリーズ → 無限再レンダリングが原因
```

## 無限再レンダリングの検出方法

### 1. useEffectのログで確認

```typescript
useEffect(() => {
  console.log('useEffect triggered', new Date().toISOString())
  // 処理...
}, [dependency])
```

ログが高速で連続出力されていたら無限ループの可能性大。

### 2. React DevToolsのProfiler

開発環境でProfilerを使い、レンダリング回数を確認します。

### 3. why-did-you-renderの導入

```typescript
// wdyr.ts
import React from 'react'

if (__DEV__) {
  const whyDidYouRender = require('@welldone-software/why-did-you-render')
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  })
}

// コンポーネントに追加
MyComponent.whyDidYouRender = true
```

## よくある原因と修正パターン

### パターン1: オブジェクト/配列の依存配列

```typescript
// ❌ 毎回新しいオブジェクトが作られて無限ループ
const config = { theme: 'dark' }

useEffect(() => {
  applyConfig(config)
}, [config])  // configは毎レンダリングで新規オブジェクト

// ✅ useMemoで参照を安定化
const config = useMemo(() => ({ theme: 'dark' }), [])

useEffect(() => {
  applyConfig(config)
}, [config])
```

### パターン2: useEffect内でのstate更新

```typescript
// ❌ state更新 → 再レンダリング → useEffect → state更新...
const [data, setData] = useState(null)

useEffect(() => {
  fetchData().then(result => {
    setData(result)  // これが毎回実行される
  })
}, [data])  // dataが依存配列に入っている

// ✅ 依存配列から除外、または条件分岐
useEffect(() => {
  if (data === null) {
    fetchData().then(result => {
      setData(result)
    })
  }
}, [data])
```

### パターン3: コールバック関数の依存

```typescript
// ❌ onChangeが毎回新しい関数
const Parent = () => {
  const handleChange = (value: string) => {
    console.log(value)
  }

  return <Child onChange={handleChange} />
}

// ✅ useCallbackで安定化
const Parent = () => {
  const handleChange = useCallback((value: string) => {
    console.log(value)
  }, [])

  return <Child onChange={handleChange} />
}
```

### パターン4: 本番環境のみ発生するケース

開発環境ではReact.StrictModeの二重レンダリングがあるため、一部の問題が隠れることがあります。

```typescript
// 本番でのみ問題が顕在化するパターン
const [isReady, setIsReady] = useState(false)

useEffect(() => {
  // 開発環境では2回実行されるため、タイミングの問題が隠れる
  someAsyncOperation().then(() => {
    setIsReady(true)
  })
}, [])
```

## 今回の修正内容

問題のコードは、APIレスポンスをstateに設定する際に、オブジェクトの参照が毎回変わっていたことが原因でした。

```typescript
// ❌ 修正前: response.dataは毎回新しいオブジェクト
useEffect(() => {
  if (response.data) {
    setRoomData(response.data)
  }
}, [response.data])

// ✅ 修正後: IDで比較して不要な更新を防ぐ
useEffect(() => {
  if (response.data && response.data.id !== roomData?.id) {
    setRoomData(response.data)
  }
}, [response.data?.id])
```

## デバッグTips

### レンダリング回数をカウント

```typescript
const renderCount = useRef(0)

useEffect(() => {
  renderCount.current += 1
  console.log(`Render count: ${renderCount.current}`)
})
```

### 依存配列の変化を追跡

```typescript
const usePrevious = <T,>(value: T): T | undefined => {
  const ref = useRef<T>()
  useEffect(() => {
    ref.current = value
  })
  return ref.current
}

const prevDeps = usePrevious(dependency)
useEffect(() => {
  console.log('Dependency changed:', { prev: prevDeps, current: dependency })
}, [dependency])
```

## まとめ

- 無限ループは依存配列の誤設定が主な原因
- オブジェクト/配列は`useMemo`、関数は`useCallback`で安定化
- useEffect内でのstate更新は条件分岐を入れる
- 本番環境でのみ発生するケースもあるため、本番ビルドでのテストも重要

## 参考資料

- [React公式 - useEffectのルール](https://react.dev/reference/react/useEffect)
- [why-did-you-render](https://github.com/welldone-software/why-did-you-render)
