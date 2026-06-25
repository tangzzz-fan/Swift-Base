# 17. Modern Architecture Patterns (The "Clean" Edition)

> **Focus**: Clean Architecture, MVVM+Reducer, Coordinator in SwiftUI, Dependency Injection.
> **Difficulty**: Architect / Tech Lead.

---

## 1. Clean Architecture & MVVM+Reducer

### Q1: MVVM+Reducer vs Clean Architecture
**Q**: 在 Swift 中使用 MVVM+Reducer (如 TCA) 架构，是否符合 Clean Architecture 的原则？
*   **Follow-up**: Clean Architecture 的核心层级（Entities, Use Cases, Interface Adapters, Frameworks）在 TCA 中如何映射？
*   **Answer**:
    *   **符合**: 核心原则是 "依赖规则" (Dependency Rule)，即内层不依赖外层。
    *   **Mapping**:
        *   **Entities**: TCA 的 `State` (Model)。
        *   **Use Cases**: TCA 的 `Reducer` (业务逻辑)。
        *   **Interface Adapters**: TCA 的 `Action` 和 `Store` (连接 View 和 Logic)。
        *   **Frameworks**: SwiftUI View, CoreData, Networking。
    *   TCA 实际上是一种更严格、函数式的 Clean Architecture 实现。

### Q2: The "Massive Reducer" Risk
**Q**: 即使使用了 Reducer，如果一个 Feature 的逻辑非常复杂，Reducer 依然会变得巨大。Clean Architecture 如何解决这个问题？
*   **Follow-up**: 什么是 "Domain Logic" vs "Application Logic"？
*   **Answer**:
    *   **Split**: 将纯业务逻辑（Domain Logic）抽离为独立的 `struct` 或 `function` (Use Cases)，Reducer 只负责调度这些 Use Cases 并更新 State。
    *   **Domain Logic**: 与 UI 无关的规则（如 "订单金额计算"）。
    *   **Application Logic**: 协调 UI 和数据的流程（如 "点击按钮 -> 发请求 -> 更新 Loading 状态"）。

---

## 2. Navigation: The Evolution of Coordinator

### Q3: The Fate of Coordinator in SwiftUI
**Q**: 在 SwiftUI 中，传统的 MVVM-C (Coordinator) 模式似乎"失效"了。为什么？
*   **Follow-up**: SwiftUI 的 `NavigationStack` (iOS 16+) 如何改变了导航方式？
*   **Answer**:
    *   **Reason**: SwiftUI 的 View 是声明式的，且导航状态往往绑定在 View 树中 (`NavigationLink`)。传统的 Coordinator 强依赖 `UINavigationController` (命令式)，难以直接控制 SwiftUI 的声明式导航。
    *   **iOS 16+**: `NavigationStack(path: $path)` 引入了数据驱动的导航。这使得 Coordinator 模式回归成为可能（且更简单）。

### Q4: Implementing "Flow Controller" in SwiftUI
**Q**: [Code Scenario] 如何在 SwiftUI 中实现一个将导航逻辑从 View 中解耦的 "Flow Controller" (或 Router)？
*   **Answer**:
    *   **State Driven**: 定义一个 `enum Route: Hashable`。
    *   **Container**: 创建一个 `class Router: ObservableObject` 持有 `[Route]` 路径。
    *   **Root View**:
        ```swift
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    switch route {
                    case .detail(let id): DetailView(id: id)
                    case .settings: SettingsView()
                    }
                }
        }
        ```
    *   **Decoupling**: 子 View 只需要调用 `router.push(.detail(1))`，无需知道目标 View 是什么。

### Q5: Handling Deep Links
**Q**: 使用上述 Router 方案，如何处理 Deep Link？
*   **Follow-up**: 如何处理 "Tab 切换 + Push" 的复杂导航？
*   **Answer**:
    *   **Deep Link**: 解析 URL -> 转换为 `[Route]` 数组 -> 直接赋值给 `router.path`。SwiftUI 会自动重建视图堆栈。
    *   **Tabs**: 每个 Tab 需要独立的 `NavigationStack` 和 `Router`。Root Router 控制 Tab Selection。

---

## 3. Dependency Injection (DI) in SwiftUI

### Q6: Environment vs Initializer Injection
**Q**: 在 SwiftUI 中，依赖注入主要有哪些方式？各自的优缺点？
*   **Follow-up**: 为什么说 `@EnvironmentObject` 是 "隐式依赖"，有时是危险的？
*   **Answer**:
    *   **Initializer Injection**: `View(viewModel: VM(service: service))`。显式，安全，但层级深时传递麻烦。
    *   **EnvironmentObject**: `View().environmentObject(store)`。方便，穿透层级。
    *   **Danger**: 如果忘记在祖先节点注入，运行时会 Crash (`MissingEnvironmentObjectError`)。它是隐式的，编译器不检查。

### Q7: The "Dependency" Library Pattern
**Q**: 现代 SwiftUI 架构（如 TCA 或 PointFree 的 Dependencies 库）推荐什么样的 DI 方式？
*   **Follow-up**: 原理是什么？(`TaskLocal`)。
*   **Answer**:
    *   **Service Locator + TaskLocal**: 类似于 SwiftUI 的 Environment，但是用于逻辑层。
    *   **Usage**: `@Dependency(\.apiClient) var apiClient`。
    *   **Mechanism**: 使用 `TaskLocal` 存储依赖容器。在测试或 Preview 中，可以在 `withDependencies` 闭包中覆盖依赖。
    *   **Code**:
        ```swift
        // Definition
        private enum APIClientKey: DependencyKey {
            static let liveValue = APIClient.live
        }
        extension DependencyValues {
            var apiClient: APIClient {
                get { self[APIClientKey.self] }
                set { self[APIClientKey.self] = newValue }
            }
        }
        // Usage
        func fetch() async {
            @Dependency(\.apiClient) var api
            let data = await api.request()
        }
        ```

---

## 4. Practical Scenarios

### Q8: Modularizing Navigation
**Q**: [Scenario] 如果 App 分为 `HomeModule` 和 `ProfileModule`，它们互相不知道对方的存在。如何实现从 Home 跳转到 Profile？
*   **Answer**:
    *   **Protocol Abstraction**: 定义 `ProfileRouteProvider` 协议。
    *   **Dependency Injection**: Home 模块依赖 `ProfileRouteProvider`。
    *   **Implementation**: 在 App 主工程中实现该协议，返回 `ProfileView` (被 `AnyView` 或 `some View` 抹除类型)。
    *   **Router**: Home 的 Router 调用 Provider 获取 View 并 Push。

### Q9: View-ViewModel Binding Cycle
**Q**: [Pitfall] 在 SwiftUI 中使用 MVVM，如果在 ViewModel 的 `init` 中引用了 View 传递的参数，并在 View 的 `init` 中创建 ViewModel，会发生什么？
*   **Answer**:
    *   **State Loss**: SwiftUI View 的 `init` 会被频繁调用。如果 ViewModel 是在 `init` 中创建的 (`@StateObject var vm = ViewModel(...)`)，SwiftUI 能够保持它的生命周期。
    *   **Trap**: 但是，如果参数变化了，`StateObject` **不会** 自动重建。ViewModel 里的旧参数不会更新。
    *   **Fix**: 使用 `.onChange(of: param) { vm.update(param) }` 或在 `onAppear` 中赋值。

### Q10: "Smart" View vs "Dumb" View
**Q**: [Best Practice] 在 MVVM 架构下，View 应该包含逻辑吗？
*   **Answer**:
    *   **Dumb View**: 只负责渲染 State，转发 Action。不包含 `if/else` 业务逻辑。
    *   **Smart View (Container)**: 负责创建 ViewModel，注入依赖，处理生命周期。
    *   **Practice**: 将复杂的 View 拆分为 Smart Container 和 Dumb Component。方便 Preview 和复用。
