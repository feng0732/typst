# WASM 插件加载与调用边界深度分析

本文档从代码层面深入解析 Typst 中 WebAssembly 插件系统的接口契约、数据传递链路和失败处理机制。

---

## 1. 插件接口层：加载、导出、导入与 Transition

### 1.1 插件加载入口

插件加载的唯一入口是全局 `plugin()` 函数，定义于 [plugin.rs#L148-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs#L148-L156)。

```rust
#[func(scope)]
pub fn plugin(
    engine: &mut Engine,
    source: Spanned<DataSource>,
) -> SourceResult<Module> {
    let loaded = source.load(engine.world)?;
    Plugin::module(loaded.data).at(source.span)
}
```

**调用流程**：

| 步骤 | 说明 | 关键代码 |
|------|------|----------|
| 1 | 接收 `Spanned<DataSource>`，即带位置信息的数据源（路径或原始字节） | `DataSource` 枚举定义于 [loading/mod.rs#L48-L53](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/loading/mod.rs#L48-L53) |
| 2 | `source.load(engine.world)` 通过 `World` trait 读取字节 | `Load` trait 实现在 [loading/mod.rs#L82-L112](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/loading/mod.rs#L82-L112) |
| 3 | `Plugin::module(bytes)` 编译 WASM 并包装为 `Module` | 带 `#[comemo::memoize]` 缓存 |
| 4 | 返回值为 `Module`，内含所有 WASM 导出函数作为 `PluginFunc` | `Plugin::into_module()` 于 [plugin.rs#L365-L380](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs#L365-L380) |

### 1.2 模块构建：从 WASM 导出到 Typst 函数

`Plugin::into_module()` 遍历 WASM 模块的所有导出，将类型为函数的导出逐一包装：

```rust
fn into_module(self) -> Module {
    let shared = Arc::new(self);
    let mut scope = Scope::new();
    for export in shared.base.module.exports() {
        if matches!(export.ty(), wasmi::ExternType::Func(_)) {
            let name = EcoString::from(export.name());
            let func = PluginFunc { plugin: shared.clone(), name: name.clone() };
            scope.bind(name, Binding::detached(Func::from(func)));
        }
    }
    Module::anonymous(scope)
}
```

**边界要点**：

- 仅 `ExternType::Func` 类型的导出会被收集，`memory`、`table` 等导出被忽略
- 每个导出函数被包装为 `PluginFunc`，再通过 `From<PluginFunc> for Func` 转为通用的 `Func` 值
- 所有 `PluginFunc` 共享同一个 `Arc<Plugin>`，即共享字节码、链接器和实例池
- 结果是一个匿名 `Module`，可直接用 `#import plugin("x.wasm"): func_name` 语法解构

### 1.3 WASM 侧必须实现的协议

#### 1.3.1 导出要求

WASM 模块**必须**导出：
- `memory`（名称固定为 "memory"）—— 用于读写参数和返回值
- 任意数量的函数，签名为 `(param i32 ...) -> (result i32)`

#### 1.3.2 导入要求（宿主提供）

WASM 模块**必须**从 `"typst_env"` 模块导入以下两个函数：

```wat
(import "typst_env" "wasm_minimal_protocol_write_args_to_buffer"
    (func (param i32)))

(import "typst_env" "wasm_minimal_protocol_send_result_to_host"
    (func (param i32 i32)))
```

这两个函数在 Rust 侧的注册位于 [plugin.rs#L283-L297](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs#L283-L297)：

```rust
let mut linker = wasmi::Linker::new(&engine);
linker.func_wrap(
    "typst_env",
    "wasm_minimal_protocol_send_result_to_host",
    wasm_minimal_protocol_send_result_to_host,
).unwrap();
linker.func_wrap(
    "typst_env",
    "wasm_minimal_protocol_write_args_to_buffer",
    wasm_minimal_protocol_write_args_to_buffer,
).unwrap();
```

**宿主导入函数的职责**：

| 导入函数 | 宿主行为 |
|----------|----------|
| `write_args_to_buffer(ptr)` | 从 `CallData.args` 取出所有字节参数，顺序写入 WASM 内存 `ptr` 起始处 |
| `send_result_to_host(ptr, len)` | 从 WASM 内存 `ptr` 处读取 `len` 字节，存入 `CallData.output` |

### 1.4 Transition API：有副作用的安全调用

由于 Typst 要求函数纯净化（`PluginFunc::call` 被 `#[comemo::memoize]` 标记），插件不能通过普通调用来修改内部状态。Transition API 是为此设计的受控"逃逸口"。

定义于 [plugin.rs#L192-L201](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs#L192-L201)：

```rust
#[func]
pub fn transition(
    func: PluginFunc,
    #[variadic] arguments: Vec<Bytes>,
) -> StrResult<Module> {
    func.transition(arguments)
}
```

**Transition 的本质**：

1. 执行一次可变调用，允许 WASM 实例修改自身内存
2. 对修改后的实例做完整内存快照（`Snapshot { mem_pages, mem_data }`）
3. 创建一个**新的 `Plugin`**，携带此快照和一个新指纹 `fingerprint`
4. 新 `Plugin` 的后续实例都从快照恢复，因此能观察到 mutation
5. 原 `Plugin` 完全不受影响（使用过的实例被移动到新 Plugin，不返回原池）

```rust
fn transition(&self, func: &str, args: Vec<Bytes>) -> StrResult<Plugin> {
    let fingerprint = typst_utils::hash128(&(self.fingerprint, func, &args));
    let mut instance = self.acquire()?;
    instance.call(func, args)?;
    let snapshot = instance.snapshot();
    Ok(Self {
        base: self.base.clone(),
        snapshot: Some(snapshot),
        fingerprint,
        pool: Mutex::new(vec![instance]),
    })
}
```

**关键边界**：
- `fingerprint` 是 `(原始bytes, func名, 参数)` 的链式哈希，保证 comemo 能正确区分不同 transition 产生的 Plugin
- 仅快照**线性内存**，不快照 WASM globals（文档中明确标注的限制）
- `Plugin` 的 `PartialEq` 和 `Hash` 同时使用 `base.bytes` 和 `fingerprint`

---

## 2. 数据传递层：从 Typst 值到 WASM 内存

### 2.1 完整调用链路

以下是一次 `p.my_func(bytes("a"), bytes("b"))` 调用的完整数据流：

```
┌─────────────────────────────────────────────────────────────────────┐
│ Typst 层 (func.rs)                                                  │
│   Func::call_impl()                                                 │
│     → match FuncInner::Plugin(func)                                 │
│     → args.all::<Bytes>()?          // 提取所有位置参数，强转为Bytes│
│     → func.call(inputs)             // PluginFunc::call (带memoize)  │
│     → Ok(Value::Bytes(output))      // 结果包装为Value               │
└───────────────────────┬─────────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 插件调度层 (plugin.rs: Plugin)                                      │
│   Plugin::call()                                                    │
│     → self.acquire()                // 从池取实例，不足则新建        │
│     → instance.call(func, args)     // 单次实例执行                  │
│     → 成功: pool.push(instance)    // 归还实例                       │
│     → 失败: 丢弃实例                // 可能已损坏，不复用             │
└───────────────────────┬─────────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 实例执行层 (plugin.rs: PluginInstance)                              │
│   PluginInstance::call()                                            │
│     1. 验证函数签名: 参数全为i32，返回单一i32                        │
│     2. 验证参数数量匹配                                             │
│     3. 将参数长度转为 Vec<Val::I32> 作为 WASM 调用参数               │
│     4. 将原始字节存入 store.data_mut().args                         │
│     5. handle.call(&mut store, &lengths, &mut [code])               │
│        └─→ WASM 执行期间回调宿主导入函数                            │
│     6. 检查 memory_error                                            │
│     7. 根据 code (0/1/其他) 解释 output                             │
└───────────────────────┬─────────────────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│ 宿主 ↔ WASM 内存交互层 (plugin.rs: 导入函数)                        │
│                                                                     │
│   write_args_to_buffer(ptr):                                        │
│     args → drain → 逐个 memory.write(ptr+offset, arg)               │
│                                                                     │
│   send_result_to_host(ptr, len):                                    │
│     memory.read(ptr, &mut buffer[0..len]) → output                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 参数提取：`Args::all::<Bytes>()`

在 [func.rs#L353-L358](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/func.rs#L353-L358)：

```rust
FuncInner::Plugin(func) => {
    let inputs = args.all::<Bytes>()?;
    let output = func.call(inputs).at(args.span)?;
    args.finish()?;
    Ok(Value::Bytes(output))
}
```

**`args.all::<Bytes>()` 的语义**（[args.rs#L192-L210](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/args.rs#L192-L210)）：
- 遍历所有**位置参数**（忽略命名参数）
- 对每个参数尝试 `T::castable(&value)` 和 `T::from_value(value)`
- 任何一个位置参数无法 cast 为 `Bytes`，就报错并带 span
- 这就是为什么测试用例 `p.hello(true, bytes(()), 10)` 同时报两个类型错误

**参数契约**：
- 插件函数的参数**必须全部是位置参数**，且**全部是 `Bytes` 类型**
- 命名参数被 `args.all()` 忽略，但随后的 `args.finish()` 会检查是否有未消费参数
- 若传递命名参数，`finish()` 会报 "unexpected named argument"

### 2.3 长度传递协议

WASM 函数接收的参数**不是实际数据**，而是每个字节缓冲区的**长度**（i32）。

假设调用 `p.shuffle(bytes("value1"), bytes("value2"), bytes("value3"))`：

| Typst 侧 | WASM 侧收到 |
|----------|-------------|
| `args[0]` = `Bytes("value1")` (len=6) | `a_1` = `6` |
| `args[1]` = `Bytes("value2")` (len=6) | `a_2` = `6` |
| `args[2]` = `Bytes("value3")` (len=6) | `a_3` = `6` |

WASM 插件需要自行：
1. 计算 `total = 6 + 6 + 6 = 18`
2. 分配 18 字节缓冲区 `buf`
3. 调用 `wasm_minimal_protocol_write_args_to_buffer(buf)`
4. 此时内存 `buf[0..6]` = "value1"，`buf[6..12]` = "value2"，`buf[12..18]` = "value3"

### 2.4 宿主-插件通信的核心：`CallData`

每个 wasmi `Store` 携带一份用户数据，类型为 `CallData`，定义于 [plugin.rs#L557-L566](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs#L557-L566)：

```rust
#[derive(Default)]
struct CallData {
    args: Vec<Bytes>,       // 写入前: 存待传入参数
    output: Vec<u8>,        // 读出后: 存插件返回数据
    memory_error: Option<MemoryError>,  // 越界错误记录
}
```

**生命周期**：
1. 调用前：`store.data_mut().args = args;`（转移所有权，`args` 被 move）
2. WASM 执行 `write_args_to_buffer` 时：`std::mem::take(&mut caller.data_mut().args)`（清空 args）
3. WASM 执行 `send_result_to_host` 时：填充 `output`
4. 调用后：`std::mem::take(&mut self.store.data_mut().output)` 取出结果

这种设计确保一次调用结束后，`CallData` 恢复默认状态，不残留上一次调用的数据。

### 2.5 实例池与多线程

`Plugin` 结构体维护一个 `Mutex<Vec<PluginInstance>>` 作为实例池：

```rust
fn acquire(&self) -> StrResult<PluginInstance> {
    if let Some(instance) = self.pool.lock().unwrap().pop() {
        return Ok(instance);
    }
    PluginInstance::new(&self.base, self.snapshot.as_ref())
}
```

**边界语义**：
- 单线程场景下，池始终保持 1 个实例循环复用
- 多线程（布局并行）时，并发调用会触发 `acquire()` 创建新实例
- 新实例从 `snapshot`（如有）恢复，确保 transition 语义在并发下依然成立
- 注意：`wasmi::Instance` 和 `wasmi::Store` 是 `!Send + !Sync` 的，因此每个线程必须持有独立实例——这也是实例池存在的根本原因

---

## 3. 失败处理层：加载期、调用前期、执行期

### 3.1 失败处理分类总览

| 阶段 | 失败类型 | 报错位置 | 用户可见错误信息 |
|------|----------|----------|------------------|
| **加载期** | WASM 字节码无效 | `Plugin::new()` | `failed to load WebAssembly module (...)` |
| **加载期** | 未导出 "memory" | `Plugin::new()` | `plugin does not export its memory` |
| **加载期** | 缺少必需导入 | `PluginInstance::new()` → `instantiate_and_start` | wasmi 输出的链接错误详情 |
| **调用前期** | 参数类型非 `Bytes` | `args.all::<Bytes>()` | `expected bytes, found X` |
| **调用前期** | 参数数量不匹配 | `PluginInstance::call()` | `plugin function takes N argument(s), but M was/were given` |
| **调用前期** | 函数签名不规范 | `PluginInstance::call()` | `has a parameter that is not a 32-bit integer` / `does not return exactly one 32-bit integer` |
| **执行期** | WASM 异常/trap | `handle.call()` | `plugin panicked: wasm 'unreachable' instruction executed` |
| **执行期** | 内存越界读写 | 导入函数检测 | `plugin tried to read/write out of bounds: pointer ...` |
| **执行期** | 插件主动返回错误 | code=1 + output | `plugin errored with: <message>` |
| **执行期** | 插件返回码非法 | code∉{0,1} | `plugin did not respect the protocol` |
| **执行期** | 错误消息非UTF-8 | code=1 但 from_utf8 失败 | `plugin errored, but did not return a valid error message` |

### 3.2 加载期失败详解

#### 3.2.1 WASM 字节码无效

```rust
let module = wasmi::Module::new(&engine, bytes.as_slice())
    .map_err(|err| format!("failed to load WebAssembly module ({err})"))?;
```

触发场景：
- 文件不是合法 WASM 魔数（`\0asm`）
- WASM 版本不兼容
- 字节码截断或损坏

#### 3.2.2 未导出 memory

```rust
if !matches!(module.get_export("memory"), Some(wasmi::ExternType::Memory(_))) {
    bail!("plugin does not export its memory");
}
```

这是**加载期**（非实例化期）就做的检查，因为 Typst 需要通过名字获取内存句柄。

### 3.3 调用前期检查：惰性签名验证

注意：签名验证**不在加载期**做，而是在**首次调用**时才执行。这是因为 WASM 模块可能导出 `_initialize`、`__data_end` 等非插件函数，这些函数签名不规范但也不会被 Typst 用户调用。

```rust
// PluginInstance::call() 内部
let ty = handle.ty(&self.store);

if ty.params().iter().any(|&v| v != wasmi::ValType::I32) {
    bail!("plugin function `{func}` has a parameter that is not a 32-bit integer");
}
if ty.results() != [wasmi::ValType::I32] {
    bail!("plugin function `{func}` does not return exactly one 32-bit integer");
}

let expected = ty.params().len();
let given = args.len();
if expected != given {
    bail!("plugin function takes {expected} argument{}, but {given} {} given", ...);
}
```

### 3.4 执行期失败：Trap 与 Panic

```rust
handle.call(&mut self.store, &lengths, std::slice::from_mut(&mut code))
    .map_err(|err| eco_format!("plugin panicked: {err}"))?;
```

WASM `unreachable` 指令、整数除零、栈溢出等都会触发 trap，被 wasmi 捕获后统一包装为 `"plugin panicked: <reason>"`。

**关键策略：实例损坏后不复用**

```rust
// Plugin::call()
let output = instance.call(func, args)?;  // 失败则 ? 提前返回
self.pool.lock().unwrap().push(instance); // 只有成功才归还
```

无论失败原因是 trap、越界、协议违规还是主动错误，**实例都不会返回池中**。
这是一种防御性设计：WASM trap 可能使实例处于不一致状态（例如半初始化的全局数据结构），后续调用可能产生非确定性行为。

### 3.5 执行期失败：内存越界检测

越界检测不依赖 wasmi 的 trap，而是由宿主导入函数在 `memory.write/read` 返回 `Err` 时主动记录。

记录结构：

```rust
struct MemoryError {
    offset: u32,   // 越界访问的指针
    length: u32,   // 尝试读写的字节数
    write: bool,   // true=写, false=读
}
```

检测流程：

1. `write_args_to_buffer` 或 `send_result_to_host` 调用 `memory.write/read`
2. 若返回 `Err`，将错误存入 `caller.data_mut().memory_error`
3. 宿主导入函数不 panic，正常返回；WASM 继续执行（可能产生垃圾结果）
4. `handle.call()` 返回后，立即检查并取出 `memory_error`
5. 格式化错误信息并返回

这样做的好处是可以提供**精确的诊断信息**（指针值、长度、读/写方向），而不是笼统的 "out of bounds"。

### 3.6 执行期失败：插件主动错误

插件通过返回码 `1` 表示业务错误，此时 `output` 被解释为 UTF-8 错误消息：

```rust
match code {
    wasmi::Val::I32(0) => {}
    wasmi::Val::I32(1) => match std::str::from_utf8(&output) {
        Ok(message) => bail!("plugin errored with: {message}"),
        Err(_) => bail!("plugin errored, but did not return a valid error message"),
    },
    _ => bail!("plugin did not respect the protocol"),
};
```

测试用例 `plugin-error` 验证此路径：`p.returns_err()` 返回 `"This is an `Err`"` 消息。

### 3.7 Transition 的失败处理

`transition` 调用中，`instance.call(func, args)?` 同样使用 `?` 提前返回：
- 失败时，实例**不被移动**（仍然属于原 Plugin 的 acquire 上下文，但因为 `?` 返回而被 drop）
- 成功时，实例才被 move 进新 Plugin 的 pool

这确保了失败的 transition 不会污染原 Plugin 或新 Plugin。

---

## 4. 核心数据结构关系图

```
                    ┌──────────────────────┐
                    │   Module (Typst)     │  ← #let m = plugin("x.wasm")
                    │  scope: name→Func    │
                    └───────┬──────────────┘
                            │ Func::from(PluginFunc)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  PluginFunc                                                     │
│  ├─ plugin: Arc<Plugin>          (共享整个插件状态)             │
│  └─ name: EcoString              (导出函数名)                   │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│  Plugin                                                         │
│  ├─ base: Arc<PluginBase>       (不可变: bytes+module+linker)   │
│  ├─ pool: Mutex<Vec<PluginInstance>>  (实例池, 可并发)          │
│  ├─ snapshot: Option<Snapshot>  (transition后内存快照)          │
│  └─ fingerprint: u128           (用于comemo缓存和相等性)        │
│                                                                 │
│  方法:                                                          │
│  · module(bytes) → Module     (memoized, 入口)                 │
│  · call(name, args) → Bytes   (memoized, 纯函数调用)           │
│  · transition(name, args) → Plugin  (memoized, 派生新Plugin)   │
└───────────┬─────────────────────────────────────────────────────┘
            │  acquire/pop         │ new() + restore(snapshot)
            ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│  PluginInstance  (每个线程独占一个, !Send !Sync)                │
│  ├─ instance: wasmi::Instance                                    │
│  ├─ store: wasmi::Store<CallData>                                │
│  │                   ├─ args: Vec<Bytes>     (入参暂存)        │
│  │                   ├─ output: Vec<u8>      (出参暂存)        │
│  │                   └─ memory_error: Option<_>(越界记录)      │
│  │                                                               │
│  方法:                                                          │
│  · call(name, args) → Bytes  (一次完整调用: 校验+执行+解包)     │
│  · snapshot() → Snapshot    (备份所有内存页)                    │
│  · restore(snapshot)        (恢复所有内存页)                    │
│  · memory() → Memory        (获取"memory"导出句柄)              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. 边界设计总结

### 5.1 设计约束与取舍

| 约束 | 实现方式 | 代价/限制 |
|------|----------|-----------|
| **Typst 函数必须纯** | `PluginFunc::call` 用 `#[comemo::memoize]`；不纯操作只能通过 `transition` | transition 只能线性持久化内存，不持久化 globals |
| **多线程布局** | 每个并发调用独立 `PluginInstance`，用实例池复用 | 内存开销与并发度成正比 |
| **WASM 沙箱隔离** | 不提供 WASI 导入，只提供两个协议函数 | 插件不能 I/O、不能打印、不能随机 |
| **确定性格式化** | `Config::wasm_relaxed_simd(false)` 关闭非确定性 SIMD | 损失部分性能优化 |
| **实例损坏容错** | 失败调用不归还实例到池 | 极端错误场景下实例池可能耗尽（但会自动新建，不会死锁） |

### 5.2 容易混淆的边界点

1. **`Plugin::call` memoize vs 实例池**：
   - memoize 在 `PluginFunc` 层（相同参数直接返回缓存结果，根本不进入 acquire）
   - 实例池在 `Plugin` 层（即使没命中 memoize，也尽量复用实例避免重新实例化开销）
   - 两层缓存是正交的

2. **`DataSource` vs `Loaded` vs `Bytes`**：
   - `DataSource`：用户输入（可能是路径字符串，也可能是原始字节），带 span
   - `Loaded`：解析后的结果，附带 `LoadSource`（路径FileId或纯Bytes元数据）
   - `Bytes`：最终进入 WASM 编译器的纯字节序列，Plugin::module 只接收 Bytes

3. **返回值 `0/1` 与 trap 的区别**：
   - 返回 `1` 是**预期内**的业务错误，插件作者明确决定返回可读消息
   - trap 是**预期外**的运行时崩溃（unreachable、除零等），属于插件 bug

4. **Transition 的"不可变性"**：
   - 旧 Plugin 和 新 Plugin 共享 `PluginBase`（字节码+链接器），这部分是纯不可变的
   - 但各自拥有独立的实例池、快照、指纹 —— mutation 被隔离在指纹分支中

---

**关键代码索引**：

| 功能 | 文件 | 行号范围 |
|------|------|----------|
| 插件加载入口 `plugin()` | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L148-L156 |
| Transition 函数 | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L192-L201 |
| `Plugin` 结构与实例池 | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L242-L400 |
| `PluginInstance::call` 核心调用 | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L447-L520 |
| 快照/恢复机制 | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L524-L545 |
| 宿主导入函数实现 | [plugin.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/plugin.rs) | L577-L612 |
| `Func` 对 Plugin 的分发 | [func.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/func.rs) | L353-L358 |
| `DataSource` 与 `Load` trait | [loading/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/loading/mod.rs) | L46-L154 |
| `Args::all::<T>()` 参数提取 | [args.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/crates/typst-library/src/foundations/args.rs) | L191-L210 |
| 插件测试用例 | [plugin.typ](file:///d:/fz/0601-2/solo-dogfeeding/code/130-typst/tests/suite/foundations/plugin.typ) | 全文 |
