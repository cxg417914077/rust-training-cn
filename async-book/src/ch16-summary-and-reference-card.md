# Summary and Reference Card

## 快速参考卡

### Async 思维模型

```text
┌─────────────────────────────────────────────────────┐
│  async fn → 状态机（enum） → impl Future            │
│  .await   → poll() 内部 future                       │
│  executor → loop { poll(); sleep_until_woken(); }   │
│  waker    → "嘿 executor，再 poll 我一次"             │
│  Pin      → "承诺我不在内存中移动"                   │
└─────────────────────────────────────────────────────┘
```

### 常见模式速查表

| 目标 | 使用 |
|------|-----|
| 并发运行两个 futures | `tokio::join!(a, b)` |
| 竞赛两个 futures | `tokio::select! { ... }` |
| 派生后台任务 | `tokio::spawn(async { ... })` |
| 在 async 中运行阻塞代码 | `tokio::task::spawn_blocking(\|\| { ... })` |
| 限制并发度 | `Semaphore::new(N)` |
| 收集许多任务结果 | `JoinSet` |
| 跨任务共享状态 | `Arc<Mutex<T>>` 或 channel |
| 优雅关闭 | `watch::channel` + `select!` |
| 一次处理 N 个 stream | `.buffer_unordered(N)` |
| Timeout 一个 future | `tokio::time::timeout(dur, fut)` |
| 带退避的重试 | 自定义组合器（见第 13 章） |

### Pinning 快速参考

| 情况 | 使用 |
|-----------|-----|
| 在堆上 pin future | `Box::pin(fut)` |
| 在栈上 pin future | `tokio::pin!(fut)` |
| Pin 一个 `Unpin` 类型 | `Pin::new(&mut val)` —— 安全，免费 |
| 返回 pinned trait object | `-> Pin<Box<dyn Future<Output = T> + Send>>` |

### Channel 选择指南

| Channel | 生产者 | 消费者 | 值 | 何时使用 |
|---------|-----------|-----------|--------|----------|
| `mpsc` | N | 1 | Stream | 工作队列、事件总线 |
| `oneshot` | 1 | 1 | 单个 | 请求/响应、完成通知 |
| `broadcast` | N | N | 所有接收者接收所有 | Fan-out 通知、关闭信号 |
| `watch` | 1 | N | 仅最新 | 配置更新、健康状态 |

### Mutex 选择指南

| Mutex | 何时使用 |
|-------|----------|
| `std::sync::Mutex` | 锁持有时间短，从不在 `.await` 跨 hold |
| `tokio::sync::Mutex` | 锁必须跨 `.await` 持有 |
| `parking_lot::Mutex` | 高竞争，无 `.await`，需要性能 |
| `tokio::sync::RwLock` | 多读少写，锁跨 `.await` |

### 决策快速参考

```text
需要并发？
├── I/O 绑定 → async/await
├── CPU 绑定 → rayon / std::thread
└── 混合 → CPU 部分用 spawn_blocking

选择 runtime？
├── 服务器应用 → tokio
├── 库 → runtime 无关（futures crate）
├── 嵌入式 → embassy
└── 最小化 → smol

需要并发 futures？
├── 可以是 'static + Send → tokio::spawn
├── 可以是 'static + !Send → LocalSet
├── 不能是 'static → FuturesUnordered
└── 需要跟踪/中止 → JoinSet
```

### 常见错误信息和修复

| 错误 | 原因 | 修复 |
|-------|-------|-----|
| `future is not Send` | 跨 `.await` 持有 `!Send` 类型 | 缩小作用域让它在 `.await` 前 drop，或使用 `current_thread` runtime |
| `borrowed value does not live long enough` 在 spawn 中 | `tokio::spawn` 需要 `'static` | 使用 `Arc`、`clone()`，或 `FuturesUnordered` |
| `the trait Future is not implemented for ()` | 缺少 `.await` | 给 async 调用添加 `.await` |
| `cannot borrow as mutable` 在 poll 中 | 自引用借用 | 正确使用 `Pin<&mut Self>`（见第 4 章） |
| 程序静默挂起 | 忘记调用 `waker.wake()` | 确保每个 `Pending` 路径注册并触发 waker |

### 进一步阅读

| 资源 | 为什么 |
|----------|-----|
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | 官方实践指南——非常适合入门项目 |
| [Async Book (official)](https://rust-lang.github.io/async-book/) | 在语言层面涵盖 `Future`、`Pin`、`Stream` |
| [Jon Gjengset — Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) | 2 小时内部深入探讨，带现场编码 |
| [Alice Ryhl — Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | 用于状态化服务的生产架构模式 |
| [Without Boats — Pin, Unpin, and why Rust needs them](https://without.boats/blog/pin/) | 来自语言设计者的原始动机 |
| [Tokio mini-redis](https://github.com/tokio-rs/mini-redis) | 完整的 async Rust 项目——学习级生产代码 |
| [Tower documentation](https://docs.rs/tower) | 被 axum、tonic、hyper 使用的中间件/service 架构 |

***

*Async Rust 训练指南结束*
