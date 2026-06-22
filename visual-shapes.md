# Typst 图形与路径：基础图形、路径数据与渲染输出的协作

## 全局数据流总览

```
用户代码 (Typst markup)
    │
    ▼  编译入口 typst::compile::<T>()  [lib.rs:74-82]
    │
    │  T 的选择决定了 Output trait 的实现：
    │  ├─ T = PagedDocument  → Target::Paged （PNG/SVG/PDF 共用）
    │  └─ T = HtmlDocument   → Target::Html   （HTML 输出）
    │
    ▼  元素定义 (typst-library/visualize/)
       LineElem / RectElem / SquareElem / EllipseElem / CircleElem
       PolygonElem / CurveElem
    │
    ▼  Show Rule 注册（按 Target 区分）
    │
    │  Target::Paged  [typst-layout/rules.rs:L96-L103]:
    │    LINE_RULE → layout_line
    │    RECT_RULE → layout_rect
    │    ...所有 7 种图形元素都有 Show Rule
    │
    │  Target::Html  [typst-html/rules.rs:L82-L83]:
    │    IMAGE_RULE （仅图片有 Show Rule）
    │    其他图形元素无 Show Rule → 在 convert.rs 中被忽略并警告
    │
    ▼  布局层
    │
    │  Target::Paged 路径 (typst-layout/shapes.rs):
    │    layout_line / layout_rect / layout_curve / ...
    │    → 产出 Frame，包含 FrameItem::Shape(Shape, Span)
    │
    │  Target::Html 路径 (typst-html/convert.rs):
    │    ├─ 普通图形元素（rect/line/curve 等）→ 无 Show Rule
    │    │   → [convert.rs:L155-L160] 警告并忽略
    │    │
    │    └─ html.frame(body)  [convert.rs:L140-L154]:
    │        ├─ 临时切换 Target::Paged 重新布局
    │        │   let style = TargetElem::target.set(Target::Paged).wrap();
    │        │   let frame = layout_frame(engine, &elem.body, locator, styles.chain(&style), ...);
    │        ├─ 包装为 HtmlFrame { inner: frame, text_size, css, ... }
    │        └─ 作为 HtmlNode::Frame 插入 DOM 树
    │
    ▼  Output 产物
    │
    ├─ PagedDocument  [typst-layout/document.rs:L63-L79]
    │   ├─ pages: Vec<Page { frame, bleed, fill, ... }>
    │   └─ 后处理导出：
    │       ├─ PDF:  typst_pdf::pdf(&document, ...)      → Vec<u8>
    │       ├─ PNG:  typst_render::render(&page, ...)    → sk::Pixmap （逐页）
    │       └─ SVG:  typst_svg::svg(&page, ...)          → String （逐页）
    │
    └─ HtmlDocument  [typst-html/dom.rs:L81-L97]
        └─ 编码 (typst-html/encode.rs):
            ├─ 普通 HTML 元素 → <div>/<p>/<span>/...
            └─ HtmlNode::Frame(frame) → [encode.rs:L391-L402]
                typst_svg::svg_in_html(
                    &frame.inner, frame.text_size, pretty, id, styles, anchors, link_resolver
                ) → 内联 <svg> 字符串
```

---

## 一、基础图形元素定义 (typst-library/visualize/)

所有图形元素都在 [mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/mod.rs) 的 `define()` 函数中注册到全局作用域。

### 1.1 元素一览

| 元素 | 文件 | 核心字段 |
|------|------|----------|
| `LineElem` | [line.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/line.rs) | `start`, `end`, `length`, `angle`, `stroke` |
| `RectElem` | [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/shape.rs#L7-L128) | `width`, `height`, `fill`, `stroke`, `radius`, `inset`, `outset`, `body` |
| `SquareElem` | [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/shape.rs#L130-L205) | 继承 rect 语义，`size` 与 `width`/`height` 互斥 |
| `EllipseElem` | [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/shape.rs#L207-L255) | `width`, `height`, `fill`, `stroke`, `inset`, `outset`, `body` |
| `CircleElem` | [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/shape.rs#L257-L330) | `radius` 与 `width`/`height` 互斥 |
| `PolygonElem` | [polygon.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/polygon.rs) | `vertices` (变长参数), `fill`, `stroke`, `fill_rule` |
| `CurveElem` | [curve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/curve.rs#L42-L93) | `components` (变长 CurveComponent), `fill`, `stroke`, `fill_rule` |

### 1.2 核心数据结构三件套

这三个结构是连接"元素定义 → 布局 → 渲染"的核心桥梁，定义在 [shape.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/shape.rs#L332-L432)：

```rust
// 几何形状 = 几何体 + 填充 + 描边 + 填充规则
pub struct Shape {
    pub geometry: Geometry,
    pub fill: Option<Paint>,
    pub fill_rule: FillRule,
    pub stroke: Option<FixedStroke>,
}

// 几何体有三种变体
pub enum Geometry {
    Line(Point),         // 直线（终点偏移）
    Rect(Size),          // 矩形（原点在左上角）
    Curve(Curve),        // 贝塞尔曲线路径
}

// 填充规则
pub enum FillRule {
    NonZero,   // 默认：非零环绕数
    EvenOdd,   // 奇偶规则
}
```

`Geometry` 提供了便捷的构造方法：
- `Geometry::Line(delta).stroked(stroke)` → 只描边
- `Geometry::Rect(size).filled(fill)` → 只填充
- `geometry.filled_and_stroked(fill, stroke)` → 填充+描边

### 1.3 Curve 路径数据结构

定义在 [curve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/curve.rs#L382-L393)：

```rust
// 一条曲线路径 = 有序的路径元素列表
pub struct Curve(pub Vec<CurveItem>);

// 路径元素（注意：没有 Quadratic，全部升阶为 Cubic）
pub enum CurveItem {
    Move(Point),               // 移动画笔
    Line(Point),               // 直线段
    Cubic(Point, Point, Point), // 三次贝塞尔 (控制点1, 控制点2, 终点)
    Close,                      // 闭合路径
}
```

关键设计：
- **统一为 Cubic**：用户层有 `CurveQuad`（二次贝塞尔），但在 `CurveBuilder` 中通过 `control_q2c()` 升阶为三次贝塞尔，最终只存储 `CurveItem::Cubic`。
- **内置构造器**：`Curve::rect(size)` 和 `Curve::ellipse(size)` 提供了矩形和椭圆的快速构造。
- **椭圆近似**：使用 `0.551784` 魔数（[StackOverflow 参考](https://stackoverflow.com/a/2007782)）以 4 段三次贝塞尔逼近椭圆。

### 1.4 CurveComponent 与 CurveElem

用户在 Typst 代码中写的 `curve(move(..), line(..), cubic(..))` 被解析为 [CurveComponent](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/visualize/curve.rs#L114-L121) 枚举：

```rust
pub enum CurveComponent {
    Move(Packed<CurveMove>),    // 起始点
    Line(Packed<CurveLine>),    // 直线段
    Quad(Packed<CurveQuad>),    // 二次贝塞尔
    Cubic(Packed<CurveCubic>),  // 三次贝塞尔
    Close(Packed<CurveClose>),  // 闭合
}
```

各子元素支持 `relative: bool` 参数控制绝对/相对坐标，以及 `auto`/`none` 控制点自动推导。

---

## 二、布局层：元素 → Frame (typst-layout/shapes.rs)

### 2.1 Show Rule 注册

在 [rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/rules.rs#L96-L103) 中，每个图形元素被注册为 Show Rule：

```rust
rules.register(Paged, LINE_RULE);     // → layout_line
rules.register(Paged, RECT_RULE);     // → layout_rect
rules.register(Paged, SQUARE_RULE);   // → layout_square
rules.register(Paged, ELLIPSE_RULE);  // → layout_ellipse
rules.register(Paged, CIRCLE_RULE);   // → layout_circle
rules.register(Paged, POLYGON_RULE);  // → layout_polygon
rules.register(Paged, CURVE_RULE);    // → layout_curve
```

每个 Rule 将元素包装为 `BlockElem::single_layouter(elem, layout_xxx)`，由排版引擎在合适时机调用布局函数。

### 2.2 布局函数详解

所有布局函数签名统一为：

```rust
fn layout_xxx(
    elem: &Packed<XxxElem>,
    engine: &mut Engine,
    locator: Locator,
    styles: StyleChain,
    region: Region,          // 可用区域
) -> SourceResult<Frame>     // 产出帧
```

#### Line 布局 ([layout_line](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L21-L53))

```
LineElem
  ├─ 解析 start (绝对坐标) 和 delta (end - start 或 angle+length)
  ├─ 计算 size = max(start, start+delta)
  ├─ 构造 Shape = Geometry::Line(delta).stroked(stroke)
  └─ Frame.push(start, FrameItem::Shape(shape, span))
```

#### Curve 布局 ([layout_curve](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L56-L141))

最复杂的布局函数，使用 `CurveBuilder` 逐步构建：

```
CurveElem
  ├─ CurveBuilder::new(region, styles)
  ├─ 遍历 components:
  │   ├─ CurveComponent::Move  → builder.move_(point)
  │   ├─ CurveComponent::Line  → builder.line(point)
  │   ├─ CurveComponent::Quad  → control_c2q() 升阶 → builder.quad() → 内部调 builder.cubic()
  │   ├─ CurveComponent::Cubic → 解析 auto/none 控制点 → builder.cubic(c1, c2, end)
  │   └─ CurveComponent::Close → builder.close(mode)
  │       ├─ CloseMode::Smooth → 自动补一段三次贝塞尔闭合
  │       └─ CloseMode::Straight → 直线闭合
  ├─ builder.finish() → (Curve, Size)
  ├─ 解析 fill/stroke/fill_rule
  └─ Frame.push(Point::zero(), FrameItem::Shape(shape, span))
```

**CurveBuilder 关键状态**（[L144-L166](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L144-L166)）：

| 字段 | 作用 |
|------|------|
| `start_point` | 当前子路径的起点（close 时回到此点） |
| `start_control_into` | 首段起点的控制点镜像（用于 Smooth close） |
| `last_point` | 画笔当前位置 |
| `last_control_from` | 上一段终点的控制点镜像（用于 auto 控制点推导） |
| `is_started` | 是否已执行过 move |
| `is_empty` | 当前子路径是否为空 |

**控制点推导公式**：
- `mirror_c(p, c) = 2*p - c`：关于当前点镜像控制点
- `control_q2c(p, c) = (p + 2*c) / 3`：二次→三次升阶
- `control_c2q(p, c) = 1.5*c - 0.5*p`：三次→二次降阶（用于 Quad 的 auto 控制点）

#### Polygon 布局 ([layout_polygon](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L307-L358))

```
PolygonElem
  ├─ 将 vertices 解析为绝对 Point 列表
  ├─ size = 所有点的 max
  ├─ Curve::new() → move(首点) → line(后续点) → close()
  └─ Frame.push(Point::zero(), FrameItem::Shape(shape, span))
```

#### Rect/Square/Ellipse/Circle 布局

四者统一汇入 [layout_shape](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L487-L596) 函数，通过 `ShapeKind` 枚举区分：

```
ShapeKind::Rect / Square / Ellipse / Circle
  │
  ├─ 有 body 内容时：
  │   ├─ 计算 inset（圆/椭圆额外增加 0.5 - √2/4 的内边距）
  │   ├─ 若 is_quadratic（正方/圆），先测量子内容取 max 边长
  │   ├─ 布局子内容 → frame
  │   └─ 应用 inset padding
  │
  ├─ 无 body 时：
  │   ├─ 默认尺寸 45pt × 30pt（受限于 region）
  │   └─ 若 is_quadratic，取 min 边长
  │
  └─ 绘制填充/描边：
      ├─ is_round（圆/椭圆）:
      │   Geometry::Curve(Curve::ellipse(size))
      │   → 单个 Shape{fill, stroke: stroke.left}
      │
      └─ 否则（矩形/正方）:
          styled_rect(size, radius, fill, stroke)
          ├─ 简单矩形（无圆角 + 统一描边）→ Geometry::Rect(size)
          └─ 分段矩形（有圆角或不同边描边）→ 分段 fill + stroke
```

### 2.3 分段矩形渲染 (styled_rect)

[styled_rect](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L686-L697) 处理带圆角和/或不同边描边的矩形：

```
styled_rect(size, radius, fill, stroke)
  │
  ├─ 简单矩形（无圆角 + 统一描边）:
  │   → 单个 Shape { Geometry::Rect(size), fill, stroke }
  │
  └─ 分段矩形:
      ├─ 1. 填充层: Curve 描绘内轮廓 → Shape { fill, stroke: None }
      ├─ 2. 描边层: 按段分组
      │   ├─ 实线段 → fill_segment()（用填充区域模拟描边，效果更好）
      │   └─ 虚线段 → stroke_segment()（沿中轴线描边）
      └─ 返回 Vec<Shape>（填充在下，描边在上）
```

**ControlPoints** 辅助结构（[L1069-L1076](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L1069-L1076)）精确计算每个角的内/中/外三层控制点，处理圆角与描边宽度的交互。

---

## 三、Output trait 与顶层编译入口

### 3.1 Output trait 与 Target 枚举

定义在 [target.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/foundations/target.rs)：

```rust
/// 编译输出，与 Target 变体一一对应
pub trait Output: Any {
    fn target() -> Target where Self: Sized;
    fn create(engine: &mut Engine, content: &Content, styles: StyleChain) -> SourceResult<Self>;
    fn introspector(&self) -> &dyn Introspector;
}

#[derive(Debug, Default, Copy, Clone, Eq, PartialEq, Hash, Cast)]
pub enum Target {
    #[default]
    Paged,   // 分页完整排版（PNG/SVG/PDF 共用）
    Html,    // HTML 导出
    Bundle,  // 多文件捆绑导出
}
```

**关键事实：Output trait 只有两个核心实现**

| 实现 | Target | 创建函数 | 导出用途 |
|------|--------|---------|---------|
| `PagedDocument` | `Target::Paged` | `typst_layout::layout_document()` | PNG / SVG / PDF（共用该 Output，后处理不同） |
| `HtmlDocument` | `Target::Html` | `typst_html::html_document()` | HTML 导出 |
| `Bundle` | `Target::Bundle` | `typst_bundle::bundle_document()` | 多文件捆绑导出 |

### 3.2 顶层编译入口

定义在 [typst/src/lib.rs:74-194](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst/src/lib.rs#L74-L194)：

```rust
pub fn compile<T>(world: &dyn World) -> Warned<SourceResult<T>>
where
    T: Output,
{
    // 1. 设置 Target 样式
    let base = StyleChain::new(&library.styles);
    let target = TargetElem::target.set(T::target()).wrap();
    let styles = base.chain(&target);

    // 2. 求值主文件为 Content
    let content = typst_eval::eval(...)?.content();

    // 3. 迭代布局直到内省稳定（最多 5 次）
    loop {
        document = T::create(&mut engine, &content, styles)?;
        if constraint.validate(document.introspector()) { break; }
        if history.is_full() { break; }
        history.push(document);
    }

    Ok(document)
}
```

### 3.3 各导出格式的调用链

```
typst-cli/src/compile.rs 顶层 export 函数
  │
  ├─ OutputFormat::Pdf / Png / Svg:
  │   let Warned { output, warnings } = typst::compile::<PagedDocument>(world);
  │   output.and_then(|doc| export_paged(&doc, config))
  │   │
  │   ├─ export_pdf(doc, config):
  │   │   typst_pdf::pdf(&doc, &options) → Vec<u8>
  │   │
  │   ├─ export_image(doc, config, Png):
  │   │   for (i, page) in doc.pages().iter().enumerate() {
  │   │       let pixmap = typst_render::render(page, &opts);
  │   │       pixmap.encode_png() → Vec<u8>
  │   │   }
  │   │
  │   └─ export_image(doc, config, Svg):
  │       for (i, page) in doc.pages().iter().enumerate() {
  │           let svg = typst_svg::svg(page, &opts);  → String
  │       }
  │
  └─ OutputFormat::Html:
      let Warned { output, warnings } = typst::compile::<HtmlDocument>(world);
      output.and_then(|doc| export_html(&doc, config))
          let html = typst_html::html(&doc, &options) → String
```

---

## 四、渲染输出管线详解

布局产出的 `PagedDocument` 包含多页 `Frame`，每页 `Frame` 是一棵 `(Point, FrameItem)` 树。三条渲染管线（PNG/SVG/PDF）各自遍历此树，对 `FrameItem::Shape` 调用各自的转换逻辑。

HTML 输出则走完全不同的路径——普通图形元素被忽略，只有 `html.frame` 内的内容通过内联 SVG 渲染。

### 4.1 PNG 渲染 (typst-render → tiny-skia)

入口：[render_shape](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-render/src/shape.rs#L11-L84)

```
Shape
  │
  ├─ Geometry::Line(target) → sk::PathBuilder::line_to()
  ├─ Geometry::Rect(size)   → sk::Rect::from_xywh() → sk::PathBuilder::from_rect()
  └─ Geometry::Curve(curve) → convert_curve(curve)
      │  CurveItem::Move(p)   → builder.move_to()
      │  CurveItem::Line(p)   → builder.line_to()
      │  CurveItem::Cubic(p1,p2,p3) → builder.cubic_to()
      │  CurveItem::Close     → builder.close()
      ▼
  sk::Path
  │
  ├─ fill → paint::to_sk_paint() → canvas.fill_path(path, paint, fill_rule, ...)
  └─ stroke → paint::to_sk_paint() + sk::Stroke → canvas.stroke_path(path, paint, stroke, ...)
```

填充规则映射：`NonZero → Winding`，`EvenOdd → EvenOdd`

### 4.2 SVG 渲染 (typst-svg → xmlwriter)

入口：[render_shape](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-svg/src/shape.rs#L13-L48)

```
Shape
  │
  ├─ 写 <path> 元素
  │   ├─ fill → write_fill() / attr("fill", "none")
  │   ├─ stroke → write_stroke()（含 dash-array, linecap, linejoin, miterlimit）
  │   └─ transform → attr("transform", ...)
  │
  └─ d 属性 → convert_geometry_to_path(&geometry)
      ├─ Geometry::Line(t)  → SvgPathBuilder::line_to()
      ├─ Geometry::Rect(s)  → SvgPathBuilder::rect()
      └─ Geometry::Curve(p) → convert_curve()
          CurveItem::Move(p)   → builder.move_to()  → "m x y" (相对坐标)
          CurveItem::Line(p)   → builder.line_to()  → "l/h/v x y"
          CurveItem::Cubic(p1,p2,p3) → builder.curve_to() → "c x1 y1 x2 y2 x y"
          CurveItem::Close     → builder.close()    → "Z"
```

[SvgPathBuilder](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-svg/src/path.rs#L8-L168) 使用**相对坐标**生成 SVG path `d` 属性，自动优化水平/垂直线段为 `h`/`v` 命令。

### 4.3 PDF 渲染 (typst-pdf → krilla)

入口：[handle_shape](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-pdf/src/shape.rs#L14-L75)

```
Shape
  │
  └─ convert_geometry(&geometry)
      ├─ Geometry::Line(l)  → PathBuilder::move_to(0,0) + line_to(l)
      ├─ Geometry::Rect(s)  → Rect::from_xywh() → PathBuilder::push_rect()
      └─ Geometry::Curve(c) → convert_path(c, &mut builder)
          CurveItem::Move(p)   → builder.move_to()
          CurveItem::Line(p)   → builder.line_to()
          CurveItem::Cubic(p1,p2,p3) → builder.cubic_to()
          CurveItem::Close     → builder.close()
          ▼
      krilla::Path
  │
  ├─ fill → paint::convert_fill() → surface.set_fill()
  └─ stroke → paint::convert_stroke() → surface.set_stroke()
  │
  └─ surface.draw_path(&path)
```

[convert_path](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-pdf/src/util.rs#L209-L225) 的实现与 PNG 的 `convert_curve` 几乎一模一样，只是目标类型从 `tiny_skia::PathBuilder` 换成了 `krilla::PathBuilder`。

### 4.4 HTML 输出边界与内联 SVG 机制

HTML 输出对图形元素的处理与 Paged 目标完全不同，分为两条路径：

#### 4.4.1 普通图形元素：直接忽略

在 `Target::Html` 下，[typst-html/rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-html/src/rules.rs) 只为 `ImageElem` 注册了 Show Rule（L83），**没有为 RectElem/LineElem/CurveElem/PolygonElem 等图形元素注册 Show Rule**。

因此，在 [convert.rs:155-160](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-html/src/convert.rs#L155-L160) 的 `handle()` 函数中，这些元素会落入 else 分支，被忽略并警告：

```rust
} else {
    converter.engine.sink.warn(warning!(
        child.span(),
        "{} was ignored during HTML export",
        child.elem().name(),
    ));
}
```

#### 4.4.2 html.frame：通过内联 SVG 渲染

`html.frame(body)` 是 HTML 目标下渲染图形的唯一途径。处理流程分为两步：

**第一步：布局阶段（convert.rs:140-154）**

```rust
} else if let Some(elem) = child.to_packed::<FrameElem>() {
    let locator = converter.locator.next(&elem.span());
    // 关键：临时切换到 Target::Paged
    let style = TargetElem::target.set(Target::Paged).wrap();
    // 调用 Paged 目标的布局流程
    let frame = (converter.engine.library.routines.layout_frame)(
        converter.engine,
        &elem.body,
        locator,
        styles.chain(&style),  // styles 叠加 Target::Paged
        Region::new(Size::splat(Abs::inf()), Axes::splat(false)),
    )?;
    // 包装为 HtmlFrame
    let mut node = HtmlFrame::new(frame, styles, elem.span()).into();
    make_block_level(&mut node).unwrap();
    converter.push(node);
}
```

**关键点**：
- 通过 `TargetElem::target.set(Target::Paged).wrap()` 临时切换目标
- 调用的是 Paged 目标的 `layout_frame`，因此内部会触发完整的 Paged 布局流程
- 产出的 `Frame` 包含 `FrameItem::Shape`，与 Paged 文档的 Frame 结构完全一致
- 包装为 `HtmlFrame` 后作为 `HtmlNode::Frame` 插入 DOM 树

**第二步：编码阶段（encode.rs:391-402）**

在 HTML 编码时，`HtmlNode::Frame` 会调用 `typst_svg::svg_in_html()` 生成内联 SVG：

```rust
fn write_frame(w: &mut Writer, frame: &HtmlFrame) {
    let svg = typst_svg::svg_in_html(
        &frame.inner,      // 来自 Paged 布局的 Frame
        frame.text_size,   // 用于 em 单位换算
        w.pretty,          // 是否格式化
        frame.id.as_deref(),  // SVG 元素 id
        &eco_format!("{}", frame.css.to_inline()),  // 内联 CSS
        &frame.anchors,    // 锚点位置（用于链接跳转）
        w.link_resolver,   // 链接解析器
    );
    w.buf.push_str(&svg);
}
```

#### 4.4.3 svg_in_html 与普通 svg 的区别

`typst_svg::svg_in_html()`（[lib.rs:82-120](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-svg/src/lib.rs#L82-L120)）与普通 `svg()` 函数的区别：

| 特性 | `svg(page, opts)`（单页导出） | `svg_in_html(frame, ...)`（内联） |
|------|-----------------------------|--------------------------------|
| 输入 | `&Page`（含 bleed 和 fill） | `&Frame`（无 bleed） |
| 输出 | 完整 XML 文档（含 `<?xml?>`） | 仅 `<svg>` 元素片段（直接嵌入 HTML） |
| 尺寸 | 用 pt 绝对单位 | 用 em 相对单位（`width: ${w}em; height: ${h}em`） |
| 额外参数 | 无 | `id`, `styles` (CSS), `anchors`, `link_resolver` |
| 共享代码 | ✅ `render_shape` / `convert_geometry_to_path` / `convert_curve` 全部共享 | 同左 |

**共享图形转换逻辑**：
- `svg_in_html` 内部创建 `SVGRenderer::with_options(Some(link_resolver))`
- 调用 `render_frame()` 遍历 Frame 树
- 对 `FrameItem::Shape` 调用同一套 `render_shape()` 函数
- `Geometry` → `SvgPathBuilder` 的转换完全复用

---

## 五、协作关系图

```
                    ┌─────────────────────────────────────────┐
                    │          编译入口 typst::compile::<T>()  │
                    │  [typst/src/lib.rs:L74-L82]              │
                    │  T=PagedDocument  or  T=HtmlDocument    │
                    └──────────────────┬──────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────┐
                    │          用户层 (Typst 代码)              │
                    │  #rect(fill: blue, stroke: red)          │
                    │  #curve(move(..), line(..), cubic(..))   │
                    │  #polygon((0pt,0pt), (10pt,10pt), ...)  │
                    │  #html.frame[#rect(...)]                 │
                    └──────────────────┬──────────────────────┘
                                       │ 解析
                    ┌──────────────────▼──────────────────────┐
                    │    元素定义 (typst-library/visualize/)   │
                    │  LineElem  RectElem  CurveElem           │
                    │  FrameElem (html.frame)                  │
                    │  Shape { geometry, fill, stroke, rule }  │
                    │  Geometry { Line, Rect, Curve }          │
                    │  Curve(Vec<CurveItem>)                   │
                    │  CurveItem { Move, Line, Cubic, Close }  │
                    └──────────────────┬──────────────────────┘
                                       │  Show Rule（按 Target 区分）
          ┌────────────────────────────┴────────────────────────────┐
          │ Target::Paged            │           Target::Html       │
          │ [typst-layout/rules.rs]  │       [typst-html/rules.rs]  │
          │ 7 种图形元素都有规则     │  仅 ImageElem 有规则         │
          ▼                          ▼                              ▼
┌─────────────────────────┐  ┌──────────────────────────┐  ┌─────────────────────┐
│ 布局层 (typst-layout/  │  │ convert.rs:L155-L160      │  │ convert.rs:L140-L154 │
│ shapes.rs)             │  │ 普通图形元素 → 忽略+警告  │  │ html.frame 处理      │
│                         │  │                         │  │  ├─ 切 Target::Paged  │
│ layout_line → Shape    │  │                         │  │  ├─ layout_frame()   │
│ layout_curve → Shape   │  │                         │  │  └─ Frame → HtmlFrame│
│ layout_rect → Shape    │  │                         │  │                        │
│ ...                    │  │                         │  │                        │
│ 产出: PagedDocument    │  │                         │  │ 产出: HtmlDocument    │
│   pages: Vec<Page>     │  │                         │  │   HtmlNode tree       │
│   Page.frame: Frame    │  │                         │  │   HtmlNode::Frame(..) │
└──────────┬─────────────┘  └──────────────────────────┘  └──────────┬───────────┘
           │                                                          │
           │ 后处理                                                  │ 编码
           │                                                          │
   ┌───────┴────────┬─────────────────┐                               │
   ▼                ▼                 ▼                               │
┌──────────┐   ┌──────────┐   ┌──────────┐                          │
│ PDF 输出 │   │ PNG 输出 │   │ SVG 输出 │                          │
│ typst-   │   │ typst-   │   │ typst-   │                          │
│ pdf      │   │ render   │   │ svg      │                          │
│          │   │          │   │          │                          │
│ Shape    │   │ Shape    │   │ Shape    │                          │
│ → krilla │   │ → sk::   │   │ → <path  │                          │
│ ::Path   │   │ Path     │   │ d="..." │                          │
│ → draw_  │   │ → fill_  │   │ → fill/  │                          │
│ path     │   │ path     │   │ stroke   │                          │
│          │   │ → stroke │   │ 属性     │                          │
│          │   │ _path    │   │          │                          │
│ krilla   │   │ tiny-    │   │ xml-     │                          │
│          │   │ skia     │   │ writer   │                          │
└──────────┘   └──────────┘   └──────┬───┘                          │
                                     │                              │
                                     │ svg_in_html()                │
                                     │ [typst-svg/lib.rs:L82-L120]  │
                                     │ 共享 render_shape 代码       │
                                     ▼                              ▼
                                 内联 <svg> 片段 ──────────────► HTML 字符串
```

---

## 六、关键设计模式总结

### 6.1 Output trait 的双实现架构

**核心事实：`Output` trait 只有两个核心实现**（定义在 [target.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-library/src/foundations/target.rs#L13-L30)）：

| 实现 | Target | 调用方 | 后处理 |
|------|--------|-------|-------|
| `PagedDocument` | `Target::Paged` | `typst::compile::<PagedDocument>()` | PNG / SVG / PDF（三个独立后处理函数） |
| `HtmlDocument` | `Target::Html` | `typst::compile::<HtmlDocument>()` | HTML 编码 |

**PNG/SVG/PDF 不是独立的 Output**，而是对 `PagedDocument` 的后处理。三者共享同一套布局流程和同一批 `Frame` 数据，只是最终渲染目标不同。

### 6.2 Geometry 三变体统一抽象

`Geometry` 的三种变体（Line / Rect / Curve）为不同复杂度的图形提供了分层表达：

| 变体 | 适用场景 | 渲染器处理 |
|------|---------|-----------|
| `Line` | 简单直线 | 各渲染器直接映射为 line_to |
| `Rect` | 无圆角矩形 | PNG/PDF 用原生矩形原语，SVG 用 4 段 path |
| `Curve` | 任意形状（含圆角矩形、椭圆、多边形） | 统一遍历 CurveItem 转换 |

圆/椭圆在布局时就已经被 `Curve::ellipse()` 转换为贝塞尔曲线，因此渲染器只需处理 `Curve`，无需特殊椭圆逻辑。

### 6.3 二次贝塞尔统一升阶

用户可写 `curve.quad()`，但 `CurveItem` 只有 `Cubic`。`CurveBuilder::quad()` 通过 `control_q2c()` 升阶：

```
二次 Q(p0, control, end) → 三次 C(p0, c1, c2, end)
  c1 = (p0 + 2*control) / 3
  c2 = (end + 2*control) / 3
```

这简化了渲染管线——所有后端只需处理三次贝塞尔。

### 6.4 矩形的双模式渲染

[styled_rect](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L686-L697) 根据复杂度选择两种策略：

1. **简单矩形**（`Geometry::Rect`）：无圆角、统一描边 → 单个 Shape，渲染器可用原生矩形原语
2. **分段矩形**（`Geometry::Curve`）：有圆角或不同边描边 → 拆分为填充 Shape + 若干描边 Shape

分段矩形中，实线描边优先使用 `fill_segment`（用填充区域模拟描边，能更好地处理角连接），虚线描边使用 `stroke_segment`。

### 6.5 圆角弧线的贝塞尔近似

所有圆角（矩形圆角、圆/椭圆）都通过 `bezier_arc_control` 函数（[L1372-L1386](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L1372-L1386)）将圆弧转换为三次贝塞尔，基于 [StackOverflow 算法](https://stackoverflow.com/a/44829356)。

### 6.6 Target 动态切换与 html.frame 桥接

HTML 目标下普通图形元素会被忽略，但 `html.frame()` 提供了桥接机制：

**关键机制**（[convert.rs:140-154](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-html/src/convert.rs#L140-L154)）：

```rust
// 1. 临时切换 Target
let style = TargetElem::target.set(Target::Paged).wrap();

// 2. 调用 Paged 目标的布局流程
let frame = layout_frame(engine, &elem.body, locator, styles.chain(&style), ...);

// 3. 包装为 HtmlFrame
HtmlFrame::new(frame, styles, elem.span())

// 4. 编码时调用 typst_svg::svg_in_html() 渲染为内联 SVG
```

**设计亮点**：
- `Target` 是一个样式（`TargetElem::target`），可以动态叠加切换
- `html.frame` 内部触发的是完整的 Paged 布局流程，因此支持所有 Paged 目标的功能
- `svg_in_html()` 与普通 SVG 导出共享 `render_shape` / `convert_geometry_to_path` 等核心转换逻辑
- 实现了 "HTML 文档中嵌入精确排版图形" 的能力，同时保持 HTML 输出的语义化

### 6.7 Frame 树作为统一中间表示

`Frame` + `FrameItem` 是布局与渲染之间的唯一契约。所有图形元素经过布局后都变成了 `(Point, FrameItem::Shape(Shape, Span))`。渲染器不需要知道"这个 Shape 来自 rect 还是 polygon"，只需遍历 Frame 树，对每个 Shape 调用 `Geometry` → 目标路径的转换。
