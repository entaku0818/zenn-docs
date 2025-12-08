---
title: "iOS 26のAlarmKit APIでアプリからアラームを鳴らす"
emoji: "⏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Swift", "iOS", "AlarmKit", "WWDC25"]
published: false
---

## はじめに

WWDC25で発表されたiOS 26およびiPadOS 26の新機能として、**AlarmKit API**が登場しました。これにより、これまで標準の時計アプリでしか実現できなかったアラーム機能を、サードパーティアプリでも実装できるようになりました。

レシピアプリのカウントダウンタイマーや、旅行計画アプリの目覚ましアラームなどで、ロック画面、Dynamic Island、その他の場所にタイマーとアラームを追加できます。

https://developer.apple.com/jp/videos/play/wwdc2025/230/


## AlarmKit APIとは

AlarmKit APIを使用すると、アプリから以下のことが可能になります：

- アラームの作成および管理
- ライブアクティビティのカスタマイズ
- カスタムのアラートアクションの実装

標準の時計アプリと同様の、フルスクリーンのアラーム画面（スヌーズ・停止ボタン付き）を表示できます。

| 標準時計アプリ | AlarmKitによるアラーム画面 |
|:---:|:---:|
| ![標準時計アプリ](/images/alarmkit-api-guide/clock-app.png) | ![AlarmKitアラーム画面](/images/alarmkit-api-guide/alarm-screen.png) |

## 実装方法

### 1. アラートの設定

まず、`AlarmPresentation.Alert`を使ってアラート時に表示するコンテンツを設定します。

```swift
let alert = AlarmPresentation.Alert(
    title: "音テスト！",
    secondaryButton: secondaryButton,
    secondaryButtonBehavior: .countdown
)
```

- `title`: アラート画面に表示されるタイトル
- `secondaryButton`: セカンダリボタン（スヌーズなど）の設定
- `secondaryButtonBehavior`: セカンダリボタンの動作（`.countdown`でカウントダウン表示）

### 2. プレゼンテーションの作成

作成したアラートをアラーム画面に設定します。

```swift
let presentation = AlarmPresentation(alert: alert)
```

### 3. アトリビュート（属性）の設定

`AlarmAttributes`でアラームの属性を定義します。

```swift
let attributes = AlarmAttributes<FlightAlarmMetadata>(
    presentation: presentation,
    tintColor: .red
)
```

- `presentation`: 先ほど作成したプレゼンテーション
- `tintColor`: アラーム画面のアクセントカラー

### 4. タイマー設定の作成

`AlarmManager.AlarmConfiguration.timer`でタイマーの設定を作成します。

```swift
// 10秒のタイマーを設定
// sound パラメータを省略するとシステムデフォルトのサウンドを使用
let configuration = AlarmManager.AlarmConfiguration.timer(
    duration: 10,
    attributes: attributes
)
```

### 5. アラームのスケジュール

最後に、`AlarmManager`を使ってアラームをスケジュールします。

```swift
let alarm = try await alarmManager.schedule(
    id: alarmID,
    configuration: configuration
)
```

### 完全なコード例

```swift
import AlarmKit

func scheduleAlarm() async throws {
    let alarmManager = AlarmManager()
    let alarmID = "my-timer-alarm"

    // セカンダリボタンの設定
    let secondaryButton = AlarmPresentation.Alert.SecondaryButton(
        title: "スヌーズ",
        action: .snooze(duration: 300) // 5分後にスヌーズ
    )

    // アラートの設定
    let alert = AlarmPresentation.Alert(
        title: "タイマー終了",
        secondaryButton: secondaryButton,
        secondaryButtonBehavior: .countdown
    )

    // プレゼンテーションの作成
    let presentation = AlarmPresentation(alert: alert)

    // アトリビュートの設定
    let attributes = AlarmAttributes<MyAlarmMetadata>(
        presentation: presentation,
        tintColor: .orange
    )

    // タイマー設定の作成（3分 = 180秒）
    let configuration = AlarmManager.AlarmConfiguration.timer(
        duration: 180,
        attributes: attributes
    )

    // アラームのスケジュール
    let alarm = try await alarmManager.schedule(
        id: alarmID,
        configuration: configuration
    )
}
```

## Info.plistの設定

AlarmKitを使用するには、`Info.plist`に以下のキーを追加する必要があります。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSAlarmKitUsageDescription</key>
    <string>搭乗時刻時間前にアラームを設定するために使用します</string>
    <key>NSSupportsLiveActivities</key>
    <true/>
</dict>
</plist>
```

:::message alert
**NSAlarmKitUsageDescriptionの設定を忘れないでください！**
このキーがないとAlarmKitの機能が動作しません。
:::

## 困ったこと・注意点

実際にAlarmKitを使用してみて、いくつかの注意点がありました。

### 1. アプリ起動中のアラーム動作

アプリがフォアグラウンドで起動中だと、アラームがうまく発火しないことがあるようです。バックグラウンド状態やアプリ終了時には正常に動作します。
フォアグラウンドの時は別の方法とも合わせた設定をする方が良さそうです。

### 2. アラーム画面での停止操作

アラームが鳴った際に表示される画面では、ユーザーが「スライドで停止」または「スヌーズ」でしかアラームを止められません。この画面で停止させないようにする（強制的に何かのアクションを要求する）といったカスタマイズは現時点ではできないようです。

## なぜAlarmKitが必要なのか - UserNotificationsの限界

従来の`UserNotifications`フレームワークには、アラームとして使用するには多くの制限がありました：


### UserNotificationsの具体的な問題点

1. **通知音が最大30秒で停止する**
   - ユーザーが気づかないまま通知が消えてしまう
   - 重要なリマインダーを見逃すリスク

2. **サイレントモード・集中モードで無効化される**
   - iPhoneをサイレントにしていると通知音が鳴らない
   - 集中モード中は通知自体が届かない

3. **通知センターに埋もれる**
   - 他のアプリの通知に紛れて見逃しやすい
   - スワイプで簡単に消せてしまう

4. **スヌーズ機能がない**
   - 「あと5分」ができない
   - 自前で再通知の仕組みを実装する必要がある

5. **ユーザーが通知をOFFにしやすい**
   - 設定から簡単に通知を無効化できる
   - アプリの重要な機能が使えなくなる

AlarmKitを使えば、これらの問題をすべて解決し、標準の時計アプリと同等の**確実に届くアラーム体験**を提供できます。

## TVerに入れるなら

AlarmKitの活用例として、私が所属しているTVerのような動画配信アプリでの導入例を考えてみましょう。

TVerには「視聴予約」機能があり、番組の配信開始時刻に通知を受け取ることができます。

![TVerの視聴予約機能](/images/alarmkit-api-guide/tver-reservation.png)

しかし、従来のUserNotificationsによる通知では：

- サイレントモードだと気づかない
- 他の通知に埋もれて見逃す

といった問題があり、せっかく視聴予約しても番組を見逃してしまう可能性がありました。

AlarmKitを導入すれば、標準の目覚ましアラームと同じように**確実に**ユーザーに配信開始を知らせることができます。「絶対に見逃したくない番組」のために、通常の通知とアラームを使い分けるUIも考えられそうです。

## まとめ

iOS 26のAlarmKit APIにより、サードパーティアプリでも標準の時計アプリと同等のアラーム機能を実装できるようになりました。

実装のポイント：

- `AlarmPresentation.Alert`でアラート画面の設定
- `AlarmAttributes`で属性の定義
- `AlarmManager.AlarmConfiguration.timer`でタイマー設定
- `alarmManager.schedule`でスケジュール
- `NSAlarmKitUsageDescription`をInfo.plistに必ず追加

今後、UserNotificationsで実装していた通知機能をAlarmKitに置き換えることで、よりユーザーに確実に届くアラーム体験を提供できるでしょう。
みなさんも確実に届くアラーム検討してみてはいかがでしょうか？

## 参考資料

- [WWDC25 - App alarms with the AlarmKit API](https://developer.apple.com/jp/videos/play/wwdc2025/230/)
- [AlarmKit | Apple Developer Documentation](https://developer.apple.com/documentation/alarmkit)
- [Bell - AlarmKit実装サンプルアプリ](https://github.com/entaku0818/Bell)
