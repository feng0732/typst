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
- `Binary` - 二元运算符（如 `+`）
- `Relation` - 关系符（如 `=`, `<`, `>`）
- `Opening` / `Closing` - 开闭分隔符（如 `(`, `)`）
- `Large` - 大型运算符（如 `∑`, `∏`）
- `Fence` - 围栏（如 `|`）
- `Punctuation` - 标点（如 `,`）
- `Vary` - 可变类（如 `-`，根据上下文决定是二元还是一元运算符）
- `Diacritic` - 变音类（重音符号）

#### 间距计算核心函数

间距计算在 [process.rs 的 `spacing()` 函数](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/process.rs#L274-L316) 中执行。该函数接收左右两个相邻 `MathItem`，使用 `match (l.rclass(), r.lclass())` 模式匹配，**首次匹配即返回**——这是理解间距规则的关键：排在前面的规则优先级更高，后面的规则在前面已匹配时不会生效。

间距量常量定义在 [typst-library/src/math/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/mod.rs#L36-L40)：

```rust
pub const THIN: Em = Em::new(1.0 / 6.0);    // ≈0.167em
pub const MEDIUM: Em = Em::new(2.0 / 9.0);   // ≈0.222em
pub const THICK: Em = Em::new(5.0 / 18.0);   // ≈0.278em
```

#### 完整间距匹配规则

以下是 `spacing()` 中 match 臂的完整顺序和逻辑：

| 序号 | 匹配模式 (l.rclass, r.lclass) | 作用 | 设置 |
|------|-------------------------------|------|------|
| 1 | `(_, Punctuation)` | 标点前不加间距 | 无 |
| 2 | `(Punctuation, _)` 非 Script | 标点后加 Thin | `l.rspace = THIN` |
| 3 | `(Opening, _)` 或 `(_, Closing)` | 开分隔符后、闭分隔符前不加间距 | 无 |
| 4 | `(Relation, Relation)` | 连续关系符间不加额外间距 | 无 |
| 5 | `(Relation, _)` 非 Script | 关系符后加 Thick | `l.rspace = THICK` |
| 6 | `(_, Relation)` 非 Script | 关系符前加 Thick | `r.lspace = THICK` |
| 7 | `(Binary, _)` 非 Script | 二元运算符后加 Medium | `l.rspace = MEDIUM` |
| 8 | `(_, Binary)` 非 Script | 二元运算符前加 Medium | `r.lspace = MEDIUM` |
| 9 | `(Large, Opening \| Fence)` | **大型运算符后接开分隔符/围栏时，不加间距** | 无 |
| 10 | `(Large, _)` | 大型运算符后加 Thin | `l.rspace = THIN` |
| 11 | `(_, Large)` | 大型运算符前加 Thin | `r.lspace = THIN` |
| 12 | `l.is_spaced() \|\| r.is_spaced()` | 用户显式标记间距的元素 | 返回显式空格 |
| 13 | `_` | 默认不加间距 | 无 |

**重要**：所有间距规则在 Script 或 ScriptScript 尺寸下被 `if !script(l/r)` 守卫跳过，即上下标内的运算符之间不自动添加间距。

#### 左右侧有效类：rclass 与 lclass

`spacing()` 匹配的不是简单 `class()`，而是 `rclass()` 和 `lclass()`。定义在 [item.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/item.rs#L118-L146)：

- **`rclass()`**（右侧有效类）：`FencedItem` 若含闭分隔符且无显式类 → 返回 `Closing`；否则返回 `class()`
- **`lclass()`**（左侧有效类）：`FencedItem` 若含开分隔符且无显式类 → 返回 `Opening`；否则返回 `class()`

这意味着整个 `(x+y)` 在其左侧视为 `Opening`，右侧视为 `Closing`。例如 `∑ (x+y)` 中，`(x+y)` 的 `lclass()` 是 `Opening`，从而匹配 `(Large, Opening|Fence)` 臂 → 不加间距。

#### Vary 类的处理：一元 vs 二元

减号 `-` 的 Unicode 数学类是 `Vary`（可变），而非 `Binary`。在 [preprocess](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/process.rs#L213-L226) 中，`Vary` 会被动态转换为 `Binary`：

```rust
if item.class() == MathClass::Vary
    && let Some(RawMathItem::Item(prev)) = last.map(|i| &resolved[i])
    && matches!(
        prev.class(),
        MathClass::Normal | MathClass::Alphabetic
            | MathClass::Closing | MathClass::Fence
    )
{
    item.set_class(MathClass::Binary);
}
```

即：当前面是字母、数字、闭分隔符或围栏时，`-` 视为二元运算符（如 `x - y`）；否则保持 `Vary`（如行首 `-x` 或 `∑ -x`），此时 `Vary` 不命中 `spacing()` 中任何臂 → 不加间距，正确表现一元运算符行为。

#### 大型运算符间距规则详解

大型运算符（`Large` 类，如 `∑`、`∏`、`∐`、`⋃`、`⋁` 等）的间距规则**不是简单的"两侧加 Thick"**，而是遵循 TeXBook 第 170 页的规则：

1. **大型运算符后接开分隔符/围栏时不加间距**——`(Large, Opening|Fence)` 臂（序号 9）
   - 这是最特殊的规则：`∑ (x)` 中 ∑ 与 `(` 之间没有间距
   - 原因：大型运算符在 Display 尺寸下上下有极限标记（limits），它们占据了 ∑ 右侧的视觉空间，如果再加间距会显得太宽

2. **大型运算符后接其他元素时加 Thin**——`(Large, _)` 臂（序号 10）
   - `∑ x` → ∑ 与 x 之间有 THIN 间距
   - `∑ ∏` → ∑ 与 ∏ 之间有 THIN 间距（序号 10 先于序号 11 匹配，只设置 `l.rspace`）

3. **大型运算符前加 Thin**——`(_, Large)` 臂（序号 11）
   - `x ∑` → x 与 ∑ 之间有 THIN 间距
   - 但如果左侧是 Binary/Relation 等，更高优先级的臂会先匹配（见下文）

#### 大型运算符与其他运算类交互的优先级影响

由于 match 臂从上到下首次匹配，当 `Large` 与 `Binary`、`Relation` 相邻时，**Binary/Relation 的规则优先**：

| 左侧项 | 右侧项 | 命中臂 | 实际间距 |
|--------|--------|--------|----------|
| `Binary` | `Large` | 序号 7 `(Binary, _)` | MEDIUM（Binary 后），无 THIN（`(_, Large)` 未命中） |
| `Large` | `Binary` | 序号 8 `(_, Binary)` | MEDIUM（Binary 前），无 THIN（`(Large, _)` 未命中） |
| `Relation` | `Large` | 序号 5 `(Relation, _)` | THICK（Relation 后），无 THIN |
| `Large` | `Relation` | 序号 6 `(_, Relation)` | THICK（Relation 前），无 THIN |
| `Large` | `Opening/Fence` | 序号 9 | **无间距** |
| `Large` | `Closing` | 序号 3 `(_, Closing)` | **无间距** |
| `Opening` | `Large` | 序号 3 `(Opening, _)` | **无间距** |
| `Large` | `Alphabetic/Normal` | 序号 10 | THIN |
| `Alphabetic/Normal` | `Large` | 序号 11 | THIN |
| `Large` | `Large` | 序号 10 | THIN（后一个 Large 无 THIN） |

关键发现：**当 Large 与 Binary/Relation 相邻时，只有一侧间距生效，且是 Binary 的 MEDIUM 或 Relation 的 THICK，而非 Large 的 THIN**。

#### 间距的落地机制

`spacing()` 通过 `l.set_rspace(Some(Em))` / `r.set_lspace(Some(Em))` 将间距量写入 `MathProperties` 的 `lspace`/`rspace` 字段。在布局阶段，[layout_realized](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/mod.rs#L467-L551) 在每个组件前后各检查一次：

```rust
// 插入左间距
if let Some(lspace) = props.lspace && !props.align_form_infix && !lspace.is_zero() {
    let width = lspace.at(styles.resolve(TextElem::size));
    ctx.push(MathFragment::Space(width));
}
// ... 布局组件本身 ...
// 插入右间距
if let Some(rspace) = props.rspace && !rspace.is_zero() {
    let width = rspace.at(styles.resolve(TextElem::size));
    ctx.push(MathFragment::Space(width));
}
```

间距以 `Em` 为单位存储，在布局时乘以当前字体大小转为绝对长度 `Abs`，作为 `MathFragment::Space` 片段插入片段流。

#### 求和符间距示例：`$sum_(i=1)^n i = (n(n+1)) / 2$`

追踪此公式中 ∑ 周围的间距计算：

1. IR 构建后，`∑_(i=1)^n` 被打包为 `ScriptsItem`，它从基底的 `raw_class()` 继承了 `Large` 类
2. 紧跟其后的 `i` 是 `Alphabetic` 类
3. 调用 `spacing(∑_i^n, i)`：`(Large, Alphabetic)` → 命中序号 10 `(Large, _)` → `∑_i^n.rspace = THIN`
4. `i` 和 `=` 之间：`(Alphabetic, Relation)` → 命中序号 6 `(_, Relation)` → `=.lspace = THICK`
5. `=` 和 `(n(n+1))` 之间：`(Relation, Opening)` → 命中序号 3 `(Opening, _)` → 无间距（因为 `(n(n+1))` 的 `lclass()` 是 `Opening`）

所以 ∑ 与 i 之间有 THIN 间距，i 与 = 之间有 THICK 间距，= 与 `(` 之间无间距。

对比另一个示例 `$sum_(k=0)^n (2k+1)$`：

1. `∑_(k=0)^n` 是 `Large`
2. `(2k+1)` 是 `FencedItem`，`lclass() = Opening`
3. `spacing(∑, (2k+1))`：`(Large, Opening)` → 命中序号 9 → **无间距**
4. 这正是 TeXBook p170 规则的体现——求和符后接括号时不加间距

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
   - `AttachElem` → `ScriptsItem`（含 base=∑, b=i=1, t=n），从基底继承 `Large` 类
   - `LrElem` → `FencedItem`（含 open='(', body, close=')'），lclass=Opening, rclass=Closing
   - `FracElem` → `FractionItem`（含 numerator, denominator）

4. **间距处理**（[process.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-library/src/math/ir/process.rs)）：
   - `∑_i^n` 与 `i` 之间：`(Large, Alphabetic)` → THIN（∑ 的 rspace）
   - `i` 与 `=` 之间：`(Alphabetic, Relation)` → THICK（= 的 lspace）
   - `=` 与 `(n(n+1))` 之间：`(Relation, Opening)` → 无间距（Opening 后不加间距规则）
   - `(n(n+1))` 与 `/` 之间：`(Closing, ...)` → 无间距（Closing 前不加间距）
   - 分数线由 `FractionItem` 自身绘制，不依赖自动间距

5. **布局**（[typst-layout/src/math/](file:///d:/fz/0601-2/solo-dogfeeding/code/123-typst/crates/typst-layout/src/math/)）：
   - `layout_scripts`: 将 `i=1` 放在 ∑ 下方（Limits 样式，因 Display 尺寸），`n` 放在 ∑ 上方
   - `layout_fenced`: 将 `(` `)` 拉伸匹配 `n(n+1)` 高度
   - `layout_fraction`: 绘制分数线，上下放置分子分母
   - 每个组件前后按 `lspace`/`rspace` 插入 Space 片段
   - 所有片段水平排列，统一基线对齐

6. **渲染**：最终 `Frame` 交给 `typst-render` 或 `typst-pdf` 输出为 PNG/PDF/SVG。
