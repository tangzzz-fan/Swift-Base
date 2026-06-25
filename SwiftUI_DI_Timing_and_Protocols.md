# SwiftUI 依赖注入时机与协议使用深度解析

本文档深入探讨在 SwiftUI 开发中，**何时**应该引入依赖注入，以及在声明式 UI 框架下，我们是否还需要像 MVVM-C 那样大量定义协议（Protocol）来实现解耦。

## 1. 什么时候使用依赖注入 (When to Inject)?

在 SwiftUI 中，依赖注入的时机主要分为两个维度：**模块内 (Intra-module)** 和 **模块间 (Inter-module)**。

### 1.1 模块内注入：逻辑与数据的解耦
**时机**: 当你的视图（View）开始包含复杂的业务逻辑，或者需要访问外部数据源（API, Database）时。

*   **View -> ViewModel**: 通常不需要协议。
    *   SwiftUI 的 `View` 只是一个配置结构体，非常轻量。直接依赖具体的 `ObservableObject` 类型通常是可以接受的，因为 `ObservableObject` 本身就是通过 `@Published` 抽象了数据变化。
    *   *例外*: 如果你需要为同一个 View 提供完全不同的逻辑实现（例如 `LiveModeViewModel` vs `TutorialModeViewModel`），则需要协议。

*   **ViewModel -> Service**: **必须使用协议**。
    *   这是单元测试的关键。ViewModel 不应直接实例化 `NetworkService`。
    *   **注入方式**: `init(service: ServiceProtocol)`。

### 1.2 模块间注入：特性与特性的解耦
**时机**: 当 Feature A 需要显示 Feature B 的界面，或者使用 Feature B 的数据时。

*   **Feature -> Feature**: **必须使用协议 (或闭包)**。
    *   永远不要在 Feature A 中 `import FeatureB`。
    *   **注入方式**: 通过 `Environment` 注入一个 `Router` 或 `Factory`。

---

## 2. SwiftUI 中还需要定义协议吗? (Protocol Necessity)

用户常问：*“MVVM-C 中我们为 View, ViewModel, Coordinator 都定义了协议。SwiftUI 还需要这么麻烦吗？”*

### 2.1 视图层 (View Layer): 大部分情况不需要
在 MVVM-C (UIKit) 中，我们定义 `UserViewProtocol` 是为了让 Presenter 不依赖具体的 `UIViewController`。

在 SwiftUI 中，我们有更好的替代品：
1.  **Generics (泛型)**: `struct Container<Content: View> { ... }`
2.  **ViewBuilder**: 闭包构建，完全抹除类型。
3.  **AnyView**: 类型擦除（虽然有性能损耗，但在模块边界处通常可忽略）。

**结论**: 除非是为了定义严格的模块边界（如上文提到的 `HomeRouter` 返回 `AnyView`），否则**不需要**为每个 View 定义协议。

### 2.2 逻辑层 (Logic Layer): 依然需要，且非常重要
SwiftUI 并没有改变业务逻辑的本质。为了测试 ViewModel，你依然需要 Mock 它的依赖（Services, Repositories）。

**对比**:
*   **MVVM-C**: 协议满天飞 (ViewInput, ViewOutput, InteractorInput...)。
*   **SwiftUI**: 协议集中在 **Service/Repository 层** 和 **模块边界 (Router)**。中间的 View 和 VM 绑定通常更直接。

---

## 3. 实际场景深度解析：订单列表嵌入用户中心

**场景**: 我们正在开发 `FeatureProfile` (用户中心)。需要在其中嵌入一个“最近订单”的卡片。这个卡片的内容属于 `FeatureOrder` (订单模块)。

### 步骤 1: 在 FeatureProfile 中定义需求 (协议)
Profile 模块不关心 Order 模块怎么实现，它只知道自己需要一个“视图”。

```swift
// FeatureProfile/Interfaces.swift
import SwiftUI

public protocol OrderHistoryProvider {
    // Profile 模块定义：我需要一个能显示最近订单的 View
    func makeRecentOrdersView(limit: Int) -> AnyView
}

// 默认的空实现 (用于 Preview 或占位)
struct EmptyOrderProvider: OrderHistoryProvider {
    func makeRecentOrdersView(limit: Int) -> AnyView {
        AnyView(Text("No Order Module Linked"))
    }
}
```

### 步骤 2: 在 FeatureProfile 中使用注入
ProfileView 通过 Environment 或 Init 获取这个 Provider。

```swift
// FeatureProfile/ProfileView.swift
struct ProfileView: View {
    let orderProvider: OrderHistoryProvider // 依赖协议
    
    var body: some View {
        VStack {
            Text("User Info")
            Divider()
            // 嵌入外部模块的 View
            orderProvider.makeRecentOrdersView(limit: 3)
        }
    }
}
```

### 步骤 3: 在 FeatureOrder 中实现协议
Order 模块依赖 `FeatureProfile` (或者它们都依赖一个公共接口模块)，并提供实现。

```swift
// FeatureOrder/OrderProvider.swift
import FeatureProfile
import SwiftUI

public struct RealOrderProvider: OrderHistoryProvider {
    public init() {}
    
    public func makeRecentOrdersView(limit: Int) -> AnyView {
        // 这里返回 Order 模块真正的 View
        let vm = RecentOrdersViewModel(limit: limit)
        return AnyView(RecentOrdersView(viewModel: vm))
    }
}
```

### 步骤 4: 在 App 层组装 (Dependency Injection)
这是 DI 发生的时刻。

```swift
// App/MyApp.swift
import FeatureProfile
import FeatureOrder

@main
struct MyApp: App {
    let orderProvider = RealOrderProvider() // 实例化实现
    
    var body: some Scene {
        WindowGroup {
            // 将 Order 模块的能力注入给 Profile 模块
            ProfileView(orderProvider: orderProvider)
        }
    }
}
```

### 步骤 5: Mock 与 预览 (The Payoff)
为什么我们要这么做？看看在 `FeatureProfile` 内部如何写 Preview：

```swift
// FeatureProfile/ProfileView_Previews.swift
#Preview {
    // 我们可以轻松 Mock 出一个“订单视图”，而不需要编译整个 Order 模块
    let mockProvider = MockOrderProvider() 
    ProfileView(orderProvider: mockProvider)
}

struct MockOrderProvider: OrderHistoryProvider {
    func makeRecentOrdersView(limit: Int) -> AnyView {
        AnyView(
            VStack {
                Text("Mock Order 1")
                Text("Mock Order 2")
            }
            .background(Color.yellow) // 方便调试布局
        )
    }
}
```

## 4. 总结

1.  **解耦 View**: 使用 `ViewBuilder` 或 `AnyView` 配合协议，让一个模块可以嵌入另一个模块的 UI，而无需编译时依赖。
2.  **解耦 Logic**: 继续使用 Protocol 定义 Service 和 Repository，以便对 ViewModel 进行单元测试。
3.  **SwiftUI 的变化**: 我们不再需要像 MVVM-C 那样为 View 和 VM 之间的通信定义繁琐的 Delegate 协议，因为 `@Published` 和 Binding 已经解决了这个问题。协议现在更多用于**架构边界**（模块之间、层级之间）。
