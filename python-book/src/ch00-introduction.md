# Rust Python 程序员指南：完整培训教程

这是一本为具有 Python 经验的开发者编写的全面 Rust 指南。本书涵盖从基础语法到高级模式的所有内容，重点关注从**动态类型、垃圾回收语言**转向**静态类型、编译时内存安全的系统编程语言**时所需的概念转变。

## 如何使用本书

**自学路径**：首先完成第一部分（第 1-6 章）—— 这些章节与你已经熟悉的 Python 概念紧密对应。第二部分（第 7-12 章）介绍 Rust 特有的概念，如所有权和 trait。第三部分（第 13-16 章）涵盖高级主题和迁移策略。

**建议学习节奏：**

| 章节 | 主题 | 建议时间 | 检查点 |
|------|------|---------|--------|
| 1–4 | 环境搭建、类型、控制流 | 1 天 | 你能用 Rust 编写一个 CLI 温度转换器 |
| 5–6 | 数据结构、枚举、模式匹配 | 1–2 天 | 你能定义带数据的枚举并进行穷尽 `match` 匹配 |
| 7 | 所有权和借用 | 1–2 天 | 你能解释*为什么* `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1 天 | 你能创建一个使用 `?` 传播错误的多文件项目 |
| 10–12 | Trait、泛型、闭包、迭代器 | 1–2 天 | 你能将列表推导式转换为迭代器链 |
| 13 | 并发 | 1 天 | 你能用 `Arc<Mutex<T>>` 编写线程安全计数器 |
| 14 | Unsafe、PyO3、测试 | 1 天 | 你能通过 PyO3 从 Python 调用 Rust 函数 |
| 15–16 | 迁移、最佳实践 | 自行安排 | 参考资料 —— 在编写实际代码时查阅 |
| 17 | 综合项目 | 2–3 天 | 构建一个完整的 CLI 应用，综合运用所有知识 |

**如何使用练习题：**
- 各章节包含可折叠 `<details>` 块中的动手练习和解答
- **务必在查看解答前先尝试完成练习。** 与借用检查器的斗争是学习过程的一部分 —— 编译器的错误信息是你的老师
- 如果卡壳超过 15 分钟，展开解答，仔细研究，然后关上解答从头再试一次
- [Rust Playground](https://play.rust-lang.org/) 让你无需本地安装就能运行代码

**难度标识：**
- 🟢 **初学者** — 直接从 Python 概念转换
- 🟡 **中级** — 需要理解所有权或 trait
- 🔴 **高级** — 生命周期、异步内部、或 unsafe 代码

**当你遇到困难时：**
- 仔细阅读编译器错误信息 —— Rust 的错误提示非常友好
- 重读相关章节；像所有权（第 7 章）这样的概念通常需要读第二遍才能理解
- [Rust 标准库文档](https://doc.rust-lang.org/std/) 非常优秀 —— 搜索任何类型或方法
- 更深入的异步模式，请参阅配套的 [Rust 异步编程培训](../async-book/)

---

## 目录

### 第一部分 — 基础

#### 1. 介绍与动机 🟢
- [Python 开发者选择 Rust 的理由](ch01-introduction-and-motivation.md#python-开发者选择 rust 的理由)
- [Rust 解决的 Python 常见痛点](ch01-introduction-and-motivation.md#rust-解决的 python 常见痛点)
- [何时选择 Rust 而非 Python](ch01-introduction-and-motivation.md#何时选择 rust 而非 python)

#### 2. 入门指南 🟢
- [安装与环境搭建](ch02-getting-started.md#安装与环境搭建)
- [你的第一个 Rust 程序](ch02-getting-started.md#你的第一个 rust 程序)
- [Cargo vs pip/Poetry](ch02-getting-started.md#cargo-vs-pippoetry)

#### 3. 内置类型与变量 🟢
- [变量与可变性](ch03-built-in-types-and-variables.md#变量与可变性)
- [基本类型对比](ch03-built-in-types-and-variables.md#基本类型对比)
- [字符串类型：String vs &str](ch03-built-in-types-and-variables.md#字符串类型 string-vs-str)

#### 4. 控制流 🟢
- [条件语句](ch04-control-flow.md#条件语句)
- [循环与迭代](ch04-control-flow.md#循环与迭代)
- [表达式块](ch04-control-flow.md#表达式块)
- [函数与类型签名](ch04-control-flow.md#函数与类型签名)

#### 5. 数据结构与集合 🟢
- [元组、数组、切片](ch05-data-structures-and-collections.md#元组和解构)
- [结构体 vs 类](ch05-data-structures-and-collections.md#结构体 vs 类)
- [Vec vs list, HashMap vs dict](ch05-data-structures-and-collections.md#vec-vs-list)

#### 6. 枚举与模式匹配 🟡
- [代数数据类型 vs 联合类型](ch06-enums-and-pattern-matching.md#代数数据类型 vs 联合类型)
- [穷尽模式匹配](ch06-enums-and-pattern-matching.md#穷尽模式匹配)
- [Option 实现 None 安全](ch06-enums-and-pattern-matching.md#option-实现 none 安全)

### 第二部分 — 核心概念

#### 7. 所有权与借用 🟡
- [理解所有权](ch07-ownership-and-borrowing.md#理解所有权)
- [移动语义 vs 引用计数](ch07-ownership-and-borrowing.md#移动语义 vs 引用计数)
- [借用与生命周期](ch07-ownership-and-borrowing.md#借用与生命周期)
- [智能指针](ch07-ownership-and-borrowing.md#智能指针)

#### 8. Crate 与模块 🟢
- [Rust 模块 vs Python 包](ch08-crates-and-modules.md#rust 模块 vs-python 包)
- [Crate vs PyPI 包](ch08-crates-and-modules.md#crate-vs-pypi 包)

#### 9. 错误处理 🟡
- [异常 vs Result](ch09-error-handling.md#异常 vs-result)
- [? 操作符](ch09-error-handling.md#--操作符)
- [使用 thiserror 定义自定义错误类型](ch09-error-handling.md#使用 thiserror 定义自定义错误类型)

#### 10. Trait 与泛型 🟡
- [Trait vs 鸭子类型](ch10-traits-and-generics.md#特征 vs 鸭子类型)
- [协议 (PEP 544) vs Trait](ch10-traits-and-generics.md#协议 pep-544-vs-特征)
- [泛型约束](ch10-traits-and-generics.md#泛型约束)

#### 11. From 和 Into Trait 🟡
- [Rust 中的类型转换](ch11-from-and-into-traits.md#rust 中的类型转换)
- [From、Into、TryFrom](ch11-from-and-into-traits.md#rust-frominto)
- [字符串转换模式](ch11-from-and-into-traits.md#字符串转换模式)

#### 12. 闭包与迭代器 🟡
- [闭包 vs Lambda 表达式](ch12-closures-and-iterators.md#rust-闭包-vs-python-lambda)
- [迭代器 vs 生成器](ch12-closures-and-iterators.md#迭代器 vs 生成器)
- [宏：编写代码的代码](ch12-closures-and-iterators.md#为什么-rust 需要宏)

### 第三部分 — 高级主题与迁移

#### 13. 并发编程 🔴
- [无 GIL：真正的并行](ch13-concurrency.md#无 gil-真正的并行)
- [线程安全：类型系统保证](ch13-concurrency.md#线程安全类型系统保证)
- [async/await 对比](ch13-concurrency.md#asyncawait-对比)

#### 14. Unsafe Rust、FFI 与测试 🔴
- [何时以及为何使用 Unsafe](ch14-unsafe-rust-and-ffi.md#何时以及为何使用 unsafe)
- [PyO3：为 Python 编写 Rust 扩展](ch14-unsafe-rust-and-ffi.md#pyo3 为 python 编写 rust 扩展)
- [单元测试 vs pytest](ch14-unsafe-rust-and-ffi.md#单元测试-vs-pytest)

#### 15. 迁移模式 🟡
- [Rust 中常见的 Python 模式](ch15-migration-patterns.md#rust-中常见的 python 模式)
- [Python 开发者必备的 Crate](ch08-crates-and-modules.md#python-开发者必备的 crate)
- [渐进式采用策略](ch15-migration-patterns.md#渐进式采用策略)

#### 16. 最佳实践 🟡
- [Python 开发者的地道 Rust](ch16-best-practices.md#python-开发者的地道 rust)
- [常见陷阱与解决方案](ch16-best-practices.md#常见陷阱与解决方案)
- [Python→Rust 对照表](ch16-best-practices.md#pythonrust-对照表)
- [学习路径与资源](ch16-best-practices.md#学习路径与资源)

---

### 第四部分 — 实战项目

#### 17. 综合项目：CLI 任务管理器 🔴
- [项目：`rustdo`](ch17-capstone-project.md#项目 rustdo)
- [数据模型、存储、命令、业务逻辑](ch17-capstone-project.md#步骤 1-定义数据模型 - 第 -3-6-10-11-章)
- [测试与扩展目标](ch17-capstone-project.md#步骤 7-测试 - 第 -14-章)

***
