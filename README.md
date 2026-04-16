# Rust Training 中文翻译项目

> 🦀 微软官方 Rust 培训教程的中文翻译版本
>
> 原项目：https://microsoft.github.io/RustTraining/

## 📚 翻译进度

| 书籍 | 难度 | 翻译进度 | 状态 |
|------|------|---------|------|
| [Rust for Python Programmers](python-book/) | 🟢 Bridge | ✅ 完成 | 可阅读 |
| [Async Rust](async-book/) | 🟡 Advanced | ✅ 完成 | 可阅读 |
| [Rust Patterns](rust-patterns-book/) | 🟡 Advanced | ✅ 完成 | 可阅读 |
| [Type-Driven Correctness](type-driven-correctness-book/) | 🟣 Expert | ✅ 完成 | 可阅读 |
| [Rust Engineering Practices](engineering-book/) | 🟤 Practices | ✅ 完成 | 可阅读 |
| Rust for C/C++ Programmers | 🟢 Bridge | ⬜ 未开始 | 📝 待翻译 |
| Rust for C# Programmers | 🟢 Bridge | ⬜ 未开始 | 📝 待翻译 |

## 📖 快速开始

### 按背景选择书籍

| 编程背景 | 推荐书籍 |
|---------|---------|
| Python 开发者 | [Rust for Python Programmers](python-book/) |
| 已有 Rust 基础 | [Async Rust](async-book/)、[Rust Patterns](rust-patterns-book/) |
| 追求类型系统深度 | [Type-Driven Correctness](type-driven-correctness-book/) |
| 工程实践导向 | [Rust Engineering Practices](engineering-book/) |

### 按难度选择

| 难度 | 书籍 |
|------|------|
| 🟢 **入门** | Rust for Python Programmers |
| 🟡 **进阶** | Async Rust、Rust Patterns |
| 🟣 **专家** | Type-Driven Correctness |
| 🟤 **工程实践** | Rust Engineering Practices |

## 🚀 本地构建

```bash
# 安装 mdbook
cargo install mdbook mdbook-mermaid

# 进入书籍目录构建
cd python-book && mdbook serve --open
cd async-book && mdbook serve --open
cd rust-patterns-book && mdbook serve --open
cd type-driven-correctness-book && mdbook serve --open
cd engineering-book && mdbook serve --open
```

## 🎯 翻译指南

### 翻译原则

1. **准确性优先** - 技术术语准确，符合 Rust 中文社区约定
2. **代码不变** - Rust 代码示例、注释保持原文
3. **链接处理** - 外部链接保留，内部链接指向对应中文版本
4. **格式一致** - 保持原文的 Markdown 格式、标题层级

### 常用术语表

详见 [术语表](terminology.md)

## 📝 如何参与

1. Fork 本项目
2. 选择一本你熟悉的书籍开始翻译
3. 提交 Pull Request
4. 社区 Review 后合并

## 📄 许可证

- 代码：[MIT License](LICENSE)
- 文档：[CC-BY-4.0](LICENSE-DOCS)

## 🔗 相关链接

- [英文原版](https://microsoft.github.io/RustTraining/)
- [Rust 官方文档](https://doc.rust-lang.org/book/)
- [Rust 中文社区](https://rustcc.cn/)
