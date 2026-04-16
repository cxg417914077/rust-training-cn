# 理念——为什么类型击败测试 🟢

> **你将学到：** 三层编译时正确性（值、状态、协议），泛型函数签名如何充当编译器检查的保证，以及何时构造正确性模式值得（或不值得）投入。
>
> **交叉引用：** [ch02](ch02-typed-command-interfaces-request-determi.md)（类型化命令）、[ch05](ch05-protocol-state-machines-type-state-for-r.md)（type-state）、[ch13](ch13-reference-card.md)（参考卡）

## 运行时检查的代价

考虑诊断代码库中典型的运行时守卫：

```rust,ignore
fn read_sensor(sensor_type: &str, raw: &[u8]) -> f64 {
    match sensor_type {
        "temperature" => raw[0] as i8 as f64,          // 有符号字节
        "fan_speed"   => u16::from_le_bytes([raw[0], raw[1]]) as f64,
        "voltage"     => u16::from_le_bytes([raw[0], raw[1]]) as f64 / 1000.0,
        _             => panic!("unknown sensor type: {sensor_type}"),
    }
}
```

这个函数有**四种失败模式**编译器无法捕获：

1. 拼写错误：`"temperture"` → 运行时 panic
2. `raw` 长度错误：`fan_speed` 只有 1 字节 → 运行时 panic
3. 调用者将返回的 `f64` 当作 RPM 使用，而实际上是 °C → 逻辑 bug，静默失败
4. 添加了新传感器类型但未更新此 `match` → 运行时 panic

每种失败模式都是在**部署后**才发现的。测试有帮助，但它们只覆盖有人想到要写的情况。类型系统覆盖**所有**情况，包括没人想象到的情况。

## 三层正确性

### 第一层 —— 值正确性
**使无效值无法表示。**

```rust,ignore
// ❌ 任何 u16 都可以是"port"——0 无效但可以编译
fn connect(port: u16) { /* ... */ }

// ✅ 只有经过验证的 port 才能存在
pub struct Port(u16);  // 私有字段

impl TryFrom<u16> for Port {
    type Error = &'static str;
    fn try_from(v: u16) -> Result<Self, Self::Error> {
        if v > 0 { Ok(Port(v)) } else { Err("port must be > 0") }
    }
}

fn connect(port: Port) { /* ... */ }
// Port(0) 永远无法构造——不变量在任何地方都成立
```

**硬件示例：** `SensorId(u8)` —— 包装原始传感器编号，并验证它在 SDR 范围内。

### 第二层 —— 状态正确性
**使无效转换无法表示。**

```rust,ignore
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Socket<State> {
    fd: i32,
    _state: PhantomData<State>,
}

impl Socket<Disconnected> {
    fn connect(self, addr: &str) -> Socket<Connected> {
        // ... connect logic ...
        Socket { fd: self.fd, _state: PhantomData }
    }
}

impl Socket<Connected> {
    fn send(&mut self, data: &[u8]) { /* ... */ }
    fn disconnect(self) -> Socket<Disconnected> {
        Socket { fd: self.fd, _state: PhantomData }
    }
}

// Socket<Disconnected> 没有 send() 方法——如果尝试则会编译错误
```

**硬件示例：** GPIO 引脚模式——`Pin<Input>` 有 `read()` 但没有 `write()`。

### 第三层 —— 协议正确性
**使无效交互无法表示。**

```rust,ignore
use std::io;

trait IpmiCmd {
    type Response;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

// 为说明简化——完整 trait 见 ch02，包含
// net_fn()、cmd_byte()、payload() 和 parse_response()。

struct ReadTemp { sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        Ok(Celsius(raw[0] as i8 as f64))
    }
}

# #[derive(Debug)] struct Celsius(f64);

fn execute<C: IpmiCmd>(cmd: &C, raw: &[u8]) -> io::Result<C::Response> {
    cmd.parse_response(raw)
}
// ReadTemp 总是返回 Celsius——不会意外得到 Rpm
```

**硬件示例：** IPMI、Redfish、NVMe Admin 命令——请求类型决定响应类型。

## 类型作为编译器检查的保证

当你写下：

```rust,ignore
fn execute<C: IpmiCmd>(cmd: &C) -> io::Result<C::Response>
```

你不仅仅是在写一个函数——你是在声明一个**保证**："对于任何实现 `IpmiCmd` 的命令类型 `C`，执行它会产生精确的 `C::Response`。"编译器**验证**这个保证，每次构建你的代码时都会检查。如果类型不匹配，程序将无法编译。

这就是为什么 Rust 的类型系统如此强大——它不仅仅是在捕获错误，而是在**编译时强制执行正确性**。

## 何时不使用这些模式

构造正确性并不总是正确的选择：

| 情况 | 建议 |
|-----------|---------------|
| 安全关键边界（电源时序、加密） | ✅ 始终使用——这里的 bug 会熔化硬件或泄露机密 |
| 跨模块公共 API | ✅ 通常使用——误用应该是编译错误 |
| 有 3+ 个状态的状态机 | ✅ 通常使用——type-state 防止错误转换 |
| 一个 50 行函数内的内部辅助 | ❌ 过度——简单的 `assert!` 就足够了 |
| 原型设计 / 探索未知硬件 | ❌ 先用原始类型——理解行为后再精炼 |
| 面向用户的 CLI 解析 | ⚠️ 边界处用 `clap` + `TryFrom`，内部用原始类型即可 |

关键问题：**"如果这个 bug 在生产环境中发生，有多糟糕？"**

- 风扇停止 → GPU 熔化 → **使用类型**
- 错误的 DER 记录 → 客户获得错误数据 → **使用类型**
- 调试日志消息略有错误 → **使用 `assert!`**

## 关键要点

1. **三层正确性** —— 值（newtypes）、状态（type-state）、协议（关联类型）—— 每一层消除更广泛的 bug 类别。
2. **类型作为保证** —— 每个泛型函数签名都是编译器在每次构建时检查的契约。
3. **代价问题** —— "如果这个 bug 发布出去，有多糟糕？"决定类型还是测试是合适的工具。
4. **类型补充测试** —— 它们消除整个*类别*；测试覆盖特定的*值*和边界情况。
5. **知道何时停止** —— 内部辅助和临时原型很少需要类型级强制执行。

---
