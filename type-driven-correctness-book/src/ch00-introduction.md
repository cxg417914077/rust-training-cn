# Rust 中的类型驱动正确性

## 演讲者介绍

- 微软 SCHIE（硅与云硬件基础设施工程）团队首席固件架构师
- 行业资深专家，专长于安全、系统编程（固件、操作系统、管理程序）、CPU 和平台架构以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此爱上了这门语言

---

本实用指南介绍如何使用 Rust 的类型系统让整个类别的 bug**无法编译**。配套的 [Rust Patterns](../../rust-patterns-book/src/SUMMARY.md) 书籍涵盖了机制（traits、关联类型、type-state），而本指南则展示如何将这些机制**应用**到现实世界领域——硬件诊断、加密、协议验证和嵌入式系统。

这里的每个模式都遵循一个原则：**将不变量从运行时检查推入类型系统，让编译器强制执行。**

## 如何使用本书

### 难度图例

| 符号 | 级别 | 目标读者 |
|:------:|-------|----------|
| 🟢 | 入门 | 熟悉所有权 + traits |
| 🟡 | 中级 | 熟悉泛型 + 关联类型 |
| 🔴 | 高级 | 准备好学习 type-state、phantom types 和 session types |

### 进度指南

| 目标 | 路径 | 时间 |
|------|------|------|
| **快速概览** | ch01, ch13（参考卡） | 30 分钟 |
| **IPMI / BMC 开发者** | ch02, ch05, ch07, ch10, ch17 | 2.5 小时 |
| **GPU / PCIe 开发者** | ch02, ch06, ch09, ch10, ch15 | 2.5 小时 |
| **Redfish 实现者** | ch02, ch05, ch07, ch08, ch17, ch18 | 3 小时 |
| **框架 / 基础设施** | ch04, ch08, ch11, ch14, ch18 | 2.5 小时 |
| **构造正确性新手** | 按顺序 ch01 → ch10，然后 ch12 练习 | 4 小时 |
| **完整深入** | 按顺序阅读所有章节 | 7 小时 |

### 带注释的目录

| 章 | 标题 | 难度 | 核心思想 |
|----|-------|:----------:|----------|
| 1 | 理念——为什么类型击败测试 | 🟢 | 三层正确性；类型作为编译器检查的保证 |
| 2 | 类型化命令接口 | 🟡 | 关联类型绑定请求 → 响应 |
| 3 | 一次性类型 | 🟡 | 移动语义作为加密的线性类型 |
| 4 | Capability 令牌 | 🟡 | 零尺寸权限证明令牌 |
| 5 | 协议状态机 | 🔴 | IPMI sessions + PCIe LTSSM 的 type-state |
| 6 | 量纲分析 | 🟢 | Newtype 包装器防止单位混淆 |
| 7 | 有效性边界 | 🟡 | 在边缘解析一次，在类型中携带证明 |
| 8 | Capability 混合 | 🟡 | 成分 traits + 空白实现 |
| 9 | Phantom 类型 | 🟡 | PhantomData 用于寄存器宽度、DMA 方向 |
| 10 | 整合一切 | 🟡 | 一个诊断平台中的全部 7 个模式 |
| 11 | 实战十四个技巧 | 🟡 | Sentinel→Option、密封 traits、builders 等 |
| 12 | 练习 | 🟡 | 六个综合项目问题附带解决方案 |
| 13 | 参考卡 | — | 模式目录 + 决策流程图 |
| 14 | 测试类型级保证 | 🟡 | trybuild, proptest, cargo-show-asm |
| 15 | Const Fn | 🟠 | 内存映射、寄存器、比特域的编译时证明 |
| 16 | Send & Sync | 🟠 | 编译时并发证明 |
| 17 | Redfish 客户端演练 | 🟡 | 八个模式组合成类型安全的 Redfish 客户端 |
| 18 | Redfish 服务器演练 | 🟡 | Builder type-state、源令牌、健康汇总、mixins |

## 前置知识

| 概念 | 学习资源 |
|---------|-------------------|
| 所有权和借用 | [Rust Patterns](../rust-patterns-book/src/SUMMARY.md), ch01 |
| Traits 和关联类型 | [Rust Patterns](../rust-patterns-book/src/SUMMARY.md), ch02 |
| Newtypes 和 type-state | [Rust Patterns](../rust-patterns-book/src/SUMMARY.md), ch03 |
| PhantomData | [Rust Patterns](../rust-patterns-book/src/SUMMARY.md), ch04 |
| 泛型和 trait 边界 | [Rust Patterns](../rust-patterns-book/src/SUMMARY.md), ch01 |

## 构造正确性谱系

```text
← 更安全度较低                                              更安全度较高 →

运行时检查        单元测试         属性测试          构造正确性
─────────────     ──────────       ──────────────    ──────────────────────

if temp > 100 {   #[test]          proptest! {        struct Celsius(f64);
  panic!("too     fn test_temp() {   |t in 0..200| {  // 类型层面无法
  hot");            assert!(         assert!(...)      // 与 Rpm 混淆
}                   check(42));    }
                  }                }
                                                       无效程序？
无效程序？        无效程序？       无效程序？          无法编译。
生产环境崩溃。    CI 中失败。      CI 中失败           永不复存在。
                                    （概率性）。
```

本指南位于最右侧的位置——在这个位置上，bug 不存在，因为类型系统**无法表达它们**。

---
