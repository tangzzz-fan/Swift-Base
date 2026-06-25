# Swift Base — 个人 Swift 知识库

> 一份面向 **Senior / Staff iOS 工程师** 的深度技术知识库，涵盖 Swift 语言底层、SwiftUI 原理、架构设计、并发编程等核心主题。

## 目录结构

### 专题文章

| 文件 | 主题 | 内容概要 |
|------|------|----------|
| `Swift_Concurrency_Task_TaskGroup.md` | Swift 并发 | Task、TaskGroup 与结构化并发深度指南 |
| `Swift_Coroutines_Explanation.md` | 协程原理 | 协程的挂起/恢复机制及其在 Swift 中的应用 |
| `Swift_Actor_LRU_Cache_Image_Loader.md` | Actor 实战 | 基于 Actor 的 LRU 缓存图片加载器实现 |
| `Swift_Type_Erasure.md` | 类型系统 | 类型擦除 (Type Erasure) 技术与 PATs 处理 |
| `SwiftUI_ResultBuilders_ViewBuilder.md` | SwiftUI 底层 | Result Builders 与 ViewBuilder 原理剖析 |
| `SwiftUI_DI_vs_Swinject.md` | 依赖注入 | SwiftUI Environment vs Swinject 五大场景对比 |
| `SwiftUI_DI_Timing_and_Protocols.md` | 依赖注入 | SwiftUI DI 时机与协议设计 |
| `SwiftUI_Modularization_Decoupling.md` | 架构解耦 | SwiftUI 模块化与解耦策略 |
| `UIKit_PropertyWrappers.md` | UIKit & SwiftUI | UIKit 中使用 SwiftUI 属性包装器的可行性 |
| `Swift_Senior_Interview_Questions.md` | 面试题集 | 15 道"灵魂拷问"级高级面试题 |

### 面试题库 (`Swift_Senior_Interview_DB/`)

系统化的 Swift 高级面试题库，共 22 个专题，覆盖：

| 编号 | 专题 | 方向 |
|------|------|------|
| 01 | Runtime & Compiler | 方法分发、Existential Container、ABI |
| 02 | Memory & Performance | ARC 内部机制、CoW、Unsafe Pointers |
| 03 | Concurrency | GCD、Actor、Sendable、线程安全 |
| 04 | SwiftUI Internals | AttributeGraph、Layout、Identity |
| 05 | Architecture & System Design | 模块化、MVVM-C、TCA、VIPER |
| 06 | Tooling & Testing | LLDB、单元测试策略、编译优化 |
| 07 | Deep Dive Analysis | 用户提交问题的深度解答 |
| 08 | Thinking in SwiftUI | View Tree、Layout Protocol、Preference Keys |
| 09 | Advanced Swift | 结构体/类、泛型、Unsafe Swift |
| 10 | SwiftUI & Combine | Publisher、数据流、副作用 |
| 11 | Swift Underlying Principles | SIL、Runtime、Metadata、Name Mangling |
| 12 | Swift Advanced Features | Macro、C++ Interop、Distributed Actors |
| 13 | SwiftUI High Performance | Render Loop、Metal、布局性能 |
| 14 | Protocol-Oriented Programming | 协议、泛型、Opaque Types |
| 15 | Swift Concurrency & Actors | Actor、Task、Sendable |
| 16 | MVVM & TCA | 状态管理、单向数据流 |
| 17 | Modern Architecture | Clean Architecture、Coordinator |
| 18 | Animation & Gestures | Transaction、GeometryEffect |
| 19 | Property Wrappers | @State、@StateObject 实现原理 |
| 20 | SwiftUI Architecture | State 内存布局、NavigationStack |
| 21 | Answers & VM Comparison | 面试答案与 ViewModel 对比 |
| 22 | Swift Expert Questions | 专家级进阶问题 |

## 使用方法

1. **系统阅读**：按专题文章逐篇深入，每篇文章聚焦一个独立主题
2. **面试准备**：进入 `Swift_Senior_Interview_DB/` 按编号顺序自测，无法回答的问题再查看详解
3. **实战验证**：对关键概念（如 CoW、Actor 重入）动手写代码验证理解

> "Talk is cheap. Show me the code." — Linus Torvalds

## 技术覆盖

- **语言底层**：SIL、Runtime、Metadata、Name Mangling、ABI
- **并发模型**：Swift Concurrency (async/await)、Actor、TaskGroup、GCD
- **SwiftUI**：AttributeGraph、Layout Protocol、View Identity、Animation
- **架构模式**：MVVM、TCA、Clean Architecture、Coordinator Pattern
- **内存管理**：ARC、CoW、Weak Reference、Side Table
- **类型系统**：泛型、Opaque Types、Existential Types、Type Erasure
- **性能优化**：编译优化、Render Loop、方法分发策略
