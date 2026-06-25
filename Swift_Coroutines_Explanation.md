# Swift 中的协程 (Coroutines): 解释与使用

## 1. 什么是协程 (Coroutines)?

协程（Coroutines）是一种程序组件，它允许执行被挂起（suspend）和恢复（resume）。不同于子例程（Subroutines/Functions）一旦调用就必须执行到返回，协程可以在执行过程中保存当前的上下文（Context），将控制权交还给调用者或调度器，并在稍后的某个时刻从中断的地方继续执行。

### 核心特性
*   **挂起与恢复 (Suspend & Resume)**: 这是协程的灵魂。函数可以“暂停”自己，稍后再“醒来”。
*   **协作式多任务 (Cooperative Multitasking)**: 线程通常是抢占式（Preemptive）的，由操作系统内核调度；而协程通常是协作式的，由用户态的运行时或程序员显式控制挂起的时机。
*   **轻量级 (Lightweight)**: 创建和切换协程的开销远小于线程。一个进程可以轻松拥有数万甚至数百万个协程，但很难拥有同等数量的线程。

---

## 2. Swift Concurrency 中的协程

在 Swift 5.5 引入的 Concurrency 模型中，`async`/`await` 就是协程的语法化表达。

### 2.1 线程 vs 协程 (Thread vs Async Task)

在传统的 GCD (Grand Central Dispatch) 模型中，当我们派发一个任务到队列时，通常会占用一个线程。如果这个任务被阻塞（例如等待 I/O 或锁），这个线程就会一直被占用，无法处理其他工作。为了维持吞吐量，系统可能会创建更多的线程，导致**线程爆炸 (Thread Explosion)** 和频繁的**上下文切换 (Context Switching)**，从而降低性能。

Swift 的协程模型采用了 **M:N 线程模型**（M 个用户态任务映射到 N 个内核线程）：
*   **运行时线程池**: Swift 运行时维护一个大小通常等于 CPU 核心数的线程池。
*   **挂起不阻塞**: 当遇到 `await` 时，如果需要等待（例如网络请求），当前任务（Task）会**挂起**。
*   **释放线程**: 关键在于，挂起时，它会**释放**当前占用的线程。这个线程立刻就可以去执行线程池中排队的其他任务。
*   **恢复**: 当等待的操作完成，任务会被重新调度，可能会在同一个线程恢复，也可能在不同的线程恢复。

### 2.2 核心概念与组件

#### `async` / `await`
*   `async`: 标记一个函数或闭包是异步的，意味着它可能在执行过程中挂起。
*   `await`: 标记一个潜在的挂起（Suspension Point）。程序执行到这里可能会暂停，释放线程控制权。

```swift
func fetchUserData() async throws -> User {
    // 1. 开始执行
    let url = URL(string: "https://api.example.com/user")!
    
    // 2. 遇到 await，开始网络请求。
    // 当前函数挂起，线程被释放去干别的事。
    let (data, _) = try await URLSession.shared.data(from: url)
    
    // 3. 网络请求返回，函数恢复执行（可能在另一个线程）。
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}
```

#### `Task` (非结构化并发)
`Task` 是异步工作的基本单元。它提供了创建异步上下文的入口。
*   `Task { ... }`: 从同步代码进入异步世界的桥梁。继承当前上下文的优先级和 Actor 隔离（如果是 `Task.detached` 则不继承）。

#### `TaskGroup` & `async let` (结构化并发)
结构化并发（Structured Concurrency）强调异步任务的生命周期应该像作用域一样清晰：父任务应当等待子任务完成。

*   **`async let`**: 简单的并行执行。
    ```swift
    async let image1 = fetchImage(url1)
    async let image2 = fetchImage(url2)
    // 在这里等待两个都完成
    let (img1, img2) = await (image1, image2)
    ```

*   **`TaskGroup`**: 动态数量的并行任务。
    ```swift
    await withTaskGroup(of: UIImage.self) { group in 
        for url in urls {
            group.addTask { await fetchImage(url) }
        }
        for await image in group {
            // 处理结果
        }
    }
    ```

#### `Continuation` (桥接旧代码)
`CheckedContinuation` 和 `UnsafeContinuation` 用于将基于回调（Closure/Delegate）的旧异步代码转换为 `async`/`await` 风格。它们手动控制了协程的挂起和恢复。

```swift
func fetchImage(url: URL) async throws -> UIImage {
    return try await withCheckedThrowingContinuation { continuation in
        legacyFetchImage(url) { result in
            switch result {
            case .success(let image):
                continuation.resume(returning: image)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

---

## 3. 深入底层 (Under the Hood)

### 3.1 栈帧与堆 (Stack vs Heap)
在同步函数调用中，所有局部变量都存储在栈（Stack）上。函数返回，栈帧销毁。

在 Swift 协程中，因为函数可能会挂起并在将来恢复，它的状态（局部变量、执行进度）不能仅仅保存在栈上（因为挂起时栈可能会被其他任务复用）。
因此，Swift 编译器会将异步函数的栈帧拆分：
*   **同步部分**: 仍然使用系统栈。
*   **异步状态**: 需要跨越挂起点的状态会被存储在**堆 (Heap)** 上。这通常被称为 **Async Frame**。

当函数挂起时，只有堆上的状态被保留。当它恢复时，运行时会重新分配栈空间，并从堆中恢复状态继续执行。

### 3.2 调度器 (Executor)
Swift Concurrency 默认使用一个全局的并发线程池（Cooperative Thread Pool）。
*   **Actor**: Actor 拥有自己的串行执行器（Serial Executor），确保 Actor 内的状态修改是线程安全的。
*   **MainActor**: 特殊的 Actor，绑定到主线程，用于 UI 更新。

## 4. 总结

Swift 中的协程不仅仅是语法糖，它是一套完整的运行时机制，旨在解决传统并发编程中的线程爆炸、死锁和复杂性问题。

| 特性 | 传统多线程 (GCD/Threads) | Swift 协程 (Concurrency) |
| :--- | :--- | :--- |
| **调度方式** | 抢占式 (Preemptive) | 协作式 (Cooperative) |
| **阻塞行为** | 阻塞线程 (Block Thread) | 挂起任务 (Suspend Task)，释放线程 |
| **上下文切换** | 昂贵 (内核态切换) | 廉价 (用户态切换) |
| **内存占用** | 高 (每个线程有独立栈) | 低 (共享线程池，按需分配堆帧) |
| **代码风格** | 回调地狱 (Callback Hell) | 线性代码 (Linear Code) |

掌握协程，意味着从“管理线程”的思维转变为“管理任务与依赖”的思维。
