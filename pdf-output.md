# Typst PDF 输出后端分析

## 概述

Typst 的 PDF 输出功能由 `typst-pdf` crate 实现，位于 `crates/typst-pdf/` 目录。该模块负责将排版后的 `PagedDocument` 转换为符合 PDF 规范的字节流。核心依赖是 **krilla** 库（v0.8.2），一个专门用于 PDF 生成的高级 Rust 库，构建在 `pdf-writer` 底层库之上。

## 核心架构

### 主要依赖

```
typst-pdf
├── krilla          # PDF 生成核心库（高级抽象）
├── krilla-svg      # SVG 渲染支持
├── pdf-writer      # 底层 PDF 语法生成（krilla 的依赖）
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

---

## 一、PDF 对象组织机制

### 1.1 全局上下文 (GlobalContext)

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
- 图像跨度映射用于错误报告（通过 `fonts_backward` 反向查找字体）
- `PageIndexConverter` 处理部分页面导出的索引映射
- `Tags` 结构管理 PDF 标签树（用于可访问性）

### 1.2 帧上下文 (FrameContext)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L220-L275) 中定义，用于单个 Frame 的转换：

```rust
pub(crate) struct FrameContext {
    pub(crate) page_idx: Option<usize>,
    states: Vec<State>,  // 变换状态栈
    link_annotations: IndexMap<GroupId, SmallVec<[LinkAnnotation; 1]>, FxBuildHasher>,
}
```

### 1.3 变换状态 (State)

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

### 1.4 Krilla 对象模型

Typst 不直接操作 PDF 语法，而是通过 krilla 提供的高级抽象。krilla 的对象可以分为两类：

| 类别 | Typst 类型 | Krilla 类型 | 是否为间接对象 |
|-----|-----------|------------|--------------|
| 文档级 | `PagedDocument` | `Document` | —（容器） |
| 页面 | 每个 `Page` | `Page` + `Surface` | 是 |
| 字体 | `FontInstance` | `Font` | 是 |
| 图像 | `Image` | `Image` | 是 |
| 颜色空间 | `ColorSpace` | `SeparationSpace` 等 | 是 |
| 填充/描边 | `Paint` | `Fill` / `Stroke` | 内联或引用 |
| 渐变 | `Gradient` | `LinearGradient` 等 | 是 |
| 图案 | `Tiling` | `Pattern` | 是 |
| 注解 | 链接等 | `Annotation` | 是 |
| 目的地 | 锚点 | `XyzDestination` 等 | 是 |
| 标签树 | 语义结构 | `TagTree` | 是 |

---

## 二、页面资源 (Page Resources) 组织机制

### 2.1 什么是页面资源

在 PDF 规范中，每个页面都有一个 `Resources` 字典，包含该页面内容流中引用的所有外部资源。krilla 的 `Surface` 自动管理这些资源。

### 2.2 Surface 的资源收集机制

`Surface` 是 krilla 的核心绘制抽象，在 [surface](https://docs.rs/krilla/0.8.2/krilla/surface/index.html) 模块中定义。它既是内容流的构建器，也是资源的收集器。

**资源类型：**

| 资源类型 | 字典键 | Krilla API | 触发时机 |
|---------|-------|-----------|---------|
| 字体 | `/Font` | `draw_glyphs()` / `draw_text()` | 首次使用字体时 |
| 颜色空间 | `/ColorSpace` | 设置填充/描边颜色时 | 使用特殊颜色空间时 |
| 图案 | `/Pattern` | `draw_path()` 使用图案填充时 | 图案首次被引用时 |
| 渐变 | `/Shading` | `draw_path()` 使用渐变填充时 | 渐变首次被引用时 |
| 外部对象 | `/XObject` | `draw_image()` / `draw_svg()` | 图像/表单首次绘制时 |
| 图形状态参数 | `/ExtGState` | 设置透明度/混合模式时 | 状态首次设置时 |

### 2.3 资源收集流程

从 Typst 代码使用模式可推断 krilla 的资源收集流程：

```
surface.draw_glyphs(..., font, ...)
    ↓
1. 检查字体是否已在当前页面资源中注册
   ├─ 是 → 使用已有的资源名称（如 /F1）
   └─ 否 → 分配新的资源名称，并将字体对象添加到文档的间接对象池中
    ↓
2. 在内容流中输出 Tf 操作符（设置字体）
    ↓
3. 输出文本显示操作符（Tj / TJ）
```

**证据：** 从 `convert_font()` 函数 [text.rs#L62-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L62-L76) 中可以看到，Typst 层只做了 `FontInstance` → `krilla::text::Font` 的转换缓存，而资源名称分配和页面资源字典的构建完全由 krilla 内部处理。

### 2.4 流构建器 (Stream Builder) 模式

在图案填充的实现中可以看到 `stream_builder()` 模式 [paint.rs#L175-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/paint.rs#L175-L189)：

```rust
let mut stream_builder = surface.stream_builder();
let mut surface = stream_builder.surface();
// ... 在 surface 上绘制图案内容 ...
surface.finish();
let stream = stream_builder.finish();
let pattern = Pattern { stream, ... };
```

这种模式表明：
- `Surface` 可以嵌套使用
- 每个 `Surface` 都有自己的内容流和资源集合
- `stream_builder()` 创建一个独立的内容流（用于 Pattern 表单 XObject）
- 嵌套的 Surface 有独立的资源字典

---

## 三、间接对象管理与引用机制

### 3.1 间接对象概念

PDF 中的间接对象（Indirect Object）是可以被其他对象通过编号引用的对象。每个间接对象有唯一的对象编号（object number）和生成号（generation number）。

### 3.2 Krilla 的间接对象池

krilla 的 `Document` 内部维护一个间接对象池。从错误类型 [convert.rs#L571-L575](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L571-L575) 可以推断：

```rust
ValidationError::TooManyIndirectObjects => error!(
    "the PDF has too many indirect objects";
    hint: "reduce the size of your document";
),
```

**间接对象的来源（按类型）：**

| 对象类型 | 数量估算 | 何时创建 |
|---------|---------|---------|
| 页面对象 | O(n) 页 | `document.start_page_with()` |
| 页面对内容流 | O(n) 页 | `surface.finish()` |
| 字体对象 | O(m) 字体 | 字体首次被使用时 |
| 字体描述符 | O(m) 字体 | 字体嵌入时 |
| 字体文件流 | O(m) 字体 | 字体子集化后 |
| ToUnicode CMap | O(m) 字体 | 字体嵌入时 |
| 图像对象 | O(k) 图像 | `Image` 首次绘制时 |
| 渐变/图案 | O(p) 填充 | 首次使用时 |
| 注解对象 | O(q) 链接 | 页面添加注解时 |
| 大纲条目 | O(r) 标题 | `document.set_outline()` |
| 标签树节点 | O(s) 标签 | `document.set_tag_tree()` |

### 3.3 对象引用机制

krilla 使用强类型的 `Id<T>` 或直接的对象引用（如 `Font`、`Image`）来管理间接对象引用。从代码中可以观察到：

1. **值语义引用**：`krilla::text::Font`、`krilla::image::Image` 等类型实现了 `Clone`，可以像值一样传递
2. **内部共享**：这些类型内部可能使用 `Arc` 或类似机制共享底层数据
3. **去重机制**：相同的字体/图像对象在序列化时会被合并为同一个间接对象

**Typst 层的缓存设计印证了这一点：**
- `fonts_forward` / `fonts_backward` 双向映射确保同一字体只创建一个 krilla `Font` 对象
- `image_to_spans` 表明相同图像可以复用同一个 krilla `Image` 对象

### 3.4 序列化阶段的对象编号分配

在 `document.finish()` 阶段，krilla 执行以下操作：

1. **对象收集**：遍历所有页面、资源、辅助对象，收集所有需要序列化的间接对象
2. **编号分配**：按顺序分配对象编号（通常从 1 开始）
3. **交叉引用表构建**：记录每个对象在文件中的字节偏移
4. **引用解析**：将所有对象引用替换为实际的对象编号

---

## 四、字体嵌入流程

### 4.1 字体转换管道

在 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L62-L104) 中实现：

```rust
fn convert_font(
    gc: &mut GlobalContext,
    typst_font: FontInstance,
) -> SourceResult<krilla::text::Font> {
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
    let font_data: Arc<dyn AsRef<[u8]> + Send + Sync> =
        Arc::new(typst_font.data().clone());
    let variations = typst_font.variations().0.iter()
        .map(|(tag, value)| (krilla::text::Tag::new(&tag.to_bytes()), value.0))
        .collect::<Vec<_>>();
    
    krilla::text::Font::new_variable(font_data.into(), typst_font.index(), &variations)
}
```

### 4.2 两级缓存策略

**第一级：单次导出内缓存（GlobalContext）**
- `fonts_forward: FxHashMap<FontInstance, Font>`：Typst 字体 → Krilla 字体
- `fonts_backward: FxHashMap<Font, FontInstance>`：Krilla 字体 → Typst 字体（用于错误报告反向查找）

**第二级：跨导出缓存（comemo memoization）**
- `#[comemo::memoize]` 标记 `build_font` 函数
- 基于输入参数的哈希值缓存结果
- 同一字体在多次导出间复用 krilla `Font` 对象

### 4.3 可变字体支持

- 收集所有 OpenType 变化轴（weight, width, italic, optical size 等）
- 通过 `Font::new_variable()` 将变化轴值传递给 krilla
- krilla 在字体描述符中设置相应的变化值
- krilla 在字体子集化时应用变化轴实例化（或保留变化轴信息）

### 4.4 字形适配层

在 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/text.rs#L106-L148) 中通过 `PdfGlyph` 透明包装 Typst 的 `Glyph`：

```rust
#[derive(Debug, TransparentWrapper)]
#[repr(transparent)]
struct PdfGlyph(Glyph);

impl krilla::text::Glyph for PdfGlyph {
    fn glyph_id(&self) -> GlyphId { GlyphId::new(self.0.id as u32) }
    fn text_range(&self) -> Range<usize> { ... }
    fn x_advance(&self, size: f32) -> f32 { ... }
    fn x_offset(&self, size: f32) -> f32 { ... }
    fn y_offset(&self, size: f32) -> f32 { ... }
    fn y_advance(&self, size: f32) -> f32 { ... }
    fn location(&self) -> Option<Location> { Some(self.0.span.0.into_raw()) }
}
```

**`Glyph` trait 的关键作用：**
- `glyph_id()`：字体中的字形编号（用于渲染）
- `text_range()`：对应原始文本中的字节范围（用于 ToUnicode 映射）
- `x_advance() / y_advance()`：字形步进宽度（用于排版）
- `x_offset() / y_offset()`：字形偏移（用于排版）
- `location()`：源位置追踪（用于错误报告）

---

## 五、字体子集化 (Font Subsetting)

### 5.1 子集化的必要性

完整的中文字体可能包含数万个字形，文件大小可达 10MB+。子集化只保留文档中实际使用的字形，大幅减小文件体积。

### 5.2 Krilla 的子集化机制

根据 krilla 文档和 Typst 代码推断，krilla 的字体子集化采用**延迟（lazy）策略**：

#### 阶段一：字形收集
- 在 `surface.draw_glyphs()` 调用时，krilla 记录使用的字形 ID
- 所有页面的所有字形引用被收集到字体的使用集合中
- 支持颜色字体（SVG, COLR, sbix, CBDT/EBDT 表）

#### 阶段二：子集化执行
在 `document.finish()` 阶段，对每个字体执行：

1. **字形收集汇总**：合并所有页面中该字体的所有使用字形
2. **添加必需字形**：自动添加 `.notdef` 等必需字形
3. **字体表裁剪**：
   - 移除未使用的字形数据
   - 调整 `glyf` / `CFF` 表
   - 更新 `cmap` 表（字符→字形映射）
   - 调整 `hmtx` / `vmtx` 度量表
   - 保留必要的 OpenType 布局表（GSUB, GPOS 等）
4. **字形 ID 重映射**：子集化后字形 ID 可能改变，需要：
   - 更新内容流中的字形引用
   - 更新 ToUnicode CMap 映射
   - 更新宽度数组（`W` / `Widths`）
5. **生成字体文件流**：将子集化后的字体序列化为 PDF 字体文件流

#### 字体格式支持
- **TTF 字体**：TrueType 轮廓（glyf 表），子集化相对简单
- **CFF 字体**：PostScript Type 1 轮廓（CFF 表），需要特殊处理
- **颜色字体**：保留相应的颜色表（SVG, COLR, sbix, CBDT/EBDT）

### 5.3 错误类型佐证

从 [convert.rs#L600-L630](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L600-L630) 的错误类型可以推断子集化的验证点：

```rust
ValidationError::ContainsNotDefGlyph(f, loc, text) => error!(
    "the text `{}` could not be displayed with {}",
    text.repr(), display_font(gc.fonts_backward.get(f));
    hint: "try using a different font";
),
```

- `.notdef` 字形检查：确保文本都能被字体显示
- 子集化过程中验证所有引用的字形都存在

---

## 六、ToUnicode CMap 处理

### 6.1 什么是 ToUnicode CMap

ToUnicode CMap 是 PDF 中的字符映射表，将字形 ID（Glyph ID）映射回 Unicode 码位。它是文本提取、搜索、复制粘贴的关键。

### 6.2 Krilla 的 ToUnicode 生成

krilla 利用 `Glyph` trait 提供的 `text_range()` 方法来构建 ToUnicode 映射：

```
文本: "Hello"
字形: [Glyph(H), Glyph(e), Glyph(l), Glyph(l), Glyph(o)]
       ↑         ↑        ↑        ↑        ↑
       0..1      1..2     2..3     3..4     4..5  ← text_range
```

#### 映射构建步骤：

1. **字形-文本关联**：通过 `Glyph::text_range()` 获取每个字形对应的文本范围
2. **码位提取**：从文本中提取对应范围的 Unicode 字符
3. **映射去重**：多个字形映射到同一字符时进行合并处理
4. **CMap 格式生成**：生成符合 PDF 规范的 ToUnicode CMap 流
   - 使用 `beginbfchar` / `endbfchar` 单字符映射
   - 使用 `beginbfrange` / `endbfrange` 范围映射优化体积

### 6.3 复杂脚本的挑战

从错误信息 [convert.rs#L607-L619](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L607-L619) 可以看到：

```rust
ValidationError::NoCodepointMapping(_, _, loc) => {
    let msg = if loc.is_some() {
        "the text was not mapped to a code point"
    } else {
        "the PDF contains text with missing codepoints"
    };
    error!(..., "{msg}";
        hint: "for complex scripts like Arabic, it might not be \
               possible to produce a compliant document";
    );
}
```

**复杂脚本的难点：**
- 连字（ligature）：一个字形对应多个字符（如 "fi" 连字）
- 上下文替换：字形根据上下文变化
- 双向文本：显示顺序与逻辑顺序不同
- 标记重排：变音符号的顺序调整

### 6.4 ToUnicode 与 PDF 标准

- **PDF/A**：要求所有文本都有 ToUnicode 映射
- **PDF/UA**：强制要求 ToUnicode 以保证可访问性
- **标签 PDF**：需要 ToUnicode 配合标签树进行文本提取

---

## 七、从页面内容到 PDF 字节的完整链路

### 7.1 整体流程总览

```
┌─────────────────────────────────────────────────────────────┐
│  阶段 1: Typst 排版输出                                      │
│  PagedDocument (N 个 Page, 每个 Page 有一个 Frame)          │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 2: 初始化与准备                                        │
│  - 创建 krilla Document + SerializeSettings                 │
│  - 创建 GlobalContext（字体缓存、图像缓存、标签上下文）      │
│  - 预构建标签树（语义结构预遍历）                            │
│  - 收集命名目标（锚点）                                      │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 3: 逐页面转换 (循环 N 次)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3.1 创建 Page + Surface (内容流构建器)                │  │
│  │     - 页面尺寸、出血、页码标签                         │  │
│  │     - 初始化页面资源字典 (空)                          │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3.2 Frame 递归遍历 (handle_frame)                     │  │
│  │     - 变换状态栈管理 (push/pop)                       │  │
│  │     - 处理 Group(变换+裁剪)                           │  │
│  │     - 处理 Text → draw_glyphs() → 记录字形使用        │  │
│  │     - 处理 Shape → draw_path() → 记录填充/描边资源    │  │
│  │     - 处理 Image → draw_image() → 注册图像资源        │  │
│  │     - 处理 Link → 收集链接注解                        │  │
│  │     - 处理 Tag → 标签树标记                           │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3.3 surface.finish()                                  │  │
│  │     - 完成内容流构建                                  │  │
│  │     - 冻结页面资源字典                                │  │
│  │     - 将内容流转为内容流对象                          │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3.4 添加页面注解 (链接等)                              │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 4: 辅助内容设置                                        │
│  - 文档大纲 (Outline)                                       │
│  - 元数据 (Metadata / XMP)                                  │
│  - 文件附件 (Embedded Files)                                │
│  - 标签树 (Tag Tree)                                        │
│  - 命名目标 (Named Destinations)                            │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│  阶段 5: document.finish() - 最终序列化                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.1 字体处理                                          │  │
│  │     - 收集所有字体的已使用字形                        │  │
│  │     - 字体子集化 (TTF/CFF 都支持)                     │  │
│  │     - 生成 ToUnicode CMap                             │  │
│  │     - 构建字体描述符 (FontDescriptor)                 │  │
│  │     - 生成字体文件流 (FontFile2 / FontFile3)          │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.2 对象收集与编号                                    │  │
│  │     - 遍历所有页面、资源、辅助对象                    │  │
│  │     - 分配间接对象编号 (1, 2, 3, ...)                 │  │
│  │     - 构建对象 → 编号映射                             │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.3 内容流重写 (如果需要)                              │  │
│  │     - 字形 ID 重映射 (子集化后字形顺序变化)           │  │
│  │     - 资源名称替换 (字体 F1, 图像 Im1 等)             │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.4 字节序列化                                        │  │
│  │     - 写入 PDF 头部 (%PDF-1.x)                        │  │
│  │     - 按编号顺序写入所有间接对象                       │  │
│  │     - 记录每个对象的偏移位置                           │  │
│  │     - 构建交叉引用表 (xref table / xref stream)       │  │
│  │     - 构建文档目录 (Catalog)                           │  │
│  │     - 写入页面树 (Pages)                              │  │
│  │     - 写入尾部 (trailer + startxref)                  │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.5 压缩与优化                                        │  │
│  │     - 内容流压缩 (FlateDecode)                        │  │
│  │     - 对象流 (Object Streams, PDF 1.5+)               │  │
│  │     - 交叉引用流 (XRef Streams, PDF 1.5+)             │  │
│  └──────────────────────┬────────────────────────────────┘  │
│                         ↓                                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5.6 标准合规性验证                                    │  │
│  │     - PDF/A 验证 (字体嵌入、元数据等)                 │  │
│  │     - PDF/UA 验证 (标签、语言、ToUnicode)             │  │
│  │     - 版本限制检查 (对象数量、字符串长度等)            │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
                  Vec<u8> (PDF 字节流)
```

### 7.2 关键函数调用链

**顶层调用链：**
```
typst_pdf::pdf()
  └─ convert::convert()
      ├─ Document::new_with()          // 创建 krilla 文档
      ├─ collect_named_destinations()  // 收集锚点
      ├─ tags::init()                  // 初始化标签树
      ├─ GlobalContext::new()          // 创建全局上下文
      ├─ convert_pages()               // 转换所有页面
      │   └─ (循环每个页面)
      │       ├─ document.start_page_with()
      │       ├─ page.surface()
      │       ├─ handle_frame()        // 递归处理 Frame
      │       ├─ surface.finish()
      │       └─ page.finish()
      ├─ attach_files()                // 附加文件
      ├─ tags::resolve()               // 解析标签树
      ├─ document.set_outline()
      ├─ document.set_metadata()
      ├─ document.set_tag_tree()
      └─ document.finish()             // 最终序列化 → Vec<u8>
```

**Frame 递归调用链：**
```
handle_frame()
  ├─ fc.push()                        // 保存变换状态
  ├─ handle_shape()                   // 背景填充
  ├─ fc.push()
  ├─ (遍历所有 frame items)
  │   ├─ FrameItem::Group → handle_group()
  │   │   ├─ fc.push() + 变换
  │   │   ├─ clip_path
  │   │   └─ handle_frame() (递归)
  │   ├─ FrameItem::Text → handle_text()
  │   │   ├─ convert_font()
  │   │   ├─ paint::convert_fill()
  │   │   └─ surface.draw_glyphs()
  │   ├─ FrameItem::Shape → handle_shape()
  │   │   ├─ convert_geometry() → Path
  │   │   ├─ paint::convert_fill/stroke()
  │   │   └─ surface.draw_path()
  │   ├─ FrameItem::Image → handle_image()
  │   │   ├─ convert_raster/svg/pdf()
  │   │   └─ surface.draw_image/draw_svg/draw_pdf_page()
  │   ├─ FrameItem::Link → handle_link()
  │   └─ FrameItem::Tag → tags::handle_start/end()
  └─ fc.pop() (多次)
```

---

## 八、PDF 标准合规性

通过 `PdfStandards` 和 `PdfStandard` 配置，在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/lib.rs) 中定义：

**支持的标准：**
- PDF 版本：1.4, 1.5, 1.6, 1.7, 2.0
- PDF/A（归档）：A-1b/a, A-2b/u/a, A-3b/u/a, A-4, A-4f, A-4e
- PDF/UA（无障碍）：UA-1

**与字体/文本相关的验证点：**
- 字体必须完全嵌入（PDF/A 要求）
- ToUnicode CMap 必须存在（PDF/UA 要求）
- 字体名称长度限制（127 字符）
- 禁止使用 .notdef 字形显示实际文本
- 字符映射有效性验证

---

## 九、性能优化策略

1. **字体缓存**：双向哈希映射 + comemo 缓存，避免重复解析字体
2. **图像缓存**：相同图像复用 krilla Image 对象（间接对象去重）
3. **延迟初始化**：`OnceLock` 用于图像通道提取
4. **memoization**：`#[comemo::memoize]` 宏缓存纯函数结果
5. **哈希映射**：使用 `rustc-hash` (FxHashMap/FxHashSet) 提高性能
6. **字体子集化**：减小最终文件体积
7. **内容流压缩**：Flate 压缩减少传输体积

---

## 十、错误处理

错误转换层在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L536-L838) 的 `convert_error` 函数中实现，将 krilla 错误映射为 Typst 友好的诊断信息。

**字体相关错误：**
- `KrillaError::Font`：字体处理失败
- `ValidationError::ContainsNotDefGlyph`：文本无法被字体显示
- `ValidationError::NoCodepointMapping`：缺少 Unicode 码位映射
- `ValidationError::InvalidCodepointMapping`：无效的码位映射

**对象相关错误：**
- `ValidationError::TooManyIndirectObjects`：间接对象过多
- `ValidationError::TooLongArray`：数组过长（>8191 元素）
- `ValidationError::TooLongDictionary`：字典过大（>4095 条目）

---

## 总结

### 对象与资源组织层次

```
PDF 文件
├─ Catalog (文档目录)
├─ Pages (页面树)
│   └─ Page (每个页面)
│       ├─ Contents (内容流)
│       ├─ Resources (页面资源)
│       │   ├─ Font (字体字典)
│       │   │   └─ /F1 → 字体对象引用
│       │   ├─ XObject (外部对象)
│       │   │   ├─ /Im1 → 图像对象引用
│       │   │   └─ /Form1 → 表单对象引用
│       │   ├─ Pattern (图案)
│       │   ├─ Shading (渐变)
│       │   ├─ ColorSpace (颜色空间)
│       │   └─ ExtGState (图形状态参数)
│       └─ Annots (注解数组)
├─ Font (字体对象) ←──┐
│   ├─ FontDescriptor │ (间接对象)
│   │   └─ FontFile2/3 (字体文件流)
│   ├─ ToUnicode CMap
│   └─ Widths (宽度数组)
├─ Outline (文档大纲)
├─ Metadata (元数据)
├─ StructTreeRoot (标签树根)
├─ EmbeddedFiles (附件)
└─ xref (交叉引用表)
```

### 字体嵌入全流程

```
FontInstance (Typst)
    ↓ convert_font()
krilla::text::Font (包装原始字体数据)
    ↓ surface.draw_glyphs()
记录使用的字形 ID + 文本范围
    ↓ (所有页面处理完成后)
document.finish()
    ↓
┌─────────────────────────────────┐
│ 1. 收集所有使用的字形            │
│ 2. 添加 .notdef 等必需字形       │
│ 3. 字体子集化 (TTF/CFF)         │
│ 4. 生成 ToUnicode CMap          │
│ 5. 构建字体描述符                │
│ 6. 生成字体文件流 (压缩)         │
│ 7. 分配对象编号                  │
└─────────────────────────────────┘
    ↓
PDF 中的字体对象体系
```
