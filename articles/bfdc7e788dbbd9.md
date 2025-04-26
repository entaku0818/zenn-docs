---
title: "Swiftでの音声再生方法：AVAudioPlayerの活用"
emoji: "🔊"
type: "tech"
topics:
  - "swift"
  - "avaudioplayer"
published: true
published_at: "2024-01-06 20:00"
---

Swiftでを使い基本的な音声再生機能の設定と使用方法について解説します。

参考となるコードはGitHubに載せておりますので参考にしてください。

https://github.com/entaku0818/AudioMaster/blob/main/Sources/AudioMaster/AudioPlayerMaster.swift


### AVAudioPlayerの初期化
SwiftでのAVAudioPlayerの初期化は、まずAVAudioSessionを設定し、その後AVAudioPlayerインスタンスを生成することから始まります。

```swift
do {
    try audioSession.setCategory(.playback, mode: .default, options: [])
    try audioSession.setActive(true)
    audioPlayer = try AVAudioPlayer(contentsOf: audioFileURL)
    audioPlayer.prepareToPlay()
} catch {
    fatalError("Error initializing audio player: \(error.localizedDescription)")
}
```

AVAudioPlayerは初期化時に音声ファイル情報もしくは音声URLが必要です。
また読み込まれた音声が再生可能かを `prepareToPlay`　で確認し初期化処理の完了としています。

https://developer.apple.com/documentation/avfaudio/avaudioplayer

### 音声の再生/一時停止/停止
基本的な音声操作には再生、一時停止、停止が含まれます。以下のメソッドはこれらの機能を実現します：

```swift
    public func playAudio(atTime time: TimeInterval) {
        audioPlayer.play(atTime: time)
    }

public func pauseAudio() {
    audioPlayer.pause()
}

public func stopAudio() {
    audioPlayer.stop()
}
```

`playAudio`メソッドで音声を再生し、`pauseAudio`で一時停止、`stopAudio`で完全に停止させることができます。
またplayAudioではatTimeで開始時間を変更することができます。

### 音量変更
音量の調整はアプリにとって重要な機能の一つです。以下のメソッドは音量を変更し、必要に応じてフェード効果を適用します：

```swift
public func setVolume(_ volume: Float, fadeDuration duration: TimeInterval) {
    audioPlayer.setVolume(volume, fadeDuration: duration)
}
```


### 　音声スピード変更

```swift
public func setVolume(_ volume: Float, fadeDuration duration: TimeInterval) {
    audioPlayer.setVolume(volume, fadeDuration: duration)
}
```

### まとめ
この記事では、Swiftを用いた基本的な音声再生機能の設定と使用方法について説明しました。AVAudioPlayerの初期化、音声の再生/一時停止/停止、および音量変更の方法を学び、これらの知識を使って様々な設定や機能を追加し、アプリのニーズに合わせた音声再生機能を実装することができます。