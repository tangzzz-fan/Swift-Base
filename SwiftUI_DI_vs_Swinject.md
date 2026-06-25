# SwiftUI 依赖注入 (Environment) vs Swinject (MVVM-C)

本文档旨在通过 5 个典型场景，对比 SwiftUI 原生依赖注入方式（主要是 `Environment` 和 `EnvironmentObject`）与传统 MVVM-C 架构中使用 Swinject 的差异，并探讨在 SwiftUI 开发中是否仍需引入第三方 DI 框架。

## 核心观念转变

*   **Swinject (MVVM-C)**: 显式注册 (Container) -> 显式解析 (Resolver)。通常在 Coordinator 或 Factory 中组装，通过 Init 注入传递给 ViewModel。
*   **SwiftUI (Environment)**: 声明式注入。数据“流”向视图树，子视图按需“抓取”。系统负责数据的传递和更新。

---

## 场景 1: 全局用户会话 (User Session / Auth)

**场景描述**: App 启动后，需要判断用户是否登录，并在整个 App 中访问当前用户信息。

### 🔴 传统 Swinject 方式
1.  定义 `AuthenticationService` 协议。
2.  在 `AppContainer` 中注册单例 `container.register(AuthenticationService.self) { _ in AuthServiceImpl() }.inObjectScope(.container)`。
3.  在 `AppCoordinator` 或 `LoginViewModel` 中 `resolve` 出来。
4.  **痛点**: 需要手动层层传递，或者在每个 VM 中重复 resolve。

### 🟢 SwiftUI Environment 方式
利用 `@EnvironmentObject` 或 `@Environment` 注入一个 `ObservableObject`。

```swift
// 1. 定义可观察对象
class UserSession: ObservableObject {
    @Published var currentUser: User?
    @Published var isLoggedIn: Bool = false
}

// 2. 在 App 入口注入 (Root)
@main
struct MyApp: App {
    @StateObject var session = UserSession() // Source of Truth

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(session) // 注入到环境中
        }
    }
}

// 3. 在任意深度的子视图中使用
struct ProfileView: View {
    // 自动从环境中获取，无需手动传递
    @EnvironmentObject var session: UserSession 

    var body: some View {
        if session.isLoggedIn {
            Text("Welcome, \(session.currentUser?.name ?? "")")
        } else {
            LoginButton()
        }
    }
}
```
**解决原理**: SwiftUI 维护了一个环境树，数据自动向下流动。只要在祖先节点注入，任何后代节点都能直接获取，消除了“构造函数传递地狱”。

---

## 场景 2: 静态配置与主题 (Theming / Configuration)

**场景描述**: App 需要统一的字体、颜色主题，或者 Feature Flag 配置。

### 🔴 传统 Swinject 方式
注册 `ThemeManager`，在每个 ViewController 的 `viewDidLoad` 中 resolve 并设置 UI 样式。

### 🟢 SwiftUI Environment 方式
利用自定义 `EnvironmentKey`。这非常适合“只读”或“配置型”数据。

```swift
// 1. 定义 Key
private struct ThemeColorKey: EnvironmentKey {
    static let defaultValue: Color = .blue // 默认值
}

// 2. 扩展 EnvironmentValues
extension EnvironmentValues {
    var themeColor: Color {
        get { self[ThemeColorKey.self] }
        set { self[ThemeColorKey.self] = newValue }
    }
}

// 3. 使用
struct CustomButton: View {
    @Environment(\.themeColor) var themeColor // 读取

    var body: some View {
        Button("Click Me") {}
            .background(themeColor)
    }
}

// 4. 注入 (可以在任意层级覆盖)
ContentView()
    .environment(\.themeColor, .red) // 覆盖为红色
```
**解决原理**: 类似于 CSS 的层叠样式表。系统提供了类型安全的 Key-Value 存储，且支持层级覆盖（Override）。Swinject 很难做到这种“局部覆盖全局”的灵活配置。

---

## 场景 3: 数据上下文 (CoreData / SwiftData)

**场景描述**: 访问数据库上下文 `NSManagedObjectContext` 或 `ModelContext`。

### 🔴 传统 Swinject 方式
注册 `CoreDataStack`，将 `context` 注入到 `Repository`，再注入到 `ViewModel`。

### 🟢 SwiftUI Environment 方式
SwiftUI 原生集成了 CoreData/SwiftData 的环境注入。

```swift
// SwiftData 示例
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: [Item.self]) // 系统自动注入 ModelContext
    }
}

struct ItemListView: View {
    // 直接获取 Context
    @Environment(\.modelContext) private var modelContext 
    // 甚至直接查询数据 (Query 也是一种特殊的注入)
    @Query var items: [Item] 

    var body: some View {
        List(items) { item in
            Text(item.name)
        }
    }
}
```
**解决原理**: 苹果官方将数据库上下文标准化为环境值。这比手动管理单例或依赖注入容器更加无缝且线程安全（配合 `@MainActor`）。

---

## 场景 4: API 服务与网络层 (Service Injection)

**场景描述**: 这里的重点是**解耦**和**Mock**。例如 `APIService`。

### 🔴 传统 Swinject 方式
```swift
container.register(APIServiceProtocol.self) { _ in RealAPIService() }
// 测试时
container.register(APIServiceProtocol.self) { _ in MockAPIService() }
```

### 🟢 SwiftUI Environment 方式
同样可以使用 `Environment` 注入 Protocol 类型（需稍作封装）。

```swift
// 1. 定义协议和 Key
protocol APIService {
    func fetch() async throws -> Data
}

struct APIServiceKey: EnvironmentKey {
    static let defaultValue: any APIService = RealAPIService()
}

extension EnvironmentValues {
    var apiService: any APIService {
        get { self[APIServiceKey.self] }
        set { self[APIServiceKey.self] = newValue }
    }
}

// 2. 视图中使用
struct DataView: View {
    @Environment(\.apiService) var apiService

    func loadData() async {
        let data = try? await apiService.fetch()
    }
}

// 3. 预览或测试时注入 Mock
#Preview {
    DataView()
        .environment(\.apiService, MockAPIService())
}
```
**解决原理**: SwiftUI 的 `.environment` modifier 本质上就是一个轻量级的、基于视图层级的 DI 容器。它天然支持“在不同层级/不同场景（如 Preview）替换实现”。

---

## 场景 5: 深度导航与参数传递 (Deep Hierarchy)

**场景描述**: 列表页 -> 详情页 -> 编辑页 -> 设置页。设置页需要访问列表页的某个状态。

### 🔴 传统 Swinject 方式
Coordinator 模式。Coordinator 持有所有 VM 的创建逻辑，负责将数据从 VM A 传递给 VM B，再传递给 VM C。代码量大，逻辑集中在 Coordinator。

### 🟢 SwiftUI Environment 方式
配合 `NavigationStack`，数据依然可以通过 Environment 穿透。

```swift
struct RootView: View {
    @StateObject var flowState = FlowState() // 跨页面的状态

    var body: some View {
        NavigationStack {
            List {
                NavigationLink("Go Deep", destination: DetailView())
            }
        }
        .environmentObject(flowState) // 注入一次
    }
}

struct DetailView: View {
    // 中间层不需要知道 flowState
    var body: some View {
        NavigationLink("Edit", destination: EditView())
    }
}

struct EditView: View {
    @EnvironmentObject var flowState: FlowState // 直接获取

    var body: some View {
        Toggle("Enable Feature", isOn: $flowState.isEnabled)
    }
}
```
**解决原理**: 只要视图在同一个 `NavigationStack` 或视图树分支下，Environment 就能穿透中间层。这消除了 Coordinator 中大量的“传声筒”代码。

---

## 讨论: SwiftUI 开发中是否还需要 Swinject?

### 结论
**对于 90% 的中小型 SwiftUI 纯原生应用，不需要 Swinject。**
SwiftUI 的 `Environment` + `ObservableObject/StateObject` 已经构成了一套完整的依赖注入和状态管理系统。

### 什么时候还需要 Swinject? (典型场景)

1.  **复杂的非视图逻辑 (Headless Logic)**:
    *   如果你的 App 有大量的后台处理逻辑、复杂的 UseCase 层、Repository 层，且这些层**完全不依赖 UI**。
    *   Environment 依赖于 View 树。如果你需要在 `AppDelegate`、后台任务、或者纯逻辑类之间进行复杂的依赖组装，Environment 鞭长莫及。Swinject 在这里依然是王者。

2.  **混合开发 (UIKit + SwiftUI)**:
    *   如果你的 App 是重度混合，且主要架构依然是 MVVM-C (UIKit 主导)。为了保持架构一致性，继续使用 Swinject 管理 ViewModel 的创建，然后将创建好的 VM 注入给 SwiftUI View 是合理的。

3.  **循环依赖与复杂的生命周期管理**:
    *   SwiftUI 的 Environment 主要处理“共享”和“层级传递”。对于复杂的对象图（Object Graph）构建（例如 A 依赖 B，B 依赖 C，C 又依赖 A 的某个 Delegate），Swinject 的 `Container` 和 `Scope` (Graph, Weak, Container) 提供了更精细的控制。

4.  **单元测试 (非 UI 测试)**:
    *   对纯逻辑类进行单元测试时，使用 Swinject 解析 Mock 对象可能比手动构造更方便，但这取决于个人习惯。

### 总结建议
*   **新开 SwiftUI 项目**: 优先使用 `Environment` 和 `EnvironmentObject`。不要引入 Swinject，除非你发现逻辑层极其复杂且脱离 UI。
*   **老项目迁移**: 如果已有 Swinject，可以保留它用于生成 ViewModel，然后通过 `@StateObject` 或 `init` 注入到 SwiftUI 视图中。
