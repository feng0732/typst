# Typst 图形与路径：基础图形、路径数据与渲染输出的协作

## 全局数据流总览

```
用户代码 (Typst markup)
    │
    ▼
元素定义 (typst-library/visualize/)
  LineElem / RectElem / SquareElem / EllipseElem / CircleElem
  PolygonElem / CurveElem
    │
    ▼  Show Rule 注册 (typst-layout/rules.rs)
    │   将元素包装为 BlockElem::single_layouter(elem, layout_xxx)
    │
布局 (typst-layout/shapes.rs)
  layout_line / layout_rect / layout_square / ...
  layout_curve / layout_polygon
    │   产出 Frame，其中包含 FrameItem::Shape(Shape, Span)
    │
    ▼
Frame 树 (typst-library/layout/frame.rs)
  Frame -> Vec<(Point, FrameItem)>
  FrameItem::Shape / FrameItem::Group / FrameItem::Text / ...
    │
    ▼  三条输出管线各自遍历 Frame 树
    │
  ┌──────────┬──────────────┬──────────────┐
  ▼          ▼              ▼              ▼
PNG 渲染   SVG 渲染      PDF 渲染      HTML 渲染
typst-render  typst-svg    typst-pdf     typst-html
(tiny-skia)  (xmlwriter)  (krilla)
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

## 三、渲染输出管线

布局产出的 `PagedDocument` 包含多页 `Frame`，每页 `Frame` 是一棵 `(Point, FrameItem)` 树。三条输出管线各自遍历此树，对 `FrameItem::Shape` 调用各自的转换逻辑。

### 3.1 PNG 渲染 (typst-render → tiny-skia)

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

### 3.2 SVG 渲染 (typst-svg → xmlwriter)

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

### 3.3 PDF 渲染 (typst-pdf → krilla)

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

---

## 四、协作关系图

```
                    ┌─────────────────────────────────────────┐
                    │          用户层 (Typst 代码)              │
                    │  #rect(fill: blue, stroke: red)          │
                    │  #curve(move(..), line(..), cubic(..))   │
                    │  #polygon((0pt,0pt), (10pt,10pt), ...)  │
                    └──────────────────┬──────────────────────┘
                                       │ 解析
                    ┌──────────────────▼──────────────────────┐
                    │    元素定义 (typst-library/visualize/)   │
                    │  LineElem  RectElem  CurveElem           │
                    │  Shape { geometry, fill, stroke, rule }  │
                    │  Geometry { Line, Rect, Curve }          │
                    │  Curve(Vec<CurveItem>)                   │
                    │  CurveItem { Move, Line, Cubic, Close }  │
                    └──────────────────┬──────────────────────┘
                                       │ Show Rule + Layout
                    ┌──────────────────▼──────────────────────┐
                    │     布局层 (typst-layout/shapes.rs)      │
                    │  layout_line → Geometry::Line.stroked()  │
                    │  layout_curve → CurveBuilder → Curve     │
                    │  layout_polygon → Curve(move+lines+close)│
                    │  layout_rect/square/ellipse/circle       │
                    │    → styled_rect / Curve::ellipse()      │
                    │  产出: Frame { Vec<(Point, FrameItem)> } │
                    │  其中 FrameItem::Shape(Shape, Span)      │
                    └──────┬──────────┬──────────┬────────────┘
                           │          │          │
              ┌────────────▼┐  ┌──────▼──────┐  ┌▼──────────────┐
              │ PNG 输出     │  │ SVG 输出    │  │ PDF 输出      │
              │ typst-render │  │ typst-svg   │  │ typst-pdf     │
              │              │  │             │  │               │
              │ Shape        │  │ Shape       │  │ Shape         │
              │  → sk::Path  │  │  → <path d> │  │  → krilla    │
              │  → fill_path │  │  → fill属性 │  │    ::Path     │
              │  → stroke_   │  │  → stroke   │  │  → set_fill  │
              │    path      │  │    属性     │  │  → set_stroke │
              │              │  │             │  │  → draw_path  │
              │ tiny-skia    │  │ xmlwriter   │  │ krilla        │
              └──────────────┘  └─────────────┘  └───────────────┘
```

---

## 五、关键设计模式总结

### 5.1 Geometry 三变体统一抽象

`Geometry` 的三种变体（Line / Rect / Curve）为不同复杂度的图形提供了分层表达：

| 变体 | 适用场景 | 渲染器处理 |
|------|---------|-----------|
| `Line` | 简单直线 | 各渲染器直接映射为 line_to |
| `Rect` | 无圆角矩形 | PNG/PDF 用原生矩形原语，SVG 用 4 段 path |
| `Curve` | 任意形状（含圆角矩形、椭圆、多边形） | 统一遍历 CurveItem 转换 |

圆/椭圆在布局时就已经被 `Curve::ellipse()` 转换为贝塞尔曲线，因此渲染器只需处理 `Curve`，无需特殊椭圆逻辑。

### 5.2 二次贝塞尔统一升阶

用户可写 `curve.quad()`，但 `CurveItem` 只有 `Cubic`。`CurveBuilder::quad()` 通过 `control_q2c()` 升阶：

```
二次 Q(p0, control, end) → 三次 C(p0, c1, c2, end)
  c1 = (p0 + 2*control) / 3
  c2 = (end + 2*control) / 3
```

这简化了渲染管线——所有后端只需处理三次贝塞尔。

### 5.3 矩形的双模式渲染

[styled_rect](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L686-L697) 根据复杂度选择两种策略：

1. **简单矩形**（`Geometry::Rect`）：无圆角、统一描边 → 单个 Shape，渲染器可用原生矩形原语
2. **分段矩形**（`Geometry::Curve`）：有圆角或不同边描边 → 拆分为填充 Shape + 若干描边 Shape

分段矩形中，实线描边优先使用 `fill_segment`（用填充区域模拟描边，能更好地处理角连接），虚线描边使用 `stroke_segment`。

### 5.4 圆角弧线的贝塞尔近似

所有圆角（矩形圆角、圆/椭圆）都通过 `bezier_arc_control` 函数（[L1372-L1386](file:///d:/fz/0601-2/solo-dogfeeding/code/125-typst/crates/typst-layout/src/shapes.rs#L1372-L1386)）将圆弧转换为三次贝塞尔，基于 [StackOverflow 算法](https://stackoverflow.com/a/44829356)。

### 5.5 Frame 树作为统一中间表示

`Frame` + `FrameItem` 是布局与渲染之间的唯一契约。所有图形元素经过布局后都变成了 `(Point, FrameItem::Shape(Shape, Span))`。渲染器不需要知道"这个 Shape 来自 rect 还是 polygon"，只需遍历 Frame 树，对每个 Shape 调用 `Geometry` → 目标路径的转换。
