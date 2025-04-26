---
title: "AVFoundation APIs for Professional Video Workflows"
emoji: "🕌"
type: "tech"
topics:
  - "avfoundation"
published: true
published_at: "2024-06-15 10:30"
---

# この記事について

この記事はApple - AVFoundationの記事をまとめたものです
https://developer.apple.com/av-foundation/

## OS X Yosemiteの新しいAV Foundation APIについて

OS X YosemiteのAV Foundationは、プロフェッショナルなビデオワークフローのための新しい機能を導入しています。これには、メディア内のサンプルを反復および検査するためのクラス、URL参照ムービーのサポート、フラグメント化ムービーファイルの書き込み、DVファイル形式のサポートの強化、および非圧縮ムービーサポートが含まれます。

### メディアサンプルのイントロスペクションと読み込みのための新しいクラス：AVSampleCursorおよびAVSampleBufferGenerator

#### AVSampleCursor
`AVAssetTrack`は、`AVSampleCursor`オブジェクトを作成するためのメソッドをサポートしています。`AVSampleCursor`オブジェクトを使用すると、メディア内のサンプルを反復および検査することができます。

```objective-c
AVAssetTrack *assetTrack = <#An AVAssetTrack#>;
if (assetTrack.canProvideSampleCursors) {
    AVSampleCursor *cursor = [assetTrack makeSampleCursorAtFirstSampleInDecodeOrder];
    // AVSampleCursorを使用して何か面白いことをする
}
```

指定されたプレゼンテーションタイムスタンプで`AVSampleCursor`を作成および初期化することもできます。

```objective-c
AVAssetTrack *assetTrack = <#An AVAssetTrack#>;
if (assetTrack.canProvideSampleCursors) {
    CMTime presentationTimeStamp = <#A time stamp#>;
    AVSampleCursor *cursor = [assetTrack makeSampleCursorWithPresentationTimeStamp:presentationTimeStamp];
    // AVSampleCursorを使用して何か面白いことをする
}
```

#### AVSampleBufferGenerator
`AVSampleBufferGenerator`クラスは、`AVSampleCursor`によって参照されるサンプルを`CMSampleBuffer`オブジェクトに読み込むための柔軟なサービスを提供します。使用するには、まず`AVSampleBufferRequest`オブジェクトを作成し、そのプロパティを設定してリクエストを構成します。

```objective-c
AVSampleCursor *cursor = <#An AVSampleCursor#>;
AVSampleBufferRequest *sampleBufferRequest = [[AVSampleBufferRequest alloc] initWithStartCursor:cursor];
if (sampleBufferRequest) {
    sampleBufferRequest.direction = AVSampleBufferRequestDirectionForward;
    // AVSampleBufferRequestを使用して何か面白いことをする
}
```

`AVSampleBufferGenerator`オブジェクトを`AVAsset`インスタンスから作成し、`createSampleBufferForRequest`メソッドを使用して`CMSampleBuffer`を生成します。

```objective-c
AVSampleBufferRequest *sampleBufferRequest = <#An AVSampleBufferRequest for the AVAsset#>;
AVAsset *asset = <#The AVAsset#>;
AVSampleBufferGenerator *generator = [[AVSampleBufferGenerator alloc] initWithAsset:asset timebase:NULL];
if (generator) {
    CMSampleBufferRef sampleBuffer = [generator createSampleBufferForRequest:sampleBufferRequest];
    // サンプルバッファを使用して何か面白いことをする
    CFRelease(sampleBuffer);
}
```

### URL参照ムービーのサポート
OS X Yosemiteでは、AV FoundationはURL参照ムービーをサポートします。これにより、ムービーファイルは実際のサンプルデータを含まず、他のファイルへの相対または絶対URLを含むことができます。

### フラグメント化ムービーのサポート
AVFoundationの`AVCaptureMovieFileOutput`および`AVAssetWriter` APIは、フラグメント化ムービーファイルの書き込みをサポートします。これにより、クラッシュ耐性が向上し、ファイル書き込みが突然停止しても最新のフラグメントのみが失われるようになります。

### DVストリームファイル形式のサポート
OS X Yosemiteでは、AVFoundation/CoreMediaワークフローにDVストリームファイルの再生サポートが統合されています。

### 非圧縮ムービーのサポート
OS X Yosemiteでサポートされている非圧縮ムービー形式には、R10k、v210、2vuyが含まれます。

このように、OS X YosemiteのAV Foundationは、プロフェッショナルなビデオワークフローに対応する新機能を提供しています。