---
title: "Agoraを使う"
emoji: "📝"
type: "tech"
topics: []
published: false
---

iOSでAgoraを使用するための基本的なサンプルコードを以下に示します。この例では、Agora SDKを使ってビデオ通話機能を実装する方法を説明します。まずは、Agora SDKをプロジェクトに組み込む手順を完了させてください。

### SwiftでのAgora SDKの初期化

以下のサンプルコードは、Agora SDKを初期化し、ビデオ通話を開始する方法を示しています。これを実装するには、まず`ViewController.swift`ファイル（または適切なViewControllerクラス）にコードを追加します。

```swift
import UIKit
import AgoraRtcKit

class ViewController: UIViewController {
    
    var agoraKit: AgoraRtcEngineKit?
    let AppID = "YourAppID" // Agora.ioから取得したApp IDをここに置き換えてください。
    let Token = "YourToken" // トークンセキュリティを有効にしている場合は、ここにトークンを置き換えてください（オプション）。

    override func viewDidLoad() {
        super.viewDidLoad()
        initializeAgoraEngine()
        setupVideo()
        joinChannel()
    }
    
    func initializeAgoraEngine() {
        agoraKit = AgoraRtcEngineKit.sharedEngine(withAppId: AppID, delegate: nil)
    }
    
    func setupVideo() {
        agoraKit?.enableVideo()  // ビデオモジュールを有効にする
        agoraKit?.setVideoEncoderConfiguration(AgoraVideoEncoderConfiguration(
            size: AgoraVideoDimension640x360,
            frameRate: .fps15,
            bitrate: AgoraVideoBitrateStandard,
            orientationMode: .adaptative
        ))
    }
    
    func joinChannel() {
        agoraKit?.joinChannel(byToken: Token, channelId: "demoChannel1", info:nil, uid:0) {(channel, uid, elapsed) -> Void in
            // チャネルに参加した時の処理
            print("Successfully joined channel: \(channel) with uid: \(uid)")
        }
    }
}
```

### 注意点

- 上記のコードでは、`AppID`と`Token`を適切な値に置き換える必要があります。これらの値は、Agoraの開発者アカウントを通じて取得できます。
- `joinChannel`メソッドでは、チャネル名（この例では`demoChannel1`）を指定しています。アプリ内で複数の通話を区別するために、異なるチャネル名を使用することができます。
- トークン認証を使用する場合は、`Token`を生成し、適切な値に置き換えてください。トークンはセキュリティを強化するために使用されますが、開発の初期段階では省略してもかまいません。

このコードは、Agora SDKの基本的なビデオ通話機能を実装するための出発点となります。アプリケーションのニーズに合わせて、より高度な機能やカスタマイズを加えることができます。