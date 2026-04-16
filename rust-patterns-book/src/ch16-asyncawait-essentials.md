# 16. Async/Await Essentials 🔴

> **你将学到：**
> - Rust 的 `Future` trait 与 Go 的 goroutine 和 Python 的 asyncio 有何不同
> - Tokio 快速入门：生成任务、`join!` 和运行时配置
> - 常见的 async 陷阱以及如何修复
> - 何时使用 `spawn_blocking` 卸载阻塞工作

## Futures、Runtimes 和 `async fn`

Rust 的 async 模型与 Go 的 goroutine 或 Python 的 `asyncio` *根本不同*。
理解三个概念就足以开始：

1. **`Future` 是惰性状态机** —— 调用 `async fn` 不会执行任何操作；
   它返回一个必须被轮询（polled）的 `Future`。
2. **你需要一个运行时** 来轮询 futures —— `tokio`、`async-std` 或 `smol`。
   标准库定义了 `Future` 但不提供运行时。
3. **`async fn` 是语法糖** —— 编译器将其转换为实现 `Future` 的状态机。

```rust
// Future 只是一个 trait：
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// async fn 脱糖为：
// fn fetch_data(url: &str) -> impl Future<Output = Result<Vec<u8>, Error>>
async fn fetch_data(url: &str) -> Result<Vec<u8>, reqwest::Error> {
    let response = reqwest::get(url).await?;  // .await 挂起直到就绪
    let bytes = response.bytes().await?;
    Ok(bytes.to_vec())
}
```

### Tokio 快速入门

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust,ignore
use tokio::time::{sleep, Duration};
use tokio::task;

#[tokio::main]
async fn main() {
    // 生成并发任务（类似轻量级线程）：
    let handle_a = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "task A done"
    });

    let handle_b = task::spawn(async {
        sleep(Duration::from_millis(50)).await;
        "task B done"
    });

    // .await 两者 —— 它们并发运行，而非顺序：
    let (a, b) = tokio::join!(handle_a, handle_b);
    println!("{}, {}", a.unwrap(), b.unwrap());
}
```

### Async 常见陷阱

| 陷阱 | 发生原因 | 修复 |
|---------|---------------|-----|
| 阻塞 async | `std::thread::sleep` 或 CPU 工作阻塞执行器 | 使用 `tokio::task::spawn_blocking` 或 `rayon` |
| `Send` 边界错误 | Future 在 `.await` 期间持有 `!Send` 类型（如 `Rc`、`MutexGuard`） | 重构代码在 `.await` 之前 drop 非 Send 值 |
| Future 未被轮询 | 调用 `async fn` 但不 `.await` 或生成 —— 什么都不发生 | 总是 `.await` 或 `tokio::spawn` 返回的 future |
| 在 `.await` 间持有 `MutexGuard` | `std::sync::MutexGuard` 是 `!Send`；async 任务可能在不同线程恢复 | 使用 `tokio::sync::Mutex` 或在 `.await` 前 drop guard |
| 意外顺序执行 | `let a = foo().await; let b = bar().await;` 顺序运行 | 使用 `tokio::join!` 或 `tokio::spawn` 实现并发 |

```rust
// ❌ 阻塞 async 执行器：
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(5)); // 阻塞整个线程！
}

// ✅ 卸载阻塞工作：
async fn good() {
    tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(5)); // 在阻塞池上运行
    }).await.unwrap();
}
```

> **全面的 async 覆盖**：关于 `Stream`、`select!`、取消安全、
> 结构化并发和 `tower` 中间件，请参阅我们专门的
> **Async Rust 培训**指南。本节仅涵盖读写基本 async 代码的足够知识。

### 生成和结构化并发

Tokio 的 `spawn` 创建一个新的异步任务 —— 类似于 `thread::spawn` 但
更轻量：

```rust,ignore
use tokio::task;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 生成三个并发任务
    let h1 = task::spawn(async {
        sleep(Duration::from_millis(200)).await;
        "fetched user profile"
    });

    let h2 = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "fetched order history"
    });

    let h3 = task::spawn(async {
        sleep(Duration::from_millis(150)).await;
        "fetched recommendations"
    });

    // 等待所有三个并发（不是顺序！）
    let (r1, r2, r3) = tokio::join!(h1, h2, h3);
    println!("{}", r1.unwrap());
    println!("{}", r2.unwrap());
    println!("{}", r3.unwrap());
}
```

**`join!` vs `try_join!` vs `select!`**：

| 宏 | 行为 | 何时使用 |
|-------|----------|----------|
| `join!` | 等待所有 futures | 所有任务必须完成 |
| `try_join!` | 等待所有，遇到第一个 `Err` 时短路 | 任务返回 `Result` |
| `select!` | 当第一个 future 完成时返回 | 超时、取消 |

```rust,ignore
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    let result = timeout(Duration::from_secs(5), async {
        // 模拟慢网络调用
        tokio::time::sleep(Duration::from_millis(100)).await;
        Ok::<_, Box<dyn std::error::Error>>("data".to_string())
    }).await??; // 第一个 ? 解包 Elapsed，第二个 ? 解包内部 Result

    Ok(result)
}
```

### `Send` 边界以及为什么 Futures 必须是 `Send`

当你 `tokio::spawn` 一个 future 时，它可能在不同的 OS 线程上恢复。
这意味着 future 必须是 `Send`。常见陷阱：

```rust,ignore
use std::rc::Rc;

async fn not_send() {
    let rc = Rc::new(42); // Rc 是 !Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", rc); // rc 在 .await 间持有 —— future 是 !Send
}

// 修复 1：在 .await 前 drop
async fn fixed_drop() {
    let data = {
        let rc = Rc::new(42);
        *rc // 复制值出来
    }; // rc 在这里 drop
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", data); // 只是 i32，它是 Send
}

// 修复 2：用 Arc 替代 Rc
async fn fixed_arc() {
    let arc = std::sync::Arc::new(42); // Arc 是 Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", arc); // ✅ Future 是 Send
}
```

> **全面的 async 覆盖**：关于 `Stream`、`select!`、取消安全、
> 结构化并发和 `tower` 中间件，请参阅我们专门的
> **Async Rust 培训**指南。本节仅涵盖读写基本 async 代码的足够知识。

> **另见：** [第 5 章 — 通道](ch05-channels-and-message-passing.md) 了解同步通道。[第 6 章 — 并发](ch06-concurrency-vs-parallelism-vs-threads.md) 了解 OS 线程 vs async 任务。

> **关键要点 —— Async**
> - `async fn` 返回惰性 `Future` —— 直到你 `.await` 或生成它之前都不会运行
> - 对 CPU 密集型或阻塞工作使用 `tokio::task::spawn_blocking`
> - 不要在 `.await` 间持有 `std::sync::MutexGuard` —— 改用 `tokio::sync::Mutex`
> - Futures 生成时必须是 `Send` —— 在 `.await` 点之前 drop `!Send` 类型

---

### 练习：带超时的并发抓取器 ★★（约 25 分钟）

编写一个 async 函数 `fetch_all`，生成三个 `tokio::spawn` 任务，每个
模拟使用 `tokio::time::sleep` 的网络调用。使用
`tokio::try_join!` 连接所有三个，并用 `tokio::time::timeout(Duration::from_secs(5), ...)` 包装。
返回 `Result<Vec<String>, ...>` 或如果任何任务失败或截止日期过期则返回错误。

<details>
<summary>🔑 解答</summary>

```rust,ignore
use tokio::time::{sleep, timeout, Duration};

async fn fake_fetch(name: &'static str, delay_ms: u64) -> Result<String, String> {
    sleep(Duration::from_millis(delay_ms)).await;
    Ok(format!("{name}: OK"))
}

async fn fetch_all() -> Result<Vec<String>, Box<dyn std::error::Error>> {
    let deadline = Duration::from_secs(5);

    let (a, b, c) = timeout(deadline, async {
        let h1 = tokio::spawn(fake_fetch("svc-a", 100));
        let h2 = tokio::spawn(fake_fetch("svc-b", 200));
        let h3 = tokio::spawn(fake_fetch("svc-c", 150));
        tokio::try_join!(h1, h2, h3)
    })
    .await??;

    Ok(vec![a?, b?, c?])
}

#[tokio::main]
async fn main() {
    let results = fetch_all().await.unwrap();
    for r in &results {
        println!("{r}");
    }
}
```

</details>

***
