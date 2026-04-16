# 11. Streams and AsyncIterator 🟡

> **你将学到：**
> - `Stream` trait：异步迭代多个值
> - 创建 streams：`stream::iter`、`async_stream`、`unfold`
> - Stream 组合器：`map`、`filter`、`buffer_unordered`、`fold`
> - Async I/O traits：`AsyncRead`、`AsyncWrite`、`AsyncBufRead`

## Stream Trait 概述

`Stream` 对于 `Iterator` 就像 `Future` 对于单个值 —— 它异步地产出多个值：

```rust
// std::iter::Iterator (同步，多个值)
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// futures::Stream (异步，多个值)
trait Stream {
    type Item;
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;
}
```

```mermaid
graph LR
    subgraph "Sync"
        VAL["Value<br/>(T)"]
        ITER["Iterator<br/>(multiple T)"]
    end

    subgraph "Async"
        FUT["Future<br/>(async T)"]
        STREAM["Stream<br/>(async multiple T)"]
    end

    VAL -->|"make async"| FUT
    ITER -->|"make async"| STREAM
    VAL -->|"make multiple"| ITER
    FUT -->|"make multiple"| STREAM

    style VAL fill:#e3f2fd,color:#000
    style ITER fill:#e3f2fd,color:#000
    style FUT fill:#c8e6c9,color:#000
    style STREAM fill:#c8e6c9,color:#000
```

### 创建 Streams

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{interval, Duration};
use tokio_stream::wrappers::IntervalStream;

// 1. 从 iterator
let s = stream::iter(vec![1, 2, 3]);

// 2. 从 async generator（使用 async_stream crate）
// Cargo.toml: async-stream = "0.3"
use async_stream::stream;

fn countdown(from: u32) -> impl futures::Stream<Item = u32> {
    stream! {
        for i in (0..=from).rev() {
            tokio::time::sleep(Duration::from_millis(500)).await;
            yield i;
        }
    }
}

// 3. 从 tokio interval
let tick_stream = IntervalStream::new(interval(Duration::from_secs(1)));

// 4. 从 channel receiver（tokio_stream::wrappers）
let (tx, rx) = tokio::sync::mpsc::channel::<String>(100);
let rx_stream = tokio_stream::wrappers::ReceiverStream::new(rx);

// 5. 从 unfold（从异步状态生成）
let s = stream::unfold(0u32, |state| async move {
    if state >= 5 {
        None // Stream 结束
    } else {
        let next = state + 1;
        Some((state, next)) // yield `state`，新状态是 `next`
    }
});
```

### 消费 Streams

```rust
use futures::stream::{self, StreamExt};

async fn stream_examples() {
    let s = stream::iter(vec![1, 2, 3, 4, 5]);

    // for_each —— 处理每个 item
    s.for_each(|x| async move {
        println!("{x}");
    }).await;

    // map + collect
    let doubled: Vec<i32> = stream::iter(vec![1, 2, 3])
        .map(|x| x * 2)
        .collect()
        .await;

    // filter
    let evens: Vec<i32> = stream::iter(1..=10)
        .filter(|x| futures::future::ready(x % 2 == 0))
        .collect()
        .await;

    // buffer_unordered —— 并发处理 N 个 items
    let results: Vec<_> = stream::iter(vec!["url1", "url2", "url3"])
        .map(|url| async move {
            // 模拟 HTTP 获取
            tokio::time::sleep(Duration::from_millis(100)).await;
            format!("response from {url}")
        })
        .buffer_unordered(10) // 最多 10 个并发获取
        .collect()
        .await;

    // take, skip, zip, chain —— 就像 Iterator
    let first_three: Vec<i32> = stream::iter(1..=100)
        .take(3)
        .collect()
        .await;
}
```

### 与 C# IAsyncEnumerable 对比

| 特性 | Rust `Stream` | C# `IAsyncEnumerable<T>` |
|------|--------------|--------------------------|
| **语法** | `stream! { yield x; }` | `await foreach` / `yield return` |
| **取消** | Drop the stream | `CancellationToken` |
| **背压** | 消费者控制 poll 速率 | 消费者控制 `MoveNextAsync` |
| **内置支持** | 否（需要 `futures` crate） | 是（从 C# 8.0 开始） |
| **组合器** | `.map()`、`.filter()`、`.buffer_unordered()` | LINQ + `System.Linq.Async` |
| **错误处理** | `Stream<Item = Result<T, E>>` | 在 async iterator 中抛出 |

```rust
// Rust: 数据库行的 Stream
// 注意：当在 body 内使用 ? 时需要 try_stream!（不是 stream!）。
// stream! 不传播错误 —— try_stream! 产出 Err(e) 并结束。
fn get_users(db: &Database) -> impl Stream<Item = Result<User, DbError>> + '_ {
    try_stream! {
        let mut cursor = db.query("SELECT * FROM users").await?;
        while let Some(row) = cursor.next().await {
            yield User::from_row(row?);
        }
    }
}

// 消费：
let mut users = pin!(get_users(&db));
while let Some(result) = users.next().await {
    match result {
        Ok(user) => println!("{}", user.name),
        Err(e) => eprintln!("Error: {e}"),
    }
}
```

```csharp
// C# 等价实现：
async IAsyncEnumerable<User> GetUsers() {
    await using var reader = await db.QueryAsync("SELECT * FROM users");
    while (await reader.ReadAsync()) {
        yield return User.FromRow(reader);
    }
}

// 消费：
await foreach (var user in GetUsers()) {
    Console.WriteLine(user.Name);
}
```

<details>
<summary><strong>🏋️ 练习：构建异步统计聚合器</strong>（点击展开）</summary>

**挑战**：给定一个传感器读数的 stream `Stream<Item = f64>`，编写一个异步函数消费这个 stream 并返回 `(count, min, max, average)`。使用 `StreamExt` 组合器 —— 不要只收集到 Vec 中。

*提示*：使用 `.fold()` 跨 stream 累积状态。

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};

#[derive(Debug)]
struct Stats {
    count: usize,
    min: f64,
    max: f64,
    sum: f64,
}

impl Stats {
    fn average(&self) -> f64 {
        if self.count == 0 { 0.0 } else { self.sum / self.count as f64 }
    }
}

async fn compute_stats<S: futures::Stream<Item = f64> + Unpin>(stream: S) -> Stats {
    stream
        .fold(
            Stats { count: 0, min: f64::INFINITY, max: f64::NEG_INFINITY, sum: 0.0 },
            |mut acc, value| async move {
                acc.count += 1;
                acc.min = acc.min.min(value);
                acc.max = acc.max.max(value);
                acc.sum += value;
                acc
            },
        )
        .await
}

#[tokio::test]
async fn test_stats() {
    let readings = stream::iter(vec![23.5, 24.1, 22.8, 25.0, 23.9]);
    let stats = compute_stats(readings).await;

    assert_eq!(stats.count, 5);
    assert!((stats.min - 22.8).abs() < f64::EPSILON);
    assert!((stats.max - 25.0).abs() < f64::EPSILON);
    assert!((stats.average() - 23.86).abs() < 0.01);
}
```

**关键要点**：像 `.fold()` 这样的 stream 组合器一次处理一个 item 而不收集到内存中 —— 这对于处理大型或无界数据流至关重要。

</details>
</details>

### Async I/O Traits：AsyncRead、AsyncWrite、AsyncBufRead

就像 `std::io::Read`/`Write` 是同步 I/O 的基础一样，它们的 async 版本是 async I/O 的基础。这些 traits 由 `tokio::io` 提供（或 `futures::io` 用于 runtime 无关的代码）：

```rust
// tokio::io —— std::io traits 的 async 版本

/// 异步从源读取字节
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &mut ReadBuf<'_>,  // Tokio 对未初始化内存的安全包装
    ) -> Poll<io::Result<()>>;
}

/// 异步写字节到 sink
pub trait AsyncWrite {
    fn poll_write(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
        buf: &[u8],
    ) -> Poll<io::Result<usize>>;

    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
    fn poll_shutdown(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<()>>;
}

/// 支持行的缓冲读取
pub trait AsyncBufRead: AsyncRead {
    fn poll_fill_buf(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<io::Result<&[u8]>>;
    fn consume(self: Pin<&mut Self>, amt: usize);
}
```

**在实践中**，你很少直接调用这些 `poll_*` 方法。相反，使用扩展 traits `AsyncReadExt` 和 `AsyncWriteExt`，它们提供 `.await` 友好的辅助方法：

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt};
use tokio::net::TcpStream;
use tokio::io::BufReader;

async fn io_examples() -> tokio::io::Result<()> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    // AsyncWriteExt: write_all, write_u32, write_buf 等
    stream.write_all(b"GET / HTTP/1.0\r\n\r\n").await?;

    // AsyncReadExt: read, read_exact, read_to_end, read_to_string
    let mut response = Vec::new();
    stream.read_to_end(&mut response).await?;

    // AsyncBufReadExt: read_line, lines(), split()
    let file = tokio::fs::File::open("config.txt").await?;
    let reader = BufReader::new(file);
    let mut lines = reader.lines();
    while let Some(line) = lines.next_line().await? {
        println!("{line}");
    }

    Ok(())
}
```

**实现自定义 async I/O** —— 在原始 TCP 之上包装协议：

```rust
use tokio::io::{AsyncRead, AsyncWrite, ReadBuf};
use std::pin::Pin;
use std::task::{Context, Poll};

/// 长度前缀协议：[u32 length][payload bytes]
struct FramedStream<T> {
    inner: T,
}

impl<T: AsyncRead + AsyncReadExt + Unpin> FramedStream<T> {
    /// 读取一个完整的帧
    async fn read_frame(&mut self) -> tokio::io::Result<Vec<u8>>
    {
        // 读取 4 字节长度前缀
        let len = self.inner.read_u32().await? as usize;

        // 准确读取那么多字节
        let mut payload = vec![0u8; len];
        self.inner.read_exact(&mut payload).await?;
        Ok(payload)
    }
}

impl<T: AsyncWrite + AsyncWriteExt + Unpin> FramedStream<T> {
    /// 写入一个完整的帧
    async fn write_frame(&mut self, data: &[u8]) -> tokio::io::Result<()>
    {
        self.inner.write_u32(data.len() as u32).await?;
        self.inner.write_all(data).await?;
        self.inner.flush().await?;
        Ok(())
    }
}
```

| 同步 Trait | Async Trait (tokio) | Async Trait (futures) | 扩展 Trait |
|-----------|--------------------|-----------------------|------------|
| `std::io::Read` | `tokio::io::AsyncRead` | `futures::io::AsyncRead` | `AsyncReadExt` |
| `std::io::Write` | `tokio::io::AsyncWrite` | `futures::io::AsyncWrite` | `AsyncWriteExt` |
| `std::io::BufRead` | `tokio::io::AsyncBufRead` | `futures::io::AsyncBufRead` | `AsyncBufReadExt` |
| `std::io::Seek` | `tokio::io::AsyncSeek` | `futures::io::AsyncSeek` | `AsyncSeekExt` |

> **tokio vs futures I/O traits**：它们相似但不完全相同 —— tokio 的 `AsyncRead` 使用 `ReadBuf`（安全处理未初始化内存），而 `futures::AsyncRead` 使用 `&mut [u8]`。使用 `tokio_util::compat` 在它们之间转换。

> **Copy 工具**：`tokio::io::copy(&mut reader, &mut writer)` 是 `std::io::copy` 的 async 版本 —— 用于代理服务器或文件传输。`tokio::io::copy_bidirectional` 并发地双向复制。

<details>
<summary><strong>🏋️ 练习：构建异步行计数器</strong>（点击展开）</summary>

**挑战**：编写一个异步函数，接受任何 `AsyncBufRead` 源并返回非空行的数量。它应该适用于文件、TCP 流或任何缓冲读取器。

*提示*：使用 `AsyncBufReadExt::lines()` 并计算 `!line.is_empty()` 的行。

<details>
<summary>🔑 解答</summary>

```rust
use tokio::io::AsyncBufReadExt;

async fn count_non_empty_lines<R: tokio::io::AsyncBufRead + Unpin>(
    reader: R,
) -> tokio::io::Result<usize> {
    let mut lines = reader.lines();
    let mut count = 0;
    while let Some(line) = lines.next_line().await? {
        if !line.is_empty() {
            count += 1;
        }
    }
    Ok(count)
}

// 适用于任何 AsyncBufRead：
// let file = tokio::io::BufReader::new(tokio::fs::File::open("data.txt").await?);
// let count = count_non_empty_lines(file).await?;
//
// let tcp = tokio::io::BufReader::new(TcpStream::connect("...").await?);
// let count = count_non_empty_lines(tcp).await?;
```

**关键要点**：通过对 `AsyncBufRead` 编程而不是具体类型，你的 I/O 代码可以在文件、socket、管道甚至内存缓冲区（`tokio::io::BufReader::new(std::io::Cursor::new(data))`）之间复用。

</details>
</details>

> **关键要点——Streams and AsyncIterator**
> - `Stream` 是 `Iterator` 的 async 版本 —— 产出 `Poll::Ready(Some(item))` 或 `Poll::Ready(None)`
> - `.buffer_unordered(N)` 并发处理 N 个 stream items —— streams 的关键并发工具
> - `async_stream::stream!` 是创建自定义 streams 的最简单方式（使用 `yield`）
> - `AsyncRead`/`AsyncBufRead` 支持通用的、可复用的 I/O 代码，适用于文件、socket 和管道

> **另见：** [第 9 章 — 何时 Tokio 不是最佳选择](ch09-when-tokio-isnt-the-right-fit.md) 了解 `FuturesUnordered`（相关模式），[第 13 章 — 生产级模式](ch13-production-patterns.md) 了解有界 channel 的背压

***
