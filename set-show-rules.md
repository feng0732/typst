# Typst 模板中 Set 和 Show 规则的匹配机制分析

本文档从代码层面深入分析 Typst 中 `set` 和 `show` 规则的注册、作用域、匹配以及递归处理流程。

---

## 一、核心数据结构总览

在深入流程之前，先理清几个关键类型之间的关系：

```
StyleChain (链表式作用域)
  └── links: [&[LazyHash<Style>], ...]
        └── Style
              ├── Property      ← set 规则 / 直接构造产生
              ├── Recipe        ← show 规则产生
              └── Revocation    ← 撤销特定 Recipe
```

代码位置：[styles.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs)

---

## 二、规则注册阶段（解析 → 求值）

### 2.1 Set 规则的求值

Set 规则的求值入口在 [rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/rules.rs#L11-L35)：

```rust
impl Eval for ast::SetRule<'_> {
    type Output = Styles;

    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        // 1. 条件判断：if 条件为假则返回空 Styles
        if let Some(condition) = self.condition()
            && !condition.eval(vm)?.cast::<bool>().at(condition.span())?
        {
            return Ok(Styles::new());
        }

        // 2. 将 target 表达式求值为 Element
        let target = target_expr.eval(vm)?
            .cast::<Func>()
            .and_then(|func| func.to_element()
                .ok_or_else(|| "only element functions can be used in set rules".into()))?;

        // 3. 调用 Element::set → vtable.set → Set trait 的 set 方法
        //    该方法将参数解析为 Property 列表，包装为 Styles 返回
        let args = self.args().eval(vm)?.spanned(self.span());
        Ok(target.set(&mut vm.engine, args)?
            .spanned(self.span())
            .liftable())  // ← 标记为 liftable（set 规则特有）
    }
}
```

**关键点**：
- `liftable()` 标记是 set 规则与直接构造调用的核心区别：set 规则产生的样式允许被提升到页面级别，而直接构造（如 `text(red)[..]`）不允许
- `Element::set` 通过 vtable 动态分派到各元素的 `Set trait` 实现
- 结果类型是 `Styles`（即 `Vec<Style>`），不是立即应用，而是返回给上层

### 2.2 Show 规则的求值

Show 规则的求值同样在 [rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/rules.rs#L37-L64)：

```rust
impl Eval for ast::ShowRule<'_> {
    type Output = Recipe;

    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        // 1. 解析选择器（可选）：无选择器即为 "everything show rule"
        let selector = self.selector()
            .map(|sel| sel.eval(vm)?
                .cast::<ShowableSelector>()
                .map(|s| s.0))
            .transpose()?;

        // 2. 解析变换部分
        let transform = match transform {
            // 特殊情况：show-set 规则（show xxx: set yyy(...)）
            ast::Expr::SetRule(set) => Transformation::Style(set.eval(vm)?),
            // 其他情况：内容替换或函数变换
            expr => expr.eval(vm)?.cast::<Transformation>()?,
        };

        // 3. 封装为 Recipe
        Ok(Recipe::new(selector, transform, self.span()))
    }
}
```

**Recipe 结构**（[styles.rs#L447-L461](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs#L447-L461)）：
```rust
pub struct Recipe {
    selector: Option<Selector>,    // None = everything show rule
    transform: Transformation,     // Content | Func | Style(show-set)
    span: Span,
    outside: bool,                 // 是否在 show 规则之外应用
}
```

**Transformation 的三种形式**：
- `Content(Content)`：直接替换为固定内容
- `Func(Func)`：对匹配元素调用函数（最常用的 show 形式）
- `Style(Styles)`：show-set 规则，仅应用样式不改变结构

### 2.3 规则在 Markup/Code 流中的挂载

规则求值后，如何作用于后续内容？关键在 [markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/markup.rs#L26-L87) 和 [code.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/code.rs#L24-L72) 的递归求值：

**Markup 模式**：
```rust
fn eval_markup(vm: &mut Vm, exprs: &mut impl Iterator<Item = ast::Expr>) {
    while let Some(expr) = exprs.next() {
        match expr {
            ast::Expr::SetRule(set) => {
                let styles = set.eval(vm)?;
                // 递归求值剩余内容，然后把 styles 包在 StyledElem 外面
                let tail = eval_markup(vm, exprs)?;
                seq.push(tail.styled_with_map(styles))  // ← 产生 StyledElem
            }
            ast::Expr::ShowRule(show) => {
                let recipe = show.eval(vm)?;
                let tail = eval_markup(vm, exprs)?;
                // 有选择器：包在 StyledElem 中延迟处理
                // 无选择器：立即 eager apply
                seq.push(tail.styled_with_recipe(&mut vm.engine, vm.context, recipe)?)
            }
            // ...
        }
    }
}
```

**`styled_with_recipe` 的特殊处理**（[content/mod.rs#L321-L333](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/content/mod.rs#L321-L333)）：
```rust
pub fn styled_with_recipe(self, engine, context, recipe) -> SourceResult<Self> {
    if recipe.selector().is_none() {
        // 无选择器（everything show rule）→ 立即 eager apply！
        recipe.apply(engine, context, self)
    } else {
        // 有选择器 → 包成 StyledElem，延迟到 realization 阶段匹配
        Ok(self.styled(recipe))
    }
}
```

> **这就是为什么模板中的 `#show: template` 能立即生效**：它是无选择器 show 规则，在求值阶段就直接对整个后续内容调用了 `template(doc)` 函数。而 `#show heading: ...` 这样的有选择器规则则是延迟匹配。

---

## 三、作用域机制：StyleChain

### 3.1 StyleChain 的本质

`StyleChain` 是一个**不可变的链表结构**，用于高效表达嵌套作用域而无需复制样式：

[styles.rs#L557-L770](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs#L557-L770)

```rust
pub struct StyleChain<'a> {
    head: &'a [LazyHash<Style>],   // 当前层的样式切片
    tail: Option<&'a Self>,        // 外层作用域（链表指向）
}
```

**链构建**（`chain` 方法）：
```
// 初始：root_styles
chain1 = StyleChain { head: root_styles, tail: None }

// 遇到 StyledElem(child, local_styles)
chain2 = StyleChain { head: local_styles.0.as_slice(), tail: Some(&chain1) }
                    ↑ 新层在 head，优先级更高
```

### 3.2 样式查找顺序

`StyleChain::entries()` 迭代器按 **从内到外**（优先级从高到低）遍历：

[styles.rs#L825-L846](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs#L825-L846)
```rust
impl<'a> Iterator for Entries<'a> {
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            // 1. 当前层从后往前遍历（内层先加入的在后面）
            if let Some(entry) = self.inner.next_back() {
                return Some(entry);
            }
            // 2. 当前层遍历完，跳到外层 tail
            match self.links.next() {
                Some(next) => self.inner = next.iter(),
                None => return None,
            }
        }
    }
}
```

**属性查找优先级**：
1. 内层 StyledElem 的最后一个 Property
2. 内层 StyledElem 的前一个 Property
3. ... 内层全部遍历完
4. 外层 StyledElem 的最后一个 Property
5. ... 以此类推直到 root

### 3.3 折叠属性（Fold）

对于需要合并的属性（如 stroke 的 thickness + color），使用 `Fold` trait：

[styles.rs#L652-L664](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs#L652-L664)

```rust
fn get_folded<T>(self, func, id, fold, default) -> T {
    // 收集所有匹配的 Property，按内→外顺序 reduce fold
    let iter = self.properties(func, id)
        .map(|block| block.downcast::<T>().clone());
    iter.reduce(fold).map(|f| fold(f, default)).unwrap_or(default)
}
```

---

## 四、匹配判定阶段

### 4.1 Selector 类型体系

[selector.rs#L74-L104](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/selector.rs#L74-L104)

```rust
pub enum Selector {
    Elem(Element, Option<SmallVec<[(u8, Value); 1]>>),  // 元素 + 字段过滤
    Label(Label),                                           // 标签匹配
    Regex(Regex),                                           // 正则文本匹配
    Can(TypeId),                                            // 能力匹配（内部用）
    Or(EcoVec<Self>),                                       // 或组合
    And(EcoVec<Self>),                                      // 与组合
    Location(Location),                                     // 位置匹配
    Before / After / Within,                                // 查询专用
}
```

### 4.2 ShowableSelector 的验证

并非所有 Selector 都可用于 show 规则。`ShowableSelector` 做了验证：

[selector.rs#L508-L544](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/selector.rs#L508-L544)

```rust
fn validate(selector: &Selector, nested: bool) -> Result {
    match selector {
        Selector::Elem(_, _) => {}              // OK
        Selector::Label(_) => {}                 // OK
        Selector::Regex(_) if !nested => {}     // OK，但不能嵌套
        Selector::Or(list) | Selector::And(list) => {
            for s in list { validate(s, true)? } // 递归验证
        }
        // Location, Can, Before, After, Within → 全部禁止用于 show
        _ => bail!("this selector cannot be used with show"),
    }
}
```

### 4.3 matches 方法实现

[selector.rs#L131-L155](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/selector.rs#L131-L155)

```rust
pub fn matches(&self, target: &Content, styles: Option<StyleChain>) -> bool {
    match self {
        Self::Elem(element, dict) => {
            // 1. 元素类型必须匹配
            target.elem() == *element
            // 2. 如果有字段过滤条件，必须全部满足
            && dict.iter().flat_map(|d| d.iter()).all(|(id, value)| {
                target.get(*id, styles).as_ref().ok() == Some(value)
            })
        }
        Self::Label(label) => target.label() == Some(*label),
        Self::Can(cap) => target.func().can_type_id(*cap),
        Self::Or(sels) => sels.iter().any(|s| s.matches(target, styles)),
        Self::And(sels) => sels.iter().all(|s| s.matches(target, styles)),
        // Regex 不在此处匹配（见 4.4 节）
        _ => false,
    }
}
```

> **关键点**：`target.get(*id, styles)` 会从 StyleChain 中查找字段值，因此 `where` 子句可以匹配 set 规则设置的样式属性，而不仅仅是元素构造时传入的属性。

### 4.4 正则 Show 规则的特殊匹配

正则选择器 `Regex` 不在 `Selector::matches` 中处理，而是在 realization 的 **TEXTUAL 分组**阶段特殊处理：

[lib.rs#L1019-L1043](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L1019-L1043)

流程：
1. `TEXTUAL` 分组规则收集连续的 `TextElem / LinebreakElem / SmartQuoteElem`（中间可穿插 `SpaceElem`）
2. 在 `finish_textual` 中调用 `find_regex_match_in_elems`，将分组内的文本合并成字符串
3. 对 StyleChain 中所有正则 Recipe 尝试匹配，取最左匹配
4. 对匹配文本切割，生成新的 TextElem，再递归走 show 流程

---

## 五、Realization 阶段的递归处理

Realization 是整个规则系统的核心执行阶段。入口在 [lib.rs#L44-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L44-L76)。

### 5.1 整体遍历流程

```
visit(content, styles)
  ├─ 1. TagElem → 直接入 sink
  ├─ 2. visit_kind_rules → 基于 kind 的特殊变换
  ├─ 3. visit_show_rules → ★ 核心：show 规则匹配与应用
  │     ├─ verdict() → 决定：show-set 样式 + 一个 show step
  │     ├─ prepare() → 第一次处理：定位、合成、物化
  │     ├─ apply step → Recipe or Builtin
  │     └─ visit_styled → 递归处理应用后的 content
  ├─ 4. SequenceElem → 对每个 child 递归 visit
  ├─ 5. StyledElem → visit_styled（构建 StyleChain）
  ├─ 6. visit_grouping_rules → 分组规则（段落、列表、引用等）
  ├─ 7. visit_filter_rules → 过滤规则（空格、段距等）
  └─ 8. 其他元素 → 直接入 sink
```

### 5.2 verdict：选择要应用的规则

[lib.rs#L438-L531](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L438-L531)

这是整个匹配逻辑的核心：

```rust
fn verdict<'a>(engine, elem, styles) -> Option<Verdict<'a>> {
    let prepared = elem.is_prepared();
    let mut map = Styles::new();    // 收集 show-set 样式
    let mut step = None;            // 最多一个 transform show step

    // 0. 预合成：为了 where(kind: table) 这种匹配能工作
    if !prepared && elem.can::<dyn Synthesize>() {
        // 克隆 + synthesize 一次（开销较大但必须）
    }

    // 1. 遍历 StyleChain 中所有 Recipe（从内到外 = 优先级从高到低）
    for (r, recipe) in styles.recipes().enumerate() {
        // 1a. 选择器不匹配 → 跳过
        if !recipe.selector()
            .is_some_and(|sel| sel.matches(elem, Some(styles)))
        { continue; }

        // 1b. show-set 规则 → 收集到 map，不终止循环
        if let Transformation::Style(transform) = recipe.transform() {
            if !prepared {
                map.apply(transform.clone());
            }
            continue;
        }

        // 1c. 已经有 transform step → 不再找新的（一个元素一次只走一个 show rule）
        if step.is_some() { continue; }

        // 1d. 防止重复应用：检查 guard
        let index = RecipeIndex(*depth - r);  // 从顶向下编号
        if elem.is_guarded(index) { continue; }

        // 1e. 选中这个 Recipe 作为 step
        step = Some(ShowStep::Recipe(recipe, index));

        // 已 prepared → 找到就停；否则继续找 show-set
        if prepared { break; }
    }

    // 2. 没有用户 show rule → 用内置 Builtin rule
    if step.is_none() {
        let target = styles.get(TargetElem::target);
        if let Some(rule) = engine.library.rules.get(target, elem) {
            step = Some(ShowStep::Builtin(rule));
        }
    }

    Some(Verdict { prepared, map, step })
}
```

**关键设计决策**：
1. **每个元素每次只应用一个 transformational show rule**，应用后重新走 visit 流程，新的 content 可以再匹配其他规则
2. **show-set 规则会全部收集**，不阻止后续 show rule 的查找
3. **guard 机制**防止同一个 show rule 被重复应用：`Content.guarded(index)` 在 Recipe 应用时设置，下次 verdict 会跳过已 guard 的 index
4. **用户 show rule 优先于内置 builtin rule**

### 5.3 visit_show_rules：递归应用

[lib.rs#L354-L434](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L354-L434)

```rust
fn visit_show_rules(s, content, styles) -> Result<bool> {
    let Some(Verdict { prepared, mut map, step }) = verdict(...) else {
        return Ok(false);
    };

    let mut output = Cow::Borrowed(content);

    // 第一次处理 → prepare：定位、标签、合成、物化、标记 prepared
    let mut tags = None;
    if !prepared {
        tags = prepare(s.engine, s.locator, output.to_mut(), &mut map, styles)?;
    }

    // 应用选中的 step（Recipe 或 Builtin）
    if let Some(step) = step {
        let chained = styles.chain(&map);
        let result = match step {
            ShowStep::Recipe(recipe, guard) => {
                let context = Context::new(output.location(), Some(chained));
                // 对 output 设置 guard，防止递归时再次应用同一个 recipe
                recipe.apply(s.engine, context.track(),
                    output.into_owned().guarded(guard))
            }
            ShowStep::Builtin(rule) => {
                rule.apply(&output, s.engine, chained)
            }
        };
        output = Cow::Owned(s.engine.delay(result));
    }

    // ★ 递归：用新的 output + 合并的样式重新走处理流程
    // 这就是 show 规则可以链式匹配的原因！
    visit_styled(s, realized, Cow::Owned(map), styles)?;

    Ok(true)
}
```

**递归匹配的原理**：假设我们有：
```typst
#show strong: emph
#show emph: it => box(it)

*Hello*
```

流程：
1. `StrongElem` → verdict 匹配 `show strong` → 应用得到 `EmphElem`
2. 递归 visit 这个 `EmphElem` → verdict 匹配 `show emph` → 应用得到 `BoxElem`
3. 递归 visit `BoxElem` → 匹配内置 Builtin rule（布局）

每一步都通过 `.guarded(index)` 确保不会重复应用已经应用过的 Recipe。

### 5.4 prepare：元素的"第一次初始化"

[lib.rs#L534-L591](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L534-L591)

只在元素第一次被处理时执行：

```rust
fn prepare(engine, locator, elem, map, styles) -> Result<Option<(Tag, Tag)>> {
    // 1. 分配 Location（用于 introspection、引用、计数器）
    if elem.location().is_none() && flags.any() {
        let loc = locator.next_location(...);
        elem.set_location(loc);
    }

    // 2. 应用内置 ShowSet（比用户 show-set 晚，但比 Synthesize 早）
    if let Some(show_settable) = elem.with::<dyn ShowSet>() {
        map.apply(show_settable.show_set(styles));
    }

    // 3. 合成字段（如 figure.caption 的自动编号等）
    if let Some(synthesizable) = elem.with_mut::<dyn Synthesize>() {
        synthesizable.synthesize(engine, styles.chain(map))?;
    }

    // 4. 将样式链中的字段值"物化"到元素本身（便于 show 规则访问）
    elem.materialize(styles.chain(map));

    // 5. 创建起止 Tag（用于布局后定位）
    let tags = elem.location().map(|loc| (Tag::Start(...), Tag::End(...)));

    // 6. 标记 prepared，确保以上步骤只执行一次
    elem.mark_prepared();

    Ok(tags)
}
```

### 5.5 visit_styled：构建 StyleChain

[lib.rs#L593-L694](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs#L593-L694)

```rust
fn visit_styled(s, content, local, outer) -> Result<()> {
    // 处理 document / text / page 样式的特殊语义
    for style in local.iter() {
        let Some(elem) = style.element() else { continue };
        if elem == DocumentElem::ELEM { ... }
        else if elem == TextElem::ELEM   { /* infer locale */ }
        else if elem == PageElem::ELEM   { /* pagebreak + outside flag */ }
    }

    // ★ outside 标记：影响 liftable 判定
    // 在最外层 document 或 show rule 中通过 page set 逃出来的 = outside
    if s.outside {
        local = Cow::Owned(local.into_owned().outside());
    }

    // ★ 构建新的 StyleChain：local 在 head，outer 在 tail
    visit(s, content, outer.chain(local))?;
}
```

---

## 六、模板中 Set/Show 规则的完整生命周期

以典型模板代码为例：

```typst
#let conf(title, doc) = {
    set page(paper: "a4", columns: 2)        // 1a
    set text(font: "Libertinus Serif", 11pt)  // 1b
    show heading.where(level: 1): smallcaps   // 2a
    show heading.where(level: 1): set align(center)  // 2b
    doc                                       // 3
}

#show: doc => conf([Paper Title], doc)        // 0

= Introduction                                 // 4
Hello world.
```

### 执行步骤分解

**Step 0：Everything Show 规则立即应用**
- `#show: doc => ...` 是无选择器 show 规则
- `styled_with_recipe` 检测到 `selector.is_none()` → **立即调用 closure**
- closure 调用 `conf([Paper Title], doc)`，参数 `doc` 是剩余的全部内容

**Step 1：Conf 函数体内执行**
- 遇到 `set page(...)` → 求值为 `Styles([Property(PageElem, paper), Property(PageElem, columns)])`
- 继续往下执行，收集更多 set/show 规则
- 每个 set/show 规则都会递归作用于其后的内容（即 `doc`）
- 结果：`doc` 被多层 `StyledElem` 包裹：
  ```
  StyledElem {
    styles: Styles([Recipe(show heading level 1 → smallcaps), Recipe(show heading level 1 → set align)]),
    child: StyledElem {
      styles: Styles([Property(text, font), Property(text, size)]),
      child: StyledElem {
        styles: Styles([Property(page, paper), Property(page, columns)]),
        child: doc (= Sequence([HeadingElem(Introduction), Text(Hello world.)]))
      }
    }
  }
  ```

**Step 2：Realization 阶段递归处理**
- 从根节点开始 visit，每遇到一层 `StyledElem` 就在 StyleChain 上追加一层
- 当访问到 `HeadingElem(level: 1)` 时：
  - 当前 StyleChain 包含 page/text properties + 两个 heading recipes
  - `verdict` 遍历 recipes：
    - Recipe 1：selector `heading.where(level: 1)` 匹配 → transform Func → 选中为 step
    - Recipe 2：selector 也匹配 → Style 变换 → 收集到 map（因为 step 已有，不替换）
  - `prepare` 执行定位、合成、materialize
  - 应用 Recipe 1：`smallcaps(heading)` 得到新内容
  - `visit_styled` 用新内容 + show-set styles（center）递归
  - 新内容再次走 verdict，最终应用 Builtin heading 布局规则

**Step 3：最终产出**
- heading 被居中 + 小型大写样式
- 所有 text 继承 Libertinus Serif 11pt
- 页面是 A4 两栏

---

## 七、关键代码索引表

| 概念 | 文件 | 关键行 |
|------|------|--------|
| Set 规则求值 | [typst-eval/src/rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/rules.rs) | L11-L35 |
| Show 规则求值 | [typst-eval/src/rules.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/rules.rs) | L37-L64 |
| Recipe 结构 | [typst-library/.../styles.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs) | L445-L511 |
| StyleChain 结构 | [typst-library/.../styles.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs) | L557-L786 |
| Selector::matches | [typst-library/.../selector.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/selector.rs) | L131-L155 |
| Realization 入口 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L44-L76 |
| visit 主流程 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L243-L296 |
| verdict 核心匹配 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L438-L531 |
| visit_show_rules 递归 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L354-L434 |
| prepare 首次初始化 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L534-L591 |
| visit_styled 构建链 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L593-L694 |
| Content::styled_with_recipe | [typst-library/.../content/mod.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/content/mod.rs) | L321-L333 |
| Element::set 分派 | [typst-library/.../content/element.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/content/element.rs) | L69-L74 |
| Set trait 定义 | [typst-library/.../content/element.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/content/element.rs) | L239-L245 |
| ShowSet trait 定义 | [typst-library/.../content/element.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/content/element.rs) | L255-L263 |
| Markup 规则挂载 | [typst-eval/src/markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/markup.rs) | L26-L87 |
| Code 规则挂载 | [typst-eval/src/code.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-eval/src/code.rs) | L24-L72 |
| TEXTUAL 分组+正则 | [typst-realize/src/lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-realize/src/lib.rs) | L1019-L1374 |
| NativeRuleMap 内置规则 | [typst-library/.../styles.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/127-typst/crates/typst-library/src/foundations/styles.rs) | L985-L1085 |

---

## 八、总结：匹配不直观的原因

模板中 set/show 规则的匹配之所以不直观，根本原因在于：

1. **两阶段处理**：求值阶段只负责"把规则包在 StyledElem 外层"，真正的匹配判定延迟到 realization 阶段执行。写代码时看不到实际执行顺序。

2. **隐式 StyleChain 构建**：每进入一层内容块、每调用一个函数，都可能隐式地向 StyleChain 添加新层，而 StyleChain 是不可变链表，其优先级顺序由遍历方向决定。

3. **单次单规则 + 递归重试**：一个元素每次只应用一个 transform show rule，但应用后会完全重新走一遍流程，导致规则匹配是"链式"而非"批量"的。

4. **show-set 与 transform 的混合处理**：verdict 中 show-set 规则会全部收集，但 transform 规则只取第一个，而且两者的选择器匹配是并行的。

5. **outside/liftable 标志**：这些位决定了样式能否"逃逸"出容器或 show rule，这在页面级样式（set page, set document）中很关键，但其设置过程完全隐式。

6. **Everything Show 的 eager 应用**：无选择器的 show 规则在求值时就立即对剩余内容执行，而不是进入 StyleChain。这解释了为什么模板函数 `#show: template` 能拿到完整的 doc 参数。
