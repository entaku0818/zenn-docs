---
title: "fastlaneを捨てて、SwiftでApp Store Connect CLIを自作した話"
emoji: "🚀"
type: "tech"
topics:
  - "swift"
  - "ios"
  - "fastlane"
  - "appstoreconnect"
  - "cli"
published: true
published_at: "2026-06-20 09:00"
---

:::message
個人で12本のiOSアプリを運用しながら fastlane を完全に置き換えた実体験です。コードは実際に動くものを掲載しています。
:::

## なぜfastlaneを捨てたのか

iOSアプリをリリースするとき、あなたはfastlaneに何をやってもらっているか説明できますか？

個人で12本のiOSアプリを運用していると、リリース作業は週次の定常業務になります。そのたびに立ちはだかるのが **Rubyの環境管理** です。

```
rbenv install → bundle install → bundle exec fastlane ...
```

Swiftを書いているのになぜRubyと戦わなければならないのか——ある日そう気づいて、fastlaneを捨てて **Swiftでリリース用CLIを自作** することにしました。

それが **[launchpad](https://github.com/entaku0818/launchpad)** です。6週間・100コミット以上かかりました。

---

## App Store Connect APIの基本：JWTは30行で書ける

App Store Connect APIはただのREST APIです。認証はJWT（ES256署名）で行います。
CryptoKitが標準ライブラリにあるので、外部依存はゼロです。

```swift
import CryptoKit
import Foundation

struct ASCCredentials {
    let keyID: String
    let issuerID: String
    let keyContent: String  // PEM形式の秘密鍵

    func generateJWT() throws -> String {
        let now = Int(Date().timeIntervalSince1970)

        let header = try jsonBase64url(["alg": "ES256", "kid": keyID, "typ": "JWT"])
        let payload = try jsonBase64url([
            "iss": issuerID,
            "iat": now,
            "exp": now + 1200,      // 20分で期限切れ
            "aud": "appstoreconnect-v1",
        ] as [String: Any])

        let signingInput = "\(header).\(payload)"
        let privateKey = try P256.Signing.PrivateKey(pemRepresentation: keyContent)
        let signature = try privateKey.signature(for: Data(signingInput.utf8))

        return "\(signingInput).\(base64url(signature.rawRepresentation))"
    }
}

// Base64URL エンコード（標準Base64の + → - / → _ = → 除去）
private func base64url(_ data: Data) -> String {
    data.base64EncodedString()
        .replacingOccurrences(of: "+", with: "-")
        .replacingOccurrences(of: "/", with: "_")
        .replacingOccurrences(of: "=", with: "")
}

private func jsonBase64url(_ dict: [String: Any]) throws -> String {
    base64url(try JSONSerialization.data(withJSONObject: dict))
}
```

これだけです。Ruby・OpenSSL・Gemfile.lockは不要。

---

## fastlaneが隠してくれていた5つの罠

APIを直接叩くと、fastlaneが優しく隠してくれていた現実が見えてきます。

### 罠1：空のrelease_notes.txtを送るとApp Storeの説明文が全消えする

```swift
// ❌ ファイルが空だと "" を送信 → 既存の説明文が上書きされて消える
let notes = try String(contentsOfFile: "fastlane/metadata/ja/release_notes.txt")

// ✅ 空チェック必須
guard !notes.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty else {
    print("⚠️ release_notes.txt が空です。スキップします。")
    return
}
```

launchpadでは `launchpad ios metadata` 実行前に自動でチェックして警告を出します。

### 罠2：スクリーンショットが何のエラーも出ずに全件スキップされる

スクリーンショット登録は **ASC → S3 → ASC の3段階**が必要です。

```swift
// Step 1: ASC にアップロード枠を予約する
let reservation = try await client.reserveScreenshot(appID: appID, displayType: displayType)

// Step 2: 返ってきた S3 URL に画像を直接 PUT する（ここが盲点）
var request = URLRequest(url: reservation.uploadOperations[0].url)
request.httpMethod = "PUT"
try await URLSession.shared.upload(for: request, from: imageData)

// Step 3: ASC に「完了した」と通知する
try await client.commitScreenshot(screenshotID: reservation.id)
```

Step 2 を忘れると、ASCは予約をそのまま黙って破棄します。エラーは一切出ません。

### 罠3：バージョンが存在するのに「not found」が返ってくる

App Store Connect には **状態機械** があります。バージョンの状態によっては、存在していても API から見えません。

```swift
// ❌ フィルタなしだと「ない」と返ってくることがある
let versions = try await client.getAppStoreVersions(appID: appID)

// ✅ 状態フィルタを明示する
let versions = try await client.getAppStoreVersions(
    appID: appID,
    filter: [.appStoreState([.prepareForSubmission, .readyForSale])]
)
```

### 罠4：スクリーンショットの `displayType` が直感に反する

「iPhone 6.5インチ用」のつもりで `APP_IPHONE_65` を指定したら `APP_IPHONE_67` が正解だった、というケースがあります。端末世代とAPIの列挙値は1:1で対応していません。

launchpad では `fastlane/screenshots/` のディレクトリ名（`iPhone65`、`iPhone67` 等）から自動的に正しい `displayType` を解決します。

### 罠5：APIエラーのスキーマがエンドポイントごとに違う

```json
// App Store Connect API のエラー
{"errors": [{"status": "409", "code": "VERSION_ALREADY_EXISTS", "title": "..."}]}

// 一部エンドポイントのエラー（フォーマットが異なる）
{"errors": [{"status": "404", "code": "NOT_FOUND"}]}
```

汎用パーサーを1つ作れば全部捌けると思うと足元をすくわれます。エンドポイントのグループごとに個別対応が必要です。

---

## fastlane越しには知らなかった機能

APIを直接触ると、fastlaneのラッパー越しでは存在すら知らなかった機能が見えてきます。launchpadでは実際にこれらを実装しています。

```bash
# Win-back Offers：解約ユーザーへの特別価格オファーを管理
launchpad ios win-back-offers list

# Product Page Optimization：ストアページのA/Bテスト
launchpad ios product-pages list

# App Store Server API：サブスクリプション状態をリアルタイム確認
launchpad ios server-api subscription --original-transaction-id 123456789

# Xcode Cloud API：CIビルドをCLIから操作
launchpad ios xcode-cloud workflows list
```

---

## launchpadで何ができるか

### ワンショットリリース

```bash
# ビルド → アップロード → 審査申請 を全自動
launchpad release

# iOSのみ
launchpad release --ios

# 実際には実行せず確認だけ
launchpad release --dry-run
```

### よく使うコマンド

```bash
# 環境・設定の一括チェック
launchpad doctor

# バージョン管理
launchpad ios version-bump version --patch   # 1.0.0 → 1.0.1
launchpad ios version-bump build             # ビルド番号を +1

# TestFlight アップロード
launchpad ios upload

# メタデータ・スクリーンショット
launchpad ios metadata
launchpad ios screenshots --overwrite

# 審査提出
launchpad ios submit

# gitコミット履歴からリリースノートを自動生成
launchpad release-notes --ios --locale all
```

### ASO（App Store最適化）もついてる

```bash
# キーワード順位チェック（無料・認証不要）
launchpad ios aso keyword-rank \
  --bundle-id com.example.app \
  --keywords "録音,ボイスメモ,文字起こし" \
  --country jp

# 出力例
# 🟢 #3   録音
# 🟡 #24  ボイスメモ
# 🔴 #87  文字起こし

# メタデータの品質チェック（文字数・重複検出）
launchpad ios aso metadata-check

# レビューの感情分析・頻出ワード抽出
launchpad ios aso review-summary --bundle-id com.example.app --limit 200
```

### Androidにも対応

```bash
launchpad android build --release
launchpad android upload --aab app-release.aab
launchpad android promote --from internal --to production
launchpad android aso metadata-check
```

---

## セットアップは3ステップ

```bash
# インストール（universal binary）
curl -fsSL https://raw.githubusercontent.com/entaku0818/launchpad/main/install.sh | sh

# プロジェクト初期化
launchpad init    # .launchpadrc と .env.template を生成

# 環境チェック
launchpad doctor
```

`.env` に App Store Connect API キーを書くだけで動き始めます。

---

## まとめ

App Store Connect APIを直接叩くことで：

1. **fastlaneが何をしていたかわかった**（と同時に優しさも知った）
2. **存在すら知らなかった機能に出会えた**（Win-back Offers, PPO, Server API...）
3. **Rubyの環境管理から完全に解放された**

「fastlaneが複雑すぎる」「Rubyが辛い」と感じているiOS開発者は、一度APIを素で叩いてみることをおすすめします。黒箱が開く体験は、思ったより気持ちいいです。

👉 https://github.com/entaku0818/launchpad

---

*個人で12本のiOSアプリを運用中。自動化ネタは [X(@entaku0818)](https://x.com/entaku0818) で発信しています。*
