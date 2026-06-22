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
| `LazyHash<T>` | 首次访问时计算并缓存 `T` 的 128 位哈希，后续直接比较 | `crates/typst-utils/src/hash.rs:72-129` |
| `Content` | 元素内容 + 样式的内容哈希（derive） | `crates/typst-library/src/foundations/content/mod.rs:81-84` |
| `Closure` | `ClosureNode(SyntaxNode)` + `defaults` + `captured(Scope)` 的内容哈希（derive） | `crates/typst-library/src/foundations/func.rs:702-721` |
| `SyntaxNode` | `Node` 数据 + `Span`（derive） | `crates/typst-syntax/src/node.rs:17-25` |
| `Scope` | 绑定数量 + 逐个绑定的内容哈希 | `crates/typst-library/src/foundations/scope.rs:226-235` |
| `Route` | 路径栈 + 栈深度（derive） | `crates/typst-library/src/engine.rs:396-429` |

### 1.2 第二层：Tracked 依赖回放

对于 `Tracked<T>` 参数，即使参数对象本身不同，只要 **之前记录的所有方法调用都返回相同结果**，缓存仍然命中。

**验证流程**：

```
第一次调用 memoized 函数时：
  1. 执行函数体
  2. 记录过程中对 Tracked 值的所有方法调用（方法名 + 参数 + 返回值）
  3. 将调用记录与结果一起存入缓存

后续调用时：
  1. 参数哈希匹配 → 进入第二层
  2. 用新的 Tracked 值，按相同顺序、相同参数重新执行所有记录的方法调用
  3. 比较每次调用的返回值是否与上次一致
  4. 全部一致 → 缓存命中；任意一个不同 → 缓存失效，重算
```

**代码证据**：Constraint 机制直接展示了这种"记录→回放→比较"的模式（`crates/typst/src/lib.rs:144-161`）：

```rust
let constraint = comemo::Constraint::new();
// track_with 将 constraint 与 introspector 关联，后续的方法调用会被记录
let tracked_intr = introspector.track_with(&constraint);
// ...布局过程中对 tracked_intr 的查询会被记录...
// 回放验证：用新的 introspector 重新执行所有记录的查询
if constraint.validate(new_introspector) {
    // 所有查询结果相同 → 缓存仍有效
}
```

Constraint 是 Tracked 依赖验证机制的显式暴露：普通 memoize 函数的缓存验证内部使用的是完全相同的逻辑。

---

## 2. 编译入口：Tracked 值的产生与传递

### 2.1 `compile` 函数的 Tracked 链

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

`world.track()`（来自 `comemo::Track` trait）将 `&dyn World` 包装为 `Tracked<dyn World>`。这个 `Tracked` 值会沿着调用链传递给所有 memoized 函数。

### 2.2 `compile_impl` 的内部流程

`crates/typst/src/lib.rs:99-194`：

1. **获取主文件**（`L117-120`）：调用 `world.source(main)` —— 这是 Tracked 依赖，返回值变化会使后续依赖该结果的缓存失效
2. **求值主模块**（`L123-131`）：调用 `typst_eval::eval(world, library, traced, sink.track_mut(), Route::default().track(), &main)`
3. **introspection 循环**（`L138-185`）：最多迭代 5 次，每次用 `track_with` 绑定 Constraint 来验证稳定性

---

## 3. 逐层分析：文件变化后的重算判定

假设用户编辑了源文件 A 的某一行（该文件 `#import "B.typ"`，B 未变化）。让我们逐层追踪哪些计算重跑、哪些复用。

### 3.0 前置：文件重置与增量解析

`crates/typst-cli/src/watch.rs:78-83` 触发重新编译前，先调用 `SystemWorld::reset()`（`crates/typst-cli/src/world.rs`）：

```rust
pub fn reset(&mut self) {
    self.store.reset();  // FileStore 重置
    self.now = Some(OffsetDateTime::now_utc());
}
```

`FileStore::reset()`（`crates/typst-kit/src/files.rs:111-116`）：
- 每个 `FileSlot` 从 `Parsed(Ok(source), _)` 转为 `Empty(Some(source))` —— 保留旧 source 作为 `stale`
- 下次访问时走 `FileSlot::source()` 逻辑（`crates/typst-kit/src/files.rs:190-218`）：
  - 重新加载文件字节
  - 如果有 `stale` source，调用 `source.replace(new_text)` 做增量更新（`crates/typst-syntax/src/source.rs:85-96`）
  - `replace` 内部调用 `edit` → `reparse`（`crates/typst-syntax/src/reparser.rs`）实现增量语法解析

**增量解析的效果**：
- 未被编辑影响的语法节点，其 `SyntaxNode` 对象的内存表示和 Hash **完全不变**
- 受影响的节点被重新解析，得到新的 Span 编号

这决定了后续各层缓存的命运。

---

### 3.1 第一层：模块求值 `eval`

`crates/typst-eval/src/lib.rs:38-97`

```rust
#[comemo::memoize]
pub fn eval(
    world: Tracked<dyn World + '_>,      // Tracked
    library: &LazyHash<Library>,          // 非 Tracked：Library 全局不变
    traced: Tracked<Traced>,              // Tracked
    sink: TrackedMut<Sink>,               // TrackedMut
    route: Tracked<Route>,                // Tracked
    source: &Source,                      // 非 Tracked：直接影响缓存键
) -> SourceResult<Module>
```

**判定逻辑**：

| 参数 | 变化情况 | 结果 |
|------|---------|------|
| `source`（文件 A） | 内容变了 → `Source` 的 `LazyHash` 重算 → 哈希不同 | **第一层参数不匹配** → 缓存失效，重算 |
| `source`（文件 B） | B 未编辑，即使经过增量解析，内容哈希不变 | **第一层匹配** → 进入第二层验证 |
| `world`（文件 B） | `world.source(B_id)` 返回的 Source 与上次相同 | 依赖回放通过 → **缓存命中** ✅ |
| `route` | 初始调用总是 `Route::default().track()`，不含任何 id | 回放通过 |

**结论**：
- 文件 A 的 `eval` **重算**（source 参数哈希变了）
- 文件 B 的 `eval` **完全命中**（source 哈希未变，world 依赖回放通过）

---

### 3.2 第二层：闭包调用 `eval_closure`

`crates/typst-eval/src/call.rs:598-707`

```rust
#[comemo::memoize]
pub fn eval_closure(
    func: &Func,
    closure: &LazyHash<Closure>,       // 非 Tracked：决定缓存键
    world: Tracked<dyn World + '_>,
    library: &LazyHash<Library>,
    introspector: Tracked<dyn Introspector + '_>,
    traced: Tracked<Traced>,
    sink: TrackedMut<Sink>,
    route: Tracked<Route>,
    context: Tracked<Context>,
    args: Args,                         // 非 Tracked：调用参数
) -> SourceResult<Value>
```

`Closure` 的哈希结构（`crates/typst-library/src/foundations/func.rs:702-721`）：
```rust
pub struct Closure {
    pub node: ClosureNode,       // SyntaxNode 引用（derive Hash）
    pub defaults: Vec<Value>,    // 默认参数值
    pub captured: Scope,         // 捕获的外部变量
    pub num_pos_params: usize,
}
```

**判定逻辑**：

| 场景 | `closure` 哈希 | `args` | `world/introspector` 回放 | 结果 |
|------|---------------|--------|--------------------------|------|
| A 中未被修改的闭包 | 语法节点未变 + 捕获变量值未变 → 哈希相同 | 相同调用参数 | 通过（假设外部文件没变化） | **缓存命中** ✅ |
| A 中被修改的闭包 | 语法节点变了 → 哈希不同 | - | - | **重算** ❌ |
| B 中所有闭包 | B 整个模块 eval 命中，闭包对象与上次完全相同 | 相同 | 通过 | **缓存命中** ✅ |
| 同一闭包，不同调用参数 | 相同 | 参数值不同 → 哈希不同 | - | **重算** ❌ |

**关键洞察**：闭包缓存的粒度比模块缓存更细。即使整个模块被重求值，只要某个闭包的**语法节点和捕获变量都没变**，它的调用结果仍然可以直接复用。这是通过 `LazyHash<Closure>` 的内容哈希实现的。

---

### 3.3 第三层：布局片段 `layout_fragment_impl`

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
    content: &Content,               // 非 Tracked：内容决定缓存键
    locator: Tracked<Locator>,       // Tracked：但延迟访问 outer
    styles: StyleChain,              // 非 Tracked
    regions: Regions,                // 非 Tracked
    columns: NonZeroUsize,           // 非 Tracked
    column_gutter: Rel<Abs>,         // 非 Tracked
) -> SourceResult<Fragment>
```

**判定逻辑**：

| 场景 | `content` 哈希 | `styles` | `locator` 依赖 | `introspector` 依赖 | 结果 |
|------|---------------|----------|---------------|--------------------|------|
| B 文件中的内容元素 | Content 对象完全相同（B 未变化） | 相同 | 外层位置可能变，但如果元素不生成 Location（不访问 outer）则不产生依赖 | 不查询或查询结果相同 | **缓存命中** ✅ |
| A 中未修改段落对应的 Content | Content 哈希相同（语法节点未变） | 相同 | 同上 | 同上 | **缓存命中** ✅ |
| A 中修改段落对应的 Content | Content 哈希不同 | - | - | - | **重算** ❌ |
| 带编号的章节元素（counter） | Content 哈希相同 | 相同 | 位置变了 → `next_location` 访问 outer → Locator 依赖不同 | `introspector.query(counter)` 返回值可能变 | **重算** ❌ |

**Locator 分层设计的作用**（`crates/typst-library/src/introspection/locator.rs`）：

`Locator` 本身实现了 `#[comemo::track]`（`crates/typst-library/src/introspection/locator.rs:208-223`）。它的 `local` 哈希只包含当前 memoization 边界内的信息，`outer` 通过 `LocatorLink` 延迟访问：

- 如果元素布局过程中**不调用** `locator.next_location()` → 不产生对 `outer` 的依赖 → 同一段内容在文档不同位置也能命中缓存
- 如果元素需要编号/定位 → 必须调用 `next_location()` → 依赖 `outer` → 位置变化时缓存失效

这是 Typst 增量布局命中率高的核心原因。

---

### 3.4 其他布局级缓存

| 函数 | 位置 | 关键参数 | 何时重算 |
|------|------|---------|---------|
| `layout_par_impl` | `crates/typst-layout/src/inline/mod.rs:70` | `elem: &Packed<ParElem>`, `locator`, `styles`, `regions` | 段落内容变了，或位置变了（若段落需要定位） |
| `layout_single_impl` | `crates/typst-layout/src/flow/collect.rs:413` | `content`, `locator`, `styles`, `region` | 单元素内容或定位变了 |
| `layout_multi_impl` | `crates/typst-layout/src/flow/collect.rs:513` | `children`, `locator`, `styles`, `regions` | 子元素列表或定位变了 |
| `layout_document_impl` | `crates/typst-layout/src/pages/mod.rs:51` | `content`, `introspector`, `styles` | 整文档内容或全局内省结果变了 |
| `layout_page_run_impl` | `crates/typst-layout/src/pages/run.rs:77` | 页面配置、内容、introspector | 页面级配置变了 |

---

### 3.5 细粒度缓存

| 函数 | 位置 | 关键参数 | 何时重算 |
|------|------|---------|---------|
| `create_shape_plan` | `crates/typst-layout/src/inline/shaping.rs:1219` | `font`, `direction`, `text`, `features`, ... | 文本内容、字体、排版特征变了 |
| `planned`（数学字形） | `crates/typst-layout/src/math/fragment/glyph.rs:108` | `world`, `styles`, `glyph`, `italic_correction` | 字形或样式变了 |
| `base`（数学字形） | `crates/typst-layout/src/math/fragment/glyph.rs:146` | `world`, `styles`, `family`, `c` | 字符或字体族变了 |
| `eval_string` | `crates/typst-eval/src/lib.rs:101` | `string`, `scope`, `world`, ... | 字符串内容或作用域变了 |

细粒度缓存的特点：参数中通常不包含 `locator`，也不依赖 `introspector`，因此只要内容本身不变，即使全局位置变化也能命中。例如某段文字的塑形结果在整篇文档中是可复用的。

---

## 4. 完整的重算链路图

以"编辑文件 A 中的一行文字"为例：

```
文件 A 磁盘内容变化
    │
    ▼
FileStore.reset()
  └─ A 的 FileSlot: Parsed → Empty(Some(stale_source))
     B 的 FileSlot: Parsed → Empty(Some(stale_source))
    │
    ▼
world.source(A_id)
  └─ 重新加载字节
     └─ stale_source.replace(new_text) → 增量解析
        └─ 受影响的 SyntaxNode 被重建，Hash 变化
        └─ 未受影响的 SyntaxNode 保持不变，Hash 相同
    │
    ▼
eval(world, ..., &source_A)
  ├─ source_A 的 LazyHash 已变化
  └─ 第一层参数不匹配 → 重算模块 A
     ├─ 模块 A 求值过程中，创建新的 Closure 对象
     │   ├─ 未修改函数的 ClosureNode 指向未变的 SyntaxNode
     │   │   └─ captured Scope 哈希不变 → Closure 哈希不变
     │   └─ 修改函数的 ClosureNode 指向新 SyntaxNode
     │       └─ Closure 哈希变化
     │
     ▼ eval(world, ..., &source_B)
       ├─ source_B 的 LazyHash 没变（B 文件内容相同）
       └─ 第一层匹配 → 第二层验证
          └─ world.source(B_id) 返回值与上次相同 → 回放通过
          └─ ✅ 缓存命中，整个模块 B 不复用
    │
    ▼ realize → layout_fragment_impl(...)
  对每个 Content 元素：
    ├─ B 文件产生的 Content
    │   └─ Content 哈希完全相同
    │   └─ styles 相同
    │   └─ locator：若不生成 Location → 无 outer 依赖
    │   └─ ✅ 缓存命中
    │
    ├─ A 文件未修改段落的 Content
    │   └─ SyntaxNode 未变 → Content 哈希相同
    │   └─ ✅ 缓存命中
    │
    └─ A 文件修改段落的 Content
        └─ SyntaxNode 重建 → Content 哈希不同
        └─ ❌ 重算布局
           └─ 内部子元素若 Content 不变，仍可能逐层命中
    │
    ▼ 细粒度计算（文本塑形、数学字形等）
    ├─ 未修改的文本字符串 → create_shape_plan 参数相同 → ✅ 命中
    └─ 修改的文本字符串 → ❌ 重新塑形
    │
    ▼
  comemo::evict(10) → 每个 memoized 函数保留最近 10 个缓存版本
```

---

## 5. 关键代码索引

### 5.1 Memoized 函数清单

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

### 5.2 Tracked Trait 清单

| Trait | 仓库相对路径 |
|-------|------------|
| `World` | `crates/typst-library/src/lib.rs:59` |
| `Introspector` | `crates/typst-library/src/introspection/introspector.rs:28` |
| `Locator` | `crates/typst-library/src/introspection/locator.rs:208` |
| `Traced` | `crates/typst-library/src/engine.rs:133` |
| `Sink` | `crates/typst-library/src/engine.rs:204` |
| `Route` | `crates/typst-library/src/engine.rs:396` |
| `Context` | `crates/typst-library/src/engine/context.rs` |

### 5.3 关键基础类型

| 类型 | 仓库相对路径 | 说明 |
|------|------------|------|
| `Source` | `crates/typst-syntax/src/source.rs:24` | `Arc<LazyHash<SourceInner>>`，基于内容的 Hash/Eq |
| `LazyHash<T>` | `crates/typst-utils/src/hash.rs:72` | 延迟计算并缓存 128 位哈希，直接比较哈希值 |
| `Content` | `crates/typst-library/src/foundations/content/mod.rs:84` | 类型擦除的内容元素，derive Hash/PartialEq |
| `Closure` | `crates/typst-library/src/foundations/func.rs:712` | 闭包节点+默认值+捕获作用域，derive Hash |
| `SyntaxNode` | `crates/typst-syntax/src/node.rs:18` | 语法节点，derive Hash/Eq/PartialEq |
| `Scope` | `crates/typst-library/src/foundations/scope.rs:105` | 变量绑定表，手动实现 Hash |
| `FileSlot` | `crates/typst-kit/src/files.rs:129` | 文件状态机：Empty/Loaded/Parsed，保留 stale Source |
| `FileStore` | `crates/typst-kit/src/files.rs:30` | 全局文件缓存，reset 时保留 stale source |

### 5.4 编译主流程

| 函数 | 仓库相对路径 | 说明 |
|------|------------|------|
| `compile` | `crates/typst/src/lib.rs:74` | 编译入口，创建 Tracked 值 |
| `compile_impl` | `crates/typst/src/lib.rs:99` | 内部实现，包含 eval + introspection 循环 |
| `replace` | `crates/typst-syntax/src/source.rs:85` | 增量更新 Source 文本 |
| `edit` | `crates/typst-syntax/src/source.rs:104` | 原地编辑语法树，保持 span 稳定 |
| `reparse` | `crates/typst-syntax/src/reparser.rs:15` | 增量语法解析核心 |
| `comemo::evict` | `crates/typst-cli/src/watch.rs:82` | watch 模式下驱逐旧缓存 |

---

## 6. 设计思想总结

### 6.1 两层验证 = 快速过滤 + 精确判定

- **第一层（参数哈希）**：快速排除大部分必然失效的情况。例如文件内容变了，`Source` 的哈希直接不同，无需再回放依赖。
- **第二层（Tracked 依赖回放）**：精确捕获"实际用到了什么"。即使 `World` 对象整体不同（每次编译都重新创建），只要具体调用过的方法返回值不变，缓存就有效。这是增量编译能跨编译轮次复用缓存的根本原因。

### 6.2 稳定性设计：让 Hash 尽可能不变

整个系统的设计目标是：**编辑后让尽可能多的值保持 Hash 不变**。

1. **增量解析**：只重建受影响的语法子树，未受影响的 `SyntaxNode` 对象哈希不变
2. **LazyHash**：基于内容的延迟哈希，只要内容没变，即使对象重新创建，哈希也相同
3. **Closure 内容哈希**：即使模块重求值，只要闭包语法和捕获变量没变，`Closure` 哈希就不变
4. **Locator 分层**：内容不依赖位置时，不产生位置依赖，位置变化不影响缓存

### 6.3 纯度假设

comemo 的所有机制建立在纯函数假设上：memoized 函数的输出完全由其参数（包括 Tracked 依赖）决定，没有隐式副作用。这就是为什么所有外部输入必须通过 `Tracked<dyn World>` 传入，以及为什么可变状态要用 `TrackedMut` 显式包装。
