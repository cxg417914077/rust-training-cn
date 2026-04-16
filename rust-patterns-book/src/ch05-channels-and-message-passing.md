# 5. Channels and Message Passing 🟢

> **你将学到：**
> - `std::sync::mpsc` 基础以及何时升级到 crossbeam-channel
> - 使用 `select!` 进行 channel 选择，处理多源消息
> - 有界 vs 无界 channel 和背压策略
> - 用于封装并发状态的 actor 模式

## std::sync::mpsc — 标准 Channel

Rust 标准库提供了一个多生产者、单消费者的 channel：

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // 创建一个 channel：tx（发送者）和 rx（接收者）
    let (tx, rx) = mpsc::channel();

    // 生成一个生产者线程
    let tx1 = tx.clone(); // 克隆以支持多生产者
    thread::spawn(move || {
        for i in 0..5 {
            tx1.send(format!("producer-1: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // 第二个生产者
    thread::spawn(move || {
        for i in 0..5 {
            tx.send(format!("producer-2: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });

    // 消费者：接收所有消息
    for msg in rx {
        // 当所有发送者都 drop 时，rx 迭代器结束
        println!("Received: {msg}");
    }
    println!("All producers done.");
}
```

> **注意：** `.send()` 上的 `.unwrap()` 是为了简洁。如果接收者已被 drop，它会 panic。生产代码应该优雅地处理 `SendError`。

**关键属性**：
- 默认**无界**（如果消费者慢，可能填满内存）
- `mpsc::sync_channel(N)` 创建一个**有界** channel，具有背压
- `rx.recv()` 阻塞当前线程，直到消息到达
- `rx.try_recv()` 立即返回，如果没有准备好则返回 `Err(TryRecvError::Empty)`
- 当所有 `Sender` 都被 drop 时，channel 关闭

```rust
// 带有背压的有界 channel：
let (tx, rx) = mpsc::sync_channel(10); // 10 条消息的缓冲区

thread::spawn(move || {
    for i in 0..1000 {
        tx.send(i).unwrap(); // 如果缓冲区满则阻塞 —— 自然的背压
    }
});
```

> **注意：** `.unwrap()` 是为了简洁。在生产环境中，应该处理 `SendError`（接收者被 drop）而不是 panic。

### crossbeam-channel — 生产主力

`crossbeam-channel` 是生产环境中 channel 使用的实际标准。它比 `std::sync::mpsc` 更快，并支持多消费者（`mpmc`）：

```rust,ignore
// Cargo.toml:
//   [dependencies]
//   crossbeam-channel = "0.5"
use crossbeam_channel::{bounded, unbounded, select, Sender, Receiver};
use std::thread;
use std::time::Duration;

fn main() {
    // 有界 MPMC channel
    let (tx, rx) = bounded::<String>(100);

    // 多个生产者
    for id in 0..4 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..10 {
                tx.send(format!("worker-{id}: item-{i}")).unwrap();
            }
        });
    }
    drop(tx); // drop 原始发送者，让 channel 可以关闭

    // 多个消费者（std::sync::mpsc 不支持！）
    let rx2 = rx.clone();
    let consumer1 = thread::spawn(move || {
        while let Ok(msg) = rx.recv() {
            println!("[consumer-1] {msg}");
        }
    });
    let consumer2 = thread::spawn(move || {
        while let Ok(msg) = rx2.recv() {
            println!("[consumer-2] {msg}");
        }
    });

    consumer1.join().unwrap();
    consumer2.join().unwrap();
}
```

### Channel 选择 (select!)

同时监听多个 channel —— 就像 Go 中的 `select`：

```rust,ignore
use crossbeam_channel::{bounded, tick, after, select};
use std::time::Duration;

fn main() {
    let (work_tx, work_rx) = bounded::<String>(10);
    let ticker = tick(Duration::from_secs(1));        // 周期性 tick
    let deadline = after(Duration::from_secs(10));     // 单次超时

    // 生产者
    let tx = work_tx.clone();
    std::thread::spawn(move || {
        for i in 0..100 {
            tx.send(format!("job-{i}")).unwrap();
            std::thread::sleep(Duration::from_millis(500));
        }
    });
    drop(work_tx);

    loop {
        select! {
            recv(work_rx) -> msg => {
                match msg {
                    Ok(job) => println!("Processing: {job}"),
                    Err(_) => {
                        println!("Work channel closed");
                        break;
                    }
                }
            },
            recv(ticker) -> _ => {
                println!("Tick — heartbeat");
            },
            recv(deadline) -> _ => {
                println!("Deadline reached — shutting down");
                break;
            },
        }
    }
}
```

> **Go 对比：** 这完全像 Go 的 `select` 语句用于 channel。
> crossbeam 的 `select!` 宏随机化顺序以防止饥饿，就像 Go 一样。

### 有界 vs 无界和背压

| 类型 | 满时的行为 | 内存 | 使用场景 |
|------|-------------------|--------|----------|
| **无界** | 从不阻塞（堆增长） | 无界 ⚠️ | 罕见 —— 仅当生产者比消费者慢时 |
| **有界** | `send()` 阻塞直到有空闲 | 固定 | 生产默认 —— 防止 OOM |
| **Rendezvous** (bounded(0)) | `send()` 阻塞直到接收者准备好 | 无 | 同步/交接 |

```rust
// Rendezvous channel —— 零容量，直接交接
let (tx, rx) = crossbeam_channel::bounded(0);
// tx.send(x) 阻塞直到 rx.recv() 被调用，反之亦然。
// 这精确地同步了两个线程。
```

**规则**：在生产环境中总是使用有界 channel，除非你能证明生产者永远不会超过消费者的速度。

### 使用 Channel 的 Actor 模式

actor 模式使用 channel 来序列化对可变状态的访问 —— 不需要 mutex：

```rust
use std::sync::mpsc;
use std::thread;

// actor 可以接收的消息
enum CounterMsg {
    Increment,
    Decrement,
    Get(mpsc::Sender<i64>), // 回复 channel
}

struct CounterActor {
    count: i64,
    rx: mpsc::Receiver<CounterMsg>,
}

impl CounterActor {
    fn new(rx: mpsc::Receiver<CounterMsg>) -> Self {
        CounterActor { count: 0, rx }
    }

    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CounterMsg::Increment => self.count += 1,
                CounterMsg::Decrement => self.count -= 1,
                CounterMsg::Get(reply) => {
                    let _ = reply.send(self.count);
                }
            }
        }
    }
}

// Actor handle —— 克隆成本低，Send + Sync
#[derive(Clone)]
struct Counter {
    tx: mpsc::Sender<CounterMsg>,
}

impl Counter {
    fn spawn() -> Self {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || CounterActor::new(rx).run());
        Counter { tx }
    }

    fn increment(&self) { let _ = self.tx.send(CounterMsg::Increment); }
    fn decrement(&self) { let _ = self.tx.send(CounterMsg::Decrement); }

    fn get(&self) -> i64 {
        let (reply_tx, reply_rx) = mpsc::channel();
        self.tx.send(CounterMsg::Get(reply_tx)).unwrap();
        reply_rx.recv().unwrap()
    }
}

fn main() {
    let counter = Counter::spawn();

    // 多个线程可以安全地使用 counter —— 不需要 mutex！
    let handles: Vec<_> = (0..10).map(|_| {
        let counter = counter.clone();
        thread::spawn(move || {
            for _ in 0..1000 {
                counter.increment();
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
    println!("Final count: {}", counter.get()); // 10000
}
```

> **何时使用 actor vs mutex**：当状态有复杂的不变量、操作需要很长时间，或者你想要序列化访问而不必考虑锁顺序时，actor 很好。Mutex 对于短的临界区更简单。

> **关键要点 —— Channels**
> - `crossbeam-channel` 是生产主力 —— 比 `std::sync::mpsc` 更快、功能更丰富
> - `select!` 用声明式的 channel 选择替代复杂的多源轮询
> - 有界 channel 提供自然的背压；无界 channel 有 OOM 风险

> **另见：** [第 6 章 — 并发](ch06-concurrency-vs-parallelism-vs-threads.md) 了解线程、Mutex 和共享状态。[第 16 章 — Async](ch16-asyncawait-essentials.md) 了解 async channel（`tokio::sync::mpsc`）。

---

### 练习：基于 Channel 的工作池 ★★★（约 45 分钟）

构建一个使用 channel 的工作池，其中：
- 调度器通过 channel 发送 `Job` 结构体
- N 个工作线程消费作业并发送结果回来
- 使用 `std::sync::mpsc` 和 `Arc<Mutex<Receiver>>` 实现共享工作队列

<details>
<summary>🔑 解答</summary>

```rust
use std::sync::mpsc;
use std::thread;

struct Job {
    id: u64,
    data: String,
}

struct JobResult {
    job_id: u64,
    output: String,
    worker_id: usize,
}

fn worker_pool(jobs: Vec<Job>, num_workers: usize) -> Vec<JobResult> {
    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let (result_tx, result_rx) = mpsc::channel::<JobResult>();

    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv()
                };
                match job {
                    Ok(job) => {
                        let output = format!("processed '{}' by worker {worker_id}", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id, output, worker_id,
                        }).unwrap();
                    }
                    Err(_) => break,
                }
            }
        }));
    }
    drop(result_tx);

    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx);

    let results: Vec<_> = result_rx.into_iter().collect();
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i, data: format!("task-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[worker {}] job {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

***
