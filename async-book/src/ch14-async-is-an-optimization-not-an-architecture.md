# 14. Async Is an Optimization, Not an Architecture 🔴

> **你将学到：**
> - 为什么 async 往往会污染整个代码库——以及为什么这是设计缺陷，不是特性
> - "Sync core, async shell"模式让大多数代码可测试、可调试
> - 如何处理困难情况：*也需要* I/O 的逻辑
> - `spawn_blocking` 何时是修复方案 vs. 症状
> - async 何时真正属于你的核心逻辑
> - 为什么 sync-first 库比 async-first 库更具可组合性

你已经花了 13 章学习 Rust 异步。但这本书还没告诉你的最重要的事情是：**你的大多数代码不应该是 async。**

## 函数染色问题

Bob Nystrom 的 ["What Color is Your Function?"](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) 识别了核心问题：async 函数可以调用 sync 函数，但 sync 函数不能调用 async 函数。一旦一个函数变成 async，调用链中它上面的所有函数都必须跟随。

在 Rust 中这比 C# 或 JavaScript **更糟**，因为 async 不仅感染函数签名——它还感染类型：

| Sync 代码 | Async 等价物 | 为什么不同 |
|---|---|---|
| `fn process(&self)` | `async fn process(&self)` | 调用者也必须是 async |
| `&mut T` | `Arc<Mutex<T>>` | 派生的任务需要 `'static + Send` |
| `std::sync::Mutex` | `tokio::sync::Mutex` | 如果跨 `.await` 持有则类型不同 |
| `impl Trait` 返回值 | `impl Future<Output = T> + Send` | 自 RPITIT (Rust 1.75, ch10) 后更简单，但仍有颜色 |
| `#[test]` | `#[tokio::test]` | 测试需要 runtime |
| 栈帧：5 层 | 栈帧：25 层 | 一半是 runtime 内部 |

每一行都是一个必须做出、做对并维护的决定——而这些都与业务逻辑无关。业界正在*远离*这种模式：Java 的 Project Loom（虚拟线程）和 Go 的 goroutine 都让你编写看起来同步的代码，由 runtime 廉价地多路复用。Rust 选择显式 async 来实现零成本控制，但这种控制有复杂度成本，应该有意识地支付，而不是默认支付。

## "但线程很昂贵"

反射式的反驳："我们需要 async 因为线程很昂贵。"在大多数团队运营的规模上，这*大多*是错误的。

- **栈内存：** 每个 OS 线程预留 8MB 虚拟地址空间（Linux 默认），但 OS 只提交触及的页面——一个基本空闲的线程使用 20-80KB 物理内存。
- **上下文切换：** 现代硬件上约 1-5µs。在 50 个并发请求下，这是噪音。在 100K 次切换/秒时，这是可测量的。
- **创建成本：** Linux 上每个线程约 10-30µs。线程池（rayon、`std::thread::scope`）将此摊销到零。

诚实地说，async 赚取其复杂度的阈值大约是 **1K-10K 个并发的大多数空闲连接**——这是 epoll/io_uring 的甜蜜点，此时每个连接的栈成为真正的成本。低于此，线程池更简单、更快调试、且足够快。高于此，async 胜出。大多数服务低于此。

## 困难示例：也需要 I/O 的逻辑

一个简单的纯函数——`fn add(a: i32, b: i32) -> i32`——显然不需要 async。这不是一个有趣的教训。有趣的情况是当业务逻辑*似乎*需要在中间进行 I/O：检查库存的验证、查询汇率的定价、查找客户的订单管道。

考虑一个订单处理服务。async-everywhere 版本看起来自然：

### 版本 A：Async 贯穿核心

```rust
// orders.rs — async 一路向下

pub async fn process_order(order: Order) -> Result<Receipt, OrderError> {
    // 步骤 1：验证——纯业务规则，无 I/O
    validate_items(&order)?;
    validate_quantities(&order)?;

    // 步骤 2：检查库存——需要数据库调用
    let stock = inventory_client.check(&order.items).await?;
    if !stock.all_available() {
        return Err(OrderError::OutOfStock(stock.missing()));
    }

    // 步骤 3：计算定价——纯数学，但 async 因为我们已经在这里
    let pricing = calculate_pricing(&order, &stock);

    // 步骤 4：应用折扣——需要外部服务调用
    let discount = discount_service.lookup(order.customer_id).await?;
    let final_price = pricing.apply_discount(discount);

    // 步骤 5：格式化收据——纯
    Ok(Receipt::new(order, final_price))
}
```

这是*合理的* async 代码。没有 `Arc<Mutex>` 滥用——只是顺序的 await。大多数开发者会这样写然后继续。但看看发生了什么：`validate_items`、`validate_quantities`、`calculate_pricing` 和 `Receipt::new` 都是纯函数，但因为步骤 2 和 4 需要 I/O 而被拖入 async 上下文。整个函数必须是 async，它的测试需要 runtime，链中的每个调用者现在都被染色。

### 版本 B：Sync Core, Async Shell

替代方案：分离*决定什么*和*如何获取*：

```rust
// core.rs — 纯业务逻辑，零 async，零 tokio 依赖

pub fn validate_order(order: &Order) -> Result<ValidatedOrder, OrderError> {
    validate_items(order)?;
    validate_quantities(order)?;
    Ok(ValidatedOrder::from(order))
}

pub fn check_stock(
    order: &ValidatedOrder,
    stock: &StockResult,
) -> Result<StockedOrder, OrderError> {
    if !stock.all_available() {
        return Err(OrderError::OutOfStock(stock.missing()));
    }
    Ok(StockedOrder::from(order, stock))
}

pub fn finalize(
    order: &StockedOrder,
    discount: Discount,
) -> Receipt {
    let pricing = calculate_pricing(order);
    let final_price = pricing.apply_discount(discount);
    Receipt::new(order, final_price)
}
```

```rust
// shell.rs — 薄 async 编排器
//
// 注意：网络调用上的 `?` 需要 `impl From<reqwest::Error> for OrderError`
//（或统一的错误 enum）。参见 ch12 了解 async 错误处理模式。

use crate::core;

pub async fn process_order(order: Order) -> Result<Receipt, OrderError> {
    // Sync：验证
    let validated = core::validate_order(&order)?;

    // Async：获取库存（这是 shell 的工作）
    let stock = inventory_client.check(&validated.items).await?;

    // Sync：将业务规则应用于获取的数据
    let stocked = core::check_stock(&validated, &stock)?;

    // Async：获取折扣
    let discount = discount_service.lookup(order.customer_id).await?;

    // Sync：完成
    Ok(core::finalize(&stocked, discount))
}
```

async shell 是一个 **fetch → decide → fetch → decide 的管道**。每个 "decide" 步骤是一个 sync 函数，它将 I/O 结果作为输入而不是自己去获取。

### 测试差异

Sync core 测试每个业务规则而不需要 runtime 或 mocks：

```rust
#[test]
fn out_of_stock_rejects_order() {
    let order = validated_order(vec![item("widget", 10)]);
    let stock = stock_result(vec![("widget", 3)]); // 只有 3 个可用

    let result = core::check_stock(&order, &stock);
    assert_eq!(result.unwrap_err(), OrderError::OutOfStock(vec!["widget"]));
}

#[test]
fn discount_applied_correctly() {
    let order = stocked_order(100_00); // 价格（美分）
    let receipt = core::finalize(&order, Discount::Percent(15));
    assert_eq!(receipt.final_price, 85_00);
}
```

Async shell 获得一个更薄的*集成*测试来验证连接，而不是逻辑：

```rust
#[tokio::test]
async fn process_order_integration() {
    let mock_inventory = mock_service(/* 返回库存 */);
    let mock_discounts = mock_service(/* 返回 10% */);
    let receipt = process_order(sample_order()).await.unwrap();
    assert!(receipt.final_price > 0);
    // 逻辑正确性已经由上面的 core 测试证明
}
```

### 为什么这很重要

| 关注点 | Async 贯穿核心 | Sync core + async shell |
|---|---|---|
| 业务规则可测试而无需 runtime | 否 | **是** |
| 需要 `#[tokio::test]` 的单元测试数量 | 全部 | **只有集成测试** |
| I/O 失败与逻辑错误纠缠 | 是——一个 `Result` 类型用于两者 | **否**——sync 返回逻辑错误，shell 处理 I/O 错误 |
| `validate_order` 可在 CLI / WASM / batch 中复用 | 否——传递依赖 tokio | **是**——纯 `fn` |
| 通过业务逻辑的栈追踪 | 与 runtime 帧交错 | **干净** |
| 以后可以交换 HTTP 客户端为 gRPC | 需要更改核心函数 | **只改 Shell** |

关键见解：**步骤 2 和 4 中的 I/O 调用*不需要*在业务逻辑内部。它们是它的输入。** Sync core 将 `StockResult` 和 `Discount` 作为参数。这些值来自哪里——HTTP、gRPC、测试夹具、缓存——是 shell 的事情。

## `spawn_blocking` 气味

第 12 章介绍了 `spawn_blocking` 作为意外阻塞 executor 的修复方案。当你有一个临时的阻塞调用时，这是正确的修复——`std::fs::read`、压缩库、传统 FFI 函数。

但如果你发现自己将大块代码包装在 `spawn_blocking` 中：

```rust
async fn handler(req: Request) -> Response {
    // 如果这是你的代码库，边界在错误的地方
    tokio::task::spawn_blocking(move || {
        let validated = validate(&req);       // sync
        let enriched = enrich(validated);      // sync
        let result = process(enriched);        // sync
        let output = format_response(result);  // sync
        output
    }).await.unwrap()
}
```

...这是代码库在告诉你：**这个逻辑从来就不是 async。** 你不需要 `spawn_blocking`——你需要一个 async handler 直接调用的 sync 模块：

```rust
async fn handler(req: Request) -> Response {
    // validate → enrich → process → format 都是 sync。
    // 不需要 spawn_blocking——它们很快且 CPU 轻量。
    let response = my_core::handle(req);
    response
}
```

保留 `spawn_blocking` 用于真正繁重的 CPU 工作（解析大型负载、图像处理、压缩），此时时间成本实际上会使 executor 饥饿。对于在微秒内运行的普通业务逻辑，直接 sync 调用更简单且正确。

## 库：Sync First, Async Wrapper Optional

边界问题对库作者影响更大。一个 sync 库可以被 sync 和 async 调用者使用：

```rust
// 一个 sync 库——到处可用
let report = my_lib::analyze(&data);

// 调用者 A：sync CLI
fn main() {
    let report = my_lib::analyze(&data);
    println!("{report}");
}

// 调用者 B：async handler，工作正常
async fn handler() -> Json<Report> {
    let report = my_lib::analyze(&data); // async 上下文中的 sync 调用——没问题
    Json(report)
}

// 调用者 C：繁重的分析——调用者决定卸载
async fn handler_heavy() -> Json<Report> {
    let data = data.clone();
    let report = tokio::task::spawn_blocking(move || {
        my_lib::analyze(&data) // 调用者控制 async 边界
    }).await.unwrap();
    Json(report)
}
```

一个 async 库强制*所有*调用者进入 runtime：

```rust
// 一个 async 库——只能从 async 上下文使用
let report = my_lib::analyze(&data).await; // 调用者*必须*是 async

// Sync 调用者？现在你需要 block_on——并希望内部没有 nested runtime
let report = tokio::runtime::Runtime::new().unwrap().block_on(
    my_lib::analyze(&data)
); // 脆弱，如果已经在 runtime 内则容易 panic
```

**默认使用 sync API。** 如果你的库做纯计算、数据转换或解析，没有理由让它变成 async。如果它做 I/O，考虑提供一个 sync core 并在特性标志后提供一个可选的 async 便利层——让调用者拥有边界决定。

## Async 何时属于核心

不是所有事情都能干净地分离。当以下情况时，Async 属于你的核心逻辑：

- **Fan-out/fan-in 就是逻辑。** 如果你的业务规则是"并发查询 5 个定价服务并返回最便宜的"，并发*就是*逻辑，不是管道。强制通过 sync + 线程是重新发明一个更糟的 async。

- **Streaming 就是逻辑。** 处理带有背压的连续事件流——流管理是非平凡的业务逻辑，不只是一个 I/O 包装器。

- **长持有状态连接。** WebSocket 处理器、gRPC 双向流和协议状态机具有与 I/O 事件固有绑定的状态转换。[ch17](ch17-capstone-project.md) 中的综合项目——一个异步聊天服务器——正是这种情况：并发连接、基于房间的 fan-out 和优雅关闭是根本上的 async 工作。

**测试：** 如果从函数中移除 `async` 需要用线程、channel 或手动轮询替换它，那么 async 在发挥作用。如果移除 `async` 只是删除关键字而没有其他更改，它从来就不需要是 async。

## 决策规则

```mermaid
graph TD
    START["这个函数应该是 async 吗？"] --> IO{"它做 I/O 吗？"}
    IO -->|否 | SYNC["sync fn —— 总是"]
    IO -->|是 | BOUNDARY{"它在边界吗？<br/>handler, main loop, accept()"}
    BOUNDARY -->|是 | ASYNC_SHELL["async fn —— 这是 shell"]
    BOUNDARY -->|否 | CORE_IO{"I/O 是核心逻辑吗？<br/>fan-out, streaming, stateful conn"}
    CORE_IO -->|是 | ASYNC_CORE["async fn —— 合理"]
    CORE_IO -->|否 | EXTRACT["将逻辑提取到 sync fn。<br/>将 I/O 结果作为参数传入。"]

    style SYNC fill:#d4efdf,stroke:#27ae60,color:#000
    style ASYNC_SHELL fill:#e8f4f8,stroke:#2980b9,color:#000
    style ASYNC_CORE fill:#e8f4f8,stroke:#2980b9,color:#000
    style EXTRACT fill:#d4efdf,stroke:#27ae60,color:#000
```

> **经验法则：** 从 sync 开始。只在最外层的 I/O 边界添加 async。只有当你能阐明*哪些并发 I/O 操作*证明复杂度税合理时，才将其向内拉。

---

<details>
<summary><strong>🏋️ 练习：提取 Sync Core</strong>（点击展开）</summary>

以下 axum handler 有 async 污染——业务逻辑与 I/O 混合。将它重构为 sync core 模块和薄 async shell。

```rust
use axum::{Json, extract::Path};

async fn get_device_report(Path(device_id): Path<String>) -> Result<Json<Report>, AppError> {
    // 通过 HTTP 从设备获取原始遥测
    let raw = reqwest::get(format!("http://bmc-{device_id}/telemetry"))
        .await?
        .json::<RawTelemetry>()
        .await?;

    // 业务逻辑：将原始传感器读数转换为校准值
    let mut readings = Vec::new();
    for sensor in &raw.sensors {
        let calibrated = (sensor.raw_value as f64) * sensor.scale + sensor.offset;
        if calibrated < sensor.min_valid || calibrated > sensor.max_valid {
            return Err(AppError::SensorOutOfRange {
                name: sensor.name.clone(),
                value: calibrated,
            });
        }
        readings.push(CalibratedReading {
            name: sensor.name.clone(),
            value: calibrated,
            unit: sensor.unit.clone(),
        });
    }

    // 业务逻辑：分类设备健康状态
    let critical_count = readings.iter()
        .filter(|r| r.value > 90.0)
        .count();
    let health = if critical_count > 2 { Health::Critical }
                 else if critical_count > 0 { Health::Warning }
                 else { Health::Ok };

    // 从库存服务获取设备元数据
    let meta = reqwest::get(format!("http://inventory/devices/{device_id}"))
        .await?
        .json::<DeviceMetadata>()
        .await?;

    Ok(Json(Report {
        device_id,
        device_name: meta.name,
        health,
        readings,
        timestamp: chrono::Utc::now(),
    }))
}
```

**你的目标：**

1. 创建 `core.rs` 带有 sync 函数：`calibrate_sensors`、`classify_health` 和 `build_report`
2. 创建 `shell.rs` 带有薄 async handler，获取后调用 sync core
3. 编写 `#[test]`（不是 `#[tokio::test]`）用于：传感器超出范围、健康分类阈值和正常报告

**提示：**
- Sync core 应该将 `RawTelemetry` 和 `DeviceMetadata` 作为输入——它永远不应该知道这些来自 HTTP。
- 你需要定义小型测试辅助函数（例如 `raw_telemetry()`、`sensor()`、`reading()`、`device_meta()`）来构造测试夹具。它们的签名应该从用法中显而易见。

<details>
<summary>🔑 解答</summary>

```rust
// core.rs — 零 async 依赖

pub fn calibrate_sensors(raw: &RawTelemetry) -> Result<Vec<CalibratedReading>, AppError> {
    raw.sensors.iter().map(|sensor| {
        let calibrated = (sensor.raw_value as f64) * sensor.scale + sensor.offset;
        if calibrated < sensor.min_valid || calibrated > sensor.max_valid {
            return Err(AppError::SensorOutOfRange {
                name: sensor.name.clone(),
                value: calibrated,
            });
        }
        Ok(CalibratedReading {
            name: sensor.name.clone(),
            value: calibrated,
            unit: sensor.unit.clone(),
        })
    }).collect()
}

pub fn classify_health(readings: &[CalibratedReading]) -> Health {
    let critical_count = readings.iter()
        .filter(|r| r.value > 90.0)
        .count();
    if critical_count > 2 { Health::Critical }
    else if critical_count > 0 { Health::Warning }
    else { Health::Ok }
}

pub fn build_report(
    device_id: String,
    readings: Vec<CalibratedReading>,
    meta: &DeviceMetadata,
) -> Report {
    Report {
        device_id,
        device_name: meta.name.clone(),
        health: classify_health(&readings),
        readings,
        timestamp: chrono::Utc::now(),
    }
}
```

```rust
// shell.rs — 只 async 边界

pub async fn get_device_report(
    Path(device_id): Path<String>,
) -> Result<Json<Report>, AppError> {
    let raw = reqwest::get(format!("http://bmc-{device_id}/telemetry"))
        .await?
        .json::<RawTelemetry>()
        .await?;

    let readings = core::calibrate_sensors(&raw)?;

    let meta = reqwest::get(format!("http://inventory/devices/{device_id}"))
        .await?
        .json::<DeviceMetadata>()
        .await?;

    Ok(Json(core::build_report(device_id, readings, &meta)))
}
```

```rust
// core_tests.rs — 不需要 runtime

// 测试夹具辅助函数——构造数据而无任何 I/O
fn sensor(name: &str, raw_value: f64, valid_range: std::ops::Range<f64>) -> RawSensor {
    RawSensor {
        name: name.into(),
        raw_value,
        scale: 1.0,
        offset: 0.0,
        min_valid: valid_range.start,
        max_valid: valid_range.end,
        unit: "unit".into(),
    }
}

fn raw_telemetry(sensors: Vec<RawSensor>) -> RawTelemetry {
    RawTelemetry { sensors }
}

fn reading(name: &str, value: f64) -> CalibratedReading {
    CalibratedReading { name: name.into(), value, unit: "unit".into() }
}

fn device_meta(name: &str) -> DeviceMetadata {
    DeviceMetadata { name: name.into() }
}

#[test]
fn sensor_out_of_range_rejected() {
    let raw = raw_telemetry(vec![sensor("gpu_temp", 105.0, 0.0..100.0)]);
    let result = core::calibrate_sensors(&raw);
    assert!(matches!(result, Err(AppError::SensorOutOfRange { .. })));
}

#[test]
fn health_classification() {
    let readings = vec![
        reading("a", 50.0),  // ok
        reading("b", 95.0),  // critical
        reading("c", 91.0),  // critical
        reading("d", 92.0),  // critical
    ];
    assert_eq!(core::classify_health(&readings), Health::Critical);
}

#[test]
fn normal_report() {
    let raw = raw_telemetry(vec![sensor("fan_rpm", 3000.0, 0.0..10000.0)]);
    let readings = core::calibrate_sensors(&raw).unwrap();
    let meta = device_meta("gpu-node-42");
    let report = core::build_report("dev-1".into(), readings, &meta);
    assert_eq!(report.health, Health::Ok);
    assert_eq!(report.readings.len(), 1);
}
```

**改变了什么：** Async handler 从 30 行混合逻辑和 I/O 变为 8 行纯编排。业务规则（校准数学、范围验证、健康阈值）现在用 `#[test]` 测试，在几毫秒内运行，对 tokio、reqwest 或任何 HTTP mock 服务器零依赖。

</details>
</details>

---

> **关键要点：**
>
> 1. Async 是**I/O 多路复用优化**，不是应用架构。大多数业务逻辑是 sync。
> 2. **Sync core, async shell：** 将业务规则保存在纯 sync 函数中，这些函数将 I/O 结果作为参数。Async shell 编排获取并调用核心。
> 3. 如果你在大块代码周围包装 `spawn_blocking`，**边界在错误的地方**——改为将逻辑重构为 sync 模块。
> 4. **库应该默认使用 sync API。** 一个 async 库强制所有调用者进入 runtime；一个 sync 库让调用者拥有 async 边界。
> 5. Async 在**fan-out/fan-in、streaming 和状态连接**中赚取其价值——这些情况下并发*就是*业务逻辑。
>
> **另见：** [Ch12 — 常见陷阱](ch12-common-pitfalls.md)（spawn_blocking 作为战术修复）· [Ch13 — 生产级模式](ch13-production-patterns.md)（背压、结构化并发）· [Ch17 — 综合项目：异步聊天服务器](ch17-capstone-project.md)（一个 async 是正确架构的案例）
