# 使用 Actor 实现 LRU Cache 与并发图片加载系统

本文将展示如何使用 Swift Concurrency (`actor`) 来构建一个线程安全的 LRU (Least Recently Used) 缓存，并基于此实现一个支持并发去重（Task Deduplication）的图片加载器。

## 1. LRU Cache 实现

LRU 缓存的核心数据结构通常是 **哈希表 (Dictionary)** + **双向链表 (Doubly Linked List)**。
*   **Dictionary**: 提供 O(1) 的数据查找。
*   **Doubly Linked List**: 维护数据的访问顺序。最近访问的移动到头部，最久未访问的在尾部。当缓存满时，移除尾部节点。

由于 Swift 的 `struct` 是值类型，不适合做链表节点，因此我们需要使用 `class` 来定义节点。而整个 Cache 的状态管理则交给 `actor` 来保证线程安全。

### 1.1 链表节点定义

```swift
final class Node<Key, Value> {
    let key: Key
    var value: Value
    var next: Node?
    weak var previous: Node?
    
    init(key: Key, value: Value) {
        self.key = key
        self.value = value
    }
}
```

### 1.2 Actor LRU Cache

```swift
actor LRUCache<Key: Hashable, Value> {
    private let capacity: Int
    private var dictionary: [Key: Node<Key, Value>] = [:]
    
    // 链表头尾指针
    private var head: Node<Key, Value>?
    private var tail: Node<Key, Value>?
    
    init(capacity: Int) {
        self.capacity = max(1, capacity)
    }
    
    // MARK: - Public API
    
    func setValue(_ value: Value, for key: Key) {
        if let node = dictionary[key] {
            // 1. 如果 key 已存在，更新值并移动到头部
            node.value = value
            moveToHead(node)
        } else {
            // 2. 如果 key 不存在，创建新节点
            let newNode = Node(key: key, value: value)
            dictionary[key] = newNode
            addToHead(newNode)
            
            // 3. 检查容量，如果超出则移除尾部
            if dictionary.count > capacity {
                removeLast()
            }
        }
    }
    
    func value(for key: Key) -> Value? {
        guard let node = dictionary[key] else { return nil }
        // 访问命中，移动到头部
        moveToHead(node)
        return node.value
    }
    
    // MARK: - Private Helper Methods (Linked List Operations)
    
    private func addToHead(_ node: Node<Key, Value>) {
        if let currentHead = head {
            node.next = currentHead
            currentHead.previous = node
            head = node
        } else {
            head = node
            tail = node
        }
    }
    
    private func moveToHead(_ node: Node<Key, Value>) {
        // 如果已经是头部，无需操作
        guard node !== head else { return }
        
        // 断开当前位置
        let prev = node.previous
        let next = node.next
        
        prev?.next = next
        next?.previous = prev
        
        if node === tail {
            tail = prev
        }
        
        // 插入头部
        node.next = head
        node.previous = nil
        head?.previous = node
        head = node
    }
    
    private func removeLast() {
        guard let last = tail else { return }
        
        // 从字典移除
        dictionary.removeValue(forKey: last.key)
        
        // 从链表移除
        if let prev = last.previous {
            prev.next = nil
            tail = prev
        } else {
            // 只有这一个节点
            head = nil
            tail = nil
        }
    }
}
```

---

## 2. 并发图片加载器 (Image Loader)

接下来我们实现一个 `ImageLoader`。它需要解决两个主要问题：
1.  **缓存**: 使用上面的 `LRUCache` 避免重复下载。
2.  **任务去重 (Task Deduplication)**: 如果同时有 10 个请求请求同一张图片，不应该发起 10 次网络请求，而应该复用同一个正在进行的任务。

### 2.1 ImageLoader 实现

```swift
import UIKit

actor ImageLoader {
    // 1. 内存缓存
    private let cache = LRUCache<URL, UIImage>(capacity: 100)
    
    // 2. 正在进行的任务字典 (用于去重)
    private var activeTasks: [URL: Task<UIImage, Error>] = [:]
    
    func image(from url: URL) async throws -> UIImage {
        // A. 检查缓存
        if let cachedImage = await cache.value(for: url) {
            return cachedImage
        }
        
        // B. 检查是否有正在进行的任务
        if let existingTask = activeTasks[url] {
            // 等待现有任务完成，复用结果
            return try await existingTask.value
        }
        
        // C. 创建新任务
        let task = Task<UIImage, Error> {
            // 下载图片
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else {
                throw URLError(.cannotDecodeContentData)
            }
            return image
        }
        
        // 保存任务以便复用
        activeTasks[url] = task
        
        do {
            let image = try await task.value
            // 下载成功，存入缓存
            await cache.setValue(image, for: url)
            // 移除任务标记
            activeTasks[url] = nil
            return image
        } catch {
            // 失败也要移除任务标记
            activeTasks[url] = nil
            throw error
        }
    }
}
```

### 2.2 使用示例

```swift
let loader = ImageLoader()

// 模拟并发请求
Task {
    let url = URL(string: "https://example.com/image.png")!
    
    // 启动 3 个并发任务
    async let image1 = loader.image(from: url)
    async let image2 = loader.image(from: url)
    async let image3 = loader.image(from: url)
    
    // 实际上只会发起 1 次网络请求
    let images = try await [image1, image2, image3]
    print("Loaded \(images.count) images")
}
```

## 3. 总结

通过这个案例，我们展示了 Swift Concurrency 的强大之处：
1.  **Actor 隔离**: `LRUCache` 和 `ImageLoader` 内部的状态（字典、链表、任务列表）都是私有的，外部无法直接访问，从而在编译期保证了线程安全。
2.  **结构化并发**: 使用 `Task` 句柄轻松实现了请求去重（Coalescing），避免了传统多线程编程中复杂的锁和条件变量。
3.  **可读性**: 代码逻辑清晰，线性流动的 `await` 语法使得异步逻辑像同步代码一样易于理解。
