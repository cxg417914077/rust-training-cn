# 参考卡片

> **14+ 种正确构造模式的快速参考**，包含选择流程图、模式目录、组合规则、crate 映射和类型即保证速查表。
>
> **交叉引用：** 每一章——这是整本书的查找表。

## 快速参考：正确构造模式

### 模式选择指南

```text
bug 如果被遗漏是否是灾难性的？
├── 是 → 能编码到类型中吗？
│         ├── 是 → 使用正确构造
│         └── 否  → 运行时检查 + 广泛测试
└── 否  → 运行时检查没问题
```

### 模式目录

| # | 模式 | 关键 Trait/类型 | 防止 | 运行时成本 | 章节 |
|---|---------|---------------|----------|:------:|---------|
| 1 | 类型化命令 | `trait IpmiCmd { type Response; }` | 错误的响应类型 | 零 | ch02 |
| 2 | 一次性类型 | `struct Nonce`（非 Clone/Copy） | Nonce/密钥重用 | 零 | ch03 |
| 3 | Capability 令牌 | `struct AdminToken { _private: () }` | 未授权访问 | 零 | ch04 |
| 4 | Type-State | `Session<Active>` | 协议违规 | 零 | ch05 |
| 5 | 量纲类型 | `struct Celsius(f64)` | 单位混淆 | 零 | ch06 |
| 6 | 验证边界 | `struct ValidFru`（通过 TryFrom） | 使用未验证数据 | 解析一次 | ch07 |
| 7 | Capability 混合 | `trait FanDiagMixin: HasSpi + HasI2c` | 缺少总线访问 | 零 | ch08 |
| 8 | Phantom 类型 | `Register<Width16>` | 宽度/方向不匹配 | 零 | ch09 |
| 9 | Sentinel → Option | `Option<u8>`（而非 `0xFF`） | Sentinel 作为值的 bug | 零 | ch11 |
| 10 | Sealed trait | `trait Cmd: private::Sealed` | 不健全的外部实现 | 零 | ch11 |
| 11 | 非穷举枚举 | `#[non_exhaustive] enum Sku` | 静默的 match 回退 | 零 | ch11 |
| 12 | Typestate 构建器 | `DerBuilder<Set, Missing>` | 不完整构造 | 零 | ch11 |
| 13 | FromStr 验证 | `impl FromStr for DiagLevel` | 未验证的字符串输入 | 解析一次 | ch11 |
| 14 | Const 泛型大小 | `RegisterBank<const N: usize>` | 缓冲区大小不匹配 | 零 | ch11 |
| 15 | 安全 `unsafe` 包装 | `MmioRegion::read_u32()` | 未检查的 MMIO/FFI | 零 | ch11 |
| 16 | 异步 Type-State | `AsyncSession<Active>` | 异步协议违规 | 零 | ch11 |
| 17 | Const 断言 | `SdrSensorId<const N: u8>` | 无效的编译时 ID | 零 | ch11 |
| 18 | 会话类型 | `Chan<SendRequest>` | 无序的通道操作 | 零 | ch11 |
| 19 | Pin 自引用 | `Pin<Box<StreamParser>>` | 悬垂的结构体内指针 | 零 | ch11 |
| 20 | RAII / Drop | `impl Drop for Session` | 任何退出路径上的资源泄漏 | 零 | ch11 |
| 21 | 错误类型层次 | `#[derive(Error)] enum DiagError` | 静默的错误吞没 | 零 | ch11 |
| 22 | `#[must_use]` | `#[must_use] struct Token` | 静默丢弃的值 | 零 | ch11 |

### 组合规则

```text
Capability 令牌 + Type-State = 授权的状態转换
类型化命令 + 量纲类型 = 物理类型的响应
验证边界 + Phantom 类型 = 已验证配置上的类型化寄存器访问
Capability 混合 + 类型化命令 = 感知总线的类型化操作
一次性类型 + Type-State = 消费即转换的协议
Sealed trait + 类型化命令 = 封闭、健全的命令集
Sentinel → Option + 验证边界 = 干净的解析一次流水线
Typestate 构建器 + Capability 令牌 = 构造完成的证明
FromStr + #[non_exhaustive] = 可演进的、快速失败的枚举解析
Const 泛型大小 + 验证边界 = 定型的、已验证的协议缓冲区
Safe unsafe 包装 + Phantom 类型 = 类型化的、安全的 MMIO 访问
异步 Type-State + Capability 令牌 = 授权的异步转换
会话类型 + 类型化命令 = 完全类型化的请求 - 响应通道
Pin + Type-State = 不能移动的自引用状态机
RAII (Drop) + Type-State = 依赖状态的清理保证
错误层次 + 验证边界 = 类型化的解析错误，带有穷举处理
#[must_use] + 一次性类型 = 难以忽略、难以重用的令牌
```

### 要避免的反模式

| 反模式 | 为什么错误 | 正确的替代方案 |
|-------------|---------------|-------------------|
| `fn read_sensor() -> f64` | 无单位——可能是°C、°F 或 RPM | `fn read_sensor() -> Celsius` |
| `fn encrypt(nonce: &[u8; 12])` | Nonce 可重用（借用） | `fn encrypt(nonce: Nonce)`（move） |
| `fn admin_op(is_admin: bool)` | 调用者可以撒谎（`true`） | `fn admin_op(_: &AdminToken)` |
| `fn send(session: &Session)` | 没有状态保证 | `fn send(session: &Session<Active>)` |
| `fn process(data: &[u8])` | 未验证 | `fn process(data: &ValidFru)` |
| 在临时密钥上使用 `Clone` | 破坏一次性保证 | 不要派生 Clone |
| `let vendor_id: u16 = 0xFFFF` | Sentinel 在内部携带 | `let vendor_id: Option<u16> = None` |
| `fn route(level: &str)` 带有回退 | 拼写错误静默默认 | `let level: DiagLevel = s.parse()?` |
| `Builder::new().finish()` 没有字段 | 构造了不完整的对象 | Typestate 构建器：`finish()` 需要 `Set` |
| `let buf: Vec<u8>` 用于固定大小的硬件缓冲区 | 大小仅在运行时检查 | `RegisterBank<4096>`（const 泛型） |
| 分散的原始 `unsafe { ptr::read(...) }` | UB 风险，无法审计 | `MmioRegion::read_u32()` 安全包装 |
| `async fn transition(&mut self)` | 可变借用不强制状态 | `async fn transition(self) -> NextState` |
| `fn cleanup()` 手动调用 | 在提前返回/panic 时忘记 | `impl Drop` —— 编译器插入调用 |
| `fn op() -> Result<T, String>` | 不透明的错误，没有变体匹配 | `fn op() -> Result<T, DiagError>` 枚举 |

### 映射到诊断代码库

| 模块 | 适用模式 |
|---------------------|----------------------|
| `protocol_lib` | 类型化命令、type-state 会话 |
| `thermal_diag` | Capability 混合、量纲类型 |
| `accel_diag` | 验证边界、phantom 寄存器 |
| `network_diag` | Type-State（链路训练）、capability 令牌 |
| `pci_topology` | Phantom 类型（寄存器宽度）、已验证配置、sentinel → Option |
| `event_handler` | 一次性审计令牌、capability 令牌、FromStr（Component） |
| `event_log` | 验证边界（SEL 记录解析） |
| `compute_diag` | 量纲类型（温度、频率） |
| `memory_diag` | 验证边界（SPD 数据）、量纲类型 |
| `switch_diag` | Type-State（端口枚举）、phantom 类型 |
| `config_loader` | FromStr（DiagLevel、FaultStatus、DiagAction） |
| `log_analyzer` | 验证边界（CompiledPatterns） |
| `diag_framework` | Typestate 构建器（DerBuilder）、会话类型（orchestrator↔worker） |
| `topology_lib` | Const 泛型寄存器银行、安全 MMIO 包装 |

### 类型即保证——快速映射

| 保证 | Rust 等价物 | 示例 |
|-----------|----------------|---------|
| "此证明存在" | 一个类型 | `AdminToken` |
| "我有证明" | 该类型的值 | `let tok = authenticate()?;` |
| "A 意味着 B" | 函数 `fn(A) -> B` | `fn activate(AdminToken) -> Session<Active>` |
| "A 和 B 都" | 元组 `(A, B)` 或多参数 | `fn op(a: &AdminToken, b: &LinkTrained)` |
| "A 或 B" | `enum { A(A), B(B) }` 或 `Result<A, B>` | `Result<Session<Active>, Error>` |
| "总是真" | `()`（单元类型） | 总是可构造 |
| "不可能" | `!`（never 类型）或 `enum Void {}` | 永远无法构造 |

---
