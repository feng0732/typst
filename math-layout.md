# Typst 数学公式排版代码分析

本文档从代码层面分析 Typst 中数学公式排版的实现原理，涵盖**符号**、**上下标**和**矩阵布局**三大核心要素，追踪它们如何从语法解析一步步落到最终页面上。

## 一、整体架构

数学公式排版在 Typst 中分为四个层次，自底向上依次为：

| 层次 | 模块 | 职责 |
|------|------|------|
| 语法解析 | `crates/typst-syntax/` | 将 `$...$` 中的源码解析为 AST |
| 求值 | `crates/typst-eval/src/math.rs` | 将 AST 求值为 `Content` 元素树 |
| IR 构建 | `crates/typst-library/src/math/ir/` | 将 `Content` 转换为数学中间表示 `MathItem` |
| 布局 | `crates/typst-layout/src/math/` | 将 `MathItem` 布局为 `Frame`（最终页面图形） |

入口函数有两个，分别对应行内公式和块级公式：

- **行内公式**：[layout_equation_inline](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/mod.rs#L51-L102)
- **块级公式**：[layout_equation_block](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/mod.rs#L106-L249)

两者共同的核心流程是：
1. 调用 `get_font()` 获取数学字体（需含 OpenType MATH 表）
2. 通过 `resolve_equation()` 构建 IR
3. 创建 `MathContext`，递归调用 `layout_into_fragments()` 完成布局
4. 处理基线对齐、方程编号等

## 二、数学符号排版

### 2.1 符号的 IR 表示

每个数学符号在 IR 层对应一个 [GlyphItem](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/item.rs)。构建过程位于 [resolve_symbol](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/resolve.rs#L330-L358)：

1. 将 `SymbolElem` 的文本按字形簇（grapheme cluster）拆分
2. 对每个字符应用数学样式转换（粗体、斜体、变体等），使用 `codex::styling::to_style()`
3. 如果字符是 `Large` 类且处于 Display 尺寸，自动添加纵向拉伸信息

### 2.2 数学类（MathClass）与间距

符号周围的间距由其**数学类**决定。`unicode-math-class` 库为每个 Unicode 数学字符定义了默认类别，包括：

- `Normal` - 普通符号
- `Alphabetic` - 字母（默认斜体）
- `Binary` - 二元运算符（如 `+`, `-`）
- `Relation` - 关系符（如 `=`, `<`, `>`）
- `Opening` / `Closing` - 开闭分隔符（如 `(`, `)`）
- `Large` - 大型运算符（如 `∑`, `∫`）
- `Fence` - 围栏
- `Punctuation` - 标点
- `Vary` - 可变类（根据上下文决定是 Binary 还是 Unary）

间距计算在 [process.rs 的 `spacing()` 函数](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/process.rs#L274-L300) 中执行，遵循 TeX 规则：

| 场景 | 间距量 |
|------|--------|
| 标点后 | `THIN` (1/6 em) |
| 关系符两侧 | `THICK` (5/18 em) |
| 二元运算符两侧 | `MEDIUM` (2/9 em) |
| 大型运算符两侧 | `THICK` |
| Script/ScriptScript 尺寸 | 自动禁用上述间距 |

间距常量定义在 [typst-library/src/math/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/mod.rs#L36-L40)：

```rust
pub const THIN: Em = Em::new(1.0 / 6.0);
pub const MEDIUM: Em = Em::new(2.0 / 9.0);
pub const THICK: Em = Em::new(5.0 / 18.0);
```

### 2.3 符号的字形布局

字形的实际绘制在 [layout_glyph](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/text.rs#L69-L119) 中完成：

1. 应用 `flac`（花体）和 `dtls`（无点字母 i/j）OpenType 特性
2. 调用 `GlyphFragment::new()` 从字体中查找字形，支持拉伸（stretch）
3. 对于 `Large` 类符号，调用 `center_on_axis()` 将其垂直居中于数学轴线（axis）上

数学轴线高度由字体 MATH 表的 `AxisHeight` 常数提供，是数学符号对齐的基准线。

## 三、上下标排版

### 3.1 上下标的元素与 IR 定义

上下标通过 [AttachElem](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/attach.rs#L20-L49) 定义，支持六个方向的附件：

```rust
pub struct AttachElem {
    pub base: Content,    // 基底
    pub t: Option<Content>,  // 顶部（智能定位：极限或上标）
    pub b: Option<Content>,  // 底部（智能定位：极限或下标）
    pub tl: Option<Content>, // 左上
    pub bl: Option<Content>, // 左下
    pub tr: Option<Content>, // 右上
    pub br: Option<Content>, // 右下
}
```

求值阶段在 [typst-eval/src/math.rs 的 `Eval for ast::MathAttach`](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-eval/src/math.rs#L111-L134) 中完成：将语法树中的 `^` 和 `_` 分别映射为 `t`/`b`。

### 3.2 智能定位：Scripts vs Limits

`t` 和 `b` 并不总是放在角上，而是根据**基底类型**和**数学尺寸**智能决定位置，由 [Limits 枚举](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/attach.rs#L148-L196) 控制：

| 基底类型 | 默认行为 |
|----------|----------|
| 大型运算符（Large）如 `∑` | Display 尺寸用 Limits（上下方），行内用 Scripts（角上） |
| 积分号 `∫` | 始终用 Scripts（即使 Display） |
| 关系符（Relation）如 `=` | 始终用 Limits |
| 其他 | 始终用 Scripts |

用户可通过 `scripts()` 和 `limits()` 函数强制覆盖。

### 3.3 上下标布局算法

核心实现在 [scripts.rs 的 `layout_attachments`](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/scripts.rs#L92-L202)，分为以下步骤：

#### 步骤 1：计算垂直偏移量

调用 [compute_script_shifts](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/scripts.rs#L318-L382) 计算上标和下标基线相对基底基线的距离：

**上标偏移 `shift_up`** 取以下各项的最大值：
- 字体常数 `SuperscriptShiftUp`（挤压样式用 `SuperscriptShiftUpCramped`）
- 非文本类符号：`base.ascent() - SuperscriptBaselineDropMax`
- `SuperscriptBottomMin + sup.descent`（确保上标底部不低于某值）

**下标偏移 `shift_down`** 取以下各项的最大值：
- 字体常数 `SubscriptShiftDown`
- 非文本类符号：`base.descent() + SubscriptBaselineDropMin`
- `sub.ascent() - SubscriptTopMax`（确保下标顶部不高于某值）

**上下标同时存在时**，还需保证两者间距至少为 `SubSuperscriptGapMin`，不足时将上标上移 `SuperscriptBottomMaxWithSubscript`，剩余空间上下平分。

#### 步骤 2：计算极限偏移量

调用 [compute_limit_shifts](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/scripts.rs#L290-L313)：

- 上极限：`base.ascent() + max(UpperLimitBaselineRiseMin, UpperLimitGapMin + limit.descent())`
- 下极限：`base.descent() + max(LowerLimitBaselineDropMin, LowerLimitGapMin + limit.ascent())`

#### 步骤 3：计算水平字距（Math Kern）

调用 [math_kern](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/scripts.rs#L389-L425)，基于 OpenType MATH 表的 `MathKernInfo` 表：

1. 对每个角（TopLeft/TopRight/BottomLeft/BottomRight）计算两个校正高度
2. 在两个高度处分别查询基底和脚本的 kern 值并求和
3. 取较大值（即更靠近的那个）作为最终 kern

正 kern 表示脚本远离基底，负 kern 表示靠近。下标还需额外减去基底的 italic correction（斜体校正）。

#### 步骤 4：组装最终 Frame

计算所有片段的宽度和高度后：
- 前脚本（左上/左下）向左扩展帧宽
- 后脚本（右上/右下）和极限向右扩展帧宽
- 所有片段按计算好的 x/y 坐标推入最终 `Frame`

### 3.4 数学尺寸级别

上标/下标会自动切换到更小的尺寸级别，由 [style_for_superscript](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/style.rs#L315-L322) 控制：

| 当前尺寸 | 脚本尺寸 |
|----------|----------|
| Display / Text | Script |
| Script / ScriptScript | ScriptScript |

下标同时应用 `cramped` 样式（降低上标高度）。缩放比例由字体 MATH 表的 `ScriptPercentScaleDown`（默认 70%）和 `ScriptScriptPercentScaleDown`（默认 50%）决定。

## 四、矩阵布局

### 4.1 矩阵元素定义

矩阵由 [MatElem](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/matrix.rs#L91-L222) 定义，核心字段：

```rust
pub struct MatElem {
    pub delim: DelimiterPair,     // 分隔符（默认圆括号）
    pub align: HAlignment,        // 单元格水平对齐（默认居中）
    pub augment: Option<Augment>, // 增广线配置
    pub row_gap: Rel<Length>,     // 行间距（默认 0.2em）
    pub column_gap: Rel<Length>,  // 列间距（默认 0.5em）
    pub rows: Vec<Vec<Content>>,  // 二维数据
}
```

向量 `VecElem` 和分支 `CasesElem` 共享相同的底层机制。

### 4.2 矩阵布局流程

核心实现在 [table.rs 的 `layout_table`](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/table.rs#L18-L192)，分三个阶段：

#### 阶段 1：独立布局每个单元格

遍历所有单元格，调用 [layout_cell](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/table.rs#L200-L208)：

1. 将单元格内容按对齐点 `&` 拆分为多个"子列"（sub-columns）
2. 记录每个单元格的最大 ascent 和 descent

为了保证矩阵与其他公式基线对齐，用合成的 `(` 字形的 ascent/descent 作为每行的最小高度。

#### 阶段 2：计算列宽和对齐点

对每一列：
- 调用 [compute_sub_column_widths](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/table.rs#L211-L221) 计算所有子列的最大宽度
- 每个子列宽度取该列所有单元格对应子列宽度的最大值

#### 阶段 3：组装矩阵

调用 [stack_rows](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/run.rs#L113-L147) 将每行堆叠：

1. 每行先用 [row_into_line_frame](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/run.rs#L302-L345) 构建水平帧：
   - 右对齐子列从对齐点向左放置
   - 左对齐子列从对齐点向右放置
2. 行与行之间用 `row_gap` 作为行距

#### 阶段 4：绘制增广线和分隔符

增广线（augment）支持：
- `hline`: 水平线数组，数字表示在第几行之后画线
- `vline`: 垂直线数组，数字表示在第几列之后画线
- `stroke`: 线条样式

默认线粗为 `0.05em`，线端为方形（Square Cap）。矩阵外的分隔符（圆括号/方括号/花括号）由 [layout_fenced](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/fenced.rs#L10-L64) 负责，它会将分隔符字形纵向拉伸以匹配矩阵总高度。

### 4.3 矩阵基线

矩阵的最终基线设置在：
```
frame.set_baseline(height / 2.0 + axis)
```

即矩阵垂直中心加上数学轴线高度，确保矩阵与周围公式在视觉上对齐。

## 五、其他关键概念

### 5.1 数学尺寸级别（MathSize）

定义在 [style.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/style.rs#L260-L282)，共四级：

| 级别 | 用途 | 典型缩放 |
|------|------|----------|
| `Display` | 块级公式 | 100% |
| `Text` | 行内公式 | 100% |
| `Script` | 一级上下标 | ~70% |
| `ScriptScript` | 二级上下标 | ~50% |

`Display` 与 `Text` 的区别主要影响：
- 分数的分子/分母偏移量和间距
- 大型运算符是否使用 Limits 样式
- 大型运算符是否自动纵向拉伸

### 5.2 挤压样式（Cramped）

上标在分母、根号内等位置时，会使用挤压样式，上标高度降低。由 `EquationElem::cramped` 字段控制。

### 5.3 斜体校正（Italic Correction）

数学斜体字母右侧会有额外的留白（italic correction），用于上标对齐。上标会自动消耗这个空间，使字母与上标更紧凑。

### 5.4 MathContext

[MathContext](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/mod.rs#L364-L463) 是数学布局的全局状态：

- `engine`: 排版引擎引用
- `region`: 可用区域
- `fonts_stack`: 字体栈（支持局部字体切换）
- `fragments`: 当前正在构建的片段列表

### 5.5 MathFragment

`MathFragment` 是布局的中间产物，有多种变体：
- `GlyphFragment`: 单个字形（含字距、数学类、italic correction）
- `FrameFragment`: 任意帧片段
- `Space`: 空格
- `Tag`: 内省标签

每个片段记录自己的宽度、高度、ascent、descent、基线等信息，供上层组合使用。

## 六、从源码到页面：完整追踪

以公式 `$ sum_(i=1)^n i = (n(n+1)) / 2 $` 为例，追踪完整流程：

1. **语法解析**：`typst-syntax` 将源码解析为 AST，包含 `Equation > Math > [MathAttach(base: MathIdent("sum"), b: Math(...), t: Math(...)), MathIdent("i"), MathDelimited(...), MathFrac(...)]`

2. **求值**（[typst-eval/src/math.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-eval/src/math.rs)）：
   - `MathAttach` → `AttachElem::new(Symbol("∑")).with_b(...).with_t(...)`
   - `MathDelimited` → `LrElem`（圆括号）
   - `MathFrac` → `FracElem`

3. **IR 构建**（[typst-library/src/math/ir/resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/resolve.rs)）：
   - `AttachElem` → `ScriptsItem`（含 base=∑, b=i=1, t=n）
   - `LrElem` → `FencedItem`（含 open='(', body, close=')'）
   - `FracElem` → `FractionItem`（含 numerator, denominator）
   - `process_group` 自动插入间距（`∑` 两侧 Thick，`=` 两侧 Thick 等）

4. **布局**（[typst-layout/src/math/](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/)）：
   - `layout_scripts`: 将 `i=1` 放在 ∑ 下方（Limits 样式，因 Display 尺寸），`n` 放在 ∑ 上方
   - `layout_fenced`: 将 `(` `)` 拉伸匹配 `n(n+1)` 高度
   - `layout_fraction`: 绘制分数线，上下放置分子分母
   - 所有片段水平排列，统一基线对齐

5. **渲染**：最终 `Frame` 交给 `typst-render` 或 `typst-pdf` 输出为 PNG/PDF/SVG。
