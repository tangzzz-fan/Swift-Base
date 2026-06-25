# Swift Concurrency 深度指南: Task, TaskGroup 与多线程模型

本文档深入探讨 Swift 5.5+ 引入的现代化并发模型，重点关注 `Task`, `TaskGroup` 的原理与应用，以及它们如何改变了我们处理多线程的方式。

## 1. 核心概念：结构化并发 (Structured Concurrency)

在 GCD (Grand Central Dispatch) 时代，我们习惯于“发射后不管” (Fire-and-forget)，例如 `DispatchQueue.global().async`。这导致了回调地狱、错误处理困难以及生命周期管理混乱。

Swift Concurrency 引入了**结构化并发**：
*   **父子关系**: 异步任务具有明确的层级关系。父任务取消，子任务自动取消。
*   **错误传播**: 异步操作抛出的错误会自动向上传播，就像同步代码一样。
*   **执行流**: 代码看起来是同步顺序执行的，但实际上在 `await` 处挂起。

---

## 2. Task: 异步工作的基本单元

`Task` 是异步工作的句柄。它提供了执行上下文。

### 2.1 创建 Task 的几种方式

1.  **`Task { ... }` (非结构化)**:
    *   从同步上下文进入异步上下文。
    *   继承当前上下文的优先级和 Actor 隔离（如果是 `Task.detached` 则不继承）。
    *   **生命周期**: 需要手动管理（返回 `Task` 实例，可调用 `.cancel()`）。

2.  **`async let` (结构化)**:
    *   最简单的并发方式。
    *   **场景**: 同时发起两个请求，然后等待结果。

    ```swift
    func loadUserProfile() async throws -> Profile {
        // 两个任务并行开始
        async let avatar = fetchAvatar()
        async let stats = fetchStats()
        
        // 在这里挂起，等待两者都完成
        return Profile(avatar: try await avatar, stats: try await stats)
    }
    ```

### 2.2 线程模型与协作式线程池

Swift Concurrency 不再像 GCD 那样为每个队列创建线程。它维护一个**宽度受限的线程池** (Width-limited thread pool)，线程数量大致等于 CPU 核心数。

*   **挂起 (Suspend)**: 当代码遇到 `await` 且需要等待时，当前线程**不会阻塞**，而是释放出来去执行其他 Task。
*   **恢复 (Resume)**: 当等待的操作完成，Task 会被调度回线程池继续执行（不一定在原来的线程）。
*   **优势**: 避免了“线程爆炸” (Thread Explosion) 和过多的上下文切换开销。

---

## 3. TaskGroup: 动态并行执行

当任务数量不确定时（例如：下载一个数组中的所有图片），`async let` 就不够用了。这时需要 `TaskGroup`。

### 3.1 使用 `withThrowingTaskGroup`

```swift
func fetchAllImages(urls: [URL]) async throws -> [UIImage] {
    // 创建一个 TaskGroup，子任务返回 UIImage
    return try await withThrowingTaskGroup(of: UIImage.self) { group in 
        var images: [UIImage] = []
        images.reserveCapacity(urls.count)
        
        // 1. 动态添加子任务
        for url in urls {
            group.addTask {
                return try await downloadImage(from: url)
            }
        }
        
        // 2. 收集结果 (流式处理)
        // group 也是一个 AsyncSequence
        for try await image in group {
            images.append(image)
        }
        
        return images
    }
}
```

### 3.2 关键特性
*   **并发控制**: 所有 `addTask` 的任务并行执行。
*   **自动取消**: 如果 `withThrowingTaskGroup` 的闭包抛出错误退出，组内所有未完成的子任务会被自动取消。
*   **父子约束**: `group` 对象不能逃逸出闭包，强制保证了结构化生命周期。

---

## 4. Actor 与数据竞争隔离

多线程编程最大的噩梦是数据竞争 (Data Race)。Swift 引入了 `Actor` 来从编译器层面解决这个问题。

### 4.1 Actor 模型
*   引用类型 (像 Class)。
*   **状态隔离**: 外部不能直接访问 Actor 的可变状态 (var)。
*   **串行执行**: 外部调用 Actor 的方法必须使用 `await`。Actor 内部维护一个邮箱 (Mailbox)，消息串行处理。

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]
    
    func image(for url: URL) -> UIImage? {
        return cache[url]
    }
    
    func insert(_ image: UIImage, for url: URL) {
        cache[url] = image
    }
}

// 使用
let cache = ImageCache()
// ❌ 编译错误: Actor-isolated property 'cache' can not be accessed
// cache.cache[url] = image 

// ✅ 正确: 通过 await 异步访问
await cache.insert(newImage, for: url)
```

### 4.2 MainActor
*   特殊的全局 Actor，代表主线程。
*   UI 更新必须在 `@MainActor` 上执行。
*   SwiftUI 的 `View` 和 `ViewModel` (`ObservableObject`) 通常都应该标记为 `@MainActor`，以确保状态更新发生在主线程。

---

## 5. 实际应用场景：高并发数据同步

**场景**: App 启动时需要从服务器拉取：配置、用户信息、未读消息、好友列表。

**最佳实践**:

```swift
struct AppData {
    let config: Config
    let user: User
    let messages: [Message]
}

func bootstrapApp() async throws -> AppData {
    // 使用 async let 并行发起请求
    async let configTask = api.fetchConfig()
    async let userTask = api.fetchUser()
    async let messagesTask = api.fetchMessages()
    
    // 可以在这里做一些不需要等待网络的事情...
    print("Requests sent...")
    
    // 统一等待结果
    // 任何一个失败都会抛出错误，并取消其他正在进行的请求
    return try await AppData(
        config: configTask,
        user: userTask,
        messages: messagesTask
    )
}
```

**总结**: Swift Concurrency 通过编译器检查和运行时优化，让编写安全、高效的并发代码变得像写同步代码一样简单。掌握 `Task` (异步上下文), `TaskGroup` (动态并行), 和 `Actor` (状态隔离) 是现代 Swift 开发的必备技能。
