---
title: "SwiftのEquatable, Comparable, Identifiableプロトコル完全解説"
emoji: "📚"
type: "tech"
topics: ["Swift", "iOS", "Protocol"]
published: true
---

# この記事は？
Swiftの基本的なプロトコルである`Equatable`、`Comparable`、`Identifiable`について、実践的なコード例を交えて解説します。これらのプロトコルを適切に使用することで、より柔軟で保守性の高いコードを書くことができます。

## サンプルコード：本の管理システム

この記事では、本を管理するシステムを例に、各プロトコルの使い方と利点を説明します。以下のような本を表す構造体を考えてみましょう：

```swift
public struct Book: Equatable, Comparable, Hashable, Identifiable {
    public let id: UUID
    public let title: String
    public let author: String
    public let publishedYear: Int
    public let price: Double
    
    public init(id: UUID = UUID(), title: String, author: String, publishedYear: Int, price: Double) {
        self.id = id
        self.title = title
        self.author = author
        self.publishedYear = publishedYear
        self.price = price
    }
}
```

## Equatableプロトコル

### Equatableとは？
`Equatable`プロトコルは、2つのインスタンスが等しいかどうかを比較するための機能を提供します。このプロトコルを採用することで、`==`演算子と`!=`演算子を使用できるようになります。

Swiftの多くの基本型（`String`、`Int`、`Double`など）は、デフォルトで`Equatable`に準拠しています。カスタム型で`Equatable`を採用する場合、その型のどのプロパティを比較に使用するかを自由に定義できます。

### 実装方法

`Equatable`プロトコルを採用するには、`==`演算子を実装する必要があります：

```swift
// Equatableの明示的な実装
public static func == (lhs: Book, rhs: Book) -> Bool {
    return lhs.title == rhs.title &&
           lhs.author == rhs.author &&
           lhs.publishedYear == rhs.publishedYear &&
           lhs.price == rhs.price
}
```

この実装では、本のタイトル、著者、出版年、価格が全て一致する場合に、2つの本が等しいと判断します。注目すべき点として、`id`プロパティは比較に含めていません。これは、同じ内容の本は同一とみなすというロジックの判断によるものです。

### Equatableのメリット

1. **配列での検索が簡単に**
```swift
let books = [book1, book2, book3]
// containsメソッドが使える
if books.contains(searchBook) {
    print("本が見つかりました")
}

// firstIndexメソッドで位置を特定
if let index = books.firstIndex(of: searchBook) {
    print("本は\(index)番目にあります")
}

// filterで条件に合う本を抽出
let matchingBooks = books.filter { $0 == searchBook }
```

2. **重複の排除**
```swift
// Setを使用して重複を排除（HashableプロトコルもEquatableを継承）
let uniqueBooks = Array(Set(books))

// 重複を確認
let hasDuplicates = books.count != Set(books).count
```

3. **コレクションでの活用**
```swift
// 辞書のキーとして使用
var bookStatus = [Book: String]()
bookStatus[book1] = "Available"
bookStatus[book2] = "On Loan"

// 集合演算
let collection1: Set<Book> = [book1, book2]
let collection2: Set<Book> = [book2, book3]
let commonBooks = collection1.intersection(collection2) // 共通の本
let allBooks = collection1.union(collection2) // すべての本（重複なし）
```

## Comparableプロトコル

### Comparableとは？
`Comparable`プロトコルは、`Equatable`を継承し、さらに要素間の順序関係を定義します。このプロトコルを採用することで、`<`、`<=`、`>`、`>=`などの比較演算子を使用できるようになります。

これにより、要素の並び替えや範囲指定による検索が可能になります。また、`min()`や`max()`などのメソッドも使用できるようになります。

### 実装方法

`Comparable`プロトコルを採用するには、`<`演算子を実装する必要があります：

```swift
// タイトルのアルファベット順で比較
public static func < (lhs: Book, rhs: Book) -> Bool {
    // localizedStandardCompareを使用することで、
    // 言語や地域に応じた適切な並び順を実現
    return lhs.title.localizedStandardCompare(rhs.title) == .orderedAscending
}
```

この実装では、本のタイトルのアルファベット順で比較を行います。`localizedStandardCompare`を使用することで、異なる言語や文字体系でも適切な並び順を実現できます。

### Comparableのメリット

1. **ソートが簡単に**
```swift
// 基本的なソート
let sortedBooks = books.sorted()

// 逆順でのソート
let reverseSortedBooks = books.sorted(by: >)

// 最小値と最大値の取得
let firstBook = books.min() // アルファベット順で最初の本
let lastBook = books.max()  // アルファベット順で最後の本
```

2. **比較演算子の使用**
```swift
// 様々な比較が可能
if book1 < book2 {
    print("\(book1.title)は\(book2.title)より前にあります")
}

// 複数の比較を組み合わせる
if book1 <= book2 && book2 < book3 {
    print("book1 <= book2 < book3 の順序関係が成り立ちます")
}
```

3. **範囲での検索と操作**
```swift
// 範囲を使った検索
let range = book1...book2  // 閉範囲
let booksInRange = books.filter { range.contains($0) }

// 範囲外の要素を取得
let booksOutsideRange = books.filter { !range.contains($0) }

// 部分範囲での操作
let partialRange = book1..<book2  // 半開範囲
let booksInPartialRange = books.filter { partialRange.contains($0) }
```

4. **高度なソート操作**
```swift
// 複数の条件でのソート
let complexSortedBooks = books.sorted { book1, book2 in
    if book1.publishedYear != book2.publishedYear {
        return book1.publishedYear > book2.publishedYear // 出版年の新しい順
    }
    return book1 < book2 // 同じ出版年ならタイトルのアルファベット順
}
```

## Identifiableプロトコル

### Identifiableとは？
`Identifiable`プロトコルは、各インスタンスに一意の識別子を提供するためのプロトコルです。このプロトコルは特にSwiftUIで重要な役割を果たしますが、データ管理全般で有用です。

プロトコルの要件は単純で、`id`という名前の一意の識別子を持つことだけです。この識別子の型は`Hashable`プロトコルに準拠している必要があります。

### 実装方法

```swift
// Identifiableプロトコルは`id`プロパティを要求
public let id: UUID

// または、カスタムIDを使用することも可能
// public let id: String
// public let id: Int
```

UUIDを使用する利点：
- グローバルで一意性が保証される
- 衝突の可能性が極めて低い
- 分散システムでも安全に使用できる

### Identifiableのメリット

1. **一意性の保証**
```swift
// 同じ内容でも異なるインスタンスとして識別可能
let book1Copy = Book(title: book1.title, author: book1.author, 
                     publishedYear: book1.publishedYear, price: book1.price)
// 同じ内容でも異なるIDを持つ
print(book1.id != book1Copy.id) // true

// SwiftUIでのリスト表示
List(books) { book in
    Text(book.title)
} // idプロパティが自動的に使用される
```

2. **IDベースの管理**
```swift
// 在庫管理システム
var bookInventory = [Book.ID: Int]() // ID -> 在庫数
bookInventory[book1.id] = 5

// 貸出管理システム
var loanStatus = [Book.ID: Date]() // ID -> 貸出日
loanStatus[book1.id] = Date()

// 複数のデータソースの関連付け
var bookReviews = [Book.ID: [Review]]() // ID -> レビューリスト
```

3. **変更追跡と状態管理**
```swift
// 変更された本のIDを記録
var modifiedBooks = Set<Book.ID>()
modifiedBooks.insert(book1.id)

// バッチ処理での使用
func updateBooks(ids: Set<Book.ID>) {
    for id in ids {
        // 特定のIDの本を更新
    }
}

// キャッシュシステム
class BookCache {
    private var cache = [Book.ID: Book]()
    
    func store(_ book: Book) {
        cache[book.id] = book
    }
    
    func retrieve(id: Book.ID) -> Book? {
        return cache[id]
    }
}
```

4. **データベース連携**
```swift
// データベースでの主キーとして使用
struct BookRecord {
    let id: Book.ID  // UUIDをプライマリーキーとして使用
    var lastModified: Date
    var isArchived: Bool
}

// 関連テーブルとの紐付け
struct BookSale {
    let bookId: Book.ID
    let saleDate: Date
    let quantity: Int
}
```

## プロトコルを組み合わせる利点

これらのプロトコルを組み合わせることで、より強力で柔軟なデータ管理が可能になります：

1. **ソート可能なコレクション管理**
```swift
// 重複を排除してソート
let sortedUniqueBooks = Set(books).sorted()

// IDベースで管理しつつ、タイトルでソート
let booksByID = Dictionary(uniqueKeysWithValues: books.map { ($0.id, $0) })
let sortedBooksByTitle = booksByID.values.sorted()

// 複雑なフィルタリングと並び替え
let filteredAndSorted = books
    .filter { $0.publishedYear >= 2020 }
    .sorted()
```

2. **高度なデータ操作**
```swift
// グループ化と並び替え
let booksByAuthor = Dictionary(grouping: books, by: { $0.author })
let sortedAuthors = booksByAuthor.keys.sorted()

// IDベースの差分管理
func findModifiedBooks(oldBooks: [Book], newBooks: [Book]) -> Set<Book.ID> {
    let oldSet = Set(oldBooks)
    let newSet = Set(newBooks)
    return Set(oldSet.symmetricDifference(newSet).map { $0.id })
}
```

3. **SwiftUIとの連携**
```swift
struct BookList: View {
    let books: [Book]
    
    var body: some View {
        List(books.sorted()) { book in // IdentifiableとComparableを活用
            BookRow(book: book)
        }
    }
}
```

## テストの重要性

プロトコルの実装が正しく機能することを確認するために、包括的なユニットテストを書くことが重要です：

```swift
class BookTests: XCTestCase {
    var book1: Book!
    var book2: Book!
    var book3: Book!
    
    override func setUp() {
        book1 = Book(title: "Swift Programming", author: "John Doe", publishedYear: 2024, price: 29.99)
        book2 = Book(title: "Swift Programming", author: "John Doe", publishedYear: 2024, price: 29.99)
        book3 = Book(title: "Advanced Swift", author: "Jane Smith", publishedYear: 2024, price: 39.99)
    }
    
    // Equatableのテスト
    func testEquatableForValueComparison() {
        // 同じ値を持つ本は等しい
        XCTAssertEqual(book1, book2)
        // 異なる値を持つ本は等しくない
        XCTAssertNotEqual(book1, book3)
        
        // コレクションでの使用
        let books = [book1, book2, book3]
        XCTAssertTrue(books.contains(book1))
        XCTAssertEqual(books.filter { $0 == book1 }.count, 2)
    }
    
    // Comparableのテスト
    func testComparableForSorting() {
        // ソートのテスト
        let books = [book3, book1, book2]
        let sorted = books.sorted()
        XCTAssertEqual(sorted.first, book3) // "Advanced Swift"が最初
        
        // 範囲のテスト
        let range = book3...book1
        XCTAssertTrue(range.contains(book2))
    }
    
    // Identifiableのテスト
    func testIdentifiableUniqueness() {
        // 同じ内容でも異なるIDを持つ
        let book1Copy = Book(title: book1.title, author: book1.author,
                            publishedYear: book1.publishedYear, price: book1.price)
        XCTAssertNotEqual(book1.id, book1Copy.id)
        
        // IDベースの管理
        var inventory = [Book.ID: Int]()
        inventory[book1.id] = 5
        XCTAssertEqual(inventory[book1.id], 5)
    }
}
```

## まとめ

Swiftの基本プロトコルを適切に実装することで、以下のような利点が得られます：

- `Equatable`: 
  - 値の比較が可能になり、検索や重複排除が簡単に
  - コレクション型での使用が容易に
  - 等価性の定義を明確に表現可能

- `Comparable`: 
  - 要素の順序付けが可能に
  - ソートや範囲検索が簡単に
  - 比較演算子による直感的な操作

- `Identifiable`: 
  - 一意の識別子による安全なデータ管理
  - SwiftUIとの親和性が高い
  - 複雑なデータ関係の管理が容易に

これらのプロトコルは、特にSwiftUIなどのモダンなフレームワークでよく使用されます。適切に実装することで、より保守性が高く、バグの少ないコードを書くことができます。

## 参考資料
- [Basic Behaviors - Apple Developer Documentation](https://developer.apple.com/documentation/swift/basic-behaviors)
