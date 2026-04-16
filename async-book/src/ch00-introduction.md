# Async Rust：从 Future 到生产级应用

## 作者介绍

- 微软 SCHIE（硅片与云硬件基础设施工程）团队首席固件架构师
- 行业资深专家，专长于安全、系统编程（固件、操作系统、虚拟机监控器）、CPU 和平台架构以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此爱上了这门语言

---

这是一本深入讲解 Rust 异步编程的指南。与大多数从 `tokio::main` 开始、对内部原理一笔带过的异步教程不同，本指南从第一性原理构建理解——`Future` trait、polling、状态机——然后逐步深入到实际模式、runtime 选择和生产环境中的陷阱。

## 适合读者

- 能编写同步 Rust 代码但觉得异步令人困惑的 Rust 开发者
- 来自 C#、Go、Python 或 JavaScript 的开发者，了解 `async/await` 但不熟悉 Rust 的模型
- 曾被 `Future is not Send`、`Pin<Box<dyn Future>>` 或"为什么我的程序卡住了？"困扰过的任何人

## 前置要求

你应该熟悉以下内容：
- 所有权、借用和生命周期
- Trait 和泛型（包括 `impl Trait`）
- 使用 `Result<T, E>` 和 `?` 操作符
- 基础多线程（`std::thread::spawn`、`Arc`、`Mutex`）

不需要有 Rust 异步编程经验。

## 如何使用本书

**第一遍请线性阅读。** 第一部分到第三部分的内容循序渐进。每章包含：

| 符号 | 含义 |
|------|------|
| 🟢 | 初学者 — 基础概念 |
| 🟡 | 中级 — 需要前面章节的知识 |
| 🔴 | 高级 — 深入内部原理或生产级模式 |

每章还包括：
- 顶部的 **"你将学到"** 模块
- **Mermaid 图表** 帮助视觉学习者理解
- **内联练习** 附带隐藏解答
- **关键要点** 总结核心思想
- **交叉引用** 指向相关章节

## 学习进度指南

| 章节 | 主题 | 建议时间 | 检查点 |
|------|------|---------|--------|
| 1–5 | 异步工作原理 | 6–8 小时 | 你能解释 `Future`、`Poll`、`Pin` 以及为什么 Rust 没有内置 runtime |
| 6–10 | 生态系统 | 6–8 小时 | 你能手动构建 future、选择 runtime 并使用 tokio 的 API |
| 11–13 | 生产级异步 | 6–8 小时 | 你能用 stream、正确的错误处理和优雅关闭编写生产级异步代码 |
| 综合项目 | Chat Server | 4–6 小时 | 你构建了一个整合所有概念的真实异步应用 |

**总预估时间：22–30 小时**

## 完成练习

每章内容都有内联练习。综合项目（第 16 章）将所有内容整合为一个完整项目。为了最大化学习效果：

1. **在查看答案前先尝试完成练习** — 挣扎是学习发生的地方
2. **动手敲代码，不要复制粘贴** — 肌肉记忆对 Rust 语法很重要
3. **运行每个示例** — `cargo new async-exercises` 并随时测试

## 目录

### 第一部分：异步工作原理

- [1. 为什么 Rust 的异步不同](ch01-why-async-is-different-in-rust.md) 🟢 — 根本区别：Rust 没有内置 runtime
- [2. The Future Trait](ch02-the-future-trait.md) 🟡 — `poll()`、`Waker` 和让它工作的契约
- [3. Poll 工作原理](ch03-how-poll-works.md) 🟡 — polling 状态机和最小化 executor
- [4. Pin 和 Unpin](ch04-pin-and-unpin.md) 🔴 — 为什么自引用 struct 需要 pinning
- [5. 状态机揭秘](ch05-the-state-machine-reveal.md) 🟢 — 编译器从 `async fn` 实际生成的代码

### 第二部分：生态系统

- [6. 手动构建 Futures](ch06-building-futures-by-hand.md) 🟡 — 从零开始实现 TimerFuture、Join、Select
- [7. Executors 和 Runtimes](ch07-executors-and-runtimes.md) 🟡 — tokio、smol、async-std、embassy — 如何选择
- [8. Tokio 深入](ch08-tokio-deep-dive.md) 🟡 — Runtime 类型、spawn、channels、同步原语
- [9. 何时 Tokio 不是最佳选择](ch09-when-tokio-isnt-the-right-fit.md) 🟡 — LocalSet、FuturesUnordered、runtime 无关设计
- [10. Async Traits](ch10-async-traits.md) 🟡 — RPITIT、dyn 分发、trait_variant、async 闭包

### 第三部分：生产级异步

- [11. Streams 和 AsyncIterator](ch11-streams-and-asynciterator.md) 🟡 — 异步迭代、AsyncRead/Write、stream 组合器
- [12. 常见陷阱](ch12-common-pitfalls.md) 🔴 — 9 个生产环境 bug 及如何避免
- [13. 生产级模式](ch13-production-patterns.md) 🔴 — 优雅关闭、背压、Tower 中间件
- [14. 异步是优化而非架构](ch14-async-is-an-optimization-not-an-architecture.md) 🔴 — 同步核心/异步外壳、函数染色开销

### 附录

- [总结与速查表](ch16-summary-and-reference-card.md) — 快速查阅表和决策树
- [综合项目：异步聊天服务器](ch17-capstone-project.md) — 构建完整的异步应用

***
