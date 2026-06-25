# 06. Tooling, Testing & CI/CD

> **Difficulty**: Senior / Staff
> **Focus**: LLDB, Compiler Flags, Unit Testing, Static Analysis

---

## 1. LLDB Advanced Debugging

### Q1: 除了 `po`，你还知道哪些 LLDB 高级指令？如何编写 Python 脚本辅助调试？
**Follow-up**:
1.  `p` vs `po` vs `v` 的区别？(提示: `v` 直接读取内存，不运行代码，最快且无副作用)。
2.  **Watchpoint**: 如何监控一个变量什么时候被修改？
3.  **Chisel**: Facebook 的 Chisel 插件原理是什么？(封装了复杂的 Python API 操作 LLDB SBValue)。

#### Key Answer:
*   **`v` (frame variable)**: 直接读取内存中的变量值，不经过 Expression Parser，不执行代码。在调试 Crash 或死锁时最安全。
*   **`watchpoint set variable myVar`**: 硬件断点。当内存地址内容变化时触发中断。
*   **Scripting**: 使用 `command script import my_script.py`。可以通过 `lldb.SBProcess`, `lldb.SBFrame` 等 API 访问运行时状态，甚至修改寄存器。

---

## 2. Unit Testing & UI Testing

### Q2: 如何提高单元测试的稳定性 (Flakiness) 和速度？
**Follow-up**:
1.  **Test Double**: Mock vs Stub vs Spy vs Fake 的区别。
2.  **Network Testing**: 为什么不应该在单元测试中发真实网络请求？如何正确 Mock `URLSession`？(使用 `URLProtocol` 拦截，或者依赖注入 `URLSessionProtocol`)。
3.  **Snapshot Testing**: 原理是什么？如何处理不同屏幕尺寸和系统的差异？

#### Key Answer:
*   **Speed**: 避免 IO，避免 `wait(for:timeout:)` (使用确定性的 Mock)，开启并行测试 (Parallel Testing)。
*   **Mocking URLSession**:
    1.  **Protocol**: 定义 `HTTPClient` 协议，测试时注入 Mock 实现。
    2.  **URLProtocol**: 注册自定义 `URLProtocol` 子类，拦截所有请求并返回预设数据。这是更底层的 Mock，能测试到 `URLSession` 的配置逻辑。

---

## 3. Compiler & Build System

### Q3: 如何优化 Swift 项目的编译时间？
**Follow-up**:
1.  **Incremental Build**: 增量编译失效的常见原因？(修改了 Header，修改了被广泛依赖的 struct 布局)。
2.  **Compiler Flags**: `-warn-long-function-bodies` 和 `-warn-long-expression-type-checking` 的作用。
3.  **dSYM**: dSYM 文件里包含了什么？Crash 符号化 (Symbolication) 的过程是怎样的？

#### Key Answer:
*   **Optimization**:
    *   模块化 (Modularization)：减少重新编译范围。
    *   显式类型标注：减少类型推断耗时。
    *   减少 `main.swift` 或全局作用域代码。
*   **dSYM**: DWARF 调试信息。包含文件名、行号、函数名与二进制地址的映射关系。符号化就是 `Address - Load Address + Slide = File Offset`，然后在 dSYM 中查找。

---

## 4. CI/CD & Static Analysis

### Q4: 你会在 CI 流水线中加入哪些静态分析工具？为什么？
**Follow-up**:
1.  **SwiftLint**: 如何自定义规则？
2.  **Periphery**: 它是如何检测无用代码 (Dead Code) 的？(基于 Index Store 分析引用关系)。
3.  **Code Coverage**: 为什么 100% 覆盖率不代表没有 Bug？

#### Key Answer:
*   **Tools**: SwiftLint (代码风格), Periphery (死代码), SonarQube (代码质量/复杂度), Danger (自动化 Code Review 评论)。
*   **Periphery**: 相比简单的正则搜索，它解析 Xcode 生成的 Index Store 数据库，能准确识别类、函数、属性的引用关系，甚至能识别只被测试代码引用的"伪"死代码。
