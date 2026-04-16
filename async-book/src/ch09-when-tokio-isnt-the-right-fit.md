# 9. When Tokio Isn't the Right Fit 🟡

> **你将学到：**
> - `'static` 问题：何时 `tokio::spawn` 迫使你到处使用 `Arc`
> - `LocalSet` 用于 `!Send` futures
> - `FuturesUnordered` 用于借用友好的并发（不需要 spawn）
> - `JoinSet` 用于管理的任务组
> - 编写 runtime 无关的库

```mermaid
graph TD
    START["Need concurrent futures?"] --> STATIC{"Can futures be 'static?"}
    STATIC -->|Yes| SEND{"Are futures Send?"}
    STATIC -->|No| FU["FuturesUnordered<br/>Runs on current task"]
    SEND -->|Yes| SPAWN["tokio::spawn<br/>Multi-threaded"]
    SEND -->|No| LOCAL["LocalSet<br/>Single-threaded"]
    SPAWN --> MANAGE{"Need to track/abort tasks?"}
    MANAGE -->|Yes| JOINSET["JoinSet / TaskTracker"]
    MANAGE -->|No| HANDLE["JoinHandle"]

    style START fill:#f5f5f5,stroke:#333,color:#000
    style FU fill:#d4efdf,stroke:#27ae60,color:#000
    style SPAWN fill:#e8f4f8,stroke:#2980b9,color:#000
    style LOCAL fill:#fef9e7,stroke:#f39c12,color:#000
    style JOINSET fill:#e8daef,stroke:#8e44ad,color:#000
    style HANDLE fill:#e8f4f8,stroke:#2980b9,color:#000
```

## 'static Future 问题

Tokio 的 `spawn` 需要 `'static` futures。这意味着你不能在派生的任务中借用本地数据：

```rust
async fn process_items(items: &[String]) {
    // ❌ 不能这样做 —— items 是借用，不是 'static
    // for item in items {
    //     tokio::spawn(async {
    //         process(item).await;
    //     });
    // }

    // 😐 变通方案 1：克隆所有数据
    for item in items {
        let item = item.clone();
        tokio::spawn(async move {
            process(&item).await;
        });
    }

    // 😐 变通方案 2：使用 Arc
    let items = Arc::new(items.to_vec());
    for i in 0..items.len() {
        let items = Arc::clone(&items);
        tokio::spawn(async move {
            process(&items[i]).await;
        });
    }
}
```

这很烦人！在 Go 中，你可以直接 `go func() { use(item) }` 使用闭包。在 Rust 中，所有权系统迫使你思考谁拥有什么以及它存活多久。

### `tokio::spawn` 的替代方案

不是每个问题都需要 `spawn`。这里有三个工具，每个解决*不同的*约束：

```rust
// 1. FuturesUnordered —— 完全避免 'static（不需要 spawn！）
use futures::stream::{FuturesUnordered, StreamExt};

async fn process_items(items: &[String]) {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            // ✅ 可以借用 item —— 不需要 spawn，不需要 'static！
            process(item).await
        })
        .collect();

    // 驱动所有 futures 完成
    futures.for_each(|result| async move {
        println!("Result: {result:?}");
    }).await;
}

// 2. tokio::task::LocalSet —— 在当前线程运行 !Send futures
//    ⚠️  仍然需要 'static —— 解决 Send 问题，不是 'static
use tokio::task::LocalSet;

let local_set = LocalSet::new();
local_set.run_until(async {
    tokio::task::spawn_local(async {
        // 可以在这里使用 Rc、Cell 和其他 !Send 类型
        let rc = std::rc::Rc::new(42);
        println!("{rc}");
    }).await.unwrap();
}).await;

// 3. tokio JoinSet (tokio 1.21+) —— 管理派生任务集
//    ⚠️  仍然需要 'static + Send —— 解决任务*管理*问题，
//    不是 'static 问题。适合跟踪、中止和加入动态任务组。
use tokio::task::JoinSet;

async fn with_joinset() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        // i 是 Copy 并被移入闭包 —— 已经是 'static。
        // 对于借用数据，你仍然需要 Arc 或 clone。
        set.spawn(async move {
            tokio::time::sleep(Duration::from_millis(100)).await;
            i * 2
        });
    }

    while let Some(result) = set.join_next().await {
        println!("Task completed: {:?}", result.unwrap());
    }
}
```

> **哪个工具解决哪个问题？**
>
> | 你遇到的约束 | 工具 | 避免 `'static`？ | 避免 `Send`？ |
> |---|---|---|---|
> | 无法让 futures 成为 `'static` | `FuturesUnordered` | ✅ 是 | ✅ 是 |
> | Futures 是 `'static` 但 `!Send` | `LocalSet` | ❌ 否 | ✅ 是 |
> | 需要跟踪/中止派生的任务 | `JoinSet` | ❌ 否 | ❌ 否 |

### 库的轻量级 Runtimes

如果你在编写库 —— 不要强迫用户使用 tokio：

```rust
// ❌ 坏：库强迫用户使用 tokio
pub async fn my_lib_function() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    // 现在你的用户*必须*使用 tokio
}

// ✅ 好：库是 runtime 无关的
pub async fn my_lib_function() {
    // 只使用来自 std::future 和 futures crate 的类型
    do_computation().await;
}

// ✅ 好：为 I/O 操作接受泛型 future
pub async fn fetch_with_retry<F, Fut, T, E>(
    operation: F,
    max_retries: usize,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    for attempt in 0..max_retries {
        match operation().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt == max_retries - 1 => return Err(e),
            Err(_) => continue,
        }
    }
    unreachable!()
}
```

> **经验法则**：库应该依赖 `futures` crate，而不是 `tokio`。
> 应用应该依赖 `tokio`（或它们选择的 runtime）。
> 这保持了生态系统的可组合性。

<details>
<summary><strong>🏋️ 练习：FuturesUnordered vs Spawn</strong>（点击展开）</summary>

**挑战**：用两种方式编写相同的函数 —— 一种使用 `tokio::spawn`（需要 `'static`），另一种使用 `FuturesUnordered`（借用数据）。函数接收 `&[String]` 并返回每个字符串在模拟异步查找后的长度。

比较：哪种方式需要 `.clone()`？哪种可以借用输入切片？

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::time::{sleep, Duration};

// Version 1: tokio::spawn — requires 'static, must clone
async fn lengths_with_spawn(items: &[String]) -> Vec<usize> {
    let mut handles = Vec::new();
    for item in items {
        let owned = item.clone(); // Must clone — spawn requires 'static
        handles.push(tokio::spawn(async move {
            sleep(Duration::from_millis(10)).await;
            owned.len()
        }));
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// Version 2: FuturesUnordered — borrows data, no clone needed
async fn lengths_without_spawn(items: &[String]) -> Vec<usize> {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            sleep(Duration::from_millis(10)).await;
            item.len() // ✅ Borrows item — no clone!
        })
        .collect();

    futures.collect().await
}

#[tokio::test]
async fn test_both_versions() {
    let items = vec!["hello".into(), "world".into(), "rust".into()];

    let v1 = lengths_with_spawn(&items).await;
    // Note: v1 preserves insertion order (sequential join)

    let mut v2 = lengths_without_spawn(&items).await;
    v2.sort(); // FuturesUnordered returns in completion order

    assert_eq!(v1, vec![5, 5, 4]);
    assert_eq!(v2, vec![4, 5, 5]);
}
```

**关键要点**：`FuturesUnordered` 通过在当前位置上运行所有 futures 来避免 `'static` 要求（无线程迁移）。权衡：所有 futures 共享一个任务 —— 如果一个阻塞，其他也会停滞。对使用单独线程运行 CPU 密集型工作使用 `spawn`。

</details>
</details>

> **关键要点——When Tokio Isn't the Right Fit**
> - `FuturesUnordered` 在当前位置上并发运行 futures —— 没有 `'static` 要求
> - `LocalSet` 在单线程 executor 上启用 `!Send` futures
> - `JoinSet` (tokio 1.21+) 提供管理的任务组，带有自动清理
> - 对于库：只依赖 `std::future::Future` + `futures` crate，不直接依赖 tokio

> **另见：** [第 8 章 — Tokio Deep Dive](ch08-tokio-deep-dive.md) 了解何时 spawn 是正确的工具，[第 11 章 — Streams](ch11-streams-and-asynciterator.md) 了解 `buffer_unordered()` 作为另一种并发限制器

***
