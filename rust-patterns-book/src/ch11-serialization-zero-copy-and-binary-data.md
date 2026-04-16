# 11. Serialization, Zero-Copy, and Binary Data 🟡

> **你将学到：**
> - serde 基础：derive 宏、属性和枚举表示
> - 高性能读密集型工作负载的零拷贝反序列化
> - serde 格式生态系统（JSON、TOML、bincode、MessagePack）
> - 使用 `repr(C)`、zerocopy 和 `bytes::Bytes` 处理二进制数据

## serde 基础

`serde` (SERialize/DEserialize) 是 Rust 的通用序列化框架。
它将**数据模型**（你的结构体）与**格式**（JSON、TOML、二进制）分离：

```rust,ignore
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
struct ServerConfig {
    name: String,
    port: u16,
    #[serde(default)]                    // 如果缺失则使用 Default::default()
    max_connections: usize,
    #[serde(skip_serializing_if = "Option::is_none")]
    tls_cert_path: Option<String>,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 从 JSON 反序列化：
    let json_input = r#"{
        "name": "hw-diag",
        "port": 8080
    }"#;
    let config: ServerConfig = serde_json::from_str(json_input)?;
    println!("{config:?}");
    // ServerConfig { name: "hw-diag", port: 8080, max_connections: 0, tls_cert_path: None }

    // 序列化为 JSON：
    let output = serde_json::to_string_pretty(&config)?;
    println!("{output}");

    // 相同的结构体，不同的格式 —— 无需代码改动：
    let toml_input = r#"
        name = "hw-diag"
        port = 8080
    "#;
    let config: ServerConfig = toml::from_str(toml_input)?;
    println!("{config:?}");

    Ok(())
}
```

> **关键洞察**：你的结构体只需 derive 一次 `Serialize` 和 `Deserialize`。
> 然后它可以与*每个* serde 兼容的格式一起工作 —— JSON、TOML、YAML、
> bincode、MessagePack、CBOR、postcard 等几十种格式。

### 常见 serde 属性

serde 通过字段和容器属性提供细粒度控制：

```rust,ignore
use serde::{Serialize, Deserialize};

// --- 容器属性（在结构体/枚举上）---
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]       // JSON 约定：field_name → fieldName
#[serde(deny_unknown_fields)]            // 拒绝额外的键 —— 严格解析
struct DiagResult {
    test_name: String,                   // 序列化为 "testName"
    pass_count: u32,                     // 序列化为 "passCount"
    fail_count: u32,                     // 序列化为 "failCount"
}

// --- 字段属性 ---
#[derive(Serialize, Deserialize)]
struct Sensor {
    #[serde(rename = "sensor_id")]       // 覆盖序列化时的字段名
    id: u64,

    #[serde(default)]                    // 如果输入中缺失则使用 Default
    enabled: bool,

    #[serde(default = "default_threshold")]
    threshold: f64,

    #[serde(skip)]                       // 从不序列化或反序列化
    cached_value: Option<f64>,

    #[serde(skip_serializing_if = "Vec::is_empty")]
    tags: Vec<String>,

    #[serde(flatten)]                    // 内联嵌套结构体字段
    metadata: Metadata,

    #[serde(with = "hex_bytes")]         // 自定义序列化/反序列化模块
    raw_data: Vec<u8>,
}

fn default_threshold() -> f64 { 1.0 }

#[derive(Serialize, Deserialize)]
struct Metadata {
    vendor: String,
    model: String,
}
// 使用 #[serde(flatten)]，JSON 看起来像：
// { "sensor_id": 1, "vendor": "Intel", "model": "X200", ... }
// 而不是：{ "sensor_id": 1, "metadata": { "vendor": "Intel", ... } }
```

**最常用属性速查表**：

| 属性 | 级别 | 效果 |
|---|---|---|
| `rename_all = "camelCase"` | 容器 | 将所有字段重命名为 camelCase/snake_case/SCREAMING_SNAKE_CASE |
| `deny_unknown_fields` | 容器 | 遇到意外键时报错（严格模式） |
| `default` | 字段 | 字段缺失时使用 `Default::default()` |
| `rename = "..."` | 字段 | 自定义序列化名称 |
| `skip` | 字段 | 完全排除在序列化/反序列化之外 |
| `skip_serializing_if = "fn"` | 字段 | 条件排除（例如 `Option::is_none`） |
| `flatten` | 字段 | 内联嵌套结构体的字段 |
| `with = "module"` | 字段 | 使用自定义序列化/反序列化函数 |
| `alias = "..."` | 字段 | 反序列化时接受替代名称 |
| `deserialize_with = "fn"` | 字段 | 仅自定义反序列化函数 |
| `untagged` | 枚举 | 按顺序尝试每个变体（输出中无判别式） |

### 枚举表示

serde 为 JSON 等格式中的枚举提供四种表示：

```rust,ignore
use serde::{Serialize, Deserialize};

// 1. 外部标记（默认）：
#[derive(Serialize, Deserialize)]
enum Command {
    Reboot,
    RunDiag { test_name: String, timeout_secs: u64 },
    SetFanSpeed(u8),
}
// "Reboot"                                          → Command::Reboot
// {"RunDiag": {"test_name": "gpu", "timeout_secs": 60}}  → Command::RunDiag { ... }

// 2. 内部标记 —— #[serde(tag = "type")]：
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    Start { timestamp: u64 },
    Error { code: i32, message: String },
    End   { timestamp: u64, success: bool },
}
// {"type": "Start", "timestamp": 1706000000}
// {"type": "Error", "code": 42, "message": "timeout"}

// 3. 相邻标记 —— #[serde(tag = "t", content = "c")]：
#[derive(Serialize, Deserialize)]
#[serde(tag = "t", content = "c")]
enum Payload {
    Text(String),
    Binary(Vec<u8>),
}
// {"t": "Text", "c": "hello"}
// {"t": "Binary", "c": [0, 1, 2]}

// 4. 无标记 —— #[serde(untagged)]：
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum StringOrNumber {
    Str(String),
    Num(f64),
}
// "hello" → StringOrNumber::Str("hello")
// 42.0    → StringOrNumber::Num(42.0)
// ⚠️ 按顺序尝试 —— 第一个匹配的变体获胜
```

> **选择哪种表示**：对于大多数 JSON API 使用内部标记（`tag = "type"`）
> —— 它最易读，符合 Go、Python 和 TypeScript 的约定。仅当
> 形状本身足以区分时才使用无标记。

### 零拷贝反序列化

serde 可以在反序列化时不分配新字符串 —— 直接从输入缓冲区借用。
这是高性能解析的关键：

```rust,ignore
use serde::Deserialize;

// --- _owned_（分配）---
// 每个 String 字段将字节从输入复制到新的堆分配中。
#[derive(Deserialize)]
struct OwnedRecord {
    name: String,           // 分配新的 String
    value: String,          // 分配另一个 String
}

// --- 零拷贝（借用）---
// &'de str 字段直接从输入借用 —— 零分配。
#[derive(Deserialize)]
struct BorrowedRecord<'a> {
    name: &'a str,          // 指向输入缓冲区
    value: &'a str,         // 指向输入缓冲区
}

fn main() {
    let input = r#"{"name": "cpu_temp", "value": "72.5"}"#;

    // Owned：分配两个 String 对象
    let owned: OwnedRecord = serde_json::from_str(input).unwrap();

    // 零拷贝：`name` 和 `value` 指向 `input` —— 无分配
    let borrowed: BorrowedRecord = serde_json::from_str(input).unwrap();

    // 输出有生命周期绑定：borrowed 不能超出 input 存活
    println!("{}: {}", borrowed.name, borrowed.value);
}
```

**理解生命周期**：

```rust,ignore
// Deserialize<'de> —— 结构体可以从生命周期为 'de 的数据借用：
//   struct BorrowedRecord<'a> where 'a == 'de
//   仅当输入缓冲区活得足够长时才有效

// DeserializeOwned —— 结构体拥有所有数据，不借用：
//   trait DeserializeOwned: for<'de> Deserialize<'de> {}
//   适用于任何输入生命周期（结构体是独立的）

use serde::de::DeserializeOwned;

// 这个函数需要 owned 类型 —— 输入可以是临时的
fn parse_owned<T: DeserializeOwned>(input: &str) -> T {
    serde_json::from_str(input).unwrap()
}

// 这个函数允许借用 —— 更高效但限制了生命周期
fn parse_borrowed<'a, T: Deserialize<'a>>(input: &'a str) -> T {
    serde_json::from_str(input).unwrap()
}
```

**何时使用零拷贝**：
- 解析大文件且只需要少数几个字段时
- 高吞吐量管道（网络包、日志行）
- 当输入缓冲区已经活得足够长时（例如内存映射文件）

**何时不使用零拷贝**：
- 输入是短暂的（重用的网络读缓冲区）
- 需要在输入的生命周期之外存储结果
- 字段需要转换（转义、规范化）

> **实用技巧**：`Cow<'a, str>` 给你两全其美 —— 可能时借用，
> 必要时分配（例如 JSON 转义序列需要解转义时）。serde 原生支持 Cow。

### 格式生态系统

| 格式 | Crate | 人类可读 | 大小 | 速度 | 使用场景 |
|---|---|:---:|:---:|:---:|---|
| JSON | `serde_json` | ✅ | 大 | 好 | 配置文件、REST API、日志 |
| TOML | `toml` | ✅ | 中 | 好 | 配置文件（Cargo.toml 风格） |
| YAML | `serde_yaml` | ✅ | 中 | 好 | 配置文件（复杂嵌套） |
| bincode | `bincode` | ❌ | 小 | 快 | IPC、缓存、Rust 到 Rust |
| postcard | `postcard` | ❌ | 极小 | 极快 | 嵌入式系统、`no_std` |
| MessagePack | `rmp-serde` | ❌ | 小 | 快 | 跨语言二进制协议 |
| CBOR | `ciborium` | ❌ | 小 | 快 | IoT、受限环境 |

```rust
// 相同的结构体，多种格式 —— serde 的力量：

#[derive(serde::Serialize, serde::Deserialize, Debug)]
struct DiagConfig {
    name: String,
    tests: Vec<String>,
    timeout_secs: u64,
}

let config = DiagConfig {
    name: "accel_diag".into(),
    tests: vec!["memory".into(), "compute".into()],
    timeout_secs: 300,
};

// JSON:   {"name":"accel_diag","tests":["memory","compute"],"timeout_secs":300}
let json = serde_json::to_string(&config).unwrap();       // 67 字节

// bincode：紧凑二进制 —— 约 40 字节，无字段名
let bin = bincode::serialize(&config).unwrap();            // 更小

// postcard：更小，varint 编码 —— 适合嵌入式
// let post = postcard::to_allocvec(&config).unwrap();
```

> **选择你的格式**：
> - 人类编辑的配置文件 → TOML 或 JSON
> - Rust 到 Rust IPC/缓存 → bincode（快、紧凑、非跨语言）
> - 跨语言二进制 → MessagePack 或 CBOR
> - 嵌入式 / `no_std` → postcard

### 二进制数据和 repr(C)

对于硬件诊断，解析二进制协议数据很常见。Rust 提供
工具来安全、零拷贝地处理二进制数据：

```rust
// --- #[repr(C)]：可预测的内存布局 ---
// 确保字段按照 C 填充规则按声明顺序布局。
// 对于匹配硬件寄存器布局和协议头至关重要。

#[repr(C)]
#[derive(Debug, Clone, Copy)]
struct IpmiHeader {
    rs_addr: u8,
    net_fn_lun: u8,
    checksum: u8,
    rq_addr: u8,
    rq_seq_lun: u8,
    cmd: u8,
}

// --- 安全二进制解析与手动反序列化 ---
impl IpmiHeader {
    fn from_bytes(data: &[u8]) -> Option<Self> {
        if data.len() < size_of::<Self>() {
            return None;
        }
        Some(IpmiHeader {
            rs_addr:     data[0],
            net_fn_lun:  data[1],
            checksum:    data[2],
            rq_addr:     data[3],
            rq_seq_lun:  data[4],
            cmd:         data[5],
        })
    }

    fn net_fn(&self) -> u8 { self.net_fn_lun >> 2 }
    fn lun(&self)    -> u8 { self.net_fn_lun & 0x03 }
}

// --- 感知字节序的解析 ---
fn read_u16_le(data: &[u8], offset: usize) -> u16 {
    u16::from_le_bytes([data[offset], data[offset + 1]])
}

fn read_u32_be(data: &[u8], offset: usize) -> u32 {
    u32::from_be_bytes([
        data[offset], data[offset + 1],
        data[offset + 2], data[offset + 3],
    ])
}

// --- #[repr(C, packed)]：移除填充（对齐 = 1）---
#[repr(C, packed)]
#[derive(Debug, Clone, Copy)]
struct PcieCapabilityHeader {
    cap_id: u8,        // 能力 ID
    next_cap: u8,      // 指向下一个能力的指针
    cap_reg: u16,      // 特定于能力的寄存器
}
// ⚠️ 打包结构体：取 `&field` 会创建未对齐的引用 —— UB。
// 总是复制字段出来：let id = header.cap_id;  // OK (Copy)
// 绝不要这样做：let r = &header.cap_reg;               // 如果未对齐则 UB
```

### zerocopy 和 bytemuck —— 安全转换

与其使用 `unsafe` transmute，不如使用在编译时验证布局安全性的 crate：

```rust
// --- zerocopy：编译时检查的零拷贝转换 ---
// Cargo.toml: zerocopy = { version = "0.8", features = ["derive"] }

use zerocopy::{FromBytes, IntoBytes, KnownLayout, Immutable};

#[derive(FromBytes, IntoBytes, KnownLayout, Immutable, Debug)]
#[repr(C)]
struct SensorReading {
    sensor_id: u16,
    flags: u8,
    _reserved: u8,
    value: u32,     // 定点数：actual = value / 1000.0
}

fn parse_sensor(raw: &[u8]) -> Option<&SensorReading> {
    // 安全零拷贝：在编译时验证对齐和大小
    SensorReading::ref_from_bytes(raw).ok()
    // 返回指向 raw 内部的 &SensorReading —— 无拷贝、无分配
}

// --- bytemuck：简单、久经考验 ---
// Cargo.toml: bytemuck = { version = "1", features = ["derive"] }

use bytemuck::{Pod, Zeroable};

#[derive(Pod, Zeroable, Clone, Copy, Debug)]
#[repr(C)]
struct GpuRegister {
    address: u32,
    value: u32,
}

fn cast_registers(data: &[u8]) -> &[GpuRegister] {
    // 安全转换：Pod 保证所有位模式都有效
    bytemuck::cast_slice(data)
}
```

**何时使用哪一个**：

| 方法 | 安全性 | 开销 | 使用场景 |
|---|:---:|:---:|---|
| 手动逐字段解析 | ✅ 安全 | 复制字段 | 小结构体、复杂布局 |
| `zerocopy` | ✅ 安全 | 零拷贝 | 大缓冲区、多次读取、编译时检查 |
| `bytemuck` | ✅ 安全 | 零拷贝 | 简单 `Pod` 类型、转换切片 |
| `unsafe { transmute() }` | ❌ 不安全 | 零拷贝 | 最后手段 —— 避免在应用程序代码中使用 |

### bytes::Bytes —— 引用计数缓冲区

`bytes` crate（被 tokio、hyper、tonic 使用）提供零拷贝字节缓冲区
和引用计数 —— `Bytes` 对于 `Vec<u8>` 就像 `Arc<[u8]>` 对于 owned 切片：

```rust
use bytes::{Bytes, BytesMut, Buf, BufMut};

fn main() {
    // --- BytesMut：用于构建数据的可变缓冲区 ---
    let mut buf = BytesMut::with_capacity(1024);
    buf.put_u8(0x01);                    // 写入一个字节
    buf.put_u16(0x1234);                 // 写入 u16（大端）
    buf.put_slice(b"hello");             // 写入原始字节
    buf.put(&b"world"[..]);              // 从切片写入

    // 冻结为不可变 Bytes（零成本）：
    let data: Bytes = buf.freeze();

    // --- Bytes：不可变、引用计数、可克隆 ---
    let data2 = data.clone();            // 便宜：增加引用计数，非深拷贝
    let slice = data.slice(3..8);        // 零拷贝子切片（共享缓冲区）

    // 使用 Buf trait 从 Bytes 读取：
    let mut reader = &data[..];
    let byte = reader.get_u8();          // 0x01
    let short = reader.get_u16();        // 0x1234

    // 不复制地分割：
    let mut original = Bytes::from_static(b"HEADER\x00PAYLOAD");
    let header = original.split_to(6);   // header = "HEADER", original = "\x00PAYLOAD"

    println!("header: {:?}", &header[..]);
    println!("payload: {:?}", &original[1..]);
}
```

**`bytes` vs `Vec<u8>`**：

| 特性 | `Vec<u8>` | `Bytes` |
|---|---|---|
| Clone 成本 | O(n) 深拷贝 | O(1) 引用计数增加 |
| 子切片 | 带生命周期的借用 | 所有的、引用计数追踪 |
| 线程安全 | 非 `Sync`（需要 `Arc`） | 内置 `Send + Sync` |
| 可变性 | 直接 `&mut` | 先分割为 `BytesMut` |
| 生态系统 | 标准库 | tokio、hyper、tonic、axum |

> **何时使用 bytes**：网络协议、包解析、任何你需要接收缓冲区
> 并将其分割为由不同组件或线程处理的部分的场景。
> 零拷贝分割是杀手级特性。

> **关键要点 —— 序列化和二进制数据**
> - serde 的 derive 宏处理 90% 的情况；对其余情况使用属性（`rename`、`skip`、`default`）
> - 零拷贝反序列化（结构体中的 `&'a str`）避免读密集型工作负载的分配
> - `repr(C)` + `zerocopy`/`bytemuck` 用于硬件寄存器布局；`bytes::Bytes` 用于引用计数缓冲区

> **另见：** [第 9 章 — 错误处理](ch10-error-handling-patterns.md) 了解如何将 serde 错误与 `thiserror` 组合。[第 11 章 — Unsafe](ch12-unsafe-rust-controlled-danger.md) 了解 `repr(C)` 和 FFI 数据布局。

```mermaid
flowchart LR
    subgraph Input
        JSON["JSON"]
        TOML["TOML"]
        Bin["bincode"]
        MsgP["MessagePack"]
    end

    subgraph serde["serde data model"]
        Ser["Serialize"]
        De["Deserialize"]
    end

    subgraph Output
        Struct["Rust struct"]
        Enum["Rust enum"]
    end

    JSON --> De
    TOML --> De
    Bin --> De
    MsgP --> De
    De --> Struct
    De --> Enum
    Struct --> Ser
    Enum --> Ser
    Ser --> JSON
    Ser --> Bin

    style JSON fill:#e8f4f8,stroke:#2980b9,color:#000
    style TOML fill:#e8f4f8,stroke:#2980b9,color:#000
    style Bin fill:#e8f4f8,stroke:#2980b9,color:#000
    style MsgP fill:#e8f4f8,stroke:#2980b9,color:#000
    style Ser fill:#fef9e7,stroke:#f1c40f,color:#000
    style De fill:#fef9e7,stroke:#f1c40f,color:#000
    style Struct fill:#d4efdf,stroke:#27ae60,color:#000
    style Enum fill:#d4efdf,stroke:#27ae60,color:#000
```

---

### 练习：自定义 serde 反序列化 ★★★（约 45 分钟）

设计一个 `HumanDuration` 包装器，使用自定义 serde 反序列化器从人类可读的字符串如 `"30s"`、`"5m"`、`"2h"` 反序列化。它也应该序列化为相同的格式。

<details>
<summary>🔑 解答</summary>

```rust,ignore
use serde::{Deserialize, Deserializer, Serialize, Serializer};
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
struct HumanDuration(std::time::Duration);

impl HumanDuration {
    fn from_str(s: &str) -> Result<Self, String> {
        let s = s.trim();
        if s.is_empty() { return Err("empty duration string".into()); }

        let (num_str, suffix) = s.split_at(
            s.find(|c: char| !c.is_ascii_digit()).unwrap_or(s.len())
        );
        let value: u64 = num_str.parse()
            .map_err(|_| format!("invalid number: {num_str}"))?;

        let duration = match suffix {
            "s" | "sec"  => std::time::Duration::from_secs(value),
            "m" | "min"  => std::time::Duration::from_secs(value * 60),
            "h" | "hr"   => std::time::Duration::from_secs(value * 3600),
            "ms"         => std::time::Duration::from_millis(value),
            other        => return Err(format!("unknown suffix: {other}")),
        };
        Ok(HumanDuration(duration))
    }
}

impl fmt::Display for HumanDuration {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let secs = self.0.as_secs();
        if secs == 0 {
            write!(f, "{}ms", self.0.as_millis())
        } else if secs % 3600 == 0 {
            write!(f, "{}h", secs / 3600)
        } else if secs % 60 == 0 {
            write!(f, "{}m", secs / 60)
        } else {
            write!(f, "{}s", secs)
        }
    }
}

impl Serialize for HumanDuration {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        serializer.serialize_str(&self.to_string())
    }
}

impl<'de> Deserialize<'de> for HumanDuration {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        let s = String::deserialize(deserializer)?;
        HumanDuration::from_str(&s).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    timeout: HumanDuration,
    retry_interval: HumanDuration,
}

fn main() {
    let json = r#"{ "timeout": "30s", "retry_interval": "5m" }"#;
    let config: Config = serde_json::from_str(json).unwrap();

    assert_eq!(config.timeout.0, std::time::Duration::from_secs(30));
    assert_eq!(config.retry_interval.0, std::time::Duration::from_secs(300));

    let serialized = serde_json::to_string(&config).unwrap();
    assert!(serialized.contains("30s"));
    println!("Config: {serialized}");
}
```

</details>

***
