---
title: "【Flutter】音声を再生する"
emoji: "🎶"
type: "tech"
topics:
  - "flutter"
  - "audio"
published: true
published_at: "2024-04-01 10:00"
---

# Flutterでオーディオプレーヤーを実装する

今回は、Flutterを使用してオーディオプレーヤーを実装する方法を解説します。このオーディオプレーヤーは、再生、停止、早送り、減速、スキップ機能を備えています。

下記にサンプルコードを置いてます
https://github.com/entaku0818/flutter_master/blob/main/lib/AudioPlayerPage.dart

## 必要なパッケージのインストール

まず、`pubspec.yaml`ファイルに以下の依存関係を追加します。

```yaml
dependencies:
  audioplayers: ^5.0.0
```

`audioplayers`パッケージは、Flutterでオーディオを再生するための機能を提供します。
https://pub.dev/packages/audioplayers

## オーディオプレーヤーの実装

次に、`AudioPlayerPage`というStatefulWidgetを作成します。このウィジェットは、オーディオプレーヤーのUIと機能を実装します。

```dart
import 'package:flutter/material.dart';
import 'package:audioplayers/audioplayers.dart';

class AudioPlayerPage extends StatefulWidget {
  @override
  _AudioPlayerPageState createState() => _AudioPlayerPageState();
}

class _AudioPlayerPageState extends State<AudioPlayerPage> {
  // 変数の宣言
  late AudioCache _audioCache;
  late AudioPlayer _audioPlayer;
  double _playbackRate = 1.0;

  // 以下にメソッドを実装していきます
}
```


### 初期化と後処理

`initState()`メソッドで、`_audioCache`と`_audioPlayer`を初期化します。`_audioCache`のプレフィックスは、オーディオファイルが格納されているディレクトリを指定します。

```dart
@override
void initState() {
  super.initState();
  _audioCache = AudioCache(prefix: 'assets/audio/');
  _audioPlayer = AudioPlayer();
}
```

`dispose()`メソッドで、`_audioPlayer`を破棄し、リソースを解放します。

```dart
@override
void dispose() {
  _audioPlayer.dispose();
  super.dispose();
}
```

### オーディオの再生と停止

`_playAudio()`メソッドは、オーディオを再生します。`_audioPlayer.play()`メソッドを呼び出し、再生するオーディオファイルのアセットパスを指定します。エラーが発生した場合は、キャッチブロックでエラーメッセージを出力します。

```dart
void _playAudio() async {
  try {
    await _audioPlayer.play(AssetSource('audio/RPG_Battle_01.mp3'));
  } catch (e) {
    print('Error playing audio: $e');
  }
}
```

`_stopAudio()`メソッドは、オーディオの再生を停止します。`_audioPlayer.stop()`メソッドを呼び出します。

```dart
void _stopAudio() async {
  await _audioPlayer.stop();
}
```

### 再生速度の変更

`_fastForward()`メソッドは、再生速度を0.5倍速くします。`_slowDown()`メソッドは、再生速度を0.5倍遅くします。どちらのメソッドも、`setState()`を呼び出して`_playbackRate`の値を更新し、`_audioPlayer.setPlaybackRate()`メソッドを使用して再生速度を設定します。再生速度は0.5から2.0の範囲内に制限されます。

```dart
void _fastForward() {
  setState(() {
    _playbackRate += 0.5;
    _playbackRate = _playbackRate.clamp(0.5, 2.0);
    _audioPlayer.setPlaybackRate(_playbackRate);
  });
}

void _slowDown() {
  setState(() {
    _playbackRate -= 0.5;
    _playbackRate = _playbackRate.clamp(0.5, 2.0);
    _audioPlayer.setPlaybackRate(_playbackRate);
  });
}
```

### スキップ機能

`_skipForward()`メソッドは、再生位置を10秒先にスキップします。`_skipBackward()`メソッドは、再生位置を10秒前にスキップします。どちらのメソッドも、`_audioPlayer.getCurrentPosition()`メソッドを使用して現在の再生位置を取得し、`_audioPlayer.seek()`メソッドを使用して再生位置を更新します。スキップ後の再生位置が負の値になる場合は、再生位置を0秒に設定します。

```dart
void _skipForward() async {
  Duration? currentPosition = await _audioPlayer.getCurrentPosition();
  if (currentPosition != null) {
    Duration newPosition = currentPosition + Duration(seconds: 10);
    await _audioPlayer.seek(newPosition);
  }
}

void _skipBackward() async {
  Duration? currentPosition = await _audioPlayer.getCurrentPosition();
  if (currentPosition != null) {
    Duration newPosition = currentPosition - Duration(seconds: 10);
    if (newPosition.inMilliseconds < 0) {
      newPosition = Duration.zero;
    }
    await _audioPlayer.seek(newPosition);
  }
}
```

### UIの構築

`build()`メソッドでは、オーディオプレーヤーのUIを構築します。`Scaffold`ウィジェットを使用して、アプリバーとボディを定義します。ボディには、再生、停止、早送り、減速、スキップボタンを配置します。ボタンをタップすると、対応するメソッドが呼び出されます。

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text('Audio Player Page'),
    ),
    body: Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          ElevatedButton(
            onPressed: _playAudio,
            child: Text('Play Audio'),
          ),
          SizedBox(height: 16),
          ElevatedButton(
            onPressed: _stopAudio,
            child: Text('Stop Audio'),
          ),
          SizedBox(height: 16),
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                onPressed: _slowDown,
                child: Text('Slow Down'),
              ),
              SizedBox(width: 16),
              ElevatedButton(
                onPressed: _fastForward,
                child: Text('Fast Forward'),
              ),
            ],
          ),
          SizedBox(height: 16),
          Row(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              ElevatedButton(
                onPressed: _skipBackward,
                child: Text('Skip Backward'),
              ),
              SizedBox(width: 16),
              ElevatedButton(
                onPressed: _skipForward,
                child: Text('Skip Forward'),
              ),
            ],
          ),
        ],
      ),
    ),
  );
}
```

## まとめ

以上が、Flutterを使用してオーディオプレーヤーを実装する方法の解説です。`audioplayers`パッケージを使用することで、簡単にオーディオの再生や操作を行うことができます。この例では、再生、停止、早送り、減速、スキップ機能を実装しましたが、必要に応じて他の機能を追加することもできます。
