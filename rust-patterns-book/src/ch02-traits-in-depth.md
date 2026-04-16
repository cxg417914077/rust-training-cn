# 2. Traits 深入 🟡

> **你将学到：**
> - 关联类型 vs generic 参数 —— 以及何时使用每种
> - GATs、blanket impls、marker traits 和 trait object 安全规则
> - vtables 和 fat pointers 如何在底层工作
> - 扩展 traits、enum dispatch 和 typed command 模式

## 关联类型 vs Generic 参数

两者都让 trait 可以与不同类型一起工作，但服务于不同目的：

```rust
// --- 关联类型：每个类型一个实现 ---
trait Iterator {
    type Item; // 每个迭代器恰好产生一种 item

    fn next(&mut self) -> Option<Self::Item>;
}

// 一个总是生成 i32 的自定义迭代器 —— 没得选
struct Counter { max: i32, current: i32 }

impl Iterator for Counter {
    type Item = i32; // 每个实现恰好一个 Item 类型
    fn next(&mut self) -> Option<i32> {
        if self.current < self.max {
            self.current += 1;
            Some(self.current)
        } else {
            None
        }
    }
}

// --- GENERIC 参数：每个类型多个实现 ---
trait Convert<T> {
    fn convert(&self) -> T;
}

// 单个类型可以为 MANY 目标类型实现 Convert：
impl Convert<f64> for i32 {
    fn convert(&self) -> f64 { *self as f64 }
}
impl Convert<String> for i32 {
    fn convert(&self) -> String { self.to_string() }
}
```

**何时使用哪个**：

| 使用 | 何时 |
|-----|------|
| **关联类型** | 每个实现类型恰好有一个自然输出/结果。`Iterator::Item`、`Deref::Target`、`Add::Output` |
| **Generic 参数** | 类型可以为许多不同类型有意义地实现 trait。`From<T>`、`AsRef<T>`、`PartialEq<Rhs>` |

**直觉**：如果问"这个迭代器的 `Item` 是什么？"有意义，使用关联类型。如果问"这个可以转换成 `f64` 吗？转换成 `String` 吗？转换成 `bool` 吗？"有意义，使用 generic 参数。

```rust
// 现实世界示例：std::ops::Add
trait Add<Rhs = Self> {
    type Output; // 关联类型 —— 加法有一个结果类型
    fn add(self, rhs: Rhs) -> Self::Output;
}

// Rhs 是 generic 参数 —— 你可以加不同类型到 Meters：
struct Meters(f64);
struct Centimeters(f64);

impl Add<Meters> for Meters {
    type Output = Meters;
    fn add(self, rhs: Meters) -> Meters { Meters(self.0 + rhs.0) }
}
impl Add<Centimeters> for Meters {
    type Output = Meters;
    fn add(self, rhs: Centimeters) -> Meters { Meters(self.0 + rhs.0 / 100.0) }
}
```

### Generic 关联类型（GATs）

自 Rust 1.65 起，关联类型可以有自己的 generic 参数。
这使得 **lending 迭代器** 成为可能 —— 返回绑定到迭代器而非底层集合的引用的迭代器：

```rust
// 无 GATs —— 无法表达 lending 迭代器：
// trait LendingIterator {
//     type Item<'a>;  // ← 这在 1.65 之前被拒绝
// }

// 有 GATs（Rust 1.65+）：
trait LendingIterator {
    type Item<'a> where Self: 'a;

    fn next(&mut self) -> Option<Self::Item<'_>>;
}

// 示例：生成重叠窗口的迭代器
struct WindowIter<'data> {
    data: &'data [u8],
    pos: usize,
    window_size: usize,
}

impl<'data> LendingIterator for WindowIter<'data> {
    type Item<'a> = &'a [u8] where Self: 'a;

    fn next(&mut self) -> Option<&[u8]> {
        if self.pos + self.window_size <= self.data.len() {
            let window = &self.data[self.pos..self.pos + self.window_size];
            self.pos += 1;
            Some(window)
        } else {
            None
        }
    }
}
```

> **何时需要 GATs**：Lending 迭代器、流式解析器，或任何关联类型的生命周期依赖于 `&self` 借用的 trait。
> 对大多数代码，普通关联类型就足够了。

### Supertraits 和 Trait 层次结构

Traits 可以要求其他 traits 作为先决条件，形成层次结构：

```mermaid
graph BT
    Display["Display"]
    Debug["Debug"]
    Error["Error"]
    Clone["Clone"]
    Copy["Copy"]
    PartialEq["PartialEq"]
    Eq["Eq"]
    PartialOrd["PartialOrd"]
    Ord["Ord"]

    Error --> Display
    Error --> Debug
    Copy --> Clone
    Eq --> PartialEq
    Ord --> Eq
    Ord --> PartialOrd
    PartialOrd --> PartialEq

    style Display fill:#e8f4f8,stroke:#2980b9,color:#000
    style Debug fill:#e8f4f8,stroke:#2980b9,color:#000
    style Error fill:#fdebd0,stroke:#e67e22,color:#000
    style Clone fill:#d4efdf,stroke:#27ae60,color:#000
    style Copy fill:#d4efdf,stroke:#27ae60,color:#000
    style PartialEq fill:#fef9e7,stroke:#f1c40f,color:#000
    style Eq fill:#fef9e7,stroke:#f1c40f,color:#000
    style PartialOrd fill:#fef9e7,stroke:#f1c40f,color:#000
    style Ord fill:#fef9e7,stroke:#f1c40f,color:#000
```

> 箭头从 subtrait 指向 supertrait：实现 `Error` 需要 `Display` + `Debug`。

一个 trait 可以要求实现者也实现其他 traits：

```rust
use std::fmt;

// Display 是 Error 的 supertrait
trait Error: fmt::Display + fmt::Debug {
    fn source(&self) -> Option<&(dyn Error + 'static)> { None }
}
// 任何实现 Error 的类型也必须实现 Display 和 Debug

// 构建你自己的层次结构：
trait Identifiable {
    fn id(&self) -> u64;
}

trait Timestamped {
    fn created_at(&self) -> chrono::DateTime<chrono::Utc>;
}

// Entity 要求两者：
trait Entity: Identifiable + Timestamped {
    fn is_active(&self) -> bool;
}

// 实现 Entity 强制你实现所有三个：
struct User { id: u64, name: String, created: chrono::DateTime<chrono::Utc> }

impl Identifiable for User {
    fn id(&self) -> u64 { self.id }
}
impl Timestamped for User {
    fn created_at(&self) -> chrono::DateTime<chrono::Utc> { self.created }
}
impl Entity for User {
    fn is_active(&self) -> bool { true }
}
```

### Blanket 实现

为所有满足某些 bound 的类型实现 trait：

```rust
// std 这样做：任何实现 Display 的类型自动获得 ToString
impl<T: fmt::Display> ToString for T {
    fn to_string(&self) -> String {
        format!("{self}")
    }
}
// 现在 i32、&str、你的自定义类型 —— 任何有 Display 的 —— 免费获得 to_string()。

// 你自己的 blanket 实现：
trait Loggable {
    fn log(&self);
}

// 每个 Debug 类型自动成为 Loggable：
impl<T: std::fmt::Debug> Loggable for T {
    fn log(&self) {
        eprintln!("[LOG] {self:?}");
    }
}

// 现在任何 Debug 类型都有 .log()：
// 42.log();              // [LOG] 42
// "hello".log();         // [LOG] "hello"
// vec![1, 2, 3].log();   // [LOG] [1, 2, 3]
```

> **注意**：Blanket impls 强大但不可逆 —— 你不能为已被 blanket impl 覆盖的类型添加
> 更具体的实现（orphan rules + coherence）。仔细设计它们。

### Marker Traits

没有方法的 traits —— 它们标记类型具有某些属性：

```rust
// 标准库 marker traits：
// Send    —— 安全在线程间转移
// Sync    —— 安全在线程间共享 (&T)
// Unpin   —— pinning 后安全移动
// Sized   —— 编译时有已知大小
// Copy    —— 可以用 memcpy 复制

// 你自己的 marker trait：
/// Marker：这个传感器已工厂校准
trait Calibrated {}

struct RawSensor { reading: f64 }
struct CalibratedSensor { reading: f64 }

impl Calibrated for CalibratedSensor {}

// 只有校准的传感器可以在生产中使用：
fn record_measurement<S: Calibrated>(sensor: &S) {
    // ...
}
// record_measurement(&RawSensor { reading: 0.0 }); // ❌ 编译错误
// record_measurement(&CalibratedSensor { reading: 0.0 }); // ✅
```

这直接连接到第 3 章的**type-state 模式**。

### Trait Object 安全规则

不是每个 trait 都可以用作 `dyn Trait`。一个 trait 仅当以下条件满足时才是**object-safe**的：

1. **trait 本身没有 `Self: Sized` bound**
2. **方法没有 generic 类型参数**
3. **没有在返回位置使用 `Self`**（除非通过 `Box<Self>` 等间接方式）
4. **没有关联函数**（方法必须有 `&self`、`&mut self` 或 `self`）

```rust
// ✅ Object-safe —— 可以用作 dyn Drawable
trait Drawable {
    fn draw(&self);
    fn bounding_box(&self) -> (f64, f64, f64, f64);
}

let shapes: Vec<Box<dyn Drawable>> = vec![/* ... */]; // ✅ 工作

// ❌ 不 object-safe —— 在返回位置使用 Self
trait Cloneable {
    fn clone_self(&self) -> Self;
    //                       ^^^^ 运行时无法知道具体大小
}
// let items: Vec<Box<dyn Cloneable>> = ...; // ❌ 编译错误

// ❌ 不 object-safe —— generic 方法
trait Converter {
    fn convert<T>(&self) -> T;
    //        ^^^ vtable 无法包含无限单态化
}

// ❌ 不 object-safe —— 关联函数（无 self）
trait Factory {
    fn create() -> Self;
    // 无 &self —— 如何通过 trait object 调用这个？
}
```

**变通方法**：

```rust
// 添加 `where Self: Sized` 将方法从 vtable 排除：
trait MyTrait {
    fn regular_method(&self); // 包含在 vtable 中

    fn generic_method<T>(&self) -> T
    where
        Self: Sized; // 从 vtable 排除 —— 不能通过 dyn MyTrait 调用
}

// 现在 dyn MyTrait 有效，但 generic_method 只能
// 在具体类型已知时调用。
```

> **经验法则**：如果计划使用 `dyn Trait`，保持方法简单 ——
> 无 generics，无返回类型中的 `Self`，无 `Sized` bounds。如有疑问，
> 尝试 `let _: Box<dyn YourTrait>;` 让编译器告诉你。

### Trait Objects 底层 —— vtables 和 Fat Pointers

`&dyn Trait`（或 `Box<dyn Trait>`）是**fat pointer** —— 两个机器字：

```text
┌──────────────────────────────────────────────────┐
│  &dyn Drawable（64 位：总共 16 字节）              │
├──────────────┬───────────────────────────────────┤
│  data_ptr    │  vtable_ptr                       │
│  (8 字节)    │  (8 字节)                         │
│  ↓           │  ↓                                │
│  ┌─────────┐ │  ┌──────────────────────────────┐ │
│  │ Circle  │ │  │ vtable for <Circle as        │ │
│  │ {       │ │  │           Drawable>          │ │
│  │  r: 5.0 │ │  │                              │ │
│  │ }       │ │  │  drop_in_place: 0x7f...a0    │ │
│  └─────────┘ │  │  size:           8           │ │
│              │  │  align:          8           │ │
│              │  │  draw:          0x7f...b4    │ │
│              │  │  bounding_box:  0x7f...c8    │ │
│              │  └──────────────────────────────┘ │
└──────────────┴───────────────────────────────────┘
```

**vtable 调用如何工作**（例如 `shape.draw()`）：

1. 从 fat pointer 加载 `vtable_ptr`（第二个字）
2. 索引到 vtable 找到 `draw` 函数指针
3. 调用它，传递 `data_ptr` 作为 `self` 参数

这与 C++ 虚函数调用的成本相似（每次调用一次指针间接），但 Rust 将 vtable 指针存储在 fat pointer 中而非对象内部 —— 所以栈上的普通 `Circle` 根本不携带 vtable 指针。

```rust
trait Drawable {
    fn draw(&self);
    fn area(&self) -> f64;
}

struct Circle { radius: f64 }

impl Drawable for Circle {
    fn draw(&self) { println!("Drawing circle r={}", self.radius); }
    fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius }
}

struct Square { side: f64 }

impl Drawable for Square {
    fn draw(&self) { println!("Drawing square s={}", self.side); }
    fn area(&self) -> f64 { self.side * self.side }
}

fn main() {
    let shapes: Vec<Box<dyn Drawable>> = vec![
        Box::new(Circle { radius: 5.0 }),
        Box::new(Square { side: 3.0 }),
    ];

    // 每个元素是 fat pointer：(data_ptr, vtable_ptr)
    // Circle 和 Square 的 vtable 不同
    for shape in &shapes {
        shape.draw();  // vtable dispatch → Circle::draw 或 Square::draw
        println!("  area = {:.2}", shape.area());
    }

    // 大小比较：
    println!("size_of::<&Circle>()        = {}", size_of::<&Circle>());
    // → 8 字节（一个指针 —— 编译器知道类型）
    println!("size_of::<&dyn Drawable>()  = {}", size_of::<&dyn Drawable>());
    // → 16 字节（data_ptr + vtable_ptr）
}
```

**性能成本模型**：

| 方面 | 静态 dispatch（`impl Trait` / generics） | 动态 dispatch（`dyn Trait`） |
|--------|:-----------:|:-------------:|
| 调用开销 | 零 —— LLVM 内联 | 每次调用一次指针间接 |
| 内联 | ✅ 编译器可以内联 | ❌ 不透明函数指针 |
| 二进制大小 | 更大（每个类型一份副本） | 更小（共享代码） |
| 指针大小 | 瘦（1 字） | 胖（2 字） |
| 异构集合 | ❌ | ✅ `Vec<Box<dyn Trait>>` |

> **vtable 成本何时重要**：在调用 trait 方法数百万次的紧密循环中，
> 间接和无法内联可能显著（2-10 倍慢）。对于冷路径、配置或插件架构，
> `dyn Trait` 的灵活性值得这个小成本。

### Higher-Ranked Trait Bounds（HRTBs）

有时需要一个函数可以使用*任何*生命周期的引用工作，而不是一个特定的生命周期。这就是 `for<'a>` 语法出现的地方：

```rust
// 问题：这个函数需要一个闭包，可以处理
// 任何生命周期的引用，而不仅是一个特定生命周期。

// ❌ 这太局限了 —— 'a 由调用者固定：
// fn apply<'a, F: Fn(&'a str) -> &'a str>(f: F, data: &'a str) -> &'a str

// ✅ HRTB：F 必须对所有可能的生命周期有效：
fn apply<F>(f: F, data: &str) -> &str
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    f(data)
}

fn main() {
    let result = apply(|s| s.trim(), "  hello  ");
    println!("{result}"); // "hello"
}
```

**何时遇到 HRTBs**：
- `Fn(&T) -> &U` traits —— 编译器在大多数情况下自动推断 `for<'a>`
- 必须跨不同借用工作的自定义 trait 实现
- 带 `serde` 的反序列化：`for<'de> Deserialize<'de>`

```rust,ignore
// serde 的 DeserializeOwned 定义为：
// trait DeserializeOwned: for<'de> Deserialize<'de> {}
// 含义："可以从任何生命周期的数据反序列化"
// （即结果不从输入借用）

use serde::de::DeserializeOwned;

fn parse_json<T: DeserializeOwned>(input: &str) -> T {
    serde_json::from_str(input).unwrap()
}
```

> **实用建议**：你很少会自己写 `for<'a>`。它主要出现在
> 闭包参数的 trait bounds 中，编译器隐式处理它。
> 但在错误消息中识别它（"expected a `for<'a> Fn(&'a ...)` bound"）
> 帮助你理解编译器要求什么。

### `impl Trait` —— 参数位置 vs 返回位置

`impl Trait` 出现在两个位置，具有**不同的语义**：

```rust
// --- 参数位置 impl Trait（APIT） ---
// "调用者选择类型" —— generic 参数的语法糖
fn print_all(items: impl Iterator<Item = i32>) {
    for item in items { println!("{item}"); }
}
// 等价于：
fn print_all_verbose<I: Iterator<Item = i32>>(items: I) {
    for item in items { println!("{item}"); }
}
// 调用者决定：print_all(vec![1,2,3].into_iter())
//             print_all(0..10)

// --- 返回位置 impl Trait（RPIT） ---
// "被调用者选择类型" —— 函数挑选一个具体类型
fn evens(limit: i32) -> impl Iterator<Item = i32> {
    (0..limit).filter(|x| x % 2 == 0)
    // 具体类型是 Filter<Range<i32>, Closure>
    // 但调用者只看到"某个 Iterator<Item = i32>"
}
```

**关键区别**：

| | APIT（`fn foo(x: impl T)`） | RPIT（`fn foo() -> impl T`） |
|---|---|---|
| 谁选择类型？ | 调用者 | 被调用者（函数体） |
| 单态化？ | 是 —— 每个类型一份副本 | 是 —— 一个具体类型 |
| Turbofish？ | 否（`foo::<X>()` 不允许） | N/A |
| 等价于 | `fn foo<X: T>(x: X)` | 存在类型 |

#### Trait 定义中的 RPIT（RPITIT）

自 Rust 1.75 起，你可以在 trait 定义中直接使用 `-> impl Trait`：

```rust
trait Container {
    fn items(&self) -> impl Iterator<Item = &str>;
    //                 ^^^^ 每个实现者返回自己的具体类型
}

struct CsvRow {
    fields: Vec<String>,
}

impl Container for CsvRow {
    fn items(&self) -> impl Iterator<Item = &str> {
        self.fields.iter().map(String::as_str)
    }
}

struct FixedFields;

impl Container for FixedFields {
    fn items(&self) -> impl Iterator<Item = &str> {
        ["host", "port", "timeout"].into_iter()
    }
}
```

> **Rust 1.75 之前**，你必须使用 `Box<dyn Iterator>` 或关联类型
> 来在 traits 中实现这个。RPITIT 消除了分配。

#### `impl Trait` vs `dyn Trait` —— 决策指南

```text
编译时知道具体类型？
├── 是 → 使用 impl Trait 或 generics（零成本，可内联）
└── 否  → 需要异构集合？
     ├── 是 → 使用 dyn Trait（Box<dyn T>、&dyn T）
     └── 否  → 需要跨 API 边界的相同 trait object？
          ├── 是 → 使用 dyn Trait
          └── 否  → 使用 generics / impl Trait
```

| 特性 | `impl Trait` | `dyn Trait` |
|---------|-------------|------------|
| Dispatch | 静态（单态化） | 动态（vtable） |
| 性能 | 最佳 —— 可内联 | 每次调用一次间接 |
| 异构集合 | ❌ | ✅ |
| 二进制大小每个类型 | 每个类型一份副本 | 共享代码 |
| Trait 必须 object-safe？ | 否 | 是 |
| 在 trait 定义中工作 | ✅（Rust 1.75+） | 总是 |

***

## 带 `Any` 和 `TypeId` 的类型擦除

有时你需要存储*未知*类型的值并在之后向下转换 —— 这种模式在 C 的 `void*` 或 C# 的 `object` 中很熟悉。Rust 通过 `std::any::Any` 提供这个：

```rust
use std::any::Any;

// 存储异构值：
fn log_value(value: &dyn Any) {
    if let Some(s) = value.downcast_ref::<String>() {
        println!("String: {s}");
    } else if let Some(n) = value.downcast_ref::<i32>() {
        println!("i32: {n}");
    } else {
        // TypeId 允许你在运行时检查类型：
        println!("Unknown type: {:?}", value.type_id());
    }
}

// 对插件系统、事件总线或 ECS 风格架构有用：
struct AnyMap(std::collections::HashMap<std::any::TypeId, Box<dyn Any + Send>>);

impl AnyMap {
    fn new() -> Self { AnyMap(std::collections::HashMap::new()) }

    fn insert<T: Any + Send + 'static>(&mut self, value: T) {
        self.0.insert(std::any::TypeId::of::<T>(), Box::new(value));
    }

    fn get<T: Any + Send + 'static>(&self) -> Option<&T> {
        self.0.get(&std::any::TypeId::of::<T>())?
            .downcast_ref()
    }
}

fn main() {
    let mut map = AnyMap::new();
    map.insert(42_i32);
    map.insert(String::from("hello"));

    assert_eq!(map.get::<i32>(), Some(&42));
    assert_eq!(map.get::<String>().map(|s| s.as_str()), Some("hello"));
    assert_eq!(map.get::<f64>(), None); // 从未插入
}
```

> **何时使用 `Any`**：插件/扩展系统、类型索引映射（`typemap`）、
> 错误向下转换（`anyhow::Error::downcast_ref`）。当类型集在编译时已知时，
> 优先 generics 或 trait objects —— `Any` 是最后手段，
> 用编译时安全换取灵活性。

***

## 扩展 Traits —— 为你不拥有的类型添加方法

Rust 的 orphan rule 阻止你为外部类型实现外部 trait。
扩展 traits 是标准的变通方法：在你的 crate 中定义一个**新 trait**，其方法
对任何满足 bound 的类型都有 blanket 实现。调用者导入 trait，
新方法出现在现有类型上。

这种模式在 Rust 生态系统中无处不在：`itertools::Itertools`、`futures::StreamExt`、
`tokio::io::AsyncReadExt`、`tower::ServiceExt`。

### 问题

```rust
// 我们想为所有生成 f64 的迭代器添加 .mean() 方法。
// 但 Iterator 在 std 中定义，f64 是原始类型 —— orphan rule 阻止：
//
// impl<I: Iterator<Item = f64>> I {   // ❌ 不能为外部类型添加固有方法
//     fn mean(self) -> f64 { ... }
// }
```

### 解决方案：扩展 Trait

```rust
/// 迭代器数值方法的扩展 trait。
pub trait IteratorExt: Iterator {
    /// 计算算术平均值。为空迭代器返回 `None`。
    fn mean(self) -> Option<f64>
    where
        Self: Sized,
        Self::Item: Into<f64>;
}

// Blanket 实现 —— 自动应用于所有迭代器
impl<I: Iterator> IteratorExt for I {
    fn mean(self) -> Option<f64>
    where
        Self: Sized,
        Self::Item: Into<f64>,
    {
        let mut sum: f64 = 0.0;
        let mut count: u64 = 0;
        for item in self {
            sum += item.into();
            count += 1;
        }
        if count == 0 { None } else { Some(sum / count as f64) }
    }
}

// 用法 —— 只需导入 trait：
use crate::IteratorExt;  // 一个导入，方法出现在所有迭代器上

fn analyze_temperatures(readings: &[f64]) -> Option<f64> {
    readings.iter().copied().mean()  // .mean() 现在可用！
}

fn analyze_sensor_data(data: &[i32]) -> Option<f64> {
    data.iter().copied().mean()  // 也适用于 i32（i32: Into<f64>）
}
```

### 现实世界示例：诊断 Result 扩展

```rust
use std::collections::HashMap;

struct DiagResult {
    component: String,
    passed: bool,
    message: String,
}

/// Vec<DiagResult> 的扩展 trait —— 添加领域特定分析方法。
pub trait DiagResultsExt {
    fn passed_count(&self) -> usize;
    fn failed_count(&self) -> usize;
    fn overall_pass(&self) -> bool;
    fn failures_by_component(&self) -> HashMap<String, Vec<&DiagResult>>;
}

impl DiagResultsExt for Vec<DiagResult> {
    fn passed_count(&self) -> usize {
        self.iter().filter(|r| r.passed).count()
    }

    fn failed_count(&self) -> usize {
        self.iter().filter(|r| !r.passed).count()
    }

    fn overall_pass(&self) -> bool {
        self.iter().all(|r| r.passed)
    }

    fn failures_by_component(&self) -> HashMap<String, Vec<&DiagResult>> {
        let mut map = HashMap::new();
        for r in self.iter().filter(|r| !r.passed) {
            map.entry(r.component.clone()).or_default().push(r);
        }
        map
    }
}

// 现在任何 Vec<DiagResult> 都有这些方法：
fn report(results: Vec<DiagResult>) {
    if !results.overall_pass() {
        let failures = results.failures_by_component();
        for (component, fails) in &failures {
            eprintln!("{component}: {} failures", fails.len());
        }
    }
}
```

### 命名约定

Rust 生态系统使用一致的 `Ext` 后缀：

| Crate | 扩展 Trait | 扩展 |
|-------|----------------|---------|
| `itertools` | `Itertools` | `Iterator` |
| `futures` | `StreamExt`、`FutureExt` | `Stream`、`Future` |
| `tokio` | `AsyncReadExt`、`AsyncWriteExt` | `AsyncRead`、`AsyncWrite` |
| `tower` | `ServiceExt` | `Service` |
| `bytes` | `BufMut`（部分） | `&mut [u8]` |
| 你的 crate | `DiagResultsExt` | `Vec<DiagResult>` |

### 何时使用

| 情况 | 使用扩展 Trait？ |
|-----------|:---:|
| 为外部类型添加便利方法 | ✅ |
| 在 generic 集合上分组领域特定逻辑 | ✅ |
| 方法需要访问私有字段 | ❌（使用 wrapper/newtype） |
| 方法逻辑上属于你控制的新类型 | ❌（直接添加到你的类型） |
| 你希望方法无需任何导入就可用 | ❌（仅固有方法） |

***

## Enum Dispatch —— 无 `dyn` 的静态多态

当你有实现 trait 的**封闭集**类型时，你可以用包含具体类型的 enum 替换 `dyn Trait`。这消除了 vtable 间接和堆分配，同时保留相同的调用者面向接口。

### `dyn Trait` 的问题

```rust
trait Sensor {
    fn read(&self) -> f64;
    fn name(&self) -> &str;
}

struct Gps { lat: f64, lon: f64 }
struct Thermometer { temp_c: f64 }
struct Accelerometer { g_force: f64 }

impl Sensor for Gps {
    fn read(&self) -> f64 { self.lat }
    fn name(&self) -> &str { "GPS" }
}
impl Sensor for Thermometer {
    fn read(&self) -> f64 { self.temp_c }
    fn name(&self) -> &str { "Thermometer" }
}
impl Sensor for Accelerometer {
    fn read(&self) -> f64 { self.g_force }
    fn name(&self) -> &str { "Accelerometer" }
}

// 带 dyn 的异构集合 —— 工作，但有成本：
fn read_all_dyn(sensors: &[Box<dyn Sensor>]) -> Vec<f64> {
    sensors.iter().map(|s| s.read()).collect()
    // 每个 .read() 通过 vtable 间接
    // 每个 Box 在堆上分配
}
```

### Enum Dispatch 解决方案

```rust
// 用 enum 替换 trait object：
enum AnySensor {
    Gps(Gps),
    Thermometer(Thermometer),
    Accelerometer(Accelerometer),
}

impl AnySensor {
    fn read(&self) -> f64 {
        match self {
            AnySensor::Gps(s) => s.read(),
            AnySensor::Thermometer(s) => s.read(),
            AnySensor::Accelerometer(s) => s.read(),
        }
    }

    fn name(&self) -> &str {
        match self {
            AnySensor::Gps(s) => s.name(),
            AnySensor::Thermometer(s) => s.name(),
            AnySensor::Accelerometer(s) => s.name(),
        }
    }
}

// 现在：无堆分配，无 vtable，内联存储
fn read_all(sensors: &[AnySensor]) -> Vec<f64> {
    sensors.iter().map(|s| s.read()).collect()
    // 每个 .read() 是 match 分支 —— 编译器可以内联所有
}

fn main() {
    let sensors = vec![
        AnySensor::Gps(Gps { lat: 47.6, lon: -122.3 }),
        AnySensor::Thermometer(Thermometer { temp_c: 72.5 }),
        AnySensor::Accelerometer(Accelerometer { g_force: 1.02 }),
    ];

    for sensor in &sensors {
        println!("{}: {:.2}", sensor.name(), sensor.read());
    }
}
```

### 为 Enum 实现 Trait

为了互操作性，你可以在 enum 本身上实现原始 trait：

```rust
impl Sensor for AnySensor {
    fn read(&self) -> f64 {
        match self {
            AnySensor::Gps(s) => s.read(),
            AnySensor::Thermometer(s) => s.read(),
            AnySensor::Accelerometer(s) => s.read(),
        }
    }

    fn name(&self) -> &str {
        match self {
            AnySensor::Gps(s) => s.name(),
            AnySensor::Thermometer(s) => s.name(),
            AnySensor::Accelerometer(s) => s.name(),
        }
    }
}

// 现在 AnySensor 可以通过 generics 在任何期望 Sensor 的地方工作：
fn report<S: Sensor>(s: &S) {
    println!("{}: {:.2}", s.name(), s.read());
}
```

### 用宏减少样板代码

match 分支委托是重复的。宏消除了它：

```rust
macro_rules! dispatch_sensor {
    ($self:expr, $method:ident $(, $arg:expr)*) => {
        match $self {
            AnySensor::Gps(s) => s.$method($($arg),*),
            AnySensor::Thermometer(s) => s.$method($($arg),*),
            AnySensor::Accelerometer(s) => s.$method($($arg),*),
        }
    };
}

impl Sensor for AnySensor {
    fn read(&self) -> f64     { dispatch_sensor!(self, read) }
    fn name(&self) -> &str    { dispatch_sensor!(self, name) }
}
```

对于更大的项目，`enum_dispatch` crate 完全自动化这个：

```rust
use enum_dispatch::enum_dispatch;

#[enum_dispatch]
trait Sensor {
    fn read(&self) -> f64;
    fn name(&self) -> &str;
}

#[enum_dispatch(Sensor)]
enum AnySensor {
    Gps,
    Thermometer,
    Accelerometer,
}
// 所有委托代码自动生成。
```

### `dyn Trait` vs Enum Dispatch —— 决策指南

```text
类型集封闭（编译时已知）？
├── 是 → 优先 enum dispatch（更快，无堆分配）
│         ├── 少数变体（< ~20）？     → 手动 enum
│         └── 许多变体或增长？ → enum_dispatch crate
└── 否  → 必须使用 dyn Trait（插件、用户提供的类型）
```

| 属性 | `dyn Trait` | Enum Dispatch |
|----------|:-----------:|:-------------:|
| Dispatch 成本 | Vtable 间接（~2ns） | 分支预测（~0.3ns） |
| 堆分配 | 通常（Box） | 无（内联） |
| 缓存友好 | 否（指针追逐） | 是（连续） |
| 对新类型开放 | ✅（任何人都可以实现） | ❌（封闭集） |
| 代码大小 | 共享 | 每个变体一份副本 |
| Trait 必须 object-safe | 是 | 否 |
| 添加变体 | 无代码更改 | 更新 enum + match 分支 |

### 何时使用 Enum Dispatch

| 场景 | 推荐 |
|----------|---------------|
| 诊断测试类型（CPU、GPU、NIC、Memory、...） | ✅ Enum dispatch —— 封闭集，编译时已知 |
| 总线协议（SPI、I2C、UART、...） | ✅ Enum dispatch 或 Config trait |
| 插件系统（用户运行时加载 .so） | ❌ 使用 `dyn Trait` |
| 2-3 个变体 | ✅ 手动 enum dispatch |
| 10+ 变体带许多方法 | ✅ `enum_dispatch` crate |
| 性能关键的内部循环 | ✅ Enum dispatch（消除 vtable） |

***

## Capability Mixins —— 组合式诊断功能

在 Ruby 中，**Mixin** 是模块化的基石 —— 你可以 `include` 一个模块来获得它的方法，同时保持继承层次结构的扁平。在 Rust 中，你可以使用 trait 的组合和 blanket impls 实现类似的效果，但要灵活得多：方法可以有条件地存在（基于 associated types），组合在编译时验证，没有运行时成本。

### 问题：跨诊断共享外设访问

固件诊断代码通常需要从多个源读取：

```rust
// ❌ 现实世界中 C/C++ 风格的诊断函数
fn check_thermal_thresholds() -> bool {
    // 从 SPI 读取 thermocouple
    let tc = spi_read_thermocouple();
    
    // 从 I2C 读取风扇转速
    let fan_rpm = i2c_read_fan(0);
    
    // 从 IPMI/BMC 读取温度传感器
    let bmc_temp = ipmi_get_sensor(0x20);
    
    // 从 GPIO 控制风扇使能
    gpio_set_fan_enable(true);
    
    tc < 85.0 && fan_rpm > 4000 && bmc_temp < 90.0
}
```

这个函数"知道"太多东西了 —— 它直接调用 `spi_read_thermocouple()`、`i2c_read_fan()` 等。测试它需要：
- 模拟这些全局函数（在 C 中很难）
- 或者在实际硬件上运行（在 CI 中不可能）

### 解决方案：Capability Mixins

这个模式将**能力**（"我有 SPI"、"我有 I2C"）与**功能**（"我可以读取 thermocouple"、"我可以检查风扇健康"）分开。

#### 步骤 1 —— 定义"能力" Traits（Ingredient Traits）

每个能力 trait 暴露一个 associated type 和一个 accessor 方法：

```rust
/// 能力：有一个 SPI 总线
pub trait HasSpi {
    type Spi: SpiBus;
    fn spi(&self) -> &Self::Spi;
}

/// 能力：有一个 I2C 总线
pub trait HasI2c {
    type I2c: I2cBus;
    fn i2c(&self) -> &Self::I2c;
}

/// 能力：有一个 GPIO 引脚
pub trait HasGpio {
    type Gpio: GpioPin;
    fn gpio(&self) -> &Self::Gpio;
}

/// 能力：有一个 IPMI BMC 连接
pub trait HasIpmi {
    type Ipmi: IpmiBmc;
    fn ipmi(&self) -> &Self::Ipmi;
}
```

每个 ingredient 都很小、generic，可以独立测试。

### 步骤 2 —— 定义"Mixin" Traits

Mixin trait 声明其所需的 ingredient 作为 supertraits，然后通过**默认方法**提供所有方法 —— 实现者自动获得它们：

```rust
/// Mixin：风扇诊断 —— 需要 I2C（转速计）+ GPIO（PWM 使能）
pub trait FanDiagMixin: HasI2c + HasGpio {
    /// 从 I2C 转速计读取风扇 RPM。
    fn read_fan_rpm(&self, fan_id: u8) -> io::Result<u32> {
        let mut buf = [0u8; 2];
        self.i2c().i2c_read(0x48 + fan_id, 0x00, &mut buf)?;
        Ok(u16::from_be_bytes(buf) as u32 * 60) // 转速计数 → RPM
    }

    /// 启用或禁用风扇 PWM 输出通过 GPIO。
    fn set_fan_pwm(&self, enable: bool) -> io::Result<()> {
        if enable { self.gpio().set_high() }
        else      { self.gpio().set_low() }
    }

    /// 完整风扇健康检查 —— 读取 RPM + 验证在阈值内。
    fn check_fan_health(&self, fan_id: u8, min_rpm: u32) -> io::Result<bool> {
        let rpm = self.read_fan_rpm(fan_id)?;
        Ok(rpm >= min_rpm)
    }
}

/// Mixin：温度监控 —— 需要 SPI（热电偶 ADC）+ IPMI（BMC 传感器）
pub trait TempMonitorMixin: HasSpi + HasIpmi {
    /// 通过 SPI ADC 读取热电偶（例如 MAX31855）。
    fn read_thermocouple(&self) -> io::Result<f64> {
        let mut rx = [0u8; 4];
        self.spi().spi_transfer(&[0x00; 4], &mut rx)?;
        let raw = i32::from_be_bytes(rx) >> 18; // 14-bit 有符号
        Ok(raw as f64 * 0.25)
    }

    /// 读取 BMC 管理的温度传感器通过 IPMI。
    fn read_bmc_temp(&self, sensor_id: u8) -> io::Result<f64> {
        self.ipmi().read_sensor(sensor_id)
    }

    /// 交叉验证：热电偶 vs BMC 必须在 delta 内一致。
    fn validate_temps(&self, sensor_id: u8, max_delta: f64) -> io::Result<bool> {
        let tc = self.read_thermocouple()?;
        let bmc = self.read_bmc_temp(sensor_id)?;
        Ok((tc - bmc).abs() <= max_delta)
    }
}

/// Mixin：电源排序 —— 需要 GPIO（rail 使能）+ IPMI（事件日志）
pub trait PowerSeqMixin: HasGpio + HasIpmi {
    /// 断言 power-good GPIO 并通过 IPMI 传感器验证。
    fn enable_power_rail(&self, sensor_id: u8) -> io::Result<bool> {
        self.gpio().set_high()?;
        std::thread::sleep(std::time::Duration::from_millis(50));
        let voltage = self.ipmi().read_sensor(sensor_id)?;
        Ok(voltage > 0.8) // 高于 80% 标称值 = 良好
    }

    /// 取消断言电源并通过 IPMI OEM 命令记录关闭。
    fn disable_power_rail(&self) -> io::Result<()> {
        self.gpio().set_low()?;
        // 记录 OEM "power rail disabled" 事件到 BMC
        self.ipmi().raw_command(0x2E, 0x01, &[0x00, 0x01])?;
        Ok(())
    }
}
```

### 步骤 3 —— Blanket Impls 使其真正成为"Mixin"

神奇的一行 —— 提供 ingredient，获得方法：

```rust
impl<T: HasI2c + HasGpio>  FanDiagMixin    for T {}
impl<T: HasSpi  + HasIpmi>  TempMonitorMixin for T {}
impl<T: HasGpio + HasIpmi>  PowerSeqMixin   for T {}
```

任何实现正确 ingredient traits 的结构体**自动**获得每个 mixin 方法 —— 无样板代码，无转发，无继承。

### 步骤 4 —— 生产环境接线

```rust
// ── 具体总线实现（Linux 平台）────────────────
struct LinuxSpi  { dev: String }
struct LinuxI2c  { dev: String }
struct SysfsGpio { pin: u32 }
struct IpmiTool  { timeout_secs: u32 }

impl SpiBus for LinuxSpi {
    fn spi_transfer(&self, _tx: &[u8], _rx: &mut [u8]) -> io::Result<()> {
        // spidev ioctl —— 省略细节
        Ok(())
    }
}
impl I2cBus for LinuxI2c {
    fn i2c_read(&self, _addr: u8, _reg: u8, _buf: &mut [u8]) -> io::Result<()> {
        // i2c-dev ioctl —— 省略细节
        Ok(())
    }
    fn i2c_write(&self, _addr: u8, _reg: u8, _data: &[u8]) -> io::Result<()> { Ok(()) }
}
impl GpioPin for SysfsGpio {
    fn set_high(&self) -> io::Result<()>  { /* /sys/class/gpio */ Ok(()) }
    fn set_low(&self) -> io::Result<()>   { Ok(()) }
    fn read_level(&self) -> io::Result<bool> { Ok(true) }
}
impl IpmiBmc for IpmiTool {
    fn raw_command(&self, _nf: u8, _cmd: u8, _data: &[u8]) -> io::Result<Vec<u8>> {
        // 调用 ipmitool —— 省略细节
        Ok(vec![])
    }
    fn read_sensor(&self, _id: u8) -> io::Result<f64> { Ok(25.0) }
}

// ── 生产平台 —— 全部四个总线 ────────────────
struct DiagPlatform {
    spi:  LinuxSpi,
    i2c:  LinuxI2c,
    gpio: SysfsGpio,
    ipmi: IpmiTool,
}

impl HasSpi  for DiagPlatform { type Spi  = LinuxSpi;  fn spi(&self)  -> &LinuxSpi  { &self.spi  } }
impl HasI2c  for DiagPlatform { type I2c  = LinuxI2c;  fn i2c(&self)  -> &LinuxI2c  { &self.i2c  } }
impl HasGpio for DiagPlatform { type Gpio = SysfsGpio; fn gpio(&self) -> &SysfsGpio { &self.gpio } }
impl HasIpmi for DiagPlatform { type Ipmi = IpmiTool;  fn ipmi(&self) -> &IpmiTool  { &self.ipmi } }

// DiagPlatform 现在拥有 ALL mixin 方法：
fn production_diagnostics(platform: &DiagPlatform) -> io::Result<()> {
    let rpm = platform.read_fan_rpm(0)?;       // 来自 FanDiagMixin
    let tc  = platform.read_thermocouple()?;   // 来自 TempMonitorMixin
    let ok  = platform.enable_power_rail(42)?;  // 来自 PowerSeqMixin
    println!("Fan: {rpm} RPM, Temp: {tc}°C, Power: {ok}");
    Ok(())
}
```

### 步骤 5 —— 使用 Mock 测试（无需硬件）

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::Cell;

    struct MockSpi  { temp: Cell<f64> }
    struct MockI2c  { rpm: Cell<u32> }
    struct MockGpio { level: Cell<bool> }
    struct MockIpmi { sensor_val: Cell<f64> }

    impl SpiBus for MockSpi {
        fn spi_transfer(&self, _tx: &[u8], rx: &mut [u8]) -> io::Result<()> {
            // 编码 mock 温度为 MAX31855 格式
            let raw = ((self.temp.get() / 0.25) as i32) << 18;
            rx.copy_from_slice(&raw.to_be_bytes());
            Ok(())
        }
    }
    impl I2cBus for MockI2c {
        fn i2c_read(&self, _addr: u8, _reg: u8, buf: &mut [u8]) -> io::Result<()> {
            let tach = (self.rpm.get() / 60) as u16;
            buf.copy_from_slice(&tach.to_be_bytes());
            Ok(())
        }
        fn i2c_write(&self, _: u8, _: u8, _: &[u8]) -> io::Result<()> { Ok(()) }
    }
    impl GpioPin for MockGpio {
        fn set_high(&self)  -> io::Result<()>   { self.level.set(true);  Ok(()) }
        fn set_low(&self)   -> io::Result<()>   { self.level.set(false); Ok(()) }
        fn read_level(&self) -> io::Result<bool> { Ok(self.level.get()) }
    }
    impl IpmiBmc for MockIpmi {
        fn raw_command(&self, _: u8, _: u8, _: &[u8]) -> io::Result<Vec<u8>> { Ok(vec![]) }
        fn read_sensor(&self, _: u8) -> io::Result<f64> { Ok(self.sensor_val.get()) }
    }

    // ── 部分平台：仅风扇相关总线 ─────────────────
    struct FanTestRig {
        i2c:  MockI2c,
        gpio: MockGpio,
    }
    impl HasI2c  for FanTestRig { type I2c  = MockI2c;  fn i2c(&self)  -> &MockI2c  { &self.i2c  } }
    impl HasGpio for FanTestRig { type Gpio = MockGpio; fn gpio(&self) -> &MockGpio { &self.gpio } }
    // FanTestRig 获得 FanDiagMixin 但没有 TempMonitorMixin 或 PowerSeqMixin

    #[test]
    fn fan_health_check_passes_above_threshold() {
        let rig = FanTestRig {
            i2c:  MockI2c  { rpm: Cell::new(6000) },
            gpio: MockGpio { level: Cell::new(false) },
        };
        assert!(rig.check_fan_health(0, 4000).unwrap());
    }

    #[test]
    fn fan_health_check_fails_below_threshold() {
        let rig = FanTestRig {
            i2c:  MockI2c  { rpm: Cell::new(2000) },
            gpio: MockGpio { level: Cell::new(false) },
        };
        assert!(!rig.check_fan_health(0, 4000).unwrap());
    }
}
```

注意 `FanTestRig` 只实现了 `HasI2c + HasGpio` —— 它自动获得 `FanDiagMixin`，但编译器**拒绝** `rig.read_thermocouple()`，因为 `HasSpi` 不满足。这是编译时强制的 mixin 作用域。

### 条件方法 —— 超越 Ruby 能做的

添加 `where` bounds 到单个默认方法。方法仅当 associated type 满足额外 bound 时才**存在**：

```rust
/// 支持 DMA 的 SPI 控制器的 marker trait
pub trait DmaCapable: SpiBus {
    fn dma_transfer(&self, tx: &[u8], rx: &mut [u8]) -> io::Result<()>;
}

/// 支持中断的 GPIO 引脚的 marker trait
pub trait InterruptCapable: GpioPin {
    fn wait_for_edge(&self, timeout_ms: u32) -> io::Result<bool>;
}

pub trait AdvancedDiagMixin: HasSpi + HasGpio {
    // 总是可用
    fn basic_probe(&self) -> io::Result<bool> {
        let mut rx = [0u8; 1];
        self.spi().spi_transfer(&[0xFF], &mut rx)?;
        Ok(rx[0] != 0x00)
    }

    // 仅当 SPI 控制器支持 DMA 时存在
    fn bulk_sensor_read(&self, buf: &mut [u8]) -> io::Result<()>
    where
        Self::Spi: DmaCapable,
    {
        self.spi().dma_transfer(&vec![0x00; buf.len()], buf)
    }

    // 仅当 GPIO 引脚支持中断时存在
    fn wait_for_fault_signal(&self, timeout_ms: u32) -> io::Result<bool>
    where
        Self::Gpio: InterruptCapable,
    {
        self.gpio().wait_for_edge(timeout_ms)
    }
}

impl<T: HasSpi + HasGpio> AdvancedDiagMixin for T {}
```

如果你的平台 SPI 不支持 DMA，调用 `bulk_sensor_read()` 是**编译错误**，不是运行时崩溃。Ruby 中最接近的等价物是 `respond_to?` 检查 —— 但它发生在部署时，不是编译时。

### 可组合性：叠加 Mixins

多个 mixins 可以共享相同的 ingredient —— 没有菱形问题：

```text
┌─────────────┐    ┌───────────┐    ┌──────────────┐
│ FanDiagMixin│    │TempMonitor│    │ PowerSeqMixin│
│  (I2C+GPIO) │    │ (SPI+IPMI)│    │  (GPIO+IPMI) │
└──────┬──────┘    └─────┬─────┘    └──────┬───────┘
       │                 │                 │
       │   ┌─────────────┴─────────────┐   │
       └──►│      DiagPlatform         │◄──┘
           │ HasSpi+HasI2c+HasGpio     │
           │        +HasIpmi           │
           └───────────────────────────┘
```

`DiagPlatform` 实现 `HasGpio` **一次**，`FanDiagMixin` 和 `PowerSeqMixin` 都使用相同的 `self.gpio()`。在 Ruby 中，这将是两个模块都调用 `self.gpio_pin` —— 但如果它们期望不同的引脚编号，你将在运行时发现冲突。在 Rust 中，你可以在类型级别消除歧义。

### 对比：Ruby Mixins vs Rust Capability Mixins

| 维度 | Ruby Mixins | Rust Capability Mixins |
|-----------|-------------|------------------------|
| Dispatch | 运行时（方法表查找） | 编译时（单态化） |
| 安全组合 | MRO 线性化隐藏冲突 | 编译器拒绝歧义 |
| 条件方法 | 运行时 `respond_to?` | 编译时 `where` bounds |
| 开销 | 方法 dispatch + GC | 零成本（内联） |
| 可测试性 | 通过元编程 stub/mock | generic 超过 mock 类型 |
| 添加新总线 | 运行时 `include` | 添加 ingredient trait，重新编译 |
| 运行时灵活性 | `extend`、`prepend`、开放类 | 无（完全静态） |

### 何时使用 Capability Mixins

| 场景 | 使用 Mixins？ |
|----------|:-----------:|
| 多个诊断共享总线读取逻辑 | ✅ |
| 测试 harness 需要不同的总线子集 | ✅（部分 ingredient 结构体） |
| 方法仅对某些总线能力有效（DMA、IRQ） | ✅（条件 `where` bounds） |
| 需要运行时模块加载（插件） | ❌（使用 `dyn Trait` 或 enum dispatch） |
| 单个结构体带一个总线 —— 不需要共享 | ❌（保持简单） |
| 带有相干性问题的 cross-crate ingredients | ⚠️（使用 newtype wrappers） |

> **关键要点 —— Capability Mixins**
>
> 1. **Ingredient trait** = associated type + accessor 方法（例如 `HasSpi`）
> 2. **Mixin trait** = ingredient 的 supertrait bounds + 默认方法体
> 3. **Blanket impl** = `impl<T: HasX + HasY> Mixin for T {}` —— 自动注入方法
> 4. **条件方法** = 单个默认方法上的 `where Self::Spi: DmaCapable`
> 5. **部分平台** = 仅实现所需 ingredient 的测试结构体
> 6. **无运行时成本** —— 编译器为每个平台类型生成专用代码

***

## Typed Commands —— GADT 风格的返回类型安全

在 Haskell 中，**广义代数数据类型（GADTs）** 让数据类型的每个构造函数细化类型参数 —— 所以 `Expr Int` 和 `Expr Bool` 被类型检查器强制执行。Rust 没有直接的 GADT 语法，但**带 associated types 的 traits** 实现相同的保证：命令类型**决定**响应类型，搞混它们是编译错误。

这个模式对于硬件诊断特别强大，其中 IPMI 命令、寄存器读取和传感器查询各自返回不同的物理量，绝不应该混淆。

### 问题：未类型的 `Vec<u8>` 沼泽

大多数 C/C++ IPMI 栈 —— 以及天真的 Rust 移植 —— 到处使用原始字节：

```rust
use std::io;

struct BmcConnectionUntyped { timeout_secs: u32 }

impl BmcConnectionUntyped {
    fn raw_command(&self, net_fn: u8, cmd: u8, data: &[u8]) -> io::Result<Vec<u8>> {
        // ... 调用 ipmitool ...
        Ok(vec![0x00, 0x19, 0x00]) // 存根
    }
}

fn diagnose_thermal_untyped(bmc: &BmcConnectionUntyped) -> io::Result<()> {
    // 读取 CPU 温度 —— 传感器 ID 0x20
    let raw = bmc.raw_command(0x04, 0x2D, &[0x20])?;
    let cpu_temp = raw[0] as f64;  // 🤞 希望字节 0 是读数

    // 读取风扇转速 —— 传感器 ID 0x30
    let raw = bmc.raw_command(0x04, 0x2D, &[0x30])?;
    let fan_rpm = raw[0] as u32;  // 🐛 BUG：风扇转速是 2 字节 LE

    // 读取入口电压 —— 传感器 ID 0x40
    let raw = bmc.raw_command(0x04, 0x2D, &[0x40])?;
    let voltage = raw[0] as f64;  // 🐛 BUG：需要除以 1000

    // 🐛 比较 °C 和 RPM —— 编译通过，但无意义
    if cpu_temp > fan_rpm as f64 {
        println!("uh oh");
    }

    // 🐛 传递 Volts 作为温度 —— 编译完全没问题
    log_temp_untyped(voltage);
    log_volts_untyped(cpu_temp);

    Ok(())
}

fn log_temp_untyped(t: f64)  { println!("Temp: {t}°C"); }
fn log_volts_untyped(v: f64) { println!("Voltage: {v}V"); }
```

**每个读数都是 `f64`** —— 编译器不知道哪一个是温度，哪一个是 RPM，哪一个是电压。四个不同的 bug 编译时没有警告：

| # | Bug | 后果 | 发现时间 |
|---|-----|------|----------|
| 1 | 风扇 RPM 解析为 1 字节而不是 2 | 读到 25 RPM 而不是 6400 | 生产环境，凌晨 3 点风扇故障告警 |
| 2 | 电压未除以 1000 | 12000V 而不是 12.0V | 阈值检查标记每个 PSU |
| 3 | 比较 °C 和 RPM | 无意义的布尔值 | 可能永远不会 |
| 4 | 电压传递给 `log_temp_untyped()` | 日志中的静默数据损坏 | 6 个月后，阅读历史记录 |

### 解决方案：通过 Associated Types 的 Typed Commands

#### 步骤 1 —— 领域 newtypes

```rust
#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Celsius(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Rpm(u32);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Volts(f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
struct Watts(f64);
```

#### 步骤 2 —— 命令 trait（GADT 等价物）

Associated type `Response` 是关键 —— 它将每个命令绑定到其返回类型：

```rust
trait IpmiCmd {
    /// GADT "索引" —— 决定 execute() 返回什么。
    type Response;

    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;

    /// 解析被封装在这里 —— 每个命令知道自己的字节布局。
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}
```

#### 步骤 3 —— 每个命令一个结构体，解析只写一次

```rust
struct ReadTemp { sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;  // ← "这个命令返回温度"
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        // 根据 IPMI SDR 有符号字节 —— 只写一次，测试一次
        Ok(Celsius(raw[0] as i8 as f64))
    }
}

struct ReadFanSpeed { fan_id: u8 }
impl IpmiCmd for ReadFanSpeed {
    type Response = Rpm;     // ← "这个命令返回 RPM"
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.fan_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Rpm> {
        // 2 字节 LE —— 正确的布局，只编码一次
        Ok(Rpm(u16::from_le_bytes([raw[0], raw[1]]) as u32))
    }
}

struct ReadVoltage { rail: u8 }
impl IpmiCmd for ReadVoltage {
    type Response = Volts;   // ← "这个命令返回电压"
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.rail] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Volts> {
        // 毫伏 → 伏特，总是正确
        Ok(Volts(u16::from_le_bytes([raw[0], raw[1]]) as f64 / 1000.0))
    }
}

struct ReadFru { fru_id: u8 }
impl IpmiCmd for ReadFru {
    type Response = String;
    fn net_fn(&self) -> u8 { 0x0A }
    fn cmd_byte(&self) -> u8 { 0x11 }
    fn payload(&self) -> Vec<u8> { vec![self.fru_id, 0x00, 0x00, 0xFF] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<String> {
        Ok(String::from_utf8_lossy(raw).to_string())
    }
}
```

#### 步骤 4 —— 执行器（零 `dyn`，单态化）

```rust
struct BmcConnection { timeout_secs: u32 }

impl BmcConnection {
    /// Generic 超过任何命令 —— 编译器为每个命令类型生成一个版本。
    fn execute<C: IpmiCmd>(&self, cmd: &C) -> io::Result<C::Response> {
        let raw = self.raw_send(cmd.net_fn(), cmd.cmd_byte(), &cmd.payload())?;
        cmd.parse_response(&raw)
    }

    fn raw_send(&self, _nf: u8, _cmd: u8, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![0x19, 0x00]) // 存根 —— 真正的实现调用 ipmitool
    }
}
```

#### 步骤 5 —— 调用者代码：所有四个 bug 都变成编译错误

```rust
fn diagnose_thermal(bmc: &BmcConnection) -> io::Result<()> {
    let cpu_temp: Celsius = bmc.execute(&ReadTemp { sensor_id: 0x20 })?;
    let fan_rpm:  Rpm     = bmc.execute(&ReadFanSpeed { fan_id: 0x30 })?;
    let voltage:  Volts   = bmc.execute(&ReadVoltage { rail: 0x40 })?;

    // Bug #1 —— 不可能：解析住在 ReadFanSpeed::parse_response
    // Bug #2 —— 不可能：缩放住在 ReadVoltage::parse_response

    // Bug #3 —— 编译错误：
    // if cpu_temp > fan_rpm { }
    //    ^^^^^^^^   ^^^^^^^
    //    Celsius    Rpm      → "类型不匹配" ❌

    // Bug #4 —— 编译错误：
    // log_temperature(voltage);
    //                 ^^^^^^^  Volts, 期望 Celsius ❌

    // 只有正确的比较编译：
    if cpu_temp > Celsius(85.0) {
        println!("CPU 过热：{:?}", cpu_temp);
    }
    if fan_rpm < Rpm(4000) {
        println!("风扇太慢：{:?}", fan_rpm);
    }

    Ok(())
}

fn log_temperature(t: Celsius) { println!("Temp: {:?}", t); }
fn log_voltage(v: Volts)       { println!("Voltage: {:?}", v); }
```

### 诊断脚本的宏 DSL

对于按顺序运行许多命令的大型诊断例程，宏提供简洁的声明式语法，同时保留完整的类型安全：

```rust
/// 执行一系列 typed IPMI 命令，返回结果元组。
/// 元组的每个元素都有命令自己的 Response 类型。
macro_rules! diag_script {
    ($bmc:expr; $($cmd:expr),+ $(,)?) => {{
        ( $( $bmc.execute(&$cmd)?, )+ )
    }};
}

fn full_pre_flight(bmc: &BmcConnection) -> io::Result<()> {
    // 展开为：(Celsius, Rpm, Volts, String) —— 每个类型都被跟踪
    let (temp, rpm, volts, board_pn) = diag_script!(bmc;
        ReadTemp     { sensor_id: 0x20 },
        ReadFanSpeed { fan_id:    0x30 },
        ReadVoltage  { rail:      0x40 },
        ReadFru      { fru_id:    0x00 },
    );

    println!("Board: {:?}", board_pn);
    println!("CPU: {:?}, Fan: {:?}, 12V: {:?}", temp, rpm, volts);

    // 类型安全的阈值检查：
    assert!(temp  < Celsius(95.0), "CPU 太热");
    assert!(rpm   > Rpm(3000),     "风扇太慢");
    assert!(volts > Volts(11.4),   "12V rail 下降");

    Ok(())
}
```

宏只是语法糖 —— 元组类型 `(Celsius, Rpm, Volts, String)` 完全由编译器推断。交换两个命令，解构在编译时失败，而不是运行时。

### Enum Dispatch 用于异构命令列表

当你需要混合命令的 `Vec`（例如，从 JSON 加载的可配置脚本），使用 enum dispatch 保持无 `dyn`：

```rust
enum AnyReading {
    Temp(Celsius),
    Rpm(Rpm),
    Volt(Volts),
    Text(String),
}

enum AnyCmd {
    Temp(ReadTemp),
    Fan(ReadFanSpeed),
    Voltage(ReadVoltage),
    Fru(ReadFru),
}

impl AnyCmd {
    fn execute(&self, bmc: &BmcConnection) -> io::Result<AnyReading> {
        match self {
            AnyCmd::Temp(c)    => Ok(AnyReading::Temp(bmc.execute(c)?)),
            AnyCmd::Fan(c)     => Ok(AnyReading::Rpm(bmc.execute(c)?)),
            AnyCmd::Voltage(c) => Ok(AnyReading::Volt(bmc.execute(c)?)),
            AnyCmd::Fru(c)     => Ok(AnyReading::Text(bmc.execute(c)?)),
        }
    }
}

/// 动态诊断脚本 —— 命令在运行时加载
fn run_script(bmc: &BmcConnection, script: &[AnyCmd]) -> io::Result<Vec<AnyReading>> {
    script.iter().map(|cmd| cmd.execute(bmc)).collect()
}
```

你失去了每个元素的类型跟踪（一切都是 `AnyReading`），但获得了运行时灵活性 —— 解析仍然封装在每个 `IpmiCmd` 实现中。

### 测试 Typed Commands

```rust
#[cfg(test)]
mod tests {
    use super::*;

    struct StubBmc {
        responses: std::collections::HashMap<u8, Vec<u8>>,
    }

    impl StubBmc {
        fn execute<C: IpmiCmd>(&self, cmd: &C) -> io::Result<C::Response> {
            let key = cmd.payload()[0]; // 传感器 ID 作为 key
            let raw = self.responses.get(&key)
                .ok_or_else(|| io::Error::new(io::ErrorKind::NotFound, "no stub"))?;
            cmd.parse_response(raw)
        }
    }

    #[test]
    fn read_temp_parses_signed_byte() {
        let bmc = StubBmc {
            responses: [( 0x20, vec![0xE7] )].into() // -25 作为 i8 = 0xE7
        };
        let temp = bmc.execute(&ReadTemp { sensor_id: 0x20 }).unwrap();
        assert_eq!(temp, Celsius(-25.0));
    }

    #[test]
    fn read_fan_parses_two_byte_le() {
        let bmc = StubBmc {
            responses: [( 0x30, vec![0x00, 0x19] )].into() // 0x1900 = 6400
        };
        let rpm = bmc.execute(&ReadFanSpeed { fan_id: 0x30 }).unwrap();
        assert_eq!(rpm, Rpm(6400));
    }

    #[test]
    fn read_voltage_scales_millivolts() {
        let bmc = StubBmc {
            responses: [( 0x40, vec![0xE8, 0x2E] )].into() // 0x2EE8 = 12008 mV
        };
        let v = bmc.execute(&ReadVoltage { rail: 0x40 }).unwrap();
        assert!((v.0 - 12.008).abs() < 0.001);
    }
}
```

每个命令的解析被独立测试。如果 `ReadFanSpeed` 在新的 IPMI 规范修订中从 2 字节 LE 改为 4 字节 BE，你只需更新**一个** `parse_response`，测试捕捉回归。

### 这如何映射到 Haskell GADTs

```text
Haskell GADT                         Rust 等价物
────────────────                     ───────────────────────
data Cmd a where                     trait IpmiCmd {
  ReadTemp :: SensorId -> Cmd Temp       type Response;
  ReadFan  :: FanId    -> Cmd Rpm        ...
                                     }

eval :: Cmd a -> IO a                fn execute<C: IpmiCmd>(&self, cmd: &C)
                                         -> io::Result<C::Response>

类型细化在 case 分支中           单态化：编译器生成
                                     execute::<ReadTemp>() → 返回 Celsius
                                     execute::<ReadFanSpeed>() → 返回 Rpm
```

两者保证：**命令决定返回类型**。Rust 通过 generic 单态化实现它，而不是类型级 case 分析 —— 相同的安全性，零运行时成本。

### 前后对比总结

| 维度 | 未类型 (`Vec<u8>`) | Typed Commands |
|-----------|:---:|:---:|
| 每传感器行数 | ~3（在每个调用点重复） | ~15（只写和测试一次） |
| 可能的解析错误 | 在每个调用点 | 在一个 `parse_response` 实现中 |
| 单位混淆 bug | 无限 | 零（编译错误） |
| 添加新传感器 | 触摸 N 个文件，复制粘贴解析 | 添加 1 个结构体 + 1 个 impl |
| 运行时成本 | — | 相同（单态化） |
| IDE 自动补全 | 到处都是 `f64` | `Celsius`、`Rpm`、`Volts` —— 自文档化 |
| 代码审查负担 | 必须验证每个原始字节解析 | 验证每个传感器一个 `parse_response` |
| 宏 DSL | N/A | `diag_script!(bmc; ReadTemp{..}, ReadFan{..})` → `(Celsius, Rpm)` |
| 动态脚本 | 手动 dispatch | `AnyCmd` enum —— 仍然无 `dyn` |

### 何时使用 Typed Commands

| 场景 | 推荐 |
|----------|:--------------:|
| IPMI 传感器读取带不同的物理单位 | ✅ Typed commands |
| 寄存器映射带不同宽度的字段 | ✅ Typed commands |
| 网络协议消息（请求 → 响应） | ✅ Typed commands |
| 单个命令类型带一个返回格式 | ❌ 过度 —— 直接返回类型 |
| 原型设计/探索未知设备 | ❌ 先用原始字节，后类型化 |
| 插件系统中命令在编译时未知 | ⚠️ 使用 `AnyCmd` enum dispatch |

> **关键要点 —— Traits**
> - Associated types = 每个类型一个实现；generic 参数 = 每个类型多个实现
> - GATs 解锁 lending 迭代器和 async-in-traits 模式
> - 对封闭集使用 enum dispatch（快）；`dyn Trait` 用于开放集（灵活）
> - `Any` + `TypeId` 是当编译时类型未知时的逃逸出口

> **另见：** [第 1 章 — Generics](ch01-generics-the-full-picture.md) 了解单态化和何时 generics 导致代码膨胀。[第 3 章 — Newtype & Type-State](ch03-the-newtype-and-type-state-patterns.md) 了解如何在 config trait 模式中使用 traits。

---

### 练习：带 Associated Types 的 Repository ★★★（约 40 分钟）

设计一个 `Repository` trait，具有 associated `Error`、`Id` 和 `Item` 类型。为内存存储实现它并演示编译时类型安全。

<details>
<summary>🔑 解答</summary>

```rust
use std::collections::HashMap;

trait Repository {
    type Item;
    type Id;
    type Error;

    fn get(&self, id: &Self::Id) -> Result<Option<&Self::Item>, Self::Error>;
    fn insert(&mut self, item: Self::Item) -> Result<Self::Id, Self::Error>;
    fn delete(&mut self, id: &Self::Id) -> Result<bool, Self::Error>;
}

#[derive(Debug, Clone)]
struct User {
    name: String,
    email: String,
}

struct InMemoryUserRepo {
    data: HashMap<u64, User>,
    next_id: u64,
}

impl InMemoryUserRepo {
    fn new() -> Self {
        InMemoryUserRepo { data: HashMap::new(), next_id: 1 }
    }
}

impl Repository for InMemoryUserRepo {
    type Item = User;
    type Id = u64;
    type Error = std::convert::Infallible;

    fn get(&self, id: &u64) -> Result<Option<&User>, Self::Error> {
        Ok(self.data.get(id))
    }

    fn insert(&mut self, item: User) -> Result<u64, Self::Error> {
        let id = self.next_id;
        self.next_id += 1;
        self.data.insert(id, item);
        Ok(id)
    }

    fn delete(&mut self, id: &u64) -> Result<bool, Self::Error> {
        Ok(self.data.remove(id).is_some())
    }
}

fn create_and_fetch<R: Repository>(repo: &mut R, item: R::Item) -> Result<(), R::Error>
where
    R::Item: std::fmt::Debug,
    R::Id: std::fmt::Debug,
{
    let id = repo.insert(item)?;
    println!("Inserted with id: {id:?}");
    let retrieved = repo.get(&id)?;
    println!("Retrieved: {retrieved:?}");
    Ok(())
}

fn main() {
    let mut repo = InMemoryUserRepo::new();
    create_and_fetch(&mut repo, User {
        name: "Alice".into(),
        email: "alice@example.com".into(),
    }).unwrap();
}
```

</details>

***

***
