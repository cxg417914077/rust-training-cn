# 12. Common Pitfalls 🔴

> **你将学到：**
> - 9 个常见 async Rust bug 及如何修复
> - 为什么阻塞 executor 是头号错误（以及 `spawn_blocking` 如何修复）
> - 取消风险：当 future 在 await 中途被 drop 时会发生什么
> - 调试：`tokio-console`、`tracing`、`#[instrument]`
> - 测试：`#[tokio::test]`、`time::pause()`、基于 trait 的 mock

## 阻塞 Executor

async Rust 中的头号错误：在 async executor 线程上运行阻塞代码。这会饿死其他任务。

```rust
// ❌ 错误：阻塞整个 executor 线程
async fn bad_handler() -> String {
    let data = std::fs::read_to_string("big_file.txt").unwrap(); // 阻塞！
    process(&data)
}

// ✅ 正确：将阻塞工作卸载到专用的线程池
async fn good_handler() -> String {
    let data = tokio::task::spawn_blocking(|| {
        std::fs::read_to_string("big_file.txt").unwrap()
    }).await.unwrap();
    process(&data)
}

// ✅ 也正确：使用 tokio 的 async fs
async fn also_good_handler() -> String {
    let data = tokio::fs::read_to_string("big_file.txt").await.unwrap();
    process(&data)
}
```

```mermaid
graph TB
    subgraph "❌ Blocking Call on Executor"
        T1_BAD["Thread 1: std::fs::read()<br/>🔴 BLOCKED for 500ms"]
        T2_BAD["Thread 2: handling requests<br/>🟢 Working alone"]
        TASKS_BAD["100 pending tasks<br/>⏳ Starved"]
        T1_BAD -->|"can't poll"| TASKS_BAD
    end

    subgraph "✅ spawn_blocking"
        T1_GOOD["Thread 1: polling futures<br/>🟢 Available"]
        T2_GOOD["Thread 2: polling futures<br/>🟢 Available"]
        BT["Blocking pool thread:<br/>std::fs::read()<br/>🔵 Separate pool"]
        TASKS_GOOD["100 tasks<br/>✅ All making progress"]
        T1_GOOD -->|"polls"| TASKS_GOOD
        T2_GOOD -->|"polls"| TASKS_GOOD
    end
```

### std::thread::sleep vs tokio::time::sleep

```rust
// ❌ 错误：阻塞 executor 线程 5 秒
async fn bad_delay() {
    std::thread::sleep(Duration::from_secs(5)); // 线程无法 poll 其他事情！
}

// ✅ 正确：yield 给 executor，其他任务可以运行
async fn good_delay() {
    tokio::time::sleep(Duration::from_secs(5)).await; // 非阻塞！
}
```

### 在 .await 之间持有 MutexGuard

```rust
use std::sync::Mutex; // std Mutex —— 不是 async 感知

// ⚠️ 风险：MutexGuard 跨 .await 持有
async fn bad_mutex(data: &Mutex<Vec<String>>) {
    let mut guard = data.lock().unwrap();
    guard.push("item".into());
    some_io().await; // Guard 在这里持有 —— 阻塞其他线程锁定！
    guard.push("another".into());
}
// 注意：这能编译！std::sync::MutexGuard 是 !Send，但编译器只在
// 你将 Future 传递给需要它的东西时（例如 tokio::spawn）才强制执行 Send。
// 直接调用 bad_mutex(...).await 编译没问题。
```

**为什么这通常是个问题** —— 但并不总是：

跨 `.await` 持有 `std::sync::Mutex` 会阻塞 **OS 线程** 整个 I/O 过程，
阻止 executor 在该线程上 poll 其他任务。对于短临界区这是浪费，
对于长 I/O 这是性能陷阱。

**然而**，有些合法情况你*必须*跨 `.await` 持有锁 —— 就像数据库
事务在读取和提交之间持有锁一样。drop 和重新获取锁引入了
**TOCTOU (time-of-check to time-of-use 竞态)**：另一个任务可以在
你的两个临界区之间修改数据。正确的修复取决于用例：

```rust
// 方案 1：限定 guard 作用域 —— 适用于操作独立时
async fn scoped_mutex(data: &Mutex<Vec<String>>) {
    {
        let mut guard = data.lock().unwrap();
        guard.push("item".into());
    } // Guard 在这里 drop
    some_io().await; // 锁释放 —— 其他任务可以进行
    {
        let mut guard = data.lock().unwrap();
        guard.push("another".into());
    }
}
// ⚠️ 注意：另一个任务可以在两次 push 之间锁定 + 修改 Vec。
//    如果两次 push 是独立的这没问题，但如果 "another" 依赖于
//    "item" 设置的状态就错了。

// 方案 2：使用 tokio::sync::Mutex —— 跨 .await 持有锁而不阻塞 OS 线程。
//           当你需要在 await 点之间进行事务性读 - 改 - 写时最好。
use tokio::sync::Mutex as AsyncMutex;

async fn async_mutex(data: &AsyncMutex<Vec<String>>) {
    let mut guard = data.lock().await; // 异步锁 —— 不阻塞线程
    guard.push("item".into());
    some_io().await; // OK —— tokio Mutex guard 是 Send
    guard.push("another".into());
    // Guard 整个过程持有 —— 没有 TOCTOU 竞态，没有线程阻塞。
}
```

> **何时使用哪种 Mutex**：
> - `std::sync::Mutex`：短临界区，内部没有 `.await`
> - `tokio::sync::Mutex`：当你需要跨 `.await` 点持有锁时（事务语义，TOCTOU 避免）
> - `parking_lot::Mutex`：`std` 的直接替代品，更快、更小，仍然不能 `.await`
>
> **经验法则**：不要盲目地在 `.await` 周围分割临界区。
> 问自己两半是否真正独立。如果不是 —— 如果第二半依赖于
> 第一半的状态 —— 使用 `tokio::sync::Mutex` 或重新设计数据流。

### 取消风险

Drop future 会取消它 —— 但这可能让事物处于不一致状态：

```rust
// ❌ 危险：取消时资源泄漏
async fn transfer(from: &Account, to: &Account, amount: u64) {
    from.debit(amount).await;  // 如果在这里取消...
    to.credit(amount).await;   // ...钱消失了！
}

// ✅ 安全：让操作原子化或使用补偿
async fn safe_transfer(from: &Account, to: &Account, amount: u64) -> Result<(), Error> {
    // 使用数据库事务（全有或全无）
    let tx = db.begin_transaction().await?;
    tx.debit(from, amount).await?;
    tx.credit(to, amount).await?;
    tx.commit().await?; // 只有所有事情成功才提交
    Ok(())
}

// ✅ 也安全：使用 tokio::select! 与取消感知
tokio::select! {
    result = transfer(from, to, amount) => {
        // 转账完成
    }
    _ = shutdown_signal() => {
        // 不要在转账中途取消 —— 让它完成
        // 或者：显式回滚
    }
}
```

### 没有 Async Drop

Rust 的 `Drop` trait 是同步的 —— 你**不能**在 `drop()` 内部 `.await`。这是一个常见的混淆源：

```rust
struct DbConnection { /* ... */ }

impl Drop for DbConnection {
    fn drop(&mut self) {
        // ❌ 不能这样做 —— drop() 是同步的！
        // self.connection.shutdown().await;

        // ✅ 变通方案 1：生成清理任务（即发即弃）
        let conn = self.connection.take();
        tokio::spawn(async move {
            let _ = conn.shutdown().await;
        });

        // ✅ 变通方案 2：使用同步关闭
        // self.connection.blocking_close();
    }
}
```

**最佳实践**：提供一个显式的 `async fn close(self)` 方法并文档化调用者应该使用它。仅将 `Drop` 作为安全网，而不是主要的清理路径。

### select! 公平性和饥饿

```rust
use tokio::sync::mpsc;

// ❌ 不公平：busy_stream 总是赢，slow_stream 饥饿
async fn unfair(mut fast: mpsc::Receiver<i32>, mut slow: mpsc::Receiver<i32>) {
    loop {
        tokio::select! {
            Some(v) = fast.recv() => println!("fast: {v}"),
            Some(v) = slow.recv() => println!("slow: {v}"),
            // 如果两者都就绪，tokio 随机选择一个。
            // 但如果 `fast` 总是就绪，`slow` 很少被 poll。
        }
    }
}

// ✅ 公平：使用 biased select 或批量 drain
async fn fair(mut fast: mpsc::Receiver<i32>, mut slow: mpsc::Receiver<i32>) {
    loop {
        tokio::select! {
            biased; // 总是按顺序检查 —— 显式优先级

            Some(v) = slow.recv() => println!("slow: {v}"),  // 优先级！
            Some(v) = fast.recv() => println!("fast: {v}"),
        }
    }
}
```

### 意外的顺序执行

```rust
// ❌ 顺序执行：总共花费 2 秒
async fn slow() {
    let a = fetch("url_a").await; // 1 秒
    let b = fetch("url_b").await; // 1 秒（等 a 先完成！）
}

// ✅ 并发执行：总共花费 1 秒
async fn fast() {
    let (a, b) = tokio::join!(
        fetch("url_a"), // 两者立即启动
        fetch("url_b"),
    );
}

// ✅ 也并发：使用 let + join
async fn also_fast() {
    let fut_a = fetch("url_a"); // 创建 future（惰性 —— 还未启动）
    let fut_b = fetch("url_b"); // 创建 future
    let (a, b) = tokio::join!(fut_a, fut_b); // 现在两者并发运行
}
```

> **陷阱**：`let a = fetch(url).await; let b = fetch(url).await;` 是顺序的！
> 第二个 `.await` 要等到第一个完成才开始。使用 `join!` 或 `spawn` 实现并发。

## 案例研究：调试挂起的生产服务

一个真实场景：服务处理请求正常 10 分钟，然后停止响应。日志中没有错误。CPU 为 0%。

**诊断步骤：**

1. **附加 `tokio-console`** —— 显示 200+ 任务卡在 `Pending` 状态
2. **检查任务详情** —— 都在等待同一个 `Mutex::lock().await`
3. **根本原因** —— 一个任务跨 `.await` 持有 `std::sync::MutexGuard` 并 panic，毒化了 mutex。所有其他任务现在在 `lock().unwrap()` 失败

**修复：**

| 之前（损坏） | 之后（修复） |
|-------------|-------------|
| `std::sync::Mutex` | `tokio::sync::Mutex` |
| 跨 `.await` 使用 `.lock().unwrap()` | 在 `.await` 前限定锁的作用域 |
| 锁获取没有超时 | `tokio::time::timeout(dur, mutex.lock())` |
| 没有从毒化 mutex 恢复 | `tokio::sync::Mutex` 不会毒化 |

**预防清单：**
- [ ] 如果 guard 跨任何 `.await`，使用 `tokio::sync::Mutex`
- [ ] 为 async 函数添加 `#[tracing::instrument]` 用于 span 跟踪
- [ ] 在 staging 中运行 `tokio-console` 以早期捕获挂起任务
- [ ] 添加健康检查端点验证任务响应能力

<details>
<summary><strong>🏋️ 练习：找出 bug</strong>（点击展开）</summary>

**挑战**：找出这段代码中所有的 async 陷阱并修复它们。

```rust
use std::sync::Mutex;

async fn process_requests(urls: Vec<String>) -> Vec<String> {
    let results = Mutex::new(Vec::new());
    
    for url in &urls {
        let response = reqwest::get(url).await.unwrap().text().await.unwrap();
        std::thread::sleep(std::time::Duration::from_millis(100)); // 限流
        let mut guard = results.lock().unwrap();
        guard.push(response);
        expensive_parse(&guard).await; // 解析目前为止的所有结果
    }
    
    results.into_inner().unwrap()
}
```

<details>
<summary>🔑 解答</summary>

**发现的 bug：**

1. **顺序获取** —— URLs 一个一个获取而不是并发
2. **`std::thread::sleep`** —— 阻塞 executor 线程
3. **MutexGuard 跨 `.await` 持有** —— `expensive_parse` 被 await 时 `guard` 还活着
4. **没有并发** —— 应该使用 `join!` 或 `FuturesUnordered`

```rust
use tokio::sync::Mutex;
use std::sync::Arc;
use futures::stream::{self, StreamExt};

async fn process_requests(urls: Vec<String>) -> Vec<String> {
    // 修复 4：使用 buffer_unordered 并发处理 URLs
    let results: Vec<String> = stream::iter(urls)
        .map(|url| async move {
            let response = reqwest::get(&url).await.unwrap().text().await.unwrap();
            // 修复 2：使用 tokio::time::sleep 而不是 std::thread::sleep
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            response
        })
        .buffer_unordered(10) // 最多 10 个并发请求
        .collect()
        .await;

    // 修复 3：收集后解析 —— 根本不需要 mutex！
    for result in &results {
        expensive_parse(result).await;
    }

    results
}
```

**关键要点**：通常你可以重构 async 代码以完全消除 mutex。使用 streams/join 收集结果，然后处理。更简单、更快、没有死锁风险。

</details>
</details>

---

### 调试 Async 代码

Async 堆栈跟踪出了名的难懂 —— 它们显示 executor 的 poll 循环而不是你的逻辑调用链。这些是必备的调试工具。

#### tokio-console：实时任务检查器

[tokio-console](https://github.com/tokio-rs/console) 给你一個 `htop` 风格的视图，显示每个派生任务：它的状态、poll 持续时间、waker 活动和资源使用。

```toml
# Cargo.toml
[dependencies]
console-subscriber = "0.4"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
#[tokio::main]
async fn main() {
    console_subscriber::init(); // 替代默认 tracing subscriber
    // ... 应用的其余部分
}
```

然后在另一个终端：

```bash
$ RUSTFLAGS="--cfg tokio_unstable" cargo run   # 需要的编译时标志
$ tokio-console                                # 连接到 127.0.0.1:6669
```

#### tracing + #[instrument]：Async 的结构化日志

[`tracing`](https://docs.rs/tracing) crate 理解 `Future` 生命周期。Spans 跨 `.await` 点保持打开，即使 OS 线程已经移开也能给你逻辑调用栈：

```rust
use tracing::{info, instrument};

#[instrument(skip(db_pool), fields(user_id = %user_id))]
async fn handle_request(user_id: u64, db_pool: &Pool) -> Result<Response> {
    info!("looking up user");
    let user = db_pool.get_user(user_id).await?;  // span 跨 .await 保持打开
    info!(email = %user.email, "found user");
    let orders = fetch_orders(user_id).await?;     // 仍然是同一个 span
    Ok(build_response(user, orders))
}
```

输出（使用 `tracing_subscriber::fmt::json()`）：

```json
{"timestamp":"...","level":"INFO","span":{"name":"handle_request","user_id":"42"},"message":"looking up user"}
{"timestamp":"...","level":"INFO","span":{"name":"handle_request","user_id":"42"},"fields":{"email":"a@b.com"},"message":"found user"}
```

#### 调试清单

| 症状 | 可能原因 | 工具 |
|------|---------|------|
| 任务永远挂起 | 缺少 `.await` 或死锁的 `Mutex` | `tokio-console` 任务视图 |
| 低吞吐量 | async 线程上的阻塞调用 | `tokio-console` poll 时间直方图 |
| `Future is not Send` | 非 Send 类型跨 `.await` 持有 | 编译器错误 + `#[instrument]` 定位 |
| 神秘取消 | 父 `select!` drop 了一个分支 | `tracing` span 生命周期事件 |

> **提示**：启用 `RUSTFLAGS="--cfg tokio_unstable"` 以在 tokio-console 中获取任务级指标。这是编译时标志，不是运行时标志。

### 测试 Async 代码

Async 代码引入独特的测试挑战 —— 你需要 runtime、时间控制以及测试并发行为的策略。

使用 `#[tokio::test]` 的**基本 async 测试**：

```rust
// Cargo.toml
// [dev-dependencies]
// tokio = { version = "1", features = ["full", "test-util"] }

#[tokio::test]
async fn test_basic_async() {
    let result = fetch_data().await;
    assert_eq!(result, "expected");
}

// 单线程测试（用于 !Send 类型）：
#[tokio::test(flavor = "current_thread")]
async fn test_single_threaded() {
    let rc = std::rc::Rc::new(42);
    let val = async { *rc }.await;
    assert_eq!(val, 42);
}

// 多线程，显式指定 worker 数量：
#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
async fn test_concurrent_behavior() {
    // 用真实并发测试竞态条件
    let counter = std::sync::Arc::new(std::sync::atomic::AtomicU32::new(0));
    let c1 = counter.clone();
    let c2 = counter.clone();
    let (a, b) = tokio::join!(
        tokio::spawn(async move { c1.fetch_add(1, std::sync::atomic::Ordering::SeqCst) }),
        tokio::spawn(async move { c2.fetch_add(1, std::sync::atomic::Ordering::SeqCst) }),
    );
    a.unwrap();
    b.unwrap();
    assert_eq!(counter.load(std::sync::atomic::Ordering::SeqCst), 2);
}
```

**时间操作** —— 测试超时而不实际等待：

```rust
use tokio::time::{self, Duration, Instant};

#[tokio::test]
async fn test_timeout_behavior() {
    // 暂停时间 —— sleep() 立即推进，没有真实的挂钟延迟
    time::pause();

    let start = Instant::now();
    time::sleep(Duration::from_secs(3600)).await; // "等待" 1 小时 —— 实际 0ms
    assert!(start.elapsed() >= Duration::from_secs(3600));
    // 测试在几毫秒内运行，而不是一小时！
}

#[tokio::test]
async fn test_retry_timing() {
    time::pause();

    // 测试重试逻辑等待预期的持续时间
    let start = Instant::now();
    let result = retry_with_backoff(|| async {
        Err::<(), _>("simulated failure")
    }, 3, Duration::from_secs(1))
    .await;

    assert!(result.is_err());
    // 1s + 2s + 4s = 7s 退避（指数）
    assert!(start.elapsed() >= Duration::from_secs(7));
}

#[tokio::test]
async fn test_deadline_exceeded() {
    time::pause();

    let result = tokio::time::timeout(
        Duration::from_secs(5),
        async {
            // 模拟慢操作
            time::sleep(Duration::from_secs(10)).await;
            "done"
        }
    ).await;

    assert!(result.is_err()); // 超时
}
```

**Mock async 依赖** —— 使用 trait 对象或泛型：

```rust
// 为依赖定义 trait：
trait Storage {
    async fn get(&self, key: &str) -> Option<String>;
    async fn set(&self, key: &str, value: String);
}

// 生产实现：
struct RedisStorage { /* ... */ }
impl Storage for RedisStorage {
    async fn get(&self, key: &str) -> Option<String> {
        // 真实的 Redis 调用
        todo!()
    }
    async fn set(&self, key: &str, value: String) {
        todo!()
    }
}

// 测试 mock：
struct MockStorage {
    data: std::sync::Mutex<std::collections::HashMap<String, String>>,
}

impl MockStorage {
    fn new() -> Self {
        MockStorage { data: std::sync::Mutex::new(std::collections::HashMap::new()) }
    }
}

impl Storage for MockStorage {
    async fn get(&self, key: &str) -> Option<String> {
        self.data.lock().unwrap().get(key).cloned()
    }
    async fn set(&self, key: &str, value: String) {
        self.data.lock().unwrap().insert(key.to_string(), value);
    }
}

// 被测试函数对 Storage 泛型：
async fn cache_lookup<S: Storage>(store: &S, key: &str) -> String {
    match store.get(key).await {
        Some(val) => val,
        None => {
            let val = "computed".to_string();
            store.set(key, val.clone()).await;
            val
        }
    }
}

#[tokio::test]
async fn test_cache_miss_then_hit() {
    let mock = MockStorage::new();

    // 第一次调用：miss → 计算并存储
    let val = cache_lookup(&mock, "key1").await;
    assert_eq!(val, "computed");

    // 第二次调用：hit → 返回存储的值
    let val = cache_lookup(&mock, "key1").await;
    assert_eq!(val, "computed");
    assert!(mock.data.lock().unwrap().contains_key("key1"));
}
```

**测试 channel 和任务通信**：

```rust
#[tokio::test]
async fn test_producer_consumer() {
    let (tx, mut rx) = tokio::sync::mpsc::channel(10);

    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
        // tx 在这里 drop —— channel 关闭
    });

    let mut received = Vec::new();
    while let Some(val) = rx.recv().await {
        received.push(val);
    }

    assert_eq!(received, vec![0, 1, 2, 3, 4]);
}
```

| 测试模式 | 何时使用 | 关键工具 |
|---------|---------|---------|
| `#[tokio::test]` | 所有 async 测试 | `tokio = { features = ["macros", "rt"] }` |
| `time::pause()` | 测试超时、重试、周期性任务 | `tokio::time::pause()` |
| Trait mocking | 测试没有 I/O 的业务逻辑 | 泛型 `<S: Storage>` |
| `current_thread` flavor | 测试 `!Send` 类型或确定性调度 | `#[tokio::test(flavor = "current_thread")]` |
| `multi_thread` flavor | 测试竞态条件 | `#[tokio::test(flavor = "multi_thread")]` |

> **关键要点——Common Pitfalls**
> - 永远不要阻塞 executor —— 对 CPU/同步工作使用 `spawn_blocking`
> - 永远不要跨 `.await` 持有 `MutexGuard` —— 严格限定锁的作用域或使用 `tokio::sync::Mutex`
> - 取消立即 drop future —— 对部分操作使用"cancel-safe"模式
> - 使用 `tokio-console` 和 `#[tracing::instrument]` 调试 async 代码
> - 使用 `#[tokio::test]` 和 `time::pause()` 测试 async 代码以实现确定性计时

> **另见：** [第 8 章 — Tokio Deep Dive](ch08-tokio-deep-dive.md) 了解同步原语，[第 13 章 — 生产级模式](ch13-production-patterns.md) 了解优雅关闭和结构化并发

***
