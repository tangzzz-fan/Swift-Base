# Swift Senior Developer Interview: The "Soul-Crushing" Edition

> "You said you know Swift? Let's see if you know how it *breathes*."

这份面试题旨在考察候选人对 Swift 语言底层、编译器行为、运行时机制以及 SwiftUI 核心原理的深度理解。不包含任何"如何声明一个变量"之类的废话。

---

## 第一部分：语言特性与底层机制 (Language Features & Under the Hood)

### 1. 泛型与类型系统 (Generics & Type System)
**Q: 请详细解释 `some Protocol` (Opaque Types) 与 `any Protocol` (Existential Types) 的本质区别。**
*   **追问 1**: 为什么编译器在处理 `any Protocol` 时需要 "Existential Container"？这个容器的内存布局是怎样的？（提示：涉及 Value Buffer, VWT, PWT）。
*   **追问 2**: 为什么 Swift 5.7 之前 `Protocol` 不能作为 Equatable 的类型约束？Swift 5.7 是如何通过 "Opening Existentials" 解决这个问题的？
*   **追问 3**: 在性能敏感的代码中，你会优先选择泛型约束 (`<T: Protocol>`) 还是 Existential (`any Protocol`)？为什么？（请从静态分发 vs 动态分发、编译器优化角度回答）。

### 2. 内存管理与对象生命周期 (Memory & Lifecycle)
**Q: 解释 Copy-on-Write (CoW) 的底层实现机制。**
*   **场景**: 如果你要自己实现一个支持 CoW 的结构体（比如一个自定义的大型数据容器），你需要利用 `isKnownUniquelyReferenced` 做什么？
*   **追问**: `Weak` 引用在底层是如何实现的？Side Table（散列表）是什么时候创建的？为什么 Swift 的对象销毁过程分为 "Deinit" 和 "Dealloc" 两个阶段？

### 3. 方法分发 (Method Dispatch)
**Q: Swift 有哪几种方法分发机制？请按性能从高到低排序，并解释触发条件。**
*   **Static Dispatch (Direct)**
*   **Table Dispatch (Witness Table / Virtual Table)**
*   **Message Dispatch (ObjC Runtime)**
*   **场景**:
    *   `extension` 中定义的方法是哪种分发？
    *   `dynamic` 关键字会产生什么影响？
    *   `@objc` 会强制使用消息分发吗？（提示：不一定，取决于是否是 `dynamic` 或协议要求）。
    *   协议扩展（Protocol Extension）中的默认实现，如果被遵循者（Conformer）重写，在不同类型的变量持有下（Existential vs Concrete Type），调用行为有何不同？（这是经典的 "Static vs Dynamic dispatch in Protocols" 陷阱）。

### 4. 并发模型 (Concurrency)
**Q: 深入剖析 Swift Actor 的工作原理。**
*   **核心**: Actor 是如何保证线程安全的？它和传统的 `DispatchQueue.sync/async` 锁机制相比，最大的区别是什么？（提示：Executor, Job 调度）。
*   **Reentrancy (重入性)**: 什么是 Actor Reentrancy？它解决了什么问题（死锁），又引入了什么新问题（状态不一致）？请举例说明如何避免重入导致的数据竞争逻辑错误。
*   **Sendable**: `@Sendable` 闭包和普通闭包在编译器检查层面有什么区别？`unchecked Sendable` 什么时候使用？

---

## 第二部分：SwiftUI 深度剖析 (SwiftUI Internals)

### 1. 视图标识与生命周期 (Identity & Lifecycle)
**Q: 解释 SwiftUI 中 "Identity" 的概念（Explicit Identity vs Structural Identity）。**
*   **场景**: 为什么在 `ForEach` 中使用 `.id(\.self)` 有时是危险的？
*   **追问**: 当你使用 `AnyView` 时，对 View 的 Identity 和性能有什么毁灭性的影响？为什么 Apple 强烈建议不要使用 `AnyView`？
*   **原理**: 解释 `AttributeGraph` 是如何工作的。当一个 `@State` 改变时，SwiftUI 是如何通过 Dependency Graph 找到最小更新路径的？

### 2. 状态管理 (State Management)
**Q: `StateObject` 和 `ObservedObject` 的区别，不要只回答 "生命周期不同"。**
*   **本质**: 从 View 的 `init` 和 `body` 调用时机来解释。如果 View 被重新求值（Re-evaluated），`ObservedObject` 里的实例会发生什么？`StateObject` 又是如何确保持久性的？（提示：SwiftUI 内部的 Storage 机制）。
*   **环境**: `EnvironmentObject` 在视图层级过深时会有性能问题吗？如果有，是因为什么？

### 3. 布局系统 (Layout System)
**Q: 描述 SwiftUI 的布局协商过程 (Layout Negotiation Protocol)。**
*   **口诀**: "Parent proposes, Child chooses, Parent places."
*   **场景**: 解释为什么 `Frame(width: 100, height: 100)` 并不总是能强制 View 大小为 100x100？（提示：有些 View 是 "Stubborn" 的）。
*   **GeometryReader**: 为什么说 `GeometryReader` 破坏了正常的布局流？它会给子 View 提议什么尺寸？它自身又会占据什么尺寸？

### 4. 动画与事务 (Animations & Transactions)
**Q: SwiftUI 的动画实际上是在插值什么？**
*   **协议**: `Animatable` 协议的作用是什么？`animatableData` 属性通常是什么类型？（`VectorArithmetic`）。
*   **场景**: 如何实现一个自定义 Shape 的变形动画（例如从圆形变形成五角星）？你需要手动实现哪个属性？

---

## 第三部分：实战鞭打 (Coding Challenge)

**Q: 现场手写一段代码（伪代码即可），实现一个简单的 `Future` 类型，并将其适配为 Swift 5.5 的 `async/await` 语法。**
*   **要求**:
    1.  实现 `withCheckedContinuation` 的桥接。
    2.  处理错误抛出。
    3.  解释 `withCheckedContinuation` 和 `withUnsafeContinuation` 的区别。

---

> **总结**: 如果你能从容回答以上 80% 的问题，并能解释清楚底层的 VWT/PWT、Existential Container 内存布局以及 AttributeGraph 的更新机制，那么你不仅是 Senior，你是 Expert。
