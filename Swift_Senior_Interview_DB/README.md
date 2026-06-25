# Swift Senior Interview Question Bank (The "Soul-Crushing" Edition)

> **Warning**: This content is designed for **Senior / Staff iOS Engineers**. It focuses on compiler internals, runtime mechanisms, and deep architectural principles.

## Directory Structure

### [01. Runtime, Compiler & Dispatch](./01_Runtime_Compiler.md)
*   Method Dispatch (Static, V-Table, Message)
*   Existential Containers & Opaque Types
*   Metadata, Reflection & ABI Stability

### [02. Memory Management & Performance](./02_Memory_Performance.md)
*   ARC Internals & Side Tables
*   Copy-on-Write (CoW) Implementation
*   Pointers (Unsafe) & Optimization Levels

### [03. Concurrency & Thread Safety](./03_Concurrency.md)
*   Swift Concurrency (Actors, Reentrancy)
*   GCD vs Concurrency Models
*   Sendable & Thread Safety Traps

### [04. SwiftUI Internals](./04_SwiftUI_Internals.md)
*   AttributeGraph & Dependency Tracking
*   Layout Negotiation Protocol
*   Identity (Structural vs Explicit) & AnyView

### [05. Architecture & System Design](./05_Architecture_SystemDesign.md)
*   Modularization (Static vs Dynamic)
*   Design Patterns (MVVM-C, TCA, VIPER)
*   Dependency Injection & Logging Systems

### [06. Tooling, Testing & CI/CD](./06_Tooling_Testing.md)
*   LLDB Advanced Debugging & Scripting
*   Unit Testing Strategies & Mocking
*   Compiler Flags & Build Optimization

### [07. Deep Dive Analysis: User Questions](./07_Deep_Dive_Analysis.md)
*   **Source**: User submitted `Swift_Senior_questions.md`
*   **Content**: Detailed answers and principle analysis for the 15 "Soul-Crushing" questions.

### [08. Thinking in SwiftUI (Book Edition)](./08_Thinking_in_SwiftUI.md)
*   **Source**: *Thinking in SwiftUI* (objc.io)
*   **Content**: View Tree, Layout Protocol, Preference Keys, Environment.

### [09. Advanced Swift (Book Edition)](./09_Advanced_Swift.md)
*   **Source**: *Advanced Swift* (objc.io)
*   **Content**: Structs/Classes, Generics, Unsafe Swift, C Interop.

### [10. SwiftUI & Combine Architecture (Book Edition)](./10_SwiftUI_Combine_Architecture.md)
*   **Source**: *App Architecture* & Combine Patterns
*   **Content**: Publishers, Data Flow, Side Effects, MVVM-C vs TCA.


### [11. Swift Underlying Principles (Black Magic)](./11_Swift_Underlying_Principles.md)
*   **Focus**: SIL, Runtime, Metadata, Mangling, ABI.
*   **Content**: 15+ questions on the deepest parts of Swift.

### [12. Swift Advanced Features (Bleeding Edge)](./12_Swift_Advanced_Features.md)
*   **Focus**: Macros, C++ Interop, Result Builders, Distributed Actors.
*   **Content**: 15+ questions on Swift 5.9+ and advanced metaprogramming.

### [13. SwiftUI High Performance (Rocket Science)](./13_SwiftUI_High_Performance.md)
*   **Focus**: Render Loop, AttributeGraph, Layout Performance, Metal.
*   **Content**: 15+ questions on optimizing SwiftUI to the limit.
### [14. Protocol-Oriented Programming (Type System)](./14_Protocol_Oriented_Programming.md)
*   **Focus**: Protocols, Generics, Type Erasure, Existentials, Opaque Types.
*   **Content**: 20+ questions on the power of Swift's type system.

### [15. Swift Concurrency & Actors (Modern Threading)](./15_Swift_Concurrency_Actors.md)
*   **Focus**: Actors, Task, Sendable, Structured Concurrency.
*   **Content**: 20+ questions on the new concurrency model.

### [16. Architecture: MVVM & TCA (State Management)](./16_Architecture_MVVM_TCA.md)
*   **Focus**: MVVM, The Composable Architecture, Unidirectional Data Flow.
*   **Content**: 20+ questions on building scalable apps.
### [17. Modern Architecture Patterns (Clean Edition)](./17_Modern_Architecture_Patterns.md)
*   **Focus**: Clean Architecture, MVVM+Reducer, Coordinator in SwiftUI, Dependency Injection.
*   **Content**: 10+ questions on modern app structure and navigation.

### [18. SwiftUI Animation & Gestures (Fluid Interface)](./18_SwiftUI_Animation_Gestures.md)
*   **Focus**: Animation System, Transactions, GeometryEffect, Gestures.
*   **Content**: 10+ questions on creating buttery smooth interactions.
### [19. Swift Property Wrappers: Deep Dive](./19_Property_Wrappers_Deep_Dive.md)
*   **Type**: Technical Article
*   **Content**: Implementation of `@State`, `@StateObject`, and Custom Wrappers.

### [20. SwiftUI Architecture Principles](./20_SwiftUI_Architecture_Principles.md)
*   **Type**: Technical Article
*   **Content**: State Memory Layout, NavigationStack Philosophy, Dependency Injection Internals.

---

## How to Use
1.  **Self-Assessment**: Try to answer the "Key Question" without looking at the answer.
2.  **Deep Dive**: If you fail a question, read the "Key Answer" and the "Follow-up" traps.
3.  **Code It**: For questions like "CoW Implementation" or "Actor Reentrancy", write actual code to verify your understanding.

> "Talk is cheap. Show me the code." — Linus Torvalds
