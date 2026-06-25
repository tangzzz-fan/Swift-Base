# UIKit 开发中使用 SwiftUI 属性包装器 (@Environment 等)

本文档回答以下问题：**在传统的 UIKit 开发中，`@Environment`, `@StateObject`, `@ObservedObject` 等属性包装器能否正常使用？**

## 简短回答
**不能直接使用。**
这些属性包装器（Property Wrappers）是 SwiftUI 框架的一部分，它们依赖于 SwiftUI 的 `View` 结构体及其底层的声明式更新机制（Graph update cycle）。UIKit 的 `UIViewController` 和 `UIView` 缺乏这种运行时支持。

---

## 详细解析

### 1. 为什么不能直接用？

*   **依赖视图树**: `@Environment` 依赖于 SwiftUI 的视图层级树（Environment Values 沿着树向下传递）。UIKit 没有这个树结构，`UIViewController` 只是视图控制器，没有“环境”的概念。
*   **依赖重绘机制**: `@State`, `@Published` 等旨在触发 SwiftUI 的 `body` 重新计算。UIKit 是命令式的，需要手动调用 `setNeedsLayout` 或更新 UI 属性（如 `label.text = "new"`）。属性包装器改变了数据，但 UIKit 不知道该去刷新哪个 Label。

**错误示例**:
```swift
class MyViewController: UIViewController {
    // ❌ 编译可能通过，但运行时无效或崩溃
    // UIKit 不会为你注入这个值，它永远是 nil 或默认值
    @Environment(\.colorScheme) var colorScheme 
    
    // ❌ 改变这个值不会触发任何 UI 刷新
    @State var count = 0 
}
```

---

## 解决方案与替代策略

虽然不能直接用，但我们有几种方式在 UIKit 中实现类似的效果或进行桥接。

### 方案 A: 使用 `UIHostingController` (官方桥接)

这是最推荐的方式。如果你想用 SwiftUI 的特性，就直接把那一小块 UI 变成 SwiftUI，然后嵌入到 UIKit 中。

*   **原理**: `UIHostingController` 是一个 `UIViewController`，它持有一个 SwiftUI `RootView`。它充当了 UIKit 和 SwiftUI 之间的适配器。
*   **环境注入**: 你可以在创建 `UIHostingController` 时注入环境。

```swift
// SwiftUI 视图
struct SwiftUIView: View {
    @EnvironmentObject var session: UserSession
    var body: some View { ... }
}

// UIKit 代码
let session = UserSession()
let swiftUIView = SwiftUIView().environmentObject(session) // ✅ 在这里注入
let hostingVC = UIHostingController(rootView: swiftUIView)

// 将 hostingVC 添加到 UIKit 视图层级
addChild(hostingVC)
view.addSubview(hostingVC.view)
hostingVC.didMove(toParent: self)
```

### 方案 B: 模拟实现 (自定义 Property Wrapper)

如果你非常喜欢 `@Injected` 这种写法，可以在 UIKit 中自己实现一个简单的依赖注入包装器（Service Locator 模式）。这**不是** SwiftUI 的 `@Environment`，但语法看起来很像。

```swift
// 1. 定义一个简单的容器 (Service Locator)
class DIContainer {
    static let shared = DIContainer()
    var services: [String: Any] = [:]
    
    func register<T>(_ service: T, for type: T.Type) {
        let key = String(describing: type)
        services[key] = service
    }
    
    func resolve<T>() -> T? {
        let key = String(describing: T.self)
        return services[key] as? T
    }
}

// 2. 定义属性包装器
@propertyWrapper
struct Injected<T> {
    var wrappedValue: T
    
    init() {
        guard let value: T = DIContainer.shared.resolve() else {
            fatalError("Service \(T.self) not registered!")
        }
        self.wrappedValue = value
    }
}

// 3. 在 UIKit 中使用
class MyViewController: UIViewController {
    // ✅ 看起来像 SwiftUI，但其实是纯 Swift 实现
    @Injected var authService: AuthenticationService 
    
    override func viewDidLoad() {
        super.viewDidLoad()
        authService.login()
    }
}
```

### 方案 C: Combine (响应式绑定)

虽然不能用 `@State` 自动刷新，但可以用 Combine 框架来实现数据驱动 UI。这是 UIKit 中最现代化的做法。

```swift
import Combine

class MyViewModel {
    @Published var userName: String = "" // 使用 Combine 的 @Published
}

class MyViewController: UIViewController {
    var viewModel = MyViewModel()
    var cancellables = Set<AnyCancellable>()
    
    @IBOutlet weak var nameLabel: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // 手动绑定数据变化到 UI
        viewModel.$userName
            .receive(on: RunLoop.main)
            .sink { [weak self] newName in
                self?.nameLabel.text = newName
            }
            .store(in: &cancellables)
    }
}
```

## 总结

1.  **不要尝试**在 `UIViewController` 或 `UIView` 中直接使用 `@Environment`, `@EnvironmentObject`, `@State`。它们是为 SwiftUI 视图树设计的。
2.  **桥接使用**: 如果需要使用依赖环境的 SwiftUI 视图，请将其包裹在 `UIHostingController` 中，并在初始化时通过 `.environment()` 修改器注入依赖。
3.  **UIKit 原生替代**: 在纯 UIKit 代码中，继续使用 **依赖注入 (Init Injection)** 或 **自定义属性包装器 (@Injected)** 来管理依赖；使用 **Combine** 来管理状态绑定。
