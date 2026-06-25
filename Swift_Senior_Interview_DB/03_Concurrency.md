# 03. Concurrency & Thread Safety

> **Difficulty**: Expert
> **Focus**: Actors, Reentrancy, Sendable, GCD vs Swift Concurrency

---

## 1. Swift Concurrency Model (Actor Model)

### Q1: 深入剖析 Swift Actor 的重入性 (Reentrancy)。它解决了什么问题？又引入了什么新问题？
**Follow-up**:
1.  为什么 Actor 设计为可重入的？(为了防止死锁)。
2.  请举一个具体的代码例子，展示 Actor 重入导致的状态不一致 (State Inconsistency) Bug。
3.  如何解决这个问题？(提示: 状态检查，或者把 mutation 逻辑放在一个同步块里)。

#### Key Answer:
*   **Reentrancy**: 当 Actor 执行一个 `await` 挂起时，它释放了锁（Executor），允许其他任务在同一个 Actor 上执行。
*   **解决的问题**: 死锁 (Deadlock)。如果 Actor 不可重入，两个 Actor 互相 `await` 对方就会死锁。
*   **引入的问题**: 逻辑竞争。在 `await` 前后，Actor 的状态可能已经被其他任务改变了。
*   **Example**:
    ```swift
    actor BankAccount {
        var balance = 100
        func withdraw(amount: Int) async {
            if balance >= amount {
                // Await 挂起，此时 balance 可能被其他任务修改！
                await verifyTransaction() 
                // 恢复执行时，balance 可能已经小于 amount 了！
                balance -= amount 
            }
        }
    }
    ```

---

## 2. Sendable & Thread Safety

### Q2: 解释 `@Sendable` 闭包和 `Sendable` 协议的编译器检查机制。
**Follow-up**:
1.  `@unchecked Sendable` 是什么？什么时候必须使用它？
2.  为什么 `lazy var` 属性会导致类无法自动推断为 `Sendable`？
3.  在 Swift 6 严格并发模式下，全局变量 (Global Variables) 会有什么限制？如何安全地使用全局变量？(提示: `MainActor`, `nonisolated(unsafe)`).

#### Key Answer:
*   **Sendable**: 标记一个类型可以安全地跨并发域（线程/Actor）传递。
*   **Compiler Check**: 编译器会检查 `Sendable` 类型的属性是否也都是 `Sendable` 的。如果是 Class，必须是 `final` 且只有不可变存储属性 (`let`)。
*   **@unchecked Sendable**: 告诉编译器 "我知道我在做什么，请闭嘴"。通常用于内部实现了锁机制（如 `os_unfair_lock`）的类，或者 C 指针包装器。
*   **Global Variables**: Swift 6 禁止可变的全局变量，除非用 `nonisolated(unsafe)` 标记或绑定到 Global Actor (如 `@MainActor`)。

---

## 3. GCD vs Swift Concurrency

### Q3: 对比 GCD 和 Swift Concurrency 的底层线程模型。
**Follow-up**:
1.  GCD 的 "Thread Explosion" (线程爆炸) 是怎么产生的？Swift Concurrency 是如何避免这个问题的？(Cooperative Thread Pool).
2.  `Task.yield()` 的作用是什么？
3.  `withCheckedContinuation` 的原理是什么？它是如何桥接基于回调的旧 API 的？

#### Key Answer:
*   **GCD**: 抢占式调度 (Preemptive)。每个 Queue 可能对应一个线程。如果大量 Block 阻塞 (Blocking Wait)，GCD 会不断创建新线程，导致上下文切换开销巨大（线程爆炸）。
*   **Swift Concurrency**: 协作式调度 (Cooperative)。底层维护一个与 CPU 核心数相当的线程池。任务挂起 (`await`) 时，线程不会阻塞，而是转去执行其他任务（Continuation Switching）。
*   **Continuation**: 保存了函数的栈帧和执行状态（寄存器等），存储在堆上。恢复时重新加载。

---

## 4. Structured Concurrency

### Q4: 什么是结构化并发 (Structured Concurrency)？它如何处理错误传播和任务取消？
**Follow-up**:
1.  `async let` 和 `TaskGroup` 的区别？
2.  如果在 `TaskGroup` 中一个子任务抛出错误，其他子任务会发生什么？
3.  `TaskLocal` 值是如何在任务树中传递的？

#### Key Answer:
*   **Structured**: 任务的生命周期绑定在作用域内。父任务必须等待所有子任务完成（或取消）才能结束。
*   **Cancellation**: 取消信号会自动向下传播。子任务需要主动检查 `Task.isCancelled`。
*   **TaskLocal**: 类似线程局部存储 (TLS)，但在异步任务链中自动传递。使用 `TaskLocal.withValue(...) { }` 绑定。
