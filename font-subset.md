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

## 五、完整链路衔接图

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

## 六、关键数据结构流转

### 6.1 `FontInfo` - 字体元数据
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

### 6.2 `Font` → `FontInstance` - 字体实例化
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

### 6.3 `Glyph` - 字形信息
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

## 七、关键衔接点总结

### 7.1 字体发现 → 字体回退
- **接口**：`World` trait 的 `book()` 和 `font()` 方法
- **数据**：`FontBook`（元数据索引）和 `Font`（实际字体数据）
- **关键调用**：
  - `book.select(family, variant)` - 按家族选择
  - `book.select_fallback(like, variant, text)` - 全局回退
  - `world.font(id)` - 惰性加载实际字体

### 7.2 字体回退 → 字体子集化
- **接口**：`TextItem` 结构
- **数据**：
  - `FontInstance` - 确定了具体使用的字体和变体坐标
  - `Vec<Glyph>` - 每个字符的 `glyph_id`（子集化的依据）
- **关键调用**：
  - `convert_font(gc, font)` - 转换为 krilla 字体
  - `surface.draw_glyphs(glyphs, font, ...)` - 绘制时记录字形使用

### 7.3 子集化输出
- **结果**：嵌入到 PDF 的子集化字体，仅包含实际使用的字形
- **好处**：大幅减小 PDF 文件大小（通常从几 MB 降到几十 KB）

---

## 八、设计亮点

1. **惰性加载**：字体数据只在实际使用时加载，减少内存占用
2. **递归回退**：字形处理时遇到缺失字符自动递归，对用户透明
3. **多维度评分**：回退时考虑字体特征相似性、变体距离、家族名相似度
4. **自动子集化**：由 krilla 库透明处理，无需手动管理字形集合
5. **双向缓存**：`fonts_forward` 和 `fonts_backward` 加速字体转换和错误报告

---

## 九、相关文件索引

| 模块 | 文件 | 核心功能 |
|------|------|----------|
| 字体发现 | [typst-kit/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-kit/src/fonts.rs) | `FontStore`, `FontSlot`, 系统/目录扫描 |
| 字体发现 | [typst-cli/src/fonts.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-cli/src/fonts.rs) | `discover_fonts()` 入口 |
| 字体索引 | [typst-library/src/text/font/book.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/book.rs) | `FontBook`, `select()`, `select_fallback()` |
| 字体结构 | [typst-library/src/text/font/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/font/mod.rs) | `Font`, `FontInstance` |
| 文本配置 | [typst-library/src/text/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/mod.rs) | `families()`, `TextElem` 字体属性 |
| 字形处理 | [typst-layout/src/inline/shaping.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-layout/src/inline/shaping.rs) | `shape_segment()`, `get_font_and_covers()` |
| 字形项 | [typst-library/src/text/item.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-library/src/text/item.rs) | `TextItem`, `Glyph` 定义 |
| PDF 文本 | [typst-pdf/src/text.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/text.rs) | `handle_text()`, `convert_font()`, `PdfGlyph` |
| PDF 转换 | [typst-pdf/src/convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/129-typst/crates/typst-pdf/src/convert.rs) | `GlobalContext`, 字体双向缓存 |
