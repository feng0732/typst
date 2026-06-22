# Typst 表格布局机制深度解析

本文档基于 Typst 源码逐行梳理表格（Table）和网格（Grid）的布局机制，重点聚焦于**行列分配**、**跨页处理**、**近似假设**、**降级策略**和**绘制顺序**五个核心难点。

---

## 一、总体架构

### 1.1 表格与网格的关系

在 Typst 中，**Table 本质上是 Grid 的特化版本**。两者共享几乎相同的布局引擎，仅在默认样式（如 `stroke`、`inset`）和语义上有所区别。

| 方面 | Table | Grid |
|------|-------|------|
| 语义 | 语义化表格数据 | 纯展示布局 |
| 默认边框 | 有边框（`stroke` 默认启用） | 无边框 |
| 默认内边距 | 5pt | 0pt |
| 辅助元素 | `table.header` / `table.footer` | `grid.header` / `grid.footer` |
| 布局引擎 | 完全相同（`GridLayouter`） | 完全相同 |

关键代码：
- 表格模型定义：[table.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/model/table.rs)
- 布局入口：[grid/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/mod.rs#L98-L122)

### 1.2 布局流程总览

表格布局遵循以下**6 个阶段**的流水线：

```
┌─────────────────────────────────────────────────────────┐
│ 阶段 1: Synthesize - 合成 CellGrid                       │
│   table_to_cellgrid() / grid_to_cellgrid()               │
│   • 解析行列配置 (columns, rows)                         │
│   • 插入 gutter 轨道 (间距)                              │
│   • 处理 colspan/rowspan，标记合并单元格                 │
│   • 解析 hline/vline（显式线条）                         │
│   • 识别 header/footer 区域                              │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 2: measure_columns() - 列宽分配                     │
│   GridLayouter::measure_columns()                        │
│   • 先分配相对列宽 (Sizing::Rel)                         │
│   • 再测量自动列宽 (Sizing::Auto)                        │
│   • 有剩余空间：分配给分数字段 (Sizing::Fr)               │
│   • 空间不足：压缩自动列                                 │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 3: 行布局循环 (逐行处理)                             │
│   for y in 0..rows.len() { layout_row(y) }               │
│   对每一行:                                              │
│   • 检查 rowspan 单元格 → 加入待处理队列                │
│   • 检查不可断行组 (unbreakable row group) → 必要时换页  │
│   • 根据行类型选择布局策略:                              │
│     - Auto 行: measure_auto_row() → 可跨多区域           │
│     - Rel 行: 固定高度，必要时换页                       │
│     - Fr 行: 先占位，最后分配剩余空间                    │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 4: finish_region() - 区域完成                       │
│   当区域满或表格结束时调用:                               │
│   • 孤儿预防 (orphan prevention) → 移除单独的 header    │
│   • 分配 Fr 行高度                                       │
│   • 输出行高，更新 rowspan 的累计高度                    │
│   • 处理已完成的 rowspan → layout_rowspan()              │
│   • 放置 footer (如果是重复 footer)                      │
│   • 如果还有后续区域:                                    │
│     - 准备下一页 footer 空间                             │
│     - 放置重复 header                                    │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 5: 剩余 rowspan 处理                                │
│   • 所有行布局完毕后，处理遗漏的 rowspan                 │
└──────────────────────┬──────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 6: render_fills_strokes() - 渲染填充和线条         │
│   对每个区域 frame:                                      │
│   • 生成垂直线段 (vlines) → 先绘制，优先级低             │
│   • 生成水平线段 (hlines) → 后绘制，优先级高             │
│   • 绘制单元格填充 (fills)                               │
│   • 按线宽排序后统一 prepend 到 frame 中                 │
└─────────────────────────────────────────────────────────┘
```

---

## 二、行列分配机制

### 2.1 轨道 (Track) 类型

行和列使用统一的 `Sizing` 类型描述尺寸：

```rust
pub enum Sizing {
    Auto,           // 自动尺寸：根据内容计算
    Rel(Rel<Length>), // 相对/固定尺寸：如 5cm、10% 等
    Fr(Fr),         // 分数字段：按比例分配剩余空间
}
```

**Gutter（间距）处理**：当设置了 `column-gutter` 或 `row-gutter` 时，会在每两个相邻轨道之间插入额外的间距轨道。最终的轨道数组会交替排列：

```
无 gutter:  [Col0, Col1, Col2]
有 gutter:  [Col0, Gutter0, Col1, Gutter1, Col2]
```

相关代码：[resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/layout/grid/resolve.rs#L681-L756) 中的 `CellGrid::new_internal()`

### 2.2 列宽分配算法

列宽的分配在 [`GridLayouter::measure_columns()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L914-L957) 中完成，遵循以下顺序：

#### 步骤 1：分配相对列 (Rel)

```rust
for (&col, rcol) in self.grid.cols.iter().zip(&mut self.rcols) {
    match col {
        Sizing::Auto => {}
        Sizing::Rel(v) => {
            let resolved = v.resolve(self.styles).relative_to(self.regions.base().x);
            *rcol = resolved;
            rel += resolved;
        }
        Sizing::Fr(v) => fr += v,
    }
}
```

相对列优先分配，占用固定空间。

#### 步骤 2：测量自动列 (Auto)

调用 [`measure_auto_columns()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L966-L1099)：

**核心规则**：
- 遍历每个自动列 x，找到该列中所有单元格的最大宽度
- 对于 **colspan 单元格**：其宽度只影响**最后一个被跨越的自动列**
- 启发式优化：如果一个 colspan 跨越了所有分数字段且区域宽度有限，则不计入自动列扩展（因为 fr 列会提供空间）
- 对于 **rowspan 单元格**：跨行单元格只在其起始行参与列宽测量一次

```
示例：Col1(20pt) + Gutter + Col2(Auto) + Col3(Fr)
      单元格 (x=1,y=0) colspan=2，内容宽 100pt
      → Col2 实际需要 = 100pt - 20pt (Col1) = 80pt
      → 用 cell_spanned_width() 减去已确定列的宽度
```

#### 步骤 3：分配剩余空间

```rust
let remaining = available - auto;
if remaining >= Abs::zero() {
    // 有剩余：分给 Fr 列
    self.grow_fractional_columns(remaining, fr);
} else {
    // 空间不足：公平压缩 Auto 列
    self.shrink_auto_columns(available, count);
}
```

#### 步骤 4：公平压缩算法 (shrink_auto_columns)

当自动列的总需求超过可用空间时，使用**迭代式公平分配**算法：

```
初始: available = 可用总宽, overlarge = Auto 列数
循环最多 overlarge 次:
  fair = available / overlarge
  遍历每个 Auto 列:
    如果 rcol <= fair: 该列不需要压缩，从 overlarge 中移除
  如果本轮有列被移除: 继续迭代
  否则: 所有 overlarge 列统一使用 fair 宽度
```

该算法确保较小的 Auto 列保持原样，仅压缩那些超过公平份额的大列。

### 2.3 行高分配机制

行高是**逐行动态确定**的，因为行可能跨页。行高分配在布局循环中进行。

#### 相对行 (Rel Row)

最简单：直接解析为固定高度。如果当前区域放不下，且不是不可断行组的一部分，则换页。

#### 分数行 (Fr Row)

最特殊：
- 在 `layout_row_internal()` 中仅推入一个占位 `Row::Fr(v, y, disambiguator)`
- 等到 `finish_region()` 时，根据区域内剩余空间按比例分配

```rust
let remaining = self.regions.full - used;
let height = v.share(fr, remaining);
```

#### 自动行 (Auto Row)

最复杂，调用 [`measure_auto_row()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1243-L1425)。

### 2.4 colspan 和 rowspan 的影响

**列维度（colspan）**：

| colspan 跨越的列类型 | 处理方式 |
|---------------------|---------|
| 全部是 Rel/Fr | 不影响 Auto 列测量 |
| 包含 Auto | 宽度需求扣除已确定的其他列后，加到最后一个 Auto 列 |
| 跨越全部 Fr 列 + 部分 Auto | 在有限宽度区域内，不计入 Auto 扩展（启发式） |

**行维度（rowspan）**：

| rowspan 跨越的行类型 | 处理方式 |
|---------------------|---------|
| 全部是 Rel | 不需要扩展，高度固定 |
| 包含 Auto | 高度需求扣除已确定高度后，加到最后一个 Auto 行 |
| 跨越 Auto + Gutter + 其他 Auto | 需要运行模拟，预测换页对 gutter 的影响 |

---

## 三、跨页处理机制 - 逐行深度分析

### 3.1 区域 (Region) 概念

表格使用 `Regions` 抽象来表示跨页：

```rust
pub struct Regions<'a> {
    pub size: Size,           // 当前区域尺寸（剩余空间递减）
    pub base: Size,           // 初始区域尺寸（基准，用于 Rel 计算）
    pub full: Abs,            // 初始完整区域高度
    pub backlog: &'a [Abs],   // 后续区域的高度列表（待处理页）
    pub last: Option<Abs>,    // 最后一个区域的高度，None 表示无限后续区域
    pub expand: Axes<bool>,   // 宽/高是否允许扩展
    pub full: bool,           // 当前区域是否已满
}
```

当 `size.y` 不足以容纳下一行时，调用 `regions.next()` 推进到下一个区域（换页），`backlog` 中的下一个高度成为新的 `size.y`。

### 3.2 finish_region() - 跨页核心逻辑逐行解析

[`finish_region()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1599-L1834) 是跨页处理的入口。以下是逐行分析：

#### 3.2.1 孤儿预防（Orphan Prevention）- L1607-L1618

```rust
if let Some(orphan_snapshot) = self.current.lrows_orphan_snapshot.take()
    && !last
{
    self.current.lrows.truncate(orphan_snapshot);
    self.current.repeated_header_rows =
        self.current.repeated_header_rows.min(orphan_snapshot);

    if orphan_snapshot == 0 {
        // Removed all repeated headers.
        self.current.last_repeated_header_end = 0;
    }
}
```

**工作原理**：
- 当 `lrows_orphan_snapshot` 存在（说明刚刚放置了 header 但还没有后续内容行），且不是最后一页时
- 将已布局的行 `lrows` 截断到快照位置，相当于"撤回"刚刚放置的 header
- 同时更新重复 header 计数等状态变量
- 被撤回的 header 仍保留在 `pending_headers` 队列中，下一页会自动重新尝试

**典型场景**：
- 页面底部刚好只能放下 header，放不下任何内容行
- 触发换页后，header 被"带回"到下一页顶部

#### 3.2.2 移除末尾 Gutter 行 - L1620-L1630

```rust
if self
    .current
    .lrows
    .last()
    .is_some_and(|row| self.grid.is_gutter_track(row.index()))
{
    // Remove the last row in the region if it is a gutter row.
    self.current.lrows.pop().unwrap();
    self.current.repeated_header_rows =
        self.current.repeated_header_rows.min(self.current.lrows.len());
}
```

**设计意图**：
- 页面末尾的 gutter（行间距）没有意义，应该移除
- 这是导致 rowspan 模拟复杂性的根源：gutter 的存在与否取决于换页位置，而换页位置又取决于行高测量结果

#### 3.2.3 Widow 预防（Footer 检查）- L1632-L1647

```rust
let footer_would_be_widow = matches!(&self.grid.footer, Some(footer) if footer.repeated)
    && self.current.lrows.is_empty()
    && self.current.could_progress_at_top;
```

**检查条件**：
1. 存在重复 footer
2. 当前页面没有任何内容行（只有 header 不算）
3. 可以换页（`could_progress_at_top` 为 true）

如果满足所有条件，**不放置 footer**，避免 footer 单独出现在页面上。

#### 3.2.4 Footer 布局 - L1648-L1663

```rust
let mut laid_out_footer_start = None;
if !footer_would_be_widow && let Some(footer) = &self.grid.footer {
    if footer.repeated
        && self.current.lrows.iter().all(|row| row.index() < footer.start)
    {
        laid_out_footer_start = Some(footer.start);
        self.layout_footer(footer, engine, self.finished.len(), true)?;
    }
}
```

**条件解读**：
- footer 是重复的（`footer.repeated == true`）
- 当前所有已布局行都在 footer 起始行之前

满足条件时，在当前页面底部放置重复 footer。注意 `is_being_repeated = true`，标记这是重复出现而非最终出现。

#### 3.2.5 高度统计与 Fr 行分配 - L1665-L1698

```rust
// 统计已使用高度和 Fr 总量
let mut used = Abs::zero();
let mut fr = Fr::zero();
for row in &self.current.lrows {
    match row {
        Row::Frame(frame, _, _) => used += frame.height(),
        Row::Fr(v, _, _) => fr += *v,
    }
}

// 确定区域大小：有 Fr 行则扩展到 full 高度
let mut size = Size::new(self.width, used).min(self.current.initial);
if fr.get() > 0.0 && self.current.initial.y.is_finite() {
    size.y = self.current.initial.y;
}

// 遍历所有行，放置 Fr 行
for (i, row) in std::mem::take(&mut self.current.lrows).into_iter().enumerate() {
    let (frame, y, is_last) = match row {
        Row::Frame(frame, y, is_last) => (frame, y, is_last),
        Row::Fr(v, y, disambiguator) => {
            let remaining = self.regions.full - used;
            let height = v.share(fr, remaining);
            (self.layout_single_row(engine, disambiguator, height, y)?, y, true)
        }
    };
    // ... 放置 frame，统计 header 高度 ...
}
```

**关键点**：
- Fr 行的高度是在区域结束时才确定的，基于 `regions.full - used`
- 如果有 Fr 行，区域高度会扩展到 `initial.y`（完整页面高度）

#### 3.2.6 Rowspan 高度累计 - L1707-L1746

```rust
for rowspan in self
    .rowspans
    .iter_mut()
    .filter(|rowspan| (rowspan.y..rowspan.y + rowspan.rowspan).contains(&y))
    .filter(|rowspan| {
        rowspan.max_resolved_row.is_none_or(|max_row| y > max_row)
    })
{
    // 设置 first_region 和 dy
    if rowspan.first_region > current_region {
        rowspan.first_region = current_region;
        rowspan.dy = pos.y;
        rowspan.region_full = self.regions.full;
    }

    // 确保 heights 数组足够长
    let amount_missing_heights = (current_region + 1)
        .saturating_sub(rowspan.heights.len() + rowspan.first_region);
    rowspan
        .heights
        .extend(std::iter::repeat_n(Abs::zero(), amount_missing_heights));

    // 累计当前行高度到该区域
    *rowspan.heights.last_mut().unwrap() += height;

    if is_last {
        rowspan.max_resolved_row = Some(y);
    }
}
```

**核心逻辑**：
- 对每个跨越当前行 `y` 的 rowspan，更新其 `heights` 数组
- `first_region` 记录 rowspan 首次出现的页面索引
- `dy` 记录在第一页的垂直偏移（因为第一页可能不是从顶部开始）
- `region_full` 记录第一页的完整高度（用于后续重新测量）
- `max_resolved_row` 防止重复累计（同一行的多帧情况）

#### 3.2.7 完成的 Rowspan 布局 - L1753-L1790

```rust
let mut i = 0;
while let Some(rowspan) = self.rowspans.get(i) {
    if laid_out_footer_start.is_none_or(|footer_start| {
        y < footer_start || rowspan.y >= footer_start
    }) && (rowspan.y + rowspan.rowspan < y + 1
        || rowspan.y + rowspan.rowspan == y + 1 && is_last)
    {
        // 条件满足：该 rowspan 在当前行或之前结束
        let rowspan = self.rowspans.remove(i);
        self.layout_rowspan(
            rowspan,
            Some((&mut output, repeated_header_row_height)),
            engine,
        )?;
    } else {
        i += 1;
    }
}
```

**触发条件（满足其一）**：
1. `rowspan.y + rowspan.rowspan < y + 1`：rowspan 结束于当前行之前
2. `rowspan.y + rowspan.rowspan == y + 1 && is_last`：rowspan 结束于当前行，且是当前行的最后一帧

**Footer 边界处理**：
- 如果 footer 已经布局，则只有完全在 footer 内或完全在 footer 外的 rowspan 才会被布局
- 避免 rowspan 跨越 footer 边界导致渲染错误

**注意**：使用 `while let Some` + `remove(i)` 而非 `for` 循环，因为删除元素后索引会变化。

#### 3.2.8 下一区域准备 - L1807-L1831

```rust
if !last {
    // 重置 header 状态
    self.current.repeated_header_rows = 0;
    self.current.last_repeated_header_end = 0;
    self.current.repeating_header_height = Abs::zero();
    self.current.repeating_header_heights.clear();

    let disambiguator = self.finished.len();
    if let Some(footer) =
        self.grid.footer.as_ref().and_then(Repeatable::as_repeated)
    {
        self.prepare_footer(footer, engine, disambiguator)?;
    }

    // 预先扣除 footer 高度
    self.regions.size.y -= self.current.footer_height;
    self.current.initial_after_repeats = self.regions.size.y;

    // 放置重复 header
    if !self.repeating_headers.is_empty() || !self.pending_headers.is_empty() {
        self.layout_active_headers(engine)?;
    }
}
```

**重要顺序**：
1. 先重置 header 状态
2. 再 prepare_footer（可能触发换页）
3. 扣除 footer 高度
4. 最后 layout_active_headers（再次可能触发换页）

### 3.3 行跨页的近似假设与测量逻辑

#### 3.3.1 核心近似假设

在 [`run_rowspan_simulation()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L879-L1002) 中有明确的注释说明局限性：

```rust
// A flaw of this approach is that we consider rowspans' content to
// be contiguous. That is, we treat rowspans' requested heights as
// a simple number, instead of properly using the vector of
// requested heights in each region. This can lead to some
// weirdness when using multi-page rowspans with content that
// reacts to the amount of space available, including paragraphs.
// However, this is probably the best we can do for now.
```

**近似假设 1：内容连续性假设**
- 将 rowspan 的内容视为连续的单一高度需求，而不是按页面分割的向量
- 简化了计算，但对于对可用空间敏感的内容（如段落、弹性布局）可能不准确
- 当 rowspan 跨多页且内容在不同页面有不同布局行为时，会出现偏差

**近似假设 2：Header/Footer 高度不变假设**

在 [`RowspanSimulator::new()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1032-L1051) 中：

```rust
// There can be no new headers or footers within a multi-page
// rowspan, since headers and footers are unbreakable, so
// assuming the repeating header height and footer height
// won't change is safe.
header_height: current.repeating_header_height,
footer_height: current.footer_height,
```

- 假设在 rowspan 跨越的多个页面中，重复 header 和 footer 的高度保持不变
- 这是合理的，因为 header/footer 本身是不可断的，且在 rowspan 开始前就已确定

**近似假设 3：仅考虑固定高度行**

在 [`prepare_rowspan_sizes()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L612-L695) 中：

```rust
// We can only predict the resolved size of upcoming fixed-size
// rows, but not fractional rows. In the future, we might be
// able to simulate and circumvent the problem with fractional
// rows. Relative rows are currently always measured relative
// to the first region as well.
// We can ignore auto rows since this is the last spanned auto
// row.
let will_be_covered_height: Abs = self
    .grid
    .rows
    .iter()
    .skip(auto_row_y + 1)
    .take(last_spanned_row - auto_row_y)
    .map(|row| match row {
        Sizing::Rel(v) => {
            v.resolve(self.styles).relative_to(self.regions.base().y)
        }
        _ => Abs::zero(),
    })
    .sum();
```

- 计算"未来"行将覆盖的高度时，只考虑 Rel（固定高度）行
- Fr 行和 Auto 行按 0 高度估计
- 这是保守估计，可能导致 auto 行扩展过多，但避免了布局不足

#### 3.3.2 Rowspan 模拟算法详解

当 rowspan 跨越 gutter 时，需要运行模拟算法。触发条件：

```rust
if auto_row_y != last_spanned_row      // 不结束于当前行
    && !sizes.is_empty()               // 有高度需求
    && self.grid.has_gutter            // 存在 gutter
    && !is_effectively_unbreakable_rowspan  // 不是不可断
{
    return true;  // 需要模拟
}
```

**模拟流程（最多 5 次迭代）**：

```rust
for _attempt in 0..5 {
    // 1. 创建模拟器，假设 auto 行扩展 amount_to_grow
    let rowspan_simulator = RowspanSimulator::new(...);

    // 2. 模拟布局，计算能覆盖的总高度
    let total_spanned_height = rowspan_simulator.simulate_rowspan_layout(
        y, max_spanned_row, amount_to_grow,
        requested_rowspan_height, ...
    )?;

    // 3. 检查是否足够
    if (total_spanned_height + amount_to_grow).fits(requested_rowspan_height) {
        // 足够：减去被其他行覆盖的高度，返回成功
        subtract_end_sizes(simulated_sizes, requested_rowspan_height - amount_to_grow);
        return Ok(true);
    }

    // 4. 不够：更新 amount_to_grow，推进模拟区域，继续迭代
    let old_amount_to_grow = std::mem::replace(
        &mut amount_to_grow,
        requested_rowspan_height - total_spanned_height,
    );

    // 5. 推进 regions，模拟 auto 行扩展后的换页
    let mut extra_amount_to_grow = amount_to_grow - old_amount_to_grow;
    while extra_amount_to_grow > Abs::zero()
        && simulated_regions.size.y < extra_amount_to_grow
    {
        extra_amount_to_grow -= simulated_regions.size.y.max(Abs::zero());
        simulated_regions.next();
        simulated_regions.size.y -=
            self.current.repeating_header_height + self.current.footer_height;
        disambiguator += 1;
    }
    simulated_regions.size.y -= extra_amount_to_grow;
}
```

**收敛性证明**：
- `amount_to_grow` 严格单调递增（因为 `total_spanned_height + old_amount < requested`）
- 最多 5 次迭代，无论是否收敛都停止

#### 3.3.3 测量时的空 Frame 跳过机制

在 [`measure_auto_row()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1338-L1355) 中有一个特殊的 HACK：

```rust
// HACK: Also consider frames empty if they only contain tags. Table
// and grid cells need to be locatable for pdf accessibility, but
// the introspection tags interfere with the layouting.
fn is_empty_frame(frame: &Frame) -> bool {
    frame.items().all(|(_, item)| matches!(item, FrameItem::Tag(_)))
}

// Skip the first region if one cell in it is empty. Then,
// remeasure.
if let Some([first, rest @ ..]) =
    frames.get(measurement_data.frames_in_previous_regions..)
    && can_skip
    && breakable
    && is_empty_frame(first)
    && rest.iter().any(|frame| !is_empty_frame(frame))
{
    return Ok(None);
}
```

**目的**：
- 第一帧只有 Tag（用于 PDF 可访问性）但后续有实际内容时，说明内容应该从下一页开始
- 返回 `None` 触发重新测量，跳过第一页

**触发重测的逻辑在 `layout_auto_row()`**：

```rust
let mut resolved = match self.measure_auto_row(
    engine, disambiguator, y, true,  // can_skip = true
    self.unbreakable_rows_left, None,
)? {
    Some(resolved) => resolved,
    None => {
        // 第一页为空，换页后重新测量
        self.finish_region(engine, false)?;
        self.measure_auto_row(
            engine, disambiguator, y, false,  // can_skip = false
            self.unbreakable_rows_left, None,
        )?.unwrap()  // 此时一定返回 Some
    }
};
```

### 3.4 出错/边界情况的降级处理策略

#### 3.4.1 Rowspan 模拟失败降级

当 5 次迭代仍未收敛时，在 [`simulate_and_measure_rowspans_in_auto_row()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L800-L822) 中降级：

```rust
if !simulations_stabilized {
    // If the simulation didn't stabilize above, we will just pretend
    // all gutters were removed, as a best effort. That means the auto
    // row will expand more than it normally should, but there isn't
    // much we can do.
    let will_be_covered_height = self
        .grid
        .rows
        .iter()
        .enumerate()
        .skip(y + 1)
        .take(max_spanned_row - y)
        .filter(|(y, _)| !self.grid.is_gutter_track(*y))  // 忽略所有 gutter
        .map(|(_, row)| match row {
            Sizing::Rel(v) => {
                v.resolve(self.styles).relative_to(self.regions.base().y)
            }
            _ => Abs::zero(),
        })
        .sum();

    subtract_end_sizes(&mut simulated_sizes, will_be_covered_height);
}
```

**降级策略**：
- **保守假设**：所有 gutter 都被移除（即所有换页都恰好发生在 gutter 位置）
- **结果**：auto 行扩展量会比实际需要的多，但避免了内容溢出
- **权衡**：宁可空白多一点，也不让内容被裁剪

#### 3.4.2 Unbreakable Rowspan 强制单页

在 [`measure_auto_row()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1299-L1314) 中：

```rust
let pod = if !breakable {
    // Force cell to fit into a single region when the row is
    // unbreakable, even when it is a breakable rowspan, as a best
    // effort.
    let mut pod: Regions = Region::new(size, self.regions.expand).into();
    pod.full = measurement_data.full;

    if measurement_data.frames_in_previous_regions > 0 {
        // Best effort to conciliate a breakable rowspan which
        // started at a previous region going through an
        // unbreakable auto row. Ensure it goes through previously
        // laid out regions, but stops at this one when measuring.
        pod.backlog = backlog;
    }

    pod
}
```

**降级策略**：
- 当 auto 行所在的 unbreakable 组包含不可断 rowspan 时，强制该 auto 行不可断
- 即使 rowspan 本身标记为可断（`breakable = true`），也强制放在单页
- 这是"尽力而为"的处理，可能导致页面底部出现大块空白

#### 3.4.3 无法换页时的溢出处理

在 [`may_progress_with_repeats()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L24-L31) 中定义了何时可以换页：

```rust
pub fn may_progress_with_repeats(&self) -> bool {
    self.current.could_progress_at_top
        || self.regions.last.is_some()
            && self.regions.size.y != self.current.initial_after_repeats
}
```

**条件解读**：
1. `could_progress_at_top`：当前区域顶部可以推进（有 backlog 或可以换页）
2. 或者还有后续区域（`last.is_some()`）且当前区域已有内容（`size.y != initial_after_repeats`）

如果不能换页但内容放不下，内容会**溢出**（Typst 的默认行为是显示警告但继续渲染）。

#### 3.4.4 Header 无法放置时的降级

在 [`layout_new_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L391-L429) 中：

```rust
let should_snapshot = !short_lived
    && self.current.lrows_orphan_snapshot.is_none()
    && self.may_progress_with_repeats();

if should_snapshot {
    self.current.lrows_orphan_snapshot = Some(self.current.lrows.len());
}

// ... 布局 header ...

if !may_progress {
    // Flush pending headers immediately, as placing them again later
    // won't help.
    self.flush_orphans();
}
```

**降级策略**：
- 如果无法换页（`may_progress = false`），立即 flush_orphans()，放弃孤儿预防
- header 会被强制放置在当前位置，即使成为孤儿

#### 3.4.5 Footer 跳过区域后的重测

在 [`prepare_footer()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L471-L512) 中：

```rust
self.current.footer_height = if skipped_region {
    // Simulate the footer again; the region's 'full' might have
    // changed.
    self.simulate_footer(footer, &self.regions, engine, disambiguator)?
        .height
} else {
    footer_height
};
```

**原因**：
- 跳过区域后，`regions.full` 可能从 `inf` 变为有限值
- 这会影响 Rel 行的高度计算（Rel 是相对于 `regions.full` 解析的）
- 因此需要重新模拟 footer 高度

### 3.5 头部和尾部高度重算的影响

> **修正说明**：之前关于"Header 高度不会在换页时重算"的说法不准确。实际上 Header 和 Footer 的重算逻辑在**普通布局流程**和**Rowspan 模拟流程**中是不同的，需要分开讨论。

#### 3.5.1 两种流程的代码路径对比

表格布局中有两条独立的代码路径都会涉及 header/footer 高度计算：

| 流程 | 代码位置 | 用途 |
|------|---------|------|
| **普通布局流程** | [repeated.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs) | 实际布局表格内容，生成真实的 Frame |
| **Rowspan 模拟流程** | [rowspans.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1150-L1244) | 预测换页对 gutter 的影响，计算 auto 行需要扩展的高度 |

#### 3.5.2 普通布局流程中的重算逻辑

普通布局在每次换页时调用 [`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L203-L352)，其内部流程如下：

```
layout_active_headers():
  │
  ├─ 步骤 1: simulate_header_height() → 预测量 header 高度
  │    （仅用于判断是否需要跳页，不用于最终布局）
  │
  ├─ 步骤 2: 跳页循环（如果空间不够）
  │    │
  │    ├─ finish_region_internal() → 跳页
  │    │
  │    ├─ ⚠️  TODO 问题点 [L234-L238]:
  │    │   // re-calculate heights of headers and footers
  │    │   // on each region if 'full' changes?
  │    │   // (Assuming height doesn't change for now...)
  │    │   → 此处 Header 高度没有重新模拟！
  │    │
  │    └─ ✅  Footer 重新模拟 [L245-L256]:
  │         if skipped_region:
  │             simulate_footer() → 重算 footer 高度
  │
  ├─ 步骤 3: 重置 header 高度 [L272-L275]
  │    repeating_header_height = 0
  │
  ├─ 步骤 4: ✅  实际布局 repeating headers [L296-L322]
  │    for header in repeating_headers:
  │        layout_header_rows() → 真实布局，返回实际高度
  │        repeating_header_height += header_height
  │
  └─ 步骤 5: ✅  实际布局 pending headers [L328-L339]
       for header in pending_headers:
           layout_header_rows() → 真实布局，返回实际高度
```

**关键结论（普通布局）**：

| 阶段 | Header | Footer |
|------|--------|--------|
| 预测量（跳页判断） | ❌ 跳过区域后不重新模拟 | ✅ 跳过区域后重新模拟 |
| 实际布局 | ✅ 每次换页都重新布局测量 | ✅ 每次换页都重新布局测量 |

**修正之前的说法**：
- ❌ 错误："Header 高度不会在换页时重算"
- ✅ 正确：**Header 在实际布局时会重新测量，但在跳页判断的预模拟阶段不会重新模拟**（这是已知的 TODO 问题）
- ✅ 正确：**Footer 在跳页判断和实际布局两个阶段都会重新模拟**

#### 3.5.3 Rowspan 模拟流程中的重算逻辑

Rowspan 模拟使用独立的 [`RowspanSimulator`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1006-L1051) 结构体，其换页时调用 [`simulate_header_footer_layout()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1150-L1244)：

```rust
// We can't just use the initial header/footer height on each region,
// because header/footer height might vary depending on region size if
// it contains rows with relative lengths. Therefore, we re-simulate
// headers and footers on each new region.
fn simulate_header_footer_layout(...) {
    // 1. 第一次模拟
    let header_height = simulate_header_height(regions);
    let footer_height = simulate_footer(regions);

    // 2. 跳页循环
    while !fits(header_height + footer_height) {
        regions.next();
        skipped_region = true;
    }

    // 3. ✅  跳过区域后，Header 重新模拟
    if skipped_region {
        header_height = simulate_header_height(new_regions);
    }

    // 4. ✅  跳过区域后，Footer 重新模拟
    if skipped_region {
        footer_height = simulate_footer(new_regions);
    }

    // 5. 扣除高度
    regions.size.y -= header_height + footer_height;
}
```

**关键结论（Rowspan 模拟）**：

| 阶段 | Header | Footer |
|------|--------|--------|
| 首次模拟 | ✅ 模拟 | ✅ 模拟 |
| 跳过区域后 | ✅ 重新模拟 | ✅ 重新模拟 |
| 每次换页时 | ✅ 重新模拟 | ✅ 重新模拟 |

**与普通布局的差异**：
- Rowspan 模拟中 **Header 和 Footer 在每次换页和跳页后都会重新模拟**，比普通布局更严谨
- 代码注释明确说明原因：header/footer 可能包含 Rel 行（相对高度，如 `10%`），其高度依赖于 `regions.full`

#### 3.5.4 触发重算的完整场景总结

| 场景 | 触发位置 | Header 重算 | Footer 重算 |
|------|---------|------------|------------|
| **普通布局 - 新页开始** | [`finish_region()` L1827-L1830](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1827-L1830) → `layout_active_headers()` | ✅ 实际布局重新测量 | ✅ `prepare_footer()` + 实际布局 |
| **普通布局 - 跳页判断阶段** | [`layout_active_headers()` L222-L256](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L222-L256) | ❌ 不重新模拟（TODO 问题） | ✅ 重新模拟 |
| **普通布局 - prepare_footer 跳页** | [`prepare_footer()` L481-L509](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L481-L509) | ❌ 不涉及 | ✅ 跳过区域后重新模拟 |
| **Rowspan 模拟 - 每次换页** | [`RowspanSimulator::finish_region()` L1259](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1259) | ✅ 重新模拟 | ✅ 重新模拟 |
| **Rowspan 模拟 - 跳页后** | [`simulate_header_footer_layout()` L1207-L1234](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1207-L1234) | ✅ 重新模拟 | ✅ 重新模拟 |

**重算原因**：
- 跳过区域意味着进入了新的页面，新页面的 `regions.full` 可能与前一页不同
- 例如：第一页是无限高度（如 float 容器），后续页面是有限高度
- Rel 行的高度是相对于 `regions.full` 解析的，因此需要重新模拟

#### 3.5.5 高度差异对跨页表格测量的影响

**影响 1：Auto 行测量的准确性**

在 [`prepare_auto_row_cell_measurement()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L402-L436) 中：

```rust
let mapped_regions = self.regions.map(&mut custom_backlog, |size| {
    Size::new(
        size.x,
        size.y
            - self.current.repeating_header_height  // ← 依赖此值
            - self.current.footer_height,           // ← 依赖此值
    )
});
```

- Auto 行测量时，backlog 中的每个区域高度都要预先减去头尾高度
- 如果头尾高度估计不准确，测量出的行高会有偏差
- Rowspan 模拟流程因为每次都重算，所以更准确

**影响 2：Rowspan 模拟的准确性**

在 [`RowspanSimulator::new()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1044-L1045) 中：

```rust
header_height: current.repeating_header_height,
footer_height: current.footer_height,
```

- 模拟开始时使用当前的头尾高度作为初始值
- 但后续每次换页都会通过 `simulate_header_footer_layout()` 重新计算
- 这确保了模拟过程中高度始终与当前区域匹配

**影响 3：跳页判断的准确性（普通布局的 TODO 问题）**

在 [`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L222-L243) 中：

```rust
// 跳过区域循环
while !fits(header_height) {
    finish_region_internal();  // 跳页
    // ⚠️  Header 高度没有更新！
    regions.size.y -= footer_height;  // footer 也还没更新（后面才更新）
}
```

- 如果 header 包含 Rel 行，跳页后 `regions.full` 变化会导致 header 实际高度变化
- 但跳页判断时使用的是旧的 header 高度，可能导致：
  - 多跳了不必要的页面
  - 跳页不够，header 在当前页放不下
  - 临界条件下的边界判断偏差
- **⚠️ 修正之前的过度说法**："结果仍然正确" 仅在 header 组不可断且当前页至少能放下 header 的前提下成立
- 如果实际高度显著大于预测量高度，且当前页剩余空间不足以容纳，**可能导致内容溢出**
- 详细分析见 3.6 节

#### 3.5.6 已知的 TODO 问题

[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L234-L238)：

```rust
// TODO(layout model): re-calculate heights of headers and footers
// on each region if 'full' changes? (Assuming height doesn't
// change for now...)
//
// Would remove the footer height update below (move it here).
```

**问题说明**：
- 普通布局的跳页循环中，Header 高度应该和 Footer 一样在每次跳页后重新模拟
- 目前只有 Footer 重新模拟了，Header 没有
- 这是一个已知的待改进点，但在大多数情况下影响不大，因为最终实际布局时会重新测量

---

### 3.6 普通重复表头的跳页影响深度分析

本节深入分析三个关键机制之间的相互作用：
1. **旧表头高度用于跳页判断**
2. **表头组不可断**
3. **实际布局不在组内递归换页**

#### 3.6.1 三者关系的完整代码路径

[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L203-L352) 的完整流程如下：

```
┌─────────────────────────────────────────────────────────┐
│ 阶段 1: 预测量 - 使用旧 regions.full 模拟 header 高度     │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 2: 跳页判断循环                                      │
│                                                          │
│ L222: while unbreakable_rows_left == 0                   │
│          && !fits(header_height)  ← 使用旧高度判断!      │
│          && may_progress()                               │
│ {                                                        │
│     finish_region_internal() → 跳页                       │
│     regions.full 可能变化！                               │
│     ⚠️  Header 高度不更新！（TODO 问题）                   │
│     footer_height 在循环结束后才更新                       │
│ }                                                        │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 3: Footer 重算（如果跳过了区域）                     │
│ L245-L256: simulate_footer() → 重新计算 footer 高度       │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 4: 表头组标记为不可断                                │
│                                                          │
│ L264-L267:                                               │
│ // Group of headers is unbreakable.                      │
│ // Thus, no risk of 'finish_region' being recursively   │
│ // called from within 'layout_row'.                      │
│ unbreakable_rows_left += header_rows + pending_rows;     │
│                                                          │
│ 关键设计：设置 unbreakable_rows_left > 0                 │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 5: 实际布局 header 行                                │
│                                                          │
│ for header in repeating_headers:                         │
│     layout_header_rows() → 调用 layout_row_with_state()   │
│         → layout_row_internal()                           │
│                                                          │
│ L435: 🔒  关键锁：                                       │
│ if unbreakable_rows_left == 0                            │
│     && is_full() && is_content_row                       │
│ {                                                        │
│     finish_region() → 只有 unbreakable_rows_left == 0    │
│                        才会触发换页！                     │
│ }                                                        │
│                                                          │
│ → 所以布局 header 期间不会换页！                          │
└─────────────────────────────┬───────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│ 阶段 6: 每行布局后递减计数器                              │
│ L461: unbreakable_rows_left -= 1                        │
└─────────────────────────────────────────────────────────┘
```

#### 3.6.2 设计意图分析

**为什么使用旧高度判断 + 不可断组 + 不递归换页的组合？**

| 机制 | 设计目的 | 代价 |
|------|---------|------|
| **旧高度预判断** | 避免在实际布局时才发现空间不够，导致需要回退 | 高度可能不准确 |
| **表头组不可断** | 确保 header 完整出现在一页顶部，不被拆分 | 失去了布局期间换页的能力 |
| **不递归换页** | 防止"header 放不下→换页→又要放 header→又放不下"的无限递归 | 一旦预判断错误，没有补救机会 |

**三者的依赖关系**：
- 如果没有"不可断组"标记，实际布局期间可能触发换页，导致递归调用 `layout_active_headers()`
- 如果没有"旧高度预判断"，就无法预先确定 header 应该放在哪一页
- 两个机制共同作用，确保了布局的终止性，但牺牲了一定的灵活性

**与其他两种场景的对比要点**：
- 普通不可断内容行同样使用"模拟 + 不可断组"的组合，但它的 **Rel 行在换页后会用新 full 重新解析**，所以即使预测量不准确，实际高度也会自动适配
- Rowspan 模拟不使用"不可断组"锁，而是**每次跳页后都重新模拟 header/footer**，确保模拟数据始终与当前 regions 匹配
- 重复表头卡在中间：Rel 行实际布局会重新解析（高度正确），但**跳页判断使用旧预测量**（换页位置可能错误），再加上锁换页，三者叠加导致溢出风险

#### 3.6.3 修正"结果一定正确"的过度说法

之前的说法 **"最终实际布局时会重新测量，所以结果仍然正确"** 是不准确的。

**正确的边界条件分析**：

实际布局时 header 会重新测量高度，但 **由于 `unbreakable_rows_left > 0`，即使测量发现高度超过了当前区域，也不会触发换页**。

```rust
// layout_row_internal() L435
if self.unbreakable_rows_left == 0  // ← 布局 header 期间不为 0！
    && self.regions.is_full()
    && is_content_row
{
    self.finish_region(engine, false)?;  // ← 不会执行！
}
```

**所以结果正确性取决于：**

| 场景 | 结果 | 正确性 |
|------|------|--------|
| 实际高度 ≤ 预测量高度 | 当前页能放下 | ✅ 正确 |
| 实际高度 > 预测量高度，但 ≤ 当前区域剩余空间 | 当前页能放下，只是占了更多空间 | ✅ 结果正确，但后续行可用空间减少 |
| 实际高度 > 预测量高度，且 > 当前区域剩余空间 | **内容溢出** | ❌ 错误！ |

**⚠️ 关键结论**：
- ❌ **过度说法**："最终实际布局时会重新测量，所以结果仍然正确"
- ✅ **正确说法**："如果预测量高度大于等于实际高度，或者实际高度虽然更大但仍能适应当前区域，结果是正确的。如果实际高度显著超过预测量且当前区域空间不足，**会发生内容溢出**。"

#### 3.6.4 多跳页、溢出、边界偏差的触发条件

##### 触发条件 1：Header 包含 Rel 行且 `regions.full` 变化

```rust
// simulate_unbreakable_row_group() L332
Sizing::Rel(v) => v.resolve(self.styles).relative_to(regions.base().y),
```

- Rel 行的高度是相对于 `regions.base().y`（即 `regions.full`）解析的
- 如果跳页前 `regions.full` 是 `inf`（如在 float 容器中），跳页后是有限值（如 A4 纸高度）
- 预测量时使用 `inf` 解析，Rel 行高度为 `0pt`（因为 `x% of inf = 0`）
- 实际布局时使用有限值解析，Rel 行高度为 `x% of page_height`
- **预测量高度 < 实际高度**，可能导致溢出

##### 触发条件 2：Footer 高度重算进一步压缩可用空间

```rust
// layout_active_headers() L251-L255
self.regions.size.y += self.current.footer_height;  // 加回旧 footer 高度
self.current.footer_height = simulate_footer(...);  // 重算，可能更大
self.regions.size.y -= self.current.footer_height;  // 减去新 footer 高度
```

- 如果 footer 也包含 Rel 行，跳页后 footer 高度也会增加
- 这进一步压缩了 header 可用的空间
- 即使 header 本身高度没变化，也可能因为 footer 变大而放不下

##### 触发条件 3：边界条件判断刚好处于临界点

假设：
- 预测量 header 高度 = 98pt
- 当前区域剩余空间 = 100pt
- 98pt ≤ 100pt → 不跳页
- 实际 header 高度 = 101pt（因为包含 Rel 行）
- 101pt > 100pt → **溢出**

这是最容易出现问题的场景：预测量刚好满足，但实际差一点点。

##### 触发条件 4：多跳不必要的页面

反过来，如果预测量高度 > 实际高度：
- 预测量 header 高度 = 120pt
- 当前区域剩余空间 = 100pt
- 120pt > 100pt → 跳页
- 实际 header 高度 = 80pt
- 80pt ≤ 100pt → **本来可以放在当前页，但多跳了一页**

这虽然不会导致溢出，但会产生不必要的空白页。

#### 3.6.5 三种场景的完整流程对比

代码中存在三种涉及"先模拟→可能换页→实际布局"的场景：
1. 普通不可断内容行组
2. 重复表头/表尾组
3. Rowspan 模拟（`RowspanSimulator`）

##### 场景 1：普通不可断内容行组的完整流程

代码路径：[`check_for_unbreakable_rows()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L237-L297) → `layout_row_internal()` → `layout_relative_row()` / `layout_auto_row()`

```
┌───────────────────────────────────────────────────────────────┐
│ 步骤 1: 模拟（使用当前 regions.full）                           │
│                                                               │
│ simulate_unbreakable_row_group(current_row, None,             │
│     &self.regions, ...)                                       │
│                                                               │
│ 内部：Rel 行高度 = v.resolve().relative_to(regions.base().y)  │
│                    = v.resolve().relative_to(regions.full)    │
│                                                               │
│ → 得到 row_group.height（使用当前 full 解析）                  │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 2: 空间不足 → 换页                                        │
│                                                               │
│ while !fits(row_group.height) {                               │
│     finish_region() → regions.next()                          │
│                                │                              │
│                                ▼                              │
│                          // Regions::next() 关键代码:          │
│                          self.size.y = new_height;             │
│                          self.full = new_height;  // full 也变!│
│ }                                                             │
│                                                               │
│ → regions.full 已更新为新页面的高度！                          │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 3: 实际布局 Rel 行（使用新 regions.full 重新解析）         │
│                                                               │
│ layout_relative_row():                                        │
│   resolved = v.resolve(self.styles)                           │
│              .relative_to(self.regions.base().y)  // ← 新 full!│
│                                                               │
│ → ✅  关键：使用换页后的新 full 重新解析！                     │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 4: 实际布局 Auto 行（使用新 regions 重新测量）             │
│                                                               │
│ layout_auto_row():                                            │
│   measure_auto_row(..., unbreakable_rows_left, ...)           │
│     → breakable = false (因为 unbreakable_rows_left > 0)      │
│     → 使用无限高度测量 (Abs::inf())                            │
│     → 但可用空间扣除了 header/footer                          │
│                                                               │
│ → Auto 行虽然是固定高度测量（unbreakable），但高度准确          │
└───────────────────────────────────────────────────────────────┘
```

**普通不可断内容行的 `regions.full` 风险**：
- ✅ **Rel 行风险低**：模拟用旧 full → 换页后实际布局用新 full 重新解析
- ⚠️ **仅存在 Auto 行模拟时的偏差**：`simulate_unbreakable_row_group()` 中的 Auto 行使用 `Abs::inf()` 测量，与实际布局一致，所以 Auto 行高度不受 full 变化影响
- **结论**：普通不可断内容行对 `regions.full` 变化的**鲁棒性最好**

##### 场景 2：重复表头组的完整流程

代码路径：[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L203-L352)

```
┌───────────────────────────────────────────────────────────────┐
│ 步骤 1: 预测量（使用当前 regions.full）                        │
│                                                               │
│ simulate_header_height(repeating_headers,                     │
│     &self.regions, ...)                                       │
│ → 内部调用 simulate_unbreakable_row_group()                  │
│ → 使用当前 regions.full 解析 Rel 行                           │
│                                                               │
│ → 得到 header_height（使用旧 full 解析）                      │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 2: 空间不足 → 换页                                        │
│                                                               │
│ while !fits(header_height) {                                  │
│     finish_region_internal() → regions.next()                 │
│                                │                              │
│                                ▼                              │
│                          regions.full = new_height  // full 变!│
│     ⚠️  Header 高度不重新模拟！（TODO 问题）                   │
│ }                                                             │
│                                                               │
│ 循环结束后：                                                  │
│   → Footer 重新模拟 (使用新 regions.full)                     │
│   → Header ❌ 不重新模拟！                                    │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 3: 标记不可断组                                          │
│                                                               │
│ unbreakable_rows_left += header_rows + pending_rows;          │
│                                                               │
│ → 🔒  实际布局期间禁止换页！                                   │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 4: 实际布局 header 行（使用新 regions.full 重新解析）     │
│                                                               │
│ layout_header_rows() → layout_row_with_state()               │
│   → layout_relative_row():                                    │
│       resolved = v.resolve()                                  │
│                  .relative_to(self.regions.base().y) // 新full│
│                                                               │
│ → ✅  Rel 行用新 full 重新解析！                             │
│                                                               │
│   → layout_auto_row():                                        │
│       breakable = false (unbreakable_rows_left > 0)           │
│       Auto 行高度准确                                          │
│                                                               │
│ → 🔒  但布局期间不能换页（即使空间不够）！                    │
└───────────────────────────────────────────────────────────────┘
```

**重复表头的 `regions.full` 风险**：
- ✅ **Rel 行实际布局使用新 full 重新解析** → 实际高度正确
- ❌ **跳页判断使用旧 full** → 预测量高度可能与实际高度不一致
  - 如果旧 full 解析高度 < 新 full 解析高度 → 预测量偏小 → 可能**溢出**
  - 如果旧 full 解析高度 > 新 full 解析高度 → 预测量偏大 → 可能**多跳页**
- **结论**：重复表头对 `regions.full` 变化的**鲁棒性最差**，是三者中唯一存在溢出风险的

##### 场景 3：Rowspan 模拟（`RowspanSimulator`）的完整流程

代码路径：[`simulate_header_footer_layout()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1150-L1244)（在 `RowspanSimulator` 内部）

```
┌───────────────────────────────────────────────────────────────┐
│ 步骤 1: 首次模拟 header/footer（使用当前模拟器 regions.full）  │
│                                                               │
│ header_height = simulate_header_height(headers,               │
│     &self.regions, ...)                                       │
│ footer_height = simulate_footer(footer, &self.regions, ...)   │
│                                                               │
│ → 使用模拟器内部的 regions.full                               │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 2: 空间不足 → 跳页                                        │
│                                                               │
│ while !fits(header_height + footer_height) {                  │
│     self.regions.next()  // 模拟器内部的 regions               │
│     self.finished += 1;                                       │
│ }                                                             │
│                                                               │
│ skipped_region = true;                                        │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 3: ✅  Header 和 Footer 都重新模拟（使用新 regions.full） │
│                                                               │
│ if skipped_region {                                           │
│     header_height = simulate_header_height(                   │
│         repeating_headers, &self.regions, ...)  // 新regions! │
│                                                               │
│     footer_height = simulate_footer(                          │
│         footer, &self.regions, ...)  // 新regions!           │
│ }                                                             │
│                                                               │
│ → 两者都使用新 full 重新模拟！                                │
└────────────────────────────────┬──────────────────────────────┘
                                 ▼
┌───────────────────────────────────────────────────────────────┐
│ 步骤 4: 后续每次换页都触发重新模拟                             │
│                                                               │
│ finish_region(layouter, engine):                              │
│   self.regions.next();                                        │
│   self.simulate_header_footer_layout(layouter, engine)        │
│     → 再次执行步骤 1-3！                                      │
│                                                               │
│ → 每次换页都重新计算 header/footer 高度                       │
└───────────────────────────────────────────────────────────────┘
```

**Rowspan 模拟的 `regions.full` 风险**：
- ✅ **Header 跳页后重新模拟** → 高度始终与当前 regions 匹配
- ✅ **Footer 跳页后重新模拟** → 高度始终与当前 regions 匹配
- ✅ **每次换页都触发重新模拟** → 不存在"旧高度判断"问题
- **结论**：Rowspan 模拟对 `regions.full` 变化的**鲁棒性最好**，三者中最严谨

##### 三种场景的完整对照表

| 对比维度 | 普通不可断内容行 | 重复表头/表尾 | Rowspan 模拟 |
|---------|---------------|-------------|-------------|
| **模拟代码位置** | [`check_for_unbreakable_rows()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L237-L297) | [`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L203-L352) | [`simulate_header_footer_layout()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1150-L1244) |
| **换页方式** | `finish_region()` → 真实换页 | `finish_region_internal()` → 只推进 regions | `regions.next()` → 模拟器内部推进 |
| **换页时 regions.full 更新** | ✅ 真实更新 | ✅ 真实更新 | ✅ 模拟器内更新 |
| **Header 模拟在跳页后重算** | N/A（内容行无 header） | ❌ **不重算**（TODO 问题） | ✅ **重算** |
| **Footer 模拟在跳页后重算** | N/A（内容行无 footer） | ✅ 重算 | ✅ **重算** |
| **Rel 行实际解析时使用的 full** | ✅ 换页后的新 full | ✅ 换页后的新 full | N/A（纯模拟，无实际布局） |
| **标记 unbreakable_rows_left** | ✅ 模拟后设置，>0 锁换页 | ✅ 模拟后设置，>0 锁换页 | N/A（模拟器独立维护状态） |
| **实际布局期间能否换页** | ❌ 不能（锁换页） | ❌ 不能（锁换页） | N/A（纯模拟，无实际布局） |
| **Regions.full 变化风险** | ⭐⭐⭐ 低（Rel 行重新解析） | ⭐ 高（跳页判断用旧高度） | ⭐⭐⭐ 低（每次都重新模拟） |
| **溢出风险** | ✅ 低（模拟和布局都用新 full 重新解析） | ❌ **高**（预测量可能偏小，且锁换页） | N/A（模拟值偏大/偏小只影响 rowspan 扩展量，不溢出） |
| **多跳页风险** | ⭐ 低 | ⭐⭐ 中（预测量可能偏大） | N/A（模拟跳页不产生真实空白页） |

#### 3.6.6 可能的改进方向

基于代码中的 TODO 注释和上述分析，可能的改进包括：

1. **在跳页循环中更新 header 高度**（TODO 已经指出）：
   ```rust
   while !fits(header_height) {
       finish_region_internal();
       // TODO(layout model): 这里应该重新模拟 header
       header_height = simulate_header_height(new_regions);
       footer_height = simulate_footer(new_regions);
   }
   ```
   这将消除重复表头的溢出风险，使其与 Rowspan 模拟的严谨性对齐。

2. **增加溢出检查和降级处理**：
   - 在 `layout_header_rows()` 完成后，检查 `self.regions.size.y` 是否为负值（溢出）
   - 如果溢出且 `may_progress_with_repeats()` 为 true：
     - 回滚 header 行的布局（从 `lrows` 中弹出）
     - 调用 `finish_region()` 换页
     - 重试布局 header

3. **使用更保守的预测量**：
   - 在 `simulate_header_height()` 中，如果检测到当前 `regions.full` 是 `inf` 或异常大
   - 同时使用 `backlog` 中最小的高度再模拟一次
   - 取两者中的较大值作为预测量结果，避免预测量偏小

4. **统一三种场景的换页模拟模式**：
   - 参考 Rowspan 模拟的严谨做法，所有场景都在每次跳页后重新模拟 header/footer
   - 这将大大简化代码的心智模型，消除不一致的行为

---

## 四、换页触发时机与不可断行组

### 4.1 换页触发的多层级检查

换页不是在单一位置检查，而是分布在布局流程的多个阶段：

#### 4.1.1 行布局前的检查 - check_for_unbreakable_rows()

[`check_for_unbreakable_rows()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L237-L297)：

```rust
if self.unbreakable_rows_left == 0 {
    // 模拟不可断行组的高度
    let row_group = self.simulate_unbreakable_row_group(
        current_row, amount_unbreakable_rows, &self.regions, engine, 0,
    )?;

    // 不够则换页
    while !self.regions.size.y.fits(row_group.height)
        && self.may_progress_with_repeats()
    {
        self.finish_region(engine, false)?;
    }

    self.unbreakable_rows_left = row_group.rows.len();
}
```

**特点**：
- 提前预测整组的高度需求
- 避免组内部分行在当前页，部分在下一页

#### 4.1.2 Rel 行的换页 - layout_relative_row()

```rust
// Skip to fitting region, but only if we aren't part of an unbreakable
// row group.
while !self.regions.size.y.fits(resolved)
    && self.unbreakable_rows_left == 0
    && self.may_progress_with_repeats()
{
    self.finish_region(engine, false)?;
}
```

#### 4.1.3 Auto 行测量后的换页 - layout_auto_row()

```rust
let mut resolved = match self.measure_auto_row(
    engine, disambiguator, y, true, self.unbreakable_rows_left, None,
)? {
    Some(resolved) => resolved,
    None => {
        // 第一帧为空，换页重测
        self.finish_region(engine, false)?;
        self.measure_auto_row(
            engine, disambiguator, y, false, self.unbreakable_rows_left, None,
        )?.unwrap()
    }
};
```

#### 4.1.4 每行布局后的检查 - layout_row_internal()

```rust
let is_content_row = !self.grid.is_gutter_track(y);
if self.unbreakable_rows_left == 0 && self.regions.is_full() && is_content_row {
    self.finish_region(engine, false)?;
}
```

### 4.2 不可断行组的组成与扩展

#### 4.2.1 组的动态扩展

[`simulate_unbreakable_row_group()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L306-L368)：

```rust
for (y, row) in self.grid.rows.iter().enumerate().skip(first_row) {
    if amount_unbreakable_rows.is_none() {
        // 动态发现更多不可断行
        let additional_unbreakable_rows = self.check_for_unbreakable_cells(y);
        unbreakable_rows_left =
            unbreakable_rows_left.max(additional_unbreakable_rows);
    }
    if unbreakable_rows_left == 0 {
        break;
    }
    // ... 测量高度 ...
    unbreakable_rows_left -= 1;
}
```

**扩展机制**：
- 从当前行开始，每遇到一个不可断单元格（`breakable = false`），就将组扩展到其 rowspan 的最后一行
- 这样形成的组可能比预期的大，确保所有关联的不可断行都在同一页

#### 4.2.2 effectively_unbreakable 标记

[`check_for_unbreakable_rows()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L278-L294)：

```rust
if self.unbreakable_rows_left > 1 {
    for rowspan_data in
        self.rowspans.iter_mut().filter(|rowspan| rowspan.y == current_row)
    {
        rowspan_data.is_effectively_unbreakable |=
            self.unbreakable_rows_left >= rowspan_data.rowspan;
    }
}
```

**用途**：
- 标记某些 rowspan 为"实际上不可断"，即使它们本身设置了 `breakable = true`
- 当整个 rowspan 都在不可断行组内时，就没有必要运行复杂的换页模拟

---

## 五、线条 (Stroke) 和填充 (Fill) 渲染 - 绘制顺序详解

### 5.1 render_fills_strokes() 逐行解析

[`render_fills_strokes()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L467-L912) 是渲染的核心。以下是绘制顺序的详细分析：

#### 5.1.1 整体流程

```rust
fn render_fills_strokes(mut self) -> SourceResult<Fragment> {
    let mut finished = std::mem::take(&mut self.finished);
    for (((frame_index, frame), rows), finished_header_rows) in
        finished.iter_mut().enumerate().zip(&self.rrows).zip(...)
    {
        if self.rcols.is_empty() || rows.is_empty() {
            continue;
        }

        let mut lines = vec![];

        // 步骤 1: 生成所有垂直线段 (vlines)
        for (x, dx) in points(self.rcols.iter().copied()).enumerate() {
            // ... 生成垂直线段 ...
            lines.extend(segments);
        }

        // 步骤 2: 生成所有水平线段 (hlines)
        for ((i, y), dy) in hline_indices.zip(hline_offsets) {
            // ... 生成水平线段 ...
            lines.extend(segments);
        }

        // 步骤 3: 按厚度和优先级排序所有线条
        lines.sort_by_key(|(thickness, priority, ..)| (*thickness, *priority));

        // 步骤 4: 生成所有填充矩形 (fills)
        let mut fills = vec![];
        for (x, &col) in self.rcols.iter().enumerate() {
            for row in rows {
                // ... 生成填充 ...
                fills.push((pos, FrameItem::Shape(rect, self.span)));
            }
        }

        // 步骤 5: 先放入 fills，再放入 lines，统一 prepend 到 frame
        frame.prepend_multiple(
            fills
                .into_iter()
                .chain(lines.into_iter().map(|(_, _, point, shape)| (point, shape))),
        );
    }

    Ok(Fragment::frames(finished))
}
```

#### 5.1.2 最终层叠顺序（从下到上）

由于使用 `prepend_multiple`（添加到 frame 底部），且 iterator 顺序是 `fills` 在前、`lines` 在后，**最终的层叠顺序（从下到上）**是：

```
┌─────────────────────────────────────┐
│  5. 单元格内容 (Cell Content)        │  ← 最上层，prepend 之前已存在
├─────────────────────────────────────┤
│  4. 水平线条 (Horizontal Lines)      │  ← 后放入 lines，先 prepend → 在上
│     (按 thickness 排序，厚在上)        │
├─────────────────────────────────────┤
│  3. 垂直线条 (Vertical Lines)        │  ← 先放入 lines，后 prepend → 在下
│     (按 thickness 排序，厚在上)        │
├─────────────────────────────────────┤
│  2. 填充 (Fills)                     │  ← 先放入 fills，最后 prepend → 最底
├─────────────────────────────────────┤
│  1. Frame 背景 (透明)                │
└─────────────────────────────────────┘
```

**为什么 hline 在上，vline 在下？**

代码注释解释：
```rust
// Render vertical lines.
// Render them first so horizontal lines have priority later.
for (x, dx) in points(self.rcols.iter().copied()).enumerate() { ... }

// Render horizontal lines.
// They are rendered second as they default to appearing on top.
for ((i, y), dy) in hline_indices.zip(hline_offsets) { ... }
```

这是排版中的常见惯例：水平线默认在垂直线之上，形成"横线压竖线"的视觉效果。

#### 5.1.3 线条排序规则

```rust
// Sort by increasing thickness, so that we draw larger strokes
// on top. When the thickness is the same, sort by priority.
//
// Sorting by thickness avoids layering problems where a smaller
// hline appears "inside" a larger vline. When both have the same
// size, hlines are drawn on top (since the sort is stable, and
// they are pushed later).
lines.sort_by_key(|(thickness, priority, ..)| (*thickness, *priority));
```

**排序键（从小到大，后 prepend 的在上）**：
1. **第一键：thickness（线宽）** - 细线在下，粗线在上
2. **第二键：priority（优先级）** - 同粗细时，低优先级在下，高优先级在上
3. **稳定排序的隐式第三键：插入顺序** - 同粗细同优先级时，vline（先插入）在下，hline（后插入）在上

**排序示例**：

| 线条 | thickness | priority | 插入顺序 | 最终位置 |
|------|-----------|----------|----------|---------|
| vline (Grid) | 1pt | 0 | 1 | 最底 |
| vline (Cell) | 1pt | 1 | 2 | 第2层 |
| hline (Grid) | 1pt | 0 | 3 | 第3层 |
| hline (Cell) | 1pt | 1 | 4 | 第4层 |
| vline (Explicit) | 1pt | 2 | 5 | 第5层 |
| hline (Explicit) | 1pt | 2 | 6 | 第6层 |
| vline (thick) | 2pt | 0 | 7 | 第7层 |
| hline (thick) | 2pt | 0 | 8 | 最上 |

#### 5.1.4 填充生成的细节

```rust
let parent = self
    .grid
    .effective_parent_cell_position(x, row.y)
    .filter(|parent| {
        parent.x == x  // 只在单元格的第一列绘制填充
            && (parent.y == row.y
                || rows
                    .iter()
                    .find(|row| row.y >= parent.y)
                    .is_some_and(|first_spanned_row| {
                        first_spanned_row.y == row.y
                    }))
    });
```

**Rowspan 填充的特殊处理**：
- 使用 `effective_parent_cell_position` 而非 `parent_cell_position`
- 允许 gutter 行成为 rowspan 在当前页的"第一行"
- 确保 rowspan 的填充从当前页的最顶端开始，即使那是 gutter 行

---

## 六、关键算法和技巧总结

### 6.1 迭代式公平压缩 (Fair Shrink)

用于空间不足时压缩 Auto 列/行：
- **避免一次性平均压缩**导致小列被过度压缩
- **每轮识别无需压缩的列**，从压缩池中移除
- **收敛很快**：最多 O(n) 轮

### 6.2 Rowspan 预测模拟 (最多 5 次)

用于预测换页时 gutter 消失对 rowspan 高度的影响：
- **问题**：Auto 行的扩展高度决定了哪些行在哪一页，而分页又决定了哪些 gutter 被移除
- **方案**：假设一个扩展值 → 模拟分页 → 计算实际覆盖高度 → 调整扩展值 → 重复
- **收敛保证**：每轮扩展值严格递增，5 轮未收敛则降级为"所有 gutter 都消失"的保守估计
- **近似假设**：rowspan 内容是连续的单一高度，header/footer 高度不变

### 6.3 快照式孤儿预防

通过保存 `lrows` 的长度快照来实现 header 的孤儿预防：
- 无副作用：即使 header 已经完成布局和 frame 构建
- 回滚简单：只需要 `truncate(snapshot)` 截断数组即可
- 自动重试：header 仍保留在 `pending_headers` 中，下一页会自动重新布局

### 6.4 延迟式 Rowspan 布局

Rowspan 不随其起始行一起布局，而是"延迟"到最后一个跨越行完成时：
- **优势**：布局时已知道所有跨越行在各页的精确高度，可以一次性生成正确数量的 frame
- **挑战**：需要在 `finish_region()` 中逐行累计高度，维护 `heights` 向量和 `first_region` 等元数据
- **实现**：通过队列 `rowspans: Vec<Rowspan>` + 完成条件检查（`y + rowspan == current_y + 1 && is_last`）

### 6.5 分级降级策略

| 场景 | 降级策略 | 位置 |
|------|---------|------|
| rowspan 模拟 5 次不收敛 | 假设所有 gutter 被移除，保守扩展 | [rowspans.rs L800-L822](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L800-L822) |
| unbreakable rowspan 跨 auto 行 | 强制单页布局 | [layouter.rs L1299-L1314](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1299-L1314) |
| 无法换页但 header 放不下 | 放弃孤儿预防，强制放置 | [repeated.rs L345-L349](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L345-L349) |
| 跳过区域后 footer 高度变化 | 重新模拟 footer | [repeated.rs L502-L509](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L502-L509) |
| 第一帧只有 Tag | 换页重测 | [layouter.rs L1338-L1377](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1338-L1377) |

---

## 七、代码文件索引

| 文件 | 核心功能 |
|------|---------|
| [table.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/model/table.rs) | TableElem / TableCell / TableHeader 等元素的定义 |
| [resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/layout/grid/resolve.rs) | table_to_cellgrid / resolve_cellgrid，构建 CellGrid |
| [grid/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/mod.rs) | 布局入口 layout_table / layout_grid，layout_cell |
| [grid/layouter.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs) | GridLayouter 核心：列宽测量、行布局循环、区域完成、线条/填充渲染 |
| [grid/lines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/lines.rs) | 线段分段生成、stroke 折叠、优先级处理 |
| [grid/rowspans.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs) | Rowspan 布局、不可断行组模拟、rowspan 测量与预测 |
| [grid/repeated.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs) | Header/Footer 重复、孤儿预防、级别冲突、高度重算 |
