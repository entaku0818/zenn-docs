---
title: "SwiftのSendableプロトコル完全ガイド：並行処理で安全にデータを共有する方法"
emoji: "🔒"
type: "tech"
topics: ["swift", "ios", "concurrency", "sendable", "actor"]
published: false
---

# この記事は？

Swift 5.5で導入されたSendableプロトコルは、並行処理において型安全性を保証する重要な仕組みです。この記事では、Sendableプロトコルの基本概念から実践的な使用方法まで、具体的なコード例とテストを交えて詳しく解説します。

## 対象読者
- Swift Concurrencyの基本を理解している方
- 並行処理でのデータ共有に関する課題を抱えている方
- 型安全な並行プログラミングを学びたい方

## Sendableプロトコルとは

Sendableプロトコルは、型が並行処理のコンテキスト間で安全に共有できることを示すマーカープロトコルです。Swift Concurrencyにおいて、データ競合を防ぐための重要な仕組みとなっています。

```swift
public protocol Sendable {}
```

## 自動的にSendable準拠する型

### 値型（構造体・列挙型）

値型は基本的に自動でSendable準拠となります：

```swift
// 自動的にSendable準拠
public struct ImmutableMessage: Sendable {
    public let id: UUID
    public let content: String
    public let timestamp: Date
    
    public init(id: UUID = UUID(), content: String, timestamp: Date = Date()) {
        self.id = id
        self.content = content
        self.timestamp = timestamp
    }
}

// 列挙型も自動的にSendable準拠
public enum MessagePriority: Sendable {
    case low
    case normal
    case high(deadline: Date)
}
```

### アクター

アクターは自動的にSendable準拠となります：

```swift
public actor MessageProcessor {
    private var processedCount: Int = 0
    
    public init() {}
    
    public func process(_ message: ImmutableMessage) async -> String {
        processedCount += 1
        return "Processed message [\(message.content)] - Count: \(processedCount)"
    }
    
    public func getProcessedCount() -> Int {
        processedCount
    }
}
```

## 参照型でのSendable実装

参照型（クラス）でSendableを実装する場合は、スレッドセーフティを保証する必要があります：

```swift
public final class MessageQueue: @unchecked Sendable {
    private let queue = DispatchQueue(label: "com.example.messagequeue")
    private var messages: [ImmutableMessage] = []
    
    public init() {}
    
    public func enqueue(_ message: ImmutableMessage) {
        queue.async {
            self.messages.append(message)
        }
    }
    
    public func dequeue() -> ImmutableMessage? {
        var result: ImmutableMessage?
        queue.sync {
            result = self.messages.isEmpty ? nil : self.messages.removeFirst()
        }
        return result
    }
    
    public var count: Int {
        var result = 0
        queue.sync {
            result = self.messages.count
        }
        return result
    }
}
```

:::message
`@unchecked Sendable`を使用する場合は、開発者がスレッドセーフティを保証する責任があります。DispatchQueueやNSLockなどの同期メカニズムを適切に使用してください。
:::

## 実践的な使用例

複数のSendable型を組み合わせたメッセージシステムの例：

```swift
public struct MessageWithPriority: Sendable {
    public let message: ImmutableMessage
    public let priority: MessagePriority
    
    public init(message: ImmutableMessage, priority: MessagePriority) {
        self.message = message
        self.priority = priority
    }
}

public actor MessageSystem {
    private let queue: MessageQueue
    private let processor: MessageProcessor
    
    public init() {
        self.queue = MessageQueue()
        self.processor = MessageProcessor()
    }
    
    public func sendMessage(_ content: String, priority: MessagePriority = .normal) {
        let message = ImmutableMessage(content: content)
        let priorityMessage = MessageWithPriority(message: message, priority: priority)
        handleMessage(priorityMessage)
    }
    
    private func handleMessage(_ priorityMessage: MessageWithPriority) {
        switch priorityMessage.priority {
        case .high:
            queue.enqueue(priorityMessage.message)
            Task {
                if let message = queue.dequeue() {
                    let result = await processor.process(message)
                    print(result)
                }
            }
        case .normal, .low:
            queue.enqueue(priorityMessage.message)
        }
    }
    
    public func processNextMessage() async -> String? {
        guard let message = queue.dequeue() else {
            return nil
        }
        return await processor.process(message)
    }
    
    public func getQueueCount() -> Int {
        queue.count
    }
    
    public func getProcessedCount() async -> Int {
        await processor.getProcessedCount()
    }
}
```

## 非Sendable型の問題

以下のような可変な参照型はSendableではありません：

```swift
public class MutableMessage {
    public var content: String
    
    public init(content: String) {
        self.content = content
    }
}
```

このような型を並行処理で使用しようとすると、コンパイルエラーが発生します。

## テストによる動作確認

### 値型のSendableテスト

```swift
func testImmutableMessageSendable() async {
    let message = ImmutableMessage(content: "Hello, World!")
    
    // 複数のタスクで同時にメッセージを使用
    let expectations = (0..<5).map { _ in expectation(description: "Task completion") }
    
    for i in 0..<5 {
        Task {
            // メッセージの内容を確認（読み取り専用なので安全）
            XCTAssertEqual(message.content, "Hello, World!")
            expectations[i].fulfill()
        }
    }
    
    await fulfillment(of: expectations, timeout: 1.0)
}
```

### 参照型のSendableテスト

```swift
func testMessageQueueSendable() async {
    let queue = MessageQueue()
    let message1 = ImmutableMessage(content: "First")
    let message2 = ImmutableMessage(content: "Second")
    
    // 複数のタスクで同時にキューを操作
    async let enqueueTask1: Void = {
        queue.enqueue(message1)
    }()
    
    async let enqueueTask2: Void = {
        queue.enqueue(message2)
    }()
    
    // すべてのタスクが完了するまで待機
    _ = await [enqueueTask1, enqueueTask2]
    
    // キューの状態を確認
    XCTAssertEqual(queue.count, 2)
}
```

### アクターのテスト

```swift
func testMessageProcessor() async {
    let processor = MessageProcessor()
    let message = ImmutableMessage(content: "Test Message")
    
    // 複数のメッセージを同時に処理
    async let process1 = processor.process(message)
    async let process2 = processor.process(message)
    
    let results = await [process1, process2]
    
    // 処理結果の確認
    XCTAssertTrue(results[0].contains("Test Message"))
    XCTAssertTrue(results[1].contains("Test Message"))
    
    // 処理カウントの確認
    let count = await processor.getProcessedCount()
    XCTAssertEqual(count, 2)
}
```

## ベストプラクティス

### 1. 値型を優先する

可能な限り値型（struct、enum）を使用し、自動的なSendable準拠を活用しましょう。

### 2. 不変性を保つ

プロパティは`let`で宣言し、不変性を保つことでSendable準拠を簡単にします。

### 3. アクターを活用する

状態を持つ必要がある場合は、アクターを使用してスレッドセーフティを保証しましょう。

### 4. @unchecked Sendableは慎重に

`@unchecked Sendable`を使用する場合は、適切な同期メカニズムを実装してください。

## まとめ

Sendableプロトコルは、Swift Concurrencyにおいて型安全な並行処理を実現するための重要な仕組みです。

**重要なポイント：**
- 値型は自動的にSendable準拠
- アクターも自動的にSendable準拠
- 参照型では明示的な実装が必要
- `@unchecked Sendable`使用時は開発者がスレッドセーフティを保証
- テストによる動作確認が重要

適切にSendableプロトコルを活用することで、データ競合のない安全な並行プログラムを作成できます。

## 参考リンク

- [Swift Evolution SE-0302: Sendable and @Sendable closures](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [Swift Documentation: Sendable](https://developer.apple.com/documentation/swift/sendable)
- [Swift Concurrency Documentation](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html) 