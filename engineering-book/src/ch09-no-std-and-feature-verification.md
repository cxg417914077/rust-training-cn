# `no_std` 与 Feature 验证 🔴

> **你将学到：**
> - 使用 `cargo-hack` 系统性地验证 feature 组合
> - Rust 的三个层级：`core` vs `alloc` vs `std` 以及各自的使用场景
> - 构建带有自定义 panic 处理器和分配器的 `no_std` crate
> - 在主机上使用 QEMU 测试 `no_std` 代码
>
> **交叉引用：**[Windows 与条件编译](ch10-windows-and-conditional-compilation.md) — 这个话题的平台另一半 · [跨平台编译](ch02-cross-compilation-one-source-many-target.md) — 跨平台编译到 ARM 和嵌入式目标 · [Miri 和清理器](ch05-miri-valgrind-and-sanitizers-verifying-u.md) — 在 `no_std` 环境中验证 `unsafe` 代码 · [构建脚本](ch01-build-scripts-buildrs-in-depth.md) — `build.rs` 发出的 `cfg` 标志

Rust 可以运行在任何地方，从 8 位微控制器到云端服务器。本章介绍基础：通过 `#![no_std]` 剥离标准库，以及验证你的 feature 组合确实能编译。

### 使用 `cargo-hack` 验证 Feature 组合

[`cargo-hack`](https://github.com/taiki-e/cargo-hack) 系统地测试所有 feature 组合——对于带有 `#[cfg(...)]` 代码的 crate 至关重要：

```bash
# 安装
cargo install cargo-hack

# 检查每个 feature 单独编译
cargo hack check --each-feature --workspace

# 核弹选项：测试 ALL feature 组合（指数级！）
# 只适用于 <8 个 feature 的 crate。
cargo hack check --feature-powerset --workspace

# 实用的妥协：单独测试每个 feature + 所有 feature + 无 feature
cargo hack check --each-feature --workspace --no-dev-deps
cargo check --workspace --all-features
cargo check --workspace --no-default-features
```

**为什么这对项目很重要：**

如果你添加平台相关的 feature（`linux`、`windows`、`direct-ipmi`、`direct-accel-api`），
`cargo-hack` 能发现破坏的组合：

```toml
# 示例：控制平台代码的 feature
[features]
default = ["linux"]
linux = []                          # Linux 特定的硬件访问
windows = ["dep:windows-sys"]       # Windows 特定的 API
direct-ipmi = []                    # unsafe IPMI ioctl (ch05)
direct-accel-api = []               # unsafe accel-mgmt FFI (ch05)
```

```bash
# 验证所有 feature 单独编译以及一起编译
cargo hack check --each-feature -p diag_tool
# 发现："'windows' feature 在没有 'direct-ipmi' 时无法编译"
# 发现："#[cfg(feature = \"linux\")] 有拼写错误——写成了 'lnux'"
```

**CI 集成：**

```yaml
# 添加到 CI 流水线（快速——只是编译检查）
- name: Feature matrix check
  run: cargo hack check --each-feature --workspace --no-dev-deps
```

> **经验法则**：对于任何有 2+ 个 feature 的 crate，在 CI 中运行 `cargo hack check --each-feature`。
> 对于核心库 crate 且 feature 数量 <8 个时运行 `--feature-powerset`——它是指数级的（$2^n$ 组合）。

### `no_std` —— 何时以及为什么

`#![no_std]` 告诉编译器："不要链接标准库。"你的 crate 只能使用 `core`（以及可选的 `alloc`）。为什么需要这样？

| 场景 | 为什么用 `no_std` |
|------|-------------|
| 嵌入式固件（ARM Cortex-M, RISC-V） | 无操作系统、无堆、无文件系统 |
| UEFI 诊断工具 | 预启动环境，无操作系统 API |
| 内核模块 | 内核空间不能使用用户态 `std` |
| WebAssembly (WASM) | 最小化二进制大小，无操作系统依赖 |
| Bootloader | 在任何操作系统存在之前运行 |
| 带有 C 接口的共享库 | 避免调用者中的 Rust 运行时 |

**对于硬件诊断**，`no_std` 在以下情况变得相关：
- 基于 UEFI 的预启动诊断工具（在操作系统加载之前）
- BMC 固件诊断（资源受限的 ARM SoC）
- 内核级 PCIe 诊断（内核模块或 eBPF 探针）

### `core` vs `alloc` vs `std` —— 三个层级

```text
┌─────────────────────────────────────────────────────────────┐
│ std                                                         │
│  core + alloc 中的所有内容，PLUS：                           │
│  • 文件 I/O (std::fs, std::io)                              │
│  • 网络 (std::net)                                          │
│  • 线程 (std::thread)                                       │
│  • 时间 (std::time)                                         │
│  • 环境变量 (std::env)                                      │
│  • 进程 (std::process)                                      │
│  • 操作系统特定 (std::os::unix, std::os::windows)           │
├─────────────────────────────────────────────────────────────┤
│ alloc          (可用 #![no_std] + extern crate    │
│                 alloc，如果有全局分配器)                       │
│  • String, Vec, Box, Rc, Arc                                │
│  • BTreeMap, BTreeSet                                       │
│  • format!() 宏                                             │
│  • 需要堆的集合和智能指针                                    │
├─────────────────────────────────────────────────────────────┤
│ core           (始终可用，即使在 #![no_std] 中)               │
│  • 原始类型（u8, bool, char 等）                             │
│  • Option, Result                                           │
│  • Iterator, slice, array, str（切片，不是 String）          │
│  • Traits: Clone, Copy, Debug, Display, From, Into          │
│  • 原子操作 (core::sync::atomic)                            │
│  • Cell, RefCell (core::cell)  — Pin (core::pin)            │
│  • core::fmt（无分配的格式化）                               │
│  • core::mem, core::ptr（底层内存操作）                      │
│  • 数学：core::num, 基本算术                                 │
└─────────────────────────────────────────────────────────────┘
```

**没有 `std` 会失去什么：**
- 没有 `HashMap`（需要哈希器——使用 `alloc` 中的 `BTreeMap`，或 `hashbrown`）
- 没有 `println!()`（需要 stdout——使用 `core::fmt::Write` 写入缓冲区）
- 没有 `std::error::Error`（自 Rust 1.81 起在 `core` 中稳定，但许多生态系统尚未迁移）
- 没有文件 I/O、没有网络、没有线程（除非由平台 HAL 提供）
- 没有 `Mutex`（使用 `spin::Mutex` 或平台特定的锁）

### 构建 `no_std` Crate

```rust
// src/lib.rs — 一个 no_std 库 crate
#![no_std]

// 可选地使用堆分配
extern crate alloc;
use alloc::string::String;
use alloc::vec::Vec;
use core::fmt;

/// 来自温度传感器的温度读数。
/// 这个结构体在任何环境中都能工作——从裸机到 Linux。
#[derive(Clone, Copy, Debug)]
pub struct Temperature {
    /// 原始传感器值（典型 I2C 传感器为 0.0625°C/LSB）
    raw: u16,
}

impl Temperature {
    pub const fn from_raw(raw: u16) -> Self {
        Self { raw }
    }

    /// 转换为摄氏度（定点数，无需 FPU）
    pub const fn millidegrees_c(&self) -> i32 {
        (self.raw as i32) * 625 / 10 // 0.0625°C 分辨率
    }

    pub fn degrees_c(&self) -> f32 {
        self.raw as f32 * 0.0625
    }
}

impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let md = self.millidegrees_c();
        // 正确处理 -0.999°C 到 -0.001°C 之间的值
        // 这些值 md / 1000 == 0 但值为负
        if md < 0 && md > -1000 {
            write!(f, "-0.{:03}°C", (-md) % 1000)
        } else {
            write!(f, "{}.{:03}°C", md / 1000, (md % 1000).abs())
        }
    }
}

/// 解析空格分隔的温度值。
/// 使用 alloc——需要全局分配器。
pub fn parse_temperatures(input: &str) -> Vec<Temperature> {
    input
        .split_whitespace()
        .filter_map(|s| s.parse::<u16>().ok())
        .map(Temperature::from_raw)
        .collect()
}

/// 无分配格式化——直接写入缓冲区。
/// 适用于 `core` 仅限环境（无 alloc，无堆）。
pub fn format_temp_into(temp: &Temperature, buf: &mut [u8]) -> usize {
    use core::fmt::Write;
    struct SliceWriter<'a> {
        buf: &'a mut [u8],
        pos: usize,
    }
    impl<'a> Write for SliceWriter<'a> {
        fn write_str(&mut self, s: &str) -> fmt::Result {
            let bytes = s.as_bytes();
            let remaining = self.buf.len() - self.pos;
            if bytes.len() > remaining {
                // 缓冲区已满——返回错误而不是静默截断
                // 调用者可以检查返回的 pos 来判断是否部分写入
                return Err(fmt::Error);
            }
            self.buf[self.pos..self.pos + bytes.len()].copy_from_slice(bytes);
            self.pos += bytes.len();
            Ok(())
        }
    }
    let mut w = SliceWriter { buf, pos: 0 };
    let _ = write!(w, "{}", temp);
    w.pos
}
```

```toml
# Cargo.toml 用于 no_std crate
[package]
name = "thermal-sensor"
version = "0.1.0"
edition = "2021"

[features]
default = ["alloc"]
alloc = []    # 启用 Vec, String 等
std = []      # 启用完整 std（隐含 alloc）

[dependencies]
# 使用 no_std 兼容的 crate
serde = { version = "1.0", default-features = false, features = ["derive"] }
# ↑ default-features = false 移除 std 依赖！
```

> **关键 crate 模式**：许多流行的 crate（serde、log、rand、embedded-hal）通过
> `default-features = false` 支持 `no_std`。在 `no_std` 上下文中使用之前，
> 始终检查依赖是否需要 `std`。注意有些 crate（如 `regex`）至少需要 `alloc`，
> 在 `core` 仅限环境中无法工作。

### 自定义 Panic 处理器和分配器

在 `#![no_std]` 二进制文件（不是库）中，你必须提供 panic 处理器和可选的全局分配器：

```rust
// src/main.rs — 一个 no_std 二进制文件（例如 UEFI 诊断）
#![no_std]
#![no_main]

extern crate alloc;

use core::panic::PanicInfo;

// 必需：panic 时做什么（无栈展开可用）
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    // 在嵌入式中：闪烁 LED、写入 UART、挂起
    // 在 UEFI 中：写入控制台、停止
    // 最简：只是永远循环
    loop {
        core::hint::spin_loop();
    }
}

// 如果需要使用 alloc：提供全局分配器
use alloc::alloc::{GlobalAlloc, Layout};

struct BumpAllocator {
    // 简单的 bump 分配器用于嵌入式/UEFI
    // 在实践中，使用 crate 如 `linked_list_allocator` 或 `embedded-alloc`
}

// 警告：这是一个非功能性的占位符！调用 alloc() 将返回 null，
// 导致立即 UB（全局分配器合约要求非零大小分配返回非 null）。
// 在实际代码中，使用成熟的分配器 crate：
//   - embedded-alloc (嵌入式目标)
//   - linked_list_allocator (UEFI / 内核)
//   - talc (通用 no_std)
unsafe impl GlobalAlloc for BumpAllocator {
    /// # 安全
    /// Layout 必须具有非零大小。返回 null（占位符——会崩溃）。
    unsafe fn alloc(&self, _layout: Layout) -> *mut u8 {
        // 占位符——会崩溃！用真正的分配逻辑替换。
        core::ptr::null_mut()
    }
    /// # 安全
    /// `_ptr` 必须由 `alloc` 返回且具有兼容的 layout。
    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // bump 分配器无需操作
    }
}

#[global_allocator]
static ALLOCATOR: BumpAllocator = BumpAllocator {};

// 入口点（平台特定，不是 fn main）
// 对于 UEFI: #[entry] 或 efi_main
// 对于嵌入式：#[cortex_m_rt::entry]
```

### 测试 `no_std` 代码

测试在主机上运行，主机有 `std`。技巧：你的库是 `no_std`，但测试工具使用 `std`：

```rust
// 你的 crate: src/lib.rs 中 #![no_std]
// 但测试自动在 std 下运行：

#[cfg(test)]
mod tests {
    use super::*;
    // std 在这里可用——println!()、assert!()、Vec 都能工作

    #[test]
    fn test_temperature_conversion() {
        let temp = Temperature::from_raw(800); // 50.0°C
        assert_eq!(temp.millidegrees_c(), 50000);
        assert!((temp.degrees_c() - 50.0).abs() < 0.01);
    }

    #[test]
    fn test_format_into_buffer() {
        let temp = Temperature::from_raw(800);
        let mut buf = [0u8; 32];
        let len = format_temp_into(&temp, &mut buf);
        let s = core::str::from_utf8(&buf[..len]).unwrap();
        assert_eq!(s, "50.000°C");
    }
}
```

**在实际目标上测试**（当 `std` 完全不可用时）：

```bash
# 使用 defmt-test 进行设备上测试（嵌入式 ARM）
# 使用 uefi-test-runner 进行 UEFI 目标
# 使用 QEMU 进行跨架构测试（无需硬件）

# 在主机上运行 no_std 库测试（始终可行）：
cargo test --lib

# 针对 no_std 目标验证 no_std 编译：
cargo check --target thumbv7em-none-eabihf  # ARM Cortex-M
cargo check --target riscv32imac-unknown-none-elf  # RISC-V
```

### `no_std` 决策树

```mermaid
flowchart TD
    START["你的代码需要<br/>标准库吗？"] --> NEED_FS{"文件系统、<br/>网络、线程？"}
    NEED_FS -->|"是"| USE_STD["使用 std<br/>普通应用程序"]
    NEED_FS -->|"否"| NEED_HEAP{"需要堆分配吗？<br/>Vec, String, Box"}
    NEED_HEAP -->|"是"| USE_ALLOC["#![no_std]<br/>extern crate alloc"]
    NEED_HEAP -->|"否"| USE_CORE["#![no_std]<br/>仅 core"]
    
    USE_ALLOC --> VERIFY["cargo-hack<br/>--each-feature"]
    USE_CORE --> VERIFY
    USE_STD --> VERIFY
    VERIFY --> TARGET{"目标有操作系统？"}
    TARGET -->|"是"| HOST_TEST["cargo test --lib<br/>标准测试"]
    TARGET -->|"否"| CROSS_TEST["QEMU / defmt-test<br/>设备上测试"]
    
    style USE_STD fill:#91e5a3,color:#000
    style USE_ALLOC fill:#ffd43b,color:#000
    style USE_CORE fill:#ff6b6b,color:#000
```

### 🏋️ 练习

#### 🟡 练习 1：Feature 组合验证

安装 `cargo-hack` 并在具有多个 feature 的项目上运行 `cargo hack check --each-feature --workspace`。它发现了任何破坏的组合吗？

<details>
<summary>解决方案</summary>

```bash
cargo install cargo-hack

# 单独检查每个 feature
cargo hack check --each-feature --workspace --no-dev-deps

# 如果 feature 组合失败：
# error[E0433]: failed to resolve: use of undeclared crate or module `std`
# → 这意味着 feature 门缺少 #[cfg] 守卫

# 检查所有 feature + 无 feature + 单独检查：
cargo hack check --each-feature --workspace
cargo check --workspace --all-features
cargo check --workspace --no-default-features
```
</details>

#### 🔴 练习 2：构建 `no_std` 库

创建一个使用 `#![no_std]` 编译的库 crate。实现一个简单的栈分配环形缓冲区。验证它能针对 `thumbv7em-none-eabihf`（ARM Cortex-M）编译。

<details>
<summary>解决方案</summary>

```rust
// lib.rs
#![no_std]

pub struct RingBuffer<const N: usize> {
    data: [u8; N],
    head: usize,
    len: usize,
}

impl<const N: usize> RingBuffer<N> {
    pub const fn new() -> Self {
        Self { data: [0; N], head: 0, len: 0 }
    }

    pub fn push(&mut self, byte: u8) -> bool {
        if self.len == N { return false; }
        let idx = (self.head + self.len) % N;
        self.data[idx] = byte;
        self.len += 1;
        true
    }

    pub fn pop(&mut self) -> Option<u8> {
        if self.len == 0 { return None; }
        let byte = self.data[self.head];
        self.head = (self.head + 1) % N;
        self.len -= 1;
        Some(byte)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn push_pop() {
        let mut rb = RingBuffer::<4>::new();
        assert!(rb.push(1));
        assert!(rb.push(2));
        assert_eq!(rb.pop(), Some(1));
        assert_eq!(rb.pop(), Some(2));
        assert_eq!(rb.pop(), None);
    }
}
```

```bash
rustup target add thumbv7em-none-eabihf
cargo check --target thumbv7em-none-eabihf
# ✅ 为裸机 ARM 编译
```
</details>

### 关键要点

- `cargo-hack --each-feature` 对于任何有条件编译的 crate 都至关重要——在 CI 中运行它
- `core` → `alloc` → `std` 是分层的：每一层都添加功能但需要更多的运行时支持
- 自定义 panic 处理器和分配器是裸机 `no_std` 二进制文件所必需的
- 使用 `cargo test --lib` 在主机上测试 `no_std` 库——无需硬件
- 只对核心库 crate 且 feature 数量 <8 个时运行 `--feature-powerset`——它是 $2^n$ 组合

---
