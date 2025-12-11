---
title: "【Expo】環境別にBundle IDとFirebaseプロジェクトを分離する"
emoji: "🔧"
type: "tech"
topics: ["Expo", "ReactNative", "Firebase", "環境構築"]
published: false
---

# この記事は？

Expoプロジェクトで開発/ステージング/本番環境ごとにBundle ID（iOS）/ Package Name（Android）とFirebaseプロジェクトを分離する方法を解説します。これにより、環境を混同することなく安全に開発・テストができます。

## 問題の状況

単一のBundle IDとFirebaseプロジェクトを使っていると：

- 開発中のプッシュ通知が本番ユーザーに届く
- テストデータが本番のAnalyticsに混入
- 同じ端末に複数環境のアプリをインストールできない

## ディレクトリ構成

```
project/
├── app.config.ts
├── eas.json
├── firebase/
│   ├── development/
│   │   ├── google-services.json
│   │   └── GoogleService-Info.plist
│   ├── staging/
│   │   ├── google-services.json
│   │   └── GoogleService-Info.plist
│   └── production/
│       ├── google-services.json
│       └── GoogleService-Info.plist
```

## 実装手順

### 1. app.config.tsの設定

```typescript
// app.config.ts
import { ExpoConfig, ConfigContext } from 'expo/config'

const ENV = process.env.APP_ENV || 'development'

const envConfig = {
  development: {
    name: 'MyApp (Dev)',
    bundleIdentifier: 'com.mycompany.myapp.dev',
    googleServicesFile: './firebase/development/google-services.json',
    googleServicesInfoPlist: './firebase/development/GoogleService-Info.plist',
  },
  staging: {
    name: 'MyApp (Staging)',
    bundleIdentifier: 'com.mycompany.myapp.staging',
    googleServicesFile: './firebase/staging/google-services.json',
    googleServicesInfoPlist: './firebase/staging/GoogleService-Info.plist',
  },
  production: {
    name: 'MyApp',
    bundleIdentifier: 'com.mycompany.myapp',
    googleServicesFile: './firebase/production/google-services.json',
    googleServicesInfoPlist: './firebase/production/GoogleService-Info.plist',
  },
}

const config = envConfig[ENV as keyof typeof envConfig]

export default ({ config: baseConfig }: ConfigContext): ExpoConfig => ({
  ...baseConfig,
  name: config.name,
  slug: 'myapp',
  ios: {
    ...baseConfig.ios,
    bundleIdentifier: config.bundleIdentifier,
    googleServicesFile: config.googleServicesInfoPlist,
  },
  android: {
    ...baseConfig.android,
    package: config.bundleIdentifier,
    googleServicesFile: config.googleServicesFile,
  },
})
```

### 2. eas.jsonの設定

```json
{
  "cli": {
    "version": ">= 5.0.0"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": {
        "APP_ENV": "development"
      },
      "ios": {
        "simulator": true
      }
    },
    "staging": {
      "distribution": "internal",
      "env": {
        "APP_ENV": "staging"
      },
      "channel": "staging"
    },
    "production": {
      "distribution": "store",
      "env": {
        "APP_ENV": "production"
      },
      "channel": "production"
    }
  }
}
```

### 3. 環境別Firebaseプロジェクトの作成

各環境用にFirebaseプロジェクトを作成し、設定ファイルをダウンロード：

1. Firebase Console で新規プロジェクト作成
2. iOS/Androidアプリを追加（環境別のBundle ID/Package Nameを使用）
3. `google-services.json`と`GoogleService-Info.plist`をダウンロード
4. 対応するディレクトリに配置

### 4. .gitignoreの設定

本番の認証情報はリポジトリに含めないようにします。

```gitignore
# Firebase設定（本番のみ除外）
firebase/production/google-services.json
firebase/production/GoogleService-Info.plist
```

開発/ステージング環境の設定は共有してOK。

### 5. ローカル開発時の環境切り替え

```bash
# 開発環境
APP_ENV=development npx expo start

# ステージング環境
APP_ENV=staging npx expo start

# 本番環境（通常はCIで実行）
APP_ENV=production npx expo start
```

### 6. EASビルドの実行

```bash
# 開発ビルド
eas build --profile development --platform ios

# ステージングビルド
eas build --profile staging --platform ios

# 本番ビルド
eas build --profile production --platform ios
```

## アプリ内で環境を判定する

```typescript
// utils/environment.ts
import Constants from 'expo-constants'

export type Environment = 'development' | 'staging' | 'production'

export const getEnvironment = (): Environment => {
  const bundleId = Constants.expoConfig?.ios?.bundleIdentifier || ''

  if (bundleId.endsWith('.dev')) {
    return 'development'
  }
  if (bundleId.endsWith('.staging')) {
    return 'staging'
  }
  return 'production'
}

export const isDevelopment = () => getEnvironment() === 'development'
export const isStaging = () => getEnvironment() === 'staging'
export const isProduction = () => getEnvironment() === 'production'
```

## APIエンドポイントの切り替え

```typescript
// config/api.ts
import { getEnvironment } from '../utils/environment'

const API_URLS = {
  development: 'http://localhost:8080',
  staging: 'https://staging-api.example.com',
  production: 'https://api.example.com',
}

export const API_BASE_URL = API_URLS[getEnvironment()]
```

## アプリアイコンの分離（オプション）

環境ごとに異なるアイコンを使うと、インストール済みアプリを一目で区別できます。

```typescript
// app.config.ts
const envConfig = {
  development: {
    // ...
    icon: './assets/icon-dev.png',
  },
  staging: {
    // ...
    icon: './assets/icon-staging.png',
  },
  production: {
    // ...
    icon: './assets/icon.png',
  },
}
```

## まとめ

- 環境別にBundle ID/Package Nameを分離
- Firebaseプロジェクトも環境ごとに作成
- `app.config.ts`と`eas.json`で環境切り替えを管理
- アプリ内で`Constants.expoConfig`から環境を判定

## 参考資料

- [Expo - Environment variables](https://docs.expo.dev/guides/environment-variables/)
- [EAS Build - Build configuration](https://docs.expo.dev/build/eas-json/)
