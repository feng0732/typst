# Typst PDF 输出后端分析

## 概述

Typst 的 PDF 输出功能由 `typst-pdf` crate 实现，位于 `crates/typst-pdf/` 目录。该模块负责将排版后的 `PagedDocument` 转换为符合 PDF 规范的字节流。核心依赖是 **krilla 0.8.2** 库，一个专门用于 PDF 生成的高级 Rust 库，构建在 **pdf-writer 0.15.0** 底层库之上。

krilla 源码已下载到 `krilla-src/krilla-0.8.2/` 目录，pdf-writer 源码已下载到 `pdf-writer-0.15.0/` 目录供参考。

## 核心架构

### 主要依赖

```
typst-pdf
├── krilla 0.8.2      # PDF 生成核心库（高级抽象）
│   └── pdf-writer 0.15.0  # 底层 PDF 语法生成（字节级写入）
├── krilla-svg        # SVG 渲染支持
├── subsetter         # 字体子集化（krilla 的依赖）
├── typst-layout      # 排版后文档结构
├── typst-library     # 类型定义和工具函数
└── ...
```

### 入口点

导出入口在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/lib.rs#L36-L38)：

```rust
pub fn pdf(document: &PagedDocument, options: &PdfOptions) -> SourceResult<Vec<u8>> {
    convert::convert(document, options, &[], None)
}
```

---

## 一、PDF 对象组织机制

### 1.1 全局上下文 (GlobalContext)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L278-L301) 中定义，贯穿整个转换过程：

```rust
pub(crate) struct GlobalContext<'a> {
    // 字体缓存（双向映射）
    pub(crate) fonts_forward: FxHashMap<FontInstance, krilla::text::Font>,
    pub(crate) fonts_backward: FxHashMap<krilla::text::Font, FontInstance>,
    // 图像跨度映射
    pub(crate) image_to_spans: FxHashMap<krilla::image::Image, Span>,
    pub(crate) image_spans: FxHashSet<Span>,
    // 文档引用
    pub(crate) document: &'a PagedDocument,
    pub(crate) options: &'a PdfOptions,
    // 链接解析器（用于 bundle 导出）
    pub(crate) link_resolver: Option<Tracked<'a, LateLinkResolver<'a>>>,
    // 命名目标映射
    pub(crate) loc_to_names: FxHashMap<Location, NamedDestination>,
    // 页面索引转换器
    pub(crate) page_index_converter: PageIndexConverter,
    // 标签上下文（无障碍支持）
    pub(crate) tags: Tags,
}
```

### 1.2 帧上下文 (FrameContext)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L220-L275) 中定义，用于单个 Frame 的转换：

```rust
pub(crate) struct FrameContext {
    pub(crate) page_idx: Option<usize>,
    states: Vec<State>,  // 变换状态栈
    link_annotations: IndexMap<GroupId, SmallVec<[LinkAnnotation; 1]>, FxBuildHasher>,
}
```

### 1.3 变换状态 (State)

在 [convert.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/crates/typst-pdf/src/convert.rs#L178-L217) 中维护坐标变换栈：

```rust
pub(crate) struct State {
    transform: Transform,              // 当前变换
    container_transform: Transform,    // 第一个硬帧的变换
    container_size: Size,              // 第一个硬帧的尺寸
}
```

### 1.4 Krilla 对象模型

Typst 不直接操作 PDF 语法，而是通过 krilla 提供的高级抽象。

---

## 二、页面资源 (Page Resources) 字典 —— 基于源码的分析

### 2.1 资源字典结构定义

在 krilla [resource.rs#L13-L19](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L13-L19) 中定义了 `Resource` trait，所有资源类型必须实现：

```rust
pub(crate) trait Resource {
    fn new(ref_: Ref) -> Self;
    fn get_ref(&self) -> Ref;
    fn get_dict<'a>(resources: &'a mut writers::Resources) -> Dict<'a>;
    fn get_prefix() -> &'static str;
    fn get_mapper(b: &mut ResourceDictionaryBuilder) -> &mut ResourceMapper<Self>;
}
```

### 2.2 资源类型与命名规则

在 krilla [resource.rs#L25-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L25-L173) 中定义了六种资源类型及其命名前缀：

| 资源类型 | 前缀 | PDF 字典键 | 实现位置 |
|---------|------|-----------|---------|
| `Font` | `"f"` | `/Font` | [resource.rs#L150-L173](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L150-L173) |
| `XObject` | `"x"` | `/XObject` | [resource.rs#L100-L123](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L100-L123) |
| `Pattern` | `"p"` | `/Pattern` | [resource.rs#L125-L148](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L125-L148) |
| `Shading` | `"s"` | `/Shading` | [resource.rs#L75-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L75-L98) |
| `ColorSpace` | `"c"` | `/ColorSpace` | [resource.rs#L50-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L50-L73) |
| `ExtGState` | `"g"` | `/ExtGState` | [resource.rs#L25-L48](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L25-L48) |

资源名称格式：`{prefix}{number}`，例如 `/F0`, `/F1`, `/Im0`, `/Im1`。

### 2.3 资源映射器 (ResourceMapper)

在 krilla [resource.rs#L337-L381](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L337-L381) 中实现了双向映射机制：

```rust
pub(crate) struct ResourceMapper<T: ?Sized> {
    forward: Vec<Ref>,           // 资源编号 → 对象引用
    backward: HashMap<Ref, ResourceNumber>,  // 对象引用 → 资源编号
    phantom: PhantomData<T>,
}

impl<T> ResourceMapper<T> where T: Resource {
    pub(crate) fn remap(&mut self, ref_: Ref) -> ResourceNumber {
        let forward = &mut self.forward;
        let backward = &mut self.backward;

        *backward.entry(ref_).or_insert_with(|| {
            let old = forward.len();
            forward.push(ref_);
            old as ResourceNumber
        })
    }

    pub(crate) fn remap_with_name(&mut self, ref_: Ref) -> String {
        Self::name_from_number(self.remap(ref_))
    }
}
```

**关键机制：**
- 使用 `HashMap` 实现引用去重，同一对象不会被注册两次
- 资源编号按注册顺序分配，从 0 开始递增
- 资源名称由前缀 + 编号组成

### 2.4 资源字典构建器 (ResourceDictionaryBuilder)

在 krilla [resource.rs#L175-L214](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L175-L214) 中：

```rust
pub(crate) struct ResourceDictionaryBuilder {
    pub(crate) color_spaces: ResourceMapper<ColorSpace>,
    pub(crate) ext_g_states: ResourceMapper<ExtGState>,
    pub(crate) patterns: ResourceMapper<Pattern>,
    pub(crate) x_objects: ResourceMapper<XObject>,
    pub(crate) shadings: ResourceMapper<Shading>,
    pub(crate) fonts: ResourceMapper<Font>,
}

impl ResourceDictionaryBuilder {
    pub(crate) fn register_resource<T>(&mut self, obj: T) -> String
    where
        T: Resource,
    {
        T::get_mapper(self).remap_with_name(obj.get_ref())
    }

    pub(crate) fn finish(self) -> ResourceDictionary {
        ResourceDictionary {
            color_spaces: self.color_spaces.into_resource_list(),
            ext_g_states: self.ext_g_states.into_resource_list(),
            patterns: self.patterns.into_resource_list(),
            x_objects: self.x_objects.into_resource_list(),
            shadings: self.shadings.into_resource_list(),
            fonts: self.fonts.into_resource_list(),
        }
    }
}
```

### 2.5 资源字典序列化

在 krilla [resource.rs#L241-L287](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L241-L287) 中，`to_pdf_resources()` 方法将资源字典写入 PDF：

```rust
pub fn to_pdf_resources<T>(
    &self,
    parent: &mut T,
    sc: &mut SerializeContext,
    resources_chunk: &mut Chunk,
) where
    T: ResourcesExt,
{
    // 检查是否需要写入 ProcSet（PDF 1.4 需要，高版本已弃用）
    let write_proc_sets = !sc.serialize_settings().pdf_version().deprecates_proc_sets();
    
    if !write_proc_sets && !has_resource_entries {
        parent.resources().finish();  // 空资源字典作为直接对象
        return;
    }

    // 创建间接对象存储资源字典
    let resources_ref = sc.new_ref();
    let mut resources = resources_chunk
        .indirect(resources_ref)
        .start::<writers::Resources>();
    
    if write_proc_sets {
        resources.proc_sets([
            ProcSet::Pdf, ProcSet::Text,
            ProcSet::ImageColor, ProcSet::ImageGrayscale,
        ]);
    }
    
    // 写入各类资源
    write_resource_type::<ColorSpace>(&mut resources, &self.color_spaces);
    write_resource_type::<ExtGState>(&mut resources, &self.ext_g_states);
    write_resource_type::<Pattern>(&mut resources, &self.patterns);
    write_resource_type::<XObject>(&mut resources, &self.x_objects);
    write_resource_type::<Shading>(&mut resources, &self.shadings);
    write_resource_type::<Font>(&mut resources, &self.fonts);
    
    parent.set_resources(resources_ref);
}
```

**关键点：**
- 资源字典作为**间接对象**存储，除非为空
- `/ProcSet` 数组在 PDF 1.5+ 已弃用，但 PDF/A-1 等标准仍要求
- 资源名称按注册顺序编号，如 `/F0`, `/F1`, `/X0`, `/X1`

### 2.6 资源注册时机

资源注册发生在内容流构建过程中，例如在 krilla [content.rs#L542-L544](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/content.rs#L542-L544) 中：

```rust
let font_name = self
    .rd_builder
    .register_resource(sc.register_font_identifier(font_identifier));
self.content.set_font(font_name.to_pdf_name(), size);
```

**注册流程：**
1. `sc.register_font_identifier()`：分配或获取字体的间接对象引用 (Ref)
2. `rd_builder.register_resource()`：将 Ref 映射到资源名称（如 `/F0`）
3. `content.set_font()`：在内容流中输出 `BT /F0 12 Tf ET` 操作符

---

## 三、间接对象编号分配 —— 基于源码的分析

### 3.1 引用计数器初始化

在 krilla [serialize.rs#L279-L310](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L279-L310) 中，`SerializeContext::new()` 初始化引用计数器：

```rust
impl SerializeContext {
    pub(crate) fn new(mut serialize_settings: SerializeSettings) -> Self {
        // ... 覆盖配置 ...
        
        let mut cur_ref = Ref::new(1);           // 初始编号 1
        let page_tree_ref = cur_ref.bump();     // 2: Pages 字典
        let pdf2_ns = Pdf2Namespaces {
            ssn_ref: cur_ref.bump(),            // 3: PDF 2.0 命名空间
            krilla_ref: cur_ref.bump(),         // 4: Krilla 命名空间
        };
        
        Self {
            cur_ref,                            // 当前编号 5
            // ...
        }
    }
}
```

**预分配的对象编号：**
- `1`: 预留（未使用，实际从 2 开始）
- `2`: `/Pages` 页面对象树的根
- `3`: PDF 2.0 结构树命名空间
- `4`: Krilla 扩展命名空间
- `5+`: 动态分配

### 3.2 动态编号分配

在 krilla [serialize.rs#L368-L370](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L368-L370) 中定义了核心分配方法：

```rust
pub(crate) fn new_ref(&mut self) -> Ref {
    self.cur_ref.bump()
}
```

`Ref::bump()` 简单递增内部计数器：`Ref { id: self.id + 1, gen: 0 }`。

### 3.3 编号分配时机

**对象首次创建时分配编号：**

| 操作 | 代码位置 | 说明 |
|-----|---------|------|
| 注册字体 | [serialize.rs#L640-L649](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L640-L649) | `register_font_identifier()` |
| 注册颜色空间 | [serialize.rs#L651-L681](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L651-L681) | `register_colorspace()` |
| 注册图像 | [serialize.rs#L614-L622](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L614-L622) | `register_image()` |
| 注册页面 | [serialize.rs#L561-L571](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L561-L571) | `register_page()` |
| 序列化字体 | [text/cid.rs#L196-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L196-L199) | 字体对象、描述符、CMap、CIDSet、字体数据 |
| 资源字典 | [resource.rs#L267](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs#L267) | `to_pdf_resources()` 中 |

**字体序列化时的编号分配示例** [text/cid.rs#L196-L199](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L196-L199)：
```rust
let cid_ref = sc.new_ref();        // CIDFont 对象
let descriptor_ref = sc.new_ref(); // FontDescriptor 对象
let cmap_ref = sc.new_ref();       // ToUnicode CMap 对象
let cid_set_ref = sc.new_ref();    // CIDSet 对象
let data_ref = sc.new_ref();       // FontFile 流对象
```

### 3.4 二次编号重映射

在最终序列化阶段，krilla 会进行二次编号重排。在 krilla [chunk_container.rs#L100-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/chunk_container.rs#L100-L120) 中：

```rust
pub(crate) fn finish(self, sc: &mut SerializeContext) -> KrillaResult<Pdf> {
    let mut remapped_ref = Ref::new(1);
    let mut remapper = HashMap::new();

    // 第一遍：收集所有对象，重新分配连续编号
    self.visit(sc, &mut |chunk| {
        for object_ref in chunk.refs() {
            let existing = remapper.insert(object_ref, remapped_ref.bump());
            debug_assert!(existing.is_none());
        }
        chunks_byte_len += chunk.len();
    })?;

    // 第二遍：使用新编号重写所有引用
    self.visit(sc, &mut |chunk| {
        chunk.renumber_into(&mut pdf, |old| remapper[&old]);
    })?;
    
    // ...
}
```

**`Chunk::renumber_into()` 实际调用流程** [pdf-writer chunk.rs#L181-L187](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L181-L187)：

```rust
pub fn renumber_into<F>(&self, target: &mut Chunk, mut mapping: F)
where
    F: FnMut(Ref) -> Ref,
{
    target.buf.reserve(self.len());
    crate::renumber::renumber(self, target, &mut mapping);
}
```

**重映射目的：**
- 确保 PDF 中的对象编号是**单调递增**的
- 提高文件结构的清晰度和可读性
- 为后续实现对象流（Object Streams）等优化做准备
- 避免跨 chunk 的引用混乱

### 3.5 对象数量限制

在 krilla [serialize.rs#L944-L947](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L944-L947) 中定义了对象数量限制：

```rust
fn check_validator_limits(&mut self) {
    if self.cur_ref > Ref::new(8388607) {
        self.register_validation_error(ValidationError::TooManyIndirectObjects)
    }
}
```

最大间接对象数：8,388,607（约 830 万），远超过 PDF 1.4 的 8,191 限制。

---

## 四、XRef 表生成与对象偏移 —— 基于 pdf-writer 源码的分析

### 4.1 对象偏移记录机制

**Chunk 内部结构** [pdf-writer chunk.rs#L34-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L34-L38)：

```rust
pub struct Chunk {
    pub(crate) buf: Buf,                    // 字节缓冲区
    pub(crate) offsets: Vec<(Ref, usize)>,  // 对象引用 → 字节偏移
    pub(crate) settings: Settings,          // 写入配置
}
```

**对象偏移记录发生在 `indirect()` 调用时** [pdf-writer chunk.rs#L193-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L193-L196)：

```rust
pub fn indirect(&mut self, id: Ref) -> Obj<'_> {
    self.offsets.push((id, self.buf.len()));  // ← 记录当前字节偏移
    Obj::indirect(&mut self.buf, id, self.settings)
}
```

**关键点：**
- `offsets` 是 `Vec<(Ref, usize)>`，按写入顺序存储
- 偏移量是对象在 `buf` 中的起始字节位置
- 每次调用 `indirect()` 或 `stream()` 都会触发偏移记录

**间接对象写入格式** [pdf-writer object.rs#L666-L676](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/object.rs#L666-L676)：

```rust
pub(crate) fn indirect(buf: &'a mut Buf, id: Ref, settings: Settings) -> Self {
    buf.push_int(id.get());
    buf.extend(b" 0 obj\n");  // 例如: "1 0 obj\n"
    Self {
        buf,
        indirect: true,
        indent: 0,
        settings,
        needs_padding: false,
    }
}
```

### 4.2 Pdf::finish() —— XRef 表生成入口

在 [pdf-writer lib.rs#L307-L322](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L307-L322) 中：

```rust
pub fn finish(self) -> Vec<u8> {
    let Chunk { mut buf, offsets, settings } = self.chunk;
    let trailer_data = self.trailer_data;
    let xref_offset = buf.len();  // 记录 xref 表的起始偏移

    // 写入 xref 表（传统文本格式）
    let mut writer = PlainXRefWriter::new(&mut buf);
    let xref_len = write_offsets(offsets, &mut writer);

    // 写入 trailer 字典
    buf.extend(b"trailer\n");
    let mut trailer = Obj::direct(&mut buf, 0, settings, false).dict();
    trailer_data.write_into_dict(&mut trailer, xref_len);
    trailer.finish();

    // 写入 startxref 和 EOF
    finish_trailer(buf, xref_offset, b"\n")
}
```

**执行流程：**
1. 从 `Chunk` 中提取 `buf`、`offsets` 和 `settings`
2. 记录当前 `buf.len()` 作为 `xref_offset`（xref 表的起始位置）
3. 创建 `PlainXRefWriter` 写入传统 xref 表
4. 调用 `write_offsets()` 处理偏移并写入 xref 条目
5. 写入 `trailer` 字典
6. 调用 `finish_trailer()` 写入 `startxref` 和 `%%EOF`

### 4.3 write_offsets() —— 核心偏移处理函数

在 [pdf-writer lib.rs#L437-L478](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L437-L478) 中：

```rust
fn write_offsets(mut offsets: Vec<(Ref, usize)>, writer: &mut impl XRefWriter) -> i32 {
    offsets.sort();  // 按 Ref 编号排序

    let xref_len = 1 + offsets.last().map_or(0, |p| p.0.get());
    writer.prologue(xref_len);

    if offsets.is_empty() {
        writer.write_free_entry(0, 65535);
    }

    let mut written = 0;
    for (i, (object_id, offset)) in offsets.iter().enumerate() {
        if written > object_id.get() {
            panic!("duplicate indirect reference id: {}", object_id.get());
        }

        // 填充空闲对象链表
        let start = written;
        for free_id in start..object_id.get() {
            let mut next = free_id + 1;
            if next == object_id.get() {
                // 查找下一个空闲 id
                for (used_id, _) in &offsets[i..] {
                    if next < used_id.get() {
                        break;
                    } else {
                        next = used_id.get() + 1;
                    }
                }
            }

            let gen = if free_id == 0 { 65535 } else { 0 };
            writer.write_free_entry((next % xref_len) as usize, gen);
            written += 1;
        }

        writer.write_occupied_entry(*offset, 0);
        written += 1;
    }

    xref_len
}
```

**关键逻辑：**
1. **排序**：`offsets.sort()` 确保按对象编号顺序处理
2. **计算 xref 长度**：`xref_len = 1 + max_id`，包含对象 0
3. **空闲链表构建**：
   - 对象 0 的生成号为 65535（表示头节点）
   - 其他空闲对象的生成号为 0
   - `next` 指向下一个空闲对象，形成循环链表
4. **重复检测**：如果 `written > object_id.get()` 说明有重复编号，触发 panic

### 4.4 PlainXRefWriter —— 传统 XRef 表格式

在 [pdf-writer lib.rs#L570-L594](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L570-L594) 中：

```rust
struct PlainXRefWriter<'a> {
    buf: &'a mut Buf,
}

impl<'a> XRefWriter for PlainXRefWriter<'a> {
    fn prologue(&mut self, xref_len: i32) {
        self.buf.extend(b"xref\n0 ");
        self.buf.push_int(xref_len);
        self.buf.push(b'\n');
    }

    fn write_free_entry(&mut self, offset: usize, gen_number: u16) {
        write!(self.buf.inner, "{offset:010} {gen_number:05} f\r\n").unwrap();
    }

    fn write_occupied_entry(&mut self, offset: usize, gen_number: u16) {
        write!(self.buf.inner, "{offset:010} {gen_number:05} n\r\n").unwrap();
    }
}
```

**传统 XRef 表格式示例：**
```
xref
0 3
0000000000 65535 f\r
0000000016 00000 n\r
0000000094 00000 n\r
```

**格式说明：**
- 偏移量：10 位十进制，前导零
- 生成号：5 位十进制，前导零
- 类型：`f` 表示空闲，`n` 表示占用
- 行尾：`\r\n`（PDF 要求的行结束符）

### 4.5 XRefStreamWriter —— 压缩 XRef 流（PDF 1.5+）

在 [pdf-writer lib.rs#L533-L568](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L533-L568) 中：

```rust
struct XRefStreamWriter {
    buf: Vec<u8>,
    field_width: u32,
}

impl XRefStreamWriter {
    fn write(&mut self, entry_type: u8, offset: usize, gen_number: u16) {
        let offset_bytes = (offset as u64).to_be_bytes();

        self.buf.push(entry_type);  // 1 字节: 0=空闲, 1=占用, 2=对象流
        self.buf.extend(
            offset_bytes
                .iter()
                .skip(offset_bytes.len() - self.field_width as usize),
        );  // N 字节: 偏移量
        self.buf.extend_from_slice(&gen_number.to_be_bytes());  // 2 字节: 生成号
    }
}

impl XRefWriter for XRefStreamWriter {
    fn prologue(&mut self, _: i32) {}

    fn write_free_entry(&mut self, offset: usize, gen_number: u16) {
        self.write(0, offset, gen_number);
    }

    fn write_occupied_entry(&mut self, offset: usize, gen_number: u16) {
        self.write(1, offset, gen_number);
    }
}
```

**压缩 XRef 流条目格式：**
```
[类型(1字节)] [偏移量(N字节)] [生成号(2字节)]
```

**类型值：**
- `0`: 空闲对象
- `1`: 普通占用对象
- `2`: 对象流中的对象（需要额外字段）

### 4.6 XRef 流写入流程（PDF 1.5+）

在 [pdf-writer lib.rs#L365-L415](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L365-L415) 中：

```rust
fn finish_with_xref_stream_inner(
    self,
    xref_id: Ref,
    filter: impl FnOnce(Vec<u8>) -> (Vec<u8>, Option<XRefFilter>),
) -> Vec<u8> {
    let Chunk { mut buf, mut offsets, settings } = self.chunk;
    let trailer_data = self.trailer_data;

    // 注意：xref 流本身也要加入 offsets！
    let xref_offset = buf.len();
    offsets.push((xref_id, xref_offset));
    let field_width = determine_field_width(xref_offset);

    // 写入 xref 流内容
    let mut writer = XRefStreamWriter::new(field_width);
    let xref_len = write_offsets(offsets, &mut writer);

    // 应用过滤器（压缩）
    let (xref_data, filter) = filter(writer.buf);

    // 写入 xref 流对象
    let mut stream =
        Stream::start(Obj::indirect(&mut buf, xref_id, settings), &xref_data);

    stream.pair(Name(b"Type"), Name(b"XRef"));

    // 写入过滤器（如 FlateDecode）
    if let Some(filter) = filter { ... }

    // 写入 trailer 字段
    trailer_data.write_into_dict(stream.deref_mut(), xref_len);

    // 写入 /W 数组：[类型字节数, 偏移字节数, 生成号字节数]
    stream
        .insert(Name(b"W"))
        .array()
        .item(1)
        .item(field_width as i32)
        .item(2);

    stream.finish();

    finish_trailer(buf, xref_offset, &[])
}
```

**关键点：**
- xref 流本身也是一个间接对象，需要加入 `offsets` 向量
- `field_width` 由最大偏移量决定（`determine_field_width()`）
- `/W` 数组定义每个字段的字节数：[1, field_width, 2]

### 4.7 Trailer 生成

**TrailerData 结构** [pdf-writer lib.rs#L500-L525](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L500-L525)：

```rust
#[derive(Default)]
struct TrailerData {
    catalog_id: Option<Ref>,    // /Root
    info_id: Option<Ref>,       // /Info
    file_id: Option<(Vec<u8>, Vec<u8>)>,  // /ID
}

impl TrailerData {
    fn write_into_dict(&self, dict: &mut Dict, xref_len: i32) {
        dict.pair(Name(b"Size"), xref_len);

        if let Some(catalog_id) = self.catalog_id {
            dict.pair(Name(b"Root"), catalog_id);
        }

        if let Some(info_id) = self.info_id {
            dict.pair(Name(b"Info"), info_id);
        }

        if let Some(file_id) = &self.file_id {
            let mut ids = dict.insert(Name(b"ID")).array();
            ids.item(Str(&file_id.0));
            ids.item(Str(&file_id.1));
        }
    }
}
```

**Trailer 字典示例：**
```
trailer
<<
  /Size 10
  /Root 5 0 R
  /Info 8 0 R
  /ID [<0123456789ABCDEF> <0123456789ABCDEF>]
>>
```

**Trailer 字段说明：**
- `/Size`: xref 表中的条目总数（包括对象 0）
- `/Root`: 文档目录（Catalog）的间接引用
- `/Info`: 文档信息字典的间接引用（可选）
- `/ID`: 文件标识符数组，两个相同的字节串（PDF/A 必需）

### 4.8 startxref 与 %%EOF 生成

**finish_trailer() 函数** [pdf-writer lib.rs#L426-L435](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L426-L435)：

```rust
fn finish_trailer(mut buf: Buf, xref_offset: usize, pad: &[u8]) -> Vec<u8> {
    buf.extend(pad);
    // 写入 startxref 指向 xref 表/流的起始偏移
    buf.extend(b"startxref\n");
    write!(buf.inner, "{}", xref_offset).unwrap();

    // 写入 EOF 标记
    buf.extend(b"\n%%EOF");
    buf.into_vec()
}
```

**最终文件尾部示例：**
```
startxref
1234
%%EOF
```

**关键点：**
- `xref_offset` 是 xref 表（或 xref 流）在文件中的字节偏移
- `startxref` 后面跟着的是十进制偏移量
- `%%EOF` 是 PDF 文件的标准结束标记

### 4.9 Chunk::extend() —— 偏移调整

当合并多个 Chunk 时，偏移量需要调整 [pdf-writer chunk.rs#L88-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L88-L93)：

```rust
pub fn extend(&mut self, other: &Chunk) {
    let base = self.len();
    self.buf.extend_buf(&other.buf);
    self.offsets
        .extend(other.offsets.iter().map(|&(id, offset)| (id, base + offset)));
}
```

**偏移调整逻辑：**
- `base = self.len()`: 当前 chunk 的字节长度
- `base + offset`: 其他 chunk 的偏移量需要加上这个基址

### 4.10 XRef 表生成完整流程图

```
krilla chunk_container.finish()
    │
    ├─ 第一遍：二次编号重映射（见 3.4 节）
    │   └─ 建立 old_ref → new_ref 映射
    │
    ├─ 第二遍：renumber_into 写入所有对象
    │   └─ Chunk::renumber_into()  [pdf-writer chunk.rs#L181]
    │       └─ renumber::renumber() 重写所有引用
    │           └─ 调用 target.indirect(new_ref) 写入对象
    │               └─ target.offsets.push((new_ref, buf.len()))  ← 记录偏移!
    │
    └─ pdf.finish()  [pdf-writer lib.rs#L307]
        │
        ├─ xref_offset = buf.len()  ← 记录 xref 起始位置
        │
        ├─ write_offsets(offsets, writer)  [pdf-writer lib.rs#L437]
        │   ├─ offsets.sort()
        │   ├─ xref_len = 1 + max_id
        │   ├─ writer.prologue(xref_len)  →  "xref\n0 N\n"
        │   ├─ 填充空闲对象链表
        │   └─ 写入占用对象条目
        │
        ├─ 写入 trailer 字典
        │   ├─ /Size: xref_len
        │   ├─ /Root: catalog_ref
        │   ├─ /Info: info_ref (可选)
        │   └─ /ID: [file_id file_id] (可选)
        │
        └─ finish_trailer(buf, xref_offset, pad)
            ├─ "startxref\n"
            ├─ xref_offset (十进制)
            └─ "\n%%EOF"
```

---

## 五、字体子集化 —— 基于源码的分析

### 5.1 字形收集阶段

字形收集在内容流构建时进行，而非在 `finish()` 阶段。

**FontContainer 结构** [text/mod.rs#L54-L72](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/mod.rs#L54-L72)：

```rust
pub(crate) struct FontContainer {
    font: Font,
    type3_mapper: Type3FontMapper,
    cid_font: CIDFont,           // ← 核心：包含 GlyphRemapper
    cid_cache: FxHashMap<u32, (FontIdentifier, PDFGlyph)>,
    type3_cache: HashMap<ColoredGlyph, (FontIdentifier, PDFGlyph)>,
}

impl FontContainer {
    #[inline]
    pub(crate) fn add_glyph(&mut self, glyph: ColoredGlyph) -> (FontIdentifier, PDFGlyph) {
        if let Some(e) = self.cid_cache.get(&glyph.glyph_id.to_u32()) {
            return e.clone();  // 缓存命中
        } else if should_outline(&self.font, glyph.glyph_id) {
            let cid = self.cid_font.add_glyph(glyph.glyph_id);  // ← 字形注册
            let res = (self.cid_font.identifier(), PDFGlyph::Cid(cid));
            self.cid_cache.insert(glyph.glyph_id.to_u32(), res.clone());
            res
        } else {
            // Type3 字体路径（颜色字体等）
        }
    }
}
```

**CIDFont::add_glyph()** [text/cid.rs#L154-L170](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L154-L170)：

```rust
/// Add a new glyph (if it has not already been added) and return its CID.
#[inline]
pub(crate) fn add_glyph(&mut self, glyph_id: GlyphId) -> Cid {
    self.is_empty = false;

    let new_id = self
        .glyph_remapper
        .remap(u16::try_from(glyph_id.to_u32()).unwrap());

    // If it's a new glyph, add its width
    if new_id as usize >= self.widths.len() {
        self.widths.push(self.font.advance_width(glyph_id).unwrap_or(0.0));
    }

    new_id
}
```

**关键点：**
- 使用 `subsetter::GlyphRemapper` 维护原字形 ID → 子集 CID 的映射
- `.notdef` 字形（GID 0）始终被包含 [text/cid.rs#L122-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L122-L124)
- 同时收集字形宽度数组

### 5.2 字形绘制时的收集触发

在 krilla [content.rs#L530-L580](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/content.rs#L530-L580) 的 `encode_consecutive_glyph_run()` 中：

```rust
fn encode_consecutive_glyph_run(
    &mut self,
    sc: &mut SerializeContext,
    cur_x: &mut f32,
    cur_y: f32,
    font_identifier: FontIdentifier,
    pdf_font: &dyn PdfFont,
    size: f32,
    context_color: rgb::Color,
    glyphs: &[impl Glyph],
    text: &str,
) {
    // 1. 注册字体资源
    let font_name = self.rd_builder.register_resource(
        sc.register_font_identifier(font_identifier)
    );
    self.content.set_font(font_name.to_pdf_name(), size);
    
    // 2. 检查 .notdef 字形使用
    for glyph in glyphs {
        if glyph.glyph_id() == GlyphId::new(0) {
            sc.register_validation_error(ValidationError::ContainsNotDefGlyph(...));
        }
    }
    
    // 3. 编码字形到内容流
    if let [glyph] = glyphs {
        self.encode_single_glyph(cur_x, pdf_font, size, context_color, glyph);
    } else {
        self.encode_glyphs_with_individual_positioning(...);
    }
}
```

### 5.3 字体子集化执行

在 krilla [text/cid.rs#L237-L239](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L237-L239) 的 `serialize()` 方法中：

```rust
let (subsetted, global_bbox) = subset_font(self.font.clone(), glyph_remapper)?;
let num_glyphs = subsetted.num_glyphs();
let subsetted_data = subsetted.font_data().0;
```

**subset_font() 函数** [text/cid.rs#L436-L458](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L436-L458)：

```rust
#[cfg_attr(feature = "comemo", comemo::memoize)]
fn subset_font(font: Font, glyph_remapper: &GlyphRemapper) -> KrillaResult<(Font, Rect)> {
    let variation_coordinates = font.variation_coordinates()
        .iter()
        .map(|v| (subsetter::Tag::new(v.0.get()), v.1.get()))
        .collect::<Vec<_>>();
    
    let font = subsetter::subset_with_variations(
        font.font_data().as_ref(),
        font.index(),
        &variation_coordinates,
        glyph_remapper,
    )
    .map_err(|e| KrillaError::Font(font.clone(), format!("failed to subset font: {e}")))
    .and_then(|data| {
        Font::new(Arc::new(data).into(), 0)
            .ok_or(KrillaError::Font(font.clone(), "failed to subset font".to_string()))
    })?;
    
    let global_bbox = font.bbox();
    Ok((font, global_bbox))
}
```

**subsetter 库功能（来自导入）：**
- `subsetter::GlyphRemapper`：跟踪使用的字形
- `subsetter::subset_with_variations()`：执行实际的子集化
- 支持可变字体变化轴应用

### 5.4 子集化后的数据处理

在 krilla [text/cid.rs#L241-L254](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L241-L254) 中：

```rust
let font_stream = {
    let mut data = subsetted_data.as_ref().as_ref();

    // If we have a CFF font, only embed the standalone CFF program.
    let subsetted_ref = skrifa::FontRef::new(data).map_err(|_| {
        KrillaError::Font(self.font.clone(), "failed to read font subset".to_string())
    })?;

    if let Some(cff) = subsetted_ref.data_for_tag(Cff::TAG) {
        data = cff.as_bytes();  // 提取独立的 CFF 程序
    }

    FilterStreamBuilder::new_from_binary_data(data).finish(&sc.serialize_settings())
};
```

**字体格式特殊处理：**
- **TTF 字体**：嵌入完整的子集化字体文件，使用 `/FontFile2`
- **CFF 字体**：提取独立的 CFF 表嵌入，使用 `/FontFile3`
- **CFF2 字体**：类似 CFF，但支持可变字体

### 5.5 字体对象体系构建

在 krilla [text/cid.rs#L263-L398](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L263-L398) 中构建完整的字体对象体系：

```
Type0 Font (根字体对象)
├── /BaseFont: "AAAAAA+FontName"
├── /Encoding: /Identity-H
├── /DescendantFonts: [ CIDFont 引用 ]
└── /ToUnicode: CMap 流引用
    ↓
CIDFont 对象
├── /Subtype: CIDFontType0 (CFF) 或 CIDFontType2 (TTF)
├── /BaseFont: 同上
├── /CIDSystemInfo: {Registry: "Adobe", Ordering: "Identity", Supplement: 0}
├── /FontDescriptor: 字体描述符引用
├── /W: 宽度数组
└── /CIDToGIDMap: /Identity (仅 TTF)
    ↓
FontDescriptor 对象
├── /FontName
├── /Flags
├── /FontBBox
├── /ItalicAngle
├── /Ascent, /Descent, /CapHeight
├── /StemV
├── /FontFile2 或 /FontFile3: 字体文件流引用
└── /CIDSet: CIDSet 流引用 (PDF < 1.7)
```

**字体子集标签生成** [text/cid.rs#L413-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L413-L434)：

```rust
pub(crate) fn subset_tag<T: Hash>(data: &T) -> String {
    const BASE: u128 = 26;
    let mut hash = stable_hash128(data);
    let mut letter = [b'A'; SUBSET_TAG_LEN];
    // 6 个大写字母的子集标签，如 "AAAAAA"
    for i in 0..SUBSET_TAG_LEN {
        letter[i] = b'A' + (hash % BASE) as u8;
        hash /= BASE;
    }
    std::str::from_utf8(&letter).unwrap().to_string()
}

fn base_font_name(font: &Font, data: impl Hash) -> String {
    let postscript_name = font.postscript_name().unwrap_or("unknown");
    let max_len = 127 - REST_LEN;  // REST_LEN = 7 (6 + "+" + 可能的 "-Identity-H")
    let trimmed = &postscript_name[..postscript_name.len().min(max_len)];
    let subset_tag = subset_tag(&data);
    format!("{subset_tag}+{trimmed}")
}
```

**最终字体名称格式：**
- TTF: `AAAAAA+ArialMT`
- CFF: `AAAAAA+ArialMT-Identity-H`

---

## 六、ToUnicode CMap 生成 —— 基于源码的分析

### 6.1 码位映射收集

码位映射在字形编码时收集，通过 `Glyph::text_range()` 方法获取。

在 krilla [content.rs#L566-L569](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/content.rs#L566-L569) 的 `encode_single_glyph()` 中（推断）：

```rust
fn encode_single_glyph(
    &mut self,
    cur_x: &mut f32,
    pdf_font: &dyn PdfFont,
    size: f32,
    context_color: rgb::Color,
    glyph: &impl Glyph,
) {
    // ... 字形绘制 ...
    
    // 记录 Unicode 映射
    let cid = pdf_font.get_gid(ColoredGlyph::new(glyph.glyph_id(), context_color)).unwrap();
    let text_range = glyph.text_range();
    let text = &text[text_range.clone()];
    pdf_font.set_codepoints(cid, text.to_string(), glyph.location());
    
    self.content.show_str(glyph_id);  // Tj 操作符
}
```

**CIDFont 中的码位存储** [text/cid.rs#L112-L180](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L112-L180)：

```rust
pub(crate) struct CIDFont {
    // ...
    /// A mapping from CIDs to their string in the original text.
    cmap_entries: FxHashMap<u16, (String, Option<Location>)>,
    // ...
}

impl CIDFont {
    #[inline]
    pub(crate) fn set_codepoints(&mut self, cid: Cid, text: String, location: Option<Location>) {
        self.cmap_entries.insert(cid, (text, location));
    }
}
```

### 6.2 ToUnicode CMap 写入

在 krilla [text/cid.rs#L378-L398](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L378-L398) 中：

```rust
let cmap = {
    let mut cmap = UnicodeCmap::new(CMAP_NAME, SYSTEM_INFO);

    // For the .notdef glyph, it's fine if no mapping exists, since it is included
    // even if it was not referenced in the text.
    for g in 1..self.glyph_remapper.num_gids() {
        let entry = self.cmap_entries.get(&g);
        write_cmap_entry(&self.font, entry, sc, &mut cmap, g);
    }

    cmap
};

let cmap_stream = cmap.finish();
let cmap_stream = FilterStreamBuilder::new_from_content_stream(
    &cmap_stream, &sc.serialize_settings()
).finish(&sc.serialize_settings());

let mut cmap = stream_chunk.cmap(cmap_ref, cmap_stream.encoded_data());
cmap_stream.write_filters(cmap.deref_mut().deref_mut());
cmap.writing_mode(WMode::Horizontal);
cmap.finish();
```

**注意：** 跳过 `.notdef` 字形（g=0）的映射，因为它不对应实际文本。

### 6.3 write_cmap_entry() 函数详解

在 krilla [text/cid.rs#L40-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L40-L98) 中实现了完整的码位映射和验证逻辑：

```rust
pub(crate) fn write_cmap_entry<G>(
    font: &Font,
    entry: Option<&(String, Option<Location>)>,
    sc: &mut SerializeContext,
    cmap: &mut UnicodeCmap<G>,
    g: G,
) where
    G: pdf_writer::types::GlyphId + Into<u32> + Copy,
{
    match entry {
        None => sc.register_validation_error(ValidationError::NoCodepointMapping(
            font.clone(), GlyphId::new(g.into()), None,
        )),
        Some((text, loc)) => {
            let mut invalid_codepoint = text.is_empty();
            let mut invalid_code = None;
            let mut private_unicode = None;

            for c in text.chars() {
                // 检查无效码位：NUL, BOM, 反向 BOM
                if matches!(c as u32, 0x0 | 0xFEFF | 0xFFFE) {
                    invalid_code = Some(c);
                    invalid_codepoint = true;
                }
                // 检查私有使用区域
                if matches!(c as u32, 0xE000..=0xF8FF | 0xF0000..=0xFFFFD | 0x100000..=0x10FFFD) {
                    private_unicode = Some(c);
                }
            }

            // 报告无效码位错误
            match invalid_code {
                Some(c) => sc.register_validation_error(ValidationError::InvalidCodepointMapping(...)),
                None if invalid_codepoint => sc.register_validation_error(
                    ValidationError::NoCodepointMapping(...)),
                _ => {}
            }

            // 报告私有使用区域警告
            if let Some(code) = private_unicode {
                sc.register_validation_error(ValidationError::UnicodePrivateArea(...));
            }

            // 写入实际的 CMap 映射
            if !text.is_empty() {
                cmap.pair_with_multiple(g, text.chars());
            }
        }
    }
}
```

**验证逻辑：**
1. **缺少映射**：字形没有对应的 Unicode 码位
2. **无效码位**：`\0`, `\u{FEFF}` (BOM), `\u{FFFE}` (反向 BOM)
3. **私有使用区域**：PUA 码位可能导致文本提取问题
4. **复杂脚本限制**：阿拉伯文等可能无法生成有效映射

### 6.4 ToUnicode CMap 格式

生成的 CMap 使用 `pdf-writer` 的 `UnicodeCmap` 构建器，输出标准的 CMap 格式：

```
/CIDInit /ProcSet findresource begin
12 dict begin
begincmap
/CIDSystemInfo
<< /Registry (Adobe)
   /Ordering (UCS)
   /Supplement 0
>> def
/CMapName /Custom def
/CMapType 2 def
1 begincodespacerange
<0000> <FFFF>
endcodespacerange
n beginbfchar
<0001> <0048>    % CID 1 → 'H'
<0002> <0065>    % CID 2 → 'e'
<0003> <006C>    % CID 3 → 'l'
<0004> <006C>    % CID 4 → 'l'
<0005> <006F>    % CID 5 → 'o'
...
endbfchar
n beginbfrange
<0006> <000A> <0020>  % CID 6-10 → ' '-'E'
...
endbfrange
endcmap
CMapName currentdict /CMap defineresource pop
end
end
```

**使用的 CMap 操作符：**
- `beginbfchar` / `endbfchar`：单个字形→字符映射
- `beginbfrange` / `endbfrange`：字形范围→字符范围映射（优化体积）

---

## 七、从页面内容到 PDF 字节的完整链路 —— 基于源码

### 7.1 阶段 1：初始化与准备

**SerializeContext 创建** [serialize.rs#L279-L310](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L279-L310)：

```
Document::new_with(settings)
  └─ SerializeContext::new(settings)
      ├─ cur_ref = Ref::new(1)
      ├─ page_tree_ref = cur_ref.bump()  // Ref(2)
      ├─ 预分配命名空间对象 (3, 4)
      ├─ cur_ref = Ref(5)  // 下一个可用编号
      └─ 初始化 GlobalObjects 容器
```

### 7.2 阶段 2：页面内容绘制（字形收集）

**调用链：**
```
Surface::draw_glyphs()  [surface.rs#L275-L367]
  └─ ContentBuilder::draw_glyphs()  [content.rs#L383-L524]
      ├─ fill_action() / stroke_action()
      │   └─ content_set_fill_properties()  [color/gradient/pattern 处理]
      └─ fill_stroke_glyph_run()
          └─ encode_consecutive_glyph_run()  [content.rs#L530-L580]
              ├─ sc.register_font_identifier(font_identifier)  [serialize.rs#L640-L649]
              │   └─ 检查 cached_mappings → new_ref() 或复用
              ├─ rd_builder.register_resource(font_resource)  [resource.rs#L197-L202]
              │   └─ fonts.remap_with_name(ref)  → "/F0"
              ├─ content.set_font(font_name, size)
              ├─ encode_single_glyph() / encode_glyphs_with_individual_positioning()
              │   ├─ pdf_font.add_glyph(colored_glyph)  [text/mod.rs#L120-L139]
              │   │   └─ cid_font.add_glyph(glyph_id)  [text/cid.rs#L154-L170]
              │   │       └─ glyph_remapper.remap(gid)  ← 字形收集!
              │   ├─ pdf_font.set_codepoints(cid, text, location)
              │   │   └─ cmap_entries.insert(cid, (text, loc))  ← 码位收集!
              │   └─ content.show_str(glyph_id)
              └─ *cur_x += glyph.x_advance
```

**字形收集数据结构：**
```
GlyphRemapper (subsetter 库)
  ├─ forward: Vec<u16>          // 新 GID → 原 GID
  └─ backward: HashMap<u16, u16> // 原 GID → 新 GID

CIDFont
  ├─ glyph_remapper: GlyphRemapper
  ├─ cmap_entries: FxHashMap<u16, (String, Option<Location>)>
  └─ widths: Vec<f32>
```

### 7.3 阶段 3：页面完成与资源冻结

**Page 完成时：**
```
surface.finish()
  └─ ResourceDictionaryBuilder::finish()  [resource.rs#L204-L213]
      ├─ 所有 ResourceMapper 转为 ResourceList
      └─ 生成 ResourceDictionary
          ├─ fonts: ResourceList<Font>
          ├─ x_objects: ResourceList<XObject>
          └─ ...
```

### 7.4 阶段 4：document.finish() —— 最终序列化

在 krilla [serialize.rs#L449-L498](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L449-L498) 中：

```
SerializeContext::finish(chunk_container)
  │
  ├─ serialize_destination_profiles()      // ICC 输出配置
  ├─ serialize_page_label_tree()           // 页码标签
  ├─ serialize_outline()                   // 文档大纲
  ├─ serialize_fonts()  ← 关键点！
  │   └─ 对每个字体调用 FontContainer.serialize()
  │       └─ CIDFont::serialize()  [text/cid.rs#L187-L398]
  │           ├─ 分配 5 个新 Ref: cid, descriptor, cmap, cidset, data
  │           ├─ subset_font(font, glyph_remapper)  ← 字体子集化!
  │           │   └─ subsetter::subset_with_variations()
  │           ├─ 写入 Type0 Font
  │           │   └─ .to_unicode(cmap_ref)  ← ToUnicode 引用
  │           ├─ 写入 CIDFont
  │           ├─ 写入 FontDescriptor
  │           │   └─ .font_file2/3(data_ref)
  │           ├─ 构建 ToUnicode CMap  [text/cid.rs#L378-L398]
  │           │   └─ UnicodeCmap + write_cmap_entry()
  │           └─ 写入字体文件流
  ├─ serialize_pages()                     // 页面对象 + 注解
  ├─ serialize_page_tree()                 // Pages 字典
  ├─ serialize_tag_tree()                  // 标签树（无障碍）
  │
  └─ chunk_container.finish(sc)  [chunk_container.rs#L100-L348]
      │
      ├─ 第一遍：二次编号重映射
      │   ├─ remapped_ref = Ref::new(1)
      │   └─ visit all chunks:
      │       └─ for object_ref in chunk.refs():
      │           remapper.insert(object_ref, remapped_ref.bump())
      │
      ├─ 创建 Pdf 写入器
      │   └─ Pdf::with_settings_and_capacity()
      │
      ├─ 第二遍：重写所有对象到 Pdf
      │   └─ visit all chunks:
      │       chunk.renumber_into(&mut pdf, |old| remapper[&old])
      │       └─ pdf-writer renumber.rs 重写所有引用
      │           └─ target.indirect(new_ref) 写入对象
      │               └─ target.offsets.push((new_ref, buf.len()))  ← 记录偏移!
      │
      ├─ 写入文档信息 (DocumentInfo)
      ├─ 生成文件 ID (stable_hash_base64)
      ├─ 写入 XMP 元数据流
      └─ 写入文档目录 (Catalog)
          ├─ /Pages → page_tree_ref
          ├─ /Metadata → meta_ref
          ├─ /StructTreeRoot → tag_tree_ref
          ├─ /Outlines → outline_ref
          ├─ /Names → (命名目标 + 附件)
          └─ /Lang, /ViewerPreferences 等
      
      └─ 返回 Pdf 对象
          └─ pdf.finish() 生成 xref 表和 trailer
```

### 7.5 阶段 5：PDF 字节生成

当 `Pdf` 对象调用 `finish()` 时，`pdf-writer` 库自动执行以下步骤 [pdf-writer lib.rs#L307-L322](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L307-L322)：

1. **写入头部**：`%PDF-1.7\n%\xe2\xe3\xcf\xd3\n`
2. **从 Chunk 提取数据**：`buf`, `offsets`, `settings`
3. **记录 xref 起始偏移**：`xref_offset = buf.len()`
4. **构建 xref 表**：
   - 调用 `write_offsets(offsets, &mut writer)` [pdf-writer lib.rs#L437-L478](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L437-L478)
   - 排序 offsets，按编号顺序处理
   - 填充空闲对象链表
   - 写入每个对象的字节偏移位置（10 位十进制）
   - 写入每个对象的生成号（5 位十进制，通常为 0）
5. **写入 trailer** [pdf-writer lib.rs#L508-L525](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L508-L525)：
   - `/Size`：对象总数（包含对象 0）
   - `/Root`：Catalog 引用
   - `/Info`：DocumentInfo 引用（可选）
   - `/ID`：文件 ID 数组（可选）
6. **写入 startxref** [pdf-writer lib.rs#L426-L435](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L426-L435)：
   - `startxref\n` + xref 表的起始偏移（十进制）
7. **写入 `%%EOF`**：文件结束标记

### 7.6 完整数据流图

```
Typography → PagedDocument
     │
     ▼
typst-pdf convert.rs
     │
     ├─ GlobalContext::new()
     │   └─ 字体缓存初始化
     │
     └─ 逐页处理:
         ├─ document.start_page_with()
         │   └─ sc.register_page() → new_ref()
         ├─ page.surface()
         │   └─ Surface::new()
         │       ├─ ContentBuilder::new()
         │       └─ ResourceDictionaryBuilder::new()
         ├─ handle_frame() (递归)
         │   ├─ handle_text()
         │   │   ├─ convert_font() [text.rs#L62-L76]
         │   │   ├─ paint::convert_fill()
         │   │   └─ surface.draw_glyphs() [surface.rs#L275]
         │   │       └─ 见 7.2 的字形收集流程
         │   ├─ handle_shape()
         │   │   └─ surface.draw_path()
         │   └─ handle_image()
         │       └─ surface.draw_image()
         ├─ surface.finish()
         │   └─ rd_builder.finish() → ResourceDictionary
         └─ page.finish()
     │
     ▼
SerializeContext::finish()  [serialize.rs#L449]
     │
     ├─ serialize_fonts()
     │   └─ CIDFont::serialize()  [text/cid.rs#L187]
     │       ├─ 字体子集化 (subsetter)
     │       ├─ 字体描述符
     │       ├─ ToUnicode CMap 生成
     │       └─ 字体文件流嵌入
     │
     └─ chunk_container.finish()  [chunk_container.rs#L100]
         ├─ 二次编号重映射
         ├─ 写入所有对象到 Pdf
         ├─ xref 表生成 (pdf-writer 内部)
         └─ 返回 Vec<u8>
```

---

## 八、性能优化与缓存机制

### 8.1 对象去重缓存

在 krilla [serialize.rs#L573-L587](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L573-L587) 中：

```rust
fn register_cached<T: SipHashable>(
    &mut self,
    item: T,
    mut func: impl FnMut(&mut Self, T, Ref),
) -> Ref {
    let hash = item.sip_hash();
    if let Some(_ref) = self.cached_mappings.get(&hash) {
        *_ref  // 缓存命中，复用现有引用
    } else {
        let root_ref = self.new_ref();
        func(self, item, root_ref);
        self.cached_mappings.insert(hash, root_ref);
        root_ref
    }
}
```

**支持缓存的对象类型：**
- 字体 (`register_font_identifier`)
- 图像 (`register_image`)
- 颜色空间 (`register_colorspace`)
- XYZ 目的地 (`register_xyz_destination`)
- 其他 `Cacheable` 对象

### 8.2 comemo memoization

krilla 支持使用 `comemo` 库缓存纯函数结果：

- `#[cfg_attr(feature = "comemo", comemo::memoize)]` 标记 `subset_font()`
- 相同字体 + 相同 GlyphRemapper 的子集化结果会被缓存
- 适用于多次导出相同文档的场景

---

## 九、PDF 标准合规性

### 9.1 字体相关验证点

在 krilla [configure/validate.rs] 和 [text/cid.rs#L40-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L40-L98) 中实现：

| 验证项 | 错误类型 | 触发条件 |
|-------|---------|---------|
| 字体许可证 | `RestrictedLicense` | OS/2.fsType 字段设置为受限（2） |
| .notdef 字形使用 | `ContainsNotDefGlyph` | 实际文本使用了 GID 0 |
| 缺少码位映射 | `NoCodepointMapping` | 字形没有对应 Unicode |
| 无效码位 | `InvalidCodepointMapping` | 映射包含 NUL/BOM 等 |
| 私有使用区域 | `UnicodePrivateArea` | 使用了 PUA 码位 |

### 9.2 许可证检查逻辑

在 krilla [text/cid.rs#L228-L235](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L228-L235) 中：

```rust
// 检查 OS/2 表的 fsType 字段
if self.font.font_ref().os2()
    .is_ok_and(|os2| os2.fs_type() & 0xF == 2)
{
    sc.register_validation_error(
        ValidationError::RestrictedLicense(self.font.clone())
    );
}
```

**注意：** OpenType 规范要求 `fsType & 0xF == 2`（只有 bit 1 置位）才是严格受限，而非 `fsType & 2 != 0`。

---

## 十、关键代码引用汇总

| 功能 | krilla 源码位置 | pdf-writer 源码位置 |
|-----|----------------|-------------------|
| 资源字典构建 | [resource.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/resource.rs) | - |
| 引用编号分配 | [serialize.rs#L368-L370](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/serialize.rs#L368-L370) | - |
| 二次编号重映射 | [chunk_container.rs#L100-L120](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/chunk_container.rs#L100-L120) | [chunk.rs#L181-L187](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L181-L187) |
| 对象偏移记录 | - | [chunk.rs#L34-L38](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L34-L38), [chunk.rs#L193-L196](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/chunk.rs#L193-L196) |
| XRef 表生成 | - | [lib.rs#L307-L322](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L307-L322), [lib.rs#L437-L478](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L437-L478) |
| XRef 流生成 | - | [lib.rs#L365-L415](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L365-L415) |
| Trailer 生成 | - | [lib.rs#L500-L525](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L500-L525) |
| startxref 生成 | - | [lib.rs#L426-L435](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/pdf-writer-0.15.0/src/lib.rs#L426-L435) |
| 字形收集 | [text/cid.rs#L154-L170](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L154-L170) | - |
| 字体子集化 | [text/cid.rs#L436-L458](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L436-L458) | - |
| ToUnicode 生成 | [text/cid.rs#L378-L398](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L378-L398) | - |
| 码位验证 | [text/cid.rs#L40-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/133-typst/krilla-src/krilla-0.8.2/src/text/cid.rs#L40-L98) | - |

---

## 总结

### 字体嵌入完整流程（基于源码）

```
Typst FontInstance
     │
     ▼  convert_font() [text.rs#L62]
Krilla Font (包装原始数据)
     │
     ▼  surface.draw_glyphs() [surface.rs#L275]
ContentBuilder::draw_glyphs() [content.rs#L383]
     │
     ├─ encode_consecutive_glyph_run() [content.rs#L530]
     │   ├─ 注册字体资源 → /F0, /F1...
     │   ├─ FontContainer::add_glyph() [text/mod.rs#L120]
     │   │   └─ CIDFont::add_glyph() [text/cid.rs#L154]
     │   │       └─ glyph_remapper.remap(gid)
     │   ├─ pdf_font.set_codepoints() → cmap_entries 收集
     │   └─ 写入内容流操作符 (Tj/TJ)
     │
     ▼  serialize_fonts() [serialize.rs#L756]
CIDFont::serialize() [text/cid.rs#L187]
     │
     ├─ 分配对象编号: cid, desc, cmap, cidset, data
     ├─ subset_font() [text/cid.rs#L436]
     │   └─ subsetter::subset_with_variations() ← 子集化!
     ├─ 写入 Type0 Font 对象
     │   └─ .to_unicode(cmap_ref)
     ├─ 写入 CIDFont 对象 (宽度数组等)
     ├─ 写入 FontDescriptor (flags, bbox, metrics)
     ├─ 生成 ToUnicode CMap [text/cid.rs#L378]
     │   └─ UnicodeCmap + write_cmap_entry()
     ├─ 写入字体文件流 (FontFile2/3)
     └─ (可选) 写入 CIDSet 流
     │
     ▼  chunk_container.finish() [chunk_container.rs#L100]
二次编号重映射 → 写入所有对象 → xref 生成 (pdf-writer)
     │
     ├─ Chunk::indirect() → offsets.push((ref, buf.len()))
     ├─ write_offsets() → 排序 + 空闲链表 + xref 条目
     ├─ TrailerData::write_into_dict() → /Size, /Root, /Info, /ID
     ├─ finish_trailer() → startxref + %%EOF
     │
     ▼
PDF 字节流
```

### XRef 生成核心机制（基于 pdf-writer 源码）

```
对象写入阶段:
  Chunk::indirect(id)
    → offsets.push((id, buf.len()))  ← 记录偏移

最终生成阶段:
  Pdf::finish()
    ├─ xref_offset = buf.len()
    ├─ write_offsets(offsets, PlainXRefWriter)
    │   ├─ offsets.sort()
    │   ├─ 写入 "xref\n0 N\n"
    │   ├─ 为每个空闲对象写入: "0000000000 65535 f\r\n"
    │   └─ 为每个占用对象写入: "0000000016 00000 n\r\n"
    ├─ 写入 "trailer\n<< /Size N /Root X 0 R ... >>\n"
    └─ finish_trailer()
        ├─ "startxref\n"
        ├─ xref_offset (十进制)
        └─ "\n%%EOF"
```
