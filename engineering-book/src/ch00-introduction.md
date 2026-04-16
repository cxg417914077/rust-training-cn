# Rust 工程实践——超越 `cargo build`

## 讲师介绍

- 微软 SCHIE（硅与云硬件基础设施工程）团队首席固件架构师
- 行业资深专家，专长于安全、系统编程（固件、操作系统、管理程序）、CPU 和平台架构，以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此爱上这门语言

---

> 本指南介绍大多数团队发现太晚的 Rust 工具链特性：
> 构建脚本、交叉编译、基准测试、代码覆盖率，以及使用 Miri 和 Valgrind 进行安全验证。
> 每一章都使用来自真实硬件诊断代码库（一个大型多 crate workspace）的具体示例，
> 因此每项技术都能直接映射到生产代码。

## 如何使用本书

本书专为**自学或团队研讨会**设计。每章大部分内容相互独立——可以按顺序阅读，也可以跳转到需要的主题。

### 难度说明

| 符号 | 级别 | 含义 |
|:------:|-------|---------|
| 🟢 | 入门 | 直接了当的工具，模式清晰——第一天就能用上 |
| 🟡 | 中级 | 需要理解工具链内部原理或平台概念 |
| 🔴 | 高级 | 深入的工具链知识、nightly 特性或多工具协作 |

### 进度指南

| 部分 | 章节 | 估计时间 | 关键成果 |
|------|----------|:---------:|-------------|
| **I — 构建与发布** | ch01–02 | 3–4 小时 | 构建元数据、交叉编译、静态二进制 |
| **II — 度量与验证** | ch03–05 | 4–5 小时 | 统计基准测试、覆盖率门禁、Miri/清理器 |
| **III — 加固与优化** | ch06–10 | 6–8 小时 | 供应链安全、发布 profile、编译时工具、`no_std`、Windows |
| **IV — 集成** | ch11–13 | 3–4 小时 | 生产级 CI/CD 流水线、技巧、综合练习 |
| | | **16–21 小时** | **完整的生产工程流水线** |

### 完成练习

每章都包含带难度指示的**🏋️ 练习**。解决方案放在可展开的 `<details>` 块中——先尝试练习，然后检查你的工作。

- 🟢 练习通常可在 10–15 分钟内完成
- 🟡 练习需要 20–40 分钟，可能涉及本地运行工具
- 🔴 练习需要大量设置和实验（1 小时以上）

## 前置知识

| 概念 | 学习地点 |
|---------|-------------------|
| Cargo workspace 布局 | [Rust Book ch14.3](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) |
| Feature 标志 | [Cargo Reference — Features](https://doc.rust-lang.org/cargo/reference/features.html) |
| `#[cfg(test)]` 和基础测试 | Rust Patterns ch12 |
| `unsafe` 块和 FFI 基础 | Rust Patterns ch10 |

## 章节依赖图

```text
                 ┌──────────┐
                 │ ch00     │
                 │  简介    │
                 └────┬─────┘
        ┌─────┬───┬──┴──┬──────┬──────┐
        ▼     ▼   ▼     ▼      ▼      ▼
      ch01  ch03 ch04  ch05   ch06   ch09
      构建  基准 覆盖率 Miri   依赖   no_std
        │     │    │    │      │      │
        │     └────┴────┘      │      ▼
        │          │           │    ch10
        │          ▼           │   Windows
        ▼        ch07        ch07    │
       交叉      RelProf     RelProf  │
        │          │           │     │
        │          ▼           │     │
        │        ch08          │     │
        │      编译时          │     │
        └──────────┴───────────┴─────┘
                   │
                   ▼
                 ch11
               CI/CD 流水线
                   │
                   ▼
                ch12 ─── ch13
               技巧     快速参考
```

**任意顺序阅读**：ch01、ch03、ch04、ch05、ch06、ch09 是独立的。
**在前置条件后阅读**：ch02（需要 ch01）、ch07–ch08（受益于 ch03–ch06）、ch10（受益于 ch09）。
**最后阅读**：ch11（整合一切）、ch12（技巧）、ch13（参考）。

##  annotated 目录

### 第一部分 — 构建与发布

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 1 | [构建脚本——`build.rs` 深入](ch01-build-scripts-buildrs-in-depth.md) | 🟢 | 编译时常量、编译 C 代码、protobuf 生成、系统库链接、反模式 |
| 2 | [交叉编译——一份源码，多个目标](ch02-cross-compilation-one-source-many-target.md) | 🟡 | 目标三元组、musl 静态二进制、ARM 交叉编译、`cross` 工具、`cargo-zigbuild`、GitHub Actions |

### 第二部分 — 度量与验证

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 3 | [基准测试——度量关键指标](ch03-benchmarking-measuring-what-matters.md) | 🟡 | Criterion.rs、Divan、`perf` 火焰图、PGO、CI 中的持续基准测试 |
| 4 | [代码覆盖率——发现测试遗漏](ch04-code-coverage-seeing-what-tests-miss.md) | 🟢 | `cargo-llvm-cov`、`cargo-tarpaulin`、`grcov`、Codecov/Coveralls CI 集成 |
| 5 | [Miri、Valgrind 和清理器](ch05-miri-valgrind-and-sanitizers-verifying-u.md) | 🔴 | MIR 解释器、Valgrind memcheck/Helgrind、ASan/MSan/TSan、cargo-fuzz、loom |

### 第三部分 — 加固与优化

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 6 | [依赖管理与供应链安全](ch06-dependency-management-and-supply-chain-s.md) | 🟢 | `cargo-audit`、`cargo-deny`、`cargo-vet`、`cargo-outdated`、`cargo-semver-checks` |
| 7 | [发布 Profile 与二进制文件大小](ch07-release-profiles-and-binary-size.md) | 🟡 | Release profile 解剖、LTO 权衡、`cargo-bloat`、`cargo-udeps` |
| 8 | [编译时与开发者工具](ch08-compile-time-and-developer-tools.md) | 🟡 | `sccache`、`mold`、`cargo-nextest`、`cargo-expand`、`cargo-geiger`、workspace lints、MSRV |
| 9 | [`no_std` 与 Feature 验证](ch09-no-std-and-feature-verification.md) | 🔴 | `cargo-hack`、`core`/`alloc`/`std` 层、自定义 panic 处理程序、测试 `no_std` 代码 |
| 10 | [Windows 与条件编译](ch10-windows-and-conditional-compilation.md) | 🟡 | `#[cfg]` 模式、`windows-sys`/`windows` crates、`cargo-xwin`、平台抽象 |

### 第四部分 — 集成

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 11 | [整合一切——生产级 CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) | 🟡 | GitHub Actions 工作流、`cargo-make`、pre-commit hooks、`cargo-dist`、综合项目 |
| 12 | [实战技巧](ch12-tricks-from-the-trenches.md) | 🟡 | 10 个经过实战验证的模式：`deny(warnings)` 陷阱、缓存调优、依赖去重、RUSTFLAGS 等 |
| 13 | [快速参考卡](ch13-quick-reference-card.md) | — | 一目了然的命令、60+ 决策表条目、延伸阅读链接 |
