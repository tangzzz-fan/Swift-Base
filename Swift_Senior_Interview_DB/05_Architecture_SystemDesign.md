# 05. Architecture & System Design

> **Difficulty**: Senior / Staff
> **Focus**: Modularization, Design Patterns, Dependency Injection, Scalability

---

## 1. Modularization (组件化/模块化)

### Q1: 设计一个大型 App 的组件化架构。如何处理组件间的通信和依赖？
**Follow-up**:
1.  **Static vs Dynamic Frameworks**: 什么时候应该用静态库，什么时候用动态库？它们对 App 启动时间有什么影响？
2.  **Dependency Hell**: 如何解决菱形依赖 (Diamond Dependency) 问题？
3.  **Router**: 路由模式 (URL Router vs Protocol Router) 的优缺点对比。

#### Key Answer:
*   **Communication**:
    *   **Protocol-Oriented**: 定义 Public Protocol 放在单独的 Interface Module 中。Feature Module 依赖 Interface Module，不互相依赖。
    *   **Dependency Injection (DI)**: 使用 DI 容器在 App 启动时组装具体实现。
*   **Static vs Dynamic**:
    *   **Static**: 编译时链接，代码拷贝到主二进制。启动快（无 dyld 加载开销），但包体积可能大（如果多个动态库依赖同一个静态库且未剥离）。
    *   **Dynamic**: 运行时加载。启动慢（dyld rebase/bind），但可以共享代码，支持插件化。Apple 建议尽量多用 Static Library (Mergeable Libraries in Xcode 15+)。

---

## 2. Design Patterns (MVVM-C, TCA, VIPER)

### Q2: 深入对比 MVVM-C (Coordinator) 和 TCA (The Composable Architecture)。
**Follow-up**:
1.  **Coordinator**: 解决了什么问题？(解耦 Navigation 和 ViewController)。
2.  **TCA**: 它的核心思想是什么？(单向数据流, Reducer, Effect)。它最大的性能瓶颈通常在哪里？(Store 的频繁更新导致整个 View 树的无效 Diff，虽然有 `WithViewStore` 优化)。
3.  **Massive View Controller**: 如果不使用 MVVM，仅靠 MVC 如何给 VC 瘦身？(Child View Controllers, Logic extraction to plain objects).

#### Key Answer:
*   **MVVM-C**: 工业界标准。ViewModel 处理状态，Coordinator 处理跳转。优点是灵活，学习曲线平滑。缺点是状态管理可能分散。
*   **TCA**: 严格的单向数据流。优点是可测试性极强（State, Action, Reducer 都是纯函数），状态可回溯。缺点是样板代码多，性能优化难度大（对于超大型列表）。

---

## 3. Dependency Injection (依赖注入)

### Q3: 为什么说 Service Locator (单例模式的变种) 是反模式？真正的依赖注入应该怎么做？
**Follow-up**:
1.  **Constructor Injection** vs **Property Injection**。
2.  Swift 5.1 的 `@propertyWrapper` 如何简化 DI？
3.  如何设计一个线程安全的 DI 容器？

#### Key Answer:
*   **Service Locator**: 隐藏了依赖关系。使用者必须知道去哪里找 Service，且难以测试（全局状态）。
*   **True DI**: 显式声明依赖（通常通过 `init`）。
*   **Container**: 应该只在 Composition Root (App入口) 使用容器组装对象，而不是把容器传递给每个对象。

---

## 4. System Design (系统设计)

### Q4: 设计一个高可用的日志系统 (Logging System)。
**Follow-up**:
1.  **IO Performance**: 如何避免日志写入阻塞主线程？(mmap, 异步队列).
2.  **Reliability**: App 崩溃时的日志如何保证不丢失？(mmap 内存映射文件是 Crash Safe 的)。
3.  **Security**: 如何处理敏感数据脱敏？

#### Key Answer:
*   **Core**: 使用 `mmap` (Memory Mapped File) 直接映射磁盘文件到内存。写入内存即写入磁盘（由 OS 管理页回写），Crash 时数据依然在磁盘上。
*   **Architecture**: Logger (API) -> Buffer (Ring Buffer) -> Writer (Background Thread/Queue) -> Strategy (File/Network/Console).
