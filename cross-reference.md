# 引用与交叉链接的多次解析过程与收敛机制

## 一、核心概念

### 1. Label（标签）
Label 是 Typst 中用于标记元素的核心机制，定义在 [label.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/foundations/label.rs)。

```rust
#[ty(scope, cast)]
#[derive(Debug, Copy, Clone, Eq, PartialEq, Hash)]
pub struct Label(PicoStr);
```

**关键特性：**
- Label 本质是一个 interned 字符串的包装
- 语法：`<name>` 或 `#label("name")`
- 标签会附加到其前面最近的非空格元素上
- 同一标签在文档中必须唯一（否则 `@ref` 会报错）

### 2. Ref（引用）
Ref 是对 Label 的交叉引用，定义在 [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs)。

```rust
#[elem(title = "Reference", Locatable, Tagged, Synthesize)]
pub struct RefElem {
    #[required]
    pub target: Label,
    pub supplement: Smart<Option<Supplement>>,
    #[default(RefForm::Normal)]
    pub form: RefForm,
    #[synthesized]
    pub citation: Option<Packed<CiteElem>>,
    #[synthesized]
    pub element: Option<Content>,
}
```

**两种引用形式：**
- `RefForm::Normal`：生成文本引用（如 "Section 1"、"Figure 2"）
- `RefForm::Page`：生成页码引用（如 "page 5"）

### 3. Outline（目录）
Outline 用于生成目录、图表目录等，定义在 [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/outline.rs)。

```rust
#[elem(scope, keywords = ["Table of Contents", "toc"], ShowSet, LocalName, Locatable, Tagged)]
pub struct OutlineElem {
    pub title: Smart<Option<Content>>,
    #[default(LocatableSelector(HeadingElem::ELEM.select()))]
    pub target: LocatableSelector,
    pub depth: Option<NonZeroUsize>,
    pub indent: Smart<OutlineIndent>,
}
```

---

## 二、多次解析的原因

交叉引用和目录生成本质上是**"鸡生蛋、蛋生鸡"**的问题：

1. **引用需要知道目标的位置**：`@intro` 需要知道 `<intro>` 标签所在的章节编号、页码
2. **目录需要知道所有章节的信息**：`#outline()` 需要知道所有标题的编号、页码
3. **但这些信息只有在布局完成后才确定**：而布局过程中又可能引用这些信息

典型的依赖循环：
```
排版 → 生成章节编号 → 目录需要这些编号 → 目录占据空间 → 
页码变化 → 章节页码变化 → 目录中的页码需要更新 → ...
```

因此，Typst 采用**多遍编译（multi-pass compilation）**机制，通过迭代直到所有依赖关系稳定。

---

## 三、Introspection（内省）系统

Introspection 是连接多次编译的桥梁，核心定义在 [convergence.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs)。

### 3.1 Introspect trait

```rust
pub trait Introspect: Debug + PartialEq + Hash + Send + Sync + Sized + 'static {
    type Output: Hash;

    fn introspect(
        &self,
        engine: &mut Engine,
        introspector: Tracked<dyn Introspector + '_>,
    ) -> Self::Output;

    fn diagnose(&self, history: &History<Self::Output>) -> SourceDiagnostic;
}
```

**重要设计决策**（注释中明确提到）：
> 输出类型中包含的信息量会影响收敛行为。例如，将查询结果简化为布尔值可能比原始查询提前一轮收敛。

### 3.2 关键 Introspection 类型

在 [query.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/query.rs) 中定义：

| 类型 | 用途 |
|------|------|
| `QueryIntrospection(Selector, Span)` | 查询所有匹配元素 |
| `QueryLabelIntrospection(Label, Span)` | 通过标签查询唯一元素 |
| `QueryFirstIntrospection(Selector, Span)` | 查询第一个匹配元素 |
| `QueryUniqueIntrospection(Selector, Span)` | 查询唯一匹配元素 |
| `PageNumberingIntrospection(Location, Span)` | 查询某位置的页码编号 |
| `PageSupplementIntrospection(Location, Span)` | 查询某位置的页码补充文本 |

### 3.3 Introspector

[Introspector](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/introspector.rs#L29-L89) trait 提供了对编译结果的查询接口：

```rust
#[comemo::track]
pub trait Introspector: Send + Sync {
    fn query(&self, selector: &Selector) -> EcoVec<Content>;
    fn query_label(&self, label: Label) -> StrResult<&Content>;
    fn page(&self, location: Location) -> Option<NonZeroUsize>;
    fn position(&self, location: Location) -> Option<DocumentPosition>;
    fn page_numbering(&self, location: Location) -> Option<&Numbering>;
    // ... 其他方法
}
```

**ElementIntrospector** 的构建过程：
1. 布局过程中，通过 `Tag::Start` 和 `Tag::End` 标记可内省元素
2. `ElementIntrospectorBuilder` 收集这些标签，构建索引：
   - `locations: FxHashMap<Location, Range<usize>>`：按位置快速查找
   - `labels: MultiMap<Label, usize>`：按标签快速查找
   - `elems: Vec<(Content, P)>`：所有元素的有序列表
3. 查询时使用这些加速结构，通过 `QueryCache` 缓存查询结果

---

## 四、编译主循环：收敛过程

编译主循环定义在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst/src/lib.rs#L99-L194)，最多迭代 5 次：

```rust
pub const MAX_ITERS: usize = 5;

fn compile_impl<T: Output>(...) -> SourceResult<T> {
    // 第一步：计算主模块内容（只做一次）
    let content = typst_eval::eval(...)?.content();
    
    let mut history: ArrayVec<T, { MAX_ITERS - 1 }> = ArrayVec::new();
    let mut document: T;

    loop {
        // 使用上一轮的 introspector（第一轮用 EmptyIntrospector）
        let introspector = history
            .last()
            .map(|doc| doc.introspector())
            .unwrap_or(&empty_introspector);
        
        let constraint = comemo::Constraint::new();
        let mut engine = Engine {
            introspector: Protected::new(introspector.track_with(&constraint)),
            // ...
        };

        // 核心：创建文档（包含 realize + layout）
        document = T::create(&mut engine, &content, styles)?;

        // 检查是否收敛
        if constraint.validate(document.introspector()) {
            break;  // 收敛，退出循环
        }

        if history.is_full() {
            // 5次迭代后仍未收敛，分析并生成警告
            let warnings = typst_library::introspection::analyze(...);
            // ...
            break;
        }

        history.push(document);
    }
}
```

### 4.1 收敛检测机制

使用 `comemo::Constraint` 进行收敛检测：
1. 每次迭代开始时创建 `Constraint`
2. `introspector.track_with(&constraint)` 将内省器与约束绑定
3. 编译过程中对 introspector 的所有访问都会被约束记录
4. `constraint.validate(new_introspector)` 检查：如果使用新的 introspector 重新计算，结果是否相同
5. 如果相同 → 收敛，退出循环；如果不同 → 继续迭代

### 4.2 迭代 N+1 观察迭代 N 的结果

关键原则：**第 N+1 次迭代观察第 N 次迭代产生的 Output 值**

```
迭代 1: 使用 EmptyIntrospector → 产生 Introspector 1
迭代 2: 使用 Introspector 1 → 产生 Introspector 2
迭代 3: 使用 Introspector 2 → 产生 Introspector 3
...
直到 Introspector N == Introspector N+1（收敛）
或 达到 5 次上限
```

### 4.3 非收敛诊断

5次迭代后仍未收敛时，调用 `analyze()` 函数，定义在 [convergence.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs#L25-L70)：

```rust
pub fn analyze(
    world: Tracked<dyn World + '_>,
    introspectors: [&dyn Introspector; INSTANCES],
    introspections: &[Introspection],
) -> EcoVec<SourceDiagnostic> {
    for introspection in introspections {
        if let Some(warning) = introspection.0.diagnose(world, introspectors) {
            sink.warn(warning);
        }
    }
    // 生成汇总警告
    let summary = warning!(
        "document did not converge within five attempts";
        hint: "see https://typst.app/help/convergence for help";
    );
}
```

**History::converged()** 检测收敛：

```rust
pub fn converged(&self) -> bool
where
    T: Hash,
{
    // 比较最后两次迭代的 hash
    typst_utils::hash128(&self.0[MAX_ITERS - 1].1)
        == typst_utils::hash128(&self.0[MAX_ITERS].1)
}
```

---

## 五、Ref 的两阶段处理

Ref 元素的处理分为 **Synthesize（合成）** 和 **Realize（实现）** 两个阶段。

### 5.1 Synthesize 阶段

在 [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L201-L224)：

```rust
impl Synthesize for Packed<RefElem> {
    fn synthesize(&mut self, engine: &mut Engine, styles: StyleChain) -> SourceResult<()> {
        let span = self.span();
        let citation = to_citation(self, engine, styles)?;

        let elem = self.as_mut();
        elem.citation = Some(Some(citation));
        elem.element = Some(None);

        if !BibliographyElem::has(engine, elem.target, span)
            && let Ok(found) =
                engine.introspect(QueryLabelIntrospection(elem.target, span))
        {
            elem.element = Some(Some(found));
            return Ok(());
        }

        Ok(())
    }
}
```

**作用：**
- 在早期阶段尝试查询目标元素
- 将查询结果存储在 `element` 合成字段中
- 用于 show rule 中访问被引用元素（如文档中示例：`it.element`）
- 如果是文献引用，转换为 CiteElem

### 5.2 Realize 阶段

在 [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L228-L325)：

```rust
impl Packed<RefElem> {
    pub fn realize(&self, engine: &mut Engine, styles: StyleChain) -> SourceResult<Content> {
        let span = self.span();
        // 再次查询目标元素（使用最新的 introspector）
        let elem = engine.introspect(QueryLabelIntrospection(self.target, span));
        let elem = elem.at(span)?;

        let form = self.form.get(styles);
        if form == RefForm::Page {
            // 页码引用：查询页码编号
            let loc = elem.location().unwrap();
            let numbering = engine.introspect(PageNumberingIntrospection(loc, span))?;
            let supplement = engine.introspect(PageSupplementIntrospection(loc, span));
            return realize_reference(self, engine, styles, 
                Counter::new(CounterKey::Page), numbering, supplement, elem);
        }

        // 普通引用：查询元素的编号
        let refable = elem.with::<dyn Refable>().ok_or_else(...)?;
        let numbering = refable.numbering().ok_or_else(...)?;

        realize_reference(self, engine, styles,
            refable.counter(), numbering.clone(), refable.supplement(), elem)
    }
}
```

**realize_reference 生成最终内容：**

```rust
fn realize_reference(...) -> SourceResult<Content> {
    let loc = elem.location().unwrap();
    // 显示计数器在该位置的值
    let numbers = counter.display_at(engine, loc, styles, &numbering.trimmed(), span)?;
    
    // 处理补充文本（如 "Section"、"Figure"）
    let supplement = match reference.supplement.get_ref(styles) {
        Smart::Auto => supplement,
        Smart::Custom(None) => Content::empty(),
        Smart::Custom(Some(supplement)) => supplement.resolve(engine, styles, [elem])?,
    };

    // 构建内容：补充文本 + 不换行空格 + 编号
    let mut content = numbers;
    if !supplement.is_empty() {
        content = supplement + TextElem::packed("\u{a0}") + content;
    }

    // 包装为链接，指向目标位置
    Ok(DirectLinkElem::new(loc, content, Some(alt)).pack().spanned(span))
}
```

---

## 六、Outline（目录）的生成过程

### 6.1 查询阶段

在 [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/outline.rs#L299-L318)：

```rust
fn realize_iter(&self, engine: &mut Engine, styles: StyleChain) -> impl Iterator<...> {
    let span = self.span();
    // 查询所有匹配的元素（如所有 heading）
    let elems = engine.introspect(QueryIntrospection(self.target.get_cloned(styles).0, span));
    let depth = self.depth.get(styles).unwrap_or(NonZeroUsize::MAX);
    
    elems.into_iter().map(move |elem| {
        let outlinable = elem.with::<dyn Outlinable>()?;
        let level = outlinable.level();
        let include = outlinable.outlined() && level <= depth;
        let entry = Packed::new(OutlineEntry::new(level, elem)).spanned(span);
        Ok((entry, level, include))
    })
}
```

### 6.2 OutlineEntry 的方法

**prefix()** - 生成前缀（如 "1.1"、"Figure 1"）：
```rust
pub fn prefix(&self, engine: &mut Engine, ...) -> SourceResult<Option<Content>> {
    let outlinable = self.outlinable().at(span)?;
    let Some(numbering) = outlinable.numbering() else { return Ok(None) };
    let loc = self.element_location().at(span)?;
    // 显示计数器在该位置的值
    let numbers = outlinable.counter().display_at(engine, loc, styles, numbering, span)?;
    Ok(Some(outlinable.prefix(numbers)))
}
```

**page()** - 生成页码：
```rust
pub fn page(&self, engine: &mut Engine, ...) -> SourceResult<Content> {
    let loc = self.element_location().at(span)?;
    // 查询该位置的页码编号
    let numbering = engine.introspect(PageNumberingIntrospection(loc, span))
        .unwrap_or_else(|| NumberingPattern::from_str("1").unwrap().into());
    // 显示页码计数器在该位置的值
    Counter::new(CounterKey::Page).display_at(engine, loc, styles, &numbering, span)
}
```

### 6.3 自动缩进的特殊依赖

自动缩进（`indent: auto`）需要测量前缀宽度，这引入了额外的依赖：

```rust
pub fn indented(&self, engine: &mut Engine, ...) -> SourceResult<Content> {
    // 测量当前前缀宽度
    let prefix_width = prefix.as_ref().map(|prefix| 
        measure_prefix(engine, prefix, outline_loc, styles, span)
    ).transpose()?;
    
    // 查询所有 PrefixInfo 元素来计算自动缩进
    let (base_indent, hanging_indent) = match &indent {
        Smart::Auto => compute_auto_indents(
            engine, outline_loc, styles, self.level, prefix_inset, span
        ),
        // ...
    };

    // 插入 PrefixInfo 元素，供下一轮迭代查询
    let mut seq = Vec::with_capacity(5);
    if indent.is_auto() {
        seq.push(PrefixInfo::new(outline_loc, self.level, prefix_inset).pack());
    }
    // ...
}
```

**compute_auto_indents 查询 PrefixInfo：**

```rust
fn compute_auto_indents(...) -> (Rel, Option<Abs>) {
    let elems = engine.introspect(QueryIntrospection(
        select_where!(PrefixInfo, key => outline_loc),
        span,
    ));
    let indents = determine_prefix_widths(&elems);
    // 根据各层级最大前缀宽度计算缩进
    // ...
}
```

这形成了一个特殊的收敛循环：
```
迭代 N：生成 PrefixInfo 元素（基于迭代 N-1 的前缀宽度）
迭代 N+1：读取这些 PrefixInfo，计算新的缩进，生成新的 PrefixInfo
直到缩进稳定
```

---

## 七、收敛时序详解

### 7.1 典型文档的收敛过程

以一个包含目录和引用的文档为例：

```typ
#set heading(numbering: "1.")
#outline()

= Introduction <intro>
See @methods for details.

= Methods <methods>
...
```

**迭代 1（使用 EmptyIntrospector）：**
- `QueryLabelIntrospection(<intro>)` → 报错（label 不存在）
- `QueryIntrospection(heading)` → 空列表
- `outline()` 生成空目录
- 实际内容排版，生成所有 heading 和它们的 location
- 产生 Introspector 1，包含所有标签和元素的位置

**迭代 2（使用 Introspector 1）：**
- `@intro` 查询到 `<intro>` 元素，生成 "Section 1"
- `@methods` 查询到 `<methods>` 元素，生成 "Section 2"
- `outline()` 查询到 2 个 heading，生成目录条目
- 目录占据一定空间，导致后续内容页码可能变化
- 产生 Introspector 2

**迭代 3（使用 Introspector 2）：**
- 如果目录空间导致页码变化，目录中的页码需要更新
- 如果使用自动缩进，PrefixInfo 可能需要调整
- 产生 Introspector 3

**迭代 4（如果需要）：**
- 检查 Introspector 3 是否与 Introspector 2 相同
- 如果相同 → 收敛，使用 Introspector 3 的结果
- 如果不同 → 继续迭代

### 7.2 收敛的条件

收敛当且仅当：**连续两次迭代产生的 Introspector 完全相同**

这意味着：
- 所有元素的位置（页码、坐标）不再变化
- 所有计数器的显示值不再变化
- 所有查询结果不再变化
- 所有自动缩进的前缀宽度不再变化

### 7.3 不收敛的情况

当文档存在"自引用"时，可能无法在5次迭代内收敛。文档中的示例：

```typ
= Real
#context {
    let elems = query(heading)
    let count = elems.len()
    count * [= Fake]
}
```

迭代过程：
- 迭代 1：1 个 heading → 生成 1 个 Fake
- 迭代 2：2 个 heading → 生成 2 个 Fake
- 迭代 3：3 个 heading → 生成 3 个 Fake
- ...
- 永远不会收敛

---

## 八、关键代码路径总结

### 8.1 Label 的生命周期

1. **解析**：`<name>` 语法解析为 `Label` 值
2. **附着**：在 realize 阶段，标签附着到前一个元素
3. **索引**：在布局阶段，`ElementIntrospectorBuilder` 构建 `labels` 索引
4. **查询**：通过 `QueryLabelIntrospection` 查询

### 8.2 Ref 的完整路径

```
解析 @target → RefElem { target: Label }
    ↓
Synthesize 阶段
    ↓ engine.introspect(QueryLabelIntrospection(target))
    ↓ 存储到 element 字段
    ↓
Realize 阶段
    ↓ engine.introspect(QueryLabelIntrospection(target)) （再次查询）
    ↓ engine.introspect(PageNumberingIntrospection(loc)) （页码引用时）
    ↓ counter.display_at(engine, loc, ...) （显示编号）
    ↓
生成 DirectLinkElem { location, content }
```

### 8.3 Outline 的完整路径

```
#outline() → OutlineElem { target: heading.select() }
    ↓
realize_flat / realize_tree
    ↓ engine.introspect(QueryIntrospection(target))
    ↓ 生成 OutlineEntry 列表
    ↓
每个 OutlineEntry 的 show 规则
    ↓ entry.prefix() → counter.display_at(...)
    ↓ entry.page()   → engine.introspect(PageNumberingIntrospection(...))
    ↓ entry.indented()
        ↓ measure_prefix(...)
        ↓ engine.introspect(QueryIntrospection(PrefixInfo))
        ↓ 插入 PrefixInfo 元素供下轮使用
    ↓
生成链接到目标位置的目录条目
```

### 8.4 编译收敛循环

```
eval() → content
    ↓
loop {
    选择 introspector（上一轮的结果或空）
        ↓
    T::create(engine, content, styles)
        → realize()
            → Synthesize
            → RefElem::realize() → introspect 查询
            → OutlineElem::realize_flat() → introspect 查询
        → layout()
            → 生成 Tag::Start / Tag::End
            → ElementIntrospectorBuilder 收集元素
            → 构建 Introspector
        ↓
    constraint.validate(new_introspector)
        → 相同 → break（收敛）
        → 不同 → 继续迭代
}
```

---

## 九、设计亮点与权衡

### 9.1 使用 comemo Constraint 进行收敛检测

**优势：**
- 不需要手动记录所有依赖
- 只验证实际访问过的数据，效率高
- 与增量计算系统无缝集成

**注意：**
> comemo 验证失败但文档实际已收敛的情况：当自定义的 Introspection 过滤了 comemo 观察到的数据时。此时 `analyze()` 会返回零诊断，不会发出收敛警告。

### 9.2 Introspection Output 的粒度选择

**关键原则**：Output 类型应恰好包含需要稳定的信息，不多不少。

例如：
- `location.page()` 可能比 `location.position().page` 提前一轮收敛
- `query(...).len() > 0` 作为 Introspection Output 比完整查询结果提前收敛

### 9.3 5 次迭代上限

**设置原因**：
- 大多数文档在 2-3 次迭代内收敛
- 页码、目录缩进等通常需要额外 1-2 次
- 5 次是安全性与性能的平衡点

**代价**：
- 极端情况下可能产生假阳性警告（文档实际收敛但需要更多迭代）
- 理论上可以通过"额外编译一次确认"解决，但目前未实现

### 9.4 Synthesize 与 Realize 分离

**Synthesize（早期）**：
- 提供元素信息给 show rule
- 可以容忍查询失败（element 可能为 None）

**Realize（晚期）**：
- 生成最终排版内容
- 查询失败会产生错误

这种分离允许用户在 show rule 中优雅处理尚未发现的元素：
```typ
#show ref: it => {
    let el = it.element
    if el == none { return it }  // 处理元素尚未发现的情况
    // 自定义渲染
}
```

---

## 十、相关文件速查表

| 文件 | 内容 |
|------|------|
| [label.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/foundations/label.rs) | Label 类型定义 |
| [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs) | RefElem 定义与实现 |
| [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/outline.rs) | OutlineElem 与 OutlineEntry |
| [convergence.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs) | Introspect trait、收敛检测 |
| [query.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/query.rs) | 各种 Introspection 类型 |
| [introspector.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/introspector.rs) | Introspector trait 与 ElementIntrospector |
| [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst/src/lib.rs) | 编译主循环与收敛逻辑 |
