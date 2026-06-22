# Typst 字体处理链路分析

本文档逐段分析 Typst 中字体发现（Font Discovery）、字体回退（Font Fallback）和字体子集化（Font Subsetting）之间的衔接关系。

---

## 一、整体架构概览

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  字体发现       │────▶│  字体回退       │────▶│  字体子集化     │
│  (Discovery)    │     │  (Fallback)     │     │  (Subsetting)   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
          │                       │                       │
          ▼                       ▼                       ▼
    FontBook + FontStore     ShapedGlyph + TextItem     krilla::Font
    元数据索引 + 惰性加载    字形ID + 字体实例         PDF嵌入子集化字体
```

---

## 二、字体发现 (Font Discovery)

### 2.1 核心代码位置
- [typst-kit/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs)
- [typst-cli/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-cli/src/fonts.rs)

### 2.2 核心数据结构

#### `FontStore` - 字体存储
[FontStore](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs#L24-L77) 是字体的中央存储：

```rust
pub struct FontStore {
    book: LazyHash<FontBook>,    // 字体元数据索引
    slots: Vec<FontSlot>,        // 字体槽位，惰性加载
}
```

- `book`：字体元数据，用于快速查询和选择
- `slots`：实际字体数据，通过 `FontSlot` 实现惰性加载

#### `FontSlot` - 惰性加载机制
[FontSlot](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs#L86-L97) 使用 `OnceLock` 实现按需加载：

```rust
struct FontSlot {
    source: Box<dyn FontSource>,  // 字体来源（路径/内存）
    font: OnceLock<Option<Font>>, // 实际字体，首次访问时加载
}

impl FontSlot {
    fn get(&self) -> Option<Font> {
        self.font.get_or_init(|| self.source.load()).clone()
    }
}
```

#### `FontBook` - 字体元数据索引
[FontBook](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs#L13-L163) 是字体选择的核心索引：

```rust
pub struct FontBook {
    families: BTreeMap<String, Vec<usize>>,  // 家族名 -> 字体索引列表
    infos: Vec<FontInfo>,                    // 每个字体的元数据
}
```

### 2.3 字体发现流程

1. **入口函数** [discover_fonts](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-cli/src/fonts.rs#L38-L55)
   ```rust
   pub fn discover_fonts(args: &FontArgs) -> FontStore {
       let mut fonts = FontStore::new();
       if !args.ignore_system_fonts {
           fonts.extend(fonts::system());      // 扫描系统字体
       }
       if !args.ignore_embedded_fonts {
           fonts.extend(fonts::embedded());    // 加载嵌入式字体
       }
       for path in &args.font_paths {
           fonts.extend(fonts::scan(path));   // 扫描指定目录
       }
       fonts
   }
   ```

2. **系统字体扫描** [system](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs#L148-L157) / [scan](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs#L163-L166)
   - 使用 `fontdb` 库扫描操作系统标准字体位置
   - 递归扫描用户指定目录
   - 对每个字体文件解析 `FontInfo`（家族名、变体、覆盖率等）

3. **构建索引** [with_db](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs#L170-L194)
   ```rust
   fn with_db(f: impl FnOnce(&mut fontdb::Database)) 
       -> impl Iterator<Item = (FontPath, FontInfo)> {
       let mut db = fontdb::Database::new();
       f(&mut db);
       db.faces().filter_map(|face| {
           // 提取字体路径和索引
           let path = FontPath { path: path.clone(), index: face.index };
           // 解析 FontInfo（包含 Unicode 覆盖率）
           let info = db.with_face_data(face.id, FontInfo::new)?;
           Some((path, info))
       })
   }
   ```

### 2.4 衔接点
- 字体发现阶段构建的 `FontBook` 被传递给 `World` trait 实现
- 通过 `World::book()` 和 `World::font()` 接口供后续阶段使用

---

## 三、字体回退 (Font Fallback)

### 3.1 核心代码位置
- [typst-library/src/text/font/book.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs) - 字体选择逻辑
- [typst-layout/src/inline/shaping.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs) - 字形处理和回退触发

### 3.2 字体选择算法

#### `FontBook::select()` - 按家族选择
[select](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs#L75-L78) 根据家族名和变体选择最佳匹配：

```rust
pub fn select(&self, family: &str, variant: FontVariant) -> Option<usize> {
    let ids = self.families.get(family)?;
    self.find_best_variant(None, variant, ids.iter().copied())
}
```

#### `FontBook::select_fallback()` - 全局回退
[select_fallback](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs#L94-L115) 是最后一道防线：

```rust
pub fn select_fallback(
    &self,
    like: Option<&FontInfo>,    // 参考字体（用于相似性匹配）
    variant: FontVariant,       // 目标变体
    text: &str,                 // 需要显示的文本
) -> Option<usize> {
    // 1. 找到文本中第一个非空白、非默认忽略字符
    let c = text.chars().find(|&c| !c.is_whitespace() && !is_default_ignorable(c))?;
    
    // 2. 过滤出所有包含该字符的字体
    let ids = self.infos.iter().enumerate()
        .filter(|(_, info)| info.coverage.contains(c as u32))
        .map(|(index, _)| index);
    
    // 3. 在候选字体中找到最佳变体
    self.find_best_variant(like, variant, ids)
}
```

#### `find_best_variant()` - 最佳匹配评分
[find_best_variant](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs#L139-L163) 使用多维度评分：

```rust
fn find_best_variant(
    &self,
    like: Option<&FontInfo>,
    variant: FontVariant,
    ids: impl IntoIterator<Item = usize>,
) -> Option<usize> {
    for id in ids {
        let current = &self.infos[id];
        let score = (
            // 1. 与参考字体的相似性（用于回退场景）
            like.map(|like| similarity(current, like)),
            // 2. 与目标变体的距离（反向排序）
            Reverse(distance(current, variant)),
            // 3. 优先选择可变字体
            current.flags.contains(FontFlags::VARIABLE),
        );
        // 选择最高分的字体
    }
}
```

**相似性评分** [similarity](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs#L168-L185)：
```rust
fn similarity(left: &FontInfo, right: &FontInfo) -> impl Ord {
    (
        // 等宽字体匹配
        left.flags.contains(FontFlags::MONOSPACE) 
            == right.flags.contains(FontFlags::MONOSPACE),
        // 衬线字体匹配
        left.flags.contains(FontFlags::SERIF) 
            == right.flags.contains(FontFlags::SERIF),
        // 家族名前缀共享词数（如 "Noto Sans" vs "Noto Sans Arabic"）
        shared_prefix_words(&left.family, &right.family),
        // 偏好更短的家族名（更通用）
        Reverse(left.family.len()),
    )
}
```

### 3.3 字形处理中的回退触发

#### `shape_segment()` - 核心回退逻辑
[shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L956-L1158) 是字形处理和回退的核心：

```rust
fn shape_segment<'a>(
    ctx: &mut ShapingContext<'a>,
    base: usize,
    text: &str,
    mut families: impl Iterator<Item = &'a FontFamily> + Clone,
) {
    // 步骤1: 获取字体（可能触发回退）
    let Some((font, covers)) = get_font_and_covers(
        ctx, text, families.by_ref(), 
        |ctx, text, font| shape_tofus(ctx, base, text, &font)  // 连豆腐块都没字体时的兜底
    ) else {
        return;
    };

    // 步骤2: 使用 rustybuzz 进行字形处理
    let buffer = rustybuzz::shape_with_plan(font.rusty(), &plan, buffer);
    let infos = buffer.glyph_infos();

    // 步骤3: 遍历处理后的字形，检查缺失
    let mut i = 0;
    while i < infos.len() {
        let info = &infos[i];
        
        // 正常字形：glyph_id != 0 且在覆盖范围内
        if info.glyph_id != 0 && is_covered(cluster) {
            // 添加到输出
            ctx.glyphs.push(ShapedGlyph { ... });
        } else {
            // 触发回退：找到连续的 tofu 序列
            let start = /* tofu 序列起始 */;
            let end = /* tofu 序列结束 */;
            
            // 移除已添加的不完整字形
            while ctx.glyphs.last().is_some_and(...) {
                ctx.glyphs.pop();
            }
            
            // 递归：用下一个字体重新处理这段文本
            shape_segment(ctx, base + start, &text[start..end], families.clone());
        }
        i += 1;
    }
    
    ctx.used.pop();  // 字体使用完毕，从已用列表移除
}
```

#### `get_font_and_covers()` - 字体获取与回退
[get_font_and_covers](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L902-L953) 封装了字体选择和回退逻辑：

```rust
pub fn get_font_and_covers<'a, C, F>(
    ctx: &mut C,
    text: &str,
    mut families: impl Iterator<Item = &'a FontFamily>,
    mut shape_tofus: F,
) -> Option<(FontInstance, Option<&'a Regex>)> {
    // 阶段1: 尝试用户指定的字体列表
    for family in families.by_ref() {
        selection = book.select(family.as_str(), ctx.variant())
            .and_then(|id| world.font(id))
            .map(|font| font.instantiate(...))
            .filter(|font| !ctx.used().contains(font));
        if selection.is_some() { break; }
    }

    // 阶段2: 如果用户字体都不行且 fallback 启用，尝试全局回退
    if selection.is_none() && ctx.fallback() {
        let first = ctx.first().map(|font| font.info());
        selection = book.select_fallback(first, ctx.variant(), text)
            .and_then(|id| world.font(id))
            .map(|font| font.instantiate(...))
            .filter(|font| !ctx.used().contains(font));
    }

    // 阶段3: 连回退都找不到，显示豆腐块
    let Some(font) = selection else {
        if let Some(font) = ctx.used().first().cloned() {
            shape_tofus(ctx, text, font);
        }
        return None;
    };

    // 记录已使用的字体（无覆盖限制的字体不会被重复使用）
    if covers.is_none() {
        ctx.used().push(font.clone());
    }

    Some((font, covers))
}
```

### 3.4 字体列表来源
[families](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs#L1083-L1099) 函数构建完整的字体优先级列表：

```rust
pub fn families(styles: StyleChain<'_>) 
    -> impl Iterator<Item = &'_ FontFamily> + Clone {
    // 内置回退字体列表
    let fallbacks = singleton!(Vec<FontFamily>, {
        [
            "libertinus serif",
            "twitter color emoji",
            "noto color emoji",
            "apple color emoji",
            "segoe ui emoji",
        ].into_iter().map(FontFamily::new).collect()
    });

    // 用户字体 + （如果启用 fallback）内置回退字体
    let tail = if styles.get(TextElem::fallback) { 
        fallbacks.as_slice() 
    } else { 
        &[] 
    };
    styles.get_ref(TextElem::font).into_iter().chain(tail.iter())
}
```

### 3.5 衔接点
- 输入：字体发现阶段构建的 `FontBook` 和 `FontStore`
- 输出：`ShapedGlyph` 列表，每个 glyph 包含：
  - `font: FontInstance` - 实际使用的字体实例（含变体坐标）
  - `glyph_id: u16` - 字体中的字形索引
  - `x_advance`, `x_offset` 等布局信息
- 这些 `ShapedGlyph` 最终被组装成 `TextItem`，传递给渲染/导出阶段

---

## 四、字体子集化 (Font Subsetting)

### 4.1 核心代码位置
- [typst-pdf/src/text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs)
- 子集化实际由 `krilla` 库（Typst 的 PDF 生成库）完成

### 4.2 字体转换

#### `handle_text()` - 文本处理入口
[handle_text](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L18-L60) 将 Typst 文本项转换为 PDF 绘制命令：

```rust
pub(crate) fn handle_text(
    fc: &mut FrameContext,
    t: &TextItem,           // 来自回退阶段的输出
    surface: &mut Surface,
    gc: &mut GlobalContext,
) -> SourceResult<()> {
    // 1. 转换字体（带缓存）
    let font = convert_font(gc, t.font.clone())?;
    
    // 2. 转换填充和描边样式
    let fill = paint::convert_fill(...);
    let stroke = ...;
    
    // 3. 绘制字形
    surface.draw_glyphs(
        krilla::geom::Point::from_xy(0.0, 0.0),
        glyphs,         // Glyph 迭代器（实现了 krilla::text::Glyph trait）
        font.clone(),   // krilla 字体对象
        text,           // 原始文本（用于 PDF 文本提取）
        size.to_f32(),  // 字体大小
        false,
    );
    
    Ok(())
}
```

#### `convert_font()` - 字体转换与缓存
[convert_font](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L62-L76) 避免重复转换相同字体：

```rust
fn convert_font(
    gc: &mut GlobalContext,
    typst_font: FontInstance,
) -> SourceResult<krilla::text::Font> {
    // 缓存命中直接返回
    if let Some(font) = gc.fonts_forward.get(&typst_font) {
        return Ok(font.clone());
    }
    
    // 构建 krilla 字体
    let font = build_font(typst_font.clone())?;
    
    // 双向缓存（正向和反向映射）
    gc.fonts_forward.insert(typst_font.clone(), font.clone());
    gc.fonts_backward.insert(font.clone(), typst_font.clone());
    
    Ok(font)
}
```

#### `build_font()` - 构建可变字体实例
[build_font](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L79-L104) 创建带变体坐标的字体：

```rust
#[comemo::memoize]
fn build_font(typst_font: FontInstance) -> SourceResult<krilla::text::Font> {
    // 共享字体数据
    let font_data: Arc<dyn AsRef<[u8]> + Send + Sync> = 
        Arc::new(typst_font.data().clone());
    
    // 转换变体坐标
    let variations = typst_font.variations().0.iter()
        .map(|(tag, value)| (krilla::text::Tag::new(&tag.to_bytes()), value.0))
        .collect::<Vec<_>>();
    
    // 创建可变字体（krilla 内部管理子集化）
    krilla::text::Font::new_variable(
        font_data.into(),
        typst_font.index(),
        &variations,
    )
}
```

### 4.3 子集化原理

#### Krilla 的自动子集化
当调用 `surface.draw_glyphs()` 时，krilla 内部会：

1. **记录字形使用**：每个 `glyph_id` 被记录在该字体的使用集合中
2. **按需子集化**：在最终序列化 PDF 时，krilla 会：
   - 遍历所有使用过该字体的页面
   - 收集所有被引用的 `glyph_id`
   - 从原始字体文件中提取这些字形及依赖的表
   - 构建子集化字体嵌入到 PDF

#### `PdfGlyph` - 字形桥接
[PdfGlyph](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L106-L148) 实现了 krilla 的 `Glyph` trait：

```rust
#[derive(Debug, TransparentWrapper)]
#[repr(transparent)]
struct PdfGlyph(Glyph);  // Glyph 来自回退阶段的输出

impl krilla::text::Glyph for PdfGlyph {
    #[inline(always)]
    fn glyph_id(&self) -> GlyphId {
        GlyphId::new(self.0.id as u32)  // 关键：传递字形ID给krilla
    }
    
    fn text_range(&self) -> Range<usize> {
        self.0.range.start as usize..self.0.range.end as usize
    }
    
    // ... 其他方法：x_advance, x_offset, y_offset, y_advance
}
```

### 4.4 全局上下文的字体管理
[GlobalContext](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs#L278-L281) 维护字体映射：

```rust
pub(crate) struct GlobalContext<'a> {
    // Typst FontInstance -> Krilla Font（正向查找）
    pub(crate) fonts_forward: FxHashMap<FontInstance, krilla::text::Font>,
    // Krilla Font -> Typst FontInstance（反向查找，用于错误报告）
    pub(crate) fonts_backward: FxHashMap<krilla::text::Font, FontInstance>,
    // ...
}
```

### 4.5 衔接点
- 输入：字体回退阶段产生的 `TextItem`，包含：
  - `font: FontInstance` - 确定了具体使用的字体和变体
  - `glyphs: Vec<Glyph>` - 每个字符对应的 `glyph_id`
- 输出：嵌入到 PDF 中的子集化字体，只包含实际使用的字形

---

## 五、数学公式字体回退链路

### 5.1 数学公式的字体设置

数学公式的字体处理与普通文本有本质区别。[EquationElem::show_set](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/math/equation.rs#L186-L204) 为数学公式设置了专用的字体：

```rust
impl ShowSet for Packed<EquationElem> {
    fn show_set(&self, styles: StyleChain) -> Styles {
        let mut out = Styles::new();
        // ... 其他设置
        out.set(TextElem::weight, FontWeight::from_number(450));
        out.set(
            TextElem::font,
            FontList(vec![FontFamily::new("New Computer Modern Math")]),
        );
        out
    }
}
```

**关键点**：
- 数学公式默认使用 `"New Computer Modern Math"` 字体（单字体列表）
- 字重设置为 450（略粗于常规 400）
- 没有设置 `covers` 限制，因此这个字体会被加入 `used` 列表

### 5.2 数学变体转换：字符级别的"回退"

数学公式的字体"回退"主要不是在字形处理阶段发生，而是发生在更早的**IR 解析阶段**。通过 [codex](https://github.com/typst/codex) 库的 `to_style` 函数，普通 Unicode 字符被转换为**数学 Unicode 变体区域**的字符。

[resolve_text](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/math/ir/resolve.rs#L272-L306) 中的转换：

```rust
fn resolve_text(...) {
    let variant = styles.get(EquationElem::variant);  // 如 DoubleStruck, SansSerif
    let bold = styles.get(EquationElem::bold);
    let italic = styles.get(EquationElem::italic).or(Some(false));

    let styled_text: EcoString = text
        .chars()
        .flat_map(|c| to_style(c, MathStyle::select(c, variant, bold, italic)))
        .collect();
    // ... 创建 TextItem 或 NumberItem
}
```

#### `MathStyle::select` 的变体选择

| 函数 | 效果 | 字符变化示例 |
|------|------|-------------|
| `math.bold(A)` | `variant=None, bold=true` | `A` (U+0041) → `𝐀` (U+1D400) |
| `math.italic(A)` | `variant=None, italic=true` | `A` (U+0041) → `𝐴` (U+1D434) |
| `math.bb(A)` | `variant=DoubleStruck` | `A` (U+0041) → `𝔸` (U+1D538) |
| `math.sans(A)` | `variant=SansSerif` | `A` (U+0041) → `𝖠` (U+1D5A0) |
| `math.frak(A)` | `variant=Fraktur` | `A` (U+0041) → `𝔄` (U+1D504) |

**设计意图**：
- 数学字体（如 New Computer Modern Math）在数学 Unicode 区域（U+1D400–U+1D7FF）提供了各种变体字形
- 通过预先转换字符，而不是在字体回退阶段处理，确保数学符号的一致性
- 这是"字符级转换"而非"字体级回退"

### 5.3 数学文本的 layout 流程

数学文本的 layout 发生在 [typst-layout/src/math/text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/math/text.rs) 中：

```rust
pub fn layout_text(...) {
    let text = &item.text;  // 已转换为数学 Unicode 变体
    let elem = TextElem::packed(text).spanned(span);
    
    // 通过 layout_inline 进入普通文本 shaping 流程
    let frame = crate::inline::layout_inline(
        ctx.engine,
        &[(&elem, styles)],
        ...
    )?;
}
```

**完整链路**：

```
数学公式输入: $ math.bb(A) + math.bold(B) $
        │
        ▼
EquationElem::show_set()
  ├─ 设置 font = ["New Computer Modern Math"]
  └─ 设置 weight = 450
        │
        ▼
math/ir/resolve.rs: resolve_text() / resolve_symbol()
  ├─ variant = styles.get(EquationElem::variant)
  ├─ bold = styles.get(EquationElem::bold)
  ├─ italic = styles.get(EquationElem::italic)
  └─ to_style(c, MathStyle::select(c, variant, bold, italic))
        │
        ├─ 'A' → '𝔸' (U+1D538, DoubleStruck)
        ├─ '+' → '+' (运算符，无变体转换)
        └─ 'B' → '𝐁' (U+1D401, Bold)
        │
        ▼
layout_inline() → shape_range() → shape() → shape_segment()
  ├─ families = ["New Computer Modern Math"]
  ├─ 检查字体是否包含 𝔸 (U+1D538)
  ├─ 如果包含 → 正常输出字形
  └─ 如果不包含 → 触发字体回退（进入普通回退链路）
        │
        ├─ 阶段1: 遍历用户字体列表（只有 "New Computer Modern Math"）
        ├─ 阶段2: 如果 fallback=true，select_fallback() 查找包含该字符的字体
        └─ 阶段3: 如果都失败，shape_tofus() 显示豆腐块
        │
        ▼
ShapedText.build() → TextItem → PDF 子集化
  └─ 与普通文本相同的子集化流程
```

### 5.4 数学字体回退的特殊性

1. **字体列表单一**：数学公式默认只有 `"New Computer Modern Math"` 一个字体
2. **回退发生晚**：先尝试字符转换，转换后才进入字形处理回退
3. **数学区域字符特殊**：U+1D400–U+1D7FF 区域的字符通常只有专业数学字体才包含
4. **连字处理**：数学字体可能有特殊的连字（如 `:=` → `≔`），通过 `flac` OpenType 特性处理

### 5.5 数学字体特性：`dtls` 和 `flac`

[layout_glyph](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/math/text.rs#L69-L119) 中处理了数学字体的特殊特性：

- **`dtls` (Dotless Forms)**：用于将 `i` 和 `j` 转换为无点形式 `ı` 和 `ȷ`，避免与上方重音冲突
- **`flac` (Flattened Accents)**：用于压扁上方的重音符号

这些 OpenType 特性在数学字体中很常见，确保数学符号的正确显示。

---

## 六、covers 与 used 的 push/pop 栈关系

### 6.1 `used` 列表的生命周期

`used` 是 `ShapingContext` 中的一个 `Vec<FontInstance>`，行为上像一个**栈**，用于追踪"已用尽"的字体（没有 `covers` 限制的字体）。

```rust
struct ShapingContext<'a> {
    used: Vec<FontInstance>,  // 栈：push 在 get_font_and_covers，pop 在 shape_segment 末尾
    // ...
}
```

### 6.2 完整的栈操作时序

让我们以文本 `"中A文"`，字体列表 `[(name: "Inria Serif", covers: "latin-in-cjk"), "Noto Serif CJK SC"]` 为例：

```
初始状态: used = []

shape_segment(ctx, 0, "中A文", families)
  │
  ├─▶ get_font_and_covers()
  │     ├─ family1: "Inria Serif", covers: latin-in-cjk
  │     │     ├─ book.select() → 找到字体
  │     │     ├─ covers = Some(regex)  ← 有覆盖限制
  │     │     └─ 不 push 到 used  ← 关键：有 covers 不加入 used
  │     │
  │     └─ 返回 (font_inria, Some(latin_in_cjk_regex))
  │
  ├─ rustybuzz.shape_with_plan(font_inria, ...)
  │     └─ infos: [glyph_id=0(cluster=0), glyph_id=120(cluster=1), glyph_id=0(cluster=2)]
  │                "中" (tofu)           "A" (ok)            "文" (tofu)
  │
  └─ 遍历 glyphs:
        │
        ├─ i=0: glyph_id=0, "中"
        │     │
        │     ├─ is_covered(0)?
        │     │     └─ "中" 不在 latin-in-cjk 的覆盖范围 → false
        │     │
        │     ├─ 找到连续 tofu 序列: [0..1] ("中")
        │     ├─ 递归调用: shape_segment(ctx, 0, "中", families.clone())
        │     │     │
        │     │     ├─▶ get_font_and_covers()
        │     │     │     ├─ family1: "Inria Serif"
        │     │     │     │     ├─ !used.contains(font_inria) → true  ← 仍可用（因为没 push）
        │     │     │     │     └─ 但是 "中" 不在覆盖范围 → 继续
        │     │     │     │
        │     │     │     ├─ family2: "Noto Serif CJK SC", covers: None
        │     │     │     │     ├─ book.select() → 找到字体
        │     │     │     │     ├─ covers = None  ← 无覆盖限制
        │     │     │     │     └─ ctx.used().push(font_cjk)  ← push!
        │     │     │     │        used = [font_cjk]
        │     │     │     │
        │     │     │     └─ 返回 (font_cjk, None)
        │     │     │
        │     │     ├─ rustybuzz.shape_with_plan(font_cjk, ...)
        │     │     │     └─ "中" 字形正常 (glyph_id=xxx)
        │     │     │
        │     │     ├─ 添加正常字形到 ctx.glyphs
        │     │     │
        │     │     └─ ctx.used.pop()  ← pop!
        │     │              used = []
        │     │
        │     └─ 递归返回，"中" 已处理
        │
        ├─ i=1: glyph_id=120, "A"
        │     ├─ is_covered(1)? → "A" 匹配 latin-in-cjk → true
        │     └─ 添加正常字形到 ctx.glyphs
        │
        └─ i=2: glyph_id=0, "文"
              ├─ 类似 "中" 的处理流程
              └─ 递归到 Noto Serif CJK SC 处理
        │
        └─ ctx.used.pop()  ← 对应 get_font_and_covers 中为 Inria Serif 的 push？不！
                           ← 注意：Inria Serif 有 covers，所以没有 push，这里 pop 的是什么？
```

**等等，这里有一个关键点需要澄清**：`ctx.used.pop()` 总是在 `shape_segment` 末尾调用，但如果字体有 `covers` 限制，就没有对应的 `push`。这意味着什么？

让我重新审视代码：

[get_font_and_covers](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L947-L950):
```rust
// This font has been exhausted and will not be used again.
if covers.is_none() {
    ctx.used().push(font.clone());  // 只有无 covers 的字体才 push
}
```

[shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L1157):
```rust
ctx.used.pop();  // 总是 pop，不管有没有 push
```

### 6.3 栈不平衡的设计意图

这种"可能 push 也可能不 push，但总是 pop"的设计实际上是**栈帧平衡**的：

```
父调用 shape_segment:
  used = []
  get_font_and_covers(covers=None) → push(fontA), used = [fontA]
  发现 tofu，递归子调用
    子调用 shape_segment:
      get_font_and_covers(covers=None) → push(fontB), used = [fontA, fontB]
      处理完成
      used.pop() → used = [fontA]  ← 平衡：子调用的 push 被子调用的 pop 抵消
  继续处理
  used.pop() → used = []  ← 平衡：父调用的 push 被父调用的 pop 抵消
```

如果字体有 `covers` 限制（不 push），则 `pop` 实际上会弹出**更外层**的字体：

```
父调用 shape_segment:
  used = [fontA]  ← 之前已 push 的无 covers 字体
  get_font_and_covers(covers=Some(...)) → 不 push, used = [fontA]
  发现 tofu，递归子调用
    子调用 shape_segment:
      get_font_and_covers(covers=None) → push(fontB), used = [fontA, fontB]
      处理完成
      used.pop() → used = [fontA]  ← 子调用平衡
  继续处理
  used.pop() → used = []  ← 弹出的是 fontA！
```

**设计意图**：
- `used` 列表追踪的是"全局已用尽"的字体，而不是"当前调用帧使用"的字体
- 当一个无 `covers` 的字体被选择时，它被标记为"已用尽"（push），避免在递归中被重复选择
- 当 `shape_segment` 返回时，该字体在**这个分支**已处理完毕，可以"释放"（pop），允许在其他分支（如果有的话）重新使用
- 有 `covers` 的字体永远不会被标记为"已用尽"，因为它们只处理特定字符，可以被重复使用

### 6.4 实际场景中的栈行为

让我们用一个更完整的例子 `"A中B"`，字体列表 `["Arial", (name: "Inria Serif", covers: "latin-in-cjk"), "Noto Serif CJK SC"]`：

```
初始: used = []

shape_segment("A中B", [Arial, Inria(latin-in-cjk), NotoCJK])
  │
  ├─ get_font_and_covers()
  │   ├─ Arial: covers=None → push(Arial), used=[Arial]
  │   └─ 返回 (Arial, None)
  │
  ├─ rustybuzz.shape(Arial, "A中B")
  │   └─ A(good), 中(tofu), B(good)
  │
  ├─ 处理:
  │   ├─ A: good → 添加
  │   ├─ 中: tofu → 递归 shape_segment("中", [Arial, Inria, NotoCJK])
  │   │     │
  │   │     ├─ get_font_and_covers()
  │   │     │   ├─ Arial: !used.contains(Arial)? → false（已在 used 中！）
  │   │     │   │                         ← 跳过 Arial，避免循环
  │   │     │   │
  │   │     │   ├─ Inria: covers=latin-in-cjk → 不 push, used=[Arial]
  │   │     │   │   "中" 不在覆盖范围 → 继续
  │   │     │   │
  │   │     │   └─ NotoCJK: covers=None → push(NotoCJK), used=[Arial, NotoCJK]
  │   │     │
  │   │     ├─ rustybuzz.shape(NotoCJK, "中") → good
  │   │     ├─ 添加字形
  │   │     └─ used.pop() → used=[Arial]
  │   │
  │   └─ B: good → 添加
  │
  └─ used.pop() → used=[]
```

**关键点**：
1. `used` 列表在递归调用中被**共享**（`&mut` 引用）
2. 无 `covers` 的字体被 push 后，在递归子调用中会被 `!used.contains(font)` 过滤掉，避免重复尝试
3. 有 `covers` 的字体不会被 push，可以在递归中被重复尝试（但每次只处理其覆盖范围内的字符）
4. 每次 `shape_segment` 返回时的 `pop` 确保了栈的整体平衡

### 6.5 避免无限循环的机制

```rust
.filter(|font| !ctx.used().contains(font))
```

这个过滤条件确保了：
- 一旦某个无 `covers` 的字体被标记为"已用尽"（push 到 used），在它被 pop 之前，不会被再次选择
- 这防止了"字体A缺字 → 回退到字体A → 还是缺字 → 无限循环"的情况

### 6.6 与 covers 的交互总结

| 场景 | push? | pop? | 结果 |
|------|-------|------|------|
| `covers=None` | ✓ 是 | ✓ 是 | 该字体在递归中被跳过，不会重复尝试 |
| `covers=Some(...)` | ✗ 否 | ✓ 是 | 弹出的是外层的字体，该字体可在递归中重复使用 |

---

## 七、fi 连字和 ActualText/ToUnicode 在文本提取中的分工

### 7.1 连字（Ligature）在 HarfBuzz 中的处理

当文本包含 `"fi"` 时，HarfBuzz 可能会将其转换为一个连字 glyph。让我们看具体的数据流：

```
输入文本: "fi" (UTF-8: 0x66 0x69)
        │
        ▼
rustybuzz.shape_with_plan(font, plan, buffer)
        │
        ▼
输出 glyph_infos: [
  GlyphInfo { glyph_id: 42, cluster: 0 },  // "fi" 连字，对应原始文本 0..2
]
```

**Cluster 值的含义**：
- HarfBuzz 的 cluster 值表示字形对应的原始文本起始偏移
- 对于连字，两个字形被合并为一个，cluster 值为第一个字符的偏移（0）
- 没有第二个 glyph，因为 `f` 和 `i` 被合并了

### 7.2 Typst 中对连字 glyph 的文本范围计算

[shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L1050-L1086) 中，Typst 通过查找相邻 glyph 的 cluster 来计算文本范围：

```rust
// 对于连字 "fi" (glyph_id=42, cluster=0)：
let start = base + cluster;  // 0 + 0 = 0
let mut k = i;  // k = 0
let step: isize = if ltr { 1 } else { -1 };  // step = 1

let end = loop {
    let Some((next, next_info)) = k.checked_add_signed(step)
        .and_then(|n| infos.get(n).map(|info| (n, info)))
    else {
        break base + text.len();  // 没有下一个 glyph，end = 0 + 2 = 2
    };
    // ...
};

// 结果：range = 0..2
```

**代码中的示例**（注释非常有启发性）：
```rust
// Assume we have the following sequence of (glyph_id, cluster):
// [(120, 0), (80, 0), (3, 3), (755, 4), (69, 4), ...]
//
// We then want the sequence of (glyph_id, text_range) to look as follows:
// [(120, 0..3), (80, 0..3), (3, 3..4), (755, 4..13), (69, 4..13), ...]
//
// Each glyph in the same cluster should be assigned the full text range.
// This is necessary because only this way krilla can properly assign
// `ActualText` attributes in complex shaping scenarios.
```

### 7.3 ToUnicode CMap 与 ActualText 的分工

PDF 文本提取依赖两个互补的机制：

| 机制 | 用途 | 适用场景 | 示例 |
|------|------|----------|------|
| **ToUnicode CMap** | glyph_id → Unicode 字符的映射表 | 简单的 1:1 映射 | glyph 42 → U+0066 ('f') |
| **ActualText** | PDF 字符串属性，标注一段内容的原始文本 | 复杂的多对多映射 | 连字 "fi"、合字、阿拉伯语形变 |

#### ToUnicode CMap 的局限

ToUnicode CMap 本质上是一个简单的映射表：
```
42 beginbfchar
<002A> <0066>  // glyph 42 → 'f'
endbfchar
```

对于连字 "fi"，如果只有 ToUnicode 映射：
- glyph 42 → 只能映射到一个 Unicode 字符（如 `f`）
- 丢失了 `i` 的信息
- 文本提取结果会是 `"f"` 而不是 `"fi"`

#### ActualText 的作用

ActualText 是 PDF 内容流中的一个标记：
```
/Span <</ActualText (fi)>> BDC
... 绘制连字 glyph ...
EMC
```

当 PDF 阅读器遇到这个标记时，它会使用括号中的 `"fi"` 作为文本提取结果，而不是尝试从 glyph_id 映射。

### 7.4 Krilla 中的具体分工

当 Typst 调用 [surface.draw_glyphs()](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L50-L57) 时：

```rust
surface.draw_glyphs(
    krilla::geom::Point::from_xy(0.0, 0.0),
    glyphs,         // &[PdfGlyph]，每个实现了 krilla::text::Glyph trait
    font.clone(),
    text,           // &str = "fi"
    size.to_f32(),
    false,
);
```

Krilla 内部的处理：

1. **遍历 glyphs**，对每个 glyph：
   ```rust
   let range = glyph.text_range();  // 0..2 （对于 "fi" 连字）
   let unicode_text = &text[range]; // "fi"
   let gid = glyph.glyph_id();       // 42
   ```

2. **构建 ToUnicode CMap**（子集化字体的一部分）：
   - 如果 `unicode_text` 是单个字符 → 直接映射 `gid → unicode_text`
   - 如果 `unicode_text` 是多个字符 → 映射到第一个字符（不完整）

3. **决定是否需要 ActualText**：
   - 如果 `unicode_text.len() > 1` → 需要 ActualText（连字、合字等）
   - 如果同 cluster 有多个 glyph → 需要 ActualText（多字形表示一个字符）
   - 否则 → 只需要 ToUnicode

4. **生成 PDF 内容流**：
   ```
   /Span <</ActualText (fi)>> BDC
   [42] TJ  % 绘制 glyph 42
   EMC
   ```

### 7.5 连字 "fi" 的完整数据流

```
原始文本: "fi"
    │
    ▼
shape_segment:
  ├─ rustybuzz.shape() → glyph_id=42, cluster=0
  ├─ 计算 range: 0..2（下一个 cluster 是 2，或文本结尾）
  └─ ShapedGlyph { glyph_id: 42, range: 0..2, ... }
    │
    ▼
ShapedText.build():
  ├─ group_by_key(font): 同字体字形分组
  ├─ 计算 group 的总 range: 0..2
  ├─ 构建 Glyph { id: 42, range: 0..2 }
  └─ TextItem { font, glyphs: [Glyph { id: 42, range: 0..2 }], text: "fi" }
    │
    ▼
handle_text():
  ├─ convert_font(font) → krilla_font
  ├─ glyphs: &[PdfGlyph] = wrap_slice(&[Glyph { id: 42, range: 0..2 }])
  │    ├─ glyph_id() → GlyphId::new(42)
  │    └─ text_range() → 0..2
  └─ surface.draw_glyphs(glyphs, krilla_font, "fi", 12.0)
    │
    ▼
krilla 内部:
  ├─ 记录 glyph_id=42 到子集化集合
  ├─ text[0..2] = "fi" → 长度 > 1
  ├─ 需要 ActualText！
  ├─ 生成 ToUnicode: 42 → "f"（第一个字符）
  └─ 生成内容流:
     /Span <</ActualText (fi)>> BDC
     [42] TJ
     EMC
    │
    ▼
PDF 子集化字体嵌入:
  ├─ 字体子集只包含 glyph 42
  ├─ ToUnicode CMap: 42 → U+0066
  └─ ActualText 确保文本提取得到 "fi"
```

### 7.6 回退场景下的连字处理

如果字体回退发生在连字的中间，情况会更复杂。例如 `"fi"` 在字体 A 中不是连字，但在字体 B 中是：

```
文本: "fi"，字体列表: [字体A, 字体B]

shape_segment("fi", [A, B]):
  ├─ get_font_and_covers() → 字体A (covers=None), push(A), used=[A]
  ├─ rustybuzz.shape(A, "fi") → f(glyph_id=42, cluster=0), i(glyph_id=43, cluster=1)
  │     两个独立字形，没有连字
  ├─ 假设 f 是 tofu (glyph_id=0)
  │
  ├─ 找到 tofu 序列 [0..2]（两个都在连续 tofu？）
  ├─ 递归 shape_segment("fi", [A, B])
  │     ├─ get_font_and_covers():
  │     │   ├─ A: used.contains(A) → true → 跳过
  │     │   └─ B: covers=None → push(B), used=[A, B]
  │     │
  │     ├─ rustybuzz.shape(B, "fi") → fi(glyph_id=100, cluster=0)
  │     │     一个连字字形！
  │     │
  │     ├─ 计算 range: 0..2（因为是最后一个 glyph）
  │     ├─ 添加 ShapedGlyph { glyph_id=100, range: 0..2 }
  │     └─ used.pop() → used=[A]
  │
  └─ used.pop() → used=[]

结果：
  TextItem { font: B, glyphs: [Glyph { id: 100, range: 0..2 }], text: "fi" }
  → 连字正常，文本提取正确
```

### 7.7 分工总结

| 组件 | 职责 | 位置 |
|------|------|------|
| **HarfBuzz cluster** | 标记 glyph 对应的原始文本起始位置 | shaping 阶段 |
| **Typst range 计算** | 扩展 cluster 为完整文本范围（考虑同 cluster 的多 glyph） | [shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L1068-L1086) |
| **TextItem.text** | 保存完整原始文本 | [ShapedText.build](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L437-L446) |
| **PdfGlyph::text_range** | 告诉 krilla 每个 glyph 对应 text 的哪个子串 | [PdfGlyph::text_range](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L117-L119) |
| **ToUnicode CMap** | 基础的 glyph_id → Unicode 映射 | krilla 子集化阶段 |
| **ActualText** | 复杂映射（连字、合字）的补救标记 | krilla 内容流生成阶段 |

---

## 八、完整链路衔接图

```
┌─────────────────────────────────────────────────────────────────────┐
│                      阶段1: 字体发现                                │
├─────────────────────────────────────────────────────────────────────┤
│  [typst-cli/src/fonts.rs]                                           │
│  discover_fonts()                                                   │
│       │                                                             │
│       ▼                                                             │
│  fonts::system()  ──┐                                               │
│  fonts::embedded() ──┼──▶ FontStore {                               │
│  fonts::scan()     ──┘        book: FontBook {                      │
│                                 families: BTreeMap<家族名, [索引]> │
│                                 infos: Vec<FontInfo>                │
│                               }                                     │
│                               slots: Vec<FontSlot> {                │
│                                   source: FontPath/Font             │
│                                   font: OnceLock (惰性加载)         │
│                               }                                     │
│                             }                                       │
│       │                                                             │
│       ▼  通过 World trait 传递                                       │
│  World::book() -> &FontBook                                         │
│  World::font(index) -> Option<Font>                                 │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      阶段2: 字体回退                                │
├─────────────────────────────────────────────────────────────────────┤
│  [typst-layout/src/inline/shaping.rs]                               │
│  shape_range()                                                      │
│       │                                                             │
│       ▼                                                             │
│  shape()                                                            │
│    │ 构建 ShapingContext {                                          │
│    │     world: Tracked<dyn World>  ◀── 来自阶段1                   │
│    │     variant: FontVariant                                       │
│    │     features: Vec<Feature>                                     │
│    │     used: Vec<FontInstance>  (已尝试字体，避免循环)            │
│    │  }                                                             │
│    │                                                               │
│    ▼                                                               │
│  shape_segment() ◀───────────┐                                      │
│    │                         │                                      │
│    ├─▶ get_font_and_covers() │                                      │
│    │     │                   │                                      │
│    │     ├─ 阶段1: 遍历用户字体列表 families()                     │
│    │     │     for family in families {                            │
│    │     │       book.select(family, variant)                      │
│    │     │       └─▶ find_best_variant()                           │
│    │     │     }                                                   │
│    │     │                                                         │
│    │     ├─ 阶段2: 字体都不行？调用 select_fallback()             │
│    │     │     book.select_fallback(like, variant, text)           │
│    │     │       │                                                 │
│    │     │       ├─ 找到 text 中第一个非空白字符 c                  │
│    │     │       ├─ 过滤所有 info.coverage.contains(c) 的字体      │
│    │     │       └─▶ find_best_variant(like, variant, candidates) │
│    │     │                                                         │
│    │     └─ 阶段3: 还是不行？shape_tofus() 显示豆腐块              │
│    │                                                               │
│    ├─ rustybuzz.shape_with_plan(font, plan, buffer)                │
│    │     (HarfBuzz 字形处理)                                       │
│    │                                                               │
│    └─ 遍历处理后的 glyphs                                          │
│         │                                                          │
│         ├─ glyph_id != 0 && is_covered                            │
│         │    └─▶ 添加到 ctx.glyphs (ShapedGlyph)                  │
│         │                                                          │
│         └─ glyph_id == 0 (tofu) 或 不在覆盖范围                    │
│              ├─ 找到连续 tofu 序列 [start..end]                    │
│              ├─ 移除不完整的字形                                   │
│              └─ 递归调用 shape_segment(ctx, start, text[..], families.clone())
│                                                                   │
│    │ 最终输出：                                                    │
│    ▼                                                               │
│  ShapedText {                                                      │
│    glyphs: Glyphs<ShapedGlyph> {                                   │
│      font: FontInstance  ◀── 确定了具体使用的字体                  │
│      glyph_id: u16     ◀── 确定了字形索引                          │
│      x_advance, x_offset, ...                                     │
│    }                                                               │
│  }                                                                 │
│                                                                   │
│    │                                                               │
│    ▼                                                               │
│  ShapedText.build() → Frame {                                      │
│    items: [FrameItem::Text(TextItem)]                              │
│  }                                                                 │
└─────────────────────────────────────┬───────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      阶段3: 字体子集化                              │
├─────────────────────────────────────────────────────────────────────┤
│  [typst-pdf/src/text.rs]                                            │
│  convert() 入口                                                     │
│    │                                                                │
│    ├─ 创建 GlobalContext {                                          │
│    │     fonts_forward: FxHashMap<FontInstance, krilla::text::Font> │
│    │     fonts_backward: FxHashMap<krilla::text::Font, FontInstance>│
│    │   }                                                            │
│    │                                                                │
│    ▼                                                                │
│  convert_pages()                                                    │
│    │                                                                │
│    ▼                                                                │
│  handle_text()                                                      │
│    │                                                                │
│    ├─▶ convert_font(gc, t.font)                                    │
│    │     │                                                          │
│    │     ├─ 缓存命中？fonts_forward.get(&typst_font)               │
│    │     │                                                          │
│    │     └─ 未命中：build_font(typst_font)                         │
│    │          ├─ 共享字体数据 Arc<Bytes>                            │
│    │          ├─ 转换变体坐标 variations                            │
│    │          └─ krilla::text::Font::new_variable(data, index, vars)
│    │                                                               │
│    └─▶ surface.draw_glyphs(                                       │
│          point,                                                    │
│          glyphs: &[PdfGlyph],  ◀── 实现 krilla::text::Glyph trait │
│          font: krilla::text::Font,                                 │
│          text, size, ...                                           │
│        )                                                           │
│                                                                   │
│        ▲ 内部：krilla 记录每个 glyph_id 的使用                    │
│        │                                                          │
│  document.serialize()                                             │
│    │                                                              │
│    └─ Krilla 自动子集化：                                         │
│       ├─ 收集所有 draw_glyphs 中使用的 glyph_id                   │
│       ├─ 从原始字体提取这些字形及相关表                           │
│       └─ 构建子集化字体嵌入到 PDF                                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 九、关键数据结构流转

### 9.1 `FontInfo` - 字体元数据
来自字体发现阶段，贯穿整个链路：
```rust
pub struct FontInfo {
    family: String,           // 家族名（用于索引）
    variant: FontVariant,     // 变体（style/weight/stretch）
    flags: FontFlags,         // 标志（等宽、衬线、可变）
    axes: Vec<FontAxis>,      // 可变字体轴
    coverage: Coverage,       // Unicode 字符覆盖率（用于回退判断）
}
```

### 9.2 `Font` → `FontInstance` - 字体实例化
- `Font`：原始字体（来自字体发现阶段的惰性加载）
- `FontInstance`：应用了变体坐标的字体实例（来自回退阶段）

```rust
pub struct FontInstance {
    font: Font,                                    // 原始字体
    variations: FontVariations,                    // 变体坐标
    metrics: FontMetrics,                          // 度量信息
    rusty: rustybuzz::Face<'static>,              // 字形处理器
}
```

### 9.3 `Glyph` - 字形信息
从回退阶段流向子集化阶段：
```rust
pub struct Glyph {
    id: u16,           // 字形索引（子集化的关键输入）
    x_advance: Em,     // 水平步进
    x_offset: Em,      // 水平偏移
    range: Range<u16>, // 对应文本范围
    span: (Span, u16), // 源码位置
}
```

---

## 十、关键衔接点总结

### 10.1 字体发现 → 字体回退
- **接口**：`World` trait 的 `book()` 和 `font()` 方法
- **数据**：`FontBook`（元数据索引）和 `Font`（实际字体数据）
- **关键调用**：
  - `book.select(family, variant)` - 按家族选择
  - `book.select_fallback(like, variant, text)` - 全局回退
  - `world.font(id)` - 惰性加载实际字体

### 10.2 字体回退 → 字体子集化
- **接口**：`TextItem` 结构
- **数据**：
  - `FontInstance` - 确定了具体使用的字体和变体坐标
  - `Vec<Glyph>` - 每个字符的 `glyph_id`（子集化的依据）
- **关键调用**：
  - `convert_font(gc, font)` - 转换为 krilla 字体
  - `surface.draw_glyphs(glyphs, font, ...)` - 绘制时记录字形使用

### 10.3 子集化输出
- **结果**：嵌入到 PDF 的子集化字体，仅包含实际使用的字形
- **好处**：大幅减小 PDF 文件大小（通常从几 MB 降到几十 KB）

---

## 十一、设计亮点

1. **惰性加载**：字体数据只在实际使用时加载，减少内存占用
2. **递归回退**：字形处理时遇到缺失字符自动递归，对用户透明
3. **多维度评分**：回退时考虑字体特征相似性、变体距离、家族名相似度
4. **自动子集化**：由 krilla 库透明处理，无需手动管理字形集合
5. **双向缓存**：`fonts_forward` 和 `fonts_backward` 加速字体转换和错误报告

---

## 十二、边界路径：关闭回退 (fallback: false)

### 12.1 触发条件

用户通过 `#set text(fallback: false)` 关闭回退。此时 [TextElem::fallback](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs#L201-L203) 属性为 `false`。

### 12.2 关闭回退后的字体列表缩减

[families](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs#L1083-L1099) 函数中：

```rust
let tail = if styles.get(TextElem::fallback) {
    fallbacks.as_slice()   // 正常模式：追加内置回退字体
} else {
    &[]                     // 关闭回退：不追加任何回退字体
};
styles.get_ref(TextElem::font).into_iter().chain(tail.iter())
```

**效果**：字体列表仅包含用户显式指定的字体家族（如 `#set text(font: "Inria Serif")`），不再追加 `"libertinus serif"`、`"twitter color emoji"` 等内置回退。

### 12.3 关闭回退后的 `get_font_and_covers()` 行为

[get_font_and_covers](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L902-L953) 中：

```rust
// 阶段1: 遍历用户字体列表（不受 fallback 开关影响）
for family in families.by_ref() {
    selection = book.select(family.as_str(), ctx.variant())...;
    if selection.is_some() { break; }
}

// 阶段2: 全局回退——此处受 fallback 开关控制
if selection.is_none() && ctx.fallback() {   // ← fallback=false 时跳过
    selection = book.select_fallback(first, ctx.variant(), text)...;
}
```

**关键差异**：`fallback=false` 时：
- 阶段1 正常执行（仍按用户字体列表顺序查找）
- 阶段2 被完全跳过（`ctx.fallback()` 返回 `false`）
- 直接进入阶段3：`shape_tofus()`

### 12.4 关闭回退后的连字符处理

[ShapedText::hyphen](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L583-L638) 中，连字符的回退也受控制：

```rust
let fallback_func = if fallback {
    Some(|| book.select_fallback(None, base.variant, "-"))  // 正常模式
} else {
    None  // 关闭回退：不使用全局回退查找连字符
};
```

**效果**：如果用户字体列表中没有包含 `-` 字形的字体，断字时找不到连字符，该行将不会添加连字符。

### 12.5 完整路径图

```
fallback=false 时的字形处理路径：

shape_segment()
  │
  ├─▶ get_font_and_covers()
  │     ├─ 阶段1: 用户字体列表 → 找到？正常字形
  │     ├─ 阶段2: 跳过（fallback=false）
  │     └─ 阶段3: shape_tofus()  ← 用户字体中无匹配时直接进入
  │
  └─▶ rustybuzz.shape_with_plan()
        │
        ├─ glyph_id != 0 && is_covered → 正常字形
        │
        └─ glyph_id == 0 (tofu) → 递归 shape_segment()
              │
              ├─ get_font_and_covers()
              │     ├─ 阶段1: 继续遍历剩余的字体列表
              │     ├─ 阶段2: 跳过
              │     └─ 阶段3: shape_tofus()
              │
              └─ 所有字体耗尽 → shape_tofus() ← 最终显示豆腐块
```

---

## 十三、边界路径：字体覆盖限制 (covers)

### 13.1 设计意图

`covers` 机制允许用户精确控制某个字体家族只用于特定范围的 Unicode 字符，避免字体回退"过度使用"某个字体。

典型场景：
```typst
#set text(font: (
  (name: "Inria Serif", covers: "latin-in-cjk"),
  "Noto Serif CJK SC"
))
```
这里 `Inria Serif` 只负责 Latin 字符，CJK 字符直接交给 `Noto Serif CJK SC`，而不是等 Inria Serif 缺字后再回退。

### 13.2 `FontFamily` 和 `Covers` 数据结构

[FontFamily](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs#L942-L969)：

```rust
pub struct FontFamily {
    name: EcoString,              // 家族名（小写）
    covers: Option<Covers>,       // 覆盖限制（None 表示无限制）
}
```

[Covers](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs#L992-L1015) 枚举：

```rust
pub enum Covers {
    LatinInCjk,           // 预定义：排除 CJK/Latin 共有标点
    Regex(Regex),          // 自定义正则
}
```

#### `latin-in-cjk` 的具体排除列表

```rust
Self::LatinInCjk => singleton!(Regex,
    Regex::new(
        "[^\u{00B7}\u{2013}\u{2014}\u{2018}\u{2019}\
           \u{201C}\u{201D}\u{2025}-\u{2027}\u{2E3A}]"
    ).unwrap()
)
```

排除的字符：
| Unicode | 字符 | 说明 |
|---------|------|------|
| U+00B7  | · | 间隔号 |
| U+2013  | – | 短破折号 |
| U+2014  | — | 长破折号 |
| U+2018  | ' | 左单引号 |
| U+2019  | ' | 右单引号 |
| U+201C  | " | 左双引号 |
| U+201D  | " | 右双引号 |
| U+2025–U+2027 | ‥…‧ | 省略号相关 |
| U+2E3A  | ⸺ | 双长破折号 |

这些字符在 Latin 和 CJK 字体中都有，但 CJK 字体通常有更适合 CJK 排版的版本。

### 13.3 覆盖限制在字形处理中的执行

[shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L956-L1158) 中的 `is_covered` 闭包：

```rust
let is_covered = |offset| {
    let end = text[offset..]
        .char_indices()
        .nth(1)
        .map(|(i, _)| offset + i)
        .unwrap_or(text.len());
    covers.is_none_or(|cov| cov.is_match(&text[offset..end]))
};
```

**核心逻辑**：对每个字符，检查它是否匹配 `covers` 正则：
- `covers == None` → 无限制，所有字符都被"覆盖"
- `covers == Some(regex)` → 仅匹配正则的字符被视为"覆盖"

### 13.4 覆盖限制影响字形选择的两个层面

#### 层面1：字形输出阶段

```rust
if info.glyph_id != 0 && is_covered(cluster) {
    // 字形有效且在覆盖范围内 → 正常输出
    ctx.glyphs.push(ShapedGlyph { ... });
} else {
    // 两种情况进入此分支：
    // A) glyph_id == 0（字体中确实没有这个字符）
    // B) glyph_id != 0 但 !is_covered（字体有这个字符，但覆盖限制排除它）
    //    → 强制回退到下一个字体
    shape_segment(ctx, base + start, &text[start..end], families.clone());
}
```

**关键**：即使字体确实包含某个字符的 glyph，如果 `covers` 排除了该字符，它也会被视为"tofu"并触发递归回退。这让用户可以精确控制某个字体只负责特定字符。

#### 层面2：字体使用记录

```rust
if covers.is_none() {
    ctx.used().push(font.clone());  // 无覆盖限制的字体：标记为已用完
}
```

**关键**：有覆盖限制的字体（`covers.is_some()`）不会被加入 `used` 列表。这意味着同一个有覆盖限制的字体可以在不同字符段上被重复使用——只是每次只处理它覆盖范围内的字符。

### 13.5 覆盖限制与连字符

[ShapedText::hyphen](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L597-L601) 中：

```rust
let mut chain = families(base.styles)
    .filter(|family| family.covers().is_none_or(|c| c.is_match("-")))
    .map(|family| book.select(family.as_str(), base.variant))
    .chain(fallback_func.iter().map(|f| f()))
    .flatten();
```

连字符 `-` 在查找时也会检查覆盖限制。如果某个字体家族的 `covers` 不包含 `-`，该字体不会用于渲染连字符。

---

## 十四、边界路径：豆腐块 (tofu) 处理

### 14.1 什么是 tofu

"tofu"（豆腐块）是指字体中缺失的字符。在 OpenType 中，glyph ID 为 0 表示 `.notdef`（未定义字形），通常显示为一个小方框。

### 14.2 tofu 的产生路径

有三种情况会产生 tofu：

#### 路径A：所有字体列表和全局回退都找不到包含该字符的字体

在 [get_font_and_covers](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L940-L945) 中：

```rust
let Some(font) = selection else {
    if let Some(font) = ctx.used().first().cloned() {
        shape_tofus(ctx, text, font);   // 用第一个曾使用过的字体显示豆腐块
    }
    return None;  // 返回 None 表示字体选择失败
};
```

**注意**：这里使用 `ctx.used().first()` 作为豆腐块的字体，而不是当前正在尝试的字体。这保证了即使没有任何字体包含该字符，至少有一个可用的字体来渲染 `.notdef`。

**特殊情况**：如果 `ctx.used()` 为空（即此前没有任何字体被成功选择），则豆腐块也不会被渲染——该字符将完全消失在输出中。

#### 路径B：字体包含该字符但 glyph_id 为 0

在 rustybuzz 处理后，某些字符可能映射到 glyph ID 0，表示字体声称覆盖了该 Unicode 码位，但实际上没有对应的字形数据。

#### 路径C：字符不在 covers 覆盖范围内

即使字体包含该字符（glyph_id != 0），如果 `covers` 正则不匹配，该字符也被视为"未覆盖"并触发回退。

### 14.3 `shape_tofus` 实现

[shape_tofus](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L1237-L1268)：

```rust
fn shape_tofus(ctx: &mut ShapingContext, base: usize, text: &str, font: &FontInstance) {
    let x_advance = font.x_advance(0).unwrap_or_default();  // .notdef 字形的步进宽度
    let add_glyph = |(cluster, c): (usize, char)| {
        let start = base + cluster;
        let end = start + c.len_utf8();
        ctx.glyphs.push(ShapedGlyph {
            font: font.clone(),
            glyph_id: 0,              // ← 关键：始终为 0（.notdef）
            x_advance,                // 使用 .notdef 的步进宽度
            x_offset: Em::zero(),
            y_offset: Em::zero(),
            size: ctx.size,
            adjustability: Adjustability::default(),
            range: start..end,        // 每个字符一个独立范围
            safe_to_break: true,      // 豆腐块总是安全可断
            c,                        // 保留原始字符（用于后续处理）
            is_justifiable: ...,
            script: c.script(),
        });
    };
    if ctx.dir.is_positive() {
        text.char_indices().for_each(add_glyph);
    } else {
        text.char_indices().rev().for_each(add_glyph);  // RTL 反向遍历
    }
}
```

### 14.4 豆腐块在 PDF 子集化中的影响

当 `glyph_id == 0` 的 `ShapedGlyph` 被构建为 `TextItem` 时：
- `Glyph.id` 为 `0`
- 通过 `PdfGlyph::glyph_id()` 传递给 krilla：`GlyphId::new(0)`
- krilla 将 `.notdef` 字形记录到子集化集合中
- **PDF 验证问题**：如果启用了 PDF/A 或 PDF/UA 等标准验证，krilla 会报告 `ValidationError::ContainsNotDefGlyph`

[convert_error](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs#L600-L606) 中的错误处理：

```rust
ValidationError::ContainsNotDefGlyph(f, loc, text) => error!(
    to_span(*loc),
    "{prefix} the text `{}` could not be displayed with {}",
    text.repr(),
    display_font(gc.fonts_backward.get(f));
    hint: "try using a different font";
),
```

**影响**：对于需要 PDF/A 合规的文档，包含 `.notdef` 字形会导致验证失败，文档无法生成。

---

## 十五、边界路径：字体构建失败

### 15.1 `build_font` 失败

[build_font](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L79-L104) 将 Typst 的 `FontInstance` 转换为 krilla 的 `krilla::text::Font`：

```rust
#[comemo::memoize]
fn build_font(typst_font: FontInstance) -> SourceResult<krilla::text::Font> {
    let font_data: Arc<dyn AsRef<[u8]> + Send + Sync> =
        Arc::new(typst_font.data().clone());
    let variations = ...;
    match krilla::text::Font::new_variable(
        font_data.into(),
        typst_font.index(),
        &variations,
    ) {
        Some(f) => Ok(f),
        None => {
            bail!(
                Span::detached(),
                "failed to process {}",
                display_font(Some(&typst_font)),
            )
        }
    }
}
```

**失败原因**：
- 字体数据损坏或不完整
- 字体格式不被 krilla 支持（krilla 内部使用 `skrifa` 解析字体）
- 字体索引超出范围
- 可变字体实例化失败

**注意**：此函数被 `#[comemo::memoize]` 装饰，意味着同一个 `FontInstance` 只会构建一次。如果首次构建失败，后续遇到同一字体会直接返回缓存的错误。

### 15.2 krilla 序列化时字体错误

[finish](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs#L434-L533) 函数中，`document.finish()` 可能返回 `KrillaError::Font`：

```rust
KrillaError::Font(f, err) => {
    bail!(
        Span::detached(),
        "failed to process {} ({err})",
        display_font(gc.fonts_backward.get(&f));
        hint: "make sure the font is valid";
        hint: "the used font might be unsupported by Typst";
    );
}
```

**与 `build_font` 失败的区别**：
- `build_font` 失败发生在字体转换阶段（页面渲染时）
- `KrillaError::Font` 失败发生在最终 PDF 序列化阶段（子集化/嵌入时）

### 15.3 字体许可证限制

[ValidationError::RestrictedLicense](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs#L641-L649)：

```rust
ValidationError::RestrictedLicense(f) => error!(
    Span::detached(),
    "{prefix} license of {} is too restrictive",
    display_font(gc.fonts_backward.get(&f));
    hint: "the font has specified \"Restricted License embedding\" in its metadata";
    hint: "restrictive font licenses are prohibited by {} because they limit \
           the suitability for archival",
    failing_validators.to_and_list();
),
```

**触发条件**：字体的 OS/2 表中 `fsType` 字段标记为"Restricted License embedding"，且导出标准（如 PDF/A）禁止受限许可证字体。

**影响**：子集化虽然可以成功，但受限许可证会阻止 PDF/A 合规文档的生成。

---

## 十六、文本提取信息与 PDF 子集化的交接

### 16.1 文本提取的数据流

PDF 文本提取依赖两个关键信息：
1. **Unicode 映射**（ToUnicode CMap）：子集化字体中 glyph ID → Unicode 的映射
2. **ActualText 属性**（/ActualText）：复杂字形（如连字、合字）的原始文本

### 16.2 `TextItem.text` 的来源

[ShapedText.build](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L437-L446) 中：

```rust
let item = TextItem {
    font,
    size: glyph_size,
    lang: self.lang,
    region: self.region,
    fill: fill.clone(),
    stroke: stroke.clone().map(|s| s.unwrap_or_default()),
    text: self.text[range.start - self.base..range.end - self.base].into(),  // ← 关键
    glyphs,
};
```

`text` 字段是通过字形的 `range` 从原始文本中切片得到的。这意味着：
- 即使发生字体回退（同一段文本被不同字体渲染），每个 `TextItem` 的 `text` 仍然是原始文本的子串
- `text` 保留了正确的 Unicode 内容，包括连字中的多字符映射

### 16.3 `PdfGlyph::text_range()` 的作用

[PdfGlyph::text_range](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L117-L119)：

```rust
fn text_range(&self) -> Range<usize> {
    self.0.range.start as usize..self.0.range.end as usize
}
```

`text_range` 告诉 krilla 这个字形对应 `text` 参数中的哪个位置。krilla 利用此信息：
1. **构建 `/ActualText`**：对于复杂字形（多个 glyph 对应同一段文本，如连字 "fi"），krilla 使用 `/ActualText` 属性标注原始文本
2. **构建 ToUnicode CMap**：krilla 在子集化字体时构建 glyph ID → Unicode 的反向映射

### 16.4 cluster 范围与文本提取的关系

[shape_segment](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs#L1050-L1086) 中，每个字形的文本范围基于 HarfBuzz 的 cluster 值计算：

```rust
// 同一 cluster 的所有 glyph 共享完整文本范围
let start = base + cluster;
let mut k = i;
let step: isize = if ltr { 1 } else { -1 };
let end = loop {
    let Some((next, next_info)) = k.checked_add_signed(step)
        .and_then(|n| infos.get(n).map(|info| (n, info)))
    else {
        break base + text.len();
    };
    if next_info.cluster != info.cluster {
        break base + next_info.cluster as usize;
    }
    k = next;
};
```

**设计意图**：注释中明确说明——

> "Each glyph in the same cluster should be assigned the full text range. This is necessary because only this way krilla can properly assign `ActualText` attributes in complex shaping scenarios."

**对 PDF 的影响**：
- 连字（如 "fi"）：两个 glyph 共享 `0..2` 的范围，krilla 知道它们对应 "fi" 而非单独的 "f" 和 "i"
- 合字（如阿拉伯语形变）：多个 glyph 对应同一段文本，`/ActualText` 确保文本提取正确
- 回退分段：不同 `TextItem` 的 `text` 不重叠，每个字符恰好属于一个 `TextItem`

### 16.5 `draw_glyphs` 的 `text` 参数

[handle_text](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs#L42-L57) 中：

```rust
let text = t.text.as_str();        // TextItem 的文本子串
let glyphs: &[PdfGlyph] = TransparentWrapper::wrap_slice(t.glyphs.as_slice());
surface.draw_glyphs(
    krilla::geom::Point::from_xy(0.0, 0.0),
    glyphs,         // 每个 glyph 的 text_range 指向 text 中的位置
    font.clone(),
    text,           // ← 原始文本，krilla 用它构建文本提取信息
    size.to_f32(),
    false,
);
```

**`text` 的作用**：
- krilla 将 `text[text_range]` 与 glyph ID 关联
- 子集化时，krilla 为每个使用的 glyph 生成 ToUnicode 映射
- 对于无法直接映射的 glyph（如连字），krilla 生成 `/ActualText` 标记

### 16.6 豆腐块对文本提取的影响

豆腐块（glyph_id == 0）对 PDF 文本提取有特殊影响：

1. **ToUnicode 映射缺失**：`.notdef` glyph 通常不在 ToUnicode CMap 中，文本提取时该字符可能丢失
2. **`/ActualText` 补救**：如果 krilla 为 `.notdef` 字形生成了 `/ActualText` 属性，文本提取仍可获取原始字符
3. **PDF/A 合规**：`ContainsNotDefGlyph` 验证错误直接阻止合规文档生成

### 16.7 回退分段对文本提取的影响

字体回退会将同一段文本拆分为多个 `TextItem`，每个使用不同字体。在 PDF 中，这会产生多个独立的文本对象（`Tj` 操作符），每个对象使用不同的子集化字体。

**对文本提取的影响**：
- 正面：每个 `TextItem` 的 `text` 是原始文本的精确子串，文本提取时拼接正确
- 潜在问题：如果 PDF 阅读器按文本对象分段提取，中间可能插入空格（取决于阅读器实现）
- 解决方案：krilla 使用 `/ActualText` 属性确保连续文本的正确提取

---

## 十七、完整边界路径决策树

```
shape_segment(ctx, base, text, families)
  │
  ├─ text 全是换行/Tab/默认忽略字符？→ 直接返回（不处理）
  │
  ├─▶ get_font_and_covers()
  │     │
  │     ├─ [阶段1] 遍历用户字体列表
  │     │     ├─ book.select(family, variant) → 找到？
  │     │     │     ├─ 字体不在 used 中？→ 选中
  │     │     │     └─ 字体已在 used 中？→ 继续下一个 family
  │     │     └─ 所有 family 都尝试过？→ selection 仍为 None
  │     │
  │     ├─ [阶段2] fallback=true？
  │     │     ├─ 是 → book.select_fallback(like, variant, text)
  │     │     │        ├─ 找到包含该字符的字体？→ 选中
  │     │     │        └─ 所有字体都不包含？→ selection 仍为 None
  │     │     └─ 否 → 跳过
  │     │
  │     └─ [阶段3] selection 仍为 None？
  │           ├─ used 列表非空 → shape_tofus(用 used[0])
  │           └─ used 列表为空 → 返回 None（字符消失）
  │
  ├─▶ rustybuzz.shape_with_plan(font, plan, buffer)
  │
  └─ 遍历 glyphs
        │
        ├─ glyph_id != 0 && is_covered(cluster)?
        │     │
        │     │   is_covered 判定：
        │     │   ├─ covers == None → 始终 true
        │     │   └─ covers == Some(regex) → regex.is_match(字符)?
        │     │
        │     └─ 是 → 输出正常字形 ShapedGlyph
        │
        └─ glyph_id == 0 || !is_covered?
              │
              ├─ 找到连续 tofu/未覆盖序列 [start..end]
              ├─ 移除不完整 cluster 的已有字形
              └─ 递归 shape_segment(text[start..end], families.clone())
                    │
                    ├─ families 已耗尽 + fallback=false
                    │     └─ shape_tofus() → glyph_id=0 的豆腐块
                    │
                    ├─ families 已耗尽 + fallback=true
                    │     └─ select_fallback → 找到/找不到
                    │           ├─ 找到 → 用新字体重新处理
                    │           └─ 找不到 → shape_tofus()
                    │
                    └─ families 还有剩余
                          └─ 用下一个字体重新处理
```

```
PDF 子集化阶段的边界处理：

TextItem (font, glyphs, text)
  │
  ├─▶ convert_font(gc, font)
  │     ├─ 缓存命中 → 返回 krilla::text::Font
  │     └─ 缓存未命中 → build_font()
  │           ├─ krilla::text::Font::new_variable() 成功 → 缓存 + 返回
  │           └─ 失败 → bail!() ← 编译中断，不生成 PDF
  │
  └─▶ surface.draw_glyphs(glyphs, font, text, size)
        │
        ├─ 每个 glyph 的 glyph_id → 记录到子集化集合
        │     ├─ glyph_id != 0 → 正常字形，子集化时包含
        │     └─ glyph_id == 0 → .notdef，可能触发验证错误
        │
        ├─ 每个 glyph 的 text_range → 构建 ActualText
        │     └─ 同 cluster 多 glyph → 合并为一个 ActualText
        │
        └─ text 参数 → 构建 ToUnicode CMap
              └─ text[range] → glyph_id 的 Unicode 映射

document.finish() → PDF 序列化
  │
  ├─ 子集化字体嵌入
  │     ├─ 收集所有 glyph_id → 生成子集
  │     ├─ 生成 ToUnicode CMap
  │     └─ 生成 ActualText 标记
  │
  ├─ 验证错误 (PDF/A, PDF/UA)
  │     ├─ ContainsNotDefGlyph → 豆腐块字体 + 文本内容
  │     ├─ RestrictedLicense → 许可证受限字体
  │     ├─ NoCodepointMapping → 无 Unicode 映射
  │     └─ InvalidCodepointMapping → 非法码点映射
  │
  └─ KrillaError::Font → 字体处理失败
        └─ 编译中断，返回错误信息
```

---

## 十八、相关文件索引

| 模块 | 文件 | 核心功能 |
|------|------|----------|
| 字体发现 | [typst-kit/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs) | `FontStore`, `FontSlot`, 系统/目录扫描 |
| 字体发现 | [typst-cli/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-cli/src/fonts.rs) | `discover_fonts()` 入口 |
| 字体索引 | [typst-library/src/text/font/book.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs) | `FontBook`, `select()`, `select_fallback()` |
| 字体结构 | [typst-library/src/text/font/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/mod.rs) | `Font`, `FontInstance` |
| 文本配置 | [typst-library/src/text/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs) | `families()`, `FontFamily`, `Covers`, `TextElem` 属性 |
| 字形处理 | [typst-layout/src/inline/shaping.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs) | `shape_segment()`, `get_font_and_covers()`, `shape_tofus()`, `used` 栈管理 |
| 字形项 | [typst-library/src/text/item.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/item.rs) | `TextItem`, `Glyph` 定义 |
| PDF 文本 | [typst-pdf/src/text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs) | `handle_text()`, `convert_font()`, `PdfGlyph` |
| PDF 转换 | [typst-pdf/src/convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs) | `GlobalContext`, 字体双向缓存, `finish()` 错误处理 |
| 数学公式 | [typst-library/src/math/equation.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/math/equation.rs) | `EquationElem::show_set()` 设置数学字体 "New Computer Modern Math" |
| 数学公式 | [typst-library/src/math/ir/resolve.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/math/ir/resolve.rs) | `resolve_text()`, `resolve_symbol()` 中 `to_style()` 数学变体转换 |
| 数学公式 | [typst-library/src/math/style.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/math/style.rs) | `bold()`, `bb()`, `sans()`, `frak()` 等数学样式函数 |
| 数学布局 | [typst-layout/src/math/text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/math/text.rs) | `layout_text()`, `layout_glyph()`, `dtls`/`flac` OpenType 特性 |
