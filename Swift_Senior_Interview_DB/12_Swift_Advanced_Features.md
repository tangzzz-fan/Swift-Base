# 12. Swift Advanced Features (The "Bleeding Edge" Edition)

> **Focus**: Macros, C++ Interop, Result Builders, Distributed Actors, Swift 6.
> **Difficulty**: Senior / Architect.

---

## 1. Metaprogramming (Macros & Result Builders)

### Q1: Swift Macros (Freestanding vs Attached)
**Q**: 解释 Freestanding Macro (`#predicate`) 和 Attached Macro (`@Observable`) 的区别。
*   **Follow-up**: 宏展开是在编译的哪个阶段进行的？(AST 解析后，类型检查前)。
*   **Answer**:
    *   **Freestanding**: 独立出现，产生一个表达式或声明。以 `#` 开头。
    *   **Attached**: 附加在某个声明上，修改或增强该声明。以 `@` 开头。
    *   **Sandbox**: 宏运行在独立的沙盒进程中，不能读取文件系统或网络。

### Q2: Result Builders Internals
**Q**: `buildPartialBlock` (Swift 5.7) 相比旧的 `buildBlock` 有什么优势？
*   **Follow-up**: 为什么 `ViewBuilder` 不再受 10 个子 View 的限制了？
*   **Answer**:
    *   `buildBlock` 需要为 1...N 个参数定义 N 个重载函数。
    *   `buildPartialBlock` 采用递归归约的方式 (Accumulator)。`first` -> `accumulated + next` -> ...
    *   支持任意数量的子元素，不再需要硬编码重载。

### Q3: Dynamic Member Lookup
**Q**: `@dynamicMemberLookup` 配合 `KeyPath` 有什么神奇用法？
*   **Follow-up**: 如何实现一个类型安全的 JSON 包装器，支持 `json.user.name` 语法？
*   **Answer**:
    *   实现 `subscript(dynamicMember keyPath: KeyPath<T, U>) -> U`。
    *   允许将包装类型的属性访问转发给内部类型，保持类型安全。

---

## 2. Interoperability (C++ & C)

### Q4: C++ Interop (Swift 5.9+)
**Q**: Swift 如何直接调用 C++ 的 `std::vector` 或 `std::map`？
*   **Follow-up**: 什么是 "Foreign Reference Type"？Swift 如何管理 C++ 对象的生命周期？
*   **Answer**:
    *   开启 `-cxx-interoperability-mode=default`。
    *   Swift 编译器能理解 C++ 头文件，生成对应的 Swift 代理类型。
    *   **Foreign Reference Type**: 映射为 Swift 的引用类型 (Class)，但由 C++ 负责内存管理（通常需要手动 retain/release 映射）。

### Q5: UnsafeBufferPointer & C Arrays
**Q**: 如何高效地将 Swift 数组传递给接受 `const float*` 的 C API，而不发生拷贝？
*   **Follow-up**: `Array` 的 `withUnsafeBufferPointer` 是零拷贝吗？
*   **Answer**:
    *   是的，是零拷贝。它直接暴露 Array 内部存储的指针。
    *   **Trap**: 不要将指针 `return` 出来。

---

## 3. Advanced Concurrency (Actors)

### Q6: Custom Executors
**Q**: 如何为 Actor 指定自定义的 Executor (例如绑定到特定的 RunLoop 或线程)？
*   **Follow-up**: `MainActor` 的底层实现是怎样的？
*   **Answer**:
    *   遵循 `Actor` 协议并实现 `unownedExecutor` 属性。
    *   `MainActor` 使用 `MainActor.shared.unownedExecutor`，底层绑定到 `DispatchQueue.main`。

### Q7: Distributed Actors
**Q**: Distributed Actor 和普通 Actor 有什么区别？
*   **Follow-up**: 什么是 `Location Transparency`？如果网络断开，调用会抛出什么错误？
*   **Answer**:
    *   Distributed Actor 可以跨进程、跨节点（网络）调用。
    *   方法必须是 `distributed func`，参数必须遵循 `Codable`。
    *   调用必须 `try await`，因为网络可能失败 (`DistributedActorSystemError`)。

### Q8: AsyncSequence & AsyncStream
**Q**: `AsyncStream` 的缓冲策略 (Buffering Policy) 有哪些？
*   **Follow-up**: 如何处理 `AsyncStream` 的背压 (Backpressure)？(它其实不支持背压，只支持缓冲或丢弃)。
*   **Answer**:
    *   `unbounded`: 无限缓冲（内存风险）。
    *   `bufferingOldest/Newest(limit)`: 有限缓冲，丢弃策略。
    *   **No Backpressure**: 生产者发送速度快于消费者时，只能丢弃或爆内存。

### Q9: Task Local Values
**Q**: `TaskLocal` 是如何跨越 `Task` 边界传递的？
*   **Follow-up**: 为什么 `TaskLocal` 只能是静态属性？
*   **Answer**:
    *   存储在 Task 的内部结构中。创建子 Task 时，会继承父 Task 的 Locals（Copy-on-Write 机制）。
    *   静态属性是为了作为 Key 使用，保证唯一性。

---

## 4. String & Collection Internals

### Q10: String Breadcrumbs
**Q**: Swift String 如何优化长字符串的随机访问 (Random Access)？
*   **Follow-up**: 为什么 `String` 不是 `RandomAccessCollection`？
*   **Answer**:
    *   **Breadcrumbs**: 对于超长 UTF-8 字符串，Swift 内部会缓存一些"面包屑"（索引标记），加速 `index(_:offsetBy:)` 的计算。
    *   依然不是 O(1)，所以不是 RandomAccess。

### Q11: Collection Protocols
**Q**: 实现一个自定义 `Collection` 需要最少实现哪些成员？
*   **Follow-up**: `Sequence` 和 `Collection` 的核心区别是什么？(多遍遍历 vs 单遍遍历)。
*   **Answer**:
    *   `startIndex`, `endIndex`, `subscript(position)`, `index(after:)`.
    *   **Sequence**: 可能是消耗性的 (Destructive)，如网络流，只能遍历一次。
    *   **Collection**: 必须支持多次遍历 (Non-destructive)，索引必须是稳定的。

---

## 5. Swift 6 & Future

### Q12: Typed Throws
**Q**: Swift 6 的 `throws(MyError)` 有什么好处？
*   **Follow-up**: 它如何影响泛型代码？`rethrows` 还有用吗？
*   **Answer**:
    *   **好处**: 明确错误类型，catch 时不需要 `as?` 转换。
    *   **泛型**: `func foo<E: Error>(block: () throws(E) -> Void) throws(E)`。允许精确传递错误类型。
    *   `rethrows` 可以被 `throws(E)` 替代。

### Q13: Noncopyable Types (MoveOnly)
**Q**: `~Copyable` (抑制 Copyable) 的作用？
*   **Follow-up**: `consuming` 和 `borrowing` 关键字在函数参数中代表什么？
*   **Answer**:
    *   创建唯一所有权类型 (Unique Ownership)，如 `FileHandle`, `Lock`。不能被拷贝，只能被移动 (Move)。
    *   **consuming**: 转移所有权给函数，调用者不能再使用。
    *   **borrowing**: 借用，函数结束归还所有权。

### Q14: Embedded Swift
**Q**: Embedded Swift 裁剪了哪些特性以适应微控制器？
*   **Follow-up**: 为什么 Embedded Swift 不支持 Reflection 和 Existentials？
*   **Answer**:
    *   裁剪了 Runtime (Metadata, Reflection), Heap Allocation (可选), Existentials (动态分发)。
    *   为了减小二进制体积 (Code Size) 和内存占用，使其能运行在几十 KB 内存的芯片上。

### Q15: Variadic Generics (Parameter Packs)
**Q**: 解释 `each T` 和 `repeat each T` 语法。
*   **Follow-up**: 如何写一个函数，接受任意数量的 Optional，并返回第一个非 nil 值？
*   **Answer**:
    *   `each T`: 类型包 (Type Pack)。
    *   `repeat`: 展开操作。
    *   允许编写 `func zip<each T>(_ arrays: repeat [each T])`，支持任意数量参数，不再需要生成 10 个重载。
