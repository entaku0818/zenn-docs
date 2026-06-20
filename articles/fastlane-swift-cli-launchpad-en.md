---
title: "I Ditched fastlane and Built My Own App Store Connect CLI in Swift"
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
This is based on my real experience replacing fastlane while managing 12 iOS apps solo. All code in this article actually runs.
:::

## Why I Ditched fastlane

When releasing an iOS app, can you explain exactly what fastlane is doing for you?

Managing 12 iOS apps solo means shipping is a weekly routine. And every week I found myself fighting the same battle: **Ruby environment management**.

```
rbenv install → bundle install → bundle exec fastlane ...
```

I write Swift — why am I constantly fighting Ruby? One day that thought stuck, and I decided to replace fastlane with a **Swift CLI I built myself**.

That tool is **[launchpad](https://github.com/entaku0818/launchpad)**. It took 6 weeks and 100+ commits.

---

## The App Store Connect API is Just a REST API

Authentication uses JWT with ES256 signing. `CryptoKit` ships with the standard library, so zero external dependencies are needed.

```swift
import CryptoKit
import Foundation

struct ASCCredentials {
    let keyID: String
    let issuerID: String
    let keyContent: String  // PEM private key

    func generateJWT() throws -> String {
        let now = Int(Date().timeIntervalSince1970)

        let header = try jsonBase64url(["alg": "ES256", "kid": keyID, "typ": "JWT"])
        let payload = try jsonBase64url([
            "iss": issuerID,
            "iat": now,
            "exp": now + 1200,      // 20-minute expiry
            "aud": "appstoreconnect-v1",
        ] as [String: Any])

        let signingInput = "\(header).\(payload)"
        let privateKey = try P256.Signing.PrivateKey(pemRepresentation: keyContent)
        let signature = try privateKey.signature(for: Data(signingInput.utf8))

        return "\(signingInput).\(base64url(signature.rawRepresentation))"
    }
}

// Base64URL: standard Base64 with + → -, / → _, = stripped
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

No Ruby. No OpenSSL. No Gemfile.lock.

---

## 5 Traps fastlane Was Hiding From You

Going raw on the API reveals what fastlane was kindly abstracting away.

### Trap 1: An empty release_notes.txt silently wipes your App Store description

```swift
// ❌ Empty file sends "" → overwrites your existing description with nothing
let notes = try String(contentsOfFile: "fastlane/metadata/en-US/release_notes.txt")

// ✅ Always guard against empty content
guard !notes.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty else {
    print("⚠️ release_notes.txt is empty. Skipping.")
    return
}
```

`launchpad ios metadata` checks this automatically before uploading.

### Trap 2: Screenshots get silently skipped — no error, no warning

Uploading screenshots requires **3 steps**: ASC → S3 → ASC.

```swift
// Step 1: Reserve an upload slot in ASC
let reservation = try await client.reserveScreenshot(appID: appID, displayType: displayType)

// Step 2: PUT the image directly to the S3 URL ASC gives you (the easy-to-miss step)
var request = URLRequest(url: reservation.uploadOperations[0].url)
request.httpMethod = "PUT"
try await URLSession.shared.upload(for: request, from: imageData)

// Step 3: Notify ASC the upload is complete
try await client.commitScreenshot(screenshotID: reservation.id)
```

Skip Step 2 and ASC silently drops the reservation. No error. Your screenshots just don't appear.

### Trap 3: A version that exists returns "not found"

App Store Connect has a **state machine**. Versions in the wrong state are invisible to the API.

```swift
// ❌ Without a filter, existing versions can appear missing
let versions = try await client.getAppStoreVersions(appID: appID)

// ✅ Always specify state filters
let versions = try await client.getAppStoreVersions(
    appID: appID,
    filter: [.appStoreState([.prepareForSubmission, .readyForSale])]
)
```

### Trap 4: Screenshot `displayType` values are counterintuitive

`APP_IPHONE_65` is not "iPhone 6.5-inch" — `APP_IPHONE_67` is. Device generation doesn't map 1:1 to API enum values.

launchpad resolves the correct `displayType` automatically from your `fastlane/screenshots/` directory names (`iPhone65`, `iPhone67`, etc.).

### Trap 5: Error response schemas differ per endpoint

```json
// Standard ASC error format
{"errors": [{"status": "409", "code": "VERSION_ALREADY_EXISTS", "title": "..."}]}

// Some endpoints use a different structure
{"errors": [{"status": "404", "code": "NOT_FOUND"}]}
```

You can't write one universal error parser. Each endpoint group needs its own handling.

---

## Features I Didn't Know Existed

Looking directly at the API, I discovered capabilities that were invisible through fastlane's abstraction. launchpad implements all of these:

```bash
# Win-back Offers: re-engage churned subscribers with special pricing
launchpad ios win-back-offers list

# Product Page Optimization: A/B test your App Store listing
launchpad ios product-pages list

# App Store Server API: check real-time subscription state
launchpad ios server-api subscription --original-transaction-id 123456789

# Xcode Cloud API: control CI builds from the terminal
launchpad ios xcode-cloud workflows list
```

---

## What launchpad Can Do

### One-shot release

```bash
# Build → upload → submit for review, all automated
launchpad release

# iOS only
launchpad release --ios

# Dry run: preview what would happen without executing
launchpad release --dry-run
```

### Daily commands

```bash
# Check your environment and credentials
launchpad doctor

# Version management
launchpad ios version-bump version --patch   # 1.0.0 → 1.0.1
launchpad ios version-bump build             # increment build number

# Upload to TestFlight
launchpad ios upload

# Sync metadata and screenshots
launchpad ios metadata
launchpad ios screenshots --overwrite

# Submit for review
launchpad ios submit

# Auto-generate release notes from git commits
launchpad release-notes --ios --locale all
```

### ASO (App Store Optimization) built in

```bash
# Keyword rank checker (free, no auth required)
launchpad ios aso keyword-rank \
  --bundle-id com.example.app \
  --keywords "voice memo,transcription,recorder" \
  --country us

# Output:
# 🟢 #3   voice memo
# 🟡 #24  recorder
# 🔴 #87  transcription

# Metadata quality check (character limits, keyword duplication)
launchpad ios aso metadata-check

# Review sentiment analysis
launchpad ios aso review-summary --bundle-id com.example.app --limit 200
```

### Android support

```bash
launchpad android build --release
launchpad android upload --aab app-release.aab
launchpad android promote --from internal --to production
launchpad android aso metadata-check
```

---

## Setup in 3 steps

```bash
# Install (universal binary, Apple Silicon + Intel)
curl -fsSL https://raw.githubusercontent.com/entaku0818/launchpad/main/install.sh | sh

# Initialize your project
launchpad init    # generates .launchpadrc and .env.template

# Verify your setup
launchpad doctor
```

Fill in `.env` with your App Store Connect API key and you're ready to go.

---

## Takeaways

By calling the App Store Connect API directly:

1. **I finally understood what fastlane was actually doing** (and why it was hard)
2. **I discovered features I never knew existed** (Win-back Offers, PPO, Server API...)
3. **I escaped Ruby environment management entirely**

If you've ever felt fastlane is too opaque or Ruby is too painful, try hitting the API yourself. Opening that black box is more satisfying than you'd expect.

👉 https://github.com/entaku0818/launchpad

---

*I maintain 12 iOS apps solo. More automation content on [X(@entaku0818)](https://x.com/entaku0818).*
