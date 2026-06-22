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

#### 3.5.1 何时触发高度重算

**场景 1：layout_active_headers 中跳过区域后**

[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L245-L256)：

```rust
if let Some(footer) = &self.grid.footer
    && footer.repeated
    && skipped_region
{
    // Simulate the footer again; the region's 'full' might have
    // changed.
    self.regions.size.y += self.current.footer_height;
    self.current.footer_height = self
        .simulate_footer(footer, &self.regions, engine, disambiguator)?
        .height;
    self.regions.size.y -= self.current.footer_height;
}
```

**触发条件**：
- 存在重复 footer
- 跳过了至少一个区域（`skipped_region = true`）

**重算原因**：
- 跳过区域意味着进入了新的页面，新页面的 `regions.full` 可能与前一页不同
- 例如：第一页是无限高度（如 float 容器），后续页面是有限高度

**场景 2：prepare_footer 中跳过区域后**

[`prepare_footer()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L502-L509) 同上。

#### 3.5.2 高度重算对布局的影响

**对 Auto 行测量的影响**：

在 [`prepare_auto_row_cell_measurement()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L402-L436) 中：

```rust
if breakable
    && (!self.repeating_headers.is_empty()
        || !self.pending_headers.is_empty()
        || matches!(&self.grid.footer, Some(footer) if footer.repeated))
{
    let mapped_regions = self.regions.map(&mut custom_backlog, |size| {
        Size::new(
            size.x,
            size.y
                - self.current.repeating_header_height  // 减去 header
                - self.current.footer_height,           // 减去 footer
        )
    });
}
```

**影响点**：
- Auto 行测量时，所有后续区域的高度都会预先减去 `repeating_header_height + footer_height`
- 如果这些高度被重算，测量结果会不同
- Rowspan 的 backlog 构造也依赖这些高度

**对 rowspan 模拟的影响**：

在 [`RowspanSimulator::new()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L1044-L1045) 中：

```rust
header_height: current.repeating_header_height,
footer_height: current.footer_height,
```

- 模拟时也使用这些高度来计算每页的可用空间
- 高度变化会影响模拟结果，进而影响 auto 行的扩展量

#### 3.5.3 高度重算的潜在问题

代码中有 TODO 注释指出了潜在问题：

[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L234-L238)：

```rust
// TODO(layout model): re-calculate heights of headers and footers
// on each region if 'full' changes? (Assuming height doesn't
// change for now...)
//
// Would remove the footer height update below (move it here).
```

**当前限制**：
- Header 高度不会在换页时重算，假设其高度不变
- 只有 Footer 高度在跳过区域后会重算
- 这可能导致 header 在不同页面有细微差异时布局不准确

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
