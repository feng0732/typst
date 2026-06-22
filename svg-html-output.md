# SVG 与 HTML 导出后端差异分析

本文档从文本处理、图形渲染、样式管理三个维度，深入分析 Typst 的 SVG 与 HTML 两类导出后端的实现机制与核心差异。

## 一、整体架构对比

| 维度 | SVG 后端 (typst-svg) | HTML 后端 (typst-html) |
|------|---------------------|----------------------|
| **输入** | 布局完成的 `Frame` / `Page` | 语义内容树 `Content` + 样式链 |
| **核心思路** | 将已布局结果逐像素翻译为 SVG 元素 | 通过 show rule 将语义元素映射为 HTML 标签 |
| **布局责任** | Typst 引擎完成全部布局 | 由浏览器 CSS 布局引擎负责 |
| **输出粒度** | 单个 SVG 文档（单页/合并多页） | 完整 HTML 文档（含 DOM 树） |
| **坐标系统** | 绝对坐标（pt 单位），精确到每个元素 | 文档流 + CSS 盒模型 |
| **关键文件** | [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs) | [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs) + [rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs) |

### 入口函数对比

**SVG 后端** 直接接收布局完成的页面/帧：
- [`svg()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L32-L43) — 单页导出
- [`svg_merged()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L128-L156) — 多页合并
- [`svg_in_html()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L82-L123) — 嵌入 HTML 的内联 SVG

**HTML 后端** 接收内容并通过 show rule 系统转换：
- [`html_document()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/document.rs) — 完整 HTML 文档
- [`convert_to_nodes()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L60-L90) — 内容到 HTML 节点的核心转换

---

## 二、文本处理差异

### 2.1 SVG 后端：字形级精确渲染

SVG 后端在 [text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/text.rs) 中实现，核心特征是**将文本转化为矢量路径或图像**，完全不依赖系统字体。

#### 渲染流程

```
TextItem → 遍历 glyphs → 分类渲染 → 去重复用
                              ↓
                   ┌──────────┴──────────┐
                   ↓                     ↓
            轮廓字形 (Path)         图像字形 (Frame)
         （普通文字、矢量字形）   （彩色 emoji、COLR/CPAL 字体）
```

#### 两种字形表示

**1. 路径字形 (`RenderedGlyph::Path`)**
- 用 `ttf_parser` 从字体文件提取字形轮廓
- 通过 [`SvgPathBuilder`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/path.rs) 转换为 SVG path 数据
- 预缩放处理：`scale = text_size / units_per_em`，避免使用时缩放影响描边粗细

**2. 图像字形 (`RenderedGlyph::Frame`)**
- 适用于 COLR/CPAL、SVG-in-OT、位图等彩色字形
- 字形尺寸较大，使用时再缩放（减少内存占用）

#### 关键实现细节

- **Y 轴翻转**：字体使用 Y-Up 坐标系，SVG 绘制前需翻转 Y 轴
  ```rust
  let state = state.pre_concat(Transform::scale(Ratio::one(), -Ratio::one()));
  ```
  见 [text.rs#L38](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/text.rs#L38)

- **字形去重**：通过 `Deduplicator` 机制，相同字形只定义一次，用 `<use>` 引用
  - 路径字形 key：`(font, glyph_id, scale)`
  - 图像字形 key：`(font, glyph_id)`
  - 定义在 `<defs>` 的 `<symbol>` 中，使用时 `<use xlink:href="#g...">`

- **文本样式**：填充和描边直接作用于 `<use>` 元素
  - 填充支持纯色、渐变、图案
  - 描边支持厚度、端点、连接、虚线等
  - 见 [`render_path_glyph()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/text.rs#L115-L161)

### 2.2 HTML 后端：语义化文本节点

HTML 后端在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs) 的 [`handle_text()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L252-L334) 中处理文本，核心特征是**保留文本语义，依赖浏览器字体渲染**。

#### 文本转换流程

```
TextElem → 应用大小写变换 → 特殊字符处理 → 生成 HtmlNode::Text
                ↓
        ┌───────┴───────┐
        ↓               ↓
    普通字符        空白字符处理
    直接输出        (空格折叠防护)
```

#### 空白字符处理（核心复杂度）

由于 HTML 会自动折叠连续空白，HTML 后端设计了精细的防护机制：

**1. 连续空白保护**
- 连续空格/制表符 → 包装在 `<span style="white-space: pre-wrap">` 中
- 见 [`pre_wrap()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L675-L683)

**2. 单空格保护**
- 单空格在普通文本之间无需保护
- 但相邻元素边界处的单空格需要保护（如 `<span>` 旁边的空格）
- 通过 [`Protector` 状态机](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L598-L665) 后处理实现

**3. 换行处理**
- 普通模式下换行 → `<br>` 元素
- `<pre>` 等预格式化元素中 → 直接保留换行字符

#### 智能引号

HTML 后端内置 `SmartQuoter` 状态机，根据上下文自动将直引号转为弯引号，支持多语言。

### 2.3 文本处理对比总结

| 特性 | SVG 后端 | HTML 后端 |
|------|---------|----------|
| 文本表示 | 字形路径 / 图像 | 纯文本节点 |
| 字体依赖 | 嵌入字形轮廓，无需系统字体 | 依赖浏览器/系统字体 |
| 排版精度 | 100% 还原 Typst 布局 | 浏览器排版，可能有差异 |
| 文本可复制 | 取决于 SVG 查看器（通常不可选） | 原生可选可复制 |
| 文本可搜索 | 取决于 SVG 查看器 | 原生可搜索 |
| 空白处理 | 精确位置，无折叠问题 | 需要专门防护空白折叠 |
| 彩色 emoji | 直接渲染字形图像 | 依赖浏览器字体支持 |
| 文件大小 | 大（每个字形都是路径） | 小（纯文本） |

---

## 三、图形处理差异

### 3.1 SVG 后端：原生矢量图形

SVG 后端在 [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/shape.rs) 中处理图形，核心是**统一转换为 SVG path**。

#### 几何图形统一为 Path

所有几何形状（直线、矩形、曲线）最终都生成 `d` 属性的路径数据：

- **直线** `Geometry::Line` → `M x1 y1 L x2 y2`
- **矩形** `Geometry::Rect` → `M 0 0 v h h -v Z`（使用相对坐标优化）
- **曲线** `Geometry::Curve` → 逐段贝塞尔曲线 `C x1 y1 x2 y2 x y`

见 [`convert_geometry_to_path()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/shape.rs#L178-L188)

#### SvgPathBuilder 优化策略

[`SvgPathBuilder`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/path.rs) 使用相对坐标减小文件体积：

| 命令 | 用途 | 优化 |
|------|------|------|
| `m` | 相对移动 | 零偏移时省略 |
| `l` | 相对直线 | 水平用 `h`，垂直用 `v` |
| `c` | 三次贝塞尔 | — |
| `a` | 弧线 | — |
| `Z` | 闭合路径 | — |

#### 组与裁剪

- **组 (`<g>`)**：承载变换矩阵、裁剪路径
- **裁剪路径 (`clip-path`)**：去重存储在 `<defs>` 中，通过 `url(#c...)` 引用
- 硬帧（Hard Frame）会重置坐标系，软帧（Soft Frame）叠加变换

见 [`render_group()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L331-L363)

### 3.2 HTML 后端：语义元素 + SVG 嵌入

HTML 后端的图形处理采取**混合策略**：

#### 1. 语义化图形元素

部分 Typst 元素直接映射为 HTML 语义标签：

| Typst 元素 | HTML 标签 | 所在规则 |
|-----------|----------|---------|
| `image` | `<img>` | [IMAGE_RULE](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs#L773-L820) |
| `table` | `<table>` / `<thead>` / `<tbody>` / `<tr>` / `<td>` / `<th>` | [TABLE_RULE](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs#L573-L687) |
| `divider` / `line` | `<hr>` | [DIVIDER_RULE](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs#L210-L212) |

图片以 base64 Data URL 内联，确保单文件可分发。

#### 2. FrameElem：精确布局内容的 SVG 嵌入

对于需要精确布局的内容（如图表、绘图），HTML 后端通过 `FrameElem` 绕过 HTML 布局：

```rust
// 用 Paged 目标进行布局，得到精确的 Frame
let frame = (converter.engine.library.routines.layout_frame)(
    converter.engine, &elem.body, locator, styles, region
)?;
// 包装为 HtmlFrame，后续编码时转为内联 SVG
let node = HtmlFrame::new(frame, styles, elem.span()).into();
```

见 [convert.rs#L140-L154](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L140-L154)

`HtmlFrame` 在编码阶段调用 `typst_svg::svg_in_html()` 生成内联 SVG，见 [encode.rs#L391-L401](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/encode.rs#L391-L401)。

### 3.3 图形处理对比总结

| 特性 | SVG 后端 | HTML 后端 |
|------|---------|----------|
| 图形表示 | 原生 SVG 矢量路径 | 混合：HTML 语义标签 + 内联 SVG |
| 几何精度 | 精确 pt 级坐标 | 浏览器像素级渲染 |
| 表格 | 用路径绘制线条和文字 | 原生 `<table>` 元素，语义化 |
| 图片 | `<image>` 嵌入 | `<img>` 标签 |
| 矩形/圆形 | 统一转为 `<path>` | 需通过 FrameElem 嵌入 SVG |
| 裁剪遮罩 | 原生 `clip-path` 支持 | 依赖 CSS clip-path 或 SVG 嵌入 |
| 布局控制 | 完全由 Typst 控制 | 浏览器盒模型 + 文档流 |
| 可访问性 | 弱（纯图形） | 强（语义标签 + ARIA 角色） |

---

## 四、样式处理差异

### 4.1 SVG 后端：属性式样式

SVG 后端在 [paint.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/paint.rs) 中管理样式，核心是**将样式映射为 SVG 元素属性**。

#### 填充 (fill)

| Paint 类型 | SVG 表示 | 实现方式 |
|-----------|---------|---------|
| 纯色 | `fill="#rrggbb"` / `fill="color(...)"` | 直接颜色值 |
| 渐变 | `fill="url(#f...)"` | `<linearGradient>` / `<radialGradient>` / `<pattern>`（锥形） |
| 图案 | `fill="url(#t...)"` | `<pattern>` 内含子帧 |

#### 描边 (stroke)

| 属性 | SVG 属性 |
|------|---------|
| 颜色/渐变/图案 | `stroke` |
| 粗细 | `stroke-width` |
| 端点 | `stroke-linecap` (butt/round/square) |
| 连接 | `stroke-linejoin` (miter/round/bevel) |
| 斜接限制 | `stroke-miterlimit` |
| 虚线 | `stroke-dasharray` + `stroke-dashoffset` |

见 [`write_stroke()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/shape.rs#L128-L173)

#### 颜色空间支持

SVG 后端支持丰富的颜色空间，见 [`SvgDisplay for Color`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/paint.rs#L444-L505)：

- RGB / Luma → 十六进制 `#rrggbb`
- CMYK → 十六进制（转换后）
- Linear RGB → `color(srgb-linear r g b)`
- Oklab → `oklab(L% a b)`
- Oklch → `oklch(L% c h)`
- HSL → `hsl(h s% l%)`
- HSV → 十六进制（转换后）

#### 渐变的两级去重优化

SVG 后端对渐变做了精巧的去重设计：

```
Level 1: 原始渐变 (gradients)
  - Key: (gradient, aspect_ratio)
  - 存储在 <defs> 中，无 transform
  - 所有相同形状和颜色的渐变共用

Level 2: 渐变引用 (gradient_refs)
  - Key: (gradient_id, transform)
  - 通过 href 引用 Level 1 的渐变
  - 附加 gradientTransform
  - 节省空间：渐变定义大，transform 小
```

见 [`push_gradient()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/paint.rs#L66-L85)

#### 锥形渐变的特殊处理

SVG 没有原生锥形渐变，SVG 后端用**分段模拟**：
- 将圆分为 360 个扇形段
- 每个扇形用线性渐变填充
- 包装在 `<pattern>` 中

见 [`write_gradients()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/paint.rs#L160-L236) 中 `Gradient::Conic` 分支。

### 4.2 HTML 后端：CSS 属性式样式

HTML 后端的样式通过 `css::Properties` 管理，最终转换为内联 `style` 属性。

#### CSS 管理机制

1. **`css::Properties`**：CSS 属性键值对集合
2. **`resolve_inline_styles()`**：遍历 DOM 树，将元素的 `css` 字段合并到 `style` 属性
   - 见 [css/resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/css/resolve.rs)
3. **内联输出**：`style="prop1: value1; prop2: value2"`

#### 显示模式 (display) 控制

HTML 后端精细管理元素的 `display` 属性：

- **块级提升**：`make_block_level()` 将合适的元素提升为 `display: block`
- **行内降级**：`make_inline_level()` 确保段落内元素为行内级
- **数学模式**：区分 `inline math` 和 `block math`

见 [convert.rs#L381-L510](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L381-L510)

#### 类名 (class) 策略

部分元素通过 `class` 而非内联样式标记，以便用户自定义 CSS：
- 目录：`nav[role="doc-toc"] ol`
- 参考文献：`section[role="doc-bibliography"]`
- 悬挂缩进：`.hanging-indent`
- 文献前缀：`.prefix`

### 4.3 样式处理对比总结

| 特性 | SVG 后端 | HTML 后端 |
|------|---------|----------|
| 样式载体 | SVG 元素属性 | CSS 属性（内联 style） |
| 填充 | `fill` 属性 | `background-color` / `background-image` |
| 描边 | `stroke*` 系列属性 | `border*` / `outline*` |
| 渐变 | SVG 渐变元素 + url() 引用 | CSS `linear-gradient()` 等 |
| 图案填充 | SVG pattern 元素 | CSS `background-image` + `repeat` |
| 变换 | `transform` 属性 (matrix/scale/translate) | CSS `transform` 属性 |
| 颜色空间 | 丰富（oklab/oklch/linearrgb 等） | 依赖浏览器 CSS 支持 |
| 去重优化 | 渐变、图案两级去重 | 无内建去重 |
| 字体样式 | 字形层面处理（无 CSS 字体属性） | CSS `font-*` 属性 |
| 可覆盖性 | 难（属性直接写死） | 易（用户 CSS 可覆盖） |

---

## 五、资源去重机制对比

### SVG 后端的 Deduplicator

SVG 后端有完善的去重系统，核心是 [`Deduplicator<T>`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L482-L526) 结构：

| 类别 | 前缀 | 存储内容 |
|------|------|---------|
| 字形 | `g` | RenderedGlyph (Path / Frame) |
| 裁剪路径 | `c` | SVG path 字符串 |
| 原始渐变 | `f` | (Gradient, Ratio) |
| 渐变引用 | `r` | GradientRef (id + transform) |
| 锥形子渐变 | `s` | SVGSubGradient |
| 原始图案 | `t` | Tiling |
| 图案引用 | `p` | TilingRef (id + transform) |

工作原理：
1. 通过 `typst_utils::hash128()` 计算 key 的 128 位哈希
2. 相同哈希复用同一个 ID
3. 所有资源统一在 `finalize()` 阶段写入 `<defs>`
4. 使用时通过 `url(#xxx)` 或 `xlink:href="#xxx"` 引用

### HTML 后端的去重策略

HTML 后端没有显式的去重机制，原因：
- 文本内容直接输出，无需去重
- CSS 属性内联在元素上，不复用
- 图片通过 base64 内联（每处一份）
- 可访问的语义元素需要各自独立存在

---

## 六、数学公式处理差异

### SVG 后端
- 数学公式经过布局引擎转换为 Frame
- 与普通图形一样，转为路径和图像
- 显示效果与 PDF 完全一致
- 不可选择、不可搜索（纯矢量图形）

### HTML 后端
- 使用 **MathML** 输出，见 [`EQUATION_RULE`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs#L822-L841)
- 由浏览器 MathML 引擎渲染
- 公式可选、可复制、可被屏幕阅读器识别
- 块级公式：`<math display="block">`
- 行内公式：`<math>`（默认行内）

---

## 七、可访问性 (Accessibility) 差异

### SVG 后端
- 纯视觉输出，语义信息弱
- 支持 `data-typst-label` 自定义标签
- 链接使用 `<a>` 元素，可被识别

### HTML 后端
- 语义化标签丰富（`<article>`, `<nav>`, `<section>`, 等）
- ARIA 角色：`doc-toc`, `doc-bibliography`, `doc-noteref`, `doc-backlink`, `doc-endnotes`
- 标题层级：`<h1>` ~ `<h6>`，超出用 `role="heading" aria-level`
- 表格语义：`<thead>`, `<tbody>`, `<th scope>`
- 图片替代文本：`alt` 属性
- 参考文献、脚注等都有对应 DPUB-ARIA 角色

---

## 八、编码与输出对比

### SVG 编码
- 使用 `xmlwriter` 库生成 XML
- 支持 pretty 格式化（2 空格缩进）
- 数字精度优化：9 位小数四舍五入，整数省略 `.0`
- 见 [write.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/write.rs)

### HTML 编码
- 手动字符串拼接（无需 XML 严格格式）
- 支持 pretty 格式化（智能缩进）
- 字符转义：`&`, `<`, `>`, `"`, `'` → 命名实体；其他 → `&#x...;`
- 特殊元素处理：void 元素、raw 元素（`<script>`, `<style>`）、可转义 raw 元素
- 见 [encode.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/encode.rs)

---

## 九、核心差异总结

### 设计哲学

- **SVG 后端**：*所见即所得*。忠实还原 Typst 布局引擎的每一个像素，输出是自包含的矢量图像。
- **HTML 后端**：*语义优先*。保留文档的结构和语义，利用浏览器的布局和渲染能力，兼顾可访问性和可扩展性。

### 适用场景

| 场景 | 推荐后端 | 原因 |
|------|---------|------|
| 打印 / 印刷 | SVG / PDF | 精确控制，所见即所得 |
| 网页阅读 | HTML | 响应式、可访问、可搜索 |
| 技术文档 | HTML | 语义化、可锚点跳转 |
| 数据可视化 | SVG（或 FrameElem） | 精确的图表布局 |
| 电子书 / 无障碍 | HTML | 屏幕阅读器支持 |
| 跨平台一致性 | SVG | 不依赖系统字体 |

### 关键代码参考

**SVG 后端入口**：[`SVGRenderer::render_frame()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L313-L327)

**HTML 后端入口**：[`convert_to_nodes()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/convert.rs#L60-L90) + [`register()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/rules.rs#L39-L92)

**两者桥接**：[`FrameElem`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-html/src/lib.rs#L137-L141) + [`svg_in_html()`](file:///d:/fz/0601-2/solo-dogfeeding/code/134-typst/crates/typst-svg/src/lib.rs#L82-L123)
