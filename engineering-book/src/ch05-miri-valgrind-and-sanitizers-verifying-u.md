# Miri、Valgrind 和清理器——验证 Unsafe 代码 🔴

> **你将学到：**
> - Miri 作为 MIR 解释器——它能捕获什么（别名、UB、泄漏）以及它不能捕获什么（FFI、系统调用）
> - Valgrind memcheck、Helgrind（数据竞争）、Callgrind（性能分析）和 Massif（堆）
> - LLVM 清理器：ASan、MSan、TSan、LSan 与 nightly 的 `-Zbuild-std`
> - `cargo-fuzz` 用于发现崩溃，`loom` 用于并发性模型检查
> - 选择正确验证工具的决策树
>
> **交叉引用：**[代码覆盖率](ch04-code-coverage-seeing-what-tests-miss.md) — 覆盖率发现未测试路径，Miri 验证已测试路径 · [`no_std` 和 Feature](ch09-no-std-and-feature-verification.md) — `no_std` 代码通常需要 Miri 验证的 `unsafe` · [CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) — 流水线中的 Miri 任务

Safe Rust 在编译时保证内存安全和无数据竞争。但一旦你编写 `unsafe`——用于 FFI、手写数据结构或性能优化——这些保证就变成了*你的*责任。本章介绍验证你的 `unsafe` 代码实际上遵守它所声称的安全合约的工具。

### Miri — Unsafe Rust 的解释器

[Miri](https://github.com/rust-lang/miri) 是 Rust 中级中间表示（MIR）的**解释器**。Miri 不编译成机器码，而是*逐步执行*你的程序，在每次操作时进行穷尽的未定义行为检查。

```bash
# 安装 Miri（仅 nightly 组件）
rustup +nightly component add miri

# 在 Miri 下运行测试套件
cargo +nightly miri test

# 在 Miri 下运行特定二进制文件
cargo +nightly miri run

# 运行特定测试
cargo +nightly miri test -- test_name
```

**Miri 工作原理：**

```text
源码 → rustc → MIR → Miri 解释 MIR
                        │
                        ├─ 跟踪每个指针的来源
                        ├─ 验证每次内存访问
                        ├─ 在每次解引用时检查对齐
                        ├─ 检测释放后使用
                        ├─ 检测数据竞争（带线程）
                        └─ 执行 Stacked Borrows / Tree Borrows 规则
```

### Miri 能捕获什么（以及不能捕获什么）

**Miri 检测：**

| 类别 | 示例 | 会在运行时崩溃吗？ |
|------|------|-------------------|
| 越界访问 | `ptr.add(100).read()` 超出分配 | 有时（取决于页面布局） |
| 释放后使用 | 通过裸指针读取已 drop 的 `Box` | 有时（取决于分配器） |
| 双重释放 | 调用 `drop_in_place` 两次 | 通常 |
| 未对齐访问 | `(ptr as *const u32).read()` 在奇数地址 | 在某些架构上 |
| 无效值 | `transmute::<u8, bool>(2)` | 静默错误 |
| 悬垂引用 | `&*ptr` 其中 ptr 已释放 | 不（静默损坏） |
| 数据竞争 | 两个线程，一个写入，无同步 | 间歇性，难以重现 |
| Stacked Borrows 违规 | 别名 `&mut` 引用 | 不（静默损坏） |

**Miri 不检测：**

| 限制 | 原因 |
|------|------|
| 逻辑 bug | Miri 检查内存安全，而非正确性 |
| 并发死锁 | Miri 检查数据竞争，而非活锁 |
| 性能问题 | 解释比原生慢 10-100 倍 |
| OS/硬件交互 | Miri 不能模拟系统调用、设备 I/O |
| 所有 FFI 调用 | 不能解释 C 代码（仅 Rust MIR） |
| 穷尽路径覆盖 | 只测试你的测试套件触及的路径 |

**具体示例——捕获实践中"能工作"的不可靠代码：**

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_miri_catches_ub() {
        // 这在 release 构建中"能工作"但是未定义行为
        let mut v = vec![1, 2, 3];
        let ptr = v.as_ptr();

        // Push 可能重新分配，使命 ptr 无效
        v.push(4);

        // ❌ UB：ptr 可能在重新分配后悬垂
        // Miri 会捕获这个，即使分配器恰好没有移动缓冲区
        // let _val = unsafe { *ptr };
        // 错误：Miri 会报告：
        //   "pointer to alloc1234 was dereferenced after this
        //    allocation got freed"
        
        // ✅ 正确：在变异后获取新的指针
        let ptr = v.as_ptr();
        let val = unsafe { *ptr };
        assert_eq!(val, 1);
    }
}
```

### 在真实 Crate 上运行 Miri

**对带有 `unsafe` 的 crate 的实用 Miri 工作流：**

```bash
# 步骤 1：在 Miri 下运行所有测试
cargo +nightly miri test 2>&1 | tee miri_output.txt

# 步骤 2：如果 Miri 报告错误，隔离它们
cargo +nightly miri test -- failing_test_name

# 步骤 3：使用 Miri 的回溯进行诊断
MIRIFLAGS="-Zmiri-backtrace=full" cargo +nightly miri test

# 步骤 4：选择借用模型
# Stacked Borrows（默认，更严格）：
cargo +nightly miri test

# Tree Borrows（实验性，更宽松）：
MIRIFLAGS="-Zmiri-tree-borrows" cargo +nightly miri test
```

**常见场景的 Miri 标志：**

```bash
# 禁用隔离（允许文件系统访问、环境变量）
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test

# 内存泄漏检测在 Miri 中默认开启
# 要抑制泄漏错误（例如对于故意泄漏）：
# MIRIFLAGS="-Zmiri-ignore-leaks" cargo +nightly miri test

# 为随机测试种子化 RNG 以获得可重现的结果
MIRIFLAGS="-Zmiri-seed=42" cargo +nightly miri test

# 启用严格的来源检查
MIRIFLAGS="-Zmiri-strict-provenance" cargo +nightly miri test

# 多个标志
MIRIFLAGS="-Zmiri-disable-isolation -Zmiri-backtrace=full -Zmiri-strict-provenance" \
    cargo +nightly miri test
```

**CI 中的 Miri：**

```yaml
# .github/workflows/miri.yml
name: Miri
on: [push, pull_request]

jobs:
  miri:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
        with:
          components: miri

      - name: Run Miri
        run: cargo miri test --workspace
        env:
          MIRIFLAGS: "-Zmiri-backtrace=full"
          # 泄漏检测默认开启
          # 跳过 Miri 无法处理的系统调用测试
          # （文件 I/O、网络等）
```

> **性能说明**：Miri 比原生执行慢 10-100 倍。原生 5 秒运行的测试套件在 Miri 下可能需要 5 分钟。在 CI 中，只在焦点子集上运行 Miri：只有包含 `unsafe` 代码的 crate。

### Valgrind 及其 Rust 集成

[Valgrind](https://valgrind.org/) 是经典的 C/C++ 内存检查器。它也适用于编译后的 Rust 二进制文件，在机器码级别检查内存错误。

```bash
# 安装 Valgrind
sudo apt install valgrind  # Debian/Ubuntu
sudo dnf install valgrind  # Fedora

# 使用调试信息构建（Valgrind 需要符号）
cargo build --tests
# 或带调试信息的 release：
# cargo build --release
# [profile.release]
# debug = true

# 在 Valgrind 下运行特定测试二进制文件
valgrind --tool=memcheck \
    --leak-check=full \
    --show-leak-kinds=all \
    --track-origins=yes \
    ./target/debug/deps/my_crate-abc123 --test-threads=1

# 运行主二进制文件
valgrind --tool=memcheck \
    --leak-check=full \
    --error-exitcode=1 \
    ./target/debug/diag_tool --run-diagnostics
```

**Valgrind 超越 memcheck 的工具：**

| 工具 | 命令 | 检测内容 |
|------|------|----------|
| **Memcheck** | `--tool=memcheck` | 内存泄漏、释放后使用、缓冲区溢出 |
| **Helgrind** | `--tool=helgrind` | 数据竞争和锁顺序违规 |
| **DRD** | `--tool=drd` | 数据竞争（不同的检测算法） |
| **Callgrind** | `--tool=callgrind` | CPU 指令性能分析（路径级别） |
| **Massif** | `--tool=massif` | 堆内存性能分析随时间变化 |
| **Cachegrind** | `--tool=cachegrind` | 缓存未命中分析 |

**使用 Callgrind 进行指令级别性能分析：**

```bash
# 记录指令计数（比挂钟时间更稳定）
valgrind --tool=callgrind \
    --callgrind-out-file=callgrind.out \
    ./target/release/diag_tool --run-diagnostics

# 使用 KCachegrind 可视化
kcachegrind callgrind.out
# 或基于文本的替代方案：
callgrind_annotate callgrind.out | head -100
```

**Miri vs Valgrind — 何时使用哪个：**

| 方面 | Miri | Valgrind |
|------|------|----------|
| 检查 Rust 特定 UB | ✅ Stacked/Tree Borrows | ❌ 不知道 Rust 规则 |
| 检查 C FFI 代码 | ❌ 不能解释 C | ✅ 检查所有机器码 |
| 需要 nightly | ✅ 是 | ❌ 否 |
| 速度 | 慢 10-100 倍 | 慢 10-50 倍 |
| 平台 | 任意（解释 MIR） | Linux、macOS（运行原生代码） |
| 数据竞争检测 | ✅ 是 | ✅ 是（Helgrind/DRD） |
| 泄漏检测 | ✅ 是 | ✅ 是（更彻底） |
| 假阳性 | 非常罕见 | 偶尔（尤其是分配器） |

**两者都使用**：
- **Miri** 用于纯 Rust `unsafe` 代码（Stacked Borrows、来源）
- **Valgrind** 用于 FFI 重代码和整体程序泄漏分析

### AddressSanitizer、MemorySanitizer、ThreadSanitizer

LLVM 清理器是编译时工具传递，在运行时插入检查。它们比 Valgrind 快（2-5 倍开销 vs 10-50 倍）并捕获不同类别的 bug。

```bash
# 必需：安装 Rust 源码以重建带清理器工具的 std
rustup component add rust-src --toolchain nightly

# AddressSanitizer (ASan) — 缓冲区溢出、释放后使用、栈溢出
RUSTFLAGS="-Zsanitizer=address" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# MemorySanitizer (MSan) — 未初始化内存读取
RUSTFLAGS="-Zsanitizer=memory" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# ThreadSanitizer (TSan) — 数据竞争
RUSTFLAGS="-Zsanitizer=thread" \
    cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# LeakSanitizer (LSan) — 内存泄漏（默认包含在 ASan 中）
RUSTFLAGS="-Zsanitizer=leak" \
    cargo +nightly test --target x86_64-unknown-linux-gnu
```

> **注意**：ASan、MSan 和 TSan 需要 `-Zbuild-std` 来重建带清理器工具的标准库。LSan 不需要。

**清理器对比：**

| 清理器 | 开销 | 捕获 | 需要 nightly？ | `-Zbuild-std`？ |
|--------|------|------|---------------|----------------|
| **ASan** | 2 倍内存，2 倍 CPU | 缓冲区溢出、释放后使用、栈溢出 | 是 | 是 |
| **MSan** | 3 倍内存，3 倍 CPU | 未初始化读取 | 是 | 是 |
| **TSan** | 5-10 倍内存，5 倍 CPU | 数据竞争 | 是 | 是 |
| **LSan** | 最小 | 内存泄漏 | 是 | 否 |

**具体示例——用 TSan 捕获数据竞争：**

```rust
use std::sync::Arc;
use std::thread;

fn racy_counter() -> u64 {
    // ❌ UB：无同步的共享可变状态
    let data = Arc::new(std::cell::UnsafeCell::new(0u64));
    let mut handles = vec![];

    for _ in 0..4 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                // 安全性：不可靠 — 数据竞争！
                unsafe {
                    *data.get() += 1;
                }
            }
        }));
    }

    for h in handles {
        h.join().unwrap();
    }

    // 值应该是 4000 但由于竞争可能是任何值
    unsafe { *data.get() }
}

// Miri 和 TSan 都捕获这个：
// Miri:  "Data race detected between (1) write and (2) write"
// TSan:  "WARNING: ThreadSanitizer: data race"
//
// 修复：使用 AtomicU64 或 Mutex<u64>
```

### 相关工具：模糊测试和并发性验证

**`cargo-fuzz` — 覆盖率引导的模糊测试**（发现解析器和解码器中的崩溃）：

```bash
# 安装
cargo install cargo-fuzz

# 初始化模糊测试目标
cargo fuzz init
cargo fuzz add parse_gpu_csv
```

```rust
// fuzz/fuzz_targets/parse_gpu_csv.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        // 模糊测试器生成数百万个输入寻找崩溃
        let _ = diag_tool::parse_gpu_csv(s);
    }
});
```

```bash
# 运行模糊测试器（运行直到被中断或发现崩溃）
cargo +nightly fuzz run parse_gpu_csv -- -max_total_time=300  # 5 分钟

# 最小化崩溃
cargo +nightly fuzz tmin parse_gpu_csv artifacts/parse_gpu_csv/crash-...
```

> **何时模糊测试**：任何解析不受信任/半受信任输入（传感器输出、配置文件、网络数据、JSON/CSV）的函数。模糊测试在每个主要 Rust 解析器 crate（serde、regex、image）中都发现了真实的 bug。

**`loom` — 并发性模型检查器**（穷尽测试原子顺序）：

```toml
[dev-dependencies]
loom = "0.7"
```

```rust
#[cfg(loom)]
mod tests {
    use loom::sync::atomic::{AtomicUsize, Ordering};
    use loom::thread;

    #[test]
    fn test_counter_is_atomic() {
        loom::model(|| {
            let counter = loom::sync::Arc::new(AtomicUsize::new(0));
            let c1 = counter.clone();
            let c2 = counter.clone();

            let t1 = thread::spawn(move || { c1.fetch_add(1, Ordering::SeqCst); });
            let t2 = thread::spawn(move || { c2.fetch_add(1, Ordering::SeqCst); });

            t1.join().unwrap();
            t2.join().unwrap();

            // loom 探索所有可能的线程交错
            assert_eq!(counter.load(Ordering::SeqCst), 2);
        });
    }
}
```

> **何时使用 `loom`**：当你有无锁数据结构或自定义同步原语时。Loom 穷尽地探索线程交错——它是模型检查器，不是压力测试。基于 `Mutex`/`RwLock` 的代码不需要。

### 何时使用哪个工具

```text
无安全验证的决策树：

代码是纯 Rust（无 FFI）吗？
├─ 是 → 使用 Miri（捕获 Rust 特定 UB、Stacked Borrows）
│        在 CI 中运行 ASan 以进行纵深防御
└─ 否（通过 FFI 调用 C/C++ 代码）
   ├─ 内存安全问题？
   │  └─ 是 → 使用 Valgrind memcheck 和 ASan
   ├─ 并发性问题？
   │  └─ 是 → 使用 TSan（更快）或 Helgrind（更彻底）
   └─ 内存泄漏问题？
      └─ 是 → 使用 Valgrind --leak-check=full
```

**推荐的 CI 矩阵：**

```yaml
# 并行运行所有工具以获得快速反馈
jobs:
  miri:
    runs-on: ubuntu-latest
    steps:
      - uses: dtolnay/rust-toolchain@nightly
        with: { components: miri }
      - run: cargo miri test --workspace

  asan:
    runs-on: ubuntu-latest
    steps:
      - uses: dtolnay/rust-toolchain@nightly
      - run: |
          RUSTFLAGS="-Zsanitizer=address" \
          cargo test -Zbuild-std --target x86_64-unknown-linux-gnu

  valgrind:
    runs-on: ubuntu-latest
    steps:
      - run: sudo apt-get install -y valgrind
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo build --tests
      - run: |
          for test_bin in $(find target/debug/deps -maxdepth 1 -executable -type f ! -name '*.d'); do
            valgrind --error-exitcode=1 --leak-check=full "$test_bin" --test-threads=1
          done
```

### 应用：零 Unsafe——以及何时需要它

项目在 90K+ 行 Rust 中包含**零 `unsafe` 块**。这对于系统级诊断工具来说是非凡的成就，证明了 Safe Rust 足以应对：
- IPMI 通信（通过 `std::process::Command` 到 `ipmitool`）
- GPU 查询（通过 `std::process::Command` 到 `accel-query`）
- PCIe 拓扑解析（纯 JSON/文本解析）
- SEL 记录管理（纯数据结构）
- DER 报告生成（JSON 序列化）

**项目何时需要 `unsafe`？**

引入 `unsafe` 的可能触发点：

| 场景 | 为什么 `unsafe` | 推荐的验证 |
|------|----------------|------------|
| 直接基于 ioctl 的 IPMI | `libc::ioctl()` 绕过 `ipmitool` 子进程 | Miri + Valgrind |
| 直接 GPU 驱动查询 | accel-mgmt FFI 而非 `accel-query` 解析 | Valgrind（C 库） |
| 内存映射 PCIe 配置 | `mmap` 用于直接配置空间读取 | ASan + Valgrind |
| 无锁 SEL 缓冲区 | `AtomicPtr` 用于并发事件收集 | Miri + TSan |
| 嵌入式/no_std 变体 | 裸机操作的裸指针操作 | Miri |

**准备**：在引入 `unsafe` 之前，将验证工具添加到 CI：

```toml
# Cargo.toml — 为 unsafe 优化添加特性标志
[features]
default = []
direct-ipmi = []     # 启用直接 ioctl IPMI 而非 ipmitool 子进程
direct-accel-api = []     # 启用 accel-mgmt FFI 而非 accel-query 解析
```

```rust
// src/ipmi.rs — 在特性标志后
#[cfg(feature = "direct-ipmi")]
mod direct {
    //! 通过 /dev/ipmi0 ioctl 直接 IPMI 设备访问。
    //!
    //! # 安全性
    //! 此模块使用 `unsafe` 进行 ioctl 系统调用。
    //! 已验证：Miri（可能时）、Valgrind memcheck、ASan。

    use std::os::unix::io::RawFd;

    // ... unsafe ioctl 实现 ...
}

#[cfg(not(feature = "direct-ipmi"))]
mod subprocess {
    //! IPMI 通过 ipmitool 子进程（默认，完全安全）。
    // ... 当前实现 ...
}
```

> **关键见解**：将 `unsafe` 保持在 [特性标志](ch09-no-std-and-feature-verification.md) 后，以便可以独立验证。在 [CI](ch11-putting-it-all-together-a-production-cic.md) 中运行 `cargo +nightly miri test --features direct-ipmi` 以持续验证 unsafe 路径，而不影响安全的默认构建。

### `cargo-careful` — 在 Stable 上的额外 UB 检查

[`cargo-careful`](https://github.com/RalfJung/cargo-careful) 运行你的代码时启用额外的标准库检查——捕获一些正常构建忽略的未定义行为，不需要 nightly 或 Miri 的 10-100 倍减速：

```bash
# 安装（需要 nightly，但以接近原生的速度运行代码）
cargo install cargo-careful

# 运行带额外 UB 检查的测试（捕获未初始化内存、无效值）
cargo +nightly careful test

# 运行带额外检查的二进制文件
cargo +nightly careful run -- --run-diagnostics
```

**`cargo-careful` 捕获正常构建不捕获的内容：**
- `MaybeUninit` 和 `zeroed()` 中未初始化内存的读取
- 通过 transmute 创建无效 `bool`、`char` 或枚举值
- 未对齐指针读取/写入
- `copy_nonoverlapping` 与重叠范围

**它在验证阶梯中的位置：**

```text
最少开销                                          最彻底
├─ cargo test ──► cargo careful test ──► Miri ──► ASan ──► Valgrind ─┤
│  (0 倍开销)    (~1.5 倍开销)       (10-100 倍)  (2 倍)     (10-50 倍)   │
│  仅 Safe Rust  捕获一些 UB        纯 Rust      FFI+Rust  FFI+Rust   │
```

> **建议**：将 `cargo +nightly careful test` 添加到 CI 作为快速安全检查。它以接近原生的速度运行（不像 Miri）并捕获 Safe Rust 抽象掩盖的真实 bug。

### Miri 和清理器故障排除

| 症状 | 原因 | 修复 |
|------|------|------|
| `Miri does not support FFI` | Miri 是 Rust 解释器；不能执行 C 代码 | 对 FFI 代码改用 Valgrind 或 ASan |
| `error: unsupported operation: can't call foreign function` | Miri 遇到 `extern "C"` 调用 | 模拟 FFI 边界或在 `#[cfg(miri)]` 后 |
| `Stacked Borrows violation` | 别名规则违规——即使代码"能工作" | Miri 是正确的；重构以避免 `&mut` 与 `&` 别名 |
| Sanitizer 说 `DEADLYSIGNAL` | ASan 检测到缓冲区溢出 | 检查数组索引、切片操作和指针算术 |
| `LeakSanitizer: detected memory leaks` | `Box::leak()`、`forget()` 或丢失 `drop()` | 故意：用 `__lsan_disable()` 抑制；非故意：修复泄漏 |
| Miri 非常慢 | Miri 解释，不编译 — 慢 10-100 倍 | 只在 `--lib` 测试上运行，或用 `#[cfg_attr(miri, ignore)]` 标记慢测试 |
| `TSan: false positive` 与原子 | TSan 不完美理解 Rust 的原子顺序模型 | 添加 `TSAN_OPTIONS=suppressions=tsan.supp` 与特定抑制 |

### 自己动手

1. **触发 Miri UB 检测**：编写一个 `unsafe` 函数创建两个 `&mut` 引用到同一个 `i32`（别名违规）。运行 `cargo +nightly miri test` 并观察 "Stacked Borrows" 错误。用 `UnsafeCell` 或独立分配修复。

2. **在故意 bug 上运行 ASan**：创建一个测试进行 `unsafe` 越界数组访问。使用 `RUSTFLAGS="-Zsanitizer=address"` 构建并观察 ASan 的报告。注意它如何精确定位到确切的行。

3. **基准测试 Miri 开销**：对同一测试套件计时 `cargo test --lib` vs `cargo +nightly miri test --lib`。计算减速因子。基于此，决定在 CI 中哪些测试在 Miri 下运行，哪些用 `#[cfg_attr(miri, ignore)]` 跳过。

### 安全验证决策树

```mermaid
flowchart TD
    START["有 unsafe 代码？"] -->|无 | SAFE["Safe Rust — 无需<br/>验证"]
    START -->|有 | KIND{"什么类型？"}
    
    KIND -->|"纯 Rust unsafe"| MIRI["Miri<br/>MIR 解释器<br/>捕获别名、UB、泄漏"]
    KIND -->|"FFI / C 互操作"| VALGRIND["Valgrind memcheck<br/>或 ASan"]
    KIND -->|"并发 unsafe"| CONC{"无锁？"}
    
    CONC -->|"原子/无锁"| LOOM["loom<br/>原子模型检查器"]
    CONC -->|"Mutex/共享状态"| TSAN["TSan 或<br/>Miri -Zmiri-check-number-validity"]
    
    MIRI --> CI_MIRI["CI: cargo +nightly miri test"]
    VALGRIND --> CI_VALGRIND["CI: valgrind --leak-check=full"]
    
    style SAFE fill:#91e5a3,color:#000
    style MIRI fill:#e3f2fd,color:#000
    style VALGRIND fill:#ffd43b,color:#000
    style LOOM fill:#ff6b6b,color:#000
    style TSAN fill:#ffd43b,color:#000
```

### 🏋️ 练习

#### 🟡 练习 1：触发 Miri UB 检测

编写一个 `unsafe` 函数创建两个 `&mut` 引用到同一个 `i32`（别名违规）。运行 `cargo +nightly miri test` 并观察 Stacked Borrows 错误。修复它。

<details>
<summary>解决方案</summary>

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn aliasing_ub() {
        let mut x: i32 = 42;
        let ptr = &mut x as *mut i32;
        unsafe {
            // BUG：两个 &mut 引用到同一位置
            let _a = &mut *ptr;
            let _b = &mut *ptr; // Miri: Stacked Borrows 违规！
        }
    }
}
```

修复：使用独立分配或 `UnsafeCell`：

```rust
use std::cell::UnsafeCell;

#[test]
fn no_aliasing_ub() {
    let x = UnsafeCell::new(42);
    unsafe {
        let a = &mut *x.get();
        *a = 100;
    }
}
```
</details>

#### 🔴 练习 2：ASan 越界检测

创建一个带有 `unsafe` 越界数组访问的测试。在 nightly 上使用 `RUSTFLAGS="-Zsanitizer=address"` 构建并观察 ASan 的报告。

<details>
<summary>解决方案</summary>

```rust
#[test]
fn oob_access() {
    let arr = [1u8, 2, 3, 4, 5];
    let ptr = arr.as_ptr();
    unsafe {
        let _val = *ptr.add(10); // 越界！
    }
}
```

```bash
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -Zbuild-std \
  --target x86_64-unknown-linux-gnu -- oob_access
# ASan 报告：stack-buffer-overflow 在 <确切地址>
```
</details>

### 关键要点

- **Miri** 是纯 Rust `unsafe` 的工具——它捕获别名违规、释放后使用和泄漏，即使编译并通过测试
- **Valgrind** 是 FFI/C 互操作的工具——它在最终二进制文件上工作，无需重新编译
- **清理器**（ASan、TSan、MSan）需要 nightly 但以接近原生的速度运行——适合大型测试套件
- **`loom`** 专为验证无锁并发数据结构而设计
- 在 CI 中对每次推送运行 Miri；在夜间计划运行清理器以避免减慢主流水线

---
