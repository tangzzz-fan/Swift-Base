# SwiftUI App 架构：模块化与解耦深入指南

本文档深入探讨在大型 SwiftUI App 开发中，如何进行有效的模块划分、协议定义以及解耦。我们将从整体架构视角出发，结合实际场景（如电商 App）进行说明。

## 1. 功能模块定义与划分

在 SwiftUI 时代，模块化依然是大型 App 的基石。与 UIKit 时代相比，SwiftUI 的声明式特性使得“垂直切分”变得更加自然。

### 1.1 模块划分策略：垂直切分 (Vertical Slicing)

传统的 MVVM-C 往往倾向于“水平切分”（所有 Model 在一起，所有 VM 在一起）。但在现代 SwiftUI 开发中，推荐**按功能特性（Feature）垂直切分**。

**推荐的模块结构 (SPM Packages 或 Frameworks):**

*   **App (Main Target)**: 仅仅是胶水层，负责组装各个 Feature。
*   **Feature Modules (业务层)**:
    *   `FeatureHome`: 首页相关的所有 View, VM, Logic。
    *   `FeatureProduct`: 商品详情, 搜索。
    *   `FeatureCart`: 购物车, 结算。
    *   `FeatureProfile`: 用户中心, 设置。
*   **Core Modules (基础层)**:
    *   `CoreUI`: 设计系统 (Colors, Fonts, Common Components)。
    *   `CoreNetwork`: 网络请求基础封装。
    *   `CoreModel`: 全局共享的数据模型 (User, Product)。

### 1.2 如何定义模块接口 (Module API)

为了实现模块间的解耦，一个 Feature 模块不应该直接暴露其内部的 View 类型，而是应该暴露**构建器 (Builder)** 或 **协议 (Protocol)**。

**场景**: `App` 需要显示 `FeatureHome` 的入口视图。

**❌ 强耦合方式 (直接引用)**:
```swift
import FeatureHome
// App 直接依赖 FeatureHome 的具体 View 类型
WindowGroup {
    HomeView() 
}
```

**✅ 解耦方式 (协议/工厂模式)**:

在 `CoreNavigation` 或独立的 `FeatureInterface` 模块中定义协议：

```swift
// 定义在公共接口层
public protocol HomeFeatureProvider {
    func makeHomeView() -> AnyView // 或者使用 some View (如果支持 opaque types in protocol)
}
```

在 `FeatureHome` 模块中实现：
```swift
public struct HomeFeature: HomeFeatureProvider {
    public init() {}
    public func makeHomeView() -> AnyView {
        return AnyView(HomeView())
    }
}
```

---

## 2. 模块间的解耦与通信

在 SwiftUI 中，模块间最常见的耦合发生在**导航 (Navigation)** 和 **数据共享 (Data Sharing)**。

### 2.1 导航解耦 (Navigation Decoupling)

**问题**: `HomeView` (在 Home 模块) 点击商品需要跳转到 `ProductDetailView` (在 Product 模块)。如果直接写 `NavigationLink(destination: ProductDetailView())`，则 Home 模块必须依赖 Product 模块，导致紧耦合。

**解决方案: 路由协议 (Router Protocol)**

1.  **定义路由协议**:
    Home 模块定义它需要的外部能力。
    ```swift
    // FeatureHome/HomeRouter.swift
    public protocol HomeRouter {
        func routeToProductDetail(id: String) -> AnyView
    }
    ```

2.  **注入路由**:
    HomeView 依赖这个协议，而不是具体的 Product 模块。
    ```swift
    struct HomeView: View {
        let router: HomeRouter // 依赖注入
        
        var body: some View {
            Button("Buy") {
                // 使用 router 获取目标视图，HomeView 不知道 ProductDetailView 的存在
                let destination = router.routeToProductDetail(id: "123")
                // ... 进行跳转逻辑
            }
        }
    }
    ```

3.  **在 App 层组装**:
    App Target 能够看到所有模块，负责实现 Router。
    ```swift
    struct AppRouter: HomeRouter {
        func routeToProductDetail(id: String) -> AnyView {
            // 这里 App 知道 FeatureProduct
            return AnyView(FeatureProduct.makeDetailView(id: id))
        }
    }
    ```

### 2.2 数据解耦 (Data Decoupling)

**问题**: `FeatureCart` 需要监听 `FeatureProfile` 中的登录状态。

**解决方案: 共享环境对象 (Shared Environment Object)**

不要让 Cart 模块直接依赖 Profile 模块的 ViewModel。而是将共享状态下沉到 `CoreModel` 或 `CoreAuth` 模块。

1.  **CoreAuth 模块**: 定义 `UserSession`。
2.  **FeatureProfile**: 修改 `UserSession`。
3.  **FeatureCart**: 监听 `UserSession`。

```swift
// FeatureCart/CartView.swift
import CoreAuth

struct CartView: View {
    @EnvironmentObject var session: UserSession // 依赖基础模块，而非业务模块
    
    var body: some View {
        if session.isLoggedIn {
            CheckoutButton()
        } else {
            LoginPrompt()
        }
    }
}
```

---

## 3. 总结：如何为不同模块定义协议

1.  **入站协议 (Inbound Protocols)**:
    *   **目的**: 让外部能够调用本模块的功能（通常是获取 View）。
    *   **定义位置**: 独立的 Interface 模块，或本模块的 Public API。
    *   **形式**: `Provider` 或 `Factory` 模式。
    *   **例子**: `ProductFeatureProvider.makeProductList()`

2.  **出站协议 (Outbound Protocols)**:
    *   **目的**: 本模块需要跳转到外部，或获取外部数据。
    *   **定义位置**: **本模块内部** (Dependency Inversion Principle)。
    *   **形式**: `Delegate` 或 `Router` 模式。
    *   **例子**: `HomeView` 定义了 `HomeRouter`，要求外部必须实现这个 Router 才能使用 HomeView。

通过这种方式，`FeatureHome` 既不依赖 `FeatureProduct`，也不依赖 `App`，它只依赖自己定义的协议和基础的 `Core` 模块。这就是极致的解耦。
