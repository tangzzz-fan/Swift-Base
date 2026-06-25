# 09. Advanced Swift (The Book Edition)

> **Source**: Based on concepts from *Advanced Swift* (objc.io).
> **Focus**: Structs/Classes, Generics, Protocols, Strings, Unsafe Swift, C Interop.

---

## 1. Structs & Classes (The Core)

### Q1: Copy-on-Write (CoW) Implementation Details
**Q**: 书中详细讲解了如何手动实现 CoW。请问 `isKnownUniquelyReferenced` 只能用于 Class 类型吗？为什么？
*   **Follow-up**: 如果我把一个 Class 包装在 Struct 里，但没有实现 CoW 逻辑，直接赋值 Struct 会发生什么？(引用语义泄露)。
*   **Answer**:
    *   是的，只能用于 Class。因为 Struct 是值类型，没有 "引用计数" 的概念（除非它包含 Class）。
    *   该函数检查的是 "指向这个 Object 的强引用是否只有 1 个"。
    *   如果直接赋值，两个 Struct 实例指向同一个 Class 实例。修改其中一个 Struct 的属性（实际上是修改 Class 的属性），另一个也会变。这违反了值语义。

### Q2: Mutating Semantics
**Q**: `mutating` 关键字对 `self` 做了什么？
*   **Follow-up**: 为什么 `let` 定义的 Struct 实例不能调用 `mutating` 方法？从内存角度解释。
*   **Answer**:
    *   `mutating` 方法会将 `self` 作为一个 `inout` 参数传递。
    *   `inout` 本质上是 "Copy-in, Copy-out" (或者引用传递优化)。
    *   `let` 实例是不可变的，无法被重写 (Write back)，所以不能传递给 `inout` 参数。

### Q3: Dictionary Internals
**Q**: Swift `Dictionary` 的底层实现是什么？(Linear Probing vs Chaining)。
*   **Follow-up**: 为什么 Dictionary 的 Key 必须遵循 `Hashable`？如果两个 Key 的 `hashValue` 相同会发生什么？
*   **Answer**:
    *   Swift 使用 **Open Addressing with Linear Probing** (开放寻址法 + 线性探测)。
    *   Hash Collision: 如果位置被占，往后找下一个空位。
    *   这也是为什么删除元素时需要 "Tombstone" (墓碑) 或者重新整理，否则探测链会断裂。

---

## 2. Strings & Encoding

### Q4: String Memory Layout
**Q**: Swift `String` 是如何存储的？Small String Optimization (SSO) 是什么？
*   **Follow-up**: `String.Index` 为什么不是简单的整数？
*   **Answer**:
    *   **SSO**: 如果字符串足够短 (15 bytes on 64-bit)，直接存放在 String 结构体内部，不分配堆内存。
    *   **Large**: 存放在堆上 (StringGuts)。
    *   **Index**: 因为 UTF-8 是变长编码。第 N 个字符的字节偏移量不是 N。Index 必须包含编码信息来确保遍历安全。

### Q5: Substring Memory Leak
**Q**: 为什么长期持有 `Substring` 会导致内存泄漏？
*   **Follow-up**: 如何修复？
*   **Answer**:
    *   `Substring` 共享原始 `String` 的内存 (Storage)。
    *   如果你有一个 10MB 的 String，切片出一个 10 bytes 的 Substring，只要这个 Substring 还在，那 10MB 的内存就无法释放。
    *   **Fix**: `String(substring)`。创建一个新的 String 实例，拷贝那 10 bytes，释放原来的大内存。

---

## 3. Generics & Protocols

### Q6: Generic Specialization
**Q**: 泛型特化 (Specialization) 是什么？它对二进制体积有什么影响？
*   **Follow-up**: 什么时候泛型不会被特化？(跨模块且未 `@inlinable`)。
*   **Answer**:
    *   编译器为具体的类型生成专门的代码版本 (e.g., `Array<Int>`, `Array<String>`)，移除动态分发开销。
    *   **体积**: 会导致代码膨胀 (Code Bloat)。
    *   **优化**: Swift 编译器会权衡，只对频繁使用的类型特化。

### Q7: Protocol Witness Table (PWT)
**Q**: 解释 PWT 的结构。当调用 `func foo<T: Equatable>(x: T, y: T)` 时，编译器如何找到 `==` 函数？
*   **Follow-up**: 为什么 `any Equatable` 无法作为参数传递给 `foo`？
*   **Answer**:
    *   编译器会隐式传递一个 PWT 指针给 `foo`。PWT 里包含了 `T` 遵循 `Equatable` 的所有方法地址。
    *   `any Equatable` 是 Existential。它隐藏了具体类型。而 `foo` 需要一个具体的 `T`。Existential 无法满足 `T` 的类型一致性要求 (x 和 y 必须是同一种 T)。

### Q8: Conditional Conformance
**Q**: `extension Array: Equatable where Element: Equatable` 是如何工作的？
*   **Follow-up**: 编译器如何检查这个约束？
*   **Answer**:
    *   只有当数组元素也遵循 Equatable 时，数组才遵循 Equatable。
    *   运行时，Swift 会检查泛型参数的 Witness Table 是否满足条件。

---

## 4. Unsafe Swift & C Interop

### Q9: Pointer Types
**Q**: `UnsafeMutablePointer<T>` 和 `UnsafeMutableRawPointer` 的区别？
*   **Follow-up**: 什么是 "Strict Aliasing" 规则？为什么 `assumingMemoryBound(to:)` 很危险？
*   **Answer**:
    *   Typed vs Untyped (void*)。Typed 指针知道步长 (Stride)。
    *   **Strict Aliasing**: 编译器假设不同类型的指针不会指向同一块内存（除了 char*）。如果违反，编译器优化可能会导致读写顺序错误。
    *   `assumingMemoryBound`: 告诉编译器 "这块内存已经是 T 类型了，信我"。如果实际上不是，就是 Undefined Behavior。

### Q10: Memory Binding
**Q**: `bindMemory(to:capacity:)` 做了什么？
*   **Follow-up**: 什么时候应该用 `withMemoryRebound`？
*   **Answer**:
    *   `bindMemory`: 更改内存的**类型状态**。这块内存从此被视为新类型。
    *   `withMemoryRebound`: 临时将指针类型转换一下（假设内存布局兼容），执行闭包，然后转回来。不永久改变内存绑定状态。

### Q11: C Function Pointers
**Q**: 如何将 Swift 闭包传递给 C 函数指针？
*   **Follow-up**: 为什么捕获了上下文 (Context) 的闭包不能传递给 C 函数指针？
*   **Answer**:
    *   C 函数指针只是一个地址。
    *   Swift 闭包 = 函数指针 + 捕获上下文 (Context Object)。
    *   如果闭包没有捕获任何变量，编译器可以将其降级为 C 函数指针 (`@convention(c)`).
    *   如果有捕获，必须通过 `Unmanaged` 传递 Context 指针给 C API (如果 C API 支持 `void* context` 参数)。

---

## 5. Reflection & Mirror

### Q12: Mirror Performance
**Q**: 为什么 `Mirror` 性能很差？
*   **Follow-up**: `Mirror` 是如何遍历 Struct 的属性的？
*   **Answer**:
    *   `Mirror` 基于运行时的 Metadata 解析。
    *   它会动态分配内存来构建属性视图，并且返回的属性值都是 `Any` 类型（涉及装箱 Boxing）。
    *   大量字符串操作和递归。

### Q13: CustomReflectable
**Q**: `CustomReflectable` 协议的作用？
*   **Follow-up**: Xcode 的调试区 (Variables View) 和 `po` 命令是否使用 Mirror？
*   **Answer**:
    *   允许类型自定义 `Mirror` 的结构。例如 `Array` 自定义 Mirror 只显示元素，不显示内部 Buffer 指针。
    *   是的，LLDB 和 Xcode 调试器大量使用 Mirror 机制来展示 Swift 变量结构。

---

## 6. Operators & Literals

### Q14: Operator Overloading
**Q**: 为什么过度使用运算符重载会拖慢编译速度？
*   **Follow-up**: `precedencegroup` 的作用？
*   **Answer**:
    *   增加了类型检查器 (Type Checker) 的搜索空间。编译器必须尝试所有可能的重载组合。
    *   `precedencegroup`: 定义运算符的优先级和结合性，帮助编译器减少歧义，缩小搜索范围。

### Q15: ExpressibleBy...Literals
**Q**: 如何让自定义类型支持 `let x: MyType = "Hello"`？
*   **Follow-up**: 这在编译期还是运行期发生的？
*   **Answer**:
    *   遵循 `ExpressibleByStringLiteral`。
    *   实现 `init(stringLiteral:)`。
    *   编译期将字面量转换为对该 init 方法的调用。
