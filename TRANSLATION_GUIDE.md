# Rust Training 中文翻译指南

## 📋 翻译原则

### 1. 准确性优先

- **技术术语**：使用 Rust 中文社区公认的翻译
- **代码示例**：保持原文，不要翻译代码、注释和字符串
- **链接处理**：外部链接保留原文，内部链接指向对应的中文版本

### 2. 术语一致性

参考 [术语表](./terminology.md) 保持一致的翻译风格。常见术语：

| 英文 | 推荐翻译 | 说明 |
|------|---------|------|
| Ownership | 所有权 | Rust 核心概念 |
| Borrowing | 借用 | Rust 核心概念 |
| Lifetime | 生命周期 | Rust 核心概念 |
| Trait | 特征 | Rust 类型系统 |
| Closure | 闭包 | 函数式编程 |
| Pattern Matching | 模式匹配 | 控制流 |
| Enum | 枚举 | 数据类型 |
| Struct | 结构体 | 数据类型 |
| Future | 未来值 | 异步编程 |
| Pin | 固定 | 异步原语 |
| Executor | 执行器 | 运行时 |
| Runtime | 运行时 | 程序执行环境 |

### 3. 格式规范

- **标题层级**：保持原文的 `#`、`##`、`###` 层级结构
- **代码块**：保留语言标识符（```rust、```python 等）
- **表格**：保持原文的表格结构
- **列表**：保持有序/无序列表格式

### 4. 风格指南

- **语言风格**：简洁、准确、专业，避免口语化
- **人称代词**：使用「你」而非「您」，保持技术文档的一致性
- **句子结构**：中文表达，避免翻译腔

## 🚀 快速开始

### 步骤 1：选择要翻译的章节

查看 [翻译进度](./README.md#翻译进度) 选择尚未翻译的章节。

### 步骤 2：创建翻译文件

```bash
# 进入项目目录
cd /Users/chengxuguang/rusttrainning

# 创建翻译文件（以 python-book 为例）
mkdir -p chinese-translation/python-book/src
```

### 步骤 3：开始翻译

```bash
# 阅读原文
code ../python-book/src/ch01-introduction-and-motivation.md

# 创建翻译
code chinese-translation/python-book/src/ch01-introduction-and-motivation.md
```

### 步骤 4：本地预览

```bash
# 构建中文版
cargo xtask build-cn

# 启动服务
cargo xtask serve-cn  # http://localhost:3001
```

## 📝 翻译技巧

### 处理长句

英文技术文档常使用复杂长句，翻译时应拆分为简短的中文句子：

**原文**：
> "This guide covers everything from basic syntax to advanced patterns, focusing on the conceptual shifts required when moving from a dynamically-typed, garbage-collected language to a statically-typed systems language with compile-time memory safety."

**推荐翻译**：
> "本指南涵盖从基础语法到高级模式的所有内容，重点关注概念转变——从动态类型、垃圾回收语言，转向静态类型、编译时内存安全的系统编程语言。"

### 处理技术术语

首次出现的术语可在括号内标注原文：

> "所有权（Ownership）是 Rust 最核心的概念之一。"

后续出现时直接使用中文翻译。

### 处理代码示例

**不要翻译**：
- 变量名、函数名、类型名
- 代码注释
- 字符串字面量
- 错误信息（可在括号内翻译）

```rust
// ✅ 保持原样
fn main() {
    let message = "Hello, world!";
    println!("{}", message);
}
```

### 处理表格和列表

保持原文格式，只翻译内容：

```markdown
| 章节 | 主题 | 建议时间 |
|------|------|---------|
| 1–4 | 环境搭建、类型、控制流 | 1 天 |
```

## ✅ 检查清单

提交翻译前请确认：

- [ ] 术语翻译与术语表一致
- [ ] 代码示例保持原文
- [ ] 链接指向正确的中文版本
- [ ] 表格格式正确
- [ ] 没有漏译或错译
- [ ] 中文表达流畅自然
- [ ] 没有翻译腔

## 🔄 提交流程

1. Fork 本项目
2. 创建翻译分支 `git checkout -b translate/python-book-ch01`
3. 提交翻译 `git add . && git commit -m "翻译：Python 指南 第 1 章"`
4. 推送到远程 `git push origin translate/python-book-ch01`
5. 创建 Pull Request

## 📚 参考资源

- [Rust 官方文档中文版](https://rustlang-cn.github.io/rust-book/)
- [Rust 术语表](https://github.com/rust-lang-cn/rust-term)
- [技术文档翻译规范](https://github.com/rust-lang-cn/translation-guide)

---

**感谢你的贡献！** 🦀
