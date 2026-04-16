# 8. Tokio Deep Dive 🟡

> **你将学到：**
> - Runtime 配置：多线程 vs 当前线程以及何时使用
> - `tokio::spawn`、`'static` 要求和 `JoinHandle`
> - 任务取消语义（drop 时取消）
> - 同步原语：Mutex、RwLock、Semaphore 和四种 channel 类型

## Runtime 配置：多线程 vs 当前线程

Tokio 提供两种 runtime 配置：

```rust
// 多线程（默认，使用 #[tokio::main]）
// 使用工作窃取线程池 —— 任务可以在线程间移动
#[tokio::main]
async fn main() {
    // N 个工作线程（默认 = CPU 核心数）
    // 任务需要 Send + 'static
}

// 当前线程 —— 所有事情在一个线程上运行
#[tokio::main(flavor = "current_thread")]
async fn main() {
    // 单线程 —— 任务不需要 Send
    // 更轻量，适合简单工具或 WASM
}

// 手动构建 runtime：
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()
    .unwrap();

rt.block_on(async {
    println!("Running on custom runtime");
});
```

```mermaid
graph TB
    subgraph "Multi-Thread (default)"
        MT_Q1["Thread 1<br/>Task A, Task D"]
        MT_Q2["Thread 2<br/>Task B"]
        MT_Q3["Thread 3<br/>Task C, Task E"]
        STEAL["Work Stealing:<br/>idle threads steal from busy ones"]
        MT_Q1 <--> STEAL
        MT_Q2 <--> STEAL
        MT_Q3 <--> STEAL
    end

    subgraph "Current-Thread"
        ST_Q["Single Thread<br/>Task A → Task B → Task C → Task D"]
    end

    style MT_Q1 fill:#c8e6c9,color:#000
    style MT_Q2 fill:#c8e6c9,color:#000
    style MT_Q3 fill:#c8e6c9,color:#000
    style ST_Q fill:#bbdefb,color:#000
```

### tokio::spawn 和 'static 要求

`tokio::spawn` 将 future 放入 runtime 的任务队列。因为它可能在*任何*工作线程上、在*任何*时间运行，future 必须是 `Send + 'static`：

```rust
use tokio::task;

async fn example() {
    let data = String::from("hello");

    // ✅ 可行：将所有权移入任务
    let handle = task::spawn(async move {
        println!("{data}");
        data.len()
    });

    let len = handle.await.unwrap();
    println!("Length: {len}");
}

async fn problem() {
    let data = String::from("hello");

    // ❌ 失败：data 是借用，不是 'static
    // task::spawn(async {
    //     println!("{data}"); // 借用 `data` —— 不是 'static
    // });

    // ❌ 失败：Rc 不是 Send
    // let rc = std::rc::Rc::new(42);
    // task::spawn(async move {
    //     println!("{rc}"); // Rc 是 !Send —— 不能跨线程边界
    // });
}
```

**为什么需要 `'static`？** 派生的任务独立运行 —— 它可能比创建它的作用域存活得更久。编译器无法证明引用会保持有效，所以要求拥有数据的所有权。

**为什么需要 `Send`？** 任务可能在不同线程上恢复执行，而不是它挂起时所在的线程。所有在 `.await` 点之间持有的数据必须安全地在线程间发送。

```rust
// 常见模式：克隆共享数据到任务中
let shared = Arc::new(config);

for i in 0..10 {
    let shared = Arc::clone(&shared); // 克隆 Arc，不是数据本身
    tokio::spawn(async move {
        process_item(i, &shared).await;
    });
}
```

### JoinHandle 和任务取消

```rust
use tokio::task::JoinHandle;
use tokio::time::{sleep, Duration};

async fn cancellation_example() {
    let handle: JoinHandle<String> = tokio::spawn(async {
        sleep(Duration::from_secs(10)).await;
        "completed".to_string()
    });

    // 通过 drop handle 来取消任务？不 —— 任务继续运行！
    // drop(handle); // 任务在后台继续运行

    // 要真正取消，调用 abort()：
    handle.abort();

    // 等待被取消的任务返回 JoinError
    match handle.await {
        Ok(val) => println!("Got: {val}"),
        Err(e) if e.is_cancelled() => println!("Task was cancelled"),
        Err(e) => println!("Task panicked: {e}"),
    }
}
```

> **重要**：Drop `JoinHandle` **不会**取消 tokio 中的任务。
> 任务变成*分离的*并继续运行。你必须显式调用 `.abort()` 来取消它。
> 这与直接 drop `Future` 不同 —— 直接 drop 会取消/停止底层的计算。

### Tokio 同步原语

Tokio 提供感知 async 的同步原语。关键原则：**不要在 `.await` 点之间使用 `std::sync::Mutex`**。

```rust
use tokio::sync::{Mutex, RwLock, Semaphore, mpsc, oneshot, broadcast, watch};

// --- Mutex ---
// 异步互斥锁：lock() 方法是异步的，不会阻塞线程
let data = Arc::new(Mutex::new(vec![1, 2, 3]));
{
    let mut guard = data.lock().await; // 非阻塞锁
    guard.push(4);
} // Guard 在这里 drop —— 锁释放

// --- Channels ---
// mpsc: 多生产者，单消费者
let (tx, mut rx) = mpsc::channel::<String>(100); // 有界缓冲区

tokio::spawn(async move {
    tx.send("hello".into()).await.unwrap();
});

let msg = rx.recv().await.unwrap();

// oneshot: 单值，单消费者
let (tx, rx) = oneshot::channel::<i32>();
tx.send(42).unwrap(); // 不需要 await —— 要么发送成功，要么失败
let val = rx.await.unwrap();

// broadcast: 多生产者，多消费者（所有接收者都收到每条消息）
let (tx, _) = broadcast::channel::<String>(100);
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// watch: 单值，多消费者（只有最新值）
let (tx, rx) = watch::channel(0u64);
tx.send(42).unwrap();
println!("Latest: {}", *rx.borrow());
```

> **注意**：这些 channel 示例中为了简洁使用了 `.unwrap()`。
> 在生产环境中，优雅地处理发送/接收错误 —— 失败的 `.send()` 意味着
> 接收者被 drop 了，失败的 `.recv()` 意味着 channel 已关闭。

```mermaid
graph LR
    subgraph "Channel Types"
        direction TB
        MPSC["mpsc<br/>N→1<br/>Buffered queue"]
        ONESHOT["oneshot<br/>1→1<br/>Single value"]
        BROADCAST["broadcast<br/>N→N<br/>All receivers get all"]
        WATCH["watch<br/>1→N<br/>Latest value only"]
    end

    P1["Producer 1"] --> MPSC
    P2["Producer 2"] --> MPSC
    MPSC --> C1["Consumer"]

    P3["Producer"] --> ONESHOT
    ONESHOT --> C2["Consumer"]

    P4["Producer"] --> BROADCAST
    BROADCAST --> C3["Consumer 1"]
    BROADCAST --> C4["Consumer 2"]

    P5["Producer"] --> WATCH
    WATCH --> C5["Consumer 1"]
    WATCH --> C6["Consumer 2"]
```

## 案例研究：为通知服务选择合适的 Channel

你正在构建一个通知服务，其中：
- 多个 API 处理器产生事件
- 单个后台任务批量发送它们
- 配置监视器在运行时更新速率限制
- 关闭信号必须到达所有组件

**每种情况选择哪种 channel？**

| 需求 | Channel | 原因 |
|------|---------|------|
| API 处理器 → Batcher | `mpsc`（有界） | N 个生产者，1 个消费者。有界用于反压 —— 如果 batcher 落后，API 处理器会减速而不是 OOM |
| 配置监视器 → 速率限制器 | `watch` | 只有最新配置重要。多个读取者（每个工作线程）看到当前值 |
| 关闭信号 → 所有组件 | `broadcast` | 每个组件必须独立接收关闭通知 |
| 单次健康检查响应 | `oneshot` | 请求/响应模式 —— 一个值，然后完成 |

```mermaid
graph LR
    subgraph "Notification Service"
        direction TB
        API1["API Handler 1"] -->|mpsc| BATCH["Batcher"]
        API2["API Handler 2"] -->|mpsc| BATCH
        CONFIG["Config Watcher"] -->|watch| RATE["Rate Limiter"]
        CTRL["Ctrl+C"] -->|broadcast| API1
        CTRL -->|broadcast| BATCH
        CTRL -->|broadcast| RATE
    end

    style API1 fill:#d4efdf,stroke:#27ae60,color:#000
    style API2 fill:#d4efdf,stroke:#27ae60,color:#000
    style BATCH fill:#e8f4f8,stroke:#2980b9,color:#000
    style CONFIG fill:#fef9e7,stroke:#f39c12,color:#000
    style RATE fill:#fef9e7,stroke:#f39c12,color:#000
    style CTRL fill:#fadbd8,stroke:#e74c3c,color:#000
```

<details>
<summary><strong>🏋️ 练习：构建任务池</strong>（点击展开）</summary>

**挑战**：构建一个函数 `run_with_limit`，接受异步闭包列表和并发限制，最多同时执行 N 个任务。使用 `tokio::sync::Semaphore`。

<details>
<summary>🔑 解答</summary>

```rust
use std::future::Future;
use std::sync::Arc;
use tokio::sync::Semaphore;

async fn run_with_limit<F, Fut, T>(tasks: Vec<F>, limit: usize) -> Vec<T>
where
    F: FnOnce() -> Fut + Send + 'static,
    Fut: Future<Output = T> + Send + 'static,
    T: Send + 'static,
{
    let semaphore = Arc::new(Semaphore::new(limit));
    let mut handles = Vec::new();

    for task in tasks {
        let permit = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            let _permit = permit.acquire().await.unwrap();
            // Permit 在任务运行时持有，然后 drop
            task().await
        });
        handles.push(handle);
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// 使用：
// let tasks: Vec<_> = urls.into_iter().map(|url| {
//     move || async move { fetch(url).await }
// }).collect();
// let results = run_with_limit(tasks, 10).await; // 最多 10 个并发
```

**关键要点**：`Semaphore` 是在 tokio 中限制并发的标准方式。每个任务在开始工作前获取一个 permit。当 semaphore 满时，新任务异步等待（非阻塞）直到有空闲槽位。

</details>
</details>

> **关键要点——Tokio Deep Dive**
> - 服务器使用 `multi_thread`（默认）；CLI 工具、测试或 `!Send` 类型使用 `current_thread`
> - `tokio::spawn` 需要 `'static` futures —— 使用 `Arc` 或 channels 共享数据
> - Drop `JoinHandle` **不会**取消任务 —— 显式调用 `.abort()`
> - 按需选择同步原语：`Mutex` 用于共享状态，`Semaphore` 用于并发限制，`mpsc`/`oneshot`/`broadcast`/`watch` 用于通信

> **另见：** [第 9 章 — 何时 Tokio 不是最佳选择](ch09-when-tokio-isnt-the-right-fit.md) 了解 spawn 的替代方案，[第 12 章 — 常见陷阱](ch12-common-pitfalls.md) 了解 MutexGuard 跨 await 的 bug

***
