# 10. SwiftUI & Combine Architecture (The Book Edition)

> **Source**: Based on concepts from *App Architecture* (objc.io) and Combine patterns.
> **Focus**: Publishers, Data Flow, Side Effects, ObservableObject, Architecture Patterns.

---

## 1. Combine Basics & Publishers

### Q1: Hot vs Cold Publishers
**Q**: `URLSession.dataTaskPublisher` 是 Hot 还是 Cold？`CurrentValueSubject` 呢？区别是什么？
*   **Follow-up**: `share()` 操作符如何将 Cold 信号转为 Hot 信号？
*   **Answer**:
    *   **Cold**: 只有被订阅 (Subscribe) 时才开始工作（发请求）。`URLSession` 是 Cold。
    *   **Hot**: 无论有无订阅者，都在产生数据。`CurrentValueSubject` 是 Hot（它持有状态）。
    *   `share()`: 让多个订阅者共享同一个上游订阅 (Subscription)。上游只执行一次，数据分发给多个下游。

### Q2: Backpressure (背压)
**Q**: Combine 如何处理背压？`Subscriber` 协议中的 `request(_ demand:)` 是做什么的？
*   **Follow-up**: 如果 Publisher 发送数据的速度快于 Subscriber 处理的速度，会发生什么？(默认是 Drop 还是 Buffer?)
*   **Answer**:
    *   Combine 是基于 **Pull Model** 的。Subscriber 告诉 Publisher "我能处理 N 个数据"。Publisher 发送的数据不能超过 N。
    *   默认行为：如果不处理背压，Publisher 可能会遵守 Demand 停止发送，或者某些操作符（如 `buffer`）会缓存。如果 buffer 满了，根据策略 Drop 或 Error。

### Q3: AnyCancellable & Memory Management
**Q**: 为什么必须持有 `AnyCancellable`？如果不持有会发生什么？
*   **Follow-up**: `store(in: &set)` 的底层实现是什么？它是线程安全的吗？
*   **Answer**:
    *   Combine 的 Subscription 是基于 RAII (Resource Acquisition Is Initialization) 的。`AnyCancellable` 销毁时会自动调用 `cancel()`。
    *   如果不持有，`AnyCancellable` 立即销毁，Subscription 立即取消，数据流中断。
    *   `store(in:)`: 简单的集合操作。`Set<AnyCancellable>` 不是线程安全的，多线程操作需要加锁。

---

## 2. SwiftUI Integration

### Q4: ObservableObject & @Published
**Q**: `@Published` 属性包装器是如何触发 `objectWillChange` 的？
*   **Follow-up**: 为什么是 `willChange` 而不是 `didChange`？这对于 SwiftUI 的 Diff 算法有什么重要意义？
*   **Answer**:
    *   `@Published` 在 `willSet` 中调用 `objectWillChange.send()`。
    *   **Coalescing**: SwiftUI 需要在变更**发生前**知道，以便在当前 RunLoop 周期内合并多次变更，只进行一次 View Update。如果是 `didChange`，可能 View 已经渲染了旧值，或者错过了合并时机。

### Q5: assign(to:) Memory Cycle
**Q**: `assign(to: \.value, on: self)` 会导致循环引用吗？为什么？
*   **Follow-up**: 如何解决？(Swift 5.7 之前的 `assign(to:on:)` vs `assign(to:)` with `@Published`)。
*   **Answer**:
    *   会。Subscription 强引用 `self` (作为 Root)，`self` 持有 Subscription (Cancellable)。
    *   **Fix**: 使用 `sink { [weak self] in self?.value = $0 }`。
    *   **Note**: `assign(to: &$publishedProperty)` (SwiftUI 扩展) 是安全的，因为它内部处理了生命周期绑定。

### Q6: StateObject vs ObservedObject (Architecture View)
**Q**: 从架构角度看，什么时候该用 `StateObject`，什么时候该用 `ObservedObject`？
*   **Follow-up**: 如果我在 `NavigationLink` 的目标 View 中使用 `StateObject`，这个对象的生命周期是怎样的？
*   **Answer**:
    *   `StateObject`: **Ownership**。View 拥有这个对象。对象的生命周期与 View 的生命周期绑定。
    *   `ObservedObject`: **Dependency**。View 只是观察这个对象，不拥有它。
    *   `NavigationLink`: 即使 View 还没 Push 进去，如果初始化是在 Parent 的 body 里，`StateObject` 可能会被提前初始化（SwiftUI 优化细节），但通常只有 View 真正显示时才建立状态连接。

---

## 3. Side Effects & Architecture

### Q7: Side Effects in Reducers (TCA style)
**Q**: 在纯函数式架构 (如 TCA) 中，如何处理 Side Effects (API 请求, 磁盘 IO)？
*   **Follow-up**: `Effect` 类型本质上是什么？(Publisher 的包装)。
*   **Answer**:
    *   Reducer 必须是纯函数。不能直接做 Side Effect。
    *   Reducer 返回一个 `Effect`。系统 (Store) 在 Reducer 执行完后，运行这个 `Effect`，并将结果 (`Action`) 再次发送回 Reducer。

### Q8: MVVM-C with SwiftUI
**Q**: 在 SwiftUI 中实现 Coordinator 模式有什么困难？
*   **Follow-up**: `NavigationStack` (iOS 16+) 如何简化了 Coordinator 的实现？
*   **Answer**:
    *   困难：SwiftUI 的导航 (`NavigationLink`) 是声明式且绑定在 View 树上的。Coordinator 需要命令式地控制导航 (`push`, `pop`)。
    *   `NavigationStack`: 允许通过绑定一个 `path` (Array) 来编程控制导航栈。Coordinator 只需要管理这个 Array 即可。

### Q9: Dependency Injection
**Q**: 在 SwiftUI 中，如何优雅地注入依赖？(Environment vs Init Injection)。
*   **Follow-up**: 为什么说过度使用 `EnvironmentObject` 是 "隐式依赖" (Implicit Dependency)？
*   **Answer**:
    *   `Environment`: 方便，穿透层级。但类型不安全（Crash risk），且依赖不透明。
    *   `Init`: 类型安全，依赖明确。但层级深时需要层层传递 (Prop Drilling)。
    *   **Balance**: 核心服务 (UserSession, Network) 用 Environment，Feature 具体的 ViewModel 用 Init。

---

## 4. Advanced Combine Operators

### Q10: flatMap vs map
**Q**: `flatMap` 在 Combine 中通常用于什么场景？
*   **Follow-up**: `flatMap` 会导致 "Stream Switching" 吗？`maxPublishers` 参数的作用？
*   **Answer**:
    *   `flatMap`: 将上游的值转换为一个新的 Publisher。常用于链式网络请求 (Login -> Get User Profile)。
    *   **Switching**: 是的。它订阅新的 Publisher。
    *   `maxPublishers`: 限制并发数。`.max(1)` 类似于 RxJava 的 `concatMap` (串行)。

### Q11: debounce vs throttle
**Q**: 搜索框输入场景，应该用 `debounce` 还是 `throttle`？区别是什么？
*   **Follow-up**: `throttle` 的 `latest` 参数设为 true 和 false 有什么区别？
*   **Answer**:
    *   **Debounce** (防抖): 停止输入 N 秒后才发送。适合搜索。
    *   **Throttle** (节流): 每隔 N 秒发送一次。适合点击事件防止连击。

### Q12: combineLatest vs zip
**Q**: `combineLatest` 和 `zip` 的区别？
*   **Follow-up**: 如果 Stream A 发了 100 个值，Stream B 发了 1 个值，`zip` 会输出几个值？`combineLatest` 呢？
*   **Answer**:
    *   `zip`: 一一配对。等待两个流都产生新值。输出 1 个值。
    *   `combineLatest`: 任一流动，取另一个流的最新值组合。输出 1 个值 (假设 B 是最后发的) 或 100 个值 (如果 B 先发，A 后发)。

---

## 5. Debugging & Testing

### Q13: Debugging Streams
**Q**: 如何调试 Combine Stream？`print()` 操作符输出了什么？
*   **Follow-up**: `handleEvents` 的作用？
*   **Answer**:
    *   `print()`: 输出 Subscription 生命周期事件 (receive subscription, request demand, receive value, completion)。
    *   `handleEvents`: 允许在各个生命周期插入 Side Effect (logging, breakpoints) 而不改变数据流。

### Q14: Testing Publishers
**Q**: 如何单元测试一个异步的 Publisher？
*   **Follow-up**: `XCTest` 中的 `expectation` 如何配合 Combine 使用？
*   **Answer**:
    *   使用 `XCTestExpectation`。
    *   在 `sink` 的 `receiveValue` 或 `receiveCompletion` 中调用 `expectation.fulfill()`。
    *   `waitForExpectations(timeout:)`。

### Q15: Custom Publisher
**Q**: 实现一个自定义 Publisher 需要遵循哪些步骤？
*   **Follow-up**: 为什么通常推荐使用 `Subject` 或现有操作符组合，而不是自己实现 Publisher 协议？
*   **Answer**:
    *   需要实现 `receive(subscriber:)`。
    *   需要创建一个遵循 `Subscription` 协议的类，处理 Demand 和 Cancellation。
    *   **难点**: 正确处理背压 (Backpressure) 和线程安全非常困难，容易出 Bug。推荐用 `PassthroughSubject` 或 `CurrentValueSubject`。
