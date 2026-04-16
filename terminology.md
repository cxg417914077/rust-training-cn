# Rust 中文术语表

本术语表收录 Rust Training 中文翻译项目中使用的标准术语翻译，确保翻译一致性。

## A

| 英文 | 翻译 | 备注 |
|------|------|------|
| Abstract Syntax Tree (AST) | 抽象语法树 | 编译器内部结构 |
| Allocation | 分配 | 内存分配 |
| Alias | 别名 | 引用别名 |
| Align | 对齐 | 内存对齐 |
| Array | 数组 | 固定长度数组 |
| Arrow | 箭头 | -> 符号 |
| Assertion | 断言 | debug_assert! 等 |
| Assignment | 赋值 | 变量赋值 |
| Associated Type | 关联类型 | trait 中的类型 |
| Async/Await | 异步/等待 | 异步编程关键字 |
| Atomic | 原子操作 | 不可分割的操作 |
| Attribute | 属性 | #[attribute] 语法 |
| Aurora | 极光 | - |
| Autoderef | 自动解引用 | 自动 deref 转换 |
| Arc | 原子引用计数 | 线程安全的 Rc |

## B

| 英文 | 翻译 | 备注 |
|------|------|------|
| Binding | 绑定 | 变量绑定 |
| Bit | 位 | 最小数据单位 |
| Bitwise | 按位 | 位运算 |
| Block | 块 | {} 包裹的代码块 |
| Borrow | 借用 | & 和 &mut |
| Borrow Checker | 借用检查器 | 编译器组件 |
| Box | 堆分配指针 | Box<T> |
| Brace | 花括号 | {} |
| Branch | 分支 | if/else 分支 |
| Break | 中断 | break 关键字 |
| Buffer | 缓冲区 | 数据缓冲 |
| Build | 构建 | 编译构建 |
| Byte | 字节 | 8 位 |

## C

| 英文 | 翻译 | 备注 |
|------|------|------|
| Call | 调用 | 函数调用 |
| Capture | 捕获 | 闭包捕获变量 |
| Cast | 类型转换 | as 关键字 |
| Channel | 通道 | 线程间通信 |
| Claim | 声明 | - |
| Clause | 子句 | where 子句 |
| Closure | 闭包 | 匿名函数 |
| Coercion | 强制转换 | 隐式类型转换 |
| Column | 列 | - |
| Combinator | 组合器 | 函数组合 |
| Command | 命令 | CLI 命令 |
| Comment | 注释 | // 或 /* */ |
| Compile | 编译 | 源代码到二进制 |
| Compiler | 编译器 | rustc |
| Compose | 组合 | 函数组合 |
| Concurrency | 并发 | 并发执行 |
| Condition | 条件 | if 条件 |
| Constant | 常量 | const 定义 |
| Constraint | 约束 | 泛型约束 |
| Context | 上下文 | 错误上下文等 |
| Continue | 继续 | continue 关键字 |
| Count | 计数 | 数量统计 |
| Crate | Crate | Rust 编译单元 |
| Cross-Compilation | 交叉编译 | 跨平台编译 |
| Cursor | 游标 | - |

## D

| 英文 | 翻译 | 备注 |
|------|------|------|
| Data | 数据 | 数据值 |
| Deallocation | 释放 | 内存释放 |
| Debug | 调试 | 调试信息 |
| Declaration | 声明 | 函数/变量声明 |
| Default | 默认 | Default trait |
| Definition | 定义 | 完整定义 |
| Dereference | 解引用 | * 操作符 |
| Derive | 派生 | #[derive] |
| Destination | 目标 | 目标位置 |
| Destructor | 析构函数 | Drop trait |
| Diagram | 图表 | 流程图等 |
| Dispatch | 分发 | 动态/静态分发 |
| Documentation | 文档 | /// 文档注释 |
| Domain | 域 | 定义域 |
| Drop | 丢弃 | Drop trait |
| Duck Typing | 鸭子类型 | Python 风格 |
| Dynamic | 动态 | 运行时 |

## E

| 英文 | 翻译 | 备注 |
|------|------|------|
| Element | 元素 | 集合元素 |
| Else | 否则 | else 分支 |
| Empty | 空 | 空值 |
| Encode | 编码 | 数据编码 |
| Enum | 枚举 | enum 定义 |
| Entry | 条目 | 入口点 |
| Equality | 相等 | == 比较 |
| Error | 错误 | 错误处理 |
| Escape | 转义 | 转义字符 |
| Evaluate | 求值 | 表达式求值 |
| Event | 事件 | 事件处理 |
| Example | 示例 | 代码示例 |
| Exception | 异常 | Python 异常 |
| Execute | 执行 | 程序执行 |
| Executor | 执行器 | 异步执行器 |
| Export | 导出 | pub 导出 |
| Expression | 表达式 | 代码表达式 |
| Extend | 扩展 | 类型扩展 |
| External | 外部 | 外部 crate |

## F

| 英文 | 翻译 | 备注 |
|------|------|------|
| False | 假 | false 值 |
| Field | 字段 | 结构体字段 |
| File | 文件 | 源文件 |
| Filter | 过滤 | 迭代器过滤 |
| Finally | 最终 | - |
| Flag | 标志 | 标志位 |
| Flow | 流 | 控制流 |
| Fn | 函数 trait | Fn/FnMut/FnOnce |
| For | 对于 | for 循环 |
| Format | 格式化 | format! 宏 |
| Frame | 帧 | 栈帧 |
| Free | 释放 | 内存释放 |
| From | 从 | From trait |
| FromIterator | 从迭代器构建 | FromIterator trait |
| Full | 完整 | 完全 |
| Function | 函数 | fn 定义 |
| Future | 未来值 | 异步 Future |

## G

| 英文 | 翻译 | 备注 |
|------|------|------|
| Garbage Collection | 垃圾回收 | GC |
| Generic | 泛型 | 泛型编程 |
| Global | 全局 | 全局变量 |
| Goroutine | 协程 | Go 协程 |
| Grammar | 语法 | 语法规则 |
| Guard | 守卫 | 模式守卫 |
| Guide | 指南 | 教程指南 |

## H

| 英文 | 翻译 | 备注 |
|------|------|------|
| Handle | 句柄 | 资源句柄 |
| Handler | 处理器 | 事件处理器 |
| Hash | 哈希 | 哈希函数 |
| Heap | 堆 | 堆内存 |
| Higher-Ranked | 高阶 | 高阶 trait |
| Hint | 提示 | 编译器提示 |
| Hole | 空洞 | 类型空洞 |

## I

| 英文 | 翻译 | 备注 |
|------|------|------|
| Identifier | 标识符 | 变量/函数名 |
| If | 如果 | if 条件 |
| Ignore | 忽略 | 忽略错误 |
| Immutable | 不可变 | 只读 |
| Implement | 实现 | impl 实现 |
| Implementation | 实现 | 实现细节 |
| Import | 导入 | use 导入 |
| In | 在...中 | in 关键字 |
| Inclusive | 包含 | 范围包含 |
| Index | 索引 | 下标索引 |
| Infer | 推断 | 类型推断 |
| Infinite | 无限 | 无限循环 |
| Inference | 推断 | 类型推断 |
| Initialize | 初始化 | 变量初始化 |
| Inline | 内联 | #[inline] |
| Input | 输入 | 输入数据 |
| Instance | 实例 | 类型实例 |
| Instantiate | 实例化 | 创建实例 |
| Integer | 整数 | 整型 |
| Interior | 内部 | 内部可变性 |
| Interface | 接口 | 类型接口 |
| Into | 转换为 | Into trait |
| Invalid | 无效 | 无效值 |
| Iterate | 迭代 | 循环迭代 |
| Iterator | 迭代器 | Iterator trait |

## K

| 英文 | 翻译 | 备注 |
|------|------|------|
| Key | 键 | HashMap 键 |
| Keyword | 关键字 | 保留关键字 |

## L

| 英文 | 翻译 | 备注 |
|------|------|------|
| Label | 标签 | 循环标签 |
| Lambda | Lambda 表达式 | Python lambda |
| Leaf | 叶子 | 树结构叶子 |
| Let | 让 | let 绑定 |
| Lexical | 词法 | 词法作用域 |
| Lifetime | 生命周期 | 'a 生命周期 |
| Link | 链接 | 链接库 |
| List | 列表 | Python list |
| Literal | 字面量 | 字符串/数字字面量 |
| Local | 局部 | 局部变量 |
| Locate | 定位 | 位置查找 |
| Lock | 锁 | 互斥锁 |
| Log | 日志 | 日志记录 |
| Loop | 循环 | loop 循环 |
| Lossy | 有损 | 有损转换 |

## M

| 英文 | 翻译 | 备注 |
|------|------|------|
| Macro | 宏 | 宏定义 |
| Main | 主函数 | main 函数 |
| Manual | 手册 | 参考手册 |
| Many | 多个 | 多个值 |
| Map | 映射 | Iterator map |
| Match | 匹配 | match 表达式 |
| Memory | 内存 | 内存管理 |
| Message | 消息 | 线程消息 |
| Method | 方法 | 对象方法 |
| Mixin | 混入 | 混入类 |
| Mode | 模式 | 运行模式 |
| Modify | 修改 | 变量修改 |
| Module | 模块 | mod 模块 |
| Monomorphization | 单态化 | 泛型单态化 |
| Move | 移动 | 所有权移动 |
| Multi | 多 | 多线程等 |
| Mutable | 可变 | mut 可变 |
| Mutex | 互斥锁 | Mutex 锁 |

## N

| 英文 | 翻译 | 备注 |
|------|------|------|
| Name | 名称 | 标识符名称 |
| Namespace | 命名空间 | 模块命名空间 |
| Native | 原生 | 原生代码 |
| Nested | 嵌套 | 嵌套结构 |
| Never | 从不 | ! 类型 |
| New | 新建 | new 方法 |
| Next | 下一个 | Iterator next |
| No | 无 | no_std 等 |
| Node | 节点 | 树节点 |
| None | 无 | Option::None |
| Non-Zero | 非零 | 非零优化 |
| Not | 非 | ! 操作符 |
| Null | 空指针 | 空引用 |
| Number | 数字 | 数值类型 |

## O

| 英文 | 翻译 | 备注 |
|------|------|------|
| Object | 对象 | 运行时对象 |
| Offset | 偏移 | 内存偏移 |
| Once | 一次 | Once 初始化 |
| Opaque | 不透明 | 不透明类型 |
| Operand | 操作数 | 运算对象 |
| Operation | 操作 | 运算操作 |
| Operator | 操作符 | + - * / 等 |
| Option | 选项 | Option 类型 |
| Or | 或 | || 操作符 |
| Order | 顺序 | 内存序 |
| Output | 输出 | 函数输出 |
| Owner | 所有者 | 所有权拥有者 |
| Ownership | 所有权 | Rust 核心概念 |

## P

| 英文 | 翻译 | 备注 |
|------|------|------|
| Package | 包 | Cargo 包 |
| Panic | 恐慌 | panic! 宏 |
| Parallel | 并行 | 并行执行 |
| Parameter | 参数 | 函数参数 |
| Parent | 父级 | 父节点 |
| Parse | 解析 | 字符串解析 |
| Partial | 部分 | PartialOrd 等 |
| Path | 路径 | 模块路径 |
| Pattern | 模式 | 模式匹配 |
| Phantom | 幻象 | PhantomData |
| Pin | 固定 | Pin 类型 |
| Pipe | 管道 | 进程管道 |
| Placeholder | 占位符 | _ 占位符 |
| Platform | 平台 | 目标平台 |
| Pointer | 指针 | 内存指针 |
| Polymorphism | 多态 | 类型多态 |
| Pool | 池 | 线程池等 |
| Position | 位置 | 索引位置 |
| Power | 幂 | 幂运算 |
| Practical | 实践 | 实践指南 |
| Prelude | 前奏 | std::prelude |
| Primitive | 原始 | 原始类型 |
| Print | 打印 | println! |
| Private | 私有 | private 字段 |
| Procedure | 过程 | 过程宏 |
| Process | 进程 | 操作系统进程 |
| Profile | 配置 | 构建配置 |
| Program | 程序 | 计算机程序 |
| Project | 项目 | Cargo 项目 |
| Property | 属性 | 对象属性 |
| Protocol | 协议 | PEP 544 协议 |
| Public | 公有 | pub 公开 |
| Push | 推送 | push 方法 |
| Python | Python | Python 语言 |

## R

| 英文 | 翻译 | 备注 |
|------|------|------|
| Range | 范围 | 范围类型 |
| Raw | 原始 | 裸指针 |
| Read | 读取 | 文件读取 |
| Receiver | 接收者 | self 接收者 |
| Record | 记录 | 数据记录 |
| Recursion | 递归 | 递归调用 |
| Reduce | 归约 | Iterator reduce |
| Reference | 引用 | & 引用 |
| Reflexive | 自反 | 等价关系 |
| Region | 区域 | 生命周期区域 |
| Register | 注册 | 注册表 |
| Regular | 常规 | 常规类型 |
| Relation | 关系 | 类型关系 |
| Release | 释放 | 资源释放 |
| Repr | 表示 | #[repr] |
| Require | 要求 | trait 要求 |
| Reserve | 预留 | 容量预留 |
| Reset | 重置 | 重置状态 |
| Resolve | 解析 | 名称解析 |
| Resource | 资源 | 系统资源 |
| Result | 结果 | Result 类型 |
| Return | 返回 | return 关键字 |
| Reverse | 反向 | 反向迭代 |
| Right | 右 | 右值 |
| Role | 角色 | 类型角色 |
| Root | 根 | 根节点 |
| Row | 行 | 数据行 |
| Rule | 规则 | 匹配规则 |
| Run | 运行 | 程序运行 |
| Runtime | 运行时 | 运行时环境 |
| Rust | Rust | Rust 语言 |

## S

| 英文 | 翻译 | 备注 |
|------|------|------|
| Safe | 安全 | 安全代码 |
| Safety | 安全性 | unsafe 安全性 |
| Sample | 示例 | 示例代码 |
| Scalar | 标量 | 标量类型 |
| Schedule | 调度 | 线程调度 |
| Scope | 作用域 | 词法作用域 |
| Send | 发送 | Send trait |
| Sequence | 序列 | 序列类型 |
| Set | 集合 | HashSet 等 |
| Shadow | 遮蔽 | 变量遮蔽 |
| Shared | 共享 | 共享引用 |
| Slice | 切片 | &[T] 切片 |
| Some | 一些 | Option::Some |
| Sort | 排序 | 数据排序 |
| Source | 来源 | 源代码 |
| Spawn | 创建 | 创建线程 |
| Specialization | 特化 | trait 特化 |
| Stack | 栈 | 栈内存 |
| Stale | 过时 | 过期引用 |
| Standard | 标准 | 标准库 |
| Statement | 语句 | 代码语句 |
| Static | 静态 | 'static 生命周期 |
| Status | 状态 | 运行状态 |
| Step | 步骤 | 迭代步骤 |
| Storage | 存储 | 数据存储 |
| Str | 字符串 | &str 类型 |
| String | 字符串 | String 类型 |
| Struct | 结构体 | struct 定义 |
| Style | 风格 | 代码风格 |
| Sub | 子 | 子类型 |
| Summary | 总结 | 章节总结 |
| Super | 超 | super 模块 |
| Support | 支持 | 特性支持 |
| Sync | 同步 | Sync trait |
| Syntax | 语法 | 语法规则 |
| System | 系统 | 操作系统 |

## T

| 英文 | 翻译 | 备注 |
|------|------|------|
| Table | 表 | 虚函数表 |
| Take | 获取 | take 方法 |
| Task | 任务 | 异步任务 |
| Target | 目标 | 编译目标 |
| Template | 模板 | 代码模板 |
| Test | 测试 | 单元测试 |
| Thread | 线程 | 操作系统线程 |
| Throughput | 吞吐量 | 性能指标 |
| Throw | 抛出 | 抛出异常 |
| Time | 时间 | 时间类型 |
| Timeout | 超时 | 超时处理 |
| Token | 令牌 | 词法令牌 |
| Trait | 特征 | Rust trait |
| Transaction | 事务 | 数据库事务 |
| Transform | 转换 | 数据转换 |
| True | 真 | true 值 |
| Try | 尝试 | try 块 |
| Tuple | 元组 | 元组类型 |
| Type | 类型 | 数据类型 |
| Typestate | 类型状态 | 类型状态模式 |

## U

| 英文 | 翻译 | 备注 |
|------|------|------|
| Unchecked | 未检查 | 未检查操作 |
| Undefined | 未定义 | 未定义行为 |
| Underscore | 下划线 | _ 符号 |
| Uniform | 统一 | 统一类型 |
| Union | 联合 | 联合类型 |
| Unit | 单元 | () 单元类型 |
| Universal | 通用 | 通用类型 |
| Unknow | 未知 | 未知类型 |
| Unsafe | 不安全 | unsafe 代码 |
| Unwrap | 解包 | unwrap 方法 |
| Update | 更新 | 变量更新 |
| Upper | 上 | 上界 |
| Use | 使用 | use 导入 |
| User | 用户 | 用户定义 |
| Utility | 工具 | 工具函数 |

## V

| 英文 | 翻译 | 备注 |
|------|------|------|
| Value | 值 | 数据值 |
| Variable | 变量 | 可变变量 |
| Variant | 变体 | 枚举变体 |
| Vector | 向量 | Vec<T> |
| Version | 版本 | 版本号 |
| View | 视图 | 数据视图 |
| Virtual | 虚拟 | 虚函数 |
| Visibility | 可见性 | pub 可见性 |
| Void | 空 | 空类型 |

## W

| 英文 | 翻译 | 备注 |
|------|------|------|
| Wait | 等待 | 等待完成 |
| Warning | 警告 | 编译警告 |
| Weak | 弱 | Weak 引用 |
| Where | 其中 | where 子句 |
| While | 当 | while 循环 |
| Wildcard | 通配符 | _ 通配符 |
| Window | 窗口 | 时间窗口 |
| With | 带有 | with 语句 |
| Without | 无 | 无 GIL |
| Wrap | 包装 | 包装类型 |
| Write | 写入 | 文件写入 |

## Z

| 英文 | 翻译 | 备注 |
|------|------|------|
| Zero | 零 | 零成本 |
| Zero-Cost | 零成本 | 零成本抽象 |
| Zip | 压缩 | Iterator zip |

---

> **注意**：本术语表会持续更新，如发现术语翻译不一致或有更好的翻译建议，欢迎提交 Issue 或 Pull Request。
