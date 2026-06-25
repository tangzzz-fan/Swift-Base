# 04. SwiftUI Internals

> **Difficulty**: Expert
> **Focus**: AttributeGraph, Identity, Layout Protocol, Performance

---

## 1. AttributeGraph & Dependency Tracking

### Q1: 深入解释 SwiftUI 的 Diff 算法和依赖追踪机制 (AttributeGraph)。
**Follow-up**:
1.  当一个 `@State` 发生变化时，SwiftUI 是如何知道哪些 View 需要重新求值 (Re-evaluate) 的？
2.  `AttributeGraph` 是什么？它在内存中是什么样子的？
3.  为什么将复杂的计算逻辑放在 `body` 属性中会严重影响性能？

#### Key Answer:
*   **AttributeGraph (AG)**: SwiftUI 的私有 C++ 核心库。它维护了一个庞大的依赖图。
*   **Node**: 每个 View, Modifier, State, Preference 都是图中的一个节点。
*   **Update Loop**:
    1.  User Action -> State Change.
    2.  Invalidate AG Node.
    3.  AG 遍历受影响的下游节点（Dirty Region）。
    4.  调用 View 的 `body` 生成新的 View 结构（Value Type）。
    5.  对比新旧结构（Diff），更新 Render Tree (UIKit/CoreAnimation)。
*   **Performance**: `body` 被调用的频率极高。如果 `body` 里有耗时操作，会阻塞主线程。AG 的优势在于它能剪枝（Pruning），如果父 View 的输入没变，就不会重新计算子 View。

---

## 2. Identity (Identity of Views)

### Q2: 详细区分 Explicit Identity (显式标识) 和 Structural Identity (结构性标识)。
**Follow-up**:
1.  为什么 `AnyView` 会破坏 Structural Identity？这会导致什么后果？(State丢失, 动画失效, 性能下降).
2.  在 `if-else` 分支中，SwiftUI 如何判断两个分支的 View 是同一个还是不同的？
3.  `id(_:)` modifier 实际上做了什么？

#### Key Answer:
*   **Structural Identity**: 根据 View 在代码结构中的位置（路径）来确定身份。例如 `VStack { Text("A"); Text("B") }`，第一个 Text 是 "VStack.0"，第二个是 "VStack.1"。
*   **Explicit Identity**: 使用 `.id(...)` 或 `ForEach` 的 `id` 参数手动指定。
*   **AnyView**: 它是 "Type Erasure"。它隐藏了 View 的具体类型结构。SwiftUI 无法透过 `AnyView` 看到内部结构，因此每次更新往往被视为"全新的 View"（Destroy old -> Create new），导致 `@State` 重置，动画中断。

---

## 3. Layout System (Layout Negotiation)

### Q3: 描述 SwiftUI 的布局协商协议 (Layout Protocol)。
**Follow-up**:
1.  口诀: "Parent proposes, Child chooses, Parent places." 请解释这三步。
2.  `frame(width:height:)` 是一个 View 吗？它对子 View 提议什么？它对父 View 报告什么？
3.  `GeometryReader` 的布局行为有什么特殊之处？它为什么被认为是 "Layout Neutral" 的破坏者？

#### Key Answer:
1.  **Propose**: 父 View 给子 View 一个建议尺寸 (Proposal: min, max, ideal)。
2.  **Choose**: 子 View 根据建议和自身内容，决定自己的需求尺寸 (Required Size)。
3.  **Place**: 父 View 根据子 View 的需求尺寸，将其放置在坐标系中。
*   **GeometryReader**: 它极其贪婪。它会接受父 View 提议的全部空间（尽可能大），然后把这个尺寸暴露给子 View。这往往会导致布局意外撑大。

---

## 4. Animation & Transactions

### Q4: SwiftUI 的动画系统是如何工作的？`Transaction` 是什么？
**Follow-up**:
1.  `withAnimation` 实际上修改了什么？(它设置了当前线程的 `Transaction` 栈)。
2.  如何打断一个正在进行的动画？(Retargeting / Velocity preservation).
3.  `Animatable` 协议的 `animatableData` 如果是 `EmptyAnimatableData` 会发生什么？

#### Key Answer:
*   **Transaction**: 包含动画上下文（Animation curve, duration）的容器。当状态改变时，Transaction 会随之传递。
*   **Interpolation**: SwiftUI 并不直接动画 View，而是动画 View 的**数据** (`AnimatableData`)。系统在每一帧多次调用 `body`（或者更底层的渲染层），传入插值后的数据。
