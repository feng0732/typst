# Typst PDF 输出后端分析

## 概述

Typst 的 PDF 输出功能由 `typst-pdf` crate 实现，位于 `crates/typst-pdf/` 目录。该模块负责将排版后的 `PagedDocument` 转换为符合 PDF 规范的字节流。核心依赖是 **krilla** 库，一个专门用于 PDF 生成的 Rust 库。

## 核心架构

### 主要依赖

```
typst-pdf
├── krilla          # PDF 生成核心库
├── krilla-svg      # SVG 渲染支持
├── typst-layout    # 排版后文档结构
├── typst-library   # 类型定义和工具函数
└── ...
```

### 入口点

导出入口在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/lib.rs#L36-L38)：

```rust
pub fn pdf(document: &PagedDocument, options: &PdfOptions) -> SourceResult<Vec<u8>> {
    convert::convert(document, options, &[], None)
}
```

## PDF 对象组织机制

### 全局上下文 (GlobalContext)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L278-L301) 中定义，贯穿整个转换过程：

```rust
pub(crate) struct GlobalContext<'a> {
    // 字体缓存（双向映射）
    pub(crate) fonts_forward: FxHashMap<FontInstance, krilla::text::Font>,
    pub(crate) fonts_backward: FxHashMap<krilla::text::Font, FontInstance>,
    // 图像跨度映射
    pub(crate) image_to_spans: FxHashMap<krilla::image::Image, Span>,
    pub(crate) image_spans: FxHashSet<Span>,
    // 文档引用
    pub(crate) document: &'a PagedDocument,
    pub(crate) options: &'a PdfOptions,
    // 链接解析器（用于 bundle 导出）
    pub(crate) link_resolver: Option<Tracked<'a, LateLinkResolver<'a>>>,
    // 命名目标映射
    pub(crate) loc_to_names: FxHashMap<Location, NamedDestination>,
    // 页面索引转换器
    pub(crate) page_index_converter: PageIndexConverter,
    // 标签上下文（无障碍支持）
    pub(crate) tags: Tags,
}
```

**设计要点：**
- 双向字体映射避免重复转换，提高性能
- 图像跨度映射用于错误报告
- `PageIndexConverter` 处理部分页面导出的索引映射
- `Tags` 结构管理 PDF 标签树（用于可访问性）

### 帧上下文 (FrameContext)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L220-L275) 中定义，用于单个 Frame 的转换：

```rust
pub(crate) struct FrameContext {
    pub(crate) page_idx: Option<usize>,
    states: Vec<State>,  // 变换状态栈
    link_annotations: IndexMap<GroupId, SmallVec<[LinkAnnotation; 1]>, FxBuildHasher>,
}
```

### 变换状态 (State)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L178-L217) 中维护坐标变换栈：

```rust
pub(crate) struct State {
    transform: Transform,              // 当前变换
    container_transform: Transform,    // 第一个硬帧的变换
    container_size: Size,              // 第一个硬帧的尺寸
}
```

**状态栈操作：**
- `push()` / `pop()` 管理状态栈
- `pre_concat()` 叠加变换
- `register_container()` 标记硬帧边界（用于渐变/图案定位）

### Krilla 对象模型

Typst 不直接操作 PDF 语法，而是通过 krilla 提供的高级抽象：

| Typst 类型 | Krilla 类型 | 用途 |
|-----------|------------|------|
| `PagedDocument` | `Document` | PDF 文档根对象 |
| `Page` | `Page` + `Surface` | 页面和内容流 |
| `TextItem` | `Font` + 字形 | 文本渲染 |
| `Shape` | `Path` + `Fill`/`Stroke` | 矢量图形 |
| `Image` | `Image` | 光栅图像 |
| `Gradient` | `LinearGradient`/`RadialGradient`/`SweepGradient` | 渐变填充 |
| `Tiling` | `Pattern` | 图案填充 |

## 字体嵌入流程

### 字体转换管道

在 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L62-L104) 中实现：

```rust
fn convert_font(gc: &mut GlobalContext, typst_font: FontInstance) -> SourceResult<krilla::text::Font> {
    if let Some(font) = gc.fonts_forward.get(&typst_font) {
        return Ok(font.clone());  // 缓存命中
    }
    let font = build_font(typst_font.clone())?;
    // 更新双向缓存
    gc.fonts_forward.insert(typst_font.clone(), font.clone());
    gc.fonts_backward.insert(font.clone(), typst_font.clone());
    Ok(font)
}

#[comemo::memoize]
fn build_font(typst_font: FontInstance) -> SourceResult<krilla::text::Font> {
    let font_data: Arc<dyn AsRef<[u8]> + Send + Sync> = Arc::new(typst_font.data().clone());
    let variations = typst_font.variations().0.iter()
        .map(|(tag, value)| (krilla::text::Tag::new(&tag.to_bytes()), value.0))
        .collect::<Vec<_>>();
    
    krilla::text::Font::new_variable(font_data.into(), typst_font.index(), &variations)
}
```

**关键技术点：**

1. **两级缓存**：
   - `GlobalContext` 中的运行时缓存（单次导出内共享）
   - `#[comemo::memoize]` 过程宏实现的跨导出缓存

2. **可变字体支持**：
   - 收集字体变化轴（weight, width, italic 等）
   - 通过 `Font::new_variable` 传递给 krilla
   - krilla 负责生成正确的字体描述符

3. **字形适配层**：

在 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L106-L148) 中通过 `PdfGlyph` 透明包装 Typst 的 `Glyph`：

```rust
#[derive(Debug, TransparentWrapper)]
#[repr(transparent)]
struct PdfGlyph(Glyph);

impl krilla::text::Glyph for PdfGlyph {
    fn glyph_id(&self) -> GlyphId { GlyphId::new(self.0.id as u32) }
    fn text_range(&self) -> Range<usize> { ... }
    fn x_advance(&self, size: f32) -> f32 { self.0.x_advance.get() as f32 * size }
    fn x_offset(&self, size: f32) -> f32 { self.0.x_offset.get() as f32 * size }
    fn location(&self) -> Option<Location> { Some(self.0.span.0.into_raw()) }
}
```

**字体嵌入由 krilla 自动处理：**
- 子集化：只嵌入使用到的字形
- 字体描述符生成（FontDescriptor）
- 编码映射（ToUnicode CMap 用于文本提取和搜索）
- 许可证检查（PDF/A 等标准可能限制某些字体）

## 完整转换流程

### 阶段一：初始化与准备

**入口函数** [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L48-L95):

```rust
pub fn convert(
    typst_document: &PagedDocument,
    options: &PdfOptions,
    anchors: &[(Location, EcoString)],
    link_resolver: Option<Tracked<LateLinkResolver>>,
) -> SourceResult<Vec<u8>> {
    // 1. 创建 krilla 序列化设置
    let settings = SerializeSettings { ... };
    
    // 2. 创建 krilla Document
    let mut document = Document::new_with(settings);
    
    // 3. 页面索引转换（处理部分导出）
    let page_index_converter = PageIndexConverter::new(typst_document, options);
    
    // 4. 收集命名目标（用于跨引用）
    let named_destinations = collect_named_destinations(...);
    
    // 5. 初始化标签树（无障碍）
    let tags = tags::init(typst_document, options)?;
    
    // 6. 创建全局上下文
    let mut gc = GlobalContext::new(...);
    
    // 7. 转换页面内容
    convert_pages(&mut gc, &mut document)?;
    
    // 8. 附加文件
    attach_files(&gc, &mut document)?;
    
    // 9. 解析标签树
    let (doc_lang, tree) = tags::resolve(&mut gc)?;
    
    // 10. 设置大纲、元数据、标签树
    document.set_outline(build_outline(&gc));
    document.set_metadata(build_metadata(&gc, doc_lang));
    document.set_tag_tree(tree);
    
    // 11. 最终序列化
    finish(document, gc, options.standards.config)
}
```

### 阶段二：标签树预构建（无障碍支持）

在 [tags/tree/build.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/tags/tree/build.rs#L180-L239) 中预构建逻辑结构树：

```rust
pub fn build(document: &PagedDocument, options: &PdfOptions) -> SourceResult<Tree> {
    let mut tree = TreeBuilder::new(document, options);
    for page in document.pages() {
        visit_frame(&mut tree, &page.frame)?;
    }
    // 验证标签闭合完整性
    // 解析逻辑父子关系
    Ok(tree.finish())
}
```

**支持的语义标签类型：**
- 文档结构：`Document`, `Part`, `Sect`, `Div`
- 标题：`H1`-`H6`
- 段落：`P`
- 列表：`L`, `LI`, `Lbl`, `LBody`
- 表格：`Table`, `TR`, `TD`, `TH`
- 链接：`Link`
- 注释：`Note`
- 代码：`Code`, `CodeBlock`
- 公式：`Formula`
- 图形：`Figure`, `Caption`
- 工件：`Artifact`（无语义内容）

### 阶段三：页面内容转换

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L97-L173) 中逐页处理：

```rust
fn convert_pages(gc: &mut GlobalContext, document: &mut Document) -> SourceResult<()> {
    for (i, typst_page) in gc.document.pages().iter().enumerate() {
        // 跳过未选中的页面
        if gc.page_index_converter.pdf_page_index(i).is_none() { continue; }
        
        // 创建设置（尺寸、出血、页码标签）
        let settings = PageSettings::from_wh(...);
        
        // 开始页面
        let mut page = document.start_page_with(settings);
        let mut surface = page.surface();
        
        // 处理页面内容
        tags::page(gc, &mut surface, |gc, surface| {
            handle_frame(&mut fc, &typst_page.frame, ..., surface, gc)
        })?;
        
        surface.finish();
        
        // 添加链接注解
        let link_annotations = fc.link_annotations.into_values().flatten();
        tags::add_link_annotations(gc, &mut page, link_annotations);
    }
    Ok(())
}
```

### 阶段四：Frame 递归遍历

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L328-L391) 中处理单个 Frame：

```rust
pub(crate) fn handle_frame(
    fc: &mut FrameContext,
    frame: &Frame,
    padding: Sides<Abs>,
    fill: Option<Paint>,
    surface: &mut Surface,
    gc: &mut GlobalContext,
) -> SourceResult<()> {
    fc.push();  // 保存状态
    
    // 硬帧注册（渐变边界）
    if frame.kind().is_hard() {
        fc.state_mut().register_container(frame.size());
    }
    
    // 绘制背景填充
    if let Some(fill) = fill {
        let shape = Geometry::Rect(frame.size() + padding.sum_by_axis()).filled(fill);
        handle_shape(fc, &shape, surface, gc, ..., ArtifactType::Background)?;
    }
    
    // 应用内边距偏移
    fc.push();
    fc.state_mut().pre_concat(Transform::translate(padding.left, padding.top));
    
    // 遍历所有项
    for (point, item) in frame.items() {
        fc.push();
        fc.state_mut().pre_concat(Transform::translate(point.x, point.y));
        
        match item {
            FrameItem::Group(g) => handle_group(fc, g, surface, gc)?,
            FrameItem::Text(t) => handle_text(fc, t, surface, gc)?,
            FrameItem::Shape(s, span) => handle_shape(fc, s, surface, gc, *span, ArtifactType::Layout)?,
            FrameItem::Image(image, size, span) => handle_image(gc, fc, image, *size, surface, *span)?,
            FrameItem::Link(dest, size) => handle_link(fc, gc, dest, *size)?,
            FrameItem::Tag(Tag::Start(_, flags)) => tags::handle_start(gc, fc, surface),
            FrameItem::Tag(Tag::End(_, _, flags)) => tags::handle_end(gc, fc, surface),
        }
        
        fc.pop();
    }
    
    fc.pop();
    fc.pop();
    Ok(())
}
```

### 阶段五：各类内容项处理

#### 1. 文本处理 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L18-L60)

```rust
pub(crate) fn handle_text(fc: &mut FrameContext, t: &TextItem, surface: &mut Surface, gc: &mut GlobalContext) -> SourceResult<()> {
    let mut handle = tags::text(gc, fc, surface, t);
    let surface = handle.surface();
    
    let font = convert_font(gc, t.font.clone())?;
    let fill = paint::convert_fill(gc, &t.fill, ...)?;
    let stroke = if let Some(stroke) = t.stroke.as_ref() {
        Some(paint::convert_stroke(gc, stroke, ...)?)
    } else { None };
    
    surface.push_transform(&fc.state().transform().to_krilla());
    let mut surface = defer(surface, |s| s.pop());
    surface.set_fill(Some(fill));
    surface.set_stroke(stroke);
    surface.draw_glyphs(Point::from_xy(0.0, 0.0), glyphs, font, text, size.to_f32(), false);
    
    Ok(())
}
```

#### 2. 图形处理 [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/shape.rs#L14-L75)

```rust
pub(crate) fn handle_shape(...) -> SourceResult<()> {
    let mut handle = tags::shape(gc, fc, surface, shape, artifact_type);
    let surface = handle.surface();
    
    surface.set_location(span.into_raw());
    surface.push_transform(&fc.state().transform().to_krilla());
    
    if let Some(path) = convert_geometry(&shape.geometry) {
        let fill = shape.fill.as_ref().map(|paint| paint::convert_fill(...)?);
        let stroke = shape.stroke.as_ref().map(|stroke| paint::convert_stroke(...)?);
        
        if fill.is_some() || stroke.is_some() {
            surface.set_fill(fill);
            surface.set_stroke(stroke);
            surface.draw_path(&path);
        }
    }
    Ok(())
}
```

**图形类型转换：**
- `Geometry::Line` → 移动+直线
- `Geometry::Rect` → 矩形路径（处理负尺寸）
- `Geometry::Curve` → 贝塞尔曲线（通过 `convert_path` 转换）

#### 3. 图像嵌入 [image.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/image.rs#L24-L81)

支持三种图像类型：

| 类型 | 处理方式 |
|------|---------|
| 光栅图像 (JPEG/PNG等) | JPEG 直接嵌入原始数据，其他格式转换为 RGB/Luma8 |
| SVG | 通过 krilla-svg 渲染为 PDF 矢量 |
| PDF 页面 | 直接嵌入 PDF 内容流 |

**光栅图像优化：**
- JPEG 直接复用原始字节（避免重新编码）
- 其他格式转换为 8-bit RGB 或灰度
- 保留 ICC 配置文件（支持的格式）
- EXIF 方向变换作为坐标变换应用

#### 4. 填充与描边 [paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/paint.rs)

**填充类型：**
- **纯色**：转换为相应的颜色空间（RGB/CMYK/Luma/Separation）
- **线性渐变**：`LinearGradient`，处理角度和纵横比校正
- **径向渐变**：`RadialGradient`，支持焦点偏移
- **锥形渐变**：`SweepGradient`，使用 PostScript 函数（注意 PDF/A 兼容性）
- **图案填充**：`Pattern`，递归调用 `handle_frame` 渲染图案内容

#### 5. 组变换 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L393-L430)

```rust
pub(crate) fn handle_group(fc: &mut FrameContext, group: &GroupItem, ...) -> SourceResult<()> {
    fc.push();
    fc.state_mut().pre_concat(group.transform);
    
    tags::group(gc, fc, surface, group.parent, |gc, fc, surface| {
        // 处理裁剪路径
        let clip_path = group.clip.as_ref().and_then(|p| {
            let mut builder = PathBuilder::new();
            convert_path(p, &mut builder);
            builder.finish()
        }).and_then(|p| p.transform(fc.state().transform.to_krilla()));
        
        if let Some(clip_path) = &clip_path {
            surface.push_clip_path(clip_path, &FillRule::NonZero);
        }
        
        let res = handle_frame(fc, &group.frame, Sides::splat(Abs::zero()), None, surface, gc);
        
        if clip_path.is_some() { surface.pop(); }
        res
    })?;
    
    fc.pop();
    Ok(())
}
```

### 阶段六：辅助内容生成

#### 元数据 [metadata.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/metadata.rs)

```rust
pub(crate) fn build_metadata(gc: &GlobalContext, doc_lang: Option<Locale>) -> Metadata {
    Metadata::new()
        .keywords(...)
        .authors(...)
        .language(...)
        .creator(...)      // "Typst x.y.z"
        .title(...)
        .description(...)
        .document_id(...)  // 基于 ident 选项
        .creation_date(...)
        .text_direction(...)
}
```

#### 文档大纲 [outline.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/outline.rs)

- 查询所有 `HeadingElem`
- 构建层次树结构
- 转换为 krilla `Outline` 对象
- 每个条目包含标题和 XYZ 目标位置

#### 链接注解 [link.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/link.rs)

链接目标类型：
- **URL**：`LinkAction`
- **位置**：`XyzDestination`（页面内跳转）
- **命名目标**：`NamedDestination`（跨文档引用）
- **跨文档链接**：解析为相对 URI

#### 文件附件 [attach.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/attach.rs)

- 查询所有 `AttachElem`
- 检测文件类型决定是否压缩
- 支持关联类型：Source/Data/Alternative/Supplement

### 阶段七：最终序列化

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L434-L533) 的 `finish` 函数中：

```rust
fn finish(document: Document, gc: GlobalContext, configuration: Configuration) -> SourceResult<Vec<u8>> {
    match document.finish() {
        Ok(r) => Ok(r),
        Err(e) => match e {
            KrillaError::Font(f, err) => { /* 字体错误处理 */ }
            KrillaError::Validation(ve) => { /* 标准合规性错误 */ }
            KrillaError::Image(_, loc, err) => { /* 图像错误 */ }
            KrillaError::Pdf(_, e, loc) => { /* PDF 语法错误 */ }
            // ... 其他错误类型
        }
    }
}
```

**krilla `Document::finish()` 负责：**
1. 生成 PDF 头部（`%PDF-1.x`）
2. 序列化所有间接对象（页面、字体、图像等）
3. 构建交叉引用表（xref）
4. 生成文档目录（Catalog）
5. 写入尾部（trailer + startxref）
6. 应用压缩（如果启用）
7. 验证标准合规性（PDF/A, PDF/UA 等）

## PDF 标准合规性

通过 `PdfStandards` 和 `PdfStandard` 配置：

**支持的标准：**
- PDF 版本：1.4, 1.5, 1.6, 1.7, 2.0
- PDF/A（归档）：A-1b/a, A-2b/u/a, A-3b/u/a, A-4, A-4f, A-4e
- PDF/UA（无障碍）：UA-1

**验证点示例：**
- 字体必须嵌入（PDF/A）
- 禁止透明度（PDF/A-1）
- 必须包含文档语言（PDF/UA）
- 图像插值控制（PDF/A）
- 字体许可证检查

## 性能优化策略

1. **字体缓存**：双向哈希映射 + comemo 缓存
2. **图像缓存**：相同图像复用 krilla Image 对象
3. **延迟初始化**：`OnceLock` 用于图像通道提取
4. **memoization**：`#[comemo::memoize]` 宏缓存纯函数结果
5. **哈希映射**：使用 `rustc-hash` (FxHashMap/FxHashSet) 提高性能

## 错误处理

错误转换层在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L536-L838) 的 `convert_error` 函数中实现，将 krilla 错误映射为 Typst 友好的诊断信息，包含：
- 错误位置（span）
- 具体错误信息
- 修复建议（hints）

## 总结

```
PagedDocument
    ↓
┌─────────────────────────────────────────┐
│  初始化: Document, GlobalContext, Tags  │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  预构建: 标签树 (语义结构)              │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  页面遍历: for page in pages            │
│  ┌───────────────────────────────────┐  │
│  │  Surface (内容流)                │  │
│  │  ┌─────────────────────────────┐ │  │
│  │  │ handle_frame 递归           │ │  │
│  │  │  - Group → 变换+裁剪        │ │  │
│  │  │  - Text  → 字形绘制         │ │  │
│  │  │  - Shape → 路径+填充        │ │  │
│  │  │  - Image → 图像嵌入         │ │  │
│  │  │  - Link  → 注解             │ │  │
│  │  └─────────────────────────────┘ │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  辅助内容: 元数据, 大纲, 附件          │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  序列化: krilla → PDF 字节流           │
└─────────────────────────────────────────┘
    ↓
Vec<u8> (PDF 文件内容)
```
