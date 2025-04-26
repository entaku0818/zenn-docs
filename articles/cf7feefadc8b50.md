---
title: "Androidの`SpeechRecognizer`を利用した音声認識入門"
emoji: "👋"
type: "tech"
topics: []
published: false
---

# Androidの`SpeechRecognizer`を利用した音声認識入門

Androidアプリケーションにおいて、ユーザーからの音声入力をテキスト化する機能は、アクセシビリティの向上やユーザーエクスペリエンスの豊かさに寄与します。Googleが提供する`SpeechRecognizer`クラスを利用することで、開発者はこのような音声認識機能をアプリに簡単に組み込むことができます。この記事では、`SpeechRecognizer`の基本的な使用方法について解説します。

## 前提条件

- Android Studioがインストールされていること。
- Android SDK Platform 21以上がインストールされていること。
- 実機またはエミュレータでのテストを行う。

## `SpeechRecognizer`の基本

`SpeechRecognizer`クラスは、ユーザーの話す言葉をテキストに変換する機能を提供します。このクラスを使用するには、まずマニフェストファイル(`AndroidManifest.xml`)に必要なパーミッションを追加する必要があります。

### パーミッションの追加

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
```

### `SpeechRecognizer`のインスタンス化

`SpeechRecognizer`は直接インスタンス化されるのではなく、`createSpeechRecognizer`コンテキストメソッドを通じて取得します。

```kotlin
val speechRecognizer = SpeechRecognizer.createSpeechRecognizer(context)
```

### `Intent`の設定

音声認識を開始する前に、認識の設定を行う`Intent`を準備します。

```kotlin
val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
    putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
    putExtra(RecognizerIntent.EXTRA_LANGUAGE, Locale.getDefault())
    putExtra(RecognizerIntent.EXTRA_PROMPT, "音声を入力してください")
}
```

### コールバックの設定

`SpeechRecognizer`のイベントを処理するために、`RecognitionListener`を実装します。

```kotlin
speechRecognizer.setRecognitionListener(object : RecognitionListener {
    override fun onReadyForSpeech(bundle: Bundle) {}
    override fun onBeginningOfSpeech() {}
    override fun onRmsChanged(rmsdB: Float) {}
    override fun onBufferReceived(buffer: ByteArray) {}
    override fun onEndOfSpeech() {}
    override fun onError(error: Int) {}
    override fun onResults(results: Bundle) {
        // 認識結果を取得
        val matches = results.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)
        if (matches != null && matches.isNotEmpty()) {
            val text = matches[0] // 最も可能性の高いテキスト
            // ここでテキストを使用する
        }
    }
    override fun onPartialResults(partialResults: Bundle) {}
    override fun onEvent(eventType: Int, params: Bundle) {}
})
```

### 音声認識の開始

設定が完了したら、`startListening`メソッドを呼び出して音声認識を開始します。

```kotlin
speechRecognizer.startListening(intent)
```

## 注意点

- 音声認識はネットワークを利用するため、オフラインでの利用は制限される場合があります。
- `SpeechRecognizer`の使用を終了する時は、`destroy`メソッドを呼び出してリソースを解放することが重要です。
- ユーザーに明確なフィードバックを提供し、音声入力の開始と終了を知らせるUIを実装することで、ユーザーエクスペリエンスを向上させます。

## まとめ

