# 01. Runtime, Compiler & Dispatch

> **Difficulty**: Expert
> **Focus**: Method Dispatch, Metadata, ABI, Existential Containers

---

## 1. Method Dispatch (方法分发)

### Q1: Swift 的方法分发机制有哪些？请从底层汇编角度解释它们是如何工作的。
**Follow-up**:
1.  `extension` 中的方法一定是 Static Dispatch 吗？如果是 `@objc` 的 extension 呢？
2.  `final` 关键字具体做了什么优化？它对 V-Table 有什么影响？
3.  解释 "Devirtualization" (去虚拟化) 优化的过程。

#### Key Answer:
*   **Static Dispatch (Direct)**: 编译器直接插入函数地址 (`call 0x1000`). 最快，利于内联。
*   **Table Dispatch (V-Table / Witness Table)**:
    *   **Class**: 使用 V-Table (Virtual Method Table)。在运行时通过对象头部的 `isa` 指针找到类元数据，再偏移找到函数指针。(`ldr x1, [x0]; ldr x2, [x1, #offset]; blr x2`).
    *   **Protocol**: 使用 PWT (Protocol Witness Table)。
*   **Message Dispatch (ObjC Runtime)**: 使用 `objc_msgSend`。完全动态，支持 Swizzling。

**Trap**: `dynamic` 关键字强制使用 Message Dispatch，但仅限于 `@objc` 兼容的类型。普通 Swift 类的方法即使加了 `dynamic` (如果不是 `@objc`) 也是通过 V-Table 分发 (Swift 5+ 的 dynamic replacement 机制略有不同，但通常面试语境下指 ObjC 消息转发)。

---

## 2. Existential Containers (存在容器)

### Q2: 详细解释 `any Protocol` 类型的内存布局。当一个具体类型被赋值给协议类型变量时，发生了什么？
**Follow-up**:
1.  什么是 "Value Buffer"？它的大小是多少？(3 words / 24 bytes on 64-bit).
2.  如果一个 Struct 大小超过了 Value Buffer，Swift 会怎么处理？(Heap allocation).
3.  VWT (Value Witness Table) 和 PWT (Protocol Witness Table) 分别存储在 Existential Container 的哪里？它们的作用是什么？

#### Key Answer:
Existential Container (Opaque Existential) 的典型布局 (5 words):
1.  **Value Buffer (3 words)**: 存储值本身（如果小于 24 字节）或指向堆上值的指针。
2.  **VWT Pointer (1 word)**: 指向 Value Witness Table。负责管理值的生命周期（allocate, copy, destruct, deallocate）。因为编译器不知道具体类型，所以需要 VWT 来操作内存。
3.  **PWT Pointer (1 word)**: 指向 Protocol Witness Table。包含协议方法的实现函数指针。

**Performance Impact**: 大于 24 字节的值类型赋值给 `any Protocol` 会触发堆分配，导致性能下降。这就是为什么 Swift 5.7 引入 `Primary Associated Types` 和鼓励使用 `some Protocol` (Opaque Types) 的原因之一。

---

## 3. Metadata & Reflection (元数据与反射)

### Q3: Swift 的 `Mirror` 是如何工作的？它对性能有什么影响？
**Follow-up**:
1.  Swift 的 Metadata (Type Descriptor) 存储在哪里？(Binary 的 `__TEXT` 段).
2.  为什么 `Mirror` 性能差？
3.  如何不使用 `Mirror` 获取一个 Struct 的属性名和值？(提示: 熟悉 `NominalTypeDescriptor` 结构并手动解析内存).

#### Key Answer:
`Mirror` 基于 Swift Runtime 的 Metadata 机制。它在运行时解析类型的 Metadata 结构（Field Descriptor 等），动态构建属性视图。
*   **性能差的原因**: 涉及大量的字符串拷贝、Any 包装（Existential 转换）以及运行时的递归解析。
*   **高级技巧**: 直接操作指针访问 Metadata 结构（如 `TargetStructMetadata`），可以实现高性能的 JSON 序列化库（如 HandyJSON 的原理）。

---

## 4. ABI Stability & Module Stability

### Q4: 什么是 ABI Stability？它解决了什么问题？Library Evolution 又是为了解决什么？
**Follow-up**:
1.  为什么 Swift 5 之前，App 必须内置 Swift 标准库？
2.  `@frozen` enum 和普通 enum 在 ABI 层面有什么区别？
3.  如果你是一个 SDK 开发者，发布了一个二进制 Framework，修改了一个 `public struct` 的私有属性，会导致使用该 SDK 的 App 崩溃吗？为什么？

#### Key Answer:
*   **ABI Stability**: 二进制接口稳定。允许 App 运行在不同版本的 Swift Runtime 上（例如 iOS 系统内置的库）。
*   **Library Evolution (Module Stability)**: 允许库在不破坏二进制兼容性的前提下升级。
*   **Resilient Types**: 默认情况下，public struct 是 "Resilient" 的，编译器不会硬编码其大小和偏移量，而是通过运行时调用 Runtime 函数获取。这允许库作者添加私有属性而不破坏 ABI。
*   **@frozen**: 承诺布局不变，允许编译器进行硬编码优化（直接偏移量访问），但失去了添加存储属性的灵活性。
