# 22. Swift Expert Interview Questions (The "God Mode" Edition)

> **Level**: Expert / Compiler Engineer / Architect
> **Focus**: Runtime Hacking, Compiler Optimization, Custom Concurrency, Manual Memory Management.

---

## 1. Runtime & Metadata Hacking

### Q1: Implementing a Custom Reflection System
**Q**: Swift 的 `Mirror` 性能较差且只读。如果让你实现一个高性能的、支持写入的反射系统（类似 KVC），你会如何操作 **Type Metadata**？
*   **Follow-up**:
    *   解释 `StructMetadata` 和 `ClassMetadata` 的内存布局差异。
    *   如何通过指针偏移（Pointer Arithmetic）找到结构体中第三个属性的 Field Offset？
    *   **Nominal Type Descriptor** 是什么？它在泛型特化中扮演什么角色？
*   **Answer Key**:
    *   **Metadata Layout**: 结构体 Metadata 包含 Kind, Description, Parent。类 Metadata 还包含 Superclass, Cache Data, V-Table。
    *   **Field Offset**: 在 Metadata 的后面通常紧跟着 Field Offset Vector。通过读取这个 Vector，可以获得每个属性相对于实例起始地址的偏移量。
    *   **Implementation**: `unsafeBitCast` 实例为指针 -> 读取 Metadata -> 读取 Field Offset -> `advanced(by: offset)` -> `assumingMemoryBound(to: T.self)`.

### Q2: Swizzling in Pure Swift?
**Q**: 在纯 Swift（无 ObjC 运行时）中，是否可能实现类似 Method Swizzling 的功能？
*   **Follow-up**:
    *   如何修改 **V-Table** (Virtual Method Table)？
    *   `@_dynamicReplacement(for:)` 的原理是什么？
    *   **Function Hooking**: 在 SIL 层级或汇编层级，如何通过修改指令（如 `jmp`）来实现函数拦截？
*   **Answer Key**:
    *   **V-Table Patching**: 可以找到类 Metadata 中的 V-Table 区域，将函数指针替换为自定义函数的指针。需要处理内存页保护 (mprotect)。
    *   **@_dynamicReplacement**: 编译器特性。编译器会为被修饰的函数生成一个 "Thunk"，该 Thunk 会检查是否有替换实现。如果有，跳转到替换实现；否则执行原实现。

---

## 2. Compiler Attributes & Optimization

### Q3: The ABI Boundaries
**Q**: 解释 `@inlinable`, `@usableFromInline`, `@frozen`, `@_alwaysEmitIntoClient` 的区别与联系。
*   **Scenario**: 你正在开发一个二进制 SDK (XCFramework)。如果你希望某个函数在调用者的 App 中内联执行，但该函数又依赖于 SDK 内部的一个私有结构体，你应该如何标记？
*   **Answer Key**:
    *   **@inlinable**: 允许函数体被包含在模块接口中，跨模块内联。
    *   **@usableFromInline**: 允许 internal 的属性/方法被 `@inlinable` 的代码访问，但依然对外部不可见。
    *   **@frozen**: 承诺枚举或结构体的布局未来不会改变（不再添加 case 或 stored property）。允许编译器进行更激进的布局优化。
    *   **@_alwaysEmitIntoClient**: 类似于内联，但它将函数的实现直接发射到客户端二进制中。用于向后兼容（Back Deployment），在旧 OS 上使用新 SDK 的功能。

### Q4: Specialization & Generics
**Q**: 什么是 **Generic Specialization**？如何使用 `@_specialize` 强制编译器生成特定版本的代码？
*   **Follow-up**: 为什么特化能带来巨大的性能提升？（提示：Devirtualization, Inlining）。
*   **Answer Key**:
    *   **Specialization**: 编译器为泛型函数生成针对特定类型（如 `Int`, `String`）的具体版本。
    *   **Benefit**: 移除了 Existential Container 的开销，移除了 PWT/VWT 的动态查找，允许函数内联。
    *   **@_specialize(where T == Int)**: 强制编译器生成 `Int` 版本的特化代码，即使在同一模块外。

---

## 3. Advanced Concurrency

### Q5: Custom Executors
**Q**: Swift 5.9 引入了自定义 Actor Executor。请描述如何实现一个 **DatabaseActor**，使其所有的 Task 都在一个特定的 `DispatchQueue`（Label: "com.db.serial"）上执行。
*   **Follow-up**:
    *   `UnownedSerialExecutor` 是什么？
    *   如何保证 `assumeIsolated` 的安全性？
*   **Answer Key**:
    *   **Implementation**: 让 Actor 遵循 `Actor` 协议，并实现 `nonisolated var unownedExecutor: UnownedSerialExecutor`。
    *   **DispatchSerialQueue**: 扩展 `DispatchQueue` 使其遵循 `SerialExecutor` 协议（或者使用 `asUnownedSerialExecutor()`）。
    *   **Why**: 保证数据库操作严格串行，且与遗留的 GCD 代码互操作。

### Q6: Task Local Internals
**Q**: `TaskLocal` 是如何实现的？它存储在哪里？
*   **Follow-up**: 为什么 `TaskLocal` 的写入（绑定）必须是 Scoped 的 (`$key.withValue(...)`)？
*   **Answer Key**:
    *   **Storage**: 存储在 `Task` 对象的内部结构中（类似于 Thread Local Storage，但是绑定到 Task）。
    *   **Structure**: 是一个链表或持久化映射。
    *   **Scoped**: 保证了结构化并发的完整性。子 Task 继承父 Task 的 Locals。如果允许随意 `set`，会导致状态管理的混乱和数据竞争。

---

## 4. Manual Memory Management

### Q7: ManagedBuffer & Tail Allocation
**Q**: Swift 的 `Array` 底层是如何实现的？如何使用 `ManagedBuffer` 实现一个自定义的、高性能的、支持 CoW 的数据结构？
*   **Follow-up**:
    *   **Tail Allocation**: 什么是尾部分配？为什么它比 `class Node { var next: Node? }` 更高效？
    *   **Alignment**: 如何手动计算内存对齐？
*   **Answer Key**:
    *   **ManagedBuffer**: 一个类，允许你在对象头部（Header）之后分配一块连续的内存（Elements）。
    *   **Tail Allocation**: Header 和 Elements 在同一块堆内存中。减少了一次 malloc，减少了内存碎片，提高了缓存局部性。
    *   **Implementation**: `ManagedBuffer<Header, Element>.create(...)`.

### Q8: Unsafe Pointers & Binding
**Q**: `withUnsafeBytes` 和 `withUnsafePointer` 的区别？
*   **Follow-up**:
    *   什么是 **Type Punning**？为什么 `assumingMemoryBound(to:)` 和 `bindMemory(to:)` 不能乱用？
    *   **Strict Aliasing Rule**: Swift 编译器如何利用别名规则进行优化？如果违反了会发生什么？
*   **Answer Key**:
    *   **Type Punning**: 将一块内存当作另一种不兼容的类型来解释。Swift 严厉禁止这样做（除非是 Trivial 类型且小心处理）。
    *   **bindMemory**: 更改内存的绑定类型。如果内存已经被绑定为 A，你强制绑定为 B，再按 A 访问，是未定义行为。
    *   **assuming**: 告诉编译器 "我知道这块内存已经是 T 了，别管了，给我个指针"。不改变绑定状态。

---

## 5. SwiftUI Internals (Hacking)

### Q9: PreferenceKey Propagation
**Q**: `PreferenceKey` 是如何从子 View 传递到父 View 的？
*   **Follow-up**:
    *   如果多个子 View 都有同一个 Key，`reduce` 方法是如何工作的？
    *   **Algorithm**: 这是一个深度优先遍历还是后续遍历？
*   **Answer Key**:
    *   **Process**: 子 View 产生值 -> 向上冒泡 -> 父 View 收集所有子 View 的值 -> 调用 `reduce` 合并 -> 继续向上。
    *   **Timing**: 发生在 Layout 之后，Render 之前。

### Q10: Custom Layout Protocol
**Q**: 实现一个瀑布流布局 (`WaterfallLayout`)。
*   **Follow-up**:
    *   `Layout` 协议的 `makeCache(subviews:)` 有什么用？
    *   如何处理 **Bidirectional Layout** (子 View 的大小依赖父 View，父 View 的大小又依赖子 View)？
*   **Answer Key**:
    *   **Cache**: 用于存储预计算的布局信息（如每个 Cell 的 Frame），避免在 `placeSubviews` 时重复计算。
    *   **Circular Dependency**: `sizeThatFits` 和 `placeSubviews` 分离。Layout 引擎会多次协商。

---

## 6. Coding Challenge: The "Impossible" Task

### Q11: Async Sequence from Scratch
**Q**: 不使用 `AsyncStream`，手动实现一个遵循 `AsyncSequence` 的类型，该类型封装了一个基于 Delegate 的旧 API（如 `CLLocationManager`）。
*   **Requirements**:
    *   必须处理 **Backpressure**（如果消费者处理慢，生产者怎么办？丢弃？缓冲？）。
    *   必须处理 **Cancellation**（Task 取消时，停止 Delegate 监听）。
    *   必须是 **Thread Safe**。
*   **Hint**: 需要实现 `AsyncIteratorProtocol`，并使用 `UnsafeContinuation` 或 `NSRecursiveLock` 来管理状态机。
