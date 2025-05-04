---
title: "Xcodeでアプリ名を言語別に設定する方法"
emoji: "🌐"
type: "tech"
topics: ["iOS", "Xcode", "Swift", "localization", "ローカライズ"]
published: true
---

# この記事は？
iOSアプリをグローバル展開する際に、各言語でアプリ名を変える必要がある場合があります。例えば、日本向けには「カウントダウン」、英語圏には「CountDown」といった具合です。この記事では、Xcodeプロジェクトでアプリ名を言語別に設定する正しい方法を解説します。

## 正しい手順

### 1. InfoPlist.strings ファイルを作成する

アプリ名のローカライズには、必ず`InfoPlist.strings`ファイルを使用します。

1. Xcodeのプロジェクトナビゲーターで右クリック
2. 「New File...」を選択
3. 「Strings File」を選択
4. ファイル名を必ず「**InfoPlist.strings**」と入力（大文字小文字も正確に）
5. 「Create」をクリック

![InfoPlist.stringsファイルの作成](/images/infoplist-strings-create.png)

### 2. 言語のローカライゼーションを追加

1. 作成した`InfoPlist.strings`ファイルを選択
2. 右側のFile inspectorパネル（右サイドバーの最初のタブ）で「Localize...」ボタンをクリック
3. ダイアログが表示されたら「Localize」をクリック
4. ベース言語を選択
5. 続いてプロジェクトエディタの「Project」→「Info」タブで「Localizations」セクションを確認
6. 「+」ボタンをクリックして他の言語（日本語など）を追加

![ローカライゼーションの追加](/images/localization-add-language.png)

### 3. 各言語バージョンにアプリ名を設定

各言語バージョンの`InfoPlist.strings`ファイルに以下のように記述します：

英語版（Base または en.lproj/InfoPlist.strings）:
```
CFBundleDisplayName = "CountDown";
```

日本語版（ja.lproj/InfoPlist.strings）:
```
CFBundleDisplayName = "カウントダウン";
```

### 4. Info.plist の設定を確認

1. プロジェクトの`Info.plist`を開く
2. `CFBundleDisplayName`キーがない場合は追加
   - 右クリックして「Add Row」を選択
   - `CFBundleDisplayName`と入力
3. 値はデフォルト言語での名前を入力（ここでは "CountDown"）

![Info.plistの設定](/images/infoplist-setting.png)

### 5. プロジェクトのビルドと検証

設定後、アプリをビルドして、異なる言語設定のシミュレータまたは実機でテストします。デバイスの言語設定を変更すると、それに応じてアプリ名が変わるはずです。

## よくある間違いと注意点

- **Localizable.strings ではなく InfoPlist.strings を使用する**
  - 多くの開発者が`Localizable.strings`ファイルにアプリ名を設定してしまいますが、これでは機能しません
- **ファイル名の大文字小文字に注意**
  - 必ず「InfoPlist.strings」という正確な名前を使用してください
- **Base言語の設定も忘れずに**
  - すべての言語に対応するために、ベース言語の設定も重要です

## 実例

私の場合、カウントダウンアプリを開発した際に、最初は`Localizable.strings`ファイルに以下のように設定していました：

```
// Localizable.strings（これは間違い）
"CFBundleDisplayName" = "カウントダウン";
```

しかし、アプリをビルドしてもホーム画面のアプリ名が変わらず、原因を調査した結果、アプリ名のローカライズには`InfoPlist.strings`を使う必要があることが分かりました。

正しい設定：
```
// InfoPlist.strings（これが正解）
CFBundleDisplayName = "カウントダウン";
```

この修正後、日本語環境では「カウントダウン」、英語環境では「CountDown」と正しく表示されるようになりました。

## まとめ

Xcodeでアプリ名を言語別に設定するには、`InfoPlist.strings`ファイルを使用するのが正しい方法です。`Localizable.strings`ファイルではアプリ名のローカライズはできないので注意しましょう。

この設定により、ユーザーのデバイス言語設定に応じて、適切な言語のアプリ名が表示されるようになります。

## 参考資料
- [Apple Developer Documentation: Localization](https://developer.apple.com/documentation/xcode/localization)
- [Human Interface Guidelines: App Icons](https://developer.apple.com/design/human-interface-guidelines/app-icons) 