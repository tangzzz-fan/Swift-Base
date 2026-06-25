# 14. Swift Protocol-Oriented Programming (The "Type System" Edition)

> **Focus**: Protocols, Generics, Type Erasure, Existentials, Opaque Types.
> **Difficulty**: Expert.

---

## 1. Protocol Essentials & Dispatch

### Q1: Protocol Witness Table (PWT)
**Q**: 当一个类型遵循某个协议时，编译器生成的 PWT 到底存了什么？它是如何支持动态分发的？
*   **Follow-up**: 如果协议继承了另一个协议 (`protocol B: A`)，遵循 `B` 的类型的 PWT 结构是怎样的？
*   **Answer**:
    *   PWT 是一张函数指针表。存储了遵循者 (Conformer) 对协议方法的具体实现地址。
    *   当通过 Existential (`any Protocol`) 调用方法时，运行时从 Existential Container 中取出 PWT，查找对应偏移量的函数指针并调用。
    *   **Inheritance**: PWT 会包含父协议的 PWT 指针（或者直接将被继承协议的方法表内联，取决于实现细节，通常是组合）。

### Q2: Static vs Dynamic Dispatch in Protocols
**Q**: 解释协议扩展 (Protocol Extension) 中的方法分发规则。
*   **Scenario**:
    ```swift
    protocol P { func foo() }
    extension P { func foo() { print("P") } func bar() { print("P_bar") } }
    struct S: P { func foo() { print("S") } func bar() { print("S_bar") } }
    let p: any P = S()
    p.foo(); p.bar()
    ```
    输出什么？为什么？
*   **Answer**:
    *   `p.foo()` -> "S"。因为 `foo` 在协议定义中有声明，是 **Witness Table Dispatch** (动态)。
    *   `p.bar()` -> "P_bar"。因为 `bar` 只在扩展中定义，未在协议声明中，是 **Static Dispatch**。编译器直接调用 `P.bar` 的实现，看不到 `S` 的覆盖。

### Q3: Protocol Inheritance vs Class Inheritance
**Q**: 协议继承和类继承在语义和内存模型上有什么本质区别？
*   **Follow-up**: 为什么 Swift 推荐 "Composition over Inheritance" (组合优于继承)？
*   **Answer**:
    *   **Class**: 单继承，共享存储布局 (VTable)，引用语义。
    *   **Protocol**: 多继承（组合），只约束行为（接口），不强制存储布局。
    *   **Composition**: 避免了 "Diamond Problem" (菱形继承) 的状态冲突，更灵活，解耦。

---

## 2. Generics & Associated Types (PATs)

### Q4: The "Self or Associated Type" Constraint
**Q**: 为什么包含 `associatedtype` 的协议不能直接作为变量类型 (`var x: Protocol`) 使用（在 Swift 5.7 之前）？
*   **Follow-up**: Swift 5.7 的 `any Protocol` 是如何解决这个问题的？
*   **Answer**:
    *   因为编译器无法确定该协议的大小和具体类型接口。`associatedtype` 意味着类型依赖于具体的实现者。
    *   **Swift 5.7**: 引入了 "Existential Opening" (隐式打开)。当调用 `any P` 的方法时，编译器隐式地将其解包为一个泛型 `T` (scope 内可见)，从而允许访问关联类型。

### Q5: Type Erasure Patterns
**Q**: 在 Swift 标准库中，`AnySequence`, `AnyIterator`, `AnyHashable` 分别使用了哪种类型抹除技术？
*   **Follow-up**: 手写一个 `AnyView` 风格的类型抹除包装器 (`Box` pattern)。
*   **Answer**:
    *   **Box Pattern**: 创建一个私有抽象基类 `_AnyBox<T>`，泛型子类 `_ConcreteBox<Base>: _AnyBox<T>`。`AnyWrapper` 持有 `_AnyBox` 指针。
    *   **Closure Pattern**: 将方法调用封装为闭包存储。`AnyIterator` 使用了这种（因为它只需要一个 `next` 函数）。

### Q6: Opaque Types (`some`)
**Q**: `some View` 返回的是什么？它和泛型返回类型 (`func foo<T: View>() -> T`) 有什么区别？
*   **Follow-up**: 为什么 `some` 被称为 "Reverse Generics" (反向泛型)？
*   **Answer**:
    *   **some**: 具体的类型由**被调用者** (Callee) 决定，但对调用者 (Caller) 隐藏。编译器知道具体类型。
    *   **Generic**: 具体的类型由**调用者** (Caller) 决定。
    *   **Reverse**: 泛型是 "Caller decides", Opaque 是 "Callee decides"。

### Q7: Primary Associated Types
**Q**: `some Sequence<String>` (Swift 5.7) 语法糖背后的真实约束是什么？
*   **Follow-up**: 如何在自定义协议中声明 Primary Associated Type？
*   **Answer**:
    *   等价于 `some Sequence where Element == String`。
    *   声明: `protocol Sequence<Element> { associatedtype Element }`。

---

## 3. Advanced Conformance

### Q8: Conditional Conformance
**Q**: `extension Array: Equatable where Element: Equatable`。如果 `Element` 不遵循 `Equatable`，`Array` 遵循吗？
*   **Follow-up**: 运行时如何检测 Conditional Conformance？(`as?` 转换)。
*   **Answer**:
    *   不遵循。
    *   运行时 `dynamic_cast` 会检查泛型参数的 Witness Table。如果满足条件，转换成功；否则失败。

### Q9: Retroactive Conformance
**Q**: 什么是 Retroactive Conformance？它有什么风险？
*   **Follow-up**: 如果两个模块都为 `Date` 实现了 `Identifiable` 协议，会发生什么？
*   **Answer**:
    *   为不属于你的类型（如系统类）添加不属于你的协议（或你的协议）的遵循。
    *   **Risk**: 命名冲突。如果两个模块都添加了同一个协议遵循，编译器或运行时会困惑，可能导致未定义行为或编译错误。

### Q10: Synthesized Conformance
**Q**: Swift 编译器能自动合成哪些协议的实现？(`Equatable`, `Hashable`, `Codable`, `CaseIterable`)。
*   **Follow-up**: 自动合成的条件是什么？
*   **Answer**:
    *   所有存储属性（或 Enum 的关联值）都必须遵循该协议。
    *   必须在类型定义的文件内声明遵循（Extension 中声明有时也可以，但有限制）。

---

## 4. Standard Library Protocols

### Q11: Sequence vs Collection
**Q**: 实现一个 `Sequence` 只需要实现什么？实现 `Collection` 呢？
*   **Follow-up**: 为什么 `Sequence` 允许是消耗性的 (Destructive)？
*   **Answer**:
    *   `Sequence`: `makeIterator()`。
    *   `Collection`: `startIndex`, `endIndex`, `subscript`, `index(after:)`。
    *   `Sequence` 设计包含了网络流等一次性数据源。

### Q12: Hashable & Equatable
**Q**: 为什么 `Hashable` 继承自 `Equatable`？
*   **Follow-up**: 如果两个对象 `hashValue` 相等，它们一定相等吗？反之呢？
*   **Answer**:
    *   哈希表的查找逻辑：先算 Hash 定位 Bucket，再用 `==` 确认是否是同一个 Key（解决哈希冲突）。所以必须支持 `==`。
    *   Hash 相等不一定相等（冲突）。
    *   相等则 Hash 必须相等（契约）。

### Q13: Identifiable
**Q**: `Identifiable` 协议的核心要求是什么？`id` 属性必须是 `UUID` 吗？
*   **Follow-up**: 在 SwiftUI 中，如果 `id` 不唯一会发生什么？
*   **Answer**:
    *   要求有一个 `ID` 关联类型，遵循 `Hashable`。
    *   不一定是 UUID，可以是 Int, String，只要在集合中唯一即可。
    *   SwiftUI 依赖 `id` 做 Diff。不唯一会导致渲染错误、动画异常、状态丢失。

### Q14: Codable (Encodable & Decodable)
**Q**: `Codable` 是如何利用编译器合成代码的？
*   **Follow-up**: 如何自定义 Key 的映射？(`CodingKeys` 枚举)。如何处理 JSON 中的动态类型？
*   **Answer**:
    *   编译器生成 `init(from:)` 和 `encode(to:)`。
    *   动态类型：使用 `KeyedDecodingContainer` 手动解码，或者使用 `UnkeyedContainer`。

---

## 5. Design Patterns with Protocols

### Q15: Protocol Composition
**Q**: `typealias Codable = Encodable & Decodable`。如何定义一个变量，要求它同时是 `UIView` 子类并且遵循 `MyProtocol`？
*   **Follow-up**: `some UIView & MyProtocol` 合法吗？
*   **Answer**:
    *   `var x: UIView & MyProtocol` (Swift 4+ 语法)。
    *   是的，Swift 5+ 支持 `some Class & Protocol`。

### Q16: Delegate Pattern vs Closures
**Q**: 在 Swift 中，Delegate 模式和闭包回调各有什么优缺点？
*   **Follow-up**: 为什么 Delegate 必须是 `weak`？协议需要限制为 `AnyObject` 吗？
*   **Answer**:
    *   **Delegate**: 结构清晰，支持多个方法，易于调试。需要定义协议。
    *   **Closure**: 简洁，由调用处定义上下文。容易导致循环引用 (`[weak self]`)。
    *   Delegate 必须 weak 防止循环引用。是的，协议必须继承 `AnyObject` (`protocol P: AnyObject`) 才能被 weak 引用。

### Q17: Type Erasure for Heterogeneous Collections
**Q**: 如何在一个数组中存储不同类型的对象（如 `Int`, `String`, `MyStruct`），并能统一调用某个方法？
*   **Follow-up**: 使用 `Any` vs 使用公共 Protocol 的区别？
*   **Answer**:
    *   定义一个 Protocol，让它们都遵循。数组类型为 `[any Protocol]`。
    *   `Any`: 需要运行时 `as?` 转换才能调用方法。
    *   `Protocol`: 可以直接调用协议方法（动态分发）。

### Q18: Dependency Injection with Protocols
**Q**: 如何利用协议实现可测试的依赖注入？
*   **Follow-up**: 什么是 "Mocking"？
*   **Answer**:
    *   定义 `ServiceProtocol`。
    *   App 代码依赖 `ServiceProtocol`。
    *   生产环境注入 `RealService`，测试环境注入 `MockService`。

### Q19: Recursive Protocol Constraints
**Q**: `protocol Tree { associatedtype Node: Tree }`。这种递归定义合法吗？
*   **Follow-up**: 编译器如何处理这种无限递归的类型检查？
*   **Answer**:
    *   合法。
    *   编译器在实例化时会检查具体类型。只要具体类型不是无限大小的（如 Struct 包含自身），就是安全的。

### Q20: Actor Protocol
**Q**: `Actor` 是一个协议吗？
*   **Follow-up**: `GlobalActor` 也是协议吗？
*   **Answer**:
    *   `Actor` 是一个协议 (`protocol Actor : AnyObject, Sendable`)。所有的 `actor` 类型都隐式遵循它。
    *   `GlobalActor` 也是一个协议，要求有一个静态的 `shared` 属性。

---

## 6. Practical Scenarios & Best Practices

### Q21: Module Division with Protocols
**Q**: [Scenario] 你正在设计一个模块化的电商 App。如何利用协议来划分 "Product Detail" (商品详情) 模块，使其解耦于具体的网络层和 UI 层？
*   **Answer**:
    *   **Feature Interface**: 定义 `ProductDetailFeatureProvider` 协议，暴露 `buildViewController(productId:) -> UIViewController`。
    *   **Dependency Interface**: 定义 `ProductServiceProtocol`，声明 `fetchProduct(id:)`。详情模块内部只依赖这个协议。
    *   **Injection**: 在 App 组装层 (Composition Root)，将具体的 `NetworkService` (遵循 `ProductServiceProtocol`) 注入给详情模块。
    *   **Benefit**: 详情模块可以独立编译、测试（使用 MockService），不依赖具体的网络库实现。

### Q22: The "Self" Pitfall in Heterogeneous Collections
**Q**: [Pitfall] 你定义了一个 `protocol Drawable { func draw() }`。然后尝试定义 `var list: [Drawable]`，但编译器报错 "Protocol 'Drawable' can only be used as a generic constraint because it has Self or associated type requirements"。这是为什么？如何解决？
*   **Answer**:
    *   **Reason**: 如果协议中有 `Self` 引用（如 `func copy() -> Self`）或 `associatedtype`，编译器无法确定异构数组中每个元素的具体大小和内存布局。
    *   **Solution 1 (Type Erasure)**: 创建 `AnyDrawable` 包装器。
    *   **Solution 2 (Existential Opening - Swift 5.7+)**: 如果只是 `associatedtype`，可以使用 `any Drawable`。但如果是 `Self` 返回值，依然受限。
    *   **Best Practice**: 尽量减少在协议接口中使用 `Self`，除非确实需要 Fluent Interface 或 Factory 方法。

### Q23: Protocol Best Practices
**Q**: [Best Practice] 在设计协议时，"Start with Concrete" 和 "Start with Protocol" 哪种更好？
*   **Answer**:
    *   **Start with Concrete**: 推荐。先写具体的 Struct/Class 实现业务逻辑。当发现有第二个类型需要类似行为，或者需要 Mock 进行测试时，再提取协议 (Refactor to Protocol)。
    *   **Premature Abstraction**: 过早定义协议往往导致接口设计不合理，或者为了适配协议而扭曲具体实现。
    *   **Rule of Thumb**: "Rule of Three". 当你有三个类似的使用场景时，是引入抽象的最佳时机。
