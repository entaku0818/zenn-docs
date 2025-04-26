---
title: "TCAのOSSにコントリビュートした話"
emoji: "🛠️"
type: "tech"
topics:
  - "swift"
  - "swiftui"
  - "tca"
published: true
published_at: "2022-09-06 09:05"
---

# TCAとは
TCAとはSwiftUIやSwiftを利用したアプリケーションフレームワークです。
async/awaitやcombineなどのSwiftUIやSwiftの仕様を生かしたフレームワークであることが特徴です。

今まで僕が触った技術の中ではRiverpodと似た設計であるような感じだなと思いました。

その他詳しい内容はyimajoさんの記事がわかりやすいので参考にしてください。
https://qiita.com/yimajo/items/77c204ab091223f9cb14


# コントリビュートした内容
今回この中でも「VoiceMemos」というExamplesを利用しようとして発覚しました。
https://github.com/pointfreeco/swift-composable-architecture/tree/main/Examples/VoiceMemos

結論から言うと変更をしたプルリクです。

https://github.com/pointfreeco/swift-composable-architecture/pull/1334

```swift
-        try AVAudioSession.sharedInstance().setCategory(.record, mode: .default)
+        try AVAudioSession.sharedInstance().setCategory(.playAndRecord, mode: .default, options: .defaultToSpeaker)
```

# どのようにして問題に気が付いたか？
Exampleをビルドして実行しただけでした。
sampleを実行してみたのですが、全く音声が再生されないっ....と言うことに気がつきました。
おそらく今まで一度もsampleアプリで音声を記録再生した人がいなかったのでしょう。

![サンプルを少し改変して試している図](https://storage.googleapis.com/zenn-user-upload/d9f71b2e99db-20220906.png =250x)

そこでコードを丁寧にみてみたのですが、AVAudioSessionの設定が自分の知っている設定と異なっていたので変更しました。
結果として上手く再生でき、せっかく参考にしたサンプルアプリなのでプルリクを出すことにしました。
即レビューが通りまた自分の使ったOSSに貢献できてよかったです。

https://github.com/pointfreeco/swift-composable-architecture/pull/1334

# まとめ
今回たまたま気づいた内容でコントリビュートできましたが、もしかしたらみなさんのご利用のOSSでも動いてないものなどあるかも...?
OSSライフの参考になったら大変嬉しいです！