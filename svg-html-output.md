# SVG 与 HTML 导出后端差异分析

本文档从**文本处理**、**图形渲染**、**样式管理**三个维度，深入分析 Typst 的 SVG 与 HTML 两类导出后端的实现机制与核心差异。所有代码引用使用仓库相对路径，便于在任意机器上核对。

---

## 代码索引

| 模块 | 路径 | 说明 |
|------|------|------|
| SVG 主入口 | `crates/typst-svg/src/lib.rs` | SVGRenderer 结构体、渲染分发、资源去重 |
| SVG 文本 | `crates/typst-svg/src/text.rs` | 字形提取、路径/图像字形渲染 |
| SVG 图形 | `crates/typst-svg/src/shape.rs` | 几何转 Path、描边属性生成 |
| SVG 样式 | `crates/typst-svg/src/paint.rs` | 填充/描边、渐变/图案/颜色空间 |
| SVG 路径构建 | `crates/typst-svg/src/path.rs` | SvgPathBuilder（相对坐标优化） |
| SVG XML 写入 | `crates/typst-svg/src/write.rs` | 元素/属性格式化、Transform 序列化 |
| HTML 主入口 | `crates/typst-html/src/lib.rs` | 模块定义、HtmlElem/FrameElem 声明 |
| HTML 内容转换 | `crates/typst-html/src/convert.rs` | 内容到 DOM 节点的核心转换逻辑 |
| HTML Show Rules | `crates/typst-html/src/rules.rs` | 各元素到 HTML 标签的映射规则 |
| HTML DOM 结构 | `crates/typst-html/src/dom.rs` | HtmlNode/HtmlElement/HtmlFrame 定义 |
| HTML 编码输出 | `crates/typst-html/src/encode.rs` | DOM 到 HTML 字符串的编码 |
| HTML CSS 解析 | `crates/typst-html/src/css/mod.rs` | CSS 属性管理 |
| HTML CSS 内联 | `crates/typst-html/src/css/resolve.rs` | CSS 属性合并为 style 属性 |

---

## 一、整体架构对比

| 维度 | SVG 后端 (typst-svg) | HTML 后端 (typst-html) |
|------|---------------------|----------------------|
| **输入** | 布局完成的 `Frame` / `Page` | 语义内容树 `Content` + 样式链 |
| **核心思路** | 将已布局结果逐像素翻译为 SVG 元素 | 通过 show rule 将语义元素映射为 HTML 标签；未注册元素被忽略；只有用户显式 `#html.frame(...)` 才嵌入内联 SVG，无自动回退 |
| **布局责任** | Typst 引擎完成全部布局 | 由浏览器 CSS 布局引擎负责 |
| **输出粒度** | 单个 SVG 文档（单页/合并多页） | 完整 HTML 文档（含 DOM 树） |
| **坐标系统** | 绝对坐标（pt 单位），精确到每个元素 | 文档流 + CSS 盒模型 |

### 入口函数对比

**SVG 后端** 直接接收布局完成的页面/帧：
- `svg()` — 单页导出：`crates/typst-svg/src/lib.rs` 第 32-43 行
- `svg_merged()` — 多页合并：`crates/typst-svg/src/lib.rs` 第 128-156 行
- `svg_in_html()` — 嵌入 HTML 的内联 SVG：`crates/typst-svg/src/lib.rs` 第 82-123 行

**HTML 后端** 接收内容并通过 show rule 系统转换：
- `html_document()` — 完整 HTML 文档：`crates/typst-html/src/document.rs`
- `convert_to_nodes()` — 内容到 HTML 节点的核心转换：`crates/typst-html/src/convert.rs` 第 60-90 行

### 架构差异的代码依据

SVG 后端的 `render_frame()`（`crates/typst-svg/src/lib.rs` 第 313-327 行）直接遍历 `FrameItem`（Group/Text/Shape/Image/Link），说明它消费的是**已布局产物**。

HTML 后端的 `handle()`（`crates/typst-html/src/convert.rs` 第 93-163 行）则对 `TagElem`/`HtmlElem`/`TextElem`/`BoxElem`/`BlockElem`/`FrameElem` 等语义元素逐一匹配，说明它消费的是**语义内容树**。

---

## 二、文本处理差异

### 2.1 SVG 后端：字形级精确渲染

**核心文件**：`crates/typst-svg/src/text.rs`

核心特征：**将文本转化为矢量路径或图像**，完全不依赖系统字体。

#### 渲染流程

```
TextItem → 遍历 glyphs → 分类渲染 → 去重复用
                              ↓
                   ┌──────────┴──────────┐
                   ↓                     ↓
            轮廓字形 (Path)         图像字形 (Frame)
         （普通文字、矢量字形）   （彩色 emoji、COLR/CPAL 字体）
```

#### 两种字形表示（`RenderedGlyph` 枚举，第 16-23 行）

**1. 路径字形 (`RenderedGlyph::Path`)**
- 代码位置：`crates/typst-svg/src/text.rs` 第 64-77 行 `render_glyph()` 中的 `should_outline` 分支
- 用 `ttf_parser::OutlineBuilder` trait（`crates/typst-svg/src/path.rs` 第 170-202 行）从字体文件提取字形轮廓
- 通过 `SvgPathBuilder`（`crates/typst-svg/src/path.rs`）转换为 SVG path 数据
- **预缩放处理**：`scale = text_size / units_per_em`，避免使用时缩放影响描边粗细

**2. 图像字形 (`RenderedGlyph::Frame`)**
- 代码位置：`crates/typst-svg/src/text.rs` 第 78-91 行 `render_glyph()` 的 else 分支
- 适用于 COLR/CPAL、SVG-in-OT、位图等彩色字形
- 字形尺寸较大，使用时再缩放（减少内存占用）

#### 关键实现细节

- **Y 轴翻转**：字体使用 Y-Up 坐标系，SVG 绘制前需翻转 Y 轴
  ```rust
  let state = state.pre_concat(Transform::scale(Ratio::one(), -Ratio::one()));
  ```
  代码位置：`crates/typst-svg/src/text.rs` 第 38 行 `render_text()`

- **字形去重**：通过 `Deduplicator` 机制，相同字形只定义一次
  - 路径字形 key：`(font, glyph_id, scale)`（第 68 行）
  - 图像字形 key：`(font, glyph_id)`（第 82 行）
  - 定义在 `<defs>` 的 `<symbol>` 中（`write_glyph_defs()` 第 182-217 行）
  - 使用时 `<use xlink:href="#g...">`（`render_path_glyph()` 第 142 行 / `render_image_glyph()` 第 108-110 行）

- **文本样式**：填充和描边直接作用于 `<use>` 元素
  - 填充支持纯色、渐变、图案（`write_fill()` 来自 paint.rs）
  - 描边支持厚度、端点、连接、虚线等
  - 代码位置：`crates/typst-svg/src/text.rs` 第 146-160 行

### 2.2 HTML 后端：语义化文本节点

**核心文件**：`crates/typst-html/src/convert.rs` 的 `handle_text()`（第 252-334 行）

核心特征：**保留文本语义，依赖浏览器字体渲染**。

#### 文本转换流程

```
TextElem → 应用大小写变换 → 特殊字符处理 → 生成 HtmlNode::Text
                ↓
        ┌───────┴───────┐
        ↓               ↓
    普通字符        空白字符处理
    直接输出        (空格折叠防护)
```

代码依据：`convert_to_nodes()`（第 60-90 行）→ `handle()`（第 93-163 行）中 `TextElem` 分支（第 104-110 行）→ `handle_text()`（第 252-334 行）。

#### 空白字符处理（核心复杂度）

由于 HTML 会自动折叠连续空白（CSS Text 规范 white-space-rules），HTML 后端设计了三层防护：

**1. 连续空白保护**
- 连续空格/制表符 → 包装在 `<span style="white-space: pre-wrap">` 中
- 触发时机：`Converter::flush_whitespace()` 第 576-584 行
- 生成函数：`pre_wrap()` 第 675-683 行

**2. 单空格保护（后处理阶段）**
- 单空格在普通文本之间无需保护
- 但相邻元素边界处的单空格需要保护（如 `<span>` 旁边的空格）
- 通过 `Protector` 状态机（第 598-665 行）在 `protect_spaces()`（第 591-595 行）中遍历整棵 DOM 实现
- 状态转换逻辑：`Collapsing` → `Supportive` → `Space`，相邻不同元素时决定是否包裹

**3. 换行处理**
- 普通模式下换行 → `<br>` 元素：`handle_text()` 第 319 行
- `<pre>` 等预格式化元素中 → 直接保留换行字符：由 `Whitespace::Pre` 模式控制（第 279-281 行）

#### 智能引号

HTML 后端内置 `SmartQuoter` 状态机（`convert.rs` 第 72-74 行构造），根据上下文自动将直引号转为弯引号，支持多语言。代码位置：`SmartQuoteElem` 分支（第 121-135 行）。

### 2.3 文本处理对比总结 + 差异对应说明

| 特性 | SVG 后端 | HTML 后端 | 差异根源代码 |
|------|---------|----------|------------|
| 文本表示 | 字形路径 / 图像 | 纯文本节点 | SVG: `RenderedGlyph` 枚举 `text.rs` 第 16-23 行；HTML: `HtmlNode::Text` `dom.rs` 第 105 行 |
| 字体依赖 | 嵌入字形轮廓，无需系统字体 | 依赖浏览器/系统字体 | SVG: `ttf_parser::outline_glyph()` `text.rs` 第 71 行；HTML: 直接输出文字无字体嵌入 `convert.rs` 第 304 行 |
| 排版精度 | 100% 还原 Typst 布局 | 浏览器排版，可能有差异 | SVG: 消费布局后 `Frame` `lib.rs` 第 314 行；HTML: 消费语义 `Content` `convert.rs` 第 80 行 |
| 文本可复制 | 通常不可选（纯图形） | 原生可选可复制 | SVG: 字形为 `<path>` 非 `<text>`；HTML: 浏览器原生文本节点 |
| 空白处理 | 精确位置，无折叠问题 | 需要专门防护空白折叠 | SVG: 绝对坐标定位 `text.rs` 第 41-52 行 x/y 累加；HTML: `Protector` 状态机 `convert.rs` 第 598-665 行 |
| 彩色 emoji | 直接渲染字形图像 | 依赖浏览器字体支持 | SVG: `RenderedGlyph::Frame` + `glyph_frame()` `text.rs` 第 82-86 行；HTML: 输出 Unicode 码点由浏览器画 |
| 文件大小 | 大（每个字形都是路径） | 小（纯文本） | SVG: path 字符串如 `M 0 0 c ...`；HTML: 直接文本字符 |

**差异核心**：SVG 后端选择"**所见即所得**"，将字体字形嵌入文档；HTML 后端选择"**语义+浏览器**"，依赖浏览器字体栈和 CSS 文本模型，因此需要额外处理空白折叠问题。

---

## 三、图形处理差异

### 3.1 SVG 后端：原生矢量图形

**核心文件**：`crates/typst-svg/src/shape.rs`

核心思路：**统一转换为 SVG path**，所有几何形状最终都输出为 `<path d="...">` 元素。

#### 几何图形统一为 Path

`convert_geometry_to_path()`（第 178-188 行）统一将三种 `Geometry` 转为 SVG 路径字符串：

| Typst 几何 | SVG path 示例 | 转换代码行 |
|-----------|--------------|-----------|
| `Geometry::Line` | `M x1 y1 L x2 y2` | `shape.rs` 第 181 行 |
| `Geometry::Rect` | `M 0 0 v h h -v Z`（相对坐标） | `path.rs` 第 67-73 行 `rect()` |
| `Geometry::Curve` | `M x y C x1 y1 x2 y2 x y ...`（逐段贝塞尔） | `shape.rs` 第 183-185 行 `convert_curve()` |

`render_shape()`（第 13-48 行）生成的最终输出是单个 `<path>` 元素，包含 fill、stroke、transform、d 四个核心属性。

#### SvgPathBuilder 优化策略

**核心文件**：`crates/typst-svg/src/path.rs`

`SvgPathBuilder` 使用**相对坐标**（小写命令）减小文件体积：

| 命令 | 含义 | 优化策略 | 代码位置 |
|------|------|---------|---------|
| `m` | 相对移动 | 相对零偏移时省略命令（第 109-113 行） | `path.rs` 第 102-114 行 |
| `l` | 相对直线 | 水平用 `h`，垂直用 `v` 单轴命令（第 122-131 行） | `path.rs` 第 116-132 行 |
| `c` | 三次贝塞尔 | — | `path.rs` 第 134-151 行 |
| `q` | 二次贝塞尔 | — | `path.rs` 第 153-162 行 |
| `a` | 弧线 | — | `path.rs` 第 76-100 行 |
| `Z` | 闭合路径 | — | `path.rs` 第 164-167 行 |

所有坐标先经过 `map()`（第 61-63 行）换算为相对偏移，再写入。

#### 组与裁剪

`render_group()`（`lib.rs` 第 331-363 行）实现帧到 `<g>` 的映射：

- **软帧 (FrameKind::Soft)**：叠加 `transform`，不重置坐标系（第 335 行）
- **硬帧 (FrameKind::Hard)**：强制生成 `<g>`，重置 `state.transform` 为 identity（第 336-347 行）
- **裁剪路径 (clip)**：通过 `Deduplicator` 去重存储在 `<defs>`，`url(#c...)` 引用（第 354-359 行）

### 3.2 HTML 后端：语义元素 + 显式 FrameElem 触发内联 SVG

**核心文件**：`crates/typst-html/src/rules.rs`（show rule 映射）+ `crates/typst-html/src/dom.rs`（`HtmlFrame` 定义）

核心策略：**双路径无自动回退**——有 show rule 的转 HTML 标签；无 show rule 的发出警告后跳过；只有用户显式用 `#html.frame(...)` 包裹内容时才嵌入内联 SVG。

> **关键区分**：`handle()`（`convert.rs` 第 93-163 行）是一个 if-else if-else 链：
> 1. `TagElem` / `HtmlElem` / `SpaceElem` / `TextElem` / `LinebreakElem` 等基础语义元素 → 直接生成 DOM 节点
> 2. `FrameElem`（即用户写 `#html.frame(...)`）→ 走 Paged 布局 → 嵌入内联 SVG（第 140-154 行）
> 3. **所有其他元素** → else 分支，发出警告 `"{} was ignored during HTML export"` 并跳过（第 155-161 行）
>
> 因此：**HTML 后端没有任何机制会自动把不支持的元素转成 SVG**。`LineElem` 被忽略、含渐变内容被静默丢弃，都不会自动回退到 `html.frame`。

#### 1. 语义化图形元素（Show Rule 映射）

`register()`（`rules.rs` 第 39-92 行）为 HTML 目标注册了约 40 条 show rule。图形相关的核心映射：

| Typst 元素 | HTML 标签 | Rule 名称 | 代码位置 |
|-----------|----------|----------|---------|
| `image` | `<img>` | `IMAGE_RULE` | `rules.rs` 第 773-820 行 |
| `table` | `<table>/<thead>/<tbody>/<tr>/<td>/<th>` | `TABLE_RULE` | `rules.rs` 第 573-687 行 |
| `divider` | `<hr>` | `DIVIDER_RULE` | `rules.rs` 第 210-212 行 |

> **注意**：`divider`（`DividerElem`，语义分割线，如 Markdown 的 `---`）映射到 `<hr>`，但 `line`（`LineElem`，几何直线，如 `#line(length: 100%)`）**没有 show rule**。HTML 后端对未注册的元素一律走 `handle()` 的 else 分支（`convert.rs` 第 155-161 行），输出警告 `"line was ignored during HTML export"` 并跳过。SVG 后端则通过布局引擎先将 `LineElem` 转为 `FrameItem::Shape`（`Geometry::Line`），再由 `render_shape()` 绘制为 `<path>`。

**图片内联**：`IMAGE_RULE`（第 774-779 行）使用 `typst_svg::WebImage::to_base64_url()` 将图片编码为 Data URL 嵌入 `<img src="data:...">`，确保单文件可分发。

**表格语义化**：`TABLE_RULE`（第 573-687 行）调用 `show_cellgrid()`（第 578-687 行）将 `CellGrid` 拆分为 `<thead>`/`<tbody>`/`<tfoot>`，处理 colspan/rowspan 以及 header 行在 body 中间出现的复杂情况。

#### 2. FrameElem：用户显式触发的 SVG 嵌入桥接

当用户在 Typst 源码中显式写 `#html.frame(content)` 时（`lib.rs` 第 126-141 行 `FrameElem` 定义，`module()` 第 38 行注册为 `html.frame` 函数），`convert.rs` 第 140-154 行的 `FrameElem` 分支才会被触发，此时 HTML 后端才会：
1. 切换到 `Target::Paged` 目标重新布局（绕过 HTML 语义布局）
2. 调用 Typst 布局引擎得到精确 `Frame`
3. 包装为 `HtmlFrame`，在最终编码时由 `write_frame()`（`encode.rs` 第 391-401 行）调用 `typst_svg::svg_in_html()` 生成内联 SVG 插入 HTML

转换流程（`convert.rs` 第 140-154 行）：

```rust
// Step 1: 用 Paged 目标重新布局，得到精确 Frame（绕过 HTML 布局）
let style = TargetElem::target.set(Target::Paged).wrap();
let frame = (converter.engine.library.routines.layout_frame)(
    converter.engine, &elem.body, locator, styles.chain(&style), region
)?;
// Step 2: 包装为 HtmlFrame，标记默认块级
let mut node = HtmlFrame::new(frame, styles, elem.span()).into();
make_block_level(&mut node).unwrap();
converter.push(node);
```

`HtmlFrame` 定义（`dom.rs` 第 506-535 行）保存了：
- `inner: Frame` — 布局结果
- `text_size: Abs` — 用 em 单位尺寸 SVG
- `anchors: EcoVec<(Point, EcoString)>` — 跨文档链接锚点

**最终编码**：`encode.rs` 第 391-401 行 `write_frame()` 调用 `typst_svg::svg_in_html()` 生成内联 SVG，直接插入 HTML 文档流。

### 3.3 图形处理对比总结 + 差异对应说明

| 特性 | SVG 后端 | HTML 后端 | 差异根源代码 |
|------|---------|----------|------------|
| 图形表示 | 原生 SVG 矢量路径 | 混合：HTML 语义标签 + **显式** `html.frame()` 触发的内联 SVG | SVG: `<path>` 统一表示 `shape.rs` 第 19 行；HTML: show rule 注册 `rules.rs` 第 42-86 行 + `FrameElem` 分支 `convert.rs` 第 140-154 行（仅用户显式包裹时触发） |
| 几何精度 | 精确 pt 级坐标 | 浏览器像素级渲染；内联 SVG 部分可精确到 pt | SVG: `SvgTransform` 矩阵 `write.rs` 第 206-241 行；HTML: 浏览器 CSS box model；`html.frame()` 内联 SVG 由 SVG 后端精确渲染 |
| 表格 | 用路径绘制线条和文字 | 原生 `<table>`，语义化 | SVG: 无特殊处理，走 Frame→path；HTML: `TABLE_RULE` 拆成 `<thead>`/`<tbody>` `rules.rs` 第 573-687 行 |
| 图片 | `<image>` 嵌入 | `<img>` Data URL | SVG: `image.rs` 模块；HTML: `IMAGE_RULE` 第 778-779 行 `to_base64_url()` |
| 矩形/圆形 | 统一转为 `<path>` | **默认被忽略**，需用户显式 `html.frame(...)` 包裹后嵌入 SVG | SVG: `Geometry::Rect → SvgPathBuilder::rect()` `path.rs` 第 67-73 行；HTML: 无对应 show rule → `convert.rs` 第 155-161 行 else 分支警告跳过 |
| 几何直线 | `Geometry::Line` → `<path>` | **默认被忽略**，需用户显式 `html.frame(...)` 包裹后嵌入 SVG | SVG: `shape.rs` 第 181 行；HTML: 无 LineElem show rule → `convert.rs` 第 155-161 行 else 分支（"line was ignored during HTML export"警告） |
| 裁剪遮罩 | 原生 `clip-path` 支持 | **默认被忽略**；可通过 `html.frame()` 包裹获得 SVG 的 clip-path | SVG: `clip_paths` Deduplicator `lib.rs` 第 354-359 行；HTML: 无裁剪相关 show rule |
| 布局控制 | 完全由 Typst 控制 | 浏览器盒模型 + 文档流 | SVG: 消费 `Frame` 绝对坐标 `lib.rs` 第 314-315 行 `pre_translate`；HTML: 文档流语义标签 |
| 可访问性 | 弱（纯图形） | 强（语义标签 + ARIA） | SVG: 仅 `data-typst-label` `lib.rs` 第 350-352 行；HTML: `HEADING_RULE` 用 `<h2>`/ARIA `rules.rs` 第 235-260 行 |

**差异核心**：SVG 后端是**统一抽象**——一切皆路径；HTML 后端是**两条明确路径**，没有自动回退：
- **路径 1（语义转换）**：注册了 show rule 的元素（`DividerElem`→`<hr>`、`TableElem`→`<table>`、`ImageElem`→`<img>` 等约 40 个）直接转成 HTML 标签，由浏览器布局
- **路径 2（用户主动回退）**：用户在 Typst 代码中显式写 `#html.frame(content)` 时（`lib.rs` 第 126-141 行 `FrameElem` 定义），`convert.rs` 第 140-154 行走 Paged 布局 → 调 `typst_svg::svg_in_html()` 嵌入内联 SVG
- **路径 3（被忽略）**：未注册 show rule 且未被显式 `html.frame()` 包裹的元素（如 `LineElem`、`RectElem`、含渐变的文字等）走 `convert.rs` 第 155-161 行 else 分支，发出警告 `"X was ignored during HTML export"` 后直接跳过，不会自动嵌入 SVG

这也解释了为什么 HTML 文件更小、更可访问，但对无 show rule 的几何元素会静默丢失。

---

## 四、样式处理差异

### 4.1 SVG 后端：属性式样式

**核心文件**：`crates/typst-svg/src/paint.rs` + `crates/typst-svg/src/shape.rs`

核心思路：**将 Typst 的 Paint/Stroke 映射为 SVG 元素属性**（`fill`、`stroke`、`stroke-width` 等）。

#### 填充 (fill)

`write_fill()`（`paint.rs` 第 32-57 行）根据 `Paint` 枚举分支：

| Paint 类型 | SVG 表示 | 生成代码 |
|-----------|---------|---------|
| `Paint::Solid` | `fill="#rrggbb"` / `fill="oklab(...)"` | `paint.rs` 第 41-43 行 |
| `Paint::Gradient` | `fill="url(#f...)"` → 引用 `<defs>` 中的渐变元素 | `paint.rs` 第 44-47 行 |
| `Paint::Tiling` | `fill="url(#t...)"` → 引用 `<defs>` 中的 pattern | `paint.rs` 第 48-51 行 |

附加 `fill-rule`（nonzero / evenodd）：第 53-56 行。

#### 描边 (stroke)

`write_stroke()`（`shape.rs` 第 128-173 行）生成完整的 SVG 描边属性族：

| Typst 属性 | SVG 属性 | 代码位置 |
|-----------|---------|---------|
| `stroke.paint` | `stroke`（纯色/渐变/图案，同 fill 逻辑） | `shape.rs` 第 135-147 行 |
| `stroke.thickness` | `stroke-width` | `shape.rs` 第 149 行 |
| `stroke.cap` | `stroke-linecap`（butt/round/square） | `shape.rs` 第 150-157 行 |
| `stroke.join` | `stroke-linejoin`（miter/round/bevel） | `shape.rs` 第 158-165 行 |
| `stroke.miter_limit` | `stroke-miterlimit` | `shape.rs` 第 166 行 |
| `stroke.dash` | `stroke-dasharray` + `stroke-dashoffset` | `shape.rs` 第 167-172 行 |

#### 颜色空间支持

`SvgDisplay for Color`（`paint.rs` 第 444-505 行）实现了丰富的颜色序列化格式：

| 颜色空间 | SVG 输出格式 | 代码分支行 |
|---------|-------------|-----------|
| RGB / Luma / CMYK / HSV | `#rrggbb` 十六进制 | `paint.rs` 第 447-452 行 |
| Linear RGB | `color(srgb-linear r g b / a)` | `paint.rs` 第 453-461 行 |
| Oklab | `oklab(L% a b / a)` | `paint.rs` 第 462-472 行 |
| Oklch | `oklch(L% c h / a)` | `paint.rs` 第 473-485 行 |
| HSL | `hsl(h s% l% / a)` / `hsla(...)` | `paint.rs` 第 486-502 行 |

#### 渐变的两级去重优化

`push_gradient()`（`paint.rs` 第 66-85 行）实现了精巧的两级去重：

```
Level 1: 原始渐变 (gradients Deduplicator)
  Key: (gradient, aspect_ratio)       ← 不含 transform，可跨元素复用
  输出: <linearGradient id="f..."> ... </linearGradient>
  写入: write_gradients() 第 110-277 行

Level 2: 渐变引用 (gradient_refs Deduplicator)
  Key: (gradient_id, transform)       ← 附加变换
  输出: <linearGradient id="r..." href="#f..." gradientTransform="matrix(...)"/>
  写入: write_gradient_refs() 第 311-331 行
```

**优化原理**：渐变定义（含全部 stops）通常几十到几百字节，而 transform 只有几十字节。相同形状不同位置的渐变共享 Level 1，Level 2 只记录变换，大幅减小文件体积。

#### 锥形渐变的特殊处理

SVG 1.1 无原生锥形渐变，`write_gradients()` 的 `Gradient::Conic` 分支（`paint.rs` 第 160-236 行）采用**分段模拟**：
1. 将圆分为 360 个扇形段（`NUM_CONIC_SEGMENTS = 360`，第 19 行）
2. 每段作为 `<path>` 用 `arc` 命令绘制（第 200-210 行）
3. 每段用独立线性渐变填充（`conic_subgradients` Deduplicator，第 214-231 行）
4. 整体包装在 `<pattern>` 中模拟锥形

### 4.2 HTML 后端：CSS 属性式样式

**核心文件**：`crates/typst-html/src/css/mod.rs` + `crates/typst-html/src/css/resolve.rs` + `crates/typst-html/src/convert.rs`

核心思路：**通过 `css::Properties` 收集样式，最终合并为内联 `style` 属性**，部分用 `class` 标记以便用户覆盖。

#### CSS 管理机制

1. **`css::Properties`**：CSS 属性键值对集合（`crates/typst-html/src/css/encode.rs` 中定义）
2. **`resolve_inline_styles()`**：遍历 DOM 树，将元素的 `css` 字段合并到 `style` 属性（`css/resolve.rs` 第 11-37 行）
   - 如果元素已有 `style` 属性，生成的 CSS 放前面，原有放后面
   - 遍历所有子元素递归处理
3. **最终输出**：`style="prop1: value1; prop2: value2"` 写入元素属性

#### 显示模式 (display) 控制

`convert.rs` 中实现了精细的 `display` 属性管理：

- **块级提升** `make_block_level()`（第 444-486 行）：根据 tag 默认 display 属性决定是否加 `display: block`
  - `<nav>`/`<section>`/`<figure>` 等已有 block → 不加
  - `<span>`/`<img>`/`<math>` 行内 → 设为 `block` / `block math`
  - `HtmlFrame`（SVG 嵌入）→ 默认 `inline`，需显式提升
- **行内降级** `make_inline_level()`（第 381-391 行）：确保 `<p>` 内元素是 phrasing content
  - `<math display="block">` → 改为 `inline math`
  - 其他 block display →  unset

#### 类名 (class) 策略

部分元素用 `class` 而非内联样式，方便用户 CSS 覆盖（而不是硬编码 inline style）：

| 类名 / 选择器 | 用途 | 代码位置 |
|-------------|------|---------|
| `.hanging-indent` | 参考文献悬挂缩进标记 | `BIBLIOGRAPHY_RULE` `rules.rs` 第 548-549 行 |
| `.prefix` | 文献/大纲条目标号前缀 | `OUTLINE_ENTRY_RULE` 第 471-483 行 / `BIBLIOGRAPHY_RULE` 第 519-527 行 |
| `.light` | CSL 较轻样式 | `CSL_LIGHT_RULE` `rules.rs` 第 556-561 行 |
| `.indent` | CSL 缩进样式 | `CSL_INDENT_RULE` `rules.rs` 第 563-571 行 |
| `nav[role="doc-toc"] ol` | 目录列表选择器 | `OUTLINE_RULE` `rules.rs` 第 431-433 行（用 CSS `list-style-type: none`） |
| `section[role="doc-bibliography"] ul` | 参考文献列表 | `BIBLIOGRAPHY_RULE` `rules.rs` 第 537-541 行 |

### 4.3 样式处理对比总结 + 差异对应说明

| 特性 | SVG 后端 | HTML 后端 | 差异根源代码 |
|------|---------|----------|------------|
| 样式载体 | SVG 元素属性（fill/stroke 等） | CSS 属性（内联 style） | SVG: `write_fill/write_stroke` `paint.rs` 32-57 行 / `shape.rs` 128-173 行；HTML: `resolve_inline_styles()` `css/resolve.rs` 11-37 行 |
| 填充 | `fill` 属性 | 不支持渐变/图案填充；纯色仅有限支持 | SVG: Paint→fill 三路全支持 `paint.rs` 40-52 行；HTML: `ToCss for Paint` 仅 `Solid` 支持，`Gradient`/`Tiling` 调用 `w.fail()` 丢弃 `css/encode.rs` 第 386-394 行 |
| 描边 | `stroke-*` 系列 7 个属性 | **不支持**，默认被忽略；需用户显式 `html.frame()` 包裹后由 SVG 后端渲染 | SVG: `shape.rs` 135-172 行 完整描边族；HTML: 无描边 show rule，含描边的几何元素走 else 分支被忽略 |
| 渐变 | `<linearGradient>/<radialGradient>/<pattern>` + url() | **不支持**，CSS 编码直接 fail 并丢弃 | SVG: `push_gradient()` 两级去重 `paint.rs` 66-85 行；HTML: `Paint::Gradient(_) => w.fail("gradient")` `css/encode.rs` 第 390 行，属性被静默忽略 |
| 锥形渐变 | 360 段扇形模拟 | **不支持**（同渐变，CSS 编码 fail） | SVG: `paint.rs` 160-236 行 Conic 分支；HTML: `Paint::Gradient` 统一 fail，不区分锥形/线性/径向 |
| 变换 | `transform` 属性（智能选择 matrix/scale/translate） | CSS `transform` 属性 | SVG: `SvgTransform` `write.rs` 206-241 行 智能格式选择；HTML: 通过 `display` 和文档流自然布局 |
| 颜色空间 | Oklab/Oklch/LinearRGB/HSL/CMYK 全部 | RGB 十六进制 + oklab/oklch/linear-rgb/hsl CSS 函数 | SVG: `SvgDisplay for Color` 7 种格式 `paint.rs` 444-505 行；HTML: `ToCss for Color` 统一转为 `ProcessColor` 再输出，RGB 用 `#hex` 或 `rgb()`、Oklab 用 `oklab()`、Oklch 用 `oklch()`、LinearRgb 用 `color(srgb-linear)`、HSL 用 `hsl()` `css/encode.rs` 第 396-409 行 |
| 去重优化 | 渐变/图案两级去重 | 无内建去重 | SVG: `gradients` + `gradient_refs` 双 Deduplicator；HTML: inline style 每处一份 |
| 字体样式 | 字形层面处理（路径已固化大小、粗细、形状） | **大部分未支持**，仅 `font-variant-caps` 和 `text-decoration` | SVG: 字形路径已固化大小 `text.rs` 67 行预缩放；HTML: `TextElem::fill`（文字颜色）尚未支持 `rules.rs` 第 759 行注释 "temporary workaround until `TextElem::fill` is supported"；`TextElem::size` 仅用于 `HtmlFrame.text_size` 缩放 SVG（`dom.rs` 第 528 行），不输出 CSS `font-size`；唯一输出的字体 CSS 为 `font-variant-caps`（`SMALLCAPS_RULE` `rules.rs` 第 718-724 行）和 `text-decoration`（`UNDERLINE_RULE`/`OVERLINE_RULE`） |
| 可覆盖性 | 难（属性写死在 SVG） | 易（用户 CSS 可覆盖） | SVG: 属性值硬编码；HTML: class 标记 + CSS 级联优先级 |

**差异核心**：SVG 后端的样式目标是**精确还原**——属性写死、两级去重优化文件体积、锥形渐变手动模拟。HTML 后端的样式能力**远比 SVG 有限**：渐变和图案填充在 CSS 编码层直接 fail 丢弃（`css/encode.rs` 第 390-391 行）；描边无对应 show rule；字体颜色/大小/粗细/族系均未输出 CSS 属性（`TextElem::fill` 尚未支持，`rules.rs` 第 759 行有明确注释）。HTML 后端目前只支持纯色填充、`font-variant-caps`、`text-decoration` 等有限样式。**含渐变、描边、几何图形的内容不会被 HTML 后端自动降级为 SVG**——必须用户显式使用 `#html.frame(...)` 包裹（`convert.rs` 第 140-154 行），否则会走 else 分支被静默忽略并发出警告。

---

## 五、资源去重机制对比

### SVG 后端的 Deduplicator

**核心结构**：`Deduplicator<T>`（`crates/typst-svg/src/lib.rs` 第 482-526 行）

工作原理：
1. 通过 `typst_utils::hash128()` 计算 key 的 128 位哈希（`insert_with_val()` 第 512 行）
2. 相同哈希复用同一个 `DedupId`（`kind` 前缀 + 16 进制 hash）
3. 所有资源统一在 `finalize()`（`lib.rs` 第 414-422 行）阶段写入 `<defs>`
4. 使用时通过 `url(#xxx)` 或 `xlink:href="#xxx"` 引用

完整去重类别表：

| 类别 | 前缀字符 | 存储内容 | 使用处引用方式 |
|------|---------|---------|--------------|
| 字形 | `g` | `Option<RenderedGlyph>` (Path / Frame) | `<use xlink:href="#g...">` `text.rs` 109/142 行 |
| 裁剪路径 | `c` | `EcoString` (SVG path) | `clip-path: url(#c...)` `lib.rs` 359 行 |
| 原始渐变 | `f` | `(Gradient, Ratio)` | 无直接引用，被 gradient_ref 引用 |
| 渐变引用 | `r` | `GradientRef` (id + transform + kind) | `fill: url(#r...)` / `stroke: url(#r...)` |
| 锥形子渐变 | `s` | `SVGSubGradient` (起止角度+颜色) | 锥形每段 `fill: url(#s...)` `paint.rs` 229 行 |
| 原始图案 | `t` | `Tiling` (tiling frame) | 被 tiling_ref 引用 |
| 图案引用 | `p` | `TilingRef` (id + transform) | `fill: url(#p...)` / `stroke: url(#p...)` |

### HTML 后端的去重策略

HTML 后端**没有显式去重机制**，原因：
- 文本内容直接输出字符串（已天然去重：相同文字输出相同字符）
- CSS 属性内联在每个元素的 `style` 上，不复用
- 图片通过 base64 Data URL 每处一份完整编码（`IMAGE_RULE` 每次调用都重新生成）
- 可访问的语义元素需要各自独立 DOM 节点，不能共享

---

## 六、数学公式处理差异

### SVG 后端
- 数学公式经过 Typst 布局引擎转换为 `Frame`
- 与普通图形完全一样处理：`FrameItem::Text` → 字形路径，`FrameItem::Shape` → SVG path
- 显示效果与 PDF **100% 一致**
- **不可选择、不可搜索**（纯矢量图形，无 `<text>` 元素）
- 代码位置：通过 `svg()` / `svg_in_html()` 消费已布局 Frame

### HTML 后端
- 使用 **MathML** 输出，不走布局引擎
- `EQUATION_RULE`（`rules.rs` 第 822-841 行）调用 `resolve_equation()` 将公式转为 IR → `convert_math_to_nodes()` 转为 MathML DOM
- 由浏览器 **MathML 引擎渲染**（Firefox/Safari 原生，Chromium 较新版本支持）
- 公式**可选、可复制、可被屏幕阅读器识别**
- 块级公式：`<math display="block">`（第 836 行）
- 行内公式：`<math>`（默认行内，不设 display）
- 数学元素的 display 转换见 `make_inline_level()` / `make_block_level()`（`convert.rs` 第 382-389 行 / 第 448-456 行）

---

## 七、可访问性 (Accessibility) 差异

### SVG 后端
- 纯视觉输出，语义信息弱
- 支持 `data-typst-label` 自定义标签（`lib.rs` 第 350-352 行）
- 链接使用 `<a>` + `href`，可被识别（`render_link()` 第 366-404 行）

### HTML 后端
HTML 后端内置大量 DPUB-ARIA 角色和语义标签：

| 功能 | 实现方式 | 代码位置 |
|------|---------|---------|
| 目录 | `<nav role="doc-toc">` | `OUTLINE_RULE` `rules.rs` 第 456-458 行 |
| 脚注引用 | `<sup role="doc-noteref">` | `FOOTNOTE_RULE` `rules.rs` 第 335-338 行 |
| 脚注回链 | `role="doc-backlink"` | `FOOTNOTE_ENTRY_RULE` 第 413-415 行 |
| 脚注容器 | `<section role="doc-endnotes">` | `FOOTNOTE_CONTAINER_RULE` 第 399-403 行 |
| 参考文献 | `<section role="doc-bibliography">` | `BIBLIOGRAPHY_RULE` 第 544-546 行 |
| 文献引用 | `role="doc-biblioref"` | `CITE_GROUP_RULE` 第 493-497 行 |
| 深层标题 | `<div role="heading" aria-level="n">` (h7+) | `HEADING_RULE` `rules.rs` 第 251-256 行 |
| 表格语义 | `<thead>`/`<tbody>`/`<tfoot>`/`<th>`/`<td>` | `TABLE_RULE` `rules.rs` 第 578-667 行 |
| 图片替代文本 | `<img alt="...">` | `IMAGE_RULE` `rules.rs` 第 781-783 行 |

---

## 八、编码与输出对比

### SVG 编码

**核心文件**：`crates/typst-svg/src/write.rs`

- 使用 `xmlwriter` 库生成标准 XML（`lib.rs` 第 36 行）
- Pretty 格式化：2 空格缩进（`xml_options()` `lib.rs` 第 165-175 行）
- 数字精度优化（`SvgWrite::push_num()` `write.rs` 第 102-118 行）：
  - 9 位小数四舍五入（减少浮点误差）
  - 整数检测：整数省略 `.0`（用 `itoa` 格式化）
  - 浮点数用 `ryu` 最短表示
- Transform 智能格式（`SvgTransform` `write.rs` 第 211-240 行）：
  - 纯缩放 → `scale(sx[, sy])`
  - 纯平移 → `translate(tx[, ty])`
  - 含错切 → `matrix(sx, ky, kx, sy, tx, ty)`（6 参数矩阵）

### HTML 编码

**核心文件**：`crates/typst-html/src/encode.rs`

- 手动字符串拼接（无需 XML 严格格式，HTML 语法更宽松）
- Pretty 格式化：智能缩进 `allows_pretty_inside()` + `wants_pretty_around()`（第 339-365 行）
  - Block 元素 + 非 `<pre>` 允许缩进
  - MathML 非 token 元素允许缩进
  - 连续子元素判断是否换行
- 字符转义 `write_escape()`（第 368-382 行）：
  - `&` → `&amp;`，`<` → `&lt;`，`>` → `&gt;`，`"` → `&quot;`，`'` → `&apos;`
  - 其他非 W3C text char → `&#xNNNN;` 数字引用
- 特殊元素处理（第 142-167 行）：
  - **Void 元素**（`<img>`, `<br>`, `<hr>` 等）：无子元素，不写闭合标签
  - **Raw 元素**（`<script>`, `<style>`）：内容不转义，禁止出现 `</tag>` 字符串
  - **Escapable Raw**（`<textarea>`, `<title>`）：内容可转义
  - `<pre>`/`<textarea>` 首字符为换行时保留（HTML 规范要求丢弃首个换行）

---

## 九、核心差异总结

### 设计哲学

- **SVG 后端**：*所见即所得*。忠实还原 Typst 布局引擎的每一个像素，输出是自包含的矢量图像。输入是布局完成的 `Frame`，所有内容都转为 `<path>` 或嵌入图像，保证跨设备完全一致。
- **HTML 后端**：*语义优先 + 用户显式回退*。保留文档的结构和语义，利用浏览器的布局和渲染能力，兼顾可访问性和可扩展性。输入是语义 `Content` 树，有 show rule 的元素映射 HTML 标签；无 show rule 的元素发出警告后被**直接忽略**；仅当用户在源码中显式写 `#html.frame(...)` 时（`lib.rs` 第 126-141 行 `FrameElem` 定义），才会调 SVG 后端嵌入内联 SVG——**没有任何自动回退机制**。

### 三类导出差异的代码对应关系总结

| 差异维度 | SVG 后端的选择 | 对应关键代码 | HTML 后端的选择 | 对应关键代码 |
|---------|--------------|------------|--------------|------------|
| **文本处理** | 字形转路径/图像，嵌入文档，保证一致性 | `crates/typst-svg/src/text.rs` 第 29-161 行 (`render_text`+`render_glyph`)；`RenderedGlyph` 第 16-23 行 | 直接输出文本节点，依赖浏览器字体，额外处理空白折叠 | `crates/typst-html/src/convert.rs` 第 252-334 行 (`handle_text`)；第 591-683 行 (`protect_spaces`+`pre_wrap`) |
| **图形处理** | 一切几何统一为 `<path>`，绝对坐标精确绘制 | `crates/typst-svg/src/shape.rs` 第 13-48 行 (`render_shape`)；第 178-201 行 (`convert_geometry_to_path`) | 三条独立路径无自动回退：show rule 转语义标签 / 用户显式 `#html.frame()` 嵌入内联 SVG / 其余元素被忽略 | `crates/typst-html/src/rules.rs` 第 573-820 行 (TABLE/IMAGE/DIVIDER rules)；`crates/typst-html/src/convert.rs` 第 140-154 行 (`FrameElem` 分支，需用户显式调用)；第 155-161 行 (未注册元素→警告跳过) |
| **样式处理** | 属性式样式 + 两级渐变去重 + 锥形手动模拟，追求精确和体积 | `crates/typst-svg/src/paint.rs` 第 32-331 行 (`write_fill`+`push_gradient`+`write_gradients`)；`crates/typst-svg/src/shape.rs` 第 128-173 行 (`write_stroke`) | 样式能力有限：渐变/图案 fail 丢弃，字体样式大部分未支持，仅 `font-variant-caps`/`text-decoration` | `crates/typst-html/src/css/encode.rs` 第 386-394 行 (`Paint::Gradient/Tiling → w.fail()`)；`crates/typst-html/src/rules.rs` 第 759 行 (`TextElem::fill` 尚未支持注释)；`crates/typst-html/src/css/resolve.rs` 第 11-37 行 (`resolve_inline_styles`) |

### 适用场景

| 场景 | 推荐后端 | 原因 |
|------|---------|------|
| 打印 / 印刷 / 出版 | SVG / PDF | 精确控制，所见即所得，不依赖系统字体 |
| 网页阅读 / 博客 | HTML | 响应式、可访问、可搜索，体积小 |
| 技术文档 / 知识库 | HTML | 语义化、可锚点跳转、屏幕阅读器支持 |
| 数据可视化 / 图表 | SVG（或 HTML + 用户显式 `#html.frame()` 包裹内容） | 精确的图表像素级布局，需用户主动调用 `html.frame` 才会嵌入 SVG |
| 电子书 / 无障碍文档 | HTML | DPUB-ARIA 角色、屏幕阅读器友好 |
| 跨平台一致性展示 | SVG | 字形内嵌，无需系统字体，所有设备相同 |

### 关键代码位置（相对路径 + 行号）

- **SVG 后端入口**：`crates/typst-svg/src/lib.rs` 第 313-327 行，函数 `SVGRenderer::render_frame()`
- **HTML 后端入口**：`crates/typst-html/src/convert.rs` 第 60-90 行（`convert_to_nodes()`）+ `crates/typst-html/src/rules.rs` 第 39-92 行（`register()`）
- **两者桥接**：`crates/typst-html/src/lib.rs` 第 137-141 行（`FrameElem` 定义）+ `crates/typst-svg/src/lib.rs` 第 82-123 行（`svg_in_html()` 函数）
