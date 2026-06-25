# 15. Swift Concurrency & Actors (The "Modern Threading" Edition)

> **Focus**: Actors, Task, Sendable, Structured Concurrency, Executors.
> **Difficulty**: Expert.

---

## 1. Actor Fundamentals

### Q1: Actor Isolation
**Q**: 什么是 Actor Isolation？编译器如何保证外部无法直接访问 Actor 的可变状态？
*   **Follow-up**: `nonisolated` 关键字的作用？什么情况下需要使用它？
*   **Answer**:
    *   Actor 内部的状态（属性）被视为 "Isolated"。外部访问必须通过 `await`（异步消息发送），由 Actor 的 Serial Executor 调度执行。
    *   编译器在 AST 阶段检查访问权限。如果是同步访问且不在 `self` 上下文中，报错。
    *   `nonisolated`: 显式声明某个方法或属性不属于 Actor 的隔离域。通常用于遵循协议（如 `CustomStringConvertible`）或访问不可变常量 (`let`)。

### Q2: Actor Reentrancy (The Trap)
**Q**: 再次强调 Actor 重入性。为什么 Swift Actor 选择支持重入？这与传统的锁（Mutex）有什么区别？
*   **Follow-up**: 给出一个具体的 "Check-Suspend-Act" 模式导致的 Bug 例子。
*   **Answer**:
    *   **Reentrancy**: 当 Actor 挂起 (`await`) 时，释放执行权，允许处理邮箱中的其他消息。
    *   **Why**: 防止死锁 (Deadlock)。如果 Actor A 等待 Actor B，Actor B 又回调 Actor A，非重入锁会死锁。
    *   **Mutex**: 传统的锁在持有期间阻塞线程或拒绝其他访问，不支持重入（或者支持同一个线程重入，但 Actor 是跨线程的）。

### Q3: MainActor
**Q**: `@MainActor` 是什么？它和 `DispatchQueue.main` 有什么关系？
*   **Follow-up**: 如何强制一个普通的函数在 MainActor 上执行？(`MainActor.run` vs `@MainActor` closure)。
*   **Answer**:
    *   它是 Global Actor 的一种。
    *   它的 Executor 包装了 `DispatchQueue.main`。所有标记为 `@MainActor` 的代码都保证在主线程执行。
    *   `await MainActor.run { ... }`。

### Q4: Global Actors
**Q**: 如何定义一个自定义的 Global Actor？
*   **Follow-up**: Global Actor 适用于什么场景？(单例服务，共享状态)。
*   **Answer**:
    *   `@globalActor actor MyActor { static let shared = MyActor() }`.
    *   使用 `@MyActor` 标注类或函数。它们共享同一个串行执行队列。

---

## 2. Tasks & Structured Concurrency

### Q5: Task Hierarchy
**Q**: 什么是结构化并发 (Structured Concurrency)？`Task` 之间的父子关系体现在哪里？
*   **Follow-up**: 如果父 Task 被取消 (Cancelled)，子 Task 会怎样？
*   **Answer**:
    *   `async let` 和 `TaskGroup` 创建子 Task。
    *   子 Task 的生命周期被绑定在父 Task 的作用域内。父 Task 必须等待所有子 Task 结束。
    *   **Cancellation**: 自动传播。父 Task 取消，所有子 Task 都会收到取消信号 (`Task.isCancelled`)。

### Q6: Detached Tasks
**Q**: `Task.detached` 和 `Task.init` (或 `Task { }`) 的区别？
*   **Follow-up**: 什么时候应该使用 Detached Task？
*   **Answer**:
    *   `Task { }`: 继承当前上下文 (Actor, Priority, Local Values)。
    *   `Task.detached`: 不继承上下文。完全独立。
    *   **Use case**: 启动一个完全不相关的后台任务（如日志上传），不希望阻碍当前 UI 任务的优先级。

### Q7: Task Cancellation
**Q**: Swift 的任务取消是抢占式的 (Preemptive) 还是协作式的 (Cooperative)？
*   **Follow-up**: 如何正确响应取消？(`Task.checkCancellation()`)。
*   **Answer**:
    *   **Cooperative**: 任务必须主动检查取消状态。系统不会强制杀掉线程。
    *   如果任务在进行繁重的计算循环，必须定期调用 `try Task.checkCancellation()` 或判断 `Task.isCancelled`。

### Q8: Async Let vs TaskGroup
**Q**: `async let` 适用于什么场景？`TaskGroup` 呢？
*   **Follow-up**: `TaskGroup` 如何收集所有子任务的结果？
*   **Answer**:
    *   `async let`: 已知数量的并发任务 (e.g., 同时 fetch User 和 Avatar)。
    *   `TaskGroup`: 动态数量的并发任务 (e.g., 遍历 URL 数组下载)。
    *   `group.addTask { ... }`，然后 `for await result in group { ... }`。

---

## 3. Sendable & Thread Safety

### Q9: The Sendable Protocol
**Q**: `Sendable` 协议的语义是什么？
*   **Follow-up**: 为什么 `Class` 默认不是 `Sendable`？什么样的 `Class` 可以是 `Sendable`？
*   **Answer**:
    *   标记一个类型可以安全地在并发域之间传递（从一个 Actor 传给另一个，或传给 Task）。
    *   `Class` 是引用类型，多线程共享可变状态不安全。
    *   `final class` + `let` properties (Immutable) 或者内部实现了锁机制 (`@unchecked Sendable`)。

### Q10: @Sendable Closures
**Q**: 为什么传递给 `Task` 的闭包必须是 `@Sendable`？
*   **Follow-up**: `@Sendable` 闭包对捕获的变量有什么限制？
*   **Answer**:
    *   因为 Task 可能在不同的线程执行。闭包捕获的变量必须能安全地跨线程。
    *   限制：不能捕获可变的局部变量 (`var`)。捕获的值类型必须是 `Sendable`。捕获的引用类型必须是 `Sendable`。

### Q11: Strict Concurrency Checking (Swift 6)
**Q**: Swift 6 的 "Complete" 并发检查模式会带来哪些破坏性变更？
*   **Follow-up**: 如何解决 "Global variable is unsafe" 警告？
*   **Answer**:
    *   编译器会严格检查所有跨隔离域的传递。非 `Sendable` 类型传递会报错。
    *   Global Variables: 必须标记为 `@MainActor` (隔离) 或 `nonisolated(unsafe)` (如果不安全) 或改为 `let`。

---

## 4. Advanced Runtime & Executors

### Q12: Cooperative Thread Pool
**Q**: Swift Concurrency 的线程池是如何工作的？它与 GCD 的线程池有什么本质区别？
*   **Follow-up**: 为什么在 `await` 之后，代码可能在不同的线程上继续执行？
*   **Answer**:
    *   **Cooperative**: 线程数限制为 CPU 核心数。
    *   **GCD**: 阻塞时创建新线程 (Overcommit)。
    *   **Swift**: 阻塞时挂起任务 (Continuation)，线程转去执行其他任务。
    *   恢复时，任务会被调度到线程池中任意空闲的线程（除非绑定了 MainActor）。

### Q13: Continuations
**Q**: `withCheckedContinuation` 和 `withUnsafeContinuation` 的区别？
*   **Follow-up**: 如果 `resume` 被调用了两次，或者一次都没调用，会发生什么？
*   **Answer**:
    *   **Checked**: 运行时检查。如果滥用 (Double resume / Leak)，会 Crash 并打印错误信息。
    *   **Unsafe**: 不检查。性能略高。滥用导致 Undefined Behavior。
    *   **Leak**: 任务永远挂起，内存泄漏。

### Q14: Task Local Values
**Q**: `TaskLocal` 的底层实现？它与 Thread Local Storage (TLS) 的区别？
*   **Follow-up**: 为什么 `TaskLocal` 只能绑定 (bind) 不能赋值 (set)？
*   **Answer**:
    *   存储在 Task 对象中。随 Task 树传播。
    *   TLS 绑定线程，但 Swift Task 可能跨线程，所以 TLS 不适用。
    *   **Bind**: 保证结构化范围。退出作用域自动恢复。防止全局状态污染。

### Q15: Actor Hopping
**Q**: 什么是 "Actor Hopping"？
*   **Follow-up**: 当一个 Actor 调用另一个 Actor 的方法时，发生了什么？
*   **Answer**:
    *   执行流从一个 Executor 切换到另一个 Executor。
    *   当前任务挂起，参数被打包发送给目标 Actor 的邮箱。目标 Actor 调度执行。结果返回时再次 Hop 回来。

---

## 5. Integration & Patterns

### Q16: AsyncSequence
**Q**: `AsyncSequence` 和 `Publisher` (Combine) 的关系？
*   **Follow-up**: 如何将 `NotificationCenter` 转换为 `AsyncSequence`？
*   **Answer**:
    *   `AsyncSequence` 是 Pull Model (await next)。Combine 是 Push Model。
    *   `NotificationCenter.default.notifications(named: ...)`。

### Q17: AsyncStream
**Q**: 如何将基于回调的旧 API 包装成 `AsyncStream`？
*   **Follow-up**: `continuation.yield()` 和 `continuation.finish()`。
*   **Answer**:
    *   `AsyncStream { continuation in ... setup callback ... }`。
    *   在回调中调用 `yield` 发送数据，结束时 `finish`。

### Q18: Testing Concurrency
**Q**: 如何测试 Actor 代码？
*   **Follow-up**: `XCTest` 支持 `async` 测试方法吗？
*   **Answer**:
    *   支持 `func testMyActor() async throws { ... }`。
    *   可以直接 `await` Actor 方法。

### Q19: Distributed Actors (Brief)
**Q**: 分布式 Actor 需要什么传输层支持？
*   **Follow-up**: 序列化要求是什么？
*   **Answer**:
    *   需要实现 `DistributedActorSystem`。
    *   参数和返回值必须遵循 `Codable` (序列化以跨网络传输)。

### Q20: Preconcurrency
**Q**: `@preconcurrency import` 的作用？
*   **Follow-up**: 用于什么场景？
*   **Answer**:
    *   告诉编译器：这个模块还没适配 Sendable，请暂时降级检查，不要报错。
    *   用于渐进式迁移旧代码库。

---

## 6. Practical Scenarios & Best Practices

### Q21: Designing Thread-Safe Services
**Q**: [Design] 你需要设计一个 `ImageCacheService`，它在内存中缓存图片，并支持并发读写。你会如何设计？使用 Actor 还是 Lock？
*   **Answer**:
    *   **Actor Approach**: `actor ImageCache { var cache: [URL: UIImage] = [:] ... }`。
        *   *Pros*: 简单，编译器保证安全。
        *   *Cons*: 所有的读取 (`object(forKey:)`) 变成异步 (`await`)。如果业务逻辑大量依赖同步读取，改造成本高。
    *   **Lock Approach (OSAllocatedUnfairLock)**: `class ImageCache: @unchecked Sendable { private let lock = ... }`。
        *   *Pros*: 可以保持同步 API (`func image(for url: URL) -> UIImage?`)。
        *   *Cons*: 需要手动管理锁，容易出错。
    *   **Best Practice**: 如果是全新的异步架构，首选 **Actor**。如果必须兼容同步代码或追求极致的读取性能（纳秒级），使用 **Lock** 并标记 `@unchecked Sendable`。

### Q22: Handling "Sendable" Compiler Errors
**Q**: [Error Handling] 编译器报错 "Type 'MyCustomClass' does not conform to protocol 'Sendable'"，但你确定这个类只在主线程使用。你应该怎么做？
*   **Answer**:
    *   **Option 1**: 标记为 `@MainActor`。这是最推荐的做法。这样它就隐式遵循了 `Sendable`（因为只能在主线程传递）。
    *   **Option 2**: 如果它确实是不可变的，改为 `final class` 并让属性都是 `let`。
    *   **Option 3**: 如果是遗留代码且无法修改架构，使用 `@unchecked Sendable`，但必须添加注释说明原因（"Safe because managed by external lock" 或 "Only used on Main Thread"）。**慎用**。

### Q23: Concurrency Pitfalls
**Q**: [Pitfall] 在 Actor 方法中调用另一个异步函数（如网络请求）时，为什么修改 Actor 自己的状态是危险的？(The "Suspension Point" Trap)。
*   **Scenario**:
    ```swift
    actor User {
        var posts: [Post] = []
        func loadPosts() async {
            if !posts.isEmpty { return } // Check
            let newPosts = await fetchFromAPI() // Suspend
            posts = newPosts // Act
        }
    }
    ```
*   **Answer**:
    *   **Problem**: 在 `await fetchFromAPI()` 挂起期间，Actor 锁被释放。其他线程可能调用 `loadPosts()`。
    *   **Result**: 两个线程都通过了 `isEmpty` 检查，都发起了 API 请求。当第一个返回并写入 `posts` 后，第二个请求返回并**覆盖**了 `posts`。
    *   **Fix**: 在 `await` 返回后，**再次检查**状态 (`if !posts.isEmpty { return }`)，或者使用 Task 状态跟踪 (`var loadingTask: Task<[Post], Error>?`) 来合并请求。
