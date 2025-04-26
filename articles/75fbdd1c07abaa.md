---
title: "【iOS】AVCaptureVideoPreviewLayerでカメラアプリを作る"
emoji: "📸"
type: "tech"
topics:
  - "ios"
  - "avfoundation"
published: true
published_at: "2024-04-04 10:00"
---


AVCaptureVideoPreviewLayerを利用してiOSでカメラアプリを作る方法を解説していきます

AVCaptureVideoPreviewLayerは、AVFoundationフレームワークの一部であり、カメラからのライブビデオフィードを表示するために使用されるレイヤーです。このレイヤーは、AVCaptureSessionと関連付けられ、ビデオ出力をリアルタイムで表示することができます

![](https://storage.googleapis.com/zenn-user-upload/2ac9c1374824-20240402.png)

https://developer.apple.com/documentation/avfoundation/avcapturevideopreviewlayer


## 1. CameraViewModelの実装

まず、カメラ機能を管理するための`CameraViewModel`クラスを実装します。このクラスは、`AVCapturePhotoCaptureDelegate`プロトコルに準拠し、カメラセッションの設定、写真の撮影、撮影した写真の処理を行います。

```swift
class CameraViewModel: NSObject, ObservableObject, AVCapturePhotoCaptureDelegate {
    @Published var capturedImage: UIImage?
    var captureSession: AVCaptureSession?
    var photoOutput: AVCapturePhotoOutput?

    func setupCamera() {
        // カメラセッションの設定
        captureSession = AVCaptureSession()
        captureSession?.sessionPreset = .high
        
        // デバイスの取得
        guard let camera = AVCaptureDevice.default(for: .video) else { return }
        
        // 入力の設定
        do {
            let input = try AVCaptureDeviceInput(device: camera)
            if captureSession!.canAddInput(input) {
                captureSession?.addInput(input)
            }
        } catch {
            print(error)
            return
        }
        
        // 出力の設定
        photoOutput = AVCapturePhotoOutput()
        if captureSession!.canAddOutput(photoOutput!) {
            captureSession?.addOutput(photoOutput!)
        }
        
        // セッションの開始
        captureSession?.startRunning()
    }

    func takePhoto() {
        // 写真撮影の設定
        let settings = AVCapturePhotoSettings()
        photoOutput?.capturePhoto(with: settings, delegate: self)
    }

    func photoOutput(_ output: AVCapturePhotoOutput, didFinishProcessingPhoto photo: AVCapturePhoto, error: Error?) {
        // 撮影した写真の処理
        guard let imageData = photo.fileDataRepresentation() else { return }
        capturedImage = UIImage(data: imageData)
    }
}
```

## 2. CameraPreviewの実装

次に、カメラプレビューを表示するための`CameraPreview`構造体を実装します。この構造体は、`UIViewRepresentable`プロトコルに準拠し、`UIView`を作成してカメラプレビューを表示します。

```swift
struct CameraPreview: UIViewRepresentable {
    @ObservedObject var cameraViewModel: CameraViewModel

    func makeUIView(context: Context) -> UIView {
        // UIViewの作成
        let view = UIView(frame: UIScreen.main.bounds)
        
        // カメラセットアップ
        cameraViewModel.setupCamera()
        
        // プレビューレイヤーの設定
        let previewLayer = AVCaptureVideoPreviewLayer(session: cameraViewModel.captureSession!)
        previewLayer.frame = view.bounds
        previewLayer.videoGravity = .resizeAspectFill
        view.layer.addSublayer(previewLayer)
        
        return view
    }

    func updateUIView(_ uiView: UIView, context: Context) {}
}
```

## 3. CameraPreviewContentViewの実装

`CameraPreviewContentView`構造体は、カメラプレビューとシャッターボタンを組み合わせたViewです。`ZStack`を使ってカメラプレビューとボタンを重ねて表示し、シャッターボタンのタップで写真撮影を行います。

```swift
struct CameraPreviewContentView: View {
    @StateObject private var cameraViewModel = CameraViewModel()

    var body: some View {
        ZStack {
            // カメラプレビュー
            CameraPreview(cameraViewModel: cameraViewModel)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                Spacer()
                
                // シャッターボタン
                Button(action: {
                    cameraViewModel.takePhoto()
                }) {
                    Image(systemName: "circle")
                        .font(.system(size: 70))
                        .foregroundColor(.white)
                }
                
                Spacer().frame(height: 50)
            }
        }
        .sheet(item: $cameraViewModel.capturedImage) { image in
            // 撮影した写真の表示
            ImageView(image: image)
        }
    }
}
```

## 4. ImageViewの実装

`ImageView`構造体は、撮影した写真を表示するためのViewです。`Image`を使って`UIImage`を表示し、`resizable()`と`aspectRatio()`モディファイアを使ってサイズ調整を行います。

```swift
struct ImageView: View {
    let image: UIImage

    var body: some View {
        Image(uiImage: image)
            .resizable()
            .aspectRatio(contentMode: .fit)
    }
}
```

## 5. UIImageのIdentifiableプロトコル適合

`UIImage`を`Identifiable`プロトコルに適合させるために拡張を定義します。これにより、`UIImage`を`.sheet`モディファイアで使用することができます。

```swift
extension UIImage: Identifiable {
    public var id: UUID {
        UUID()
    }
}
```

以上が、SwiftUIでカメラ機能を実装するためのサンプルコードです。`CameraViewModel`でカメラの設定と写真撮影を行い、`CameraPreview`でカメラプレビューを表示します。`CameraPreviewContentView`でカメラプレビューとシャッターボタンを組み合わせ、撮影した写真を`ImageView`で表示します。

サンプルコード
https://github.com/entaku0818/AudioMaster/tree/main/AudioMasterApp/AudioMasterApp/AudioMasterApp/AVF/CameraPreview