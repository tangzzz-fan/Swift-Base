# 20. SwiftUI Architecture Principles: State, Navigation & DI

> **Format**: Technical Article (Not Q&A)
> **Focus**: Memory Management, Navigation Design, Dependency Injection Internals.

---

## 1. The Mystery of @State Memory

在 SwiftUI 中，我们经常听到 "View 是值类型，是廉价的，会被频繁销毁重建"。那么问题来了：**如果 View 结构体被销毁了，存储在里面的 `@State var count: Int` 的值为什么没有丢失？**

### 1.1 The "Shadow" Storage
当你声明 `@State var count = 0` 时，`count` 的实际值**并不**存储在 View 结构体实例中。
View 结构体只是一个**蓝图 (Blueprint)** 或 **描述符 (Descriptor)**。

*   **AttributeGraph**: SwiftUI 维护了一个持久的依赖图 (AttributeGraph)。
*   **Identity**: 每个 View 在 Graph 中都有一个唯一的 Identity (由结构性位置或 `.id()` 决定)。
*   **Allocation**: 当 View 首次被挂载 (Mount) 到 Graph 上时，SwiftUI 运行时会为所有的 `@State` 属性在**堆内存**中分配存储空间，并将这块内存与 Graph Node 关联。
*   **Pointer**: View 结构体中的 `@State` 包装器，实际上只持有一个指向这块堆内存的**指针** (或者说是 Key)。

### 1.2 The Lifecycle
1.  **Init**: View 结构体被初始化。`@State` 只是被赋予了初始值（作为默认值）。
2.  **Body Evaluation**: SwiftUI 调用 `body`。在读取 `@State` 时，Runtime 会根据当前 View 的 Identity 去 Graph 中查找对应的堆内存。
    *   如果找到，返回存储的值（忽略 View init 时的默认值）。
    *   如果没找到（首次渲染），使用默认值初始化堆内存。
3.  **Update**: 当 `@State` 被修改，它修改的是堆内存中的值，并标记 Graph Node 为 Dirty。
4.  **Re-render**: SwiftUI 创建新的 View 结构体。新的结构体依然通过 Identity 关联到**同一个**堆内存。所以状态得以保留。

---

## 2. Navigation: Why Stack Replaces Coordinator?

在 UIKit 时代，Coordinator 模式是为了解决 `UIViewController` 之间的强耦合。但在 SwiftUI 中，Apple 推出了 `NavigationStack` (iOS 16) 来彻底改变导航。

### 2.1 The Problem with "View-Driven" Navigation
早期的 `NavigationView` + `NavigationLink` 是 **View-Driven** 的。
*   导航状态分散在各个 View 中（`isActive` 绑定）。
*   很难通过代码一次性 Pop 到 Root，或者直接跳转到深层页面 (Deep Link)。
*   Coordinator 必须持有 View 的引用才能操作，这违背了 SwiftUI 的声明式原则。

### 2.2 The "State-Driven" Philosophy
`NavigationStack(path: $path)` 是 **State-Driven** 的。
*   **Single Source of Truth**: 导航路径不再是 View 层级的副作用，而是一个纯数据结构 (`[Route]`)。
*   **Decoupling**:
    *   **Router**: 只需要管理这个数组。`path.append(.detail)`。
    *   **View**: 只需要响应数据。`.navigationDestination(for: Route.self)`。
*   **Why it replaces Coordinator**:
    *   它内置了 Coordinator 的核心能力（解耦路由）。
    *   它天然支持 Deep Link（直接构造数组）。
    *   它支持状态保存与恢复（Codable Path）。
    *   它不需要手动管理 View Controller 的压栈出栈，SwiftUI 引擎自动同步 State 和 UI。

---

## 3. Dependency Injection: Internals & Patterns

SwiftUI 的依赖注入主要通过 `Environment` 实现。这是一种 **Service Locator** 模式的变体，结合了 **Scoped Propagation**。

### 3.1 Implementation Principle
`Environment` 的底层实现类似于一个**链表**或**字典树**。

1.  **EnvironmentValues**: 这是一个容器，存储所有的 Key-Value 对。
2.  **Propagation**: 当你在父 View 上调用 `.environment(\.key, value)` 时，SwiftUI 会创建一个新的 Environment Context，覆盖指定 Key 的值，并将这个 Context 传递给所有子节点。
3.  **O(1) Access**: 虽然逻辑上是树状传递，但 SwiftUI 内部优化了查找性能。通常通过 Bitmask 或 Index 索引，使得读取 Environment 的开销极低。
4.  **Dynamic Dependency**: `Environment` 也是 `DynamicProperty`。当上层修改了 Environment 值，所有读取该 Key 的子 View 都会自动刷新。

### 3.2 Similar Scenarios (TaskLocal)
在 Swift Concurrency 中，有一个极其相似的概念：**TaskLocal**。

*   **Concept**: `TaskLocal` 是 Task 作用域内的全局变量。
*   **Propagation**: 当创建一个子 Task 时，它会继承父 Task 的所有 TaskLocal 值。
*   **Usage in Architecture**:
    *   TCA 的 `Dependency` 系统正是基于 `TaskLocal` 实现的。
    *   它允许我们在不显式传递参数的情况下，将依赖注入到深层的函数调用栈中。
    *   `withDependencies { $0.api = .mock } operation: { ... }`。

### 3.3 Comparison
| Feature | Environment | TaskLocal |
| :--- | :--- | :--- |
| **Scope** | View Tree (UI) | Task Tree (Async Context) |
| **Update** | Reactive (Triggers View Update) | Static (Copy-on-Write at creation) |
| **Use Case** | Theme, Font, Data Store (UI related) | API Client, Date Provider, Analytics (Logic related) |

---

## 4. Summary

*   **State Memory**: View 是临时的，State 是永恒的（在 Graph 中）。理解这一点是理解 SwiftUI 生命周期的关键。
*   **Navigation**: 从 "命令式操作 UI" 转向 "声明式操作数据"。NavigationStack 是这一哲学的终极体现。
*   **DI**: 无论是 `Environment` 还是 `TaskLocal`，核心思想都是 **Implicit Propagation** (隐式传播)，减少手动传递参数的 "Drilling" 痛苦。
