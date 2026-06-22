# 增量编译缓存机制详解

本文档梳理 Typst 增量编译缓存的失效与重算机制，重点解释依赖追踪与缓存命中的关系，以及整个缓存系统是如何运作的。

## 1. 概述

Typst 的增量编译建立在 [`comemo`](https://github.com/typst/comemo/) 框架之上。核心思想是：**将编译过程拆分为多个可记忆化的函数，通过追踪函数的输入依赖来决定缓存是否仍然有效**。当输入未变化时，直接复用缓存结果；当输入变化时，只重新计算受影响的部分。

增量编译发生在编译的各个阶段：
- **解析阶段**：增量语法解析
- **求值阶段**：模块级和闭包级缓存
- **布局阶段**：元素级布局缓存
- **渲染/导出阶段**：各种细粒度缓存

## 2. 核心框架：comemo

### 2.1 核心概念

comemo 提供了三个核心原语：

| 原语 | 作用 |
|------|------|
| `#[comemo::memoize]` | 标记函数为可记忆化的，其结果会被缓存 |
| `#[comemo::track]` | 标记 trait/impl 为可追踪的，其方法调用会被记录为依赖 |
| `Tracked<T>` / `TrackedMut<T>` | 被追踪的值包装器，作为 memoized 函数的参数 |

### 2.2 记忆化函数（memoize）

使用 `#[comemo::memoize]` 标记的函数会：
- 第一次调用时执行函数体，缓存结果
- 后续调用时，如果输入相同且所有依赖都未变化，直接返回缓存结果
- 如果依赖发生变化，重新执行函数体，更新缓存

典型的 memoized 函数示例：

- 模块求值：[`eval`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/lib.rs#L38-L97)
- 闭包调用：[`eval_closure`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/call.rs#L598-L700)
- 布局片段：[`layout_fragment_impl`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-layout/src/flow/mod.rs#L108-L150)

### 2.3 可追踪类型（track）

使用 `#[comemo::track]` 标记的 trait 或 impl 块，其方法调用会被记录为依赖。当这些方法的返回值变化时，依赖它们的 memoized 函数缓存会失效。

Typst 中主要的可追踪类型：

| 类型 | 位置 | 作用 |
|------|------|------|
| `World` | [`lib.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/lib.rs#L59-L98) | 编译环境（文件、字体、日期等） |
| `Introspector` | [`introspector.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/introspector.rs#L28-L89) | 文档内省查询 |
| `Locator` | [`locator.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/locator.rs#L208-L223) | 元素定位器 |
| `Traced` | [`engine.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L133-L143) | 追踪的 span |
| `Sink` | [`engine.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L204-L254) | 诊断信息接收器 |
| `Route` | [`engine.rs`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L396-L429) | 编译路径/栈深度 |
| `Context` | [`context.rs`] | 求值上下文 |

## 3. 依赖追踪与缓存命中的关系

### 3.1 基本原理

依赖追踪是缓存命中的基础。当一个 memoized 函数被调用时：

1. **参数哈希**：函数的所有输入参数（包括 `Tracked` 值）会被哈希，作为缓存键的一部分
2. **依赖记录**：函数执行过程中，所有对 `Tracked` 值的方法调用都会被记录为依赖
3. **缓存存储**：函数结果连同其依赖列表一起被缓存
4. **缓存验证**：下次调用时，先检查参数是否相同，再验证所有依赖是否仍然有效

### 3.2 World 的核心作用

[`World`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/lib.rs#L59-L98) trait 是最核心的依赖来源，因为：
- 它提供了所有外部输入（源文件、图像、字体、日期等）
- 它被 `#[comemo::track]` 标记，所有方法调用都会被追踪
- 当文件内容变化时，`world.source(id)` 的返回值会变化，导致所有依赖该文件的缓存失效

```
文件变化 → World 方法返回值变化 → 依赖该方法的缓存失效 → 相关函数重算
```

### 3.3 缓存命中的条件

一个 memoized 函数的缓存命中需要满足：
1. 所有非 `Tracked` 参数值完全相同（通过 `Hash` + `Eq` 比较）
2. 所有 `Tracked` 参数的依赖集合并未发生变化
3. 缓存条目未被驱逐（evict）

## 4. 缓存层级与粒度

Typst 的缓存是分层的，不同阶段有不同的缓存粒度：

### 4.1 解析层：增量解析

**粒度**：语法节点级

通过 [`reparse`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-syntax/src/reparser.rs#L15-L29) 函数实现：
- 只重新解析受编辑影响的最小范围
- 优先尝试在子块内重解析
- 如果失败则向外扩展，直到顶层
- 保持尽可能多的 span 编号稳定

**关键优化**：[`Source::edit`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-syntax/src/source.rs#L104-L112) 方法在原地修改语法树，而不是重新创建，这使得 span 编号尽可能保持稳定，从而提高后续阶段的缓存命中率。

### 4.2 求值层：模块与闭包

**粒度**：模块级 + 闭包级

- **模块缓存**：[`eval`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/lib.rs#L38-L97) 函数缓存整个源文件的求值结果
- **闭包缓存**：[`eval_closure`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/call.rs#L598-L700) 函数缓存闭包调用的结果

闭包缓存的精妙之处在于：即使模块被重新求值，只要闭包的语法和捕获变量不变，之前的闭包调用结果仍然可以复用。

### 4.3 布局层：元素级缓存

**粒度**：布局片段级

[`layout_fragment_impl`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-layout/src/flow/mod.rs#L108-L150) 是布局缓存的核心入口：
- 缓存内容在特定区域中的布局结果
- 依赖 content、locator、styles、regions 等参数
- 通过 Locator 的设计优化缓存命中率（见下文）

### 4.4 细粒度缓存

除了上述主要层级，还有大量细粒度的 memoized 函数，例如：
- 文本塑形（shaping）
- 图像解码
- 数学公式布局
- 书目处理
- 计数器/状态查询

## 5. 缓存边界优化：Locator 的设计

[`Locator`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/locator.rs) 是 Typst 增量编译设计中最精妙的部分之一，它展示了如何通过精心设计数据结构来提高缓存命中率。

### 5.1 问题背景

如果布局函数直接接收完整的位置信息作为参数，那么同一段内容在文档的不同位置布局时，缓存会完全失效，因为参数不同。

但实际上，很多内容的布局并不依赖于它在文档中的具体位置（例如，不包含任何需要编号的元素）。对于这些内容，我们希望能够复用缓存。

### 5.2 分层设计

Locator 采用了分层设计：

```
Locator {
    local: u128,      // 本地哈希（当前层的信息）
    outer: Option<&LocatorLink>  // 外层链接（延迟解析）
}
```

- **local**：只包含当前 memoization 边界内的信息
- **outer**：指向更外层的 locator，但通过 `LocatorLink` 间接访问

### 5.3 延迟解析

只有当真正需要生成 `Location`（即调用 `next_location`）时，才会通过 `resolve()` 方法去访问 outer 信息。

这意味着：
- 如果一段内容布局不需要生成任何 Location（不包含可定位元素），它就不会访问 outer
- 不访问 outer 意味着不会产生对 outer 的依赖
- 没有对 outer 的依赖意味着：同一段内容在不同位置布局时，缓存仍然有效！

### 5.4 LocatorLink 的作用

[`LocatorLink`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/locator.rs#L343-L395) 作为缓存边界：
- 将 `Tracked<Locator>` 包装起来
- 缓存 resolve 的结果
- 只有在真正需要时才 traverse 到外层

## 6. 缓存失效机制

### 6.1 自动失效（依赖驱动）

这是最主要的失效方式。当被追踪的依赖发生变化时，相关缓存自动失效。

**典型场景**：
1. 源文件内容变化 → `world.source(id)` 返回新的 `Source` → 依赖该文件的 `eval` 缓存失效
2. 内省结果变化 → `introspector.query(...)` 返回不同结果 → 依赖该查询的布局缓存失效
3. 日期变化 → `world.today(...)` 返回不同值 → 依赖日期的函数缓存失效

### 6.2 手动驱逐（evict）

[`comemo::evict(n)`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-cli/src/watch.rs#L82) 用于手动驱逐旧的缓存条目。

**在 watch 模式下**：
```rust
loop {
    // ... 等待文件变化 ...
    compile_once(world, &mut config)?;  // 重新编译
    comemo::evict(10);  // 驱逐旧缓存，保留最近 10 个版本
}
```

参数 `10` 的含义：保留每个 memoized 函数最近的 10 个缓存版本。这是为了控制内存使用，防止无限制增长。

### 6.3 FileStore 的 reset 机制

[`FileStore::reset()`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-kit/src/files.rs#L111-L116) 是文件层面的重置：

```rust
pub fn reset(&mut self) {
    for slot in self.slots.get_mut().values_mut() {
        slot.reset();
    }
}
```

重置后：
- 所有文件槽位变为 `Empty` 状态
- 但保留旧的 `Source` 作为 `stale`（陈旧）状态
- 下次访问时，会重新加载文件内容
- 如果有 stale source，会尝试用 `source.replace(new_text)` 进行增量更新，而不是从头解析

**这很重要**：通过保留 stale source 并进行增量更新，span 编号尽可能保持稳定，从而提高后续编译阶段的缓存命中率。

## 7. Introspection 循环与 Constraint

### 7.1 为什么需要循环

文档布局存在循环依赖：
- 布局依赖于内省信息（如页码、计数器、章节号等）
- 内省信息又来自于布局结果

为了解决这个问题，Typst 采用迭代的方式：
1. 用空的/上一轮的内省结果开始布局
2. 布局完成后得到新的内省结果
3. 检查内省结果是否稳定
4. 如果不稳定，用新的内省结果再次布局
5. 最多迭代 5 次

### 7.2 Constraint 的作用

[`comemo::Constraint`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst/src/lib.rs#L144-L158) 用于验证内省结果是否稳定，而无需完全重新计算。

工作流程：

```rust
let constraint = comemo::Constraint::new();
let introspector = history.last()
    .map(|doc| doc.introspector())
    .unwrap_or(&empty_introspector);

// 使用 constraint 来追踪 introspector
let tracked_introspector = introspector.track_with(&constraint);

// ... 执行布局 ...
document = T::create(&mut engine, &content, styles)?;

// 验证：新的 introspector 是否与 constraint 兼容
if constraint.validate(document.introspector()) {
    // 稳定了，退出循环
    break;
}
```

**原理**：
- `track_with` 将 constraint 与 tracked value 关联
- 在布局过程中，所有对 introspector 的查询都会记录到 constraint 中
- `validate` 检查：如果用新的 introspector 重新执行所有记录的查询，结果是否相同
- 如果所有查询结果都相同，说明内省结果对本次布局没有影响，可以认为稳定了

**关键优势**：不是比较整个 introspector 是否相同，而是只比较实际被访问过的查询结果。这大大提高了收敛速度。

## 8. Watch 模式下的完整流程

让我们通过 watch 模式的完整流程来串联所有机制：

### 8.1 初始编译

```
1. 创建 SystemWorld
2. 调用 typst::compile()
   ├─ world.track() → 创建 Tracked<dyn World>
   ├─ 调用 eval() 求值主模块（memoized）
   │   └─ 依赖：world.source(main)
   ├─ 进入 introspection 循环
   │   ├─ 用空 introspector 开始
   │   ├─ 调用 layout_fragment_impl()（memoized）
   │   │   └─ 依赖：world, introspector, locator, content, styles, ...
   │   ├─ 生成 document 和新的 introspector
   │   └─ 检查是否稳定，不稳定则继续迭代
   └─ 返回最终结果
3. 收集依赖：world.dependencies()
4. 监听依赖文件变化
```

### 8.2 文件变化后重新编译

```
1. 文件变化 → 触发重新编译
2. world.reset()
   ├─ FileStore.reset() → 所有文件标记为空（但保留 stale source）
   └─ 重置日期
3. 调用 typst::compile()
   ├─ world.track() → 新的 Tracked<dyn World>
   ├─ 访问 world.source(main)
   │   └─ 重新加载文件，尝试增量解析
   ├─ 调用 eval() 求值主模块
   │   ├─ 如果 source 没变 → 缓存命中，直接返回
   │   └─ 如果 source 变了 → 缓存失效，重新求值
   ├─ 进入布局阶段
   │   ├─ 对于没变的内容部分 → 缓存命中
   │   └─ 对于变化的内容部分 → 重新计算
   └─ ...
4. comemo::evict(10) → 清理旧缓存
5. 更新依赖监听列表
```

### 8.3 缓存复用的层次

当只有一小部分内容变化时，缓存复用是分层的：

1. **未变化的源文件** → `eval` 缓存完全命中
2. **变化文件中未变化的闭包** → `eval_closure` 缓存可能命中
3. **未变化的内容元素** → `layout_fragment_impl` 缓存可能命中
4. **更细粒度的计算** → 文本塑形、图像解码等缓存可能命中

## 9. 关键数据结构与代码路径总结

### 9.1 核心 memoized 函数

| 函数 | 位置 | 缓存粒度 |
|------|------|----------|
| `eval` | [typst-eval/src/lib.rs:38](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/lib.rs#L38) | 源文件/模块 |
| `eval_string` | [typst-eval/src/lib.rs:101](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/lib.rs#L101) | 字符串求值 |
| `eval_closure` | [typst-eval/src/call.rs:598](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-eval/src/call.rs#L598) | 闭包调用 |
| `layout_fragment_impl` | [typst-layout/src/flow/mod.rs:108](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-layout/src/flow/mod.rs#L108) | 布局片段 |
| `layout_pages` | [typst-layout/src/pages/mod.rs:51](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-layout/src/pages/mod.rs#L51) | 页面布局 |

### 9.2 核心 tracked trait

| Trait | 位置 | 依赖内容 |
|-------|------|----------|
| `World` | [typst-library/src/lib.rs:59](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/lib.rs#L59) | 文件、字体、日期等所有外部输入 |
| `Introspector` | [typst-library/src/introspection/introspector.rs:28](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/introspector.rs#L28) | 文档内省查询 |
| `Locator` | [typst-library/src/introspection/locator.rs:208](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/introspection/locator.rs#L208) | 元素位置/编号 |
| `Traced` | [typst-library/src/engine.rs:133](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L133) | 追踪的 span |
| `Sink` | [typst-library/src/engine.rs:204](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L204) | 诊断信息 |
| `Route` | [typst-library/src/engine.rs:396](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst-library/src/engine.rs#L396) | 编译路径/栈深度 |

### 9.3 编译主入口

[`compile`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst/src/lib.rs#L74-L82) 函数是编译的主入口，内部的 [`compile_impl`](file:///d:/fz/0601-2/solo-dogfeeding/code/131-typst/crates/typst/src/lib.rs#L99-L194) 包含了完整的编译流程和 introspection 循环。

## 10. 设计思想总结

### 10.1 纯度假设

comemo 的缓存机制建立在一个重要假设之上：**所有 memoized 函数都是纯函数**。

- 相同的输入总是产生相同的输出
- 函数没有副作用（或者副作用通过 `TrackedMut` 显式管理）
- 所有外部依赖都通过 `Tracked` 参数传入

这就是为什么 `World` 必须是唯一的外部依赖入口，以及为什么所有可变状态都要用 `TrackedMut` 包装。

### 10.2 细粒度 vs 粗粒度

Typst 采用了多层次的缓存策略：
- **粗粒度**：模块级缓存，命中率高但失效代价大
- **细粒度**：元素级缓存，命中率低但失效代价小

两者结合，既保证了大部分未变化内容的快速复用，又能在变化发生时将重算范围控制在最小。

### 10.3 稳定性设计

整个系统的设计都在追求一个目标：**让尽可能多的东西在编辑后保持稳定**。

- 增量解析 → 保持语法树和 span 编号的稳定
- 基于哈希的 location → 保持元素标识的稳定
- Locator 的分层设计 → 最大化布局缓存的复用
- Constraint 验证 → 加速内省循环的收敛

这些设计共同作用，使得 Typst 在编辑文档时能够实现近乎实时的增量编译。
