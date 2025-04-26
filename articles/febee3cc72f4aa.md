---
title: "Xcodeプロジェクトで環境変数を設定し、読み込む方法"
emoji: "👁️"
type: "tech"
topics:
  - "xcodecloud"
published: true
published_at: "2024-05-27 09:30"
publication_name: "voicy"
---

## Xcodeプロジェクトで環境変数を設定し、読み込む方法

この記事では、Xcodeプロジェクトで環境変数を設定し、それらを`AppDelegate`で読み込む方法について説明します。`xcconfig`ファイルやXcode Cloudでの環境変数設定を利用して、プロジェクトのビルドや実行時に適切な環境変数を使用する方法を紹介します。

### 環境変数の設定

まず、環境変数を`xcconfig`ファイルおよびXcode Cloudで設定する方法を見ていきます。

#### 1. xcconfig ファイルの設定

`xcconfig`ファイルはビルド設定を定義するためのファイルです。以下に例を示します。

**Debug.xcconfig**
```plaintext
ROLLBAR_KEY = debug-rollbar-key
ADMOB_KEY = debug-admob-key
REVENUECAT_KEY = debug-revenuecat-key
```

**Release.xcconfig**
```plaintext
ROLLBAR_KEY = release-rollbar-key
ADMOB_KEY = release-admob-key
REVENUECAT_KEY = release-revenuecat-key
```

#### 2. Xcode Cloudの設定

Xcode Cloudの設定ページで環境変数を追加します。以下のように設定します：
- 名前： `ROLLBAR_KEY`
  - 値： `cloud-rollbar-key`
- 名前： `ADMOB_KEY`
  - 値： `cloud-admob-key`
- 名前： `REVENUECAT_KEY`
  - 値： `cloud-revenuecat-key`

### Info.plist に環境変数を追加

`Info.plist`にカスタムキーを追加し、`xcconfig`ファイルの設定を参照できるようにします。

**Info.plist**
```xml
<key>ROLLBAR_KEY</key>
<string>$(ROLLBAR_KEY)</string>
<key>ADMOB_KEY</key>
<string>$(ADMOB_KEY)</string>
<key>REVENUECAT_KEY</key>
<string>$(REVENUECAT_KEY)</string>
```

### AppDelegateのExtensionで環境変数を読み込む

次に、AppDelegateに環境変数を読み込む処理を追加し、構造体に格納します。

**EnvironmentConfig.swift**
```swift
struct EnvironmentConfig {
    let rollbarKey: String
    let admobKey: String
    let revenueCatKey: String
}
```

**AppDelegate.swift**
```swift
import UIKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {

    var window: UIWindow?
    var environmentConfig: EnvironmentConfig?

    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // 他の初期化処理

        // 環境変数の読み込み
        environmentConfig = loadEnvironmentVariables()

        if let config = environmentConfig {
            print("Rollbar Key: \(config.rollbarKey)")
            print("AdMob Key: \(config.admobKey)")
            print("RevenueCat Key: \(config.revenueCatKey)")
        }

        return true
    }
}

extension AppDelegate {
    func loadEnvironmentVariables() -> EnvironmentConfig {
        guard
            let rollbarKey = Bundle.main.object(forInfoDictionaryKey: "ROLLBAR_KEY") as? String ?? ProcessInfo.processInfo.environment["ROLLBAR_KEY"],
            let admobKey = Bundle.main.object(forInfoDictionaryKey: "ADMOB_KEY") as? String ?? ProcessInfo.processInfo.environment["ADMOB_KEY"],
            let revenueCatKey = Bundle.main.object(forInfoDictionaryKey: "REVENUECAT_KEY") as? String ?? ProcessInfo.processInfo.environment["REVENUECAT_KEY"]
        else {
            fatalError("One or more environment variables are missing")
        }

        return EnvironmentConfig(rollbarKey: rollbarKey, admobKey: admobKey, revenueCatKey: revenueCatKey)
    }
}
```

### 説明

1. **EnvironmentConfig構造体**:
   - `rollbarKey`、`admobKey`、および `revenueCatKey` の3つの変数を持つ構造体を定義します。

2. **AppDelegate**:
   - `environmentConfig`プロパティを定義し、環境変数を格納するためのインスタンスを保持します。
   - `application(_:didFinishLaunchingWithOptions:)` メソッド内で `loadEnvironmentVariables()` メソッドを呼び出し、読み込んだ環境変数を `environmentConfig` に格納します。

3. **loadEnvironmentVariablesメソッド**:
   - `Info.plist` または `ProcessInfo.processInfo.environment` から環境変数を取得し、1つでも不足している場合は `fatalError` でアプリをクラッシュさせます。
   - 環境変数がすべて揃っている場合は、`EnvironmentConfig` のインスタンスを作成して返します。

### まとめ

この方法により、Xcodeプロジェクトで環境変数を設定し、`AppDelegate`で読み込んで使用することができます。`xcconfig`ファイルやXcode Cloudの環境変数設定を活用して、プロジェクトのビルドや実行時に適切な設定を適用できるようになります。環境変数が不足している場合はアプリをクラッシュさせることで、デバッグを容易にし、設定ミスを防ぐことができます。