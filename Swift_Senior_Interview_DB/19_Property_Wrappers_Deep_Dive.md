# 19. Swift Property Wrappers: Deep Dive & Internals

> **Format**: Technical Article (Not Q&A)
> **Focus**: Implementation Principles, SwiftUI Wrappers, Custom Scenarios.

---

## 1. The Anatomy of a Property Wrapper

Property Wrapper 是 Swift 5.1 引入的语法糖，本质上是将属性的**存储**和**访问逻辑**封装在一个独立的结构体或类中。

### 1.1 Compiler Translation
当你写下：
```swift
@MyWrapper var x: Int
```
编译器实际上生成了三个东西：
1.  **_x (Backing Storage)**: 存储 `MyWrapper` 实例本身。`private var _x: MyWrapper<Int>`。
2.  **x (Computed Property)**: 访问 `_x.wrappedValue`。
3.  **$x (Projected Value)**: 访问 `_x.projectedValue` (如果有定义)。

### 1.2 Key Components
*   `wrappedValue`: **必须**。代表被包装的实际值。
*   `projectedValue`: **可选**。通过 `$` 访问。通常用于暴露 Wrapper 自身的引用、Binding 或者其他辅助对象（如 Combine 的 Publisher）。
*   `init(wrappedValue:)`: **可选**。支持 `@MyWrapper var x = 10` 这种初始化语法。

---

## 2. SwiftUI Wrappers: Under the Hood

SwiftUI 的魔法很大程度上依赖于特定的 Property Wrappers。它们不仅仅是简单的封装，还遵循了 `DynamicProperty` 协议，与 SwiftUI 的 Runtime (AttributeGraph) 深度绑定。

### 2.1 @State
*   **Definition**: `@propertyWrapper struct State<Value>: DynamicProperty`
*   **Memory Principle**:
    *   `State` 结构体本身非常轻量，只包含一个指向堆内存的指针（或 Key）。
    *   **Heap Allocation**: 真正的值**不**存储在 View 结构体中（因为 View 是值类型，且会被频繁销毁重建）。值存储在 SwiftUI 维护的 **AttributeGraph** 节点中，或者一个独立的堆内存区域。
    *   **Persistence**: 当 View 重建时，SwiftUI 根据 View 的位置 (Identity) 找到对应的 Graph Node，从而恢复旧的 State 值。这就是为什么 View 刷新了，State 却还在。
*   **Setter Magic**:
    *   当修改 `wrappedValue` 时，`State` 内部会通过 Runtime 发送一个 "Dirty" 信号给 AttributeGraph。
    *   Graph 标记该 View 为 Dirty，并在下一个 RunLoop 周期触发 `body` 重算。

### 2.2 @StateObject
*   **Difference from @ObservedObject**:
    *   `@ObservedObject` 只是被动观察。如果 View 重绘，`ObservedObject` 实例本身由外部传入，或者如果在 View `init` 中创建，会被重新创建（导致状态丢失）。
    *   `@StateObject` 是**所有者** (Owner)。它利用 `AutoRelease` 机制或 Graph 的生命周期管理，保证即使 View struct 重建，Object 实例**只会被创建一次**。
*   **Implementation**:
    *   它内部持有一个 `thunk`，在 View 首次加载 (`onAppear` 之前) 懒加载创建对象，并将其存入 Graph 的存储中。

### 2.3 @Environment
*   **Mechanism**:
    *   依赖于 `EnvironmentValues` 结构体。这是一个类似于 `Dictionary<KeyPath, Any>` 的容器。
    *   **Propagation**: Environment 值是沿着 View Tree 向下传递的。
    *   **Injection**: 当你写 `@Environment(\.colorScheme) var color` 时，Wrapper 内部记录了 `\.colorScheme` 这个 KeyPath。
    *   **DynamicProperty**: 在 `update()` 阶段，SwiftUI 会从当前的 Environment Context 中根据 KeyPath 取出值，赋给 Wrapper。

---

## 3. Practical Scenarios for Custom Wrappers

在实际开发中，除了使用系统提供的 Wrapper，我们也可以自定义 Wrapper 来消除样板代码。

### 3.1 @UserDefault (Persistence)
最经典的用法。将 `UserDefaults` 的读写封装起来。
```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }
}

// Usage
@UserDefault(key: "has_seen_onboarding", defaultValue: false)
static var hasSeenOnboarding: Bool
```

### 3.2 @Clamped (Validation)
限制数值的范围。
```swift
@propertyWrapper
struct Clamped<Value: Comparable> {
    var value: Value
    let range: ClosedRange<Value>
    
    init(wrappedValue: Value, _ range: ClosedRange<Value>) {
        self.range = range
        self.value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }
    
    var wrappedValue: Value {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

// Usage
@Clamped(0...100) var percentage: Int = 150 // Becomes 100
```

### 3.3 @ThreadSafe (Concurrency)
利用锁机制保护属性的读写。
```swift
@propertyWrapper
class ThreadSafe<Value> {
    private var value: Value
    private let lock = NSLock()
    
    init(wrappedValue: Value) { self.value = wrappedValue }
    
    var wrappedValue: Value {
        get { lock.withLock { value } }
        set { lock.withLock { value = newValue } }
    }
}
```

### 3.4 @CopyOnWrite (Optimization)
为自定义的结构体实现 CoW 语义。
```swift
@propertyWrapper
struct CopyOnWrite<Value> {
    private var ref: Box<Value>
    
    var wrappedValue: Value {
        get { ref.value }
        set {
            if !isKnownUniquelyReferenced(&ref) {
                ref = Box(newValue) // Copy
            } else {
                ref.value = newValue // Mutate in place
            }
        }
    }
}
```

---

## 4. Summary

*   **Syntactic Sugar**: Property Wrapper 本质是编译器生成的辅助代码，极大地简化了 getter/setter 逻辑。
*   **SwiftUI Core**: SwiftUI 利用 Wrapper + `DynamicProperty` 协议，打通了数据 (State) 与 视图 (View) 以及 渲染引擎 (AttributeGraph) 之间的通道。
*   **Power**: 通过自定义 Wrapper，我们可以将持久化、验证、线程安全等横切关注点 (Cross-Cutting Concerns) 封装起来，保持业务代码的整洁。
