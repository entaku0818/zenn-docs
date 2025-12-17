---
title: "【Expo】EASビルドサイズを最適化する（GIF圧縮編）"
emoji: "📦"
type: "tech"
topics: ["Expo", "ReactNative", "EAS", "パフォーマンス", "最適化"]
published: false
---

# この記事は？

Expoアプリのビルドサイズが大きくなると、ダウンロード時間やストレージ使用量に影響します。この記事では、特にGIF画像の圧縮を中心に、EASビルドサイズを最適化する方法を解説します。

## 問題の状況

アバターやスタンプにGIFアニメーションを多用していたところ、ビルドサイズが100MBを超えてしまいました。

```
最適化前: 105MB
最適化後: 42MB（60%削減）
```

## GIF圧縮の手順

### 1. gifskiによる最適化

高品質を維持しながらGIFを圧縮できるツールです。

```bash
# インストール（macOS）
brew install gifski

# 圧縮（品質80%、最大幅200px）
gifski --quality 80 --width 200 -o output.gif input.gif
```

### 2. gifsicleによる最適化

フレーム数削減やカラー最適化に有効です。

```bash
# インストール
brew install gifsicle

# 最適化（カラー数削減 + 最適化レベル3）
gifsicle -O3 --colors 128 input.gif -o output.gif

# リサイズも同時に行う
gifsicle --resize-width 150 -O3 input.gif -o output.gif
```

### 3. 一括処理スクリプト

```bash
#!/bin/bash
# optimize-gifs.sh

INPUT_DIR="./assets/avatars/original"
OUTPUT_DIR="./assets/avatars"

mkdir -p "$OUTPUT_DIR"

for file in "$INPUT_DIR"/*.gif; do
  filename=$(basename "$file")
  echo "Processing: $filename"

  # gifsicleで最適化
  gifsicle --resize-width 150 -O3 --colors 128 "$file" -o "$OUTPUT_DIR/$filename"

  # サイズ比較
  original_size=$(stat -f%z "$file")
  new_size=$(stat -f%z "$OUTPUT_DIR/$filename")
  reduction=$((100 - new_size * 100 / original_size))
  echo "  $original_size bytes → $new_size bytes ($reduction% reduction)"
done
```

## Expo設定での最適化

### metro.config.jsでのアセット圧縮

```javascript
// metro.config.js
const { getDefaultConfig } = require('expo/metro-config')

const config = getDefaultConfig(__dirname)

// アセットの圧縮を有効化
config.transformer.minifierConfig = {
  compress: {
    drop_console: true,
  },
}

module.exports = config
```

### app.jsonでのアセット除外

開発用アセットを本番ビルドから除外：

```json
{
  "expo": {
    "assetBundlePatterns": [
      "assets/images/**",
      "assets/avatars/**"
    ]
  }
}
```

## 画像フォーマットの選択

| フォーマット | 用途 | 特徴 |
|------------|------|------|
| GIF | 短いアニメーション | 互換性高い、色数制限 |
| WebP | 静止画・アニメーション | 高圧縮、要対応確認 |
| Lottie | ベクターアニメーション | 最小サイズ、スケーラブル |
| PNG | 透過静止画 | 品質維持、サイズ大 |

### Lottieへの移行

GIFよりLottieの方が大幅に軽量です。

```bash
# GIF: 500KB → Lottie: 20KB（96%削減）
```

```typescript
import LottieView from 'lottie-react-native'

const Avatar = () => (
  <LottieView
    source={require('./assets/avatar.json')}
    autoPlay
    loop
    style={{ width: 100, height: 100 }}
  />
)
```

## ビルドサイズの確認方法

### EASビルドログで確認

```bash
eas build --platform ios --profile production
# ビルド完了後、Expo管理画面でサイズを確認
```

### ローカルでの確認

```bash
# iOSアプリのサイズ確認
npx expo export --platform ios
du -sh dist/

# Androidアプリのサイズ確認
npx expo export --platform android
du -sh dist/
```

## その他の最適化テクニック

### 1. 不要な依存関係の削除

```bash
# 使用していないパッケージを検出
npx depcheck

# 削除
npm uninstall unused-package
```

### 2. Tree Shakingの確認

```javascript
// ❌ 全体インポート
import _ from 'lodash'

// ✅ 必要な関数のみインポート
import debounce from 'lodash/debounce'
```

### 3. 画像の遅延読み込み

```typescript
import { Image } from 'expo-image'

// expo-imageは自動的にキャッシュと最適化を行う
<Image
  source={{ uri: 'https://example.com/avatar.gif' }}
  contentFit="contain"
  transition={200}
/>
```

## 最適化チェックリスト

- [ ] GIF/PNGを可能な限り圧縮
- [ ] LottieやWebPへの移行を検討
- [ ] 不要なアセットを削除
- [ ] 開発用ファイルをビルドから除外
- [ ] 依存関係を定期的に見直し
- [ ] 画像は必要なサイズのみバンドル

## まとめ

- GIFはgifsicle/gifskiで圧縮
- 可能ならLottieに移行（大幅なサイズ削減）
- 定期的にビルドサイズを監視
- 不要なアセット・依存関係を削除

## 参考資料

- [Expo - Asset optimization](https://docs.expo.dev/guides/assets/)
- [gifsicle](https://www.lcdf.org/gifsicle/)
- [Lottie for React Native](https://github.com/lottie-react-native/lottie-react-native)
