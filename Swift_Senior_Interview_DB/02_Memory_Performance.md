# 02. Memory Management & Performance

> **Difficulty**: Senior / Expert
> **Focus**: ARC Internals, Side Tables, Pointers, Optimization

---

## 1. ARC Internals & Side Tables

### Q1: 深入解释 Swift 对象的销毁过程。为什么会有 "Side Table" (散列表)？
**Follow-up**:
1.  对象内存中的 RefCount 字段是如何布局的？(Inline RefCount).
2.  `weak` 引用是如何实现的？当对象销毁时，`weak` 指针是如何自动置 nil 的？
3.  解释 `unowned` (safe) 和 `unowned(unsafe)` 的区别。

#### Key Answer:
*   **Inline RefCount**: 默认情况下，引用计数直接存储在对象头部。
*   **Side Table**: 当发生以下情况时，会分配 Side Table，对象头部的 RefCount 字段变为指向 Side Table 的指针：
    1.  有 `weak` 引用指向该对象（Weak 引用必须指向 Side Table，因为对象销毁后 Side Table 还需要存活以支持 weak 置 nil）。
    2.  引用计数溢出（Inline 存不下了）。
*   **销毁流程**:
    1.  **Deinit**: 强引用归零 -> 调用 `deinit` -> 对象变为 "Deiniting" 状态。
    2.  **Dealloc**: 弱引用归零（Weak 指针检查 Side Table 状态发现对象已 deinit，返回 nil）-> 释放对象内存。
    3.  **Free Side Table**: 当所有 weak 引用也都释放后，Side Table 才被释放。

---

## 2. Copy-on-Write (CoW) Implementation

### Q2: 现场手写代码：为一个泛型类 `Box<T>` 实现 Copy-on-Write 语义。
**Follow-up**:
1.  `isKnownUniquelyReferenced` 的作用是什么？它是原子操作吗？
2.  如果在多线程环境下访问 CoW 结构体，会发生什么？如何保证线程安全？

#### Key Answer:
```swift
struct MyData<T> {
    private class Box<T> {
        var value: T
        init(_ value: T) { self.value = value }
    }
    
    private var box: Box<T>
    
    init(_ value: T) {
        self.box = Box(value)
    }
    
    var value: T {
        get { box.value }
        set {
            // 核心逻辑
            if !isKnownUniquelyReferenced(&box) {
                box = Box(newValue) // Copy
            } else {
                box.value = newValue // Mutate in place
            }
        }
    }
}
```
**Thread Safety Trap**: `isKnownUniquelyReferenced` 本身是线程安全的（检查引用计数），但 CoW 模式整体**不是**线程安全的。如果两个线程同时读取并准备写入，都可能判断为 "Unique"（如果当时只有一个引用），然后导致竞态条件。CoW 只是为了性能优化，不是锁。

---

## 3. Pointers & Unsafe Swift

### Q3: Swift 有哪几种指针类型？它们之间的转换规则是什么？
**Follow-up**:
1.  `UnsafePointer` vs `UnsafeRawPointer`。
2.  `withUnsafeBytes` 和 `withUnsafePointer` 的生命周期陷阱是什么？能否将闭包里的指针 return 出来使用？
3.  如何使用 `bindMemory`？什么时候需要 `assumingMemoryBound`？

#### Key Answer:
*   **Typed Pointers**: `UnsafePointer<T>`, `UnsafeMutablePointer<T>`. 知道步长 (Stride) 和对齐 (Alignment)。
*   **Raw Pointers**: `UnsafeRawPointer`, `UnsafeMutableRawPointer`. 类似 `void*`，按字节操作。
*   **Buffer Pointers**: `UnsafeBufferPointer<T>`. 类似数组，带 count。
*   **生命周期陷阱**: 绝对不能将 `withUnsafe...` 闭包中的指针逃逸出去。因为指针指向的内存可能在闭包结束时就被释放或重用（例如指向栈变量）。

---

## 4. Optimization Levels

### Q4: Swift 编译器的优化等级 `-O` 和 `-Osize` 有什么区别？
**Follow-up**:
1.  `-O` 会做哪些激进优化？(Inlining, Generic Specialization, Devirtualization).
2.  为什么有时候 Release 包会 Crash，但 Debug 包不会？(除了断言移除外，可能是 Exclusivity Enforcement 导致的，或者未定义行为被优化暴露了)。
3.  `@inlinable` 属性的作用和风险？

#### Key Answer:
*   **-O (Performance)**: 追求最快速度。大量内联，泛型特化（为每个具体类型生成代码，增加二进制体积），循环展开。
*   **-Osize (Size)**: 追求最小体积。减少内联，倾向于复用泛型代码（使用 Witness Table 动态分发），代码生成更保守。
*   **@inlinable**: 允许跨模块内联。
    *   **优点**: 性能提升。
    *   **风险**: 暴露了实现细节，破坏了库的二进制兼容性（如果库更新了实现，App 不重新编译的话，用的还是旧的内联代码）。
