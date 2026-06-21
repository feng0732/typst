# Typst 编译主流程分析

本文档按代码执行顺序分析 Typst 文档从输入到产物的完整编译流程，重点阐述解析、求值、排版和后端输出四个阶段之间的协作机制。

---

## 整体架构概览

Typst 采用多 crate 分层架构，编译流程贯穿以下核心模块：

| 阶段 | Crate | 主要职责 |
|------|-------|----------|
| 解析 | `typst-syntax` | 文本 → 语法树 (AST) |
| 求值 | `typst-eval` | AST → 内容树 (Content) |
| 实现 | `typst-realize` | Content → 扁平化元素列表 |
| 排版 | `typst-layout` | 元素 → 页面帧 (Frame) |
| 输出 | `typst-pdf/svg/render/html/bundle` | 帧 → 最终文件 |

```
输入文本
    ↓
[解析] parse() → SyntaxNode
    ↓
[求值] eval() → Module { Content, .. }
    ↓
[内省循环] × N (最多5次)
    ├─ [实现] realize() → Vec<Pair> (Content + StyleChain)
    └─ [排版] layout_document() → PagedDocument
    ↓
[输出] export() → PDF / PNG / SVG / HTML / Bundle
```

---

## 1. CLI 入口层

### 1.1 编译命令入口

文件：[compile.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs)

入口函数 [compile()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L38-L48) 处理命令行参数，构建 `CompileConfig` 和 `SystemWorld`，然后调用 [compile_once()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L258-L314)。

### 1.2 单次编译与多目标分发

核心分发函数 [compile_and_export()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L317-L341) 根据输出格式选择不同的泛型参数调用 `typst::compile::<T>()`：

```rust
match config.output_format {
    OutputFormat::Pdf | OutputFormat::Png | OutputFormat::Svg => {
        let Warned { output, warnings } = typst::compile::<PagedDocument>(world);
        let result = output.and_then(|document| export_paged(&document, config));
        Warned { output: result, warnings }
    }
    OutputFormat::Html => {
        let Warned { output, warnings } = typst::compile::<HtmlDocument>(world);
        let result = output.and_then(|document| export_html(&document, config));
        Warned { output: result.map(|()| vec![config.output.clone()]), warnings }
    }
    OutputFormat::Bundle => {
        let Warned { output, warnings } = typst::compile::<Bundle>(world);
        let result = output.and_then(|bundle| export_bundle(bundle, config));
        Warned { output: result, warnings }
    }
}
```

---

## 2. 核心编译层

### 2.1 泛型编译入口

文件：[typst/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst/src/lib.rs)

主入口函数 [compile::<T>()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst/src/lib.rs#L74-L82) 是对外暴露的 API，内部调用 [compile_impl::<T>()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst/src/lib.rs#L99-L194)。

### 2.2 Output trait —— 多目标抽象

文件：[target.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/foundations/target.rs)

[Output trait](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/foundations/target.rs#L13-L30) 定义了三种输出目标的统一接口：

```rust
pub trait Output: Any {
    fn target() -> Target;
    fn create(engine: &mut Engine, content: &Content, styles: StyleChain) -> SourceResult<Self>;
    fn introspector(&self) -> &dyn Introspector;
}
```

三种实现：

| 类型 | Target | create() 实现位置 |
|------|--------|-------------------|
| `PagedDocument` | `Target::Paged` | [document.rs#L72-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-layout/src/document.rs#L72-L79) |
| `HtmlDocument` | `Target::Html` | [dom.rs#L90-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/dom.rs#L90-L96) |
| `Bundle` | `Target::Bundle` | [lib.rs#L65-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L65-L71) |

### 2.3 compile_impl 完整流程

[compile_impl()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst/src/lib.rs#L99-L194) 是编译流程的核心编排函数：

#### 步骤 1：前置准备 (L104-L131)

```rust
// 1. 获取标准库和样式链
let library = world.library();
let base = StyleChain::new(&library.styles);
let target = TargetElem::target.set(T::target()).wrap();
let styles = base.chain(&target);

// 2. 读取主源文件
let main = world.main();
let main = world.source(main).map_err(...)?;

// 3. 求值主文件 → Content
let content = typst_eval::eval(
    world, library, traced, sink.track_mut(),
    Route::default().track(), &main,
)?.content();
```

#### 步骤 2：内省循环 (L133-L185)

这是 Typst 最核心的机制之一。由于文档中存在交叉引用（如 `@label`、`counter(page)`、`locate()` 等），需要多次迭代直到文档稳定：

```rust
let mut history: ArrayVec<T, { MAX_ITERS - 1 }> = ArrayVec::new();
let mut document: T;

loop {
    // 使用上一次的 Introspector（首次用 EmptyIntrospector）
    let introspector = history.last()
        .map(|doc| doc.introspector())
        .unwrap_or(&empty_introspector);

    // 创建 Engine 和约束
    let constraint = comemo::Constraint::new();
    let mut engine = Engine {
        library, world,
        introspector: Protected::new(introspector.track_with(&constraint)),
        traced, sink: subsink.track_mut(),
        route: Route::default(),
    };

    // 调用 Output::create() → 内部包含 realize + layout
    document = T::create(&mut engine, &content, styles)?;

    // 检查是否稳定
    if constraint.validate(document.introspector()) {
        sink.extend_from_sink(subsink);
        break;
    }

    // 最多迭代 5 次
    if history.is_full() {
        // 分析不收敛原因并发出警告
        let warnings = typst_library::introspection::analyze(...);
        break;
    }

    history.push(document);
}
```

#### 步骤 3：延迟错误处理 (L187-L193)

在之前的步骤中，某些错误（如 show rule 执行失败）可能被延迟处理：

```rust
let delayed = sink.delayed();
if !delayed.is_empty() {
    return Err(delayed);
}

Ok(document)
```

---

## 3. 阶段一：解析 (Parsing)

### 3.1 解析入口

文件：[parser.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-syntax/src/parser.rs)

入口函数 [parse()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-syntax/src/parser.rs#L16-L21)：

```rust
pub fn parse(text: &str) -> SyntaxNode {
    let _scope = typst_timing::TimingScope::new("parse");
    let mut p = Parser::new(text, 0, SyntaxMode::Markup);
    markup_exprs(&mut p, true, syntax_set!(End));
    p.finish_into(SyntaxKind::Markup)
}
```

### 3.2 解析流程

1. **词法分析**：`Lexer` 将文本转换为 token 流
2. **语法分析**：`Parser` 递归下降解析，构建无类型 `SyntaxNode` 树
3. **AST 层**：`ast` 模块提供类型安全的 AST 访问层

### 3.3 关键解析函数

| 函数 | 作用 |
|------|------|
| [markup_expr()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-syntax/src/parser.rs#L87-L134) | 解析单个标记表达式（文本、标题、列表、公式等） |
| [markup_exprs()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-syntax/src/parser.rs#L50-L61) | 解析表达式序列 |
| `code_expr()` / `math_expr()` | 解析代码和数学模式 |

> **注意**：解析阶段不直接被 `compile_impl()` 调用，而是在 `typst_eval::eval()` 内部通过 `source.root()` 获取已缓存的解析结果。

---

## 4. 阶段二：求值 (Evaluation)

### 4.1 求值入口

文件：[typst-eval/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-eval/src/lib.rs)

核心函数 [eval()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-eval/src/lib.rs#L40-L97) 使用 `#[comemo::memoize]` 缓存结果：

```rust
#[comemo::memoize]
pub fn eval(
    world: Tracked<dyn World + '_>,
    library: &LazyHash<Library>,
    traced: Tracked<Traced>,
    sink: TrackedMut<Sink>,
    route: Tracked<Route>,
    source: &Source,
) -> SourceResult<Module> {
    // 1. 防循环检测
    let id = source.id();
    if route.contains(id) {
        panic!("Tried to cyclicly evaluate {:?}", id.vpath());
    }

    // 2. 创建 Engine 和 VM
    let introspector = EmptyIntrospector;
    let engine = Engine {
        library, world,
        introspector: Protected::new(introspector.track()),
        traced, sink,
        route: Route::extend(route).with_id(id),
    };

    let context = Context::none();
    let scopes = Scopes::new(Some(library));
    let root = source.root();
    let mut vm = Vm::new(engine, context.track(), scopes, root.span());

    // 3. 语法错误检查（非追踪模式下遇错终止）
    let (errors, warnings) = root.errors_and_warnings();
    for warning in warnings {
        vm.engine.sink.warn(warning.into());
    }
    if !errors.is_empty() && vm.inspected.is_none() {
        return Err(errors.into_iter().map(Into::into).collect());
    }

    // 4. 求值标记模块根
    let markup = root.cast::<ast::Markup>().unwrap();
    let output = markup.eval(&mut vm)?;

    // 5. 控制流检查（禁止顶层 return/break/continue）
    if let Some(flow) = vm.flow {
        bail!(flow.forbidden());
    }

    // 6. 组装 Module
    let name = id.vpath().file_stem().unwrap_or_default();
    Ok(Module::new(name, vm.scopes.top)
        .with_content(output)
        .with_file_id(id))
}
```

### 4.2 Eval trait

文件中定义了 [Eval trait](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-eval/src/lib.rs#L178-L184)，所有 AST 节点都实现此 trait：

```rust
pub trait Eval {
    type Output;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output>;
}
```

### 4.3 虚拟机 (Vm)

文件：[vm.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-eval/src/vm.rs)

[Vm](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-eval/src/vm.rs#L16-L28) 持有求值所需的全部状态：

```rust
pub struct Vm<'a> {
    pub engine: Engine<'a>,           // 编译引擎
    pub flow: Option<FlowEvent>,      // 控制流事件（return/break/continue）
    pub scopes: Scopes<'a>,           // 变量作用域栈
    pub inspected: Option<Span>,      // 追踪模式下的目标 span
    pub context: Tracked<'a, Context<'a>>, // 上下文（样式等）
}
```

### 4.4 求值阶段的输出

求值完成后返回 `Module`，其核心是 `Content`—— 一个结构化、样式化、与顺序无关的内容树表示。

---

## 5. 阶段三：实现 (Realization)

### 5.1 实现入口

文件：[typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-realize/src/lib.rs)

核心函数 [realize()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-realize/src/lib.rs#L45-L76)：

```rust
pub fn realize<'a>(
    kind: RealizationKind,
    engine: &mut Engine,
    locator: &mut SplitLocator,
    arenas: &'a Arenas,
    content: &'a Content,
    styles: StyleChain<'a>,
) -> SourceResult<Vec<Pair<'a>>> {
    let mut s = State {
        engine, locator, arenas,
        rules: match kind {
            RealizationKind::Bundle => BUNDLE_RULES,
            RealizationKind::Document { .. } => FLOW_RULES,
            RealizationKind::Fragment { .. } => FLOW_RULES,
            RealizationKind::Par => PAR_RULES,
            RealizationKind::Math => MATH_RULES,
        },
        sink: vec![],
        groupings: ArrayVec::new(),
        outside: matches!(kind, RealizationKind::Document { .. }),
        may_attach: false,
        saw_parbreak: false,
        kind,
    };

    visit(&mut s, content, styles)?;
    finish(&mut s)?;

    Ok(s.sink)
}
```

### 5.2 RealizationKind

文件：[routines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/routines.rs#L153-L169)

实现类型决定了分组规则和行为：

```rust
pub enum RealizationKind<'a> {
    Bundle,                           // Bundle 目标
    Document { info: &'a mut DocumentInfo }, // 文档根
    Fragment { kind: &'a mut FragmentKind }, // 容器内
    Par,                              // 段落内
    Math,                             // 数学模式
}
```

### 5.3 核心处理流程：visit()

[visit()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-realize/src/lib.rs#L244-L296) 按以下顺序处理每个 Content 元素：

```
Content
  ↓
├─ 是 TagElem? → 直接 push
├─ visit_kind_rules() → 按类型特殊转换
│   ├─ 数学模式：递归进入 EquationElem
│   ├─ 非数学模式：Mathy → EquationElem
│   └─ SymbolElem → TextElem
├─ visit_show_rules() → 应用 show 规则（核心！）
│   ├─ verdict() → 决定 show 规则应用策略
│   ├─ prepare() → 首次访问时执行
│   │   ├─ 生成 Location
│   │   ├─ 应用 show-set 规则
│   │   ├─ 合成字段 (Synthesize)
│   │   ├─ 物化样式
│   │   └─ 生成 Tag
│   └─ 应用 show 规则（用户定义 or 内置）
├─ 递归进入 SequenceElem
├─ 递归进入 StyledElem（带样式链扩展）
├─ visit_grouping_rules() → 分组处理
│   ├─ TEXTUAL 分组 → 文本 show 规则
│   ├─ PAR 分组 → 段落
│   ├─ CITES 分组 → 引用组
│   └─ LIST/ENUM/TERMS 分组 → 列表
├─ visit_filter_rules() → 过滤（空格、分段符等）
└─ 最终 push 到 sink
```

### 5.4 Show 规则裁决：verdict()

[verdict()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-realize/src/lib.rs#L439-L532) 是实现阶段最核心的逻辑：

1. **预合成**：对 `Synthesize` 元素先进行预合成以支持 `show figure.where(kind: table)` 等匹配
2. **遍历样式链**：查找匹配的 show 规则
   - show-set 规则 → 收集到 `map`
   - 转换型 show 规则 → 取第一个未应用的
3. **内置 show 规则**：无用户规则时使用 `engine.library.rules.get(target, elem)`
4. **返回裁决**：`Verdict { prepared, map, step }`

### 5.5 分组机制

分组规则定义在 [GroupingRule](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-realize/src/lib.rs#L113-L128)，按优先级嵌套处理：

| 分组 | 优先级 | 作用 |
|------|--------|------|
| TEXTUAL | 3 | 相邻文本 → 正则 show 规则 |
| CITES | 2 | 相邻引用 → CiteGroup |
| LIST/ENUM/TERMS | 2 | 列表项 → 列表元素 |
| PAR | 1 | 行内元素 → 段落 |

### 5.6 实现阶段的输出

输出为 `Vec<Pair<'a>>`，即 `Vec<(&'a Content, StyleChain<'a>)>`—— 扁平化的内容元素与对应样式链对。

---

## 6. 阶段四：排版 (Layout)

### 6.1 排版入口（Paged 目标）

文件：[typst-layout/src/pages/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-layout/src/pages/mod.rs)

[layout_document()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-layout/src/pages/mod.rs#L33-L48) → [layout_document_common()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-layout/src/pages/mod.rs#L128-L172)

```rust
fn layout_document_common(...) -> SourceResult<PagedDocument> {
    // 1. 准备 Engine 和样式
    let mut engine = Engine { ... };
    let styles = styles.to_map().outside();
    let styles = StyleChain::new(&styles);
    let arenas = Arenas::default();

    // 2. 初始化文档信息
    let mut info = DocumentInfo::default();
    info.populate(styles);
    info.populate_locale(styles);

    // 3. 调用 realize() → 扁平化元素列表
    let mut children = (engine.library.routines.realize)(
        RealizationKind::Document { info: &mut info },
        &mut engine, &mut locator, &arenas,
        content, styles,
    )?;

    // 4. 排版页面
    let pages = layout_pages(&mut engine, &mut children, &mut locator, styles)?;

    // 5. 返回 PagedDocument（含 Introspector）
    Ok(PagedDocument::new(pages, info))
}
```

### 6.2 页面排版：layout_pages()

[layout_pages()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-layout/src/pages/mod.rs#L175-L241) 处理页面级逻辑：

```rust
fn layout_pages(...) -> SourceResult<EcoVec<Page>> {
    // 1. 分段：收集 → 将元素按页面配置分片
    let items = collect(children, locator, styles);

    // 2. 并行排版页面运行
    let mut runs = engine.parallelize(
        items.iter().filter_map(|item| match item {
            Item::Run(children, initial, locator) => {
                Some((children, initial, locator.relayout()))
            }
            _ => None,
        }),
        |engine, (children, initial, locator)| {
            layout_page_run(engine, children, locator, *initial)
        },
    );

    // 3. 收集并终结页面（处理页奇偶、标签等）
    for item in &items {
        match item {
            Item::Run(..) => {
                let layouted = runs.next().unwrap()?;
                for layouted in layouted {
                    let page = finalize(engine, &mut counter, &mut tags, layouted)?;
                    pages.push(page);
                }
            }
            Item::Parity(parity, initial, locator) => {
                // 处理奇偶页
            }
            Item::Tags(items) => {
                // 收集跨页标签
            }
        }
    }

    Ok(pages)
}
```

### 6.3 排版阶段的输出

输出为 `PagedDocument`，包含：
- `pages: EcoVec<Page>` —— 每页一个 `Frame`（固定位置的布局项）
- `info: DocumentInfo` —— 文档元数据
- `introspector: Arc<PagedIntrospector>` —— 用于查询和下一次迭代

---

## 7. 阶段五：后端输出 (Export)

三种输出目标在编译阶段共享相同的 realize + create 框架，但在中间产物形态和导出写入方式上差异显著：

| 维度 | Paged (PDF/PNG/SVG) | HTML | Bundle |
|------|---------------------|------|--------|
| 编译产物 | `PagedDocument { pages, info }` | `HtmlDocument { output, info }` | `Bundle { files, introspector }` |
| 中间表示 | `Frame`（绝对定位的布局项树） | `HtmlNode`（语义化 DOM 树） | 混合：多文档各自独立编译 |
| 内省器 | `PagedIntrospector` | `HtmlIntrospector` | `BundleIntrospector`（聚合子内省器） |
| 导出写入 | 单文件字节流 | 单文件 HTML 字符串 | `VirtualFs` → 多文件并行写入磁盘 |
| 跨文档链接 | 无 | 无 | `LateLinkResolver` + 锚点 |

---

### 7.1 Paged 目标：Frame → 像素/矢量/PDF

#### 7.1.1 Frame 数据结构

文件：[frame.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/layout/frame.rs)

[Frame](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/layout/frame.rs#L18-L30) 是排版阶段的核心产物，每页对应一个 Frame：

```rust
pub struct Frame {
    size: Size,
    baseline: Option<Abs>,
    items: Arc<LazyHash<Vec<(Point, FrameItem)>>>,
    kind: FrameKind,
}
```

[FrameItem](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/layout/frame.rs#L486-L499) 枚举了帧内所有可能的布局项：

| 变体 | 描述 |
|------|------|
| `Group(GroupItem)` | 子帧 + 变换 + 裁剪（递归结构） |
| `Text(TextItem)` | 一段已塑形的文本 |
| `Shape(Shape, Span)` | 几何图形（填充/描边） |
| `Image(Image, Size, Span)` | 位图/矢量图 |
| `Link(Destination, Size)` | 超链接区域 |
| `Tag(Tag)` | 内省标签（起始/结束标记） |

Frame 是一棵**绝对定位**的树：每个 item 带有 `(Point, FrameItem)` 对，Point 是相对于父帧左上角的偏移。GroupItem 通过 `transform` 字段支持旋转/缩放等仿射变换，通过 `clip` 字段支持裁剪路径。

#### 7.1.2 PDF 导出

文件：[typst-pdf/src/convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs)

入口函数 [pdf()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/lib.rs#L36-L38) 调用 [convert()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L48-L95)，完整流程如下：

**1. 初始化 krilla 文档和全局上下文**

```rust
let mut document = Document::new_with(settings);
let named_destinations = collect_named_destinations(...);
let tags = tags::init(typst_document, options)?;
let mut gc = GlobalContext::new(...);
```

Typst 使用 [krilla](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L16) 库作为 PDF 后端，`GlobalContext` 负责管理字体映射、链接注解、标签树等跨页状态。

**2. 逐页转换 Frame → PDF 页面**

[convert_pages()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L97-L173) 遍历 `PagedDocument.pages()`，对每页：

1. 根据 Frame 尺寸 + bleed 构建 `PageSettings`（含页面标签、裁剪框等）
2. 调用 `document.start_page_with(settings)` 开启新页面，获取 `Surface`
3. 创建 [FrameContext](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L220-L226) 管理变换栈和链接注解
4. 调用 [handle_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L328-L391) 递归遍历 Frame 树

**3. Frame → Surface 递归映射**

[handle_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L328-L391) 是 PDF 导出的核心递归函数：

```
handle_frame(fc, frame, padding, fill, surface, gc)
  ├─ fc.push() → 保存变换状态
  ├─ 若 frame.kind == Hard → fc.register_container(size)
  ├─ 若 fill 存在 → handle_shape() 绘制背景
  ├─ fc.push() → 平移 padding
  ├─ 遍历 frame.items():
  │   ├─ FrameItem::Group → handle_group() → 递归 handle_frame()
  │   ├─ FrameItem::Text  → handle_text()  → 写入字形到 Surface
  │   ├─ FrameItem::Shape → handle_shape() → 路径/填充/描边
  │   ├─ FrameItem::Image → handle_image() → 嵌入图像 XObject
  │   ├─ FrameItem::Link  → handle_link()  → 记录链接注解
  │   └─ FrameItem::Tag   → tags::handle_start/end → 结构标签树
  ├─ fc.pop() → 恢复 padding
  └─ fc.pop() → 恢复变换状态
```

[FrameContext](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L220-L226) 维护一个 `Vec<State>` 变换栈，模拟 PDF 内容流的图形状态保存/恢复机制。`State` 追踪当前变换矩阵和第一个硬帧的变换/尺寸（用于渐变坐标解析）。

**4. 完成导出**

```rust
convert_pages(&mut gc, &mut document)?;          // 所有页面
attach_files(&gc, &mut document)?;                // 附件嵌入
let (doc_lang, tree) = tags::resolve(&mut gc)?;   // 解析标签树
document.set_outline(build_outline(&gc));          // 书签大纲
document.set_metadata(build_metadata(&gc, doc_lang)); // 元数据
document.set_tag_tree(tree);                       // 无障碍标签树
finish(document, gc, options.standards.config)     // 序列化为 Vec<u8>
```

[finish()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/convert.rs#L434-L438) 调用 krilla 的 `document.finish()` 将整个文档序列化为 PDF 字节流，并处理字体处理错误和 PDF/A、PDF/UA 等标准验证错误。

**CLI 写入**：[export_pdf()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L379-L388) 将 `Vec<u8>` 写入输出文件。

#### 7.1.3 PNG 导出

文件：[typst-render/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs)

[render()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs#L21-L48) 将单页 Frame 光栅化为 `tiny_skia::Pixmap`：

```rust
pub fn render(page: &Page, opts: &RenderOptions) -> sk::Pixmap {
    let size = page.frame.size() + bleed.sum_by_axis();
    let pixel_per_pt = opts.pixel_per_pt.get() as f32;
    let pxw = (pixel_per_pt * size.x.to_f32()).round().max(1.0) as u32;
    let pxh = (pixel_per_pt * size.y.to_f32()).round().max(1.0) as u32;
    let ts = sk::Transform::from_scale(pixel_per_pt, pixel_per_pt);
    let mut canvas = sk::Pixmap::new(pxw, pxh).unwrap();
    // 填充背景色
    if let Some(fill) = page.fill_or_white() { ... }
    // 递归渲染 Frame
    render_frame(&mut canvas, state, &page.frame);
    canvas
}
```

[render_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs#L186-L205) 递归遍历 FrameItem：

| FrameItem | 处理方式 |
|-----------|----------|
| `Group` | [render_group()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs#L208-L262) → 变换 + 裁剪遮罩 + 递归 render_frame |
| `Text` | `text::render_text()` → 逐字形光栅化到 Pixmap |
| `Shape` | `shape::render_shape()` → 路径填充/描边 |
| `Image` | `image::render_image()` → 缩放绘制到 Pixmap |
| `Link` | 忽略（光栅化无链接概念） |
| `Tag` | 忽略 |

[render_merged()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs#L50-L86) 可将多页纵向拼合为单张图。

**CLI 写入**：[export_image()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L462-L466) 支持模板化文件名（如 `page-{n}.png`），逐页渲染并写入。

#### 7.1.4 SVG 导出

文件：[typst-svg/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs)

[svg()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L32-L43) 将单页 Frame 转为 SVG 字符串：

```rust
pub fn svg(page: &Page, opts: &SvgOptions) -> String {
    let (size, ts) = page_bleed(page, opts);
    let mut renderer = SVGRenderer::new();
    let mut xml = XmlWriter::new(xml_options(opts.pretty));
    let mut svg = svg_header(&mut xml, size);
    let state = State::new(size);
    renderer.render_page(&mut svg, &state, ts, page);
    renderer.finalize(svg);
    xml.end_document()
}
```

[SVGRenderer](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L191-L228) 使用 [Deduplicator](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L481-L486) 对字形、渐变、铺排和裁剪路径进行去重，写入 `<defs>` 区域复用：

| 去重目标 | 前缀 | 作用 |
|----------|------|------|
| 字形 | `g` | 相同字形的 SVG 路径只定义一次 |
| 裁剪路径 | `c` | 相同裁剪路径复用 |
| 渐变 | `f` | 无变换渐变定义 |
| 渐变引用 | `r` | 带变换的渐变通过 `href` 引用 |
| 铺排 | `t`/`p` | Tiling 模式及引用 |

[render_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L313-L327) 递归遍历，为每个 FrameItem 生成对应 SVG 元素。与 PDF/PNG 不同的是，SVG 额外处理 `FrameItem::Link`，生成 `<a>` 元素。

[svg_merged()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L128-L156) 将多页纵向拼合为单个 SVG。

**CLI 写入**：同 PNG 路径，但输出 `.svg` 文件。

---

### 7.2 HTML 目标：DOM → HTML 字符串

#### 7.2.1 HTML 编译流程

文件：[typst-html/src/document.rs](file:///d:/fz/0601-2\solo-dogfeeding\code\120-typst\crates\typst-html\src\document.rs)

[html_document()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/document.rs#L25-L40) → [html_document_common()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/document.rs#L128-L218)：

```rust
fn html_document_common(...) -> SourceResult<HtmlDocument> {
    // 1. 同 Paged 目标：realize() → 扁平化元素列表
    let children = (engine.library.routines.realize)(
        RealizationKind::Document { info: &mut info },
        &mut engine, &mut locator, &arenas,
        content, styles,
    )?;

    // 2. 不同：将扁平化元素转为 HtmlNode 树（而非排版为 Frame）
    let nodes = crate::convert::convert_to_nodes(
        &mut engine, &mut locator,
        children.iter().copied(),
        ConversionLevel::Block,
        Whitespace::Normal,
    )?;

    // 3. 包装为完整 HTML 文档结构
    let mut output = finalize_dom(&mut engine, nodes, &info, ...)?;

    // 4. 解析内联样式（延迟到最后因为 finalize_dom 可能插入新节点）
    css::resolve_inline_styles(output.root_mut());

    // 5. 若文档含公式，注入 MathML CSS
    if has_equations { ... }

    Ok(HtmlDocument::new(output, info))
}
```

关键差异：HTML 目标**跳过排版阶段**，realize 之后的元素直接转为语义化 DOM 节点，不需要计算绝对位置。

#### 7.2.2 HtmlNode 数据结构

文件：[dom.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/dom.rs)

[HtmlNode](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/dom.rs#L100-L110) 是 HTML DOM 的节点表示：

```rust
pub enum HtmlNode {
    Tag(Tag),                        // 内省标签（不出现在输出中）
    Text(EcoString, Span),          // 文本
    Element(HtmlElement),           // HTML 元素
    Frame(HtmlFrame),               // 嵌入的排版帧（转 SVG）
}
```

[HtmlElement](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/dom.rs#L183-L206) 表示一个 HTML 元素：

```rust
pub struct HtmlElement {
    pub tag: HtmlTag,                // 标签名（如 div、p、span）
    pub attrs: HtmlAttrs,           // 属性列表
    pub css: css::Properties,       // 内联 CSS 属性
    pub children: EcoVec<HtmlNode>, // 子节点
    pub parent: Option<Location>,   // 逻辑父元素位置
    pub span: Span,                 // 源码位置
    pub pre_span: bool,             // 是否 pre-wrap 空白保护 span
}
```

[HtmlFrame](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/dom.rs#L505-L521) 是 HTML 和 Paged 世界的桥梁：无法直接转为 HTML 元素的内容（如复杂排版、图像）会先排版为 Frame，然后以 SVG 形式嵌入 HTML：

```rust
pub struct HtmlFrame {
    pub inner: Frame,               // 排版帧
    pub text_size: Abs,             // 文本大小（用于 em 单位换算）
    pub id: Option<EcoString>,      // SVG 元素 ID
    pub css: css::Properties,       // CSS 属性
    pub anchors: EcoVec<(Point, EcoString)>, // 锚点位置
    pub span: Span,
}
```

#### 7.2.3 元素转换：convert_to_nodes()

文件：[typst-html/src/convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs)

[convert_to_nodes()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs#L60-L90) 将 realize 产物转为 HtmlNode 列表。[handle()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs#L93-L163) 对每个元素分派：

| 元素类型 | 处理 |
|----------|------|
| `TagElem` | 直接 push 内省标签 |
| `HtmlElem` | [handle_html_elem()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs#L166-L248) → 构建 HtmlElement，递归处理子元素 |
| `SpaceElem` | push 文本空格节点 |
| `TextElem` | 应用大小写转换后 push |
| `SmartQuoteElem` | 智能引号处理 |
| `BoxElem` | 内联排版帧 |
| `BlockElem` | 块级排版帧 |
| `FrameElem` | 调用 `Routines::layout_frame` 排版后包装为 HtmlFrame |
| 其他 | 发出警告并忽略 |

[ConversionLevel](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs#L23-L31) 控制转换上下文：
- `Block`：顶层块级上下文，独立的智能引号状态
- `Inline(&mut SmartQuoter)`：共享内联上下文，共享引号状态

[Whitespace](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/convert.rs#L34-L57) 控制空白处理：
- `Normal`：使用 `white-space: pre-wrap` span 保护空白
- `Pre`：原样输出（`<pre>` 内）

#### 7.2.4 DOM 终结化：finalize_dom()

[finalize_dom()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/document.rs#L259-L311) 将用户生成的节点包装为完整 HTML 文档：

```
用户节点列表
  ├─ 若唯一元素为 <html> → 直接使用（自定义 DOM）
  ├─ 若唯一元素为 <body> → 需要包 <html>
  └─ 否则 → 包装为 <body>，再包 <html>
      ├─ <head>: charset、viewport、title、description、author、keywords
      ├─ <body>: 用户内容 + 脚注容器
      └─ <html lang=...>: 语言属性来自 locale
```

#### 7.2.5 HTML 编码导出

文件：[typst-html/src/encode.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs)

[html()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L23-L27) 将 HtmlDocument 编码为 HTML 字符串：

```rust
pub fn html(document: &HtmlDocument, options: &HtmlOptions) -> SourceResult<String> {
    let link_resolver = LateLinkResolver::new(None, document.introspector().as_ref());
    let w = Writer::new(link_resolver.track(), options.pretty);
    html_impl(w, document.root())
}
```

[Writer](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L54-L64) 维护缩进级别和输出缓冲区，[write_node()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L89-L97) 按节点类型分派：

| 节点 | 编码方式 |
|------|----------|
| `Tag` | 忽略（内省标签不出现在输出中） |
| `Text` | [write_text()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L100-L109) → HTML 转义（`&amp;` `<` `>` `"` `'`） |
| `Element` | [write_element()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L112-L167) → 开标签 + 属性 + 子节点 + 闭标签 |
| `Frame` | [write_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L391-L401) → 调用 `typst_svg::svg_in_html()` 嵌入 SVG |

**HtmlFrame → SVG 的特殊路径**：HTML 文档中的 `FrameElem` 会先排版为 Frame，编码时通过 [write_frame()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L391-L401) 调用 [svg_in_html()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs#L82-L123)，生成带 `overflow: visible` 和 em 单位尺寸的 SVG 片段，实现排版内容在 HTML 中的自然嵌入。

---

### 7.3 Bundle 目标：多文档 + 资源 → VirtualFs

#### 7.3.1 Bundle 编译流程

文件：[typst-bundle/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs)

[bundle()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L120-L136) → [bundle_impl()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L139-L219)：

```rust
fn bundle_impl(...) -> SourceResult<Bundle> {
    // 1. Bundle 专用的 realize（RealizationKind::Bundle）
    let children = (engine.library.routines.realize)(
        RealizationKind::Bundle, ...
    )?;

    // 2. 收集所有文档和资源
    let children = collect(&children, &mut engine, &mut locator)?;

    // 3. 并行编译各个子文档
    let mut items = engine
        .parallelize(children, |engine, child| {
            Ok(match child {
                Child::Tag(tag) => Item::Tag(tag.clone()),
                Child::Asset(asset) => Item::Asset(path, data, location),
                Child::Document(document, styles, locator) =>
                    Item::Document(path, compile_document(engine, document, styles, locator)?, location),
            })
        })
        .collect_combined_result::<Vec<_>>()?;

    // 4. 构建 BundleIntrospector（聚合所有子文档的内省器）
    let mut introspector = BundleIntrospector::new(&items);
    let targets = introspector.link_targets();
    let anchors = crate::link::create_link_anchors(&mut items, &targets);
    introspector.set_anchors(anchors);

    // 5. 组装 Bundle
    let mut files = IndexMap::default();
    for item in items {
        match item {
            Item::Asset(path, bytes, _) => files.insert(path, BundleFile::Asset(bytes)),
            Item::Document(path, doc, _) => files.insert(path, BundleFile::Document(doc)),
            _ => {}
        }
    }
    Ok(Bundle { files: Arc::new(files), introspector: Arc::new(introspector) })
}
```

#### 7.3.2 Bundle 数据结构

[Bundle](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L44-L54) 是一个多文件容器：

```rust
pub struct Bundle {
    pub files: Arc<IndexMap<VirtualPath, BundleFile, FxBuildHasher>>,
    pub introspector: Arc<BundleIntrospector>,
}
```

[BundleFile](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L76-L82) 枚举了两种文件类型：

```rust
pub enum BundleFile {
    Document(BundleDocument),  // document 元素产生的文档
    Asset(Bytes),              // asset 元素产生的原始数据
}
```

[BundleDocument](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L87-L92) 支持两种格式：

```rust
pub enum BundleDocument {
    Paged(Box<PagedDocument>, PagedExtras),  // Paged + 额外信息
    Html(Box<HtmlDocument>),                 // HTML
}
```

[PagedExtras](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L104-L114) 携带导出所需的额外数据：

```rust
pub struct PagedExtras {
    pub format: PagedFormat,                    // 导出格式：Pdf/Png/Svg
    pub anchors: Vec<(Location, EcoString)>,    // 跨文档链接锚点
}
```

#### 7.3.3 子文档编译：compile_document()

[compile_document()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L288-L339) 根据 `DocumentFormat` 分派到不同的编译路径：

```
DocumentElem { body, path, format }
  ├─ PagedFormat::Pdf  → typst_layout::layout_document_for_bundle() → PagedDocument
  ├─ PagedFormat::Png  → typst_layout::layout_document_for_bundle() → PagedDocument (单页)
  ├─ PagedFormat::Svg  → typst_layout::layout_document_for_bundle() → PagedDocument (单页)
  └─ DocumentFormat::Html → typst_html::html_document_for_bundle() → HtmlDocument
```

Png/Svg 格式要求单页文档，否则报错。

#### 7.3.4 BundleIntrospector —— 跨文档内省

文件：[introspect.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/introspect.rs)

[BundleIntrospector](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/introspect.rs#L23-L34) 聚合所有子文档的内省器：

```rust
pub struct BundleIntrospector {
    children: Vec<(VirtualPath, ChildIntrospector, Location)>,
    elements: ElementIntrospector<Option<NonZeroUsize>>,  // off-by-one 索引
    anchors: FxHashMap<Location, EcoString>,
}
```

[ChildIntrospector](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/introspect.rs#L176-L179) 是子文档内省器的枚举：

```rust
enum ChildIntrospector {
    Paged(Arc<PagedIntrospector>),
    Html(Arc<HtmlIntrospector>),
}
```

BundleIntrospector 实现 `Introspector` trait，所有查询委托给对应子文档的内省器。`document()` 方法可定位元素所属文档，`path()` 方法返回文档路径，支持跨文档引用解析。

#### 7.3.5 Bundle 导出和写入

文件：[typst-bundle/src/export.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs)

[export()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L24-L40) 使用 rayon **并行**导出所有文件到 VirtualFs：

```rust
pub fn export(bundle: &Bundle, options: &BundleOptions) -> SourceResult<VirtualFs> {
    bundle.files.par_iter()
        .map(|(path, file)| {
            let data = match file {
                BundleFile::Document(doc) => {
                    let link_resolver = LateLinkResolver::new(
                        Some(path), bundle.introspector.as_ref()
                    );
                    export_document(doc, options, link_resolver.track())
                }
                BundleFile::Asset(bytes) => Ok(bytes.clone()),
            };
            data.map(|data| (path.clone(), data))
        })
        .collect_combined_result()
}
```

[export_document()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L56-L75) 根据子文档格式选择导出方式，每个都用 `#[comemo::memoize]` 缓存：

| 格式 | 函数 | 调用 |
|------|------|------|
| PDF | [export_pdf()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L78-L87) | `typst_pdf::pdf_in_bundle(doc, options, anchors, link_resolver)` |
| PNG | [export_png()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L90-L98) | `typst_render::render()` → `encode_png()` |
| SVG | [export_svg()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L101-L125) | `typst_svg::svg_in_bundle(page, options, anchors, link_resolver)` |
| HTML | [export_html()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L134-L141) | `typst_html::html_in_bundle(root, options, link_resolver)` |

**关键差异**：Bundle 内的导出函数使用 `*_in_bundle()` 变体，它们额外接受 `anchors`（命名锚点）和 `LateLinkResolver`（跨文档链接解析器），支持文档间的交叉链接。

**VirtualFs 写入磁盘**：CLI 中 [write_virtual_fs()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-cli/src/compile.rs#L418-L438) 将 VirtualFs 并行写入目标目录：

```rust
fn write_virtual_fs(root: &Path, fs: &VirtualFs) -> StrResult<Vec<Output>> {
    std::fs::create_dir_all(root)?;
    fs.par_iter()
        .map(|(path, data)| {
            let realized = path.realize(root)?;
            std::fs::create_dir_all(parent)?;
            std::fs::write(&realized, data)?;
            Ok(Output::Path(realized))
        })
        .collect()
}
```

每个 `VirtualPath` 通过 `realize(root)` 解析为绝对文件路径，自动创建中间目录，然后写入字节内容。

---

### 7.4 三类目标的协作差异总结

#### 编译阶段差异

| 阶段 | Paged | HTML | Bundle |
|------|-------|------|--------|
| realize | `RealizationKind::Document` | `RealizationKind::Document` | `RealizationKind::Bundle` |
| realize 后 | `layout_pages()` → Frame 树 | `convert_to_nodes()` → HtmlNode 树 | `collect()` → 子文档并行编译 |
| 子内容处理 | 全部排版为 Frame | HtmlElem → DOM；其他 → HtmlFrame(SVG) | 各子文档独立编译（Paged/HTML） |
| 内省器 | `PagedIntrospector`（单文档） | `HtmlIntrospector`（单文档） | `BundleIntrospector`（聚合多文档） |

#### 导出阶段差异

| 差异点 | Paged | HTML | Bundle |
|--------|-------|------|--------|
| Frame 处理 | 直接转换为 PDF/PNG/SVG 图元 | HtmlFrame → svg_in_html() 嵌入 | 各子文档独立导出 |
| 链接解析 | 页内链接 + URL | 页内链接 + URL + 跨帧链接 | **跨文档链接** via LateLinkResolver |
| 锚点 | PDF 命名目标 | HTML fragment ID | 锚点传播到各子文档导出函数 |
| 并行化 | 页面级并行（排版时） | 无 | 文件级并行（导出时） |
| 输出形式 | 单文件字节流 | 单文件字符串 | 多文件 VirtualFs → 磁盘目录 |

#### LateLinkResolver —— 跨文档链接机制

Bundle 独有的 [LateLinkResolver](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/model/mod.rs) 解决文档间的交叉引用：

1. **编译时**：各子文档独立编译，引用指向目标 `Location`
2. **导出时**：`LateLinkResolver::new(Some(path), &introspector)` 创建，根据 `BundleIntrospector.path()` 和 `BundleIntrospector.anchor()` 将 Location 解析为相对 URI
3. **锚点注入**：[create_link_anchors()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/link.rs) 为被引用的位置分配 HTML fragment ID / PDF 命名目标
4. **传递给导出函数**：`pdf_in_bundle()` / `svg_in_bundle()` / `html_in_bundle()` 接受 anchors 和 link_resolver 参数

---

## 8. 核心协作机制

### 8.1 Routines 动态分发

文件：[routines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/routines.rs)

为了支持 crate 拆分，Typst 使用 `Routines` 结构体存储函数指针表，在 [typst/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst/src/lib.rs#L311-L325) 初始化：

```rust
static ROUTINES: LazyLock<Routines> = LazyLock::new(|| Routines {
    rules: || { ... },
    eval_string: typst_eval::eval_string,
    eval_closure: typst_eval::eval_closure,
    realize: typst_realize::realize,
    layout_frame: typst_layout::layout_frame,
    html_module: typst_html::module,
    html_mathml_body: typst_html::html_mathml_body,
    html_span_filled: typst_html::html_span_filled,
});
```

### 8.2 Engine —— 编译上下文

文件：[engine.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/engine.rs)

[Engine](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/engine.rs#L18-L36) 贯穿所有阶段：

```rust
pub struct Engine<'a> {
    pub world: Tracked<'a, dyn World + 'a>,        // 文件系统、字体等
    pub library: &'a LazyHash<Library>,            // 标准库
    pub introspector: Protected<Tracked<'a, dyn Introspector + 'a>>, // 查询接口
    pub traced: Tracked<'a, Traced>,               // 追踪 span
    pub sink: TrackedMut<'a, Sink>,                // 警告、延迟错误、追踪值
    pub route: Route<'a>,                           // 调用栈（循环检测）
}
```

### 8.3 Introspector —— 内省机制

`Introspector` 是连接各次迭代的桥梁：
- 第 1 次：`EmptyIntrospector`（所有查询返回默认值）
- 第 2-N 次：上一次 `PagedDocument` 生成的 `PagedIntrospector`（可查询元素位置、页码等）

当 `constraint.validate(introspector)` 返回 true 时，说明文档不再变化，达到稳定状态。

### 8.4 Sink —— 延迟错误与警告

[Sink](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-library/src/engine.rs#L152-L167) 收集：
- `introspections`：内省操作记录（用于分析不收敛原因）
- `delayed`：延迟错误（非最后一次迭代不致命）
- `warnings`：警告（自动去重）
- `values`：追踪值（IDE tooltip）

---

## 9. 完整调用链（按代码顺序）

以 PDF 输出为例：

```
typst-cli::compile::compile()
└─ compile_once()
   └─ compile_and_export()
      └─ typst::compile::<PagedDocument>()
         └─ compile_impl::<PagedDocument>()
            ├─ world.main() → FileId
            ├─ world.source() → Source
            ├─ typst_eval::eval()
            │  ├─ source.root() → SyntaxNode (已解析)
            │  ├─ Vm::new()
            │  └─ Markup::eval() → Content
            │     └─ ... (递归求值所有子节点)
            │
            └─ 内省循环 (最多5次):
               ├─ PagedDocument::create()
               │  └─ typst_layout::layout_document()
               │     └─ layout_document_common()
               │        ├─ Routines::realize()
               │        │  └─ typst_realize::realize()
               │        │     ├─ visit() → 递归应用 show 规则
               │        │     └─ finish() → 完成分组
               │        └─ layout_pages()
               │           ├─ collect() → 分片
               │           ├─ parallelize(layout_page_run) → 并行排版
               │           └─ finalize() → 生成 Page
               │
               ├─ constraint.validate() → 检查稳定?
               ├─ 稳定 → break
               └─ 不稳定 → 记录到 history，继续迭代
```

---

## 10. 关键设计要点

1. **内省循环**：解决交叉引用的根本机制，通过多次迭代直到稳定
2. **comemo 缓存**：`#[memoize]` 和 `#[track]` 宏实现增量编译
3. **延迟错误**：show rule 错误在早期迭代中被延迟，避免因内省不稳定导致误报
4. **Output trait**：统一的多目标抽象，支持 Paged/HTML/Bundle 三种输出
5. **Routines 表**：函数指针表实现 crate 间解耦，避免循环依赖
6. **Engine 传递**：所有编译阶段共享同一 Engine，确保上下文一致
