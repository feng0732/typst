# 增量编译缓存机制详解

本文档沿着 Typst 代码的缓存重算链路，详细解释 **tracked 依赖记录怎样参与缓存命中验证**，以及 **文件变化后哪些计算会重跑、哪些结果仍能复用**。

---

## 1. 缓存命中验证的两层机制

一个 `#[comemo::memoize]` 函数的缓存是否命中，需要经过 **两层验证**。理解这两层验证是看懂整条链路的关键。

### 1.1 第一层：参数哈希匹配

所有非 `Tracked`/`TrackedMut` 的参数通过 `Hash + Eq` 比较。这是快速路径：如果参数不同，直接缓存失效，无需进入第二层验证。

**关键数据结构的哈希实现**：

| 类型 | 哈希依据 | 代码位置 |
|------|---------|---------|
| `Source` | `Arc<LazyHash<SourceInner>>`，哈希值是 `(id, root, text)` 的内容哈希（延迟计算） | `crates/typst-syntax/src/source.rs:23-32` |
| `LazyHash<T>` | 首次访问时计算并缓存 `T` 的 128 位哈希，后续直接比较哈希值 | `crates/typst-utils/src/hash.rs:72-129` |
| `Content` | 元素内容 + 样式的内容哈希（derive） | `crates/typst-library/src/foundations/content/mod.rs:81-84` |
| `Closure` | `ClosureNode(SyntaxNode)` + `defaults` + `captured(Scope)` 的内容哈希（derive） | `crates/typst-library/src/foundations/func.rs:702-721` |
| `SyntaxNode` | `Node` 数据 + `Span`（derive） | `crates/typst-syntax/src/node.rs:17-25` |
| `Scope` | 绑定数量 + 逐个绑定的内容哈希 | `crates/typst-library/src/foundations/scope.rs:226-235` |
| `Route` | 路径栈 + 栈深度（tracked trait，不参与第一层哈希） | `crates/typst-library/src/engine.rs:396-429` |

### 1.2 第二层：Tracked 依赖验证

对于 `Tracked<T>` 参数，它们不参与第一层的哈希匹配。即使参数对象本身不同，只要 **该 T 上实际发生过的所有方法调用都返回相同结果**，缓存仍然命中。

从 Typst 代码中可以观察到这种模式的实际运作：

- 每次 `compile()` 都创建新的 `world.track()`（`crates/typst/src/lib.rs:75`），即不同编译轮次的 `Tracked<dyn World>` 是不同的对象实例
- 但如果 `world.source(id)` 返回的 `Source` 内容没变，`eval` 的缓存仍然可以命中
- 这说明第二次调用时，不是比较 Tracked 对象本身是否相同，而是验证实际访问过的方法调用链

**Typed Constraint：模式的显式暴露**

在 introspection 循环中使用的 `comemo::Constraint`（`crates/typst/src/lib.rs:144-161`）提供了一个可直接观察的等价模式：

```rust
let constraint = comemo::Constraint::new();
// 将 constraint 与 introspector 绑定：此后对 tracked_intr 的方法调用会被约束记录
let tracked_intr = introspector.track_with(&constraint);

// ...执行布局，过程中对 tracked_intr 的所有查询被记录...
document = T::create(&mut engine, &content, styles)?;

// 用新的 introspector 回放约束记录中的所有查询
// 返回 true 表示所有查询结果与上次一致，返回 false 表示至少有一个不同
if timed!("check stabilized", constraint.validate(document.introspector())) {
    sink.extend_from_sink(subsink);
    break;
}
```

从这段代码可以确认的事实：
1. `track_with(&constraint)` 建立了"记录器"与 tracked 值的绑定
2. 方法调用被记录为包含（方法标识 + 参数 + 返回值）的条目
3. `validate(new_value)` 用新值按相同参数重放所有记录的调用，逐一比较返回值

> 注：comemo 0.5.1 是 Typst 外部 crate，缓存验证内部实现不在本仓库中。上文对 Tracked 第二层验证流程的描述，是基于 `Constraint::new()/track_with()/validate()` 公开 API 的使用模式、结合多次编译轮次间缓存复用的实际行为所做的同构推断。两者在"记录方法调用、回放验证返回值"这个层面上运作方式是一致的。

---

## 2. Watch 模式的缓存重置入口

### 2.1 触发点：`watch.rs` 的主循环

`crates/typst-cli/src/watch.rs:68-83` 是 watch 模式的核心循环：

```rust
loop {
    // 用上一轮编译得到的依赖列表更新文件监听
    watcher.update(world.dependencies())?;
    // 阻塞等待文件事件
    watcher.wait()?;

    // 重置世界状态
    world.reset();

    // 执行新一轮编译
    timer.record(&mut world, |world| compile_once(world, &mut config))??;

    // 驱逐旧的缓存条目，每个 memoized 函数保留最近 10 个版本
    comemo::evict(10);
}
```

重置入口是 `world.reset()`，它分别处理**文件缓存**和**日期缓存**两个独立的部分。

### 2.2 `SystemWorld::reset` 的双重置

`crates/typst-cli/src/world.rs:104-107`：

```rust
/// Reset the compilation state in preparation of a new compilation.
pub fn reset(&mut self) {
    self.files.reset();   // 文件缓存重置
    self.now.reset();     // 日期缓存重置
}
```

`SystemWorld` 结构（`crates/typst-cli/src/world.rs:25-38`）：

```rust
pub struct SystemWorld {
    workdir: Option<PathBuf>,
    library: LazyHash<Library>,
    fonts: LazyLock<FontStore, Box<dyn Fn() -> FontStore + Send + Sync>>,
    files: FileStore<SystemFiles>,   // ← 文件缓存
    now: Time,                        // ← 日期缓存
}
```

注意：
- `library` 从未被 reset，它在整个 watch 生命周期内全局恒定
- `fonts` 是 `LazyLock<FontStore>`，在首次访问后**全程复用、永不重置**
- **只有 `files` 和 `now` 会在每次编译前被 reset**

### 2.3 文件缓存重置：`FileStore::reset`

`crates/typst-kit/src/files.rs:101-116`：

```rust
pub fn reset(&mut self) {
    for slot in self.slots.get_mut().values_mut() {
        slot.reset();
    }
}
```

每个 `FileSlot` 的 reset 逻辑（`crates/typst-kit/src/files.rs:167-174`）：

```rust
fn reset(&mut self) {
    let stale = match mem::take(self) {
        // 只有成功解析过的 Source 才被保留为 stale
        Self::Parsed(Ok(source), _) => Some(source),
        _ => None,
    };
    // 统一转为 Empty 状态，附带可能的 stale Source
    *self = Self::Empty(stale);
}
```

`FileSlot` 的状态机（`crates/typst-kit/src/files.rs:129-154`）：

| 状态 | 含义 |
|------|------|
| `Empty(Stale<Source>)` | reset 后的初始态。未被访问，但可能持有上一轮的陈旧 Source 用于增量更新 |
| `Loaded(Bytes, Stale<Source>)` | 已被 `file()` 访问，加载了原始字节但还没被解析成 Source |
| `Parsed(Result<Source, Utf8Error>, Bytes)` | 已被 `source()` 访问并完成解析 |

**reset 之后，下次 `source()` 访问时的流程**（`crates/typst-kit/src/files.rs:190-237`）：

1. 处于 `Empty(stale)` 状态 → 调用 `loader.load(id)` 重新从磁盘加载字节
2. 如果 `stale` 是 `Some(source)`，走增量更新路径：
   ```rust
   str::from_utf8(...).map(|new| {
       source.replace(new);  // crates/typst-syntax/src/source.rs:85-96
       source
   })
   ```
3. `replace` 计算前后文本的公共前后缀，只对中间不同部分调用 `edit` → `reparse`（`crates/typst-syntax/src/reparser.rs`）
4. 最终槽位进入 `Parsed` 状态，本轮后续访问直接返回缓存

**重置对依赖追踪的影响**：
- reset 后 `accessed()` 条件（`!matches!(self, Self::Empty(_))`）变为 false，所有文件重新变为"未访问"
- 编译完成后 `world.dependencies()`（`crates/typst-kit/src/files.rs:91-99`）只收集本轮实际访问过的文件
- 这样 watcher 监听列表会**自动跟随 import 变化而增减**

### 2.4 日期缓存重置：`Time::reset`

`Time` 结构（`crates/typst-kit/src/datetime.rs:16-25`）有两种内部变体：

```rust
pub struct Time(TimeInner);

enum TimeInner {
    // 用户用 SOURCE_DATE_EPOCH 或 --creation-timestamp 指定的固定时间
    Fixed(DateTime<Utc>),
    // 系统时间，内部用 OnceLock 保证单次编译内多次调用返回一致的值
    System(OnceLock<DateTime<Utc>>),
}
```

`Time::reset` 实现（`crates/typst-kit/src/datetime.rs:124-128`）：

```rust
pub fn reset(&mut self) {
    if let TimeInner::System(ref mut time_lock) = self.0 {
        time_lock.take();  // 清空 OnceLock
    }
    // Fixed 变体什么也不做
}
```

`Time::today` 实现（`crates/typst-kit/src/datetime.rs:83-118`）：

```rust
pub fn today(&self, offset: Option<Duration>) -> Option<Datetime> {
    let now = match &self.0 {
        TimeInner::Fixed(time) => time.fixed_offset(),
        TimeInner::System(time) => {
            // OnceLock::get_or_init：第一次调用执行 Utc::now，
            // 后续调用在 OnceLock 被 take() 之前都返回同一个值
            let now_utc = time.get_or_init(Utc::now);
            // ...处理时区...
        }
    };
    // ...用 now 计算当前日期...
}
```

**日期重置效果**：
- **`Time::System` 变体**：reset 清空 OnceLock → 下一轮编译首次调用 `today()` 时重新取系统时间
- **`Time::Fixed` 变体**：全程无视 reset，永远返回构造时的固定值（用于可重现构建）
- **单轮编译内一致性**：无论哪种变体，一轮编译中多次调用 `World::today` 总是返回相同日期

`SystemWorld::today` 直接转发（`crates/typst-cli/src/world.rs:142-144`）：

```rust
fn today(&self, offset: Option<Duration>) -> Option<Datetime> {
    self.now.today(offset)
}
```

由于 `World` trait 被 `#[comemo::track]` 标记，`today()` 返回值的变化会通过 Tracked 依赖验证链路传播：如果下一轮日期不同，所有调用过 `world.today(...)` 的 memoized 函数其第二层验证会失败 → 重算。

### 2.5 字体元数据缓存：永不重置的第三类缓存

`SystemWorld.fonts` 是**第三种独立的缓存类别**——它既不像文件缓存那样每次 reset，也不像日期缓存那样有条件 reset，而是**初始化后全程复用、永不重置**。

#### 2.5.1 初始化时机

在 watch 模式下，字体缓存在**首次编译前**就被主动初始化（`crates/typst-cli/src/watch.rs:55-57`）：

```rust
// Eagerly scan fonts if we expect to need them so that it's not counted as
// part of the displayed compilation time.
if config.output_format.is_paged() {
    world.scan_fonts();
}
```

`SystemWorld::scan_fonts` 只是强制 `LazyLock` 初始化（`crates/typst-cli/src/world.rs:110-114`）：

```rust
pub fn scan_fonts(&mut self) {
    LazyLock::force(&self.fonts);
}
```

初始化闭包调用 `discover_fonts` 完成字体扫描（`crates/typst-cli/src/world.rs:79-81`）：

```rust
fonts: LazyLock::new(Box::new(|| {
    crate::fonts::discover_fonts(&world_args.font)
})),
```

`discover_fonts` 的完整流程（`crates/typst-cli/src/fonts.rs:38-55`）：

```rust
pub fn discover_fonts(args: &FontArgs) -> FontStore {
    let mut fonts = FontStore::new();
    if !args.ignore_system_fonts {
        fonts.extend(fonts::system());      // 扫描系统字体
    }
    #[cfg(feature = "embedded-fonts")]
    if !args.ignore_embedded_fonts {
        fonts.extend(fonts::embedded());    // 加载嵌入字体
    }
    for path in &args.font_paths {
        fonts.extend(fonts::scan(path));    // 扫描自定义字体目录
    }
    fonts
}
```

#### 2.5.2 内部结构与两级缓存

`FontStore` 内部有两级缓存（`crates/typst-kit/src/fonts.rs:24-27`）：

```rust
pub struct FontStore {
    book: LazyHash<FontBook>,     // 第一级：字体元数据（家族名、变体、字符覆盖等）
    slots: Vec<FontSlot>,         // 第二级：实际字体对象，延迟加载
}
```

`FontSlot` 实现了按需加载（`crates/typst-kit/src/fonts.rs:86-97`）：

```rust
struct FontSlot {
    source: Box<dyn FontSource>,  // 字体来源（文件路径或已加载对象）
    font: OnceLock<Option<Font>>, // 实际字体对象，OnceLock 保证只加载一次
}

impl FontSlot {
    fn get(&self) -> Option<Font> {
        // OnceLock::get_or_init：首次调用时从 source.load() 加载，后续直接返回
        self.font.get_or_init(|| self.source.load()).clone()
    }
}
```

`FontBook` 是字体元数据的索引结构（`crates/typst-library/src/text/font/book.rs:12-18`）：

```rust
#[derive(Debug, Default, Clone, Hash)]
pub struct FontBook {
    families: BTreeMap<String, Vec<usize>>,  // 小写家族名 → 字体索引列表
    infos: Vec<FontInfo>,                    // 每个字体的元数据
}
```

注意 `FontBook` 实现了 `Hash`，被 `LazyHash` 包装后哈希值一旦计算就缓存。

#### 2.5.3 为什么不参与 reset

**技术层面**：
- `LazyLock<FontStore>` 一旦初始化就没有 reset 接口（Rust 标准库设计）
- `FontSlot.font` 是 `OnceLock<Option<Font>>`，一旦 set 就无法清空
- `LazyHash<FontBook>` 的哈希值一旦计算就缓存，不会重新计算

**设计层面**（`crates/typst-library/src/lib.rs:51-54` 的注释明确说明）：

> The compiler doesn't do the caching itself because the world has much more
> information on when something can change. For example, fonts typically don't
> change and can thus even be cached across multiple compilations (for
> long-running applications like `typst watch`).

翻译：字体**通常不会变化**，因此可以跨多次编译缓存。

**性能层面**：
- 系统字体扫描可能涉及上千个字体文件，是昂贵的 I/O + 解析操作
- 每次编译前重新扫描会完全抵消增量编译的性能收益
- watch 模式也不监听字体目录的变化（只监听源文件依赖）

**语义层面**：
`World::font()` 方法的注释（`crates/typst-library/src/lib.rs:84-88`）暗示字体索引跨轮次稳定：

> Note that the index is not guaranteed to be in bounds of the font book
> returned by this world's `book()` function. This is the case because
> this function may be invoked with indices from an outdated or different
> font book during incremental compilation validation.

这说明 comemo 在缓存验证回放时，可能用"上一轮记录的字体索引"调用 `world.font(index)`，因此**字体索引在整个 watch 生命周期内必须保持稳定**。

#### 2.5.4 参与缓存验证的方式

由于 `World` trait 被 `#[comemo::track]` 标记，以下方法调用都会被记录为 Tracked 依赖：
- `world.book()` → 返回 `&LazyHash<FontBook>`
- `world.font(index)` → 返回 `Option<Font>`

但因为：
1. `FontBook` 内容永不改变 → `LazyHash<FontBook>` 的哈希值永远不变
2. `FontSlot.get()` 一旦加载就永远返回相同的 `Font` 对象

所以字体相关的 Tracked 依赖回放**永远通过**，不会成为缓存失效的原因。

---

## 3. 编译入口：Tracked 值的产生与传递

### 3.1 `compile` 函数的 Tracked 链

从 `crates/typst/src/lib.rs:74-82` 开始：

```rust
pub fn compile<T>(world: &dyn World) -> Warned<SourceResult<T>>
where T: Output,
{
    let mut sink = Sink::new();
    let output = compile_impl::<T>(
        world.track(),          // Tracked<dyn World>
        Traced::default().track(), // Tracked<Traced>
        &mut sink,
    ).map_err(deduplicate);
    Warned { output, warnings: sink.warnings() }
}
```

`world.track()`（来自 `comemo::Track` trait）将 `&dyn World` 包装为 `Tracked<dyn World>`。**注意**：每一轮 `compile()` 都会产生新的 `Tracked<dyn World>` 实例——这就是为什么需要第二层 Tracked 依赖验证，而非简单比较对象引用。

### 3.2 `compile_impl` 的内部流程

`crates/typst/src/lib.rs:99-194`：

1. **获取主文件**（`L117-120`）：调用 `world.source(main)` —— 该调用会被记录为后续 `eval` 的依赖
2. **求值主模块**（`L123-131`）：调用 `typst_eval::eval(world, library, traced, sink.track_mut(), Route::default().track(), &main)`
3. **introspection 循环**（`L138-185`）：最多迭代 5 次，每次用 `track_with` 绑定一个新的 `Constraint` 来判定内省稳定性

> introspection 循环中的 Constraint 不是用来控制 comemo 缓存的——`layout_fragment_impl` 等函数的缓存验证完全由 comemo 内部机制驱动。Constraint 的作用是**在迭代之间判断是否需要再跑一轮**：如果本轮布局中对内省结果的所有查询，用上一轮布局产生的新 introspector 重新查询也得到相同答案，就说明已经收敛，可以终止循环。

---

## 4. 逐层分析：文件变化后的重算判定

假设用户编辑了源文件 A 的某一行（该文件 `#import "B.typ"`，B 未变化）。让我们逐层追踪哪些计算重跑、哪些复用。

### 4.0 前置：文件重置与增量解析

watch 循环中（`crates/typst-cli/src/watch.rs:75-76`）：

```rust
world.reset();  // SystemWorld::reset
```

执行流程：
1. `FileStore::reset()` 把所有 `FileSlot` 转为 `Empty(Some(source))`（如果上一轮成功解析了 Source）
2. `Time::reset()` 清空 `OnceLock`（如果是 System 变体）
3. 下一次 `world.source(A_id)` 时：重新加载字节 → `stale_source.replace(new)` → 增量解析
4. 增量解析效果：未被编辑范围影响的语法节点其 `SyntaxNode` 的 Hash 保持不变；影响范围内的节点被重建，获得新 Span

### 4.1 第一层：模块求值 `eval`

`crates/typst-eval/src/lib.rs:38-97`

```rust
#[comemo::memoize]
pub fn eval(
    world: Tracked<dyn World + '_>,      // Tracked：第二层验证
    library: &LazyHash<Library>,          // 非 Tracked：全局恒定，第一层永远匹配
    traced: Tracked<Traced>,              // Tracked：第二层验证
    sink: TrackedMut<Sink>,               // TrackedMut
    route: Tracked<Route>,                // Tracked：第二层验证
    source: &Source,                      // 非 Tracked：第一层匹配
) -> SourceResult<Module>
```

**判定逻辑**：

| 参数 | 变化情况 | 结果 |
|------|---------|------|
| `source`（文件 A） | 文本变了 → `Source` 的 `LazyHash` 重算 → 哈希不同 | **第一层参数不匹配** → 缓存失效，重算 |
| `source`（文件 B） | B 未编辑，经过 `replace` 后文本与上一轮相同，`LazyHash` 不变 | **第一层匹配** → 进入第二层验证 |
| `world`（文件 B） | 模块内需要 import 其他文件时会调用 `world.source(x_id)`，如果 x 文件未变，返回值相同；`world.today(...)` 若日期未跨天也相同 | 依赖回放通过 |
| `route` | 初始调用 `Route::default().track()` | 回放通过 |
| `traced` | watch 模式下始终 `Traced::default()`（无 span inspect） | 回放通过 |

**结论**：
- 文件 A 的 `eval` **重算**（source 参数哈希变了）
- 文件 B 的 `eval` **完全命中**（source 哈希未变，world 等 Tracked 依赖回放通过）

### 4.2 第二层：闭包调用 `eval_closure`

`crates/typst-eval/src/call.rs:598-707`

```rust
#[comemo::memoize]
pub fn eval_closure(
    func: &Func,
    closure: &LazyHash<Closure>,       // 非 Tracked：第一层匹配
    world: Tracked<dyn World + '_>,    // Tracked：第二层验证
    library: &LazyHash<Library>,
    introspector: Tracked<dyn Introspector + '_>,
    traced: Tracked<Traced>,
    sink: TrackedMut<Sink>,
    route: Tracked<Route>,
    context: Tracked<Context>,
    args: Args,                         // 非 Tracked：第一层匹配
) -> SourceResult<Value>
```

`Closure` 的哈希结构（`crates/typst-library/src/foundations/func.rs:702-721`）：
```rust
pub struct Closure {
    pub node: ClosureNode,       // 内含 SyntaxNode 引用（derive Hash）
    pub defaults: Vec<Value>,    // 默认参数值
    pub captured: Scope,         // 捕获的外部变量绑定
    pub num_pos_params: usize,
}
```

**判定逻辑**：

| 场景 | `closure` 哈希 | `args` | Tracked 依赖回放 | 结果 |
|------|---------------|--------|-----------------|------|
| A 中未被修改的闭包 | 语法节点未重建 + 捕获变量值未变 → `LazyHash<Closure>` 哈希相同 | 相同调用参数 | world.source(x) 等返回值与上次相同 | **缓存命中** ✅ |
| A 中被修改的闭包 | 语法节点被重建 → 哈希不同 | - | - | **重算** ❌ |
| B 中所有闭包 | B 模块 eval 缓存命中，返回的 Module 中闭包对象与上次完全相同 | 相同 | 通过 | **缓存命中** ✅ |
| 同一闭包，不同调用参数 | 相同 | 参数值不同 → 哈希不同 | - | **重算** ❌ |

**关键洞察**：闭包缓存的粒度比模块缓存更细。即使整个模块被重求值，只要某个闭包的**语法节点引用和捕获变量哈希都没变**，它的调用结果仍然可以直接复用。这正是 `LazyHash<Closure>` 基于内容哈希的意义所在。

### 4.3 第三层：布局片段 `layout_fragment_impl`

`crates/typst-layout/src/flow/mod.rs:108-123`

```rust
#[comemo::memoize]
fn layout_fragment_impl(
    world: Tracked<dyn World + '_>,
    library: &LazyHash<Library>,
    introspector: Tracked<dyn Introspector + '_>,
    traced: Tracked<Traced>,
    sink: TrackedMut<Sink>,
    route: Tracked<Route>,
    content: &Content,               // 非 Tracked：第一层匹配
    locator: Tracked<Locator>,       // Tracked：但延迟访问 outer
    styles: StyleChain,              // 非 Tracked：第一层匹配
    regions: Regions,                // 非 Tracked：第一层匹配
    columns: NonZeroUsize,           // 非 Tracked：第一层匹配
    column_gutter: Rel<Abs>,         // 非 Tracked：第一层匹配
) -> SourceResult<Fragment>
```

**判定逻辑**：

| 场景 | `content` 哈希 | `styles` | Tracked 依赖回放 | 结果 |
|------|---------------|----------|-----------------|------|
| B 文件中的内容元素 | Content 对象与上一轮完全相同（B eval 缓存命中） | 相同 | locator：若不生成 Location 则无 outer 依赖；introspector：若不查询计数器等则无依赖；world：不直接访问 | **缓存命中** ✅ |
| A 中未修改段落的 Content | SyntaxNode 未重建 → Content 哈希不变 | 相同 | 同上 | **缓存命中** ✅ |
| A 中修改段落的 Content | SyntaxNode 被重建 → Content 哈希不同 | - | - | **重算** ❌ |
| 带编号的章节元素 | Content 哈希相同（编号显示不改变 Content 本身） | 相同 | locator：`next_location()` 访问 outer → 若位置变了则回放失败；introspector：`query(counter)` 若编号值变了则回放失败 | **重算** ❌ |

**Locator 分层设计的作用**（`crates/typst-library/src/introspection/locator.rs`）：

`Locator` 被 `#[comemo::track]` 标记（`crates/typst-library/src/introspection/locator.rs:208-223`）。结构上：
- `local: u128` —— 只包含当前 memoization 边界内的信息
- `outer: Option<Tracked<Self>>` —— 指向外层的 Tracked<Locator>，**只有访问时才产生依赖**

如果一段内容布局不调用 `locator.next_location()`（例如不包含 `counter()`、`locate()` 调用的普通段落），就不会触发对 `outer` 的方法调用 → 不产生 outer 依赖 → 同一段内容在文档不同位置也能命中缓存。

### 4.4 其他布局级缓存

| 函数 | 位置 | 关键参数 | 何时重算 |
|------|------|---------|---------|
| `layout_par_impl` | `crates/typst-layout/src/inline/mod.rs:70` | `elem: &Packed<ParElem>`, `locator`, `styles`, `regions` | 段落内容变了（第一层不匹配），或 Tracked 依赖回放失败 |
| `layout_single_impl` | `crates/typst-layout/src/flow/collect.rs:413` | `content`, `locator`, `styles`, `region` | content 变了或 locator/introspector 回放失败 |
| `layout_multi_impl` | `crates/typst-layout/src/flow/collect.rs:513` | `children`, `locator`, `styles`, `regions` | 子元素列表变了或 Tracked 依赖回放失败 |
| `layout_document_impl` | `crates/typst-layout/src/pages/mod.rs:51` | `content`, `introspector`, `styles` | 整文档内容变了或全局内省查询回放失败 |
| `layout_page_run_impl` | `crates/typst-layout/src/pages/run.rs:77` | 页面配置、content、introspector | 页面级配置变化或依赖回放失败 |

### 4.5 细粒度缓存

| 函数 | 位置 | 关键参数 | 何时重算 |
|------|------|---------|---------|
| `create_shape_plan` | `crates/typst-layout/src/inline/shaping.rs:1219` | `font`, `direction`, `text`, `features` | 文本内容、字体、排版特征变化（全部是第一层哈希参数） |
| `planned`（数学字形） | `crates/typst-layout/src/math/fragment/glyph.rs:108` | `world`, `styles`, `glyph`, `italic_correction` | 字形/样式变化（第一层），或 world.font() 返回不同字体（第二层） |
| `base`（数学字形） | `crates/typst-layout/src/math/fragment/glyph.rs:146` | `world`, `styles`, `family`, `c` | 字符或字体族变化 |
| `eval_string` | `crates/typst-eval/src/lib.rs:101` | `string`, `scope`, `world`, `introspector`, `context`, ... | 字符串或作用域哈希变化，或 Tracked 依赖回放失败 |

细粒度缓存的特点：参数中通常**不含 `locator`**，也极少依赖 `introspector`，因此即使章节编号、页码等全局位置变化，只要文本内容本身不变也能命中。

---

## 5. 完整的重算链路图

以"编辑文件 A 中的一行文字"为例：

```
文件 A 磁盘内容变化 → watcher.wait() 返回
    │
    ▼ crates/typst-cli/src/watch.rs:76
world.reset()
  ├─ files.reset()  crates/typst-kit/src/files.rs:111
  │   └─ A 的 FileSlot: Parsed(Ok(src), _) → Empty(Some(src))  ← 保留为 stale
  │      B 的 FileSlot: Parsed(Ok(src), _) → Empty(Some(src))  ← 保留为 stale
  │
  ├─ now.reset()  crates/typst-kit/src/datetime.rs:124
  │   └─ TimeInner::System(lock) → lock.take()，清空 OnceLock
  │      （如果是 TimeInner::Fixed 则什么都不做）
  │
  └─ fonts：不参与 reset，全程复用
      └─ LazyLock<FontStore> 已初始化 → LazyHash<FontBook> 哈希不变
         FontSlot.OnceLock 已加载的字体永远不变
    │
    ▼ 编译过程中首次访问 world.source(A_id)
    └─ crates/typst-kit/src/files.rs:190 source()
        └─ Empty(stale) → loader.load(id) → 读到新字节
           └─ stale.is_some() → source.replace(new_text) 增量更新
              └─ crates/typst-syntax/src/reparser.rs 增量解析
                 └─ 影响范围外 SyntaxNode：对象引用不变，Hash 不变
                    影响范围内 SyntaxNode：重建，获得新 Span，新 Hash
    │
    ▼ crates/typst/src/lib.rs:123 eval(world, ..., &source_A)
    ├─ 第一层：source_A 的 LazyHash 变了 → 参数不匹配
    └─ → 重算模块 A
        ├─ 遍历 A 中所有函数定义创建 Closure 对象
        │   ├─ 未修改函数：ClosureNode 引用未变 SyntaxNode
        │   │   └─ captured Scope 哈希未变 → LazyHash<Closure> 不变
        │   └─ 修改函数：ClosureNode 引用新 SyntaxNode
        │       └─ LazyHash<Closure> 变化
        │
        └─ eval 过程中 import B → 调用 eval(world, ..., &source_B)
             ├─ 第一层：source_B 的 LazyHash 不变
             └─ 第二层：用新 Tracked<World> 回放 B 模块中所有 world 调用
                └─ world.source(x) 返回值均相同；world.today() 若日期相同也不变
                └─ ✅ 缓存命中，直接返回上一轮的 Module
    │
    ▼ realize 阶段 → 为每个 Content 元素调用 layout_fragment_impl(...)
    ├─ B 文件产生的 Content
    │   └─ Content 对象指针相同 → 第一层哈希匹配
    │   └─ styles、regions 等相同
    │   └─ 若不生成 Location、不查内省 → Tracked 回放通过
    │   └─ ✅ 缓存命中
    │
    ├─ A 文件未修改段落的 Content
    │   └─ 语法节点未变 → Content 哈希相同
    │   └─ ✅ 缓存命中
    │
    └─ A 文件修改段落的 Content
        └─ 语法节点重建 → Content 第一层哈希不匹配
        └─ ❌ 重算布局
           └─ 重算过程中其内部的 create_shape_plan 等细粒度函数
              └─ 子文本未变 → 细粒度缓存仍可能命中
    │
    ▼ 所有细粒度函数
    ├─ 文本未变段落：create_shape_plan 的 text/font 参数相同 → ✅ 命中
    │   （font 参数来自 world.font(index)，字体索引跨轮次稳定，OnceLock 已加载）
    └─ 文本已变段落：text 参数不同 → ❌ 重新塑形
    │
    ▼ crates/typst-cli/src/watch.rs:82
comemo::evict(10)
  └─ 每个 memoized 函数，只保留最近 10 个（参数哈希 + 依赖集）版本
     更早的缓存条目被驱逐以控制内存
```

---

## 6. 关键代码索引

### 6.1 Watch 模式与重置入口

| 函数/结构 | 仓库相对路径 | 说明 |
|-----------|------------|------|
| `watch()` 主循环 | `crates/typst-cli/src/watch.rs:18-84` | 文件监听 → reset → 编译 → evict 循环 |
| `SystemWorld::reset` | `crates/typst-cli/src/world.rs:104-107` | 双重置入口（files + now） |
| `SystemWorld` 结构 | `crates/typst-cli/src/world.rs:25-38` | files + now + library + fonts |
| `FileStore::reset` | `crates/typst-kit/src/files.rs:111-116` | 批量 reset 所有 FileSlot |
| `FileSlot::reset` | `crates/typst-kit/src/files.rs:167-174` | Parsed→Empty(stale) 状态转换 |
| `FileSlot` 状态机 | `crates/typst-kit/src/files.rs:129-154` | Empty/Loaded/Parsed |
| `FileSlot::source` | `crates/typst-kit/src/files.rs:190-237` | stale source 增量更新路径 |
| `Time` 结构 | `crates/typst-kit/src/datetime.rs:16-25` | Fixed / System(OnceLock) |
| `Time::reset` | `crates/typst-kit/src/datetime.rs:124-128` | 仅清空 System 的 OnceLock |
| `Time::today` | `crates/typst-kit/src/datetime.rs:83-118` | get_or_init 保证单轮一致 |
| `SystemWorld::today` | `crates/typst-cli/src/world.rs:142-144` | 直接转发 now.today |
| `SystemWorld::scan_fonts` | `crates/typst-cli/src/world.rs:110-114` | 主动强制字体缓存初始化 |
| `discover_fonts` | `crates/typst-cli/src/fonts.rs:38-55` | 扫描系统/嵌入/自定义字体 |
| `FontStore` 结构 | `crates/typst-kit/src/fonts.rs:24-27` | book(LazyHash) + slots(FontSlot) |
| `FontSlot` 结构 | `crates/typst-kit/src/fonts.rs:86-97` | source + OnceLock<Font> |
| `FontBook` 结构 | `crates/typst-library/src/text/font/book.rs:12-18` | families + infos，derive Hash |
| `World::font` 注释 | `crates/typst-library/src/lib.rs:84-88` | 暗示字体索引跨轮次稳定 |
| `World` trait 注释 | `crates/typst-library/src/lib.rs:51-54` | 说明字体可跨编译缓存 |
| `comemo::evict(10)` | `crates/typst-cli/src/watch.rs:82` | 保留最近 10 个缓存版本 |

### 6.2 Memoized 函数清单

| 函数 | 仓库相对路径 |
|------|------------|
| `eval` | `crates/typst-eval/src/lib.rs:38` |
| `eval_string` | `crates/typst-eval/src/lib.rs:101` |
| `eval_closure` | `crates/typst-eval/src/call.rs:598` |
| `layout_fragment_impl` | `crates/typst-layout/src/flow/mod.rs:108` |
| `layout_single_impl` | `crates/typst-layout/src/flow/collect.rs:413` |
| `layout_multi_impl` | `crates/typst-layout/src/flow/collect.rs:513` |
| `layout_par_impl` | `crates/typst-layout/src/inline/mod.rs:70` |
| `layout_document_impl` | `crates/typst-layout/src/pages/mod.rs:51` |
| `layout_document_for_bundle_impl` | `crates/typst-layout/src/pages/mod.rs:98` |
| `layout_page_run_impl` | `crates/typst-layout/src/pages/run.rs:77` |
| `create_shape_plan` | `crates/typst-layout/src/inline/shaping.rs:1219` |
| `planned` (math glyph) | `crates/typst-layout/src/math/fragment/glyph.rs:108` |
| `base` (math glyph) | `crates/typst-layout/src/math/fragment/glyph.rs:146` |

### 6.3 Tracked Trait 清单

| Trait | 仓库相对路径 |
|-------|------------|
| `World` | `crates/typst-library/src/lib.rs:59` |
| `Introspector` | `crates/typst-library/src/introspection/introspector.rs:28` |
| `Locator` | `crates/typst-library/src/introspection/locator.rs:208` |
| `Traced` | `crates/typst-library/src/engine.rs:133` |
| `Sink` | `crates/typst-library/src/engine.rs:204` |
| `Route` | `crates/typst-library/src/engine.rs:396` |
| `Context` | `crates/typst-library/src/engine/context.rs` |

### 6.4 关键基础类型

| 类型 | 仓库相对路径 | 说明 |
|------|------------|------|
| `Source` | `crates/typst-syntax/src/source.rs:24` | `Arc<LazyHash<SourceInner>>`，基于内容的 Hash/Eq |
| `LazyHash<T>` | `crates/typst-utils/src/hash.rs:72` | 延迟计算并缓存 128 位哈希，直接比较哈希值 |
| `Content` | `crates/typst-library/src/foundations/content/mod.rs:84` | 类型擦除的内容元素，derive Hash/PartialEq |
| `Closure` | `crates/typst-library/src/foundations/func.rs:712` | 闭包节点+默认值+捕获作用域，derive Hash |
| `SyntaxNode` | `crates/typst-syntax/src/node.rs:18` | 语法节点，derive Hash/Eq/PartialEq |
| `Scope` | `crates/typst-library/src/foundations/scope.rs:105` | 变量绑定表，手动实现 Hash |
| `FileStore` | `crates/typst-kit/src/files.rs:36` | 全局文件缓存，reset 时保留 stale source |

### 6.5 编译主流程与 Constraint

| 函数 | 仓库相对路径 | 说明 |
|------|------------|------|
| `compile` | `crates/typst/src/lib.rs:74` | 编译入口，创建 Tracked 值 |
| `compile_impl` | `crates/typst/src/lib.rs:99` | 内部实现：eval + introspection 循环 |
| `Constraint::new/track_with/validate` | `crates/typst/src/lib.rs:144,150,158` | 内省稳定性约束的使用 |
| `Source::replace` | `crates/typst-syntax/src/source.rs:85` | 增量更新 Source 文本 |
| `Source::edit` | `crates/typst-syntax/src/source.rs:104` | 原地编辑语法树，保持 span 稳定 |
| `reparse` | `crates/typst-syntax/src/reparser.rs:15` | 增量语法解析核心 |

---

## 7. 设计思想总结

### 7.1 两层验证 = 快速过滤 + 精确判定

- **第一层（参数哈希）**：快速排除必然失效的情况。文件内容变了 → `Source` 的 `LazyHash` 直接不同，无需进入第二层。
- **第二层（Tracked 依赖验证）**：精确捕获"实际用到了什么"。每一轮编译都产生新的 `Tracked<dyn World>` 实例，但只要实际调用过的方法（`source()`、`today()`、`book()` 等）返回值与上一轮一致，缓存就有效。这是增量编译能跨编译轮次复用缓存的根本原因。

### 7.2 三重缓存策略

Watch 模式下的缓存分为三个独立维度，根据变化频率采用不同的重置策略：

| 缓存类型 | 重置策略 | 核心机制 | 设计考量 |
|---------|---------|---------|---------|
| **文件缓存** | 每次编译前 reset | `FileStore::reset` → `FileSlot` 转为 `Empty(stale)`，保留陈旧 Source 用于增量解析 | 源文件可能随时变化，必须重新从磁盘确认，但尽量保持语法树稳定以利于缓存命中 |
| **日期缓存** | 每次编译前条件 reset | `Time::reset` 仅清空 `System(OnceLock)`，`Fixed` 变体无视 | 日期可能跨天变化，但单轮编译内必须一致；固定时间戳模式下永远不变用于可重现构建 |
| **字体元数据缓存** | 永不 reset | `LazyLock<FontStore>` + `LazyHash<FontBook>` + `OnceLock<Font>` | 字体通常不会变化，扫描成本高昂，且跨轮次稳定的字体索引是增量缓存验证的语义前提 |

### 7.3 稳定性设计：让 Hash 尽可能不变

整个系统的设计目标是：**编辑后让尽可能多的值保持 Hash 不变**。

1. **增量解析**：只重建受影响的语法子树，未受影响的 `SyntaxNode` 对象哈希不变
2. **LazyHash**：基于内容的延迟哈希，只要内容没变，即使对象重新创建（或者通过 replace 原地更新了部分字段）哈希也相同
3. **Closure 内容哈希**：即使模块重求值，只要闭包语法节点引用和捕获变量没变，`LazyHash<Closure>` 就不变
4. **Locator 分层**：内容不依赖位置时不产生 outer 依赖，位置变化不影响缓存
5. **Constraint 稳定性检查**：内省循环不是盲目跑 5 次，而是用约束回放验证"实际用到的内省查询是否已稳定"，一旦稳定立刻终止减少无效布局

### 7.4 纯度假设

comemo 的所有机制建立在纯函数假设上：memoized 函数的输出完全由其参数（包括 Tracked 依赖）决定，没有隐式副作用。`World` trait 作为唯一外部依赖入口、`Library` 全局恒定不变、可变状态（`Sink` 等）用 `TrackedMut` 显式包装——所有这些设计约束都是为了保证这个假设成立。
