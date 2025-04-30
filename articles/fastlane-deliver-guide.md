---
title: "fastlaneのdeliverアクションでApp Storeへのデプロイを自動化する"
emoji: "🚀"
type: "tech"
topics: ["ios", "fastlane", "appstore", "ci"]
published: false
---

# fastlaneのdeliverアクションでApp Storeへのデプロイを自動化する

## はじめに

iOSアプリのApp Storeへのデプロイは、スクリーンショットのアップロード、メタデータの更新、バイナリのアップロードなど、多くの手動作業が必要です。これらの作業を自動化するために、fastlaneの`deliver`アクションを使用することができます。

## セットアップ手順

### 1. fastlaneの初期化

プロジェクトのルートディレクトリで以下のコマンドを実行します：

```bash
fastlane init
```

### 2. deliverの初期化

```bash
fastlane deliver init
```

このコマンドを実行すると、以下のファイルが作成されます：
- `fastlane/metadata/`: メタデータを格納するディレクトリ
- `fastlane/screenshots/`: スクリーンショットを格納するディレクトリ

### 3. Fastfileの設定

`fastlane/Fastfile`に以下のような設定を追加します：

```ruby
# This file contains the fastlane.tools configuration
# You can find the documentation at https://docs.fastlane.tools
#
# For a list of all available actions, check out
#
#     https://docs.fastlane.tools/actions
#
# For a list of all available plugins, check out
#
#     https://docs.fastlane.tools/plugins/available-plugins
#

# Uncomment the line if you want fastlane to automatically update itself
# update_fastlane

default_platform(:ios)

platform :ios do
  # 各レーンで共通して実行する認証処理
  before_all do
    # APIキー認証を使用する場合
    app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
      key_content: ENV["APP_STORE_CONNECT_API_KEY_CONTENT"]  
    )
  end

  # メタデータとスクリーンショットのアップロード、審査提出
  desc "Upload metadata and screenshots to App Store Connect and submit for review"
  lane :upload_metadata do
    # Xcodeプロジェクトからバージョン番号を取得
    version_number = get_version_number(
      xcodeproj: "VoiLog.xcodeproj",
      target: "VoiLog"
    )
    
    # バージョン番号が取得できなかった場合はエラー
    UI.user_error!("バージョン番号をXcodeプロジェクトから取得できませんでした。") unless version_number && !version_number.empty?
    
    UI.success("Xcodeプロジェクトからバージョン番号を取得しました: #{version_number}")
    
    deliver(
      skip_binary_upload: false,     # バイナリのアップロードを実行
      skip_app_version_update: false, # アプリバージョンの更新をスキップしない
      app_version: version_number,    # アプリのバージョン番号を指定
      skip_metadata: false,          # メタデータの更新をスキップしない
      skip_screenshots: false,       # スクリーンショットのアップロードをスキップしない
      force: true,                   # 確認なしで強制的に実行
      overwrite_screenshots: true,   # 既存のスクリーンショットを上書き
      ignore_language_directory_validation: true, # 言語ディレクトリの検証を無視
      run_precheck_before_submit: true,  # 提出前の事前チェックを実行
      precheck_include_in_app_purchases: false,  # In-app purchasesのチェックをスキップ
      submit_for_review: true,       # 審査提出を実行
      automatic_release: false,      # 審査通過後の自動リリースは無効
      submission_information: {
        add_id_info_uses_idfa: false,  # IDFAの使用なし
        export_compliance_uses_encryption: false,  # 暗号化なし
        export_compliance_platform: "ios"  # プラットフォームはiOS
      }
    )
  end
end
```

このFastfileでは以下の設定を行っています：

1. App Store Connect API認証の設定
   - APIキーを使用した認証を設定
   - 環境変数から認証情報を取得
   - 必要な環境変数：
     - `APP_STORE_CONNECT_API_KEY_KEY_ID`: APIキーのID
     - `APP_STORE_CONNECT_API_KEY_ISSUER_ID`: 発行者ID
     - `APP_STORE_CONNECT_API_KEY_CONTENT`: APIキーの内容（.p8ファイルの内容）

   APIキーの取得方法：
   1. App Store Connectにログイン
   2. 「ユーザーとアクセス」→「キー」を選択
   3. 「+」ボタンをクリックして新しいキーを生成
   4. キー名を入力し、必要な権限を選択
   5. 生成されたキーをダウンロード（.p8ファイル）
   6. キーIDと発行者IDをメモ
   7. .p8ファイルの内容を環境変数に設定

2. `upload_metadata`レーンの設定
   - Xcodeプロジェクトからバージョン番号を自動取得
   - メタデータとスクリーンショットのアップロード
   - バイナリのアップロード
   - 審査提出の設定
   - 提出情報（IDFA、暗号化など）の設定

## 既存のメタデータとスクリーンショットのダウンロード

既存のApp Store Connectの設定をfastlaneに移行するのは、実はとても簡単です。手動で設定をコピーする必要はなく、`deliver`アクションが自動的に既存の設定をダウンロードしてくれます。以下の手順で、現在のApp Store Connectの設定を簡単にfastlaneの環境に移行できます：

### 1. deliverの初期化

```bash
fastlane deliver init
```

このコマンドを実行すると、以下のファイルとディレクトリが作成されます：
- `fastlane/metadata/`: メタデータを格納するディレクトリ
- `fastlane/screenshots/`: スクリーンショットを格納するディレクトリ

### 2. メタデータとスクリーンショットのダウンロード

```bash
fastlane deliver download_metadata
fastlane deliver download_screenshots
```

これらのコマンドを実行すると、App Store Connectから以下の情報がダウンロードされます：

- メタデータ
  - アプリ名
  - 説明文
  - キーワード
  - プライバシーポリシーURL
  - サポートURL
  - マーケティングURL
  - プライス情報
  - カテゴリ
  - 年齢制限
  - 著作権情報
  - リリースノート

- スクリーンショット
  - 各デバイスサイズのスクリーンショット
  - 各言語のスクリーンショット
  - App Preview動画（存在する場合）

### 3. ダウンロードしたファイルの確認

ダウンロードしたファイルは以下のディレクトリに保存されます：

```
fastlane/
  ├── metadata/
  │   ├── en-US/
  │   │   ├── description.txt
  │   │   ├── keywords.txt
  │   │   ├── name.txt
  │   │   └── release_notes.txt
  │   └── ja/
  │       ├── description.txt
  │       ├── keywords.txt
  │       ├── name.txt
  │       └── release_notes.txt
  └── screenshots/
      ├── en-US/
      │   ├── iPhone 14 Pro-1.png
      │   ├── iPhone 14 Pro-2.png
      │   └── ...
      └── ja/
          ├── iPhone 14 Pro-1.png
          ├── iPhone 14 Pro-2.png
          └── ...
```

## deliverアクションとは

[`deliver`](https://docs.fastlane.tools/actions/deliver/)アクションは、App Store Connectへのアップロードを自動化するためのfastlaneのアクションです。以下の機能を提供します：

- スクリーンショットのアップロード
- メタデータの更新
- バイナリ（.ipa）のアップロード
- アプリの審査提出
- リリースノートの更新

## 使用方法

Fastfileで定義した`upload_metadata`レーンを使用して、メタデータとスクリーンショットのアップロード、バイナリのアップロード、審査提出を実行できます：

```bash
fastlane upload_metadata
```

## Xcode Cloudとの連携

Xcode Cloudを使用することで、CI/CDパイプラインを構築し、App Storeへのデプロイをさらに自動化することができます。以下に、Xcode Cloudとfastlaneの`deliver`を連携させる方法を説明します。

### 1. Xcode Cloudの設定

1. Xcodeでプロジェクトを開き、Product > Xcode Cloud > Create Workflowを選択
2. ワークフローの設定で以下を選択：
   - トリガー：手動実行または特定のブランチへのプッシュ
   - アクション：Build and Test
   - 環境：macOS

### 2. Post-Actionsスクリプトの追加

Xcode Cloudのワークフロー設定で、Post-Actionsスクリプトを追加します。以下のようなスクリプトを作成します：

```bash
#!/bin/bash

# fastlaneのインストール
brew install fastlane

# プロジェクトディレクトリに移動
cd $CI_WORKSPACE

# deliverの実行
fastlane deliver \
  --ipa "$CI_PRODUCT_PATH" \
  --skip_screenshots \
  --skip_metadata \
  --force \
  --precheck_include_in_app_purchases
```

### 3. 環境変数の設定

Xcode Cloudのワークフロー設定で、以下の環境変数を設定します：

- `FASTLANE_USER`: App Store Connectのユーザー名
- `FASTLANE_PASSWORD`: App Store Connectのパスワード
- `FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD`: アプリケーション固有のパスワード（2要素認証を使用している場合）

### 4. 証明書とプロビジョニングプロファイルの設定

Xcode Cloudで使用する証明書とプロビジョニングプロファイルを設定します：

1. Xcode Cloudのワークフロー設定で「Signing & Capabilities」を選択
2. 適切な証明書とプロビジョニングプロファイルを選択
3. 必要に応じて「Automatically manage signing」を有効化

### 5. ワークフローの実行

設定が完了したら、以下の方法でワークフローを実行できます：

- 手動実行：Xcode Cloudのダッシュボードから実行
- 自動実行：指定したブランチへのプッシュ時に自動実行

## トラブルシューティング

### 遭遇したトラブル

#### 1. メタデータのダウンロードエラー

App Store Connectに正しい情報が登録されていても、以下のようなエラーが発生する場合があります：

```
The provided entity is missing a required attribute - You must provide a value for the attribute 'contactFirstName' with this request
The provided entity is missing a required attribute - You must provide a value for the attribute 'contactLastName' with this request
The provided entity is missing a required attribute - You must provide a value for the attribute 'contactEmail' with this request
The provided entity is missing a required attribute - You must provide a value for the attribute 'contactPhone' with this request
```

この場合、`fastlane/metadata/`ディレクトリ内の`app_review_information.txt`ファイルを手動で更新する必要があります：

```txt
first_name: John
last_name: Doe
phone_number: +819012345678
email_address: review@example.com
demo_user: demo
demo_password: password
notes: テスト用アカウントでログインできます
```

### エラー解決の手順

1. `fastlane/metadata/`ディレクトリ内の`app_review_information.txt`ファイルを確認
2. 必須項目（first_name, last_name, phone_number, email_address）が全て含まれているか確認
3. 電話番号が国際形式（+から始まる）になっているか確認
4. ファイルを更新後、再度`fastlane deliver`を実行

## まとめ

fastlaneの`deliver`アクションを使用することで、App Storeへのデプロイプロセスを大幅に自動化できます。これにより、開発者はより重要な開発作業に集中することができ、デプロイのミスを減らすことができます。

## 参考リンク

- [fastlane deliver公式ドキュメント](https://docs.fastlane.tools/actions/deliver/)
- [App Store Connect API](https://developer.apple.com/app-store-connect/api/) 