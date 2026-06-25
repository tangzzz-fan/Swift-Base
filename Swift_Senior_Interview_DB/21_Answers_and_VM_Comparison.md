# 21. Answers & VM Comparison: UIKit vs SwiftUI

> **Format**: Detailed Q&A + Architecture Comparison
> **Source**: `Swift_Senior_Interview_Questions.md`

---

## Part 1: Detailed Answers to "Soul-Crushing" Questions

### 1. Generics & Type System
**Q: `some Protocol` (Opaque) vs `any Protocol` (Existential)**

*   **Answer**:
    *   `some P`: **反向泛型**。编译器知道具体类型，但对调用者隐藏。它保留了类型标识 (Type Identity)，支持关联类型 (Associated Types)。性能上是静态分发。
    *   `any P`: **类型擦除**。它是一个包装容器 (Box)，可以容纳任何遵循 P 的类型。它丢失了类型标识。性能上是动态分发。
*   **Existential Container**:
    *   当使用 `any P` 时，编译器生成一个固定大小的容器（通常是 3 个字长用于 Value Buffer + 1 个 VWT 指针 + 1 个 PWT 指针）。
    *   如果值太得进 Buffer，就内联存储；否则堆分配并存储指针。
    *   **VWT (Value Witness Table)**: 管理值的生命周期（分配、拷贝、销毁）。
    *   **PWT (Protocol Witness Table)**: 映射协议方法到具体实现。
*   **Swift 5.7 Opening Existentials**:
    *   之前 `Protocol` 不能遵循 `Equatable` 因为 `Self` 类型未知。
    *   Swift 5.7 允许在使用 `any P` 调用泛型函数时，自动将 `any P` 内部的具体类型 "打开" (Open) 并传递给泛型参数 `<T: P>`。

### 2. Memory & Lifecycle
**Q: Copy-on-Write (CoW) Internals**

*   **Answer**:
    *   CoW 不是编译器魔法，而是标准库手动实现的。
    *   **Implementation**: 结构体内部持有一个类实例 (Reference Type) 作为存储。
    *   **Mutation**: 在修改前，调用 `isKnownUniquelyReferenced(&storage)`。
        *   如果返回 `true` (引用计数为 1)，直接修改。
        *   如果返回 `false` (多处引用)，先深拷贝一份 Storage，再修改。
*   **Side Table**:
    *   当对象被 `weak` 引用指向时，或者引用计数溢出时，会创建 Side Table。
    *   对象的 RefCount 字段变为指向 Side Table 的指针。
    *   `Deinit` (逻辑销毁，调用析构函数) -> `Dealloc` (内存释放)。如果还有 `unowned` 引用，对象内存不释放；如果还有 `weak` 引用，Side Table 不释放。

### 3. Method Dispatch
**Q: Dispatch Mechanisms & Ranking**

1.  **Static Dispatch (Direct)**: 最快。编译器硬编码地址。(`final`, `static`, `struct` methods, WMO optimized).
2.  **Table Dispatch (V-Table)**: 中等。通过虚表查找。(`class` methods).
3.  **Message Dispatch (ObjC)**: 最慢。通过 `objc_msgSend` 字符串查找。(KVO, CoreData, `dynamic`).

*   **Extension**: 默认是 Static Dispatch (不能被 override)。除非在 Class Extension 中且有 `@objc` (Message) 或 `@nonobjc` (Static)。
*   **Protocol Extension**:
    *   如果变量类型是 `Protocol` (Existential): 只有协议定义的方法走 PWT (Table)，扩展里的默认实现走 Static。
    *   如果变量类型是 `Concrete`: 优先调用具体实现。

### 4. Concurrency
**Q: Actor Internals**

*   **Thread Safety**: Actor 通过 **Serial Executor** 保证同一时间只有一个 Task 在执行 Actor 的代码。它没有使用锁 (Lock)，而是使用 **Job Queue**。
*   **Reentrancy**:
    *   当 Actor 执行 `await` (挂起) 时，它会释放 Executor 的控制权。其他 Task 可以插入执行。
    *   **Problem**: 挂起前后，Actor 的状态可能被改变。
    *   **Fix**: 在 `await` 后必须重新检查假设条件 (Assumptions)。
*   **Sendable**:
    *   `@Sendable` 闭包：编译器检查捕获的值是否是线程安全的。
    *   `unchecked`: 告诉编译器 "我知道我在做什么，闭嘴"。通常用于底层锁保护的类。

### 5. Identity & Lifecycle (SwiftUI)
**Q: Explicit vs Structural Identity**

*   **Explicit**: 使用 `.id(...)` 或 `Identifiable` 协议。明确告诉 SwiftUI "我是谁"。
*   **Structural**: 基于 View 在代码中的位置 (View Tree Path)。`if/else` 分支会产生不同的结构性 ID。
*   **AnyView**: 抹除了结构性信息。SwiftUI 无法知道 `AnyView` 内部的内容是否改变，通常会导致极其激进的重绘 (Destroy & Recreate)，性能杀手。

### 6. State Management
**Q: StateObject vs ObservedObject**

*   **ObservedObject**: 依赖注入。View 不拥有它。如果 View 重绘 (Re-eval)，`ObservedObject` 只是一个参数，不会变。但如果 View 在 `init` 里创建它 (`ObservedObject(wrappedValue: VM())`)，每次 View 重绘都会创建新的 VM 实例，导致状态丢失。
*   **StateObject**: 只有 View 拥有它。SwiftUI 保证在 View 的整个生命周期内（只要 Identity 不变），只创建一次实例，即使 View struct 被多次重建。

### 7. Layout System
**Q: Layout Negotiation Protocol**

1.  **Parent Proposes**: 父视图给子视图一个建议尺寸 (Proposal)。
2.  **Child Chooses**: 子视图根据建议，决定自己需要多大 (Claim)。
3.  **Parent Places**: 父视图根据子视图的选择，决定把子视图放在哪里 (Position)。

*   **Stubborn Views**: 图片 (`Image`) 默认是 Stubborn 的，它会忽略 Proposal，按原始尺寸显示。除非加 `.resizable()`。
*   **GeometryReader**: 它是一个 "贪婪" 的 View。它会接受父视图给的所有空间 (Propose = Claim)，这可能会破坏原本由内容撑开的布局。

### 8. Animations
**Q: Interpolation**

*   **Animatable**: 协议定义了 `animatableData`。
*   **VectorArithmetic**: 这是一个数学协议，定义了如何对两个值进行加减乘除（插值）。
*   **Custom Shape**: 需要实现 `animatableData` (通常是 `Double` 或 `AnimatablePair`)。系统在每一帧设置这个属性，Shape 的 `path(in:)` 方法利用这个属性绘制中间状态。

---

## Part 2: UIKit MVVM-C vs SwiftUI VM Comparison

### Scenario: User Profile Page (用户个人主页)
*   **Features**:
    1.  显示头像、姓名、简介。
    2.  "Edit" 按钮，点击跳转到编辑页。
    3.  "Follow" 按钮，点击发起网络请求，成功后更新 UI。

### 1. UIKit MVVM-C (Coordinator)

**ViewModel (`ProfileViewModel`)**:
*   **Input**: `func didTapFollow()`, `func didTapEdit()`
*   **Output**: `var onUpdate: (() -> Void)?`, `var onError: ((String) -> Void)?` (或者使用 RxSwift/Combine 的 `Driver`/`Publisher`)。
*   **Navigation**: **不包含**。VM 通过 `delegate` 或 `closure` 通知 Coordinator。
    *   `var coordinatorDelegate: ProfileViewModelCoordinatorDelegate?`
*   **State**: 通常是命令式的。`var user: User`。

**Coordinator (`ProfileCoordinator`)**:
*   **Role**: 负责创建 VM 和 VC，处理跳转。
*   **Code**:
    ```swift
    func showEdit(user: User) {
        let editVM = EditProfileViewModel(user: user)
        let editVC = EditProfileViewController(viewModel: editVM)
        navigationController.pushViewController(editVC, animated: true)
    }
    ```

**ViewController**:
*   **Role**: 绑定 VM 数据到 UI 控件 (Label, Button)。
*   **Binding**: `viewModel.onUpdate = { [weak self] in self?.nameLabel.text = ... }`

---

### 2. SwiftUI MVVM (State-Driven)

**ViewModel (`ProfileViewModel`)**:
*   **Input**: `func follow()` (Intent/Action)
*   **Output**: `@Published var state: ProfileState` (Single Source of Truth).
*   **Navigation**: **包含状态**。
    *   `@Published var presentedRoute: Route?` (或者由父级 Router 控制)。
*   **State**: 声明式的。

**View (`ProfileView`)**:
*   **Role**: 声明 UI 是 State 的函数。
*   **Binding**: 直接绑定。`Text(vm.state.name)`。
*   **Navigation**:
    ```swift
    .navigationDestination(item: $vm.presentedRoute) { route in
        switch route {
        case .edit: EditProfileView()
        }
    }
    ```

### 3. Key Differences & Similarities

| Feature | UIKit MVVM-C | SwiftUI VM |
| :--- | :--- | :--- |
| **View Binding** | 手动 (Closure/KVO/Rx) | 自动 (AttributeGraph, @Published) |
| **Navigation** | **External** (Coordinator 只有逻辑) | **State-Driven** (Navigation 也是 State 的一部分) |
| **Life Cycle** | VM 通常与 VC 存活时间一致 | VM (`StateObject`) 存活于 View Identity 期间 |
| **State Granularity** | 细粒度 (nameString, ageString) | 粗粒度 (ViewState struct) 推荐 |
| **Dependency Injection** | Initializer Injection (in Coordinator) | Environment / Initializer |

### 4. Why the Shift?
在 SwiftUI 中，**Navigation is just another piece of State**.
*   UIKit: "Push this ViewController" (Imperative Action).
*   SwiftUI: "The Navigation Stack contains [Home, Profile, Edit]" (Declarative State).

因此，传统的 Coordinator (作为一个持有 NavController 的 Object) 变得尴尬。在 SwiftUI 中，Coordinator 演变成了 **Router Object** (一个持有 `[Route]` 数组的 `ObservableObject`)。
