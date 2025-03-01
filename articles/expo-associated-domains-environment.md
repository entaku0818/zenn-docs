---
title: "【React Native】環境別のAssociated Domains設定（Universal Links対応）"
emoji: "🔗"
type: "tech"
topics: ["ReactNative", "Expo", "UniversalLinks", "iOS"]
published: false
---

# この記事は？

Expo/React Nativeアプリで開発・ステージング・本番環境ごとに異なるドメインでUniversal Linksを動作させたいことはありませんか？この記事では、`app.config.ts`を使って環境別にAssociated Domainsを設定する方法を解説します。

## 問題の背景

Universal Links（iOSのディープリンク）を使用するには、`Associated Domains`の設定が必要です。しかし、複数の環境で異なるドメインを使用する場合、設定が煩雑になります：

- 開発環境: `dev.example.com`
- ステージング環境: `staging.example.com`
- 本番環境: `example.com`

## 解決方法

`app.config.ts`で環境変数に基づいてAssociated Domainsを動的に設定します。

### 1. 環境変数の定義

```bash
# .env.development
APP_ENV=development
ASSOCIATED_DOMAIN=dev.example.com

# .env.staging
APP_ENV=staging
ASSOCIATED_DOMAIN=staging.example.com

# .env.production
APP_ENV=production
ASSOCIATED_DOMAIN=example.com
```

### 2. app.config.tsの設定

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config'

export default ({ config }: ConfigContext): ExpoConfig => {
  const appEnv = process.env.APP_ENV || 'development'
  const associatedDomain = process.env.ASSOCIATED_DOMAIN || 'dev.example.com'

  // 環境ごとのBundle ID
  const bundleIdentifier = {
    development: 'com.example.app.dev',
    staging: 'com.example.app.staging',
    production: 'com.example.app',
  }[appEnv]

  return {
    ...config,
    name: appEnv === 'production' ? 'MyApp' : `MyApp (${appEnv})`,
    slug: 'my-app',
    ios: {
      bundleIdentifier,
      associatedDomains: [
        `applinks:${associatedDomain}`,
        `webcredentials:${associatedDomain}`,
      ],
    },
    // ...
  }
}
```

### 3. 複数ドメインの設定

複数のドメインをサポートする場合：

```typescript
// app.config.ts
const getAssociatedDomains = (appEnv: string): string[] => {
  const domains: Record<string, string[]> = {
    development: [
      'applinks:dev.example.com',
      'applinks:localhost:3000',
    ],
    staging: [
      'applinks:staging.example.com',
    ],
    production: [
      'applinks:example.com',
      'applinks:www.example.com',
    ],
  }

  return domains[appEnv] || domains.development
}

export default ({ config }: ConfigContext): ExpoConfig => {
  const appEnv = process.env.APP_ENV || 'development'

  return {
    ...config,
    ios: {
      bundleIdentifier: getBundleIdentifier(appEnv),
      associatedDomains: getAssociatedDomains(appEnv),
    },
  }
}
```

## EAS Buildでの環境切り替え

### eas.jsonの設定

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_ENV": "development",
        "ASSOCIATED_DOMAIN": "dev.example.com"
      }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging",
        "ASSOCIATED_DOMAIN": "staging.example.com"
      }
    },
    "production": {
      "env": {
        "APP_ENV": "production",
        "ASSOCIATED_DOMAIN": "example.com"
      }
    }
  }
}
```

## Apple App Site Associationファイル

各ドメインのサーバーに`apple-app-site-association`ファイルを配置します：

```json
// https://example.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.example.app",
        "paths": ["/talkrooms/*", "/invite/*"]
      }
    ]
  }
}
```

環境ごとに異なるBundle IDを使用する場合：

```json
// https://staging.example.com/.well-known/apple-app-site-association
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAM_ID.com.example.app.staging",
        "paths": ["/talkrooms/*", "/invite/*"]
      }
    ]
  }
}
```

## 確認方法

### 1. 設定の確認

```bash
# 設定内容を確認
npx expo config --type introspect

# 特定の環境で確認
APP_ENV=staging npx expo config --type introspect
```

### 2. Universal Linksのテスト

```bash
# シミュレータでテスト
xcrun simctl openurl booted "https://staging.example.com/talkrooms/123"

# 実機でテスト
# Notesアプリにリンクを貼り付けてタップ
```

## 注意点

### 1. ネイティブビルドが必要

Associated Domainsの変更は、OTAアップデート（EAS Update）では反映されません。ネイティブビルドが必要です。

### 2. Apple Developer Consoleの設定

Apple Developer ConsoleでApp IDのAssociated Domains機能を有効にする必要があります。

### 3. HTTPSが必須

Associated Domainsで指定するドメインはHTTPSが必須です。開発環境でlocalhostを使う場合は注意が必要です。

## まとめ

- `app.config.ts`で環境変数に基づいてAssociated Domainsを動的に設定
- EAS Buildの`env`でビルドプロファイルごとに環境変数を指定
- 各ドメインのサーバーに`apple-app-site-association`ファイルを配置
- 変更にはネイティブビルドが必要

## 参考資料

- [Expo - Associated Domains](https://docs.expo.dev/guides/linking/#universal-links-on-ios)
- [Apple - Supporting Associated Domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains)
