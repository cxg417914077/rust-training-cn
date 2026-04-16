# 实战技巧 🟡

> **你将学到：**
> - 不适合归入单一章节的实战模式
> - 常见陷阱及其修复——从 CI 不稳定到二进制膨胀
> - 可以立即应用到任何 Rust 项目的快速取胜技巧
>
> **交叉引用：**本书的每一章——这些技巧跨越所有主题

本章收集在生产 Rust 代码库中反复出现的工程模式。每个技巧都是自包含的——按任意顺序阅读。

---

### 1. `deny(warnings)` 陷阱

**问题**：源代码中的 `#![deny(warnings)]` 会在 Clippy 添加新 lint 时破坏构建——昨天还能编译的代码今天失败了。

**修复**：在 CI 中使用 `CARGO_ENCODED_RUSTFLAGS` 而不是源代码级别的属性：

```yaml
# CI：将警告视为错误，无需触碰源代码
env:
  CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"
```

或者使用 `[workspace.lints]` 获取更细粒度的控制：

```toml
# Cargo.toml
[workspace.lints.rust]
unsafe_code = "deny"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
```

> 完整模式参见 [编译时工具，Workspace Lints](ch08-compile-time-and-developer-tools.md)。

---

### 2. 编译一次，随处测试

**问题**：`cargo test` 在切换 `--lib`、`--doc` 和 `--test` 时重新编译，因为它们使用不同的 profile。

**修复**：使用 `cargo nextest` 进行单元/集成测试，单独运行 doc 测试：

```bash
cargo nextest run --workspace        # 快速：并行、缓存
cargo test --workspace --doc         # Doc 测试（nextest 无法运行这些）
```

> `cargo-nextest` 设置参见 [编译时工具](ch08-compile-time-and-developer-tools.md)。

---

### 3. Feature 标志卫生

**问题**：一个库 crate 有 `default = ["std"]` 但没人测试 `--no-default-features`。某天嵌入式用户报告无法编译。

**修复**：在 CI 中添加 `cargo-hack`：

```yaml
- name: Feature matrix
  run: |
    cargo hack check --each-feature --no-dev-deps
    cargo check --no-default-features
    cargo check --all-features
```

> 完整模式参见 [`no_std` 与 Feature 验证](ch09-no-std-and-feature-verification.md)。

---

### 4. Lock 文件争论——提交还是忽略？

**经验法则：**

| Crate 类型 | 提交 `Cargo.lock`？ | 为什么 |
|------------|---------------------|--------|
| 二进制/应用程序 | **是** | 可重现的构建 |
| 库 | **否**（`.gitignore`） | 让下游选择版本 |
| 同时包含两者的 Workspace | **是** | 二进制优先 |

添加 CI 检查以确保 lock 文件保持最新：

```yaml
- name: Check lock file
  run: cargo update --locked  # 如果 Cargo.lock 过期则失败
```

---

### 5. 带有优化依赖的 Debug 构建

**问题**：Debug 构建慢得令人痛苦，因为依赖（尤其是 `serde`、`regex`）没有优化。

**修复**：在 dev profile 中优化依赖，同时保持代码未优化以快速重新编译：

```toml
# Cargo.toml
[profile.dev.package."*"]
opt-level = 2  # 在 dev 模式下优化所有依赖
```

这会稍微减慢第一次构建，但使运行时在开发期间显著更快。对于数据库支持的服务和解析器尤其有影响。

> 每个 crate 的 profile 覆盖参见 [Release Profiles](ch07-release-profiles-and-binary-size.md)。

---

### 6. CI 缓存抖动

**问题**：`Swatinem/rust-cache@v2` 在每个 PR 上保存新缓存，膨胀存储并减慢恢复时间。

**修复**：仅从 `main` 保存缓存，从任何地方恢复：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

对于有多个二进制文件的 workspace，添加 `shared-key`：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci-${{ matrix.target }}"
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

> 完整工作流参见 [CI/CD Pipeline](ch11-putting-it-all-together-a-production-cic.md)。

---

### 7. `RUSTFLAGS` vs `CARGO_ENCODED_RUSTFLAGS`

**问题**：`RUSTFLAGS="-Dwarnings"` 应用于*所有东西*——包括构建脚本和 proc-macro。`serde_derive` 的 build.rs 中的警告会导致 CI 失败。

**修复**：使用 `CARGO_ENCODED_RUSTFLAGS`，它只应用于顶层 crate：

```bash
# 糟糕——在第三方构建脚本警告时失败
RUSTFLAGS="-Dwarnings" cargo build

# 良好——只影响你的 crate
CARGO_ENCODED_RUSTFLAGS="-Dwarnings" cargo build

# 也良好——workspace lints（Cargo.toml）
[workspace.lints.rust]
warnings = "deny"
```

---

### 8. 使用 `SOURCE_DATE_EPOCH` 的可重现构建

**问题**：在 `build.rs` 中嵌入 `chrono::Utc::now()` 使构建不可重现——每次构建产生不同的二进制哈希。

**修复**：遵从 `SOURCE_DATE_EPOCH`：

```rust
// build.rs
let timestamp = std::env::var("SOURCE_DATE_EPOCH")
    .ok()
    .and_then(|s| s.parse::<i64>().ok())
    .unwrap_or_else(|| chrono::Utc::now().timestamp());
println!("cargo:rustc-env=BUILD_TIMESTAMP={timestamp}");
```

> 完整 build.rs 模式参见 [构建脚本](ch01-build-scripts-buildrs-in-depth.md)。

---

### 9. `cargo tree` 去重工作流

**问题**：`cargo tree --duplicates` 显示 5 个版本的 `syn` 和 3 个 `tokio-util`。编译时间痛苦。

**修复**：系统性去重：

```bash
# 步骤 1：查找重复
cargo tree --duplicates

# 步骤 2：找出谁拉取了旧版本
cargo tree --invert --package syn@1.0.109

# 步骤 3：更新罪魁祸首
cargo update -p serde_derive  # 可能拉取 syn 2.x

# 步骤 4：如果没有可用更新，在 [patch] 中锁定
# [patch.crates-io]
# old-crate = { git = "...", branch = "syn2-migration" }

# 步骤 5：验证
cargo tree --duplicates  # 应该更短
```

> `cargo-deny` 和供应链安全参见 [依赖管理](ch06-dependency-management-and-supply-chain-s.md)。

---

### 10. 推送前冒烟测试

**问题**：你推送了，CI 花了 10 分钟，因为格式问题失败了。

**修复**：在推送前本地运行快速检查：

```toml
# Makefile.toml (cargo-make)
[tasks.pre-push]
description = "推送前本地冒烟测试"
script = '''
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace --lib
'''
```

```bash
cargo make pre-push  # < 30 秒
git push
```

或者使用 git pre-push hook：

```bash
#!/bin/sh
# .git/hooks/pre-push
cargo fmt --all -- --check && cargo clippy --workspace -- -D warnings
```

> `Makefile.toml` 模式参见 [CI/CD Pipeline](ch11-putting-it-all-together-a-production-cic.md)。

---

### 🏋️ 练习

#### 🟢 练习 1：应用三个技巧

从本章中选择三个技巧并应用到现有的 Rust 项目。哪个影响最大？

<details>
<summary>解决方案</summary>

典型的高影响组合：

1. **`[profile.dev.package."*"] opt-level = 2`** —— dev 模式运行时的即时改进（对于解析密集型代码快 2-10 倍）

2. **`CARGO_ENCODED_RUSTFLAGS`** —— 消除来自第三方警告的虚假 CI 失败

3. **`cargo-hack --each-feature`** —— 通常在具有 3+ 个 feature 的任何项目中至少发现一个破坏的 feature 组合

```bash
# 应用技巧 5：
echo '[profile.dev.package."*"]' >> Cargo.toml
echo 'opt-level = 2' >> Cargo.toml

# 应用技巧 7 在 CI 中：
# 用 CARGO_ENCODED_RUSTFLAGS 替换 RUSTFLAGS

# 应用技巧 3：
cargo install cargo-hack
cargo hack check --each-feature --no-dev-deps
```
</details>

#### 🟡 练习 2：去重你的依赖树

在真实项目上运行 `cargo tree --duplicates`。消除至少一个重复项。测量前后的编译时间。

<details>
<summary>解决方案</summary>

```bash
# 之前
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 计数重复行

# 查找并修复一个重复
cargo tree --duplicates
cargo tree --invert --package <duplicate-crate>@<old-version>
cargo update -p <parent-crate>

# 之后
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 应该更少

# 典型结果：每消除一个重复项编译时间减少 5-15%
# （尤其是对于 syn、tokio 等重型 crate）
```
</details>

### 关键要点

- 使用 `CARGO_ENCODED_RUSTFLAGS` 而不是 `RUSTFLAGS` 避免破坏第三方构建脚本
- `[profile.dev.package."*"] opt-level = 2` 是单个影响最大的开发体验技巧
- 缓存调优（仅 `save-if` 在 main 上）防止活跃仓库的 CI 缓存膨胀
- `cargo tree --duplicates` + `cargo update` 是免费的编译时间胜利——每月做一次
- 使用 `cargo make pre-push` 在本地运行快速检查，避免 CI 往返浪费

---
