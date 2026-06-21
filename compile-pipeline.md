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

### 7.1 PagedDocument 导出

#### PDF 导出

文件：[typst-pdf/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/lib.rs)

[pdf()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-pdf/src/lib.rs#L36-L38) → `convert::convert()`

#### PNG 导出

文件：[typst-render/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/lib.rs)

[render()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-render/src/image.rs) → Pixmap → encode_png()

#### SVG 导出

文件：[typst-svg/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/lib.rs)

[svg()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-svg/src/write.rs) → String

### 7.2 HtmlDocument 导出

文件：[typst-html/src/document.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/document.rs)

[html_document()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/document.rs#L25-L40) → 内部同样调用 realize()，但使用不同的内置 show 规则（生成 HTML DOM 而非 Frame）

导出：[html()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-html/src/encode.rs#L23-L27) → String

### 7.3 Bundle 导出

文件：[typst-bundle/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs)

[bundle()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/lib.rs#L121-L136) → 可生成多文档 + 资源

导出：[export()](file:///d:/fz/0601-2/solo-dogfeeding/code/120-typst/crates/typst-bundle/src/export.rs#L24-L40) → VirtualFs（内存文件系统）

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
