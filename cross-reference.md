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

## 三、Label 附着到可内省元素的完整路径

Label 从语法到最终可被查询，经历五个阶段：**求值附着 → 准备打标 → 布局嵌入 → 内省器构建 → 标签查询**。

### 3.1 求值阶段：Label 附着到 Content

在 [markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-eval/src/markup.rs#L53-L64) 中，当求值遇到 `Value::Label` 时，**向前回溯**找到第一个非 `Unlabellable` 的元素，调用 `.labelled(label)` 将标签写入 `Content` 的元数据：

```rust
Value::Label(label) => {
    if let Some(elem) =
        seq.iter_mut().rev().find(|node| !node.can::<dyn Unlabellable>())
    {
        if elem.label().is_some() {
            vm.engine.sink.warn(/* 标签重复警告 */);
        }
        *elem = std::mem::take(elem).labelled(label);
    } else {
        vm.engine.sink.warn(/* 标签未附着警告 */);
    }
}
```

[Content::labelled()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/foundations/content/mod.rs#L121-L124) 内部调用 `set_label`，将标签写入 `meta().label`：

```rust
pub fn labelled(mut self, label: Label) -> Self {
    self.set_label(label);
    self
}
pub fn set_label(&mut self, label: Label) {
    self.0.meta_mut().label = Some(label);
}
```

**关键**：此时 Label 只是 Content 元数据中的一个字段，还没有 Location，也还没有进入 Introspector。

### 3.2 准备阶段：分配 Location、生成 Tag

在 [realize/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-realize/src/lib.rs#L535-L591) 的 `prepare()` 函数中，系统为有标签的元素分配 Location 并生成 `Tag`：

```rust
fn prepare(engine, locator, elem, map, styles) -> SourceResult<Option<(Tag, Tag)>> {
    let key = typst_utils::hash128(&elem);
    let flags = TagFlags {
        // 三种情况使元素可内省：Locatable / 有 label / 手动设了 location
        introspectable: elem.can::<dyn Locatable>()
            || elem.label().is_some()
            || elem.location().is_some(),
        tagged: elem.can::<dyn Tagged>(),
    };
    // 为没有 location 的元素分配一个
    if elem.location().is_none() && flags.any() {
        let loc = locator.next_location(engine, key, elem.span());
        elem.set_location(loc);
    }

    // ... Synthesize, materialize ...

    // 生成 Start/End 标签对，Start 中包含元素的完整 Content 克隆
    let tags = elem
        .location()
        .map(|loc| (Tag::Start(elem.clone(), flags), Tag::End(loc, key, flags)));

    elem.mark_prepared();
    Ok(tags)
}
```

[Tag](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/tag.rs#L12-L24) 是标记可内省元素起止的枚举：

```rust
pub enum Tag {
    Start(Content, TagFlags),
    End(Location, u128, TagFlags),
}
```

[TagFlags](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/tag.rs#L46-L55) 中的 `introspectable` 标志决定该元素是否会被插入 Introspector：

```rust
pub struct TagFlags {
    /// 元素会被插入 Introspector：
    /// 因为它是 Locatable、有 label、或手动设了 location
    pub introspectable: bool,
    pub tagged: bool,
}
```

生成的 Tag 对被包装为 [TagElem](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/tag.rs#L67-L73)，插入到 realize 的输出流中（Start 在元素内容之前，End 在之后）：

```rust
if let Some(tag) = start {
    visit(s, s.store(TagElem::packed(tag)), styles)?;
}
// ... visit realized content ...
if let Some(tag) = end {
    visit(s, s.store(TagElem::packed(tag)), styles)?;
}
```

### 3.3 布局阶段：Tag 进入 Frame

布局时，`TagElem` 被转换为 `FrameItem::Tag(tag)`，嵌在帧的 item 列表中，位置与元素在页面上的物理位置对应。

### 3.4 内省器构建阶段：从 Frame 中提取 Tag 并建立索引

[PagedIntrospector::new()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-layout/src/introspect.rs#L37-L58) 遍历所有页面帧，发现 Tag：

```rust
pub fn new(pages: &[Page]) -> PagedIntrospector {
    let mut builder = PagedIntrospectorBuilder::default();
    for (i, page) in pages.iter().enumerate() {
        let nr = NonZeroUsize::new(1 + i).unwrap();
        builder.discover_frame(&page.frame, Transform::identity(), &mut |point| {
            PagedPosition { page: nr, point }
        });
    }
    builder.finish(/* ... */)
}
```

[discover_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-layout/src/introspect.rs#L179-L209) 递归遍历帧树，遇到 `FrameItem::Tag` 时调用 `elements.discover_tag(tag, position)`：

```rust
fn discover_frame(&mut self, frame: &Frame, ts: Transform, to_pos: &mut F) {
    for (pos, item) in frame.items() {
        match item {
            FrameItem::Tag(tag) => {
                self.elements.discover_tag(tag, to_pos(pos.transform(ts)));
            }
            FrameItem::Group(group) => { /* 递归，处理 parent insertion */ }
            FrameItem::Link(dest, _) => { /* 收集链接目标 */ }
            FrameItem::Text(..) | FrameItem::Shape(..) | FrameItem::Image(..) => {}
        }
    }
}
```

[ElementIntrospectorBuilder::discover_tag()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/introspector.rs#L513-L530) 只处理 `introspectable` 的 Tag：

```rust
pub fn discover_tag(&mut self, tag: &Tag, position: P) {
    match tag {
        Tag::Start(elem, flags) => {
            if flags.introspectable {
                let loc = elem.location().unwrap();
                if self.seen.insert(loc) {
                    self.sink.push(BuilderItem::Start(elem.clone(), position));
                }
            }
        }
        Tag::End(loc, key, flags) => {
            if flags.introspectable {
                self.keys.insert(*key, *loc);
                self.sink.push(BuilderItem::End(*loc));
            }
        }
    }
}
```

[finalize()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/introspector.rs#L578-L594) 阶段的 `visit()` 方法在处理 `BuilderItem::Start` 时，如果元素有 label，将其插入 `labels` 索引：

```rust
fn visit(&mut self, elems: &mut Vec<(Content, P)>, item: BuilderItem<P>) {
    match item {
        BuilderItem::Start(elem, pos) => {
            let loc = elem.location().unwrap();
            let idx = elems.len();
            self.locations.insert(loc, idx..idx + 1);
            // 关键：有 label 的元素被插入 labels 索引
            if let Some(label) = elem.label() {
                self.labels.insert(label, idx);
            }
            elems.push((elem, pos));
            // 处理子元素插入
            if let Some(insertions) = self.insertions.take(&loc) { /* ... */ }
        }
        BuilderItem::End(loc) => { /* 更新 range end */ }
    }
}
```

### 3.5 查询阶段：通过 Label 查找元素

[QueryLabelIntrospection](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/query.rs#L257-L273) 调用 `introspector.query_label(label)`，最终通过 `labels` 索引直接定位：

```rust
pub fn query_label(&self, label: Label) -> StrResult<&Content> {
    match *self.labels.get(&label) {
        [idx] => Ok(self.get_by_idx(idx)),
        [] => bail!("label `{}` does not exist in the document", label.repr()),
        _ => bail!("label `{}` occurs multiple times in the document", label.repr()),
    }
}
```

### 3.6 完整路径总结

```
语法 <intro> → 解析为 Label 值
    ↓
求值 markup.rs：回溯找到非 Unlabellable 元素，调用 elem.labelled(label)
    ↓ （elem.meta.label = Some(label)，但还没有 Location）
    ↓
准备 realize/lib.rs::prepare()：
    ↓   检查 elem.label().is_some() → TagFlags.introspectable = true
    ↓   分配 Location → elem.set_location(loc)
    ↓   生成 Tag::Start(elem.clone(), flags) / Tag::End(loc, key, flags)
    ↓   包装为 TagElem::packed(tag) 插入 realize 输出流
    ↓
布局：TagElem → FrameItem::Tag(tag)，携带物理位置
    ↓
内省器构建 introspect.rs::discover_frame()：
    ↓   从 FrameItem::Tag 中提取 tag
    ↓   ElementIntrospectorBuilder::discover_tag(tag, position)
    ↓   仅 flags.introspectable 的 Tag 被收集
    ↓   finalize() 时 visit() 检查 elem.label()，插入 labels 索引
    ↓
查询：engine.introspect(QueryLabelIntrospection(label))
    ↓   → introspector.query_label(label)
    ↓   → 通过 labels 索引 O(1) 定位
    ↓
返回 Content（带有 Location、Label、所有字段）
```

---

## 四、Introspection（内省）系统

Introspection 是连接多次编译的桥梁，核心定义在 [convergence.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs)。

### 4.1 Introspect trait

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

### 4.2 关键 Introspection 类型

在 [query.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/query.rs) 中定义：

| 类型 | 用途 |
|------|------|
| `QueryIntrospection(Selector, Span)` | 查询所有匹配元素 |
| `QueryLabelIntrospection(Label, Span)` | 通过标签查询唯一元素 |
| `QueryFirstIntrospection(Selector, Span)` | 查询第一个匹配元素 |
| `QueryUniqueIntrospection(Selector, Span)` | 查询唯一匹配元素 |
| `PageNumberingIntrospection(Location, Span)` | 查询某位置的页码编号 |
| `PageSupplementIntrospection(Location, Span)` | 查询某位置的页码补充文本 |

### 4.3 Introspector

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
1. 布局完成后，从 Frame 中提取 `FrameItem::Tag`
2. `ElementIntrospectorBuilder` 收集 introspectable 的 Tag，构建索引：
   - `locations: FxHashMap<Location, Range<usize>>`：按位置快速查找
   - `labels: MultiMap<Label, usize>`：按标签快速查找
   - `elems: Vec<(Content, P)>`：所有元素的有序列表
3. 查询时使用这些加速结构，通过 `QueryCache` 缓存查询结果

### 4.4 engine.introspect()：查询与记录双轨

[engine.introspect()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/engine.rs#L109-L117) 在执行查询的同时，将 Introspection 记录到 Sink 中：

```rust
pub fn introspect<I>(&mut self, introspection: I) -> I::Output
where
    I: Introspect,
{
    let introspector = *self.introspector.access("is okay since we're recording it");
    let output = introspection.introspect(self, introspector);
    // 副作用：记录到 Sink，供收敛失败后诊断使用
    self.sink.introspection(Introspection::new(introspection));
    output
}
```

这些记录在 `comemo::Constraint` 验证通过时被丢弃；仅在验证失败且达到迭代上限时，才被传给 `analyze()` 做历史诊断。

---

## 五、编译主循环：两阶段收敛判断

编译主循环定义在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst/src/lib.rs#L99-L194)，最多迭代 5 次（`MAX_ITERS = 5`）。

**注意**：收敛判断并非单一机制，而是由两个不同层次的检查组成——**依赖约束验证**（comemo）和**历史输出诊断**（History）。前者在每次迭代后执行，是快速路径；后者仅在前者反复失败、达到迭代上限后才触发，用于生成面向用户的诊断信息。两者关注的粒度和语义不同，可能出现一方通过而另一方不通过的情况。

```rust
fn compile_impl<T: Output>(...) -> SourceResult<T> {
    let content = typst_eval::eval(...)?.content();

    let mut history: ArrayVec<T, { MAX_ITERS - 1 }> = ArrayVec::new();
    let mut document: T;

    loop {
        let introspector = history
            .last()
            .map(|doc| doc.introspector())
            .unwrap_or(&empty_introspector);

        // ---- 阶段 A：绑定 comemo Constraint ----
        let constraint = comemo::Constraint::new();
        let mut engine = Engine {
            introspector: Protected::new(introspector.track_with(&constraint)),
            // ...
        };

        document = T::create(&mut engine, &content, styles)?;

        // ---- 阶段 B-1：依赖约束验证（快速路径）----
        if constraint.validate(document.introspector()) {
            break;  // comemo 验证通过 → 约束跟踪的所有方法调用在新旧 introspector 上结果相同 → 收敛
        }

        // ---- 阶段 B-2：迭代上限后的历史诊断（慢路径）----
        if history.is_full() {
            // comemo 验证失败 + 已达 5 次上限
            // 将所有迭代对应的 introspector 传给 analyze()
            let mut introspectors =
                [&empty_introspector as &dyn Introspector; MAX_ITERS + 1];
            for i in 1..MAX_ITERS {
                introspectors[i] = history[i - 1].introspector();
            }
            introspectors[MAX_ITERS] = document.introspector();

            let warnings = typst_library::introspection::analyze(
                world,
                introspectors,
                subsink.introspections(),  // 编译期间记录的所有 Introspection
            );

            sink.extend_from_sink(subsink);
            for warning in warnings {
                sink.warn(warning);
            }
            break;
        }

        history.push(document);
    }
}
```

### 5.1 阶段 B-1：comemo 依赖约束验证

**做什么**：检查"用新的 Introspector 替换本轮的 Introspector 之后，约束**实际跟踪到的那些**方法调用是否都返回相同结果"。

**⚠️ 不是整个 Introspector 一致**：`constraint.validate(new_introspector)` 不做"新旧两个 Introspector 对象是否完全等价"的逐字段比较，也不检查所有可能的方法调用。它只验证**本迭代编译过程中，通过 `track_with(&constraint)` 绑定后，确实调用过、并被 comemo 缓存记录下来的那些 `Introspector` trait 方法**——在使用 `new_introspector` 调用时，返回值与本轮完全相同。没有被访问到的数据即便变了，也不会导致验证失败。

**原理**：
1. 每次迭代创建 `comemo::Constraint`
2. `introspector.track_with(&constraint)` 将内省器与约束绑定——从此之后，对这个 `introspector` 的 trait 方法调用会被约束"观察到"，并记录其签名、输入以及来自 comemo 缓存的返回值
3. 编译过程中，所有通过 `engine.introspect()` 间接触发的 `Introspector` 调用（如 `query_label`、`page`、`query` 等），都会进入约束的观察范围（因为 Introspector trait 带 `#[comemo::track]`）
4. `constraint.validate(new_introspector)` 的验证过程：**对约束中记录的每一条观察到的方法调用，以相同参数、`new_introspector` 作为 self，重新调用一次**；如果所有调用的返回值都与本轮缓存的值相同 → 验证通过；**任一条返回值不同 → 验证失败**

**语义**：验证的是**被访问依赖输入的稳定性**，即"本轮我读到了哪些 introspector 数据，这些数据在新 introspector 下返回值是否完全一致"。这是**结构级/输入级**的等价检查，但只覆盖**被实际访问过**的子集。

**通过条件**：约束中记录的所有 trait 方法调用，在新旧 introspector 之间返回值完全一致。

**关键推论**：
- 若新旧 introspector 之间某些字段不同，但这些字段对应的方法**本轮没被调用过** → 验证仍可能通过
- 因此，comemo 通过的条件弱于"整个 Introspector 内容一致"，但强于"语义输出一致"（见 5.4 节）

**局限**：comemo 看到的是原始输入。如果某个 Introspection 对原始输入做了过滤/归约，那么 comemo 看到变化不代表过滤后的输出也变化了。

### 5.2 阶段 B-2：History 历史输出诊断

**触发条件**：comemo 验证失败 **且** 已达 5 次迭代上限。

**做什么**：对每个编译期间记录的 `Introspection`，用全部 6 个 introspector（empty + 5 轮）分别重算 Output，比较最后两轮的 hash 是否相同。

[analyze()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs#L25-L70) 的逻辑：

```rust
pub fn analyze(
    world: Tracked<dyn World + '_>,
    introspectors: [&dyn Introspector; INSTANCES],
    introspections: &[Introspection],
) -> EcoVec<SourceDiagnostic> {
    let mut sink = Sink::new();
    for introspection in introspections {
        // 对每个 Introspection，用 6 个 introspector 重算，
        // 检查最后两轮输出是否相同
        if let Some(warning) = introspection.0.diagnose(world, introspectors) {
            sink.warn(warning);
        }
    }

    let mut diags = sink.warnings();
    // 只有存在具体诊断时，才发出总汇总警告
    if !diags.is_empty() {
        let summary = warning!(
            "document did not converge within five attempts";
            hint: "see {} additional warning{} for more details", ...;
            hint: "see https://typst.app/help/convergence for help";
        );
        diags.insert(0, summary);
    }
    diags
}
```

[Bounds::diagnose()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs#L174-L188) 内部调用 `History::compute()` 对 6 个 introspector 各算一遍：

```rust
fn diagnose(&self, world, introspectors) -> Option<SourceDiagnostic> {
    let history = History::compute(world, introspectors, |engine, introspector| {
        self.introspect(engine, introspector)
    });
    // 仅当最后两轮的 Output 不同时，才生成诊断
    (!history.converged()).then(|| self.diagnose(&history))
}
```

[History::converged()](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs#L240-L250) 比较最后两次迭代的 hash：

```rust
pub fn converged(&self) -> bool
where T: Hash,
{
    typst_utils::hash128(&self.0[MAX_ITERS - 1].1)
        == typst_utils::hash128(&self.0[MAX_ITERS].1)
}
```

**语义**：验证的是**Introspection 输出的稳定性**，即"特定查询在最后两轮给出的结果是否一致"。这是值级/输出级的等价检查。

**通过条件**：每个被记录的 Introspection，其最后两轮的 Output hash 相同。

### 5.3 两种机制的对比

| | comemo 依赖约束验证 | History 历史输出诊断 |
|---|---|---|
| **触发时机** | 每次迭代后 | 仅 comemo 失败 + 迭代上限后 |
| **检查对象** | 本轮编译中**实际调用过**的 Introspector trait 方法 | 编译期间显式记录的 `Introspection` 类型 |
| **检查粒度** | 被访问的原始输入级（具体方法的参数与返回值逐次比较） | 输出级（Introspect::Output 的 hash） |
| **等价语义** | "本轮我调用过的每一条 introspector 方法，在新 introspector 上返回值与本轮完全相同" | "每个 Introspection 在最后两轮给出的 Output 值相同" |
| **用途** | 判断是否继续迭代 | 生成面向用户的诊断信息 |
| **开销** | 轻（只重放约束记录的方法调用） | 重（对每个 Introspection 用 6 个 introspector 重算） |

### 5.4 两者可能不一致的情况

[convergence.rs 注释](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs#L37-L55) 明确描述了这种情况：

**情况 1：comemo 通过 → 文档一定收敛**
约束跟踪的所有方法调用在新旧 introspector 上返回值都相同 → 下一轮迭代读到的所有数据都和本轮一致 → 编译结果不会再变化 → 确定收敛。

**情况 2：comemo 失败 + History 通过 → 文档实际已收敛，不发出警告**
如果自定义的 Introspection 对原始查询做了过滤/归约（如将查询结果简化为布尔值），那么：
- comemo 可能看到某些**被调用过的**方法返回值变化了（例如 `query(selector)` 返回的列表长度或元素字段变了）→ 验证失败
- 但归约后的 Output（例如 `list.is_empty()`）没变 → History 认为已收敛

此时 `analyze()` 中每个 Introspection 的 `diagnose()` 都返回 `None`（因为 `history.converged()` 为 true），`diags` 为空，不会发出"document did not converge"的汇总警告。

```rust
// analyze() 中的关键判断：
if !diags.is_empty() {
    // 只有存在具体诊断时才发汇总警告
    diags.insert(0, summary);
}
```

**情况 3：comemo 失败 + History 失败 → 真正未收敛，发出警告**

另一种 comemo 失败但不影响结果的情况：**被访问方法返回值变了，但变化的方法与任何 Introspection 无关**（例如其他 comemo 追踪的非 introspection 数据变化）。由于 History 只关心记录下来的 Introspection，这类变化也不会被诊断出来，因此不发警告。

### 5.5 迭代 N+1 观察迭代 N 的结果

关键原则：**第 N+1 次迭代观察第 N 次迭代产生的 Introspector**

```
迭代 1: 使用 EmptyIntrospector → 产生 Introspector 1
迭代 2: 使用 Introspector 1 → 产生 Introspector 2
迭代 3: 使用 Introspector 2 → 产生 Introspector 3
...
直到 comemo 验证通过（收敛）
或 达到 5 次上限 → 进入 History 诊断
```

注意：comemo 验证的是"本轮**实际调用过的** introspector 方法，用新 introspector 重放时结果相同"——这**不是**"连续两次迭代的 Introspector 对象内容相同"，而是"迭代 N+1 看到的、与 N+1 自己实际访问相关的那部分数据，与迭代 N 产生的 introspector 上对应方法返回值相同"。History 诊断检查的则是另一件事："连续两次 Introspection Output 的 hash 值相同"。

---

## 六、Ref 的两阶段处理

Ref 元素的处理分为 **Synthesize（合成）** 和 **Realize（实现）** 两个阶段。

### 6.1 Synthesize 阶段

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

### 6.2 Realize 阶段

在 [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L228-L325)。

`RefElem::realize` 按 form 和目标元素类型，分为 5 条分支：**页码引用**、**Bibliography 引用**、**Footnote 引用**、**普通 Refable 引用**、以及**出错分支**（重复标签 / 无可引用接口）。

```rust
impl Packed<RefElem> {
    pub fn realize(&self, engine: &mut Engine, styles: StyleChain) -> SourceResult<Content> {
        let span = self.span();
        let elem = engine.introspect(QueryLabelIntrospection(self.target, span));

        let form = self.form.get(styles);
```

**分支 A：RefForm::Page —— 页码引用**（[reference.rs:L237-L260](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L237-L260)）

```rust
        if form == RefForm::Page {
            let elem = elem.at(span)?;
            let elem = elem.clone();

            let loc = elem.location().unwrap();
            // 查询目标位置的页码 Numbering
            let numbering = engine
                .introspect(PageNumberingIntrospection(loc, span))
                .ok_or_else(|| eco_format!("cannot reference without page numbering"))
                .hint(eco_format!(
                    "you can enable page numbering with `#set page(numbering: \"1\")`"
                ))
                .at(span)?;
            // 查询目标位置的页码 supplement
            let supplement = engine.introspect(PageSupplementIntrospection(loc, span));

            return realize_reference(
                self, engine, styles,
                Counter::new(CounterKey::Page),  // 用页码计数器
                numbering,
                supplement,
                elem,
            );
        }
```

**分支 B：BibliographyElem::has(target) —— 文献引用**（[reference.rs:L263-L276](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L263-L276)）

```rust
        // RefForm::Normal 的后续分支：
        if BibliographyElem::has(engine, self.target, span) {
            // 如果在 Bibliography 条目中存在此 label
            if let Ok(elem) = elem {
                // 冲突：文档中存在同名可内省元素
                bail!(
                    span,
                    "label `{}` occurs both in the document and a bibliography",
                    self.target.repr();
                    hint: "change either the {}'s label or the \
                           bibliography key to resolve the ambiguity",
                    elem.func().name();
                );
            }
            // 只在 Bibliography 中存在 → 转成 Citation
            return Ok(to_citation(self, engine, styles)?.pack().spanned(span));
        }
```

**分支 C：目标是 FootnoteElem —— 脚注引用**（[reference.rs:L280-L282](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L280-L282)）

```rust
        let elem = elem.at(span)?;  // 确保 elem 存在，否则前面已经出错

        if let Some(footnote) = elem.to_packed::<FootnoteElem>() {
            // 转为脚注引用（把 @myfootnote 渲染为脚注上标编号的链接）
            return Ok(footnote.into_ref(self.target).pack().spanned(span));
        }
```

**分支 D：目标实现了 Refable —— 普通元素引用**（[reference.rs:L284-L323](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs#L284-L323)）

```rust
        let elem = elem.clone();
        let refable = elem
            .with::<dyn Refable>()
            .ok_or_else(|| {
                if elem.can::<dyn Figurable>() {
                    eco_format!(
                        "cannot reference {} directly, try putting it into a figure",
                        elem.func().name()
                    )
                } else {
                    eco_format!("cannot reference {}", elem.func().name())
                }
            })
            .at(span)?;

        let numbering = refable
            .numbering()
            .ok_or_else(|| {
                eco_format!("cannot reference {} without numbering", elem.func().name())
            })
            .hint(eco_format!(
                "you can enable {} numbering with `#set {}(numbering: \"1.\")`",
                elem.func().name(),
                if elem.func() == EquationElem::ELEM {
                    "math.equation"
                } else {
                    elem.func().name()
                }
            ))
            .at(span)?;

        realize_reference(
            self, engine, styles,
            refable.counter(),          // 元素自身的计数器（heading counter, figure counter 等）
            numbering.clone(),
            refable.supplement(),       // 元素的默认补充文本（如 "Section"、"Figure"）
            elem,
        )
    }
}
```

**各分支的关键差异：**

| 分支 | 触发条件 | 使用的 Counter | 使用的 Numbering | 最终形态 |
|---|---|---|---|---|
| A 页码引用 | `form: page` | `Counter::Page` | `PageNumberingIntrospection` 查询 | 直接链接到目标位置的页码文本 |
| B 文献引用 | `BibliographyElem::has(target)` + 文档中无同名标签 | ——（由 `to_citation` 处理） | —— | `CiteElem` |
| C 脚注引用 | 目标是 `FootnoteElem` | ——（由 `footnote.into_ref` 处理） | —— | 脚注编号链接 |
| D 普通引用 | 目标实现了 `Refable` | `refable.counter()` | `refable.numbering()` | 补充文本 + 不换行空格 + 计数器编号 |

**realize_reference 生成最终内容：**

```rust
fn realize_reference(...) -> SourceResult<Content> {
    let loc = elem.location().unwrap();
    let numbers = counter.display_at(engine, loc, styles, &numbering.trimmed(), span)?;

    let supplement = match reference.supplement.get_ref(styles) {
        Smart::Auto => supplement,
        Smart::Custom(None) => Content::empty(),
        Smart::Custom(Some(supplement)) => supplement.resolve(engine, styles, [elem])?,
    };

    let mut content = numbers;
    if !supplement.is_empty() {
        content = supplement + TextElem::packed("\u{a0}") + content;
    }

    Ok(DirectLinkElem::new(loc, content, Some(alt)).pack().spanned(span))
}
```

---

## 七、Outline（目录）的生成过程

### 7.1 查询阶段

在 [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/outline.rs#L299-L318)：

```rust
fn realize_iter(&self, engine: &mut Engine, styles: StyleChain) -> impl Iterator<...> {
    let span = self.span();
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

### 7.2 OutlineEntry 的方法

**prefix()** - 生成前缀（如 "1.1"、"Figure 1"）：
```rust
pub fn prefix(&self, engine: &mut Engine, ...) -> SourceResult<Option<Content>> {
    let outlinable = self.outlinable().at(span)?;
    let Some(numbering) = outlinable.numbering() else { return Ok(None) };
    let loc = self.element_location().at(span)?;
    let numbers = outlinable.counter().display_at(engine, loc, styles, numbering, span)?;
    Ok(Some(outlinable.prefix(numbers)))
}
```

**page()** - 生成页码：
```rust
pub fn page(&self, engine: &mut Engine, ...) -> SourceResult<Content> {
    let loc = self.element_location().at(span)?;
    let numbering = engine.introspect(PageNumberingIntrospection(loc, span))
        .unwrap_or_else(|| NumberingPattern::from_str("1").unwrap().into());
    Counter::new(CounterKey::Page).display_at(engine, loc, styles, &numbering, span)
}
```

### 7.3 自动缩进的特殊依赖

自动缩进（`indent: auto`）需要测量前缀宽度，这引入了额外的依赖：

```rust
pub fn indented(&self, engine: &mut Engine, ...) -> SourceResult<Content> {
    let prefix_width = prefix.as_ref().map(|prefix|
        measure_prefix(engine, prefix, outline_loc, styles, span)
    ).transpose()?;

    let (base_indent, hanging_indent) = match &indent {
        Smart::Auto => compute_auto_indents(
            engine, outline_loc, styles, self.level, prefix_inset, span
        ),
        // ...
    };

    let mut seq = Vec::with_capacity(5);
    if indent.is_auto() {
        seq.push(PrefixInfo::new(outline_loc, self.level, prefix_inset).pack());
    }
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

## 八、收敛时序详解

### 8.1 典型文档的收敛过程

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
- `QueryLabelIntrospection(<intro>)` → label 不存在
- `QueryIntrospection(heading)` → 空列表
- `outline()` 生成空目录
- 实际内容排版，生成所有 heading 和它们的 location
- 产生 Introspector 1，包含所有标签和元素的位置
- comemo 验证：对 EmptyIntrospector 调用过的方法（如 `query_label(<intro>)` → 失败、`query(heading)` → 空），用 Introspector 1 重放时结果**不同** → 验证失败

**迭代 2（使用 Introspector 1）：**
- `@intro` 查询到 `<intro>` 元素，生成 "Section 1"
- `@methods` 查询到 `<methods>` 元素，生成 "Section 2"
- `outline()` 查询到 2 个 heading，生成目录条目
- 目录占据一定空间，导致后续内容页码可能变化
- 产生 Introspector 2
- comemo 验证：对 Introspector 1 调用过的方法（如 `query_label(<intro>)` → 返回第 1 版 heading、`page(loc_of_intro)` → 第 1 版页码），用 Introspector 2 重放时若有任何一条返回值不同 → 验证失败

**迭代 3（使用 Introspector 2）：**
- 如果目录空间导致页码变化，目录中的页码需要更新
- 如果使用自动缩进，PrefixInfo 可能需要调整
- 产生 Introspector 3
- comemo 验证：逐条重放本轮调用过的 Introspector 方法，全相同 → 通过（收敛）；有一条不同 → 继续迭代

**迭代 4+（如果需要）：**
- comemo 验证通过 → 收敛，退出循环
- 若 5 轮后 comemo 仍失败 → 进入 History 诊断

### 8.2 comemo 验证通过 ≠ History 诊断通过 ≠ "整个 Introspector 相同"

四者的语义层次不同（从强到弱）：

1. **整个 Introspector 对象相同**（最强）：逐字段、逐索引完全等价。comemo 验证**不要求**达到这一点——没被访问过的数据即便是不同的，也不影响验证结果。

2. **comemo 验证通过**：本轮编译中**实际调用过的每一条** Introspector trait 方法，在用新 introspector 作为 self、相同参数重放时，返回值与本轮完全相同。→ 文档**确定**收敛，因为下一轮迭代读到的数据（调用结果）和本轮完全一致。

3. **History 诊断通过**：每个 `Introspection` 在最后两轮给出的 `Output` hash 相同。这只说明特定查询的**归约后输出**稳定了，但 comemo 跟踪到的原始输入可能还在变化（只是变化被 Introspection 的过滤逻辑丢掉了）。→ 文档**可能**已收敛（从用户可观察的语义上说），但 comemo 层面仍不稳定。

4. **两者都失败**：真正未收敛。

### 8.3 不收敛的情况

当文档存在"自引用"时，可能无法在5次迭代内收敛：

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
- comemo 始终不通过 → 5 轮后进入 History 诊断 → History 也不通过 → 发出收敛警告

---

## 九、关键代码路径总结

### 9.1 Label 的完整生命周期

```
语法 <intro> → 解析为 Label 值
    ↓
求值 markup.rs：回溯找到非 Unlabellable 元素 → elem.labelled(label)
    ↓ （meta.label = Some(label)，尚无 Location）
    ↓
准备 realize/lib.rs::prepare()：
    ↓   label.is_some() → TagFlags.introspectable = true
    ↓   分配 Location → 生成 Tag::Start / Tag::End
    ↓   包装为 TagElem 插入输出流
    ↓
布局：TagElem → FrameItem::Tag
    ↓
内省器构建：discover_frame → discover_tag → BuilderItem
    ↓   finalize() 中 visit() 检查 elem.label() → labels.insert(label, idx)
    ↓
查询：engine.introspect(QueryLabelIntrospection) → query_label(label) → labels 索引
```

### 9.2 Ref 的完整路径

```
解析 @target → RefElem { target: Label }
    ↓
Synthesize 阶段
    ↓   BibliographyElem::has(target) ? 是 → to_citation
    ↓   否 → engine.introspect(QueryLabelIntrospection(target)) → 存 element 字段
    ↓
Realize 阶段
    ↓ engine.introspect(QueryLabelIntrospection(target))
    ↓
    ├── 分支 A：form == Page
    │       ↓   engine.introspect(PageNumberingIntrospection(loc))
    │       ↓   engine.introspect(PageSupplementIntrospection(loc))
    │       ↓   Counter::Page.display_at(loc)
    │       ↓   realize_reference → 页码链接
    │
    ├── 分支 B：BibliographyElem::has(target)
    │       ↓   elem 存在？→ 报错（文档和文献库都有）
    │       ↓   elem 不存在 → to_citation → CiteElem
    │
    ├── 分支 C：目标是 FootnoteElem
    │       ↓   footnote.into_ref(target) → 脚注编号链接
    │
    └── 分支 D：目标实现 Refable
            ↓   refable.counter() / refable.numbering() / refable.supplement()
            ↓   counter.display_at(loc)
            ↓   realize_reference → 补充文本 + 编号的链接
```

### 9.3 Outline 的完整路径

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

### 9.4 编译收敛循环

```
eval() → content
    ↓
loop {
    选择 introspector（上一轮的结果或空）
        ↓
    创建 comemo::Constraint，调用 introspector.track_with(&constraint)
        ↓   此后对该 introspector 的所有 trait 方法调用都被约束观察
        ↓
    T::create(engine, content, styles)
        → realize()
            → prepare()：label → TagFlags → Tag → TagElem
            → Synthesize：engine.introspect()  查询并记录 Introspection
            → RefElem::realize()
                → QueryLabelIntrospection
                → Page / Bibliography / Footnote / Refable 分支
                → 进一步的 PageNumberingIntrospection 等
            → OutlineElem::realize_flat()
                → QueryIntrospection(heading)
                → PrefixInfo 查询/生成
        → layout()
            → TagElem → FrameItem::Tag
            → PagedIntrospector::new() → discover_tag → labels 索引
            → 生成新 Introspector（带所有元素的最新位置/页码）
        ↓
    constraint.validate(new_introspector)
        → 逐条重放约束中记录的每一次 Introspector trait 方法调用，
          以相同参数、new_introspector 作为 self
        → 所有调用返回值与本轮完全相同 → 通过 → break（确定收敛）
        → 有任何一条不同 → 失败
            → 未达迭代上限 → push document，继续下一轮
            → 已达 5 次上限 → History 诊断
                → 对每个记录的 Introspection，用 6 个 introspector 重算
                → 最后两轮 Output hash 相同？→ 认为已收敛，不警告
                → Output hash 仍不同？→ 发出具体警告 + 汇总警告
                → break
}
```

---

## 十、设计亮点与权衡

### 10.1 comemo Constraint 作为快速收敛路径

**优势：**
- 不需要手动记录所有依赖——所有 Introspector trait 方法调用自动被约束观察
- 只验证**实际访问过**的方法，没调用过的 Introspector 数据即便不同也不影响结果
- 与增量计算系统无缝集成——验证过程就是重放缓存过的方法调用
- 验证通过即可**确定**收敛：因为下一轮用到的所有 introspector 数据（调用返回值）和本轮完全一致

**⚠️ 关键澄清：它不是"比较整个 Introspector"**

`constraint.validate(new_introspector)` 做的事情是：
1. 遍历本轮约束中记录的**每一条**观察到的 Introspector 方法调用（如 `query_label(<intro>)`、`page(loc1)`、`query(heading)` 等）
2. 对每一条：用相同参数、`new_introspector` 作为 self 调用一次
3. 比较返回值是否与本轮缓存的值相同
4. **所有都相同 → 通过；任一条不同 → 失败**

这不是对两个 Introspector 对象做 `==` 比较。如果新旧 Introspector 之间有差异，但差异对应的方法本轮**没被调用过**，验证仍然通过。因此，它的通过条件**弱于**"整个 Introspector 内容一致"。

**局限：**
- comemo 看到的是原始输入（方法返回值本身），不是归约后的 Introspection Output
- 因此可能"假阴性"：comemo 说没收敛（某个 query 返回的列表元素字段变了），但实际语义已稳定（如 Introspection 只关心 `list.is_empty()`）

### 10.2 History 诊断作为慢速但更精确的后备

**优势：**
- 检查的是用户可观察的 Introspection Output，语义更精确
- 能区分"comemo 看到变化但输出已稳定"的情况（不发出警告）
- 为每个未收敛的 Introspection 生成精确的诊断信息（告诉用户是哪个查询还在抖）

**代价：**
- 需要对每个 Introspection 用 6 个 introspector 重算，开销大
- 仅在达到迭代上限后才触发，不能用来提前终止迭代

### 10.3 Introspection Output 的粒度选择

**关键原则**：Output 类型应恰好包含需要稳定的信息，不多不少。

例如：
- `location.page()` 可能比 `location.position().page` 提前一轮收敛
- `query(...).len() > 0` 作为 Introspection Output 比完整查询结果提前收敛

这直接影响 History 诊断的收敛判断，但对 comemo 验证无影响（comemo 看原始输入）。

### 10.4 5 次迭代上限

**设置原因**：
- 大多数文档在 2-3 次迭代内 comemo 验证通过
- 页码、目录缩进等通常需要额外 1-2 次
- 5 次是安全性与性能的平衡点

**代价**：
- comemo 验证失败时，如果 History 诊断也失败，会发出警告
- 但存在"边界情况"：若 Introspection Output 在第 4 轮与第 5 轮恰好相同（History 通过），但第 5 轮与第 6 轮（如果存在）又会不同，则警告被抑制——这可能隐藏真正的非收敛。不过注释中提到，理论上可以"额外编译一次确认"来消除这种情况，目前未实现。

### 10.5 Synthesize 与 Realize 分离（含 Bibliography / Footnote 分支）

**Synthesize（早期）**：
- 提供元素信息给 show rule
- 可以容忍查询失败（element 可能为 None）
- Bibliography 类的标签在这一步已经转为 CiteElem 的引用

**Realize（晚期）**：
- 生成最终排版内容
- 包含 4 条分支：页码引用、Bibliography 引用（冲突检测 + CiteElem 生成）、Footnote 引用（`footnote.into_ref`）、普通 Refable 引用
- 查询失败会产生错误

这种分离允许用户在 show rule 中优雅处理尚未发现的元素：
```typ
#show ref: it => {
    let el = it.element
    if el == none { return it }
    // 自定义渲染
}
```

---

## 十一、相关文件速查表

| 文件 | 内容 |
|------|------|
| [label.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/foundations/label.rs) | Label 类型定义 |
| [content/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/foundations/content/mod.rs) | Content::labelled / set_label |
| [markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-eval/src/markup.rs) | 求值阶段 Label 回溯附着 |
| [reference.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/reference.rs) | RefElem 定义与四分支 realize |
| [footnote.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/footnote.rs) | FootnoteElem 与 into_ref |
| [bibliography.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/bibliography.rs) | BibliographyElem::has / to_citation |
| [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/model/outline.rs) | OutlineElem 与 OutlineEntry |
| [tag.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/tag.rs) | Tag、TagFlags、TagElem 定义 |
| [realize/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-realize/src/lib.rs) | prepare() 中 Label → TagFlags → Tag 的转换 |
| [convergence.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/convergence.rs) | Introspect trait、History、analyze() |
| [query.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/query.rs) | 各种 Introspection 类型 |
| [introspector.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/introspection/introspector.rs) | Introspector trait 与 ElementIntrospector |
| [introspect.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-layout/src/introspect.rs) | PagedIntrospector、discover_frame |
| [engine.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst-library/src/engine.rs) | engine.introspect() 与 Sink 记录 |
| [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/126-typst/crates/typst/src/lib.rs) | 编译主循环（comemo 约束 + History 诊断） |
