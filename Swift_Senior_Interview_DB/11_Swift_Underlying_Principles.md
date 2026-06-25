# 11. Swift Underlying Principles (The "Black Magic" Edition)

> **Focus**: SIL, Runtime, Metadata, Mangling, ABI.
> **Difficulty**: Expert / Compiler Engineer level.

---

## 1. SIL & Compilation Pipeline

### Q1: SIL (Swift Intermediate Language) Stages
**Q**: Swift 编译过程中，Raw SIL 和 Canonical SIL 有什么区别？
*   **Follow-up**: `alloc_box` 和 `alloc_stack` 指令在 SIL 中分别代表什么？编译器如何进行 "Stack Promotion" (栈提升) 优化？
*   **Answer**:
    *   **Raw SIL**: 刚从 AST 转换来，包含未检查的逻辑。
    *   **Canonical SIL**: 经过了一系列优化（如 Mandatory Inlining, Definite Initialization）和检查后的 SIL，准备生成 LLVM IR。
    *   **Stack Promotion**: 如果一个逃逸分析证明对象没有逃逸出当前作用域，编译器会将 `alloc_box` (堆分配) 优化为 `alloc_stack` (栈分配)。

### Q2: Method Swizzling in Swift
**Q**: 在纯 Swift (无 `@objc`) 中，如何实现类似 Method Swizzling 的功能？
*   **Follow-up**: `dynamic` 关键字在 Swift 5+ 中开启了什么机制？(Dynamic Replacement)。
*   **Answer**:
    *   纯 Swift 默认不支持 Swizzling (因为是静态分发或 VTable)。
    *   **Dynamic Replacement**: 使用 `@_dynamicReplacement(for: originalMethod)`。需要编译时开启 `-enable-implicit-dynamic` 或在源码加 `dynamic`。
    *   底层原理：函数入口处会检查一个全局的替换表，如果有替换实现，跳转执行。

### Q3: Mangling Scheme
**Q**: Swift 的符号重整 (Mangling) 规则是怎样的？比如 `_$s5MyApp4UserC4nameSSvp` 代表什么？
*   **Follow-up**: 如何在运行时通过 Mangled Name 获取 Type Metadata？(`_typeByName`)。
*   **Answer**:
    *   `_s`: Swift 符号前缀。
    *   `5MyApp`: Module Name (Length 5 + "MyApp").
    *   `4User`: Class Name.
    *   `C`: Class.
    *   `4name`: Property Name.
    *   `SS`: Swift.String.
    *   `vp`: Variable Property.

---

## 2. Runtime & Metadata

### Q4: Type Metadata Layout
**Q**: 一个 Class 的 Metadata 在内存中具体包含哪些字段？
*   **Follow-up**: `Generic Metadata Pattern` 是什么？泛型类型的 Metadata 是如何动态实例化的？
*   **Answer**:
    *   Header (Destructor, Value Witness Table, Superclass), Type Descriptor, Generic Arguments, V-Table。
    *   **Generic Pattern**: 泛型类的 Metadata 不是静态生成的，而是一个模板。运行时根据具体的泛型参数 (Generic Arguments) 填充模板，生成具体的 Metadata 实例。

### Q5: RefCount Implementation
**Q**: Swift 的引用计数是原子操作吗？它使用了哪种内存序 (Memory Ordering)？
*   **Follow-up**: 为什么 `swift_retain` 比 ObjC 的 `objc_retain` 更快？(Inline RefCount vs Side Table lookup)。
*   **Answer**:
    *   是原子的。使用 `Relaxed` 用于增加，`Release` 用于减少（需要保证析构前的操作都完成）。
    *   Swift 优先使用 Inline RefCount (对象头部)，无锁访问（原子指令）。ObjC (旧版) 需要查全局哈希表，新版 ObjC 也有优化，但 Swift 的 Inline 布局更紧凑。

### Q6: Weak Reference State Machine
**Q**: 详细描述 `WeakReference` 指向的对象销毁时的状态机变化。
*   **Follow-up**: 什么是 "Zombie Object" (僵尸对象) 在 Swift 里的对应概念？(Deiniting vs Deinited)。
*   **Answer**:
    *   Live -> Deiniting (调用 deinit) -> Deinited (内存未释放，但在 Side Table 中标记为僵尸) -> Dead (内存释放)。
    *   Weak 指针指向 Side Table。当对象进入 Deinited 状态，Side Table 里的标记位变了。Weak 指针读取时发现标记位，返回 nil。

### Q7: Existential Container (Opaque vs Class)
**Q**: `any Protocol` (Opaque) 和 `any AnyObject & Protocol` (Class-constrained) 的内存布局一样吗？
*   **Follow-up**: 为什么 Class-constrained Existential 性能更好？
*   **Answer**:
    *   不一样。
    *   **Opaque**: 3 words buffer + VWT + PWT (5 words)。
    *   **Class-constrained**: 只需要存储 Object Reference + PWT (2 words)。不需要 VWT (因为引用类型生命周期管理统一)，不需要 Buffer (直接存指针)。

---

## 3. Memory & Layout

### Q8: Type Layout (Size, Stride, Alignment)
**Q**: `Size` 和 `Stride` 的区别？举例说明。
*   **Follow-up**: `struct A { var a: Bool; var b: Int }` 的 Size 和 Stride 分别是多少？(64-bit)。
*   **Answer**:
    *   **Size**: 实际数据占用的字节数 (不含尾部填充)。
    *   **Stride**: 在数组中连续存储时，两个元素起点的距离 (含尾部填充，必须是 Alignment 的倍数)。
    *   `A`: Bool(1) + padding(7) + Int(8) = 16 bytes. Size=16, Stride=16. (如果是 `Int, Bool`，Size=9, Stride=16)。

### Q9: KeyPath Memory Layout
**Q**: `KeyPath` 对象在内存中存了什么？
*   **Follow-up**: 为什么 KeyPath 访问比闭包慢？
*   **Answer**:
    *   KeyPath 包含一系列的 "Component" (Offset, Computed Property ID, Optional Chaining 等)。
    *   访问时需要运行时解释执行这些 Component（类似字节码解释器），计算偏移量。比直接内存访问或闭包调用慢。

### Q10: COW Implementation Details
**Q**: 标准库的 `Array` 是如何处理 `subscript` 的 `mutating` 的？
*   **Follow-up**: `_makeUniqueAndReserveCapacityIfNotUnique` 内部逻辑。
*   **Answer**:
    *   在 `subscript set` 中，首先调用 `isUniquelyReferenced`。
    *   如果不是唯一引用，分配新 Buffer，拷贝旧数据，修改新 Buffer。
    *   如果是唯一引用，直接修改 Buffer。

---

## 4. Advanced Runtime

### Q11: Runtime Entry Points
**Q**: `swift_allocObject` 和 `swift_bridgeObjectRetain` 分别在什么时候调用？
*   **Follow-up**: BridgeObject 是为了优化什么？(Swift String 与 NSString 的桥接)。
*   **Answer**:
    *   `allocObject`: 创建 Class 实例或 Box 时。
    *   `bridgeObjectRetain`: 当 String/Array 内部可能持有 ObjC 对象时。BridgeObject 使用 Tagged Pointer 技术，低位标记是否是 ObjC 对象，如果是则调用 ObjC retain，否则调用 Swift retain。

### Q12: Reflection (Mirror) Internals
**Q**: `Mirror` 如何在不知道类型结构的情况下遍历属性？
*   **Follow-up**: `Field Descriptor` 存储在哪里？
*   **Answer**:
    *   编译器将类型的 `Field Descriptor` (包含属性名、类型引用、偏移量计算函数) 存放在二进制的 `__TEXT` 段。
    *   运行时 `Mirror` 读取 Metadata 找到 Descriptor，解析出属性信息。

### Q13: Module Stability
**Q**: `.swiftinterface` 文件包含了什么？它和 `.swiftmodule` 的区别？
*   **Follow-up**: 为什么 Library Evolution 需要开启 "Build Libraries for Distribution"？
*   **Answer**:
    *   `.swiftmodule`: 二进制格式，包含 AST，依赖特定编译器版本。
    *   `.swiftinterface`: 文本格式，类似头文件，包含 public 接口和 `@inlinable` 实现。保证跨编译器版本兼容。

### Q14: ABI Stability Details
**Q**: ABI 稳定意味着什么？Struct 的内存布局是 ABI 的一部分吗？
*   **Follow-up**: 如果给一个 public struct 增加一个存储属性，会破坏 ABI 吗？
*   **Answer**:
    *   意味着运行时库 (libswiftCore.dylib) 可以内置在 OS 中，App 不需要自带。
    *   是的。如果 Struct 不是 `@frozen` 的，它的布局是不透明的，增加属性不破坏 ABI (因为访问是通过 Runtime Offset 函数)。如果是 `@frozen`，增加属性破坏 ABI。

### Q15: Pointer Rebinding
**Q**: `withMemoryRebound` 和 `assumingMemoryBound` 的底层区别？
*   **Follow-up**: 为什么不能随意将 `Int` 指针强转为 `Float` 指针并解引用？
*   **Answer**:
    *   **TBAA (Type Based Alias Analysis)**。编译器假设不同类型的指针不重叠。
    *   `withMemoryRebound`: 临时通知编译器 "这里暂时当作 Float 看"，防止优化错误。
    *   随意强转违反 Strict Aliasing，可能导致读取旧值（编译器认为 Float 写入不会影响 Int 读取）。
