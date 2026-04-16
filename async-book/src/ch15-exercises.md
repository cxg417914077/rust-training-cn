## 练习

### 练习 1：Async Echo 服务器

构建一个 TCP echo 服务器，并发处理多个客户端。

**要求**：
- 监听 `127.0.0.1:8080`
- 接受连接并回显每一行
- 优雅地处理客户端断开连接
- 在客户端连接/断开时打印日志

<details>
<summary>🔑 解答</summary>

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Echo server listening on :8080");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("[{addr}] Connected");

        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) => {
                        println!("[{addr}] Disconnected");
                        break;
                    }
                    Ok(_) => {
                        print!("[{addr}] Echo: {line}");
                        if writer.write_all(line.as_bytes()).await.is_err() {
                            println!("[{addr}] Write error, disconnecting");
                            break;
                        }
                    }
                    Err(e) => {
                        eprintln!("[{addr}] Read error: {e}");
                        break;
                    }
                }
            }
        });
    }
}
```

</details>

---

### 练习 2：带限流的并发 URL 获取器

并发获取 URL 列表，最多 5 个并发请求。

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

async fn fetch_urls(urls: Vec<String>) -> Vec<Result<String, String>> {
    // buffer_unordered(5) 确保最多 5 个 futures 并发 poll
    // —— 这里不需要单独的 Semaphore。
    let results: Vec<_> = stream::iter(urls)
        .map(|url| {
            async move {
                println!("Fetching: {url}");

                match reqwest::get(&url).await {
                    Ok(resp) => match resp.text().await {
                        Ok(body) => Ok(body),
                        Err(e) => Err(format!("{url}: {e}")),
                    },
                    Err(e) => Err(format!("{url}: {e}")),
                }
            }
        })
        .buffer_unordered(5) // ← 这 alone 限制并发度为 5
        .collect()
        .await;

    results
}

// 注意：当你需要在独立派生的任务（tokio::spawn）之间限制并发度时使用 Semaphore。
// 当处理 stream 时使用 buffer_unordered。不要为相同的限制结合两者。
```

</details>

---

### 练习 3：带工作池的优雅关闭

构建一个任务处理器，具有：
- 基于 channel 的工作队列
- N 个工作任务从队列消费
- Ctrl+C 时的优雅关闭：停止接受，完成进行中的工作

<details>
<summary>🔑 解答</summary>

```rust
use tokio::sync::{mpsc, watch};
use tokio::time::{sleep, Duration};

struct WorkItem {
    id: u64,
    payload: String,
}

#[tokio::main]
async fn main() {
    let (work_tx, work_rx) = mpsc::channel::<WorkItem>(100);
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // 派生 4 个工作者
    let mut worker_handles = Vec::new();
    let work_rx = std::sync::Arc::new(tokio::sync::Mutex::new(work_rx));

    for id in 0..4 {
        let rx = work_rx.clone();
        let mut shutdown = shutdown_rx.clone();
        let handle = tokio::spawn(async move {
            loop {
                let item = {
                    let mut rx = rx.lock().await;
                    tokio::select! {
                        item = rx.recv() => item,
                        _ = shutdown.changed() => {
                            if *shutdown.borrow() { None } else { continue }
                        }
                    }
                };

                match item {
                    Some(work) => {
                        println!("Worker {id}: processing item {}", work.id);
                        sleep(Duration::from_millis(200)).await; // 模拟工作
                        println!("Worker {id}: done with item {}", work.id);
                    }
                    None => {
                        println!("Worker {id}: channel closed, exiting");
                        break;
                    }
                }
            }
        });
        worker_handles.push(handle);
    }

    // 生产者：提交一些工作
    let producer = tokio::spawn(async move {
        for i in 0..20 {
            let _ = work_tx.send(WorkItem {
                id: i,
                payload: format!("task-{i}"),
            }).await;
            sleep(Duration::from_millis(50)).await;
        }
    });

    // 等待 Ctrl+C
    tokio::signal::ctrl_c().await.unwrap();
    println!("\nShutdown signal received!");
    shutdown_tx.send(true).unwrap();
    producer.abort(); // 取消生产者任务

    // 等待工作者完成
    for handle in worker_handles {
        let _ = handle.await;
    }
    println!("All workers shut down. Goodbye!");
}
```

</details>

---

### 练习 4：从零开始构建简单的 Async Mutex

实现一个 async 感知的 mutex，使用 channel（不使用 `tokio::sync::Mutex`）。

*提示*：使用带有 1 个 permit 的 `tokio::sync::Semaphore` 来序列化访问。

<details>
<summary>🔑 解答</summary>

```rust
use std::cell::UnsafeCell;
use std::sync::Arc;
use tokio::sync::{OwnedSemaphorePermit, Semaphore};

pub struct SimpleAsyncMutex<T> {
    data: Arc<UnsafeCell<T>>,
    semaphore: Arc<Semaphore>,
}

// 安全性：对 T 的访问被 semaphore 序列化（最多 1 个 permit）。
unsafe impl<T: Send> Send for SimpleAsyncMutex<T> {}
unsafe impl<T: Send> Sync for SimpleAsyncMutex<T> {}

pub struct SimpleGuard<T> {
    data: Arc<UnsafeCell<T>>,
    _permit: OwnedSemaphorePermit, // 在 guard drop 时释放——释放锁
}

impl<T> SimpleAsyncMutex<T> {
    pub fn new(value: T) -> Self {
        SimpleAsyncMutex {
            data: Arc::new(UnsafeCell::new(value)),
            semaphore: Arc::new(Semaphore::new(1)),
        }
    }

    pub async fn lock(&self) -> SimpleGuard<T> {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        SimpleGuard {
            data: self.data.clone(),
            _permit: permit,
        }
    }
}

impl<T> std::ops::Deref for SimpleGuard<T> {
    type Target = T;
    fn deref(&self) -> &T {
        // 安全性：我们持有唯一的 semaphore permit，所以没有其他
        // SimpleGuard 存在——保证独占访问。
        unsafe { &*self.data.get() }
    }
}

impl<T> std::ops::DerefMut for SimpleGuard<T> {
    fn deref_mut(&mut self) -> &mut T {
        // 安全性：同样的推理——单个 permit 保证独占性。
        unsafe { &mut *self.data.get() }
    }
}

// 当 SimpleGuard 被 drop 时，_permit 被 drop，
// 这释放 semaphore permit——另一个 lock() 可以进行。

// 用法：
// let mutex = SimpleAsyncMutex::new(vec![1, 2, 3]);
// {
//     let mut guard = mutex.lock().await;
//     guard.push(4);
// } // permit 在这里释放
```

**关键要点**：Async mutex 通常构建在 semaphore 之上。Semaphore 提供 async 等待机制——当锁定时，`acquire()` 挂起任务直到 permit 被释放。这正是 `tokio::sync::Mutex` 内部的工作方式。

> **为什么使用 `UnsafeCell` 而不是 `std::sync::Mutex`？** 这个练习的早期版本使用 `Arc<Mutex<T>>` 搭配 `Deref`/`DerefMut` 调用 `.lock().unwrap()`。那无法编译——返回的 `&T` 从临时的 `MutexGuard` 借用，而它立即被 drop。`UnsafeCell` 避免了中间 guard，而基于 semaphore 的序列化使 `unsafe` 可靠。

</details>

---

### 练习 5：Stream 管道

构建一个使用 stream 的数据处理管道：
1. 生成数字 1..=100
2. 过滤偶数
3. 映射每个数字为其平方
4. 一次并发处理 10 个（用 sleep 模拟）
5. 收集结果

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let results: Vec<u64> = stream::iter(1u64..=100)
        // 步骤 2：过滤偶数
        .filter(|x| futures::future::ready(x % 2 == 0))
        // 步骤 3：平方每个
        .map(|x| x * x)
        // 步骤 4：并发处理（模拟 async 工作）
        .map(|x| async move {
            sleep(Duration::from_millis(50)).await;
            println!("Processed: {x}");
            x
        })
        .buffer_unordered(10) // 10 个并发
        // 步骤 5：收集
        .collect()
        .await;

    println!("Got {} results", results.len());
    println!("Sum: {}", results.iter().sum::<u64>());
}
```

</details>

---

### 练习 6：用 Timeout 实现 Select

不使用 `tokio::select!` 或 `tokio::time::timeout`，实现一个函数将 future 与截止时间竞赛，返回 `Either::Left(result)` 或超时时返回 `Either::Right(())`。

*提示*：基于第 6 章的 `Select` 组合器和同一章中的 `TimerFuture`。

<details>
<summary>🔑 解答</summary>

```rust,ignore
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

pub enum Either<A, B> {
    Left(A),
    Right(B),
}

pub struct Timeout<F> {
    future: F,
    timer: TimerFuture, // 来自第 6 章
}

impl<F: Future + Unpin> Timeout<F> {
    pub fn new(future: F, duration: Duration) -> Self {
        Timeout {
            future,
            timer: TimerFuture::new(duration),
        }
    }
}

impl<F: Future + Unpin> Future for Timeout<F> {
    type Output = Either<F::Output, ()>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 检查主 future 是否完成
        if let Poll::Ready(val) = Pin::new(&mut self.future).poll(cx) {
            return Poll::Ready(Either::Left(val));
        }

        // 检查定时器是否到期
        if let Poll::Ready(()) = Pin::new(&mut self.timer).poll(cx) {
            return Poll::Ready(Either::Right(()));
        }

        Poll::Pending
    }
}

// 用法：
// match Timeout::new(fetch_data(), Duration::from_secs(5)).await {
//     Either::Left(data) => println!("Got data: {data}"),
//     Either::Right(()) => println!("Timed out!"),
// }
```

**关键要点**：`select`/`timeout` 只是 poll 两个 futures 并看哪个先完成。整个 async 生态系统构建在这个简单的原语之上：poll、Pending/Ready、Waker。

</details>

***
