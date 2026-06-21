# Typst 表格布局机制深度解析

本文档基于 Typst 源码梳理表格（Table）和网格（Grid）的布局机制，重点聚焦于**行列分配**和**跨页处理**两个核心难点。

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

**关键处理逻辑**：

1. **换页检测**：如果第一个区域中某列单元格为空但后续区域有内容（说明内容从换页后开始），跳过第一个区域重新测量

2. **Rowspan 高度分配**：
   - 跨行单元格的高度同样只影响**最后一个被跨越的自动行**
   - 需要减去已经确定高度的其他跨越行（Rel 行等）
   - 如果跨越 gutter，且 gutter 可能因换页消失，则需要**模拟**

3. **Rowspan 模拟算法**：
   - 当 rowspan 跨越 gutter 时，需要预测换页情况
   - 使用 [`run_rowspan_simulation()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L879-L1002) 最多迭代 5 次
   - 每次模拟：假设 auto 行扩展 `amount_to_grow`，检查是否覆盖 rowspan 高度需求
   - 模拟稳定后，从尾部减去被其他行覆盖的高度

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

## 三、跨页处理机制

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

### 3.2 换页触发条件

在 [`layout_row_internal()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L425-L464) 中，换页在以下情况下发生：

```rust
let is_content_row = !self.grid.is_gutter_track(y);
if self.unbreakable_rows_left == 0 && self.regions.is_full() && is_content_row {
    self.finish_region(engine, false)?;
}
```

条件解读：
1. **当前不在不可断行组中** (`unbreakable_rows_left == 0`)
2. **区域已满** (`self.regions.is_full()`)
3. **是内容行而非 gutter 行**

此外，Rel 行和 Auto 行在测量阶段也可能主动换页。

### 3.3 不可断行组 (Unbreakable Row Group)

不可断行组确保相关的多行必须保持在同一页面中。

#### 形成条件

在 [`check_for_unbreakable_rows()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L237-L297) 中检查：

1. **不可断行 rowspan**：如果某个单元格 `breakable == false` 且 `rowspan > 1`，则其所有跨越行形成不可断行组
2. **Header/Footer**：所有 header 和 footer 行天然不可断
3. **非重复 footer**：也被视为不可断行组

#### 处理流程

```
simulate_unbreakable_row_group():
  从当前行开始，模拟高度累加:
    遇到不可断行 rowspan → 扩展组到其跨越的最后一行
    计算该组总高度
  如果当前区域放不下:
    循环 finish_region() 换页，直到放得下或无法继续
  设置 unbreakable_rows_left = 组内行数
```

换页后，`unbreakable_rows_left` 在每行处理后递减，确保组内行不会再次换页。

### 3.4 Header 重复机制

Header 的处理是跨页中最复杂的部分之一。

#### Header 状态机

一个 header 在其生命周期中可能处于以下状态：

```
                    首次发现 header
                         │
                         ▼
              ┌───  upcoming_headers  ───┐
              │   (等待被处理的所有 header)│
              └─────────────┬────────────┘
                            │ 被 place_new_headers() 处理
                            ▼
              ┌───  pending_headers  ────┐
              │(首次放置中，受孤儿预防约束)│
              └─────────────┬────────────┘
                            │ flush_orphans() 被调用
                            │ (有后续行放置，确认非孤儿)
                            ▼
              ┌─── repeating_headers  ───┐
              │  (将在每页顶部重复出现)   │
              └─────────────┬────────────┘
                            │ 遇到低级别的冲突 header
                            ▼
                      停止重复（被截断）
```

相关代码：[repeated.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs)

#### Header 级别 (Level) 冲突规则

每个 header 有一个 `level` 字段（默认 1，从 1 开始递增）：

- **低级 header → 替换高级**：当发现一个级别更低的新 header 时，所有相同或更高级别的旧 header 停止重复
- **并行共存**：不同级别的 header 可以同时重复（前提是级别严格递增）
- **短命 header (short_lived)**：如果一个 header 后紧跟相同或更低级别的 header，则标记为短命，不进行重复和孤儿预防

#### 孤儿预防 (Orphan Prevention)

为避免 header 单独出现在页面底部（后面没有内容），使用以下机制：

1. 首次放置 header 后，保存 `lrows_orphan_snapshot = 当前行数`
2. 如果在 `flush_orphans()` 前就触发 `finish_region()`，且不是最后一页：
   - 将 `current.lrows` 截断到快照位置（移除刚刚放置的 header）
3. 当后续有任何行被成功放置后，调用 `flush_orphans()`，清除快照

**效果**：如果 header 后没有任何内容行就换页了，header 会被"收回"，在下一页重新尝试。

#### 区域顶部的 Header 放置

在 [`finish_region()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L1599-L1834) 末尾，准备下一页时：

```rust
if !self.repeating_headers.is_empty() || !self.pending_headers.is_empty() {
    self.layout_active_headers(engine)?;
}
```

[`layout_active_headers()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs#L203-L352)：

1. 模拟 header 高度，不够则换页
2. 将 header 行标记为不可断行组
3. 依次布局 `repeating_headers`（标记 `is_being_repeated = true`）和 `pending_headers`
4. 重置 `repeating_header_height` 等统计量，供 Auto 行测量使用

### 3.5 Footer 重复机制

Footer 相对简单，关键要点：

#### 准备阶段

在主布局循环开始前，如果有重复 footer：

```rust
if let Some(footer) = &self.grid.footer && footer.repeated {
    self.prepare_footer(footer, engine, 0)?;
    self.regions.size.y -= self.current.footer_height;
}
```

**预先扣除 footer 高度**，确保后续行不会侵占该空间。每次换页后在 `finish_region()` 末尾重新准备。

#### Widow 预防

如果 footer 前没有任何内容行（header 不算），且可以换页，则不放置 footer，避免 footer 成为"寡妇"单独出现在页面上。

```rust
let footer_would_be_widow = matches!(&self.grid.footer, Some(footer) if footer.repeated)
    && self.current.lrows.is_empty()
    && self.current.could_progress_at_top;
```

#### 最终布局

在主循环中，到达 footer 起始行时：

```rust
if y == footer.start {
    self.layout_footer(footer, engine, self.finished.len(), false)?;
}
```

`is_being_repeated=false` 表示这是 footer 的最终真实出现（之前每页顶部的预留空间是模拟布局，最终在 `finish_region()` 中真正放置）。

### 3.6 Rowspan 的跨页处理

Rowspan（跨行单元格）是跨页中最棘手的问题。

#### Rowspan 数据结构

```rust
pub struct Rowspan {
    pub x: usize,                    // 起始列
    pub y: usize,                    // 起始行
    pub rowspan: usize,              // 跨越行数
    pub is_effectively_unbreakable: bool,  // 是否实际上不可断
    pub dx: Abs,                     // 水平偏移
    pub dy: Abs,                     // 第一页的垂直偏移
    pub first_region: usize,         // 首次出现的区域索引
    pub region_full: Abs,            // 首个区域的完整高度
    pub heights: Vec<Abs>,           // 每页的累计高度
    pub max_resolved_row: Option<usize>, // 已处理的最大行号
    pub is_being_repeated: bool,     // 是否为重复 header 中的 rowspan
}
```

#### 生命周期

1. **发现阶段**：在 `check_for_rowspans()` 中，当处理 rowspan 的起始行时，将其加入 `rowspans` 队列，设置 `x, y, rowspan, dx`

2. **高度累计阶段**：在 `finish_region()` 中，对每个已完成的行，更新所有跨越该行的 rowspan 的 `heights` 数组

3. **布局触发**：当一行是某个 rowspan 的最后一个跨越行（且为该行最后一帧）时，调用 `layout_rowspan()`，将其从队列中移除

4. **兜底阶段**：所有行处理完后，处理可能遗漏的 rowspan（如最后一行为空 Auto 行的情况）

#### 多页渲染

在 [`layout_rowspan()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs#L103-L199) 中：

```rust
for ((i, (finished, header_dy)), frame) in ... {
    let dy = if i == 0 {
        dy  // 第一页：原始垂直位置
    } else {
        header_dy  // 后续页：从 header 下方开始
    };
    finished.push_frame(Point::new(dx, dy), frame);
}
```

关键点：后续页的 rowspan 从 **header 下方**开始，避免与重复 header 重叠。

---

## 四、线条 (Stroke) 和填充 (Fill) 渲染

### 4.1 线段优先级

绘制线条时，有三级优先级（影响同厚度线条的上下层关系）：

| 优先级 | 来源 | 说明 |
|-------|------|------|
| 0 (最低) | GridStroke | 表格/网格的全局 `stroke` 设置 |
| 1 | CellStroke | 单元格级别的 `stroke` 覆盖 |
| 2 (最高) | ExplicitLine | 显式的 `hline` / `vline` 元素 |

相关代码：[lines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/lines.rs#L10-L26) 中的 `StrokePriority`

### 4.2 线段分段生成

[`generate_line_segments()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/lines.rs#L79-L253) 是线条渲染的核心：

**核心思路**：逐轨道（行/列）推进，连续相同 stroke 的轨道合并为一个线段，遇到以下情况则断开：

1. **遇到合并单元格**：例如 vline 穿过 colspan 区域时必须断开（因为该区域不存在实际分隔线）

   ```
   vline at x=2 穿过 row y=1:
     检查 cell(x=2,y=1) 的 parent，如果 parent.x < 2 → 说明有 colspan → 跳过
   ```

2. **Stroke 变化**：线条属性（颜色、粗细）或优先级变化

3. **用户显式 hline/vline**：指定 `start/end` 范围的显式线条

### 4.3 线条折叠 (Folding) 规则

线条的最终 stroke 通过**折叠**（fold）多层来源决定，优先级从高到低：

#### 垂直线 (vline) 折叠顺序：

```
显式 vline.stroke  →  右侧单元格.left  →  左侧单元格.right  →  全局 stroke
```

#### 水平线 (hline) 折叠顺序：

```
显式 hline.stroke
  → 下方单元格.top (或底部 border / footer 顶部)
  → 上方单元格.bottom (或顶部 border / header 底部)
  → 全局 stroke
```

折叠规则（Sides::fold）：
- `None` (未指定) ← `Some(x)` → 使用 `x`
- `Some(None)` (指定为 none) ← `Some(Some(y))` → 使用 `None`
- `Some(Some(a))` ← `Some(Some(b))` → 合并 stroke 属性

### 4.4 跨页时的特殊线条处理

#### 顶部边框优先级提升

在非首页的区域顶部，原表格的顶部边框线条会获得额外优先级（仿佛是新的 header 线），确保页面顶部有清晰的边界。

#### Header 下方线条优先级

当 header 被重复时，原 header 最后一行下方的线条被提升优先级，确保其覆盖住页面内部正常的线条。

#### 底部边框处理

非尾页的底部边框同样提升优先级，但尾页的底部边框按正常规则处理。

### 4.5 单元格填充 (Fill)

填充在 [`render_fills_strokes()`](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs#L467-L912) 中处理，先于线条绘制。

填充的关键规则：
- **Rowspan 填充**：从该 rowspan 在当前区域的**第一个实际跨越行**开始，不是逻辑起始行（因为前面的行可能在另一页或被删除）
- **Colspan 填充**：仅在第一列绘制一次，宽度跨越所有 colspan 列
- **Gutter 处理**：如果 gutter 行是 rowspan 在该页的第一行，则从 gutter 行开始填充

---

## 五、关键算法和技巧总结

### 5.1 迭代式公平压缩 (Fair Shrink)

用于空间不足时压缩 Auto 列/行：
- **避免一次性平均压缩**导致小列被过度压缩
- **每轮识别无需压缩的列**，从压缩池中移除
- **收敛很快**：最多 O(n) 轮

### 5.2 Rowspan 预测模拟 (最多 5 次)

用于预测换页时 gutter 消失对 rowspan 高度的影响：
- **问题**：Auto 行的扩展高度决定了哪些行在哪一页，而分页又决定了哪些 gutter 被移除
- **方案**：假设一个扩展值 → 模拟分页 → 计算实际覆盖高度 → 调整扩展值 → 重复
- **收敛保证**：每轮扩展值严格递增，5 轮未收敛则降级为"所有 gutter 都消失"的保守估计

### 5.3 快照式孤儿预防

通过保存 `lrows` 的长度快照来实现 header 的孤儿预防：
- 无副作用：即使 header 已经完成布局和 frame 构建
- 回滚简单：只需要 `truncate(snapshot)` 截断数组即可
- 自动重试：header 仍保留在 `pending_headers` 中，下一页会自动重新布局

### 5.4 延迟式 Rowspan 布局

Rowspan 不随其起始行一起布局，而是"延迟"到最后一个跨越行完成时：
- **优势**：布局时已知道所有跨越行在各页的精确高度，可以一次性生成正确数量的 frame
- **挑战**：需要在 `finish_region()` 中逐行累计高度，维护 `heights` 向量和 `first_region` 等元数据
- **实现**：通过队列 `rowspans: Vec<Rowspan>` + 完成条件检查（`y + rowspan == current_y + 1 && is_last`）

---

## 六、代码文件索引

| 文件 | 核心功能 |
|------|---------|
| [table.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/model/table.rs) | TableElem / TableCell / TableHeader 等元素的定义 |
| [resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-library/src/layout/grid/resolve.rs) | table_to_cellgrid / resolve_cellgrid，构建 CellGrid |
| [grid/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/mod.rs) | 布局入口 layout_table / layout_grid，layout_cell |
| [grid/layouter.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/layouter.rs) | GridLayouter 核心：列宽测量、行布局循环、区域完成、线条/填充渲染 |
| [grid/lines.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/lines.rs) | 线段分段生成、stroke 折叠、优先级处理 |
| [grid/rowspans.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/rowspans.rs) | Rowspan 布局、不可断行组模拟、rowspan 测量与预测 |
| [grid/repeated.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/124-typst/crates/typst-layout/src/grid/repeated.rs) | Header/Footer 重复、孤儿预防、级别冲突 |
