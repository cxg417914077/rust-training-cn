# 类型化命令接口——请求决定响应 🟡

> **你将学到：** 命令 trait 上的关联类型如何在请求和响应之间创建编译时绑定，消除 IPMI、Redfish 和 NVMe 协议中的不匹配解析、单位混淆和静默类型强制。
>
> **交叉引用：** [ch01](ch01-the-philosophy-why-types-beat-tests.md)（理念）、[ch06](ch06-dimensional-analysis-making-the-compiler.md)（量纲类型）、[ch07](ch07-validated-boundaries-parse-dont-validate.md)（有效性边界）、[ch10](ch10-putting-it-all-together-a-complete-diagn.md)（集成）

## 无类型泥潭

大多数硬件管理栈——IPMI、Redfish、NVMe Admin、PLDM——开始时都是`原始字节进 → 原始字节出`。这创造了一类测试只能部分发现的 bug：

```rust,ignore
use std::io;

struct BmcRaw { /* ipmitool handle */ }

impl BmcRaw {
    fn raw_command(&self, net_fn: u8, cmd: u8, data: &[u8]) -> io::Result<Vec<u8>> {
        // ... shells out to ipmitool ...
        Ok(vec![0x00, 0x19, 0x00]) // stub
    }
}

fn diagnose_thermal(bmc: &BmcRaw) -> io::Result<()> {
    let raw = bmc.raw_command(0x04, 0x2D, &[0x20])?;
    let cpu_temp = raw[0] as f64;        // 🤞 字节 0 是读数吗？

    let raw = bmc.raw_command(0x04, 0x2D, &[0x30])?;
    let fan_rpm = raw[0] as u32;         // 🐛 风扇速度是 2 字节 LE

    let raw = bmc.raw_command(0x04, 0x2D, &[0x40])?;
    let voltage = raw[0] as f64;         // 🐛 需要除以 1000

    if cpu_temp > fan_rpm as f64 {       // 🐛 比较 °C 和 RPM
        println!("uh oh");
    }

    log_temp(voltage);                   // 🐛 传递伏特作为温度
    Ok(())
}

fn log_temp(t: f64) { println!("Temp: {t}°C"); }
```

| # | Bug | 发现时间 |
|---|-----|----------|
| 1 | 风扇 RPM 解析为 1 字节而非 2 | 生产环境，凌晨 3 点 |
| 2 | 电压未缩放 | 所有 PSU 标记为过压 |
| 3 | 比较 °C 和 RPM | 可能永远不会 |
| 4 | 伏特传递给温度日志 | 6 个月后，读取历史数据时 |

**根本原因：** 一切都是 `Vec<u8>` → `f64` → 祈祷。

## 类型化命令模式

### 步骤 1 —— 领域 newtypes

```rust,ignore
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Rpm(pub u32);  // u32: 原始 IPMI 传感器值（整数 RPM）

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Volts(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Watts(pub f64);
```

> **关于 `Rpm(u32)` vs `Rpm(f64)` 的说明：** 本章中内部类型是 `u32`，因为 IPMI 传感器读数是整数值。在 ch06（量纲分析）中，`Rpm` 使用 `f64` 以支持算术运算（平均、缩放）。两者都有效——无论内部类型如何，newtype 都能防止跨单位混淆。

### 步骤 2 —— 命令 trait（类型索引分发）

关联类型 `Response` 是关键——它将每个命令结构绑定到其返回类型。每个实现结构将 `Response` 固定到特定的领域类型，因此 `execute()` 总是返回精确的正确类型：

```rust,ignore
pub trait IpmiCmd {
    /// "类型索引" —— 决定 execute() 返回什么。
    type Response;

    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;

    /// 解析封装在这里——每个命令知道自己的字节布局。
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}
```

### 步骤 3 —— 每个命令一个结构

```rust,ignore
pub struct ReadTemp { pub sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        if raw.is_empty() {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "empty response"));
        }
        // 注意：ch01 的无类型示例使用 `raw[0] as i8 as f64`（有符号）
        // 因为该函数演示没有 SDR 元数据的通用解析。这里使用无符号（`as f64`）
        // 因为 IPMI 规范 §35.5 中的 SDR 线性化公式将无符号原始读数转换为校准值。
        // 在生产中，应用完整的 SDR 公式：result = (M × raw + B) × 10^(R_exp)。
        Ok(Celsius(raw[0] as f64))  // 无符号原始字节，根据 SDR 公式转换
    }
}

pub struct ReadFanSpeed { pub fan_id: u8 }
impl IpmiCmd for ReadFanSpeed {
    type Response = Rpm;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.fan_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Rpm> {
        if raw.len() < 2 {
            return Err(io::Error::new(io::ErrorKind::InvalidData,
                format!("fan speed needs 2 bytes, got {}", raw.len())));
        }
        Ok(Rpm(u16::from_le_bytes([raw[0], raw[1]]) as u32))
    }
}

pub struct ReadVoltage { pub rail: u8 }
impl IpmiCmd for ReadVoltage {
    type Response = Volts;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.rail] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Volts> {
        if raw.len() < 2 {
            return Err(io::Error::new(io::ErrorKind::InvalidData,
                format!("voltage needs 2 bytes, got {}", raw.len())));
        }
        Ok(Volts(u16::from_le_bytes([raw[0], raw[1]]) as f64 / 1000.0))
    }
}
```

### 步骤 4 —— 执行器（零 `dyn`，单态化）

```rust,ignore
pub struct BmcConnection { pub timeout_secs: u32 }

impl BmcConnection {
    pub fn execute<C: IpmiCmd>(&self, cmd: &C) -> io::Result<C::Response> {
        let raw = self.raw_send(cmd.net_fn(), cmd.cmd_byte(), &cmd.payload())?;
        cmd.parse_response(&raw)
    }

    fn raw_send(&self, _nf: u8, _cmd: u8, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![0x19, 0x00]) // stub
    }
}
```

### 步骤 5 —— 所有四个 bug 变成编译错误

```rust,ignore
fn diagnose_thermal_typed(bmc: &BmcConnection) -> io::Result<()> {
    let cpu_temp: Celsius = bmc.execute(&ReadTemp { sensor_id: 0x20 })?;
    let fan_rpm:  Rpm     = bmc.execute(&ReadFanSpeed { fan_id: 0x30 })?;
    let voltage:  Volts   = bmc.execute(&ReadVoltage { rail: 0x40 })?;

    // Bug #1 —— 不可能：解析逻辑在 ReadFanSpeed::parse_response 中
    // Bug #2 —— 不可能：单位缩放在 ReadVoltage::parse_response 中

    // Bug #3 —— 编译错误：
    // if cpu_temp > fan_rpm { }
    //    ^^^^^^^^   ^^^^^^^ Celsius vs Rpm → "类型不匹配" ❌

    // Bug #4 —— 编译错误：
    // log_temperature(voltage);
    //                 ^^^^^^^ Volts, 期望 Celsius ❌

    if cpu_temp > Celsius(85.0) { println!("CPU overheating: {:?}", cpu_temp); }
    if fan_rpm < Rpm(4000)      { println!("Fan too slow: {:?}", fan_rpm); }

    Ok(())
}

fn log_temperature(t: Celsius) { println!("Temp: {:?}", t); }
fn log_voltage(v: Volts)       { println!("Voltage: {:?}", v); }
```

## IPMI：无法混淆的传感器读数

添加新传感器是一个结构 + 一个实现——没有分散的解析：

```rust,ignore
pub struct ReadPowerDraw { pub domain: u8 }
impl IpmiCmd for ReadPowerDraw {
    type Response = Watts;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.domain] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Watts> {
        if raw.len() < 2 {
            return Err(io::Error::new(io::ErrorKind::InvalidData,
                format!("power draw needs 2 bytes, got {}", raw.len())));
        }
        Ok(Watts(u16::from_le_bytes([raw[0], raw[1]]) as f64))
    }
}

// 任何使用 bmc.execute(&ReadPowerDraw { domain: 0 }) 的代码
// 自动获得 Watts 返回——其他地方没有解析代码
```

### 隔离测试每个命令

```rust,ignore
#[cfg(test)]
mod tests {
    use super::*;

    struct StubBmc {
        responses: std::collections::HashMap<u8, Vec<u8>>,
    }

    impl StubBmc {
        fn execute<C: IpmiCmd>(&self, cmd: &C) -> io::Result<C::Response> {
            let key = cmd.payload()[0];
            let raw = self.responses.get(&key)
                .ok_or_else(|| io::Error::new(io::ErrorKind::NotFound, "no stub"))?;
            cmd.parse_response(raw)
        }
    }

    #[test]
    fn read_temp_parses_raw_byte() {
        let bmc = StubBmc {
            responses: [(0x20, vec![0x19])].into(), // 25 decimal = 0x19
        };
        let temp = bmc.execute(&ReadTemp { sensor_id: 0x20 }).unwrap();
        assert_eq!(temp, Celsius(25.0));
    }

    #[test]
    fn read_fan_parses_two_byte_le() {
        let bmc = StubBmc {
            responses: [(0x30, vec![0x00, 0x19])].into(), // 0x1900 = 6400
        };
        let rpm = bmc.execute(&ReadFanSpeed { fan_id: 0x30 }).unwrap();
        assert_eq!(rpm, Rpm(6400));
    }

    #[test]
    fn read_voltage_scales_millivolts() {
        let bmc = StubBmc {
            responses: [(0x40, vec![0xE8, 0x2E])].into(), // 0x2EE8 = 12008 mV
        };
        let v = bmc.execute(&ReadVoltage { rail: 0x40 }).unwrap();
        assert!((v.0 - 12.008).abs() < 0.001);
    }
}
```

## Redfish：模式类型的 REST 端点

Redfish 甚至是更好的例子——每个端点返回 DMTF 定义的 JSON schema：

```rust,ignore
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct ThermalResponse {
    #[serde(rename = "Temperatures")]
    pub temperatures: Vec<RedfishTemp>,
    #[serde(rename = "Fans")]
    pub fans: Vec<RedfishFan>,
}

#[derive(Debug, Deserialize)]
pub struct RedfishTemp {
    #[serde(rename = "Name")]
    pub name: String,
    #[serde(rename = "ReadingCelsius")]
    pub reading: f64,
    #[serde(rename = "UpperThresholdCritical")]
    pub critical_hi: Option<f64>,
    #[serde(rename = "Status")]
    pub status: RedfishHealth,
}

#[derive(Debug, Deserialize)]
pub struct RedfishFan {
    #[serde(rename = "Name")]
    pub name: String,
    #[serde(rename = "Reading")]
    pub rpm: u32,
    #[serde(rename = "Status")]
    pub status: RedfishHealth,
}

#[derive(Debug, Deserialize)]
pub struct PowerResponse {
    #[serde(rename = "Voltages")]
    pub voltages: Vec<RedfishVoltage>,
    #[serde(rename = "PowerSupplies")]
    pub psus: Vec<RedfishPsu>,
}

#[derive(Debug, Deserialize)]
pub struct RedfishVoltage {
    #[serde(rename = "Name")]
    pub name: String,
    #[serde(rename = "ReadingVolts")]
    pub reading: f64,
    #[serde(rename = "Status")]
    pub status: RedfishHealth,
}

#[derive(Debug, Deserialize)]
pub struct RedfishPsu {
    #[serde(rename = "Name")]
    pub name: String,
    #[serde(rename = "PowerOutputWatts")]
    pub output_watts: Option<f64>,
    #[serde(rename = "Status")]
    pub status: RedfishHealth,
}

#[derive(Debug, Deserialize)]
pub struct ProcessorResponse {
    #[serde(rename = "Model")]
    pub model: String,
    #[serde(rename = "TotalCores")]
    pub cores: u32,
    #[serde(rename = "Status")]
    pub status: RedfishHealth,
}

#[derive(Debug, Deserialize)]
pub struct RedfishHealth {
    #[serde(rename = "State")]
    pub state: String,
    #[serde(rename = "Health")]
    pub health: Option<String>,
}

/// 类型化 Redfish 端点——每个知道自己的响应类型。
pub trait RedfishEndpoint {
    type Response: serde::de::DeserializeOwned;
    fn method(&self) -> &'static str;
    fn path(&self) -> String;
}

pub struct GetThermal { pub chassis_id: String }
impl RedfishEndpoint for GetThermal {
    type Response = ThermalResponse;
    fn method(&self) -> &'static str { "GET" }
    fn path(&self) -> String {
        format!("/redfish/v1/Chassis/{}/Thermal", self.chassis_id)
    }
}

pub struct GetPower { pub chassis_id: String }
impl RedfishEndpoint for GetPower {
    type Response = PowerResponse;
    fn method(&self) -> &'static str { "GET" }
    fn path(&self) -> String {
        format!("/redfish/v1/Chassis/{}/Power", self.chassis_id)
    }
}

pub struct GetProcessor { pub system_id: String, pub proc_id: String }
impl RedfishEndpoint for GetProcessor {
    type Response = ProcessorResponse;
    fn method(&self) -> &'static str { "GET" }
    fn path(&self) -> String {
        format!("/redfish/v1/Systems/{}/Processors/{}", self.system_id, self.proc_id)
    }
}

pub struct RedfishClient {
    pub base_url: String,
    pub auth_token: String,
}

impl RedfishClient {
    pub fn execute<E: RedfishEndpoint>(&self, endpoint: &E) -> io::Result<E::Response> {
        let url = format!("{}{}", self.base_url, endpoint.path());
        let json_bytes = self.http_request(endpoint.method(), &url)?;
        serde_json::from_slice(&json_bytes)
            .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))
    }

    fn http_request(&self, _method: &str, _url: &str) -> io::Result<Vec<u8>> {
        Ok(vec![]) // stub — 真实实现使用 reqwest/hyper
    }
}

// 用法——完全类型化，自文档化
fn redfish_pre_flight(client: &RedfishClient) -> io::Result<()> {
    let thermal: ThermalResponse = client.execute(&GetThermal {
        chassis_id: "1".into(),
    })?;
    let power: PowerResponse = client.execute(&GetPower {
        chassis_id: "1".into(),
    })?;

    // ❌ 编译错误——不能传递 PowerResponse 给 thermal 检查：
    // check_thermals(&power);  → "期望 ThermalResponse, 找到 PowerResponse"

    for temp in &thermal.temperatures {
        if let Some(crit) = temp.critical_hi {
            if temp.reading > crit {
                println!("CRITICAL: {} at {}°C (threshold: {}°C)",
                    temp.name, temp.reading, crit);
            }
        }
    }
    Ok(())
}
```

## NVMe Admin：Identify 不返回日志页

NVMe 管理命令遵循相同的形状。控制器区分命令操作码，但在 C 中调用者必须知道在 4 KB 完成缓冲区上覆盖哪个结构。类型化命令模式使这不可能出错：

```rust,ignore
use std::io;

/// NVMe Admin 命令 trait——与 IpmiCmd 形状相同。
pub trait NvmeAdminCmd {
    type Response;
    fn opcode(&self) -> u8;
    fn parse_completion(&self, data: &[u8]) -> io::Result<Self::Response>;
}

// ── Identify (opcode 0x06) ──

#[derive(Debug, Clone)]
pub struct IdentifyResponse {
    pub model_number: String,   // 字节 24–63
    pub serial_number: String,  // 字节 4–23
    pub firmware_rev: String,   // 字节 64–71
    pub total_capacity_gb: u64,
}

pub struct Identify {
    pub nsid: u32, // 0 = 控制器，>0 = 命名空间
}

impl NvmeAdminCmd for Identify {
    type Response = IdentifyResponse;
    fn opcode(&self) -> u8 { 0x06 }
    fn parse_completion(&self, data: &[u8]) -> io::Result<IdentifyResponse> {
        if data.len() < 4096 {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "short identify"));
        }
        Ok(IdentifyResponse {
            serial_number: String::from_utf8_lossy(&data[4..24]).trim().to_string(),
            model_number: String::from_utf8_lossy(&data[24..64]).trim().to_string(),
            firmware_rev: String::from_utf8_lossy(&data[64..72]).trim().to_string(),
            total_capacity_gb: u64::from_le_bytes(
                data[280..288].try_into().unwrap()
            ) / (1024 * 1024 * 1024),
        })
    }
}

// ── Get Log Page (opcode 0x02) ──

#[derive(Debug, Clone)]
pub struct SmartLog {
    pub critical_warning: u8,
    pub temperature_kelvin: u16,
    pub available_spare_pct: u8,
    pub data_units_read: u128,
}

pub struct GetLogPage {
    pub log_id: u8, // 0x02 = SMART/Health
}

impl NvmeAdminCmd for GetLogPage {
    type Response = SmartLog;
    fn opcode(&self) -> u8 { 0x02 }
    fn parse_completion(&self, data: &[u8]) -> io::Result<SmartLog> {
        if data.len() < 512 {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "short log page"));
        }
        Ok(SmartLog {
            critical_warning: data[0],
            temperature_kelvin: u16::from_le_bytes([data[1], data[2]]),
            available_spare_pct: data[3],
            data_units_read: u128::from_le_bytes(data[32..48].try_into().unwrap()),
        })
    }
}

// ── 执行器 ──

pub struct NvmeController { /* fd, BAR, etc. */ }

impl NvmeController {
    pub fn admin_cmd<C: NvmeAdminCmd>(&self, cmd: &C) -> io::Result<C::Response> {
        let raw = self.submit_and_wait(cmd.opcode())?;
        cmd.parse_completion(&raw)
    }

    fn submit_and_wait(&self, _opcode: u8) -> io::Result<Vec<u8>> {
        Ok(vec![0u8; 4096]) // stub — 真实实现发出 doorbell + 等待 CQ 条目
    }
}

// ── 用法 ──

fn nvme_health_check(ctrl: &NvmeController) -> io::Result<()> {
    let id: IdentifyResponse = ctrl.admin_cmd(&Identify { nsid: 0 })?;
    let smart: SmartLog = ctrl.admin_cmd(&GetLogPage { log_id: 0x02 })?;

    // ❌ 编译错误——Identify 返回 IdentifyResponse，不是 SmartLog：
    // let smart: SmartLog = ctrl.admin_cmd(&Identify { nsid: 0 })?;

    println!("{} (FW {}): {}°C, {}% spare",
        id.model_number, id.firmware_rev,
        smart.temperature_kelvin.saturating_sub(273),
        smart.available_spare_pct);

    Ok(())
}
```

这三种协议的进展现在遵循一个**渐进的弧线**（ch07 对有效性边界使用的相同技术）：

| 节拍 | 协议 | 复杂度 | 添加内容 |
|:----:|----------|-----------|----------|
| 1 | IPMI | 简单：传感器 ID → 读数 | 核心模式：`trait + 关联类型` |
| 2 | Redfish | REST：端点 → 类型化 JSON | Serde 集成，模式类型响应 |
| 3 | NVMe | 二进制：操作码 → 4 KB 结构覆盖 | 原始缓冲区解析，多结构完成数据 |

## 扩展：命令脚本的宏 DSL

```rust,ignore
/// 执行一系列类型化 IPMI 命令，返回结果元组。
macro_rules! diag_script {
    ($bmc:expr; $($cmd:expr),+ $(,)?) => {{
        ( $( $bmc.execute(&$cmd)?, )+ )
    }};
}

fn full_pre_flight(bmc: &BmcConnection) -> io::Result<()> {
    let (temp, rpm, volts) = diag_script!(bmc;
        ReadTemp     { sensor_id: 0x20 },
        ReadFanSpeed { fan_id:    0x30 },
        ReadVoltage  { rail:      0x40 },
    );
    // 类型：(Celsius, Rpm, Volts) —— 完全推断，交换 = 编译错误
    assert!(temp  < Celsius(95.0), "CPU 太热");
    assert!(rpm   > Rpm(3000),     "风扇太慢");
    assert!(volts > Volts(11.4),   "12V 轨下降");
    Ok(())
}
```

## 扩展：用于动态脚本的 Enum 分发

当命令在运行时来自 JSON 配置时：

```rust,ignore
pub enum AnyReading {
    Temp(Celsius),
    Rpm(Rpm),
    Volt(Volts),
    Watt(Watts),
}

pub enum AnyCmd {
    Temp(ReadTemp),
    Fan(ReadFanSpeed),
    Voltage(ReadVoltage),
    Power(ReadPowerDraw),
}

impl AnyCmd {
    pub fn execute(&self, bmc: &BmcConnection) -> io::Result<AnyReading> {
        match self {
            AnyCmd::Temp(c)    => Ok(AnyReading::Temp(bmc.execute(c)?)),
            AnyCmd::Fan(c)     => Ok(AnyReading::Rpm(bmc.execute(c)?)),
            AnyCmd::Voltage(c) => Ok(AnyReading::Volt(bmc.execute(c)?)),
            AnyCmd::Power(c)   => Ok(AnyReading::Watt(bmc.execute(c)?)),
        }
    }
}

fn run_dynamic_script(bmc: &BmcConnection, script: &[AnyCmd]) -> io::Result<Vec<AnyReading>> {
    script.iter().map(|cmd| cmd.execute(bmc)).collect()
}
```

## 模式家族

这个模式适用于**每个**硬件管理协议：

| 协议 | 请求类型 | 响应类型 |
|----------|-------------|---------------|
| IPMI 传感器读数 | `ReadTemp` | `Celsius` |
| Redfish REST | `GetThermal` | `ThermalResponse` |
| NVMe Admin | `Identify` | `IdentifyResponse` |
| PLDM | `GetFwParams` | `FwParamsResponse` |
| MCTP | `GetEid` | `EidResponse` |
| PCIe 配置空间 | `ReadCapability` | `CapabilityHeader` |
| SMBIOS/DMI | `ReadType17` | `MemoryDeviceInfo` |

请求类型**决定**响应类型——编译器在每个地方强制执行它。

## 类型化命令流程

```mermaid
flowchart LR
    subgraph "编译时"
        RT["ReadTemp"] -->|"type Response = Celsius"| C[Celsius]
        RF["ReadFanSpeed"] -->|"type Response = Rpm"| R[Rpm]
        RV["ReadVoltage"] -->|"type Response = Volts"| V[Volts]
    end
    subgraph "运行时"
        E["bmc.execute(&cmd)"] -->|"单态化"| P["cmd.parse_response(raw)"]
    end
    style RT fill:#e1f5fe,color:#000
    style RF fill:#e1f5fe,color:#000
    style RV fill:#e1f5fe,color:#000
    style C fill:#c8e6c9,color:#000
    style R fill:#c8e6c9,color:#000
    style V fill:#c8e6c9,color:#000
    style E fill:#fff3e0,color:#000
    style P fill:#fff3e0,color:#000
```

## 练习：PLDM 类型化命令

设计一个 `PldmCmd` trait（与 `IpmiCmd` 形状相同）用于两个 PLDM 命令：
- `GetFwParams` → `FwParamsResponse { active_version: String, pending_version: Option<String> }`
- `QueryDeviceIds` → `DeviceIdResponse { descriptors: Vec<Descriptor> }`

要求：静态分发，`parse_response` 返回 `io::Result<Self::Response>`。

<details>
<summary>解决方案</summary>

```rust,ignore
use std::io;

pub trait PldmCmd {
    type Response;
    fn pldm_type(&self) -> u8;
    fn command_code(&self) -> u8;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

#[derive(Debug, Clone)]
pub struct FwParamsResponse {
    pub active_version: String,
    pub pending_version: Option<String>,
}

pub struct GetFwParams;
impl PldmCmd for GetFwParams {
    type Response = FwParamsResponse;
    fn pldm_type(&self) -> u8 { 0x05 } // Firmware Update
    fn command_code(&self) -> u8 { 0x02 }
    fn parse_response(&self, raw: &[u8]) -> io::Result<FwParamsResponse> {
        // 简化——真实实现解码 PLDM FW Update 规范字段
        if raw.len() < 4 {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "too short"));
        }
        Ok(FwParamsResponse {
            active_version: String::from_utf8_lossy(&raw[..4]).to_string(),
            pending_version: None,
        })
    }
}

#[derive(Debug, Clone)]
pub struct Descriptor { pub descriptor_type: u16, pub data: Vec<u8> }

#[derive(Debug, Clone)]
pub struct DeviceIdResponse { pub descriptors: Vec<Descriptor> }

pub struct QueryDeviceIds;
impl PldmCmd for QueryDeviceIds {
    type Response = DeviceIdResponse;
    fn pldm_type(&self) -> u8 { 0x05 }
    fn command_code(&self) -> u8 { 0x04 }
    fn parse_response(&self, raw: &[u8]) -> io::Result<DeviceIdResponse> {
        Ok(DeviceIdResponse { descriptors: vec![] }) // stub
    }
}
```

</details>

## 关键要点

1. **关联类型 = 编译时契约** —— 命令 trait 上的 `type Response` 将每个请求锁定到精确的一个响应类型。
2. **解析被封装** —— 字节布局知识存在于 `parse_response` 中，而不是分散在调用者中。
3. **零成本分发** —— 泛型 `execute<C: IpmiCmd>` 单态化为直接调用，没有虚表。
4. **一个模式，多个协议** —— IPMI、Redfish、NVMe、PLDM、MCTP 都适合同一个 `trait Cmd { type Response; }` 形状。
5. **Enum 分发桥接静态和动态** —— 将类型化命令包装在 enum 中用于运行时驱动的脚本，同时不丢失每个分支内的类型安全。
6. **渐进复杂度增强直觉** —— IPMI（传感器 ID → 读数）、Redfish（端点 → JSON schema）和 NVMe（操作码 → 4 KB 结构覆盖）都使用相同的 trait 形状，但每个节拍增加一层解析复杂度。

---
