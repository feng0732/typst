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

代码位置：[crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs)

---

## 二、规则注册阶段（解析 → 求值）

### 2.1 Set 规则的求值

Set 规则的求值入口在 [crates/typst-eval/src/rules.rs#L11-L35](crates/typst-eval/src/rules.rs#L11-L35)：

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

Show 规则的求值同样在 [crates/typst-eval/src/rules.rs#L37-L64](crates/typst-eval/src/rules.rs#L37-L64)：

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

**Recipe 结构**（[crates/typst-library/src/foundations/styles.rs#L447-L461](crates/typst-library/src/foundations/styles.rs#L447-L461)）：
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

规则求值后，如何作用于后续内容？关键在 [crates/typst-eval/src/markup.rs#L26-L87](crates/typst-eval/src/markup.rs#L26-L87) 和 [crates/typst-eval/src/code.rs#L24-L72](crates/typst-eval/src/code.rs#L24-L72) 的递归求值：

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

**`styled_with_recipe` 的特殊处理**（[crates/typst-library/src/foundations/content/mod.rs#L321-L333](crates/typst-library/src/foundations/content/mod.rs#L321-L333)）：
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

[crates/typst-library/src/foundations/styles.rs#L557-L770](crates/typst-library/src/foundations/styles.rs#L557-L770)

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

[crates/typst-library/src/foundations/styles.rs#L825-L846](crates/typst-library/src/foundations/styles.rs#L825-L846)
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

[crates/typst-library/src/foundations/styles.rs#L652-L664](crates/typst-library/src/foundations/styles.rs#L652-L664)

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

[crates/typst-library/src/foundations/selector.rs#L74-L104](crates/typst-library/src/foundations/selector.rs#L74-L104)

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

[crates/typst-library/src/foundations/selector.rs#L508-L544](crates/typst-library/src/foundations/selector.rs#L508-L544)

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

[crates/typst-library/src/foundations/selector.rs#L131-L155](crates/typst-library/src/foundations/selector.rs#L131-L155)

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

[crates/typst-realize/src/lib.rs#L1019-L1043](crates/typst-realize/src/lib.rs#L1019-L1043)

流程：
1. `TEXTUAL` 分组规则收集连续的 `TextElem / LinebreakElem / SmartQuoteElem`（中间可穿插 `SpaceElem`）
2. 在 `finish_textual` 中调用 `find_regex_match_in_elems`，将分组内的文本合并成字符串
3. 对 StyleChain 中所有正则 Recipe 尝试匹配，取最左匹配
4. 对匹配文本切割，生成新的 TextElem，再递归走 show 流程

---

## 五、Realization 阶段的递归处理

Realization 是整个规则系统的核心执行阶段。入口在 [crates/typst-realize/src/lib.rs#L44-L76](crates/typst-realize/src/lib.rs#L44-L76)。

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

[crates/typst-realize/src/lib.rs#L438-L531](crates/typst-realize/src/lib.rs#L438-L531)

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

[crates/typst-realize/src/lib.rs#L354-L434](crates/typst-realize/src/lib.rs#L354-L434)

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

[crates/typst-realize/src/lib.rs#L534-L591](crates/typst-realize/src/lib.rs#L534-L591)

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

[crates/typst-realize/src/lib.rs#L593-L694](crates/typst-realize/src/lib.rs#L593-L694)

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
| Set 规则求值 | [crates/typst-eval/src/rules.rs](crates/typst-eval/src/rules.rs) | L11-L35 |
| Show 规则求值 | [crates/typst-eval/src/rules.rs](crates/typst-eval/src/rules.rs) | L37-L64 |
| Recipe 结构 | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L445-L511 |
| StyleChain 结构 | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L557-L786 |
| Selector::matches | [crates/typst-library/src/foundations/selector.rs](crates/typst-library/src/foundations/selector.rs) | L131-L155 |
| Realization 入口 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L44-L76 |
| visit 主流程 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L243-L296 |
| verdict 核心匹配 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L438-L531 |
| visit_show_rules 递归 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L354-L434 |
| prepare 首次初始化 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L534-L591 |
| visit_styled 构建链 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L593-L694 |
| Content::styled_with_recipe | [crates/typst-library/src/foundations/content/mod.rs](crates/typst-library/src/foundations/content/mod.rs) | L321-L333 |
| Element::set 分派 | [crates/typst-library/src/foundations/content/element.rs](crates/typst-library/src/foundations/content/element.rs) | L69-L74 |
| Set trait 定义 | [crates/typst-library/src/foundations/content/element.rs](crates/typst-library/src/foundations/content/element.rs) | L239-L245 |
| ShowSet trait 定义 | [crates/typst-library/src/foundations/content/element.rs](crates/typst-library/src/foundations/content/element.rs) | L255-L263 |
| Markup 规则挂载 | [crates/typst-eval/src/markup.rs](crates/typst-eval/src/markup.rs) | L26-L87 |
| Code 规则挂载 | [crates/typst-eval/src/code.rs](crates/typst-eval/src/code.rs) | L24-L72 |
| TEXTUAL 分组+正则 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L1019-L1374 |
| NativeRuleMap 内置规则 | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L985-L1085 |

---

## 八、总结：匹配不直观的原因

模板中 set/show 规则的匹配之所以不直观，根本原因在于：

1. **两阶段处理**：求值阶段只负责"把规则包在 StyledElem 外层"，真正的匹配判定延迟到 realization 阶段执行。写代码时看不到实际执行顺序。

2. **隐式 StyleChain 构建**：每进入一层内容块、每调用一个函数，都可能隐式地向 StyleChain 添加新层，而 StyleChain 是不可变链表，其优先级顺序由遍历方向决定。

3. **单次单规则 + 递归重试**：一个元素每次只应用一个 transform show rule，但应用后会完全重新走一遍流程，导致规则匹配是"链式"而非"批量"的。

4. **show-set 与 transform 的混合处理**：verdict 中 show-set 规则会全部收集，但 transform 规则只取第一个，而且两者的选择器匹配是并行的。

5. **outside/liftable 标志**：这些位决定了样式能否"逃逸"出容器或 show rule，这在页面级样式（set page, set document）中很关键，但其设置过程完全隐式。

6. **Everything Show 的 eager 应用**：无选择器的 show 规则在求值时就立即对剩余内容执行，而不是进入 StyleChain。这解释了为什么模板函数 `#show: template` 能拿到完整的 doc 参数。

---

## 九、模板匹配边界：容器与 RealizationKind

### 9.1 RealizationKind 枚举

Realization 有 5 种模式，决定了当前处理上下文是否被视为"容器内部"：

[crates/typst-library/src/routines.rs#L153-L169](crates/typst-library/src/routines.rs#L153-L169)

```rust
pub enum RealizationKind<'a> {
    Bundle,                      // 顶层 bundle 导出 → 非容器
    Document { info },            // 顶层文档 → 非容器，outside = true
    Fragment { kind },           // 嵌套容器（block、html.div 等）→ outside = false
    Par,                        // 段落内部 realization → outside = false
    Math,                      // 数学公式内部 → outside = false
}
```

**outside 标志的初始化**（[crates/typst-realize/src/lib.rs#L66](crates/typst-realize/src/lib.rs#L66)：
```rust
outside: matches!(kind, RealizationKind::Document { .. }),
```

只有 `Document` 模式初始化时 outside=true，其他模式（包括 Bundle 虽然在顶层）outside 初始为 false。

### 9.2 容器边界判定：set document / set page 的禁区

在 `visit_styled` 中对 document 和 page 样式做严格的边界检查（[crates/typst-realize/src/lib.rs#L605-L656](crates/typst-realize/src/lib.rs#L605-L656)：

| RealizationKind | set document | set page |
|-----------------|--------------|----------|
| `Document{..}` | ✅ 允许，填充 info | ✅ 允许（Paged）/ ⚠️ HTML 告警 / ✅ Bundle |
| `Bundle` | ✅ 允许（Bundle 专用）| ✅ 允许 |
| `Fragment{..}` | ❌ "document set rules are not allowed inside of containers" | ❌ "page configuration is not allowed inside of containers" |
| `Par` | ❌ 同上 | ❌ 同上 |
| `Math` | ❌ 同上 | ❌ 同上 |

**set page 在 Document + Paged Target 的特殊行为**（[crates/typst-realize/src/lib.rs#L636-L641](crates/typst-realize/src/lib.rs#L636-L641)：
```rust
Target::Paged => {
    // 当遇到 page styles 时，我们从 show rule 笼子里"破笼而出"
    pagebreak = true;
    s.outside = true;  // ★ 标记当前及后续样式为 outside，可被提升
}
```

这意味着：当你在 **show rule 内部**（`s.outside` 本应为 false）写 `set page(...)`，会触发"越狱"：
1. 设置 `s.outside = true`
2. 产生一个弱分页符 PagebreakElem
3. 之后的样式都获得 `outside()` 标记，可被提升到页面级

### 9.3 liftable + outside = 可提升样式

一个样式（Property/Recipe）要能被提升到页面级，**必须同时满足**：
```
style.liftable()  ← 源自 set 规则（非直接构造）
    AND
style.outside()   ← 源自 Document 顶层或被 set page 越狱后标记
```

两个标志独立设置，通过 `Styles::outside()` 遍历所有样式设置 outside=true（[crates/typst-library/src/foundations/styles.rs#L92-L102](crates/typst-library/src/foundations/styles.rs#L92-L102)。

**outside 标志在 show rule 中的继承**（[crates/typst-realize/src/lib.rs#L419-L420](crates/typst-realize/src/lib.rs#L419-L420)：
```rust
let prev_outside = s.outside;
s.outside &= content.is::<ContextElem>();  // 进入 show rule 时 outside 清零
// ... 执行 show rule ...
s.outside = prev_outside;  // 退出 show rule 时恢复
```

进入任何 show rule 时，`s.outside` 被 **AND** 上 `content.is::<ContextElem>()`。
- 普通元素：`false AND false = false` → show rule 内部无法产生可提升样式
- ContextElem：`false AND true = false`（初始 Document 时是 true AND true = true）

这是 show rule 的"笼子"机制：默认情况下，show rule 内部的 set rule 无法逃逸到页面级——除非你在里面写 `set page` 触发越狱。

---

## 十、Target 系统与 show page 告警

### 10.1 Target 枚举与 TargetElem

[crates/typst-library/src/foundations/target.rs#L65-L91](crates/typst-library/src/foundations/target.rs#L65-L91)

```rust
pub enum Target {
    #[default] Paged,    // PDF/PNG/SVG 等分页布局
    Html,                // HTML 导出（连续流，不分页）
    Bundle,              // Bundle 多文件导出
}
```

`TargetElem` 是一个**永远不被用户构造**的"幽灵元素"，只用于在 StyleChain 中承载 `target` 字段：

```rust
#[elem]
pub struct TargetElem {
    pub target: Target,  // 仅作样式链属性容器
}
```

### 10.2 Target 属性注入入口

[crates/typst/src/lib.rs#L111-L113](crates/typst/src/lib.rs#L111-L113)

```rust
let base = StyleChain::new(&library.styles);
// ★ 在根 StyleChain 上注入目标属性
let target = TargetElem::target.set(T::target()).wrap();
let styles = base.chain(&target);
```

这是编译入口处的注入：PDF 导出时 `Target=Paged`，HTML 导出时 `Target=Html`，Bundle 时 `Target=Bundle`。

### 10.3 两种 Page 告警的本质区别

与 page 相关的告警有**两种完全独立**的机制，发生在不同阶段：

#### 告警 A：show page 规则求值告警（发生在 Eval 阶段，与 Target 无关）

代码位置：[crates/typst-eval/src/rules.rs#L66-L77](crates/typst-eval/src/rules.rs#L66-L77)

```rust
/// Warns that `show page` rules currently have no effect.
fn check_show_page_rule(vm: &mut Vm, recipe: &Recipe) {
    if let Some(Selector::Elem(elem, _)) = recipe.selector()
        && *elem == Element::of::<PageElem>()
    {
        vm.engine.sink.warn(warning!(
            recipe.span(),
            "`show page` is not supported and has no effect";
            hint: "customize pages with `set page(..)` instead";
        ));
    }
}
```

调用时机：Show 规则求值时（[crates/typst-eval/src/rules.rs#L59](crates/typst-eval/src/rules.rs#L59)）
```rust
let recipe = Recipe::new(selector, transform, self.span());
check_show_page_rule(vm, &recipe);        // ← 告警 A
check_show_par_set_block(vm, &recipe);    // ← spacing 迁移提示
```

**触发条件**：
- 任何 Target（Paged / Html / Bundle 都一样）
- show 规则的选择器精确匹配 `PageElem`
- 即：用户写了 `#show page: ...`

**告警内容**："`show page` is not supported and has no effect"
**提示**：用 `set page(..)` 代替

**根本原因**：PageElem 是一个"容器元信息元素"，不是通过 show 规则布局的——它的参数在页面级 realization 中直接被读取，没有内置 show rule 可被用户覆盖。写了 `show page: func` 也永远不会匹配到任何元素。

---

#### 告警 B：HTML 下 set page 告警（发生在 Realization 阶段，Target=Html 专属）

代码位置：[crates/typst-realize/src/lib.rs#L642-L646](crates/typst-realize/src/lib.rs#L642-L646)

```rust
Target::Html => {
    s.engine.sink.warn(warning!(
        style.span(),
        "page set rule was ignored during HTML export"
    ));
}
```

**触发条件**：
- 当前在 `Document{..}` realization 中
- `outer.get(TargetElem::target)` 返回 `Target::Html`
- 当前 Styles 中检测到 `PageElem` 属性（即用户写了 `set page(...)`）

**告警内容**："page set rule was ignored during HTML export"

**根本原因**：HTML 导出是连续流式布局，没有"页面"概念（没有分页、没有纸张大小），所以 set page(paper, margin, columns 等) 完全没有意义，只能告警后丢弃。

---

**两种告警对比表**：

| 维度 | 告警 A：show page 求值告警 | 告警 B：HTML set page 告警 |
|------|--------------------------|--------------------------|
| 代码位置 | typst-eval/src/rules.rs#L66-L77 | typst-realize/src/lib.rs#L642-L646 |
| 触发阶段 | 求值（Eval）阶段 | 实现（Realization）阶段 |
| 触发代码 | `check_show_page_rule(vm, recipe)` | `visit_styled` 中遍历 Styles 时 |
| 用户写法 | `#show page: ...` | `#set page(paper: ...)` + HTML 导出 |
| Target 依赖 | 与 Target 无关，任何导出目标都告警 | 仅 Target=Html 时告警 |
| 提示内容 | 改用 `set page(..)` | 无迁移建议（直接忽略） |

### 10.4 Builtin Show Rule 与 Target 的关系

每个元素的内置 show rule 按 Target 分别注册在 `NativeRuleMap` 中（[crates/typst-library/src/foundations/styles.rs#L985-L1085](crates/typst-library/src/foundations/styles.rs#L985-L1085)）：

在 verdict 中查找 builtin rule 时：
```rust
let target = styles.get(TargetElem::target);
if let Some(rule) = engine.library.rules.get(target, elem) {
    step = Some(ShowStep::Builtin(rule));
}
```

这意味着同一个元素在 Paged 和 Html 目标下可以有**完全不同的内置 show 实现**。

---

## 十一、show par 与 set block 的 spacing 迁移边界

### 11.1 核心事实：ParElem 没有内置 Show Rule

这是最关键的代码事实，必须作为分析起点。

在布局规则注册表 [crates/typst-layout/src/rules.rs#L39-L112](crates/typst-layout/src/rules.rs#L39-L112) 中，`register` 函数注册了所有内置 show rule：
```rust
pub fn register(rules: &mut NativeRuleMap) {
    use Target::Paged;
    // Model.
    rules.register(Paged, STRONG_RULE);
    rules.register(Paged, EMPH_RULE);
    rules.register(Paged, LIST_RULE);
    rules.register(Paged, ENUM_RULE);
    rules.register(Paged, TERMS_RULE);
    rules.register(Paged, HEADING_RULE);
    rules.register(Paged, FIGURE_RULE);
    rules.register(Paged, TABLE_RULE);
    // ...还有很多（共约40+规则）
    // 但是——没有 ParElem！
}
```

**ParElem 不在注册列表中**。它既不是 Strong/Emph 那种简单的 Content 转换，也不是 Heading/Figure/Block 那种需要复杂布局的元素。

**ParElem 的真实布局链路**（完全绕过 show rule 系统）：

1. **Realization 阶段**：PAR 分组规则收集行内元素，`finish_par` 构造 `ParElem::new(body)`（[crates/typst-realize/src/lib.rs#L1203](crates/typst-realize/src/lib.rs#L1203)）
2. `visit` → `prepare` → `verdict`：如果有 `show par` 规则，verdict 会匹配，但...
3. **verdict 查找 builtin rule**：`engine.library.rules.get(target, ParElem::ELEM)` → 返回 `None`（因为没注册）
4. `visit_show_rules`：发现 `step.is_none()`，直接返回 content（不进入 show rule 应用循环）
5. **Flow Layout 阶段**：流式收集器直接识别 ParElem，调用 `layout_par`（[crates/typst-layout/src/flow/collect.rs#L83-L84](crates/typst-layout/src/flow/collect.rs#L83-L84)）
6. 调用 `crate::inline::layout_par` → 行内布局 → 生成行 frames → 推送到 flow 的 Child 列表

**重要结论**：ParElem 不经过 Builtin Show Rule 这一层。它直接进入流式布局，由 `layout_par` 和 `layout_inline` 函数处理。

### 11.2 架构变更背景：Par 不再是 Block

迁移提示的核心代码在 [crates/typst-eval/src/rules.rs#L79-L95](crates/typst-eval/src/rules.rs#L79-L95)：

```rust
/// Migration hint for `show par: set block(spacing: ..)`.
fn check_show_par_set_block(vm: &mut Vm, recipe: &Recipe) {
    if let Some(Selector::Elem(elem, _)) = recipe.selector()
        && *elem == Element::of::<ParElem>()
        && let Transformation::Style(styles) = recipe.transform()
        && let styles = StyleChain::new(styles)
        && (styles.has(BlockElem::above) || styles.has(BlockElem::below))
    {
        vm.engine.sink.warn(warning!(
            recipe.span(),
            "`show par: set block(spacing: ..)` has no effect anymore";
            hint: "write `set par(spacing: ..)` instead";
            hint: "this is specific to paragraphs as they are not considered blocks \
                   anymore";
        ));
    }
}
```

**核心设计决策（提示中明确说明）**：**"paragraphs are not considered blocks anymore"**

这意味着 Typst 经历了一个架构变更：
- **旧版本**：Paragraph 被当作 Block 的一种，通过 BlockElem 的 `above` / `below` 属性控制段间距
- **新版本**：Paragraph 是独立的元素类型，拥有自己的 `spacing` 属性，不再读取 BlockElem 的 `above`/`below`

### 11.3 ParElem 的 spacing vs BlockElem 的 above/below

两种间距属性定义在不同元素上，用途完全不同：

**ParElem.spacing**（[crates/typst-library/src/model/par.rs#L213-L225](crates/typst-library/src/model/par.rs#L213-L225)）：
```rust
/// The spacing between paragraphs.
#[default(Em::new(1.2).into())]
pub spacing: Length,
```

**BlockElem.above / BlockElem.below**（[crates/typst-library/src/layout/container.rs#L318-L354](crates/typst-library/src/layout/container.rs#L318-L354)）：
```rust
/// The spacing between this block and its predecessor.
pub above: Smart<Spacing>,

/// The spacing between this block and its successor.
pub below: Smart<Spacing>,
```

**关键关系**：
- 相邻 Par ↔ Par：使用 `par.spacing`（两边值取大=折叠）
- 相邻 Par ↔ Block：**Block 的 above/below 优先**（Par 文档中明确写了）
- 相邻 Block ↔ Block：`block.below` 和 `block.above` 取大

### 11.4 代码事实：show par: set block(...) 的所有 Block 属性都无效

这是与之前分析最大的不同点。现在有明确的代码证据：

**Block 属性的消费位置**：BlockElem 的所有属性（width、fill、stroke、inset、above、below、radius、clip 等）只在 `layout_single_block` 和 `layout_multi_block` 中被读取。

证据：[crates/typst-layout/src/flow/block.rs#L27-L28](crates/typst-layout/src/flow/block.rs#L27-L28)
```rust
// 只在 BlockElem 布局时读取：
let width = elem.width.get(styles);     // BlockElem.width
let height = elem.height.get(styles);   // BlockElem.height
let inset = elem.inset.resolve(styles); // BlockElem.inset
// ... fill、stroke、outset、radius、clip 等也是如此
```

**ParElem 的布局不读取任何 BlockElem 属性**：

在 [crates/typst-layout/src/flow/collect.rs#L159-L186](crates/typst-layout/src/flow/collect.rs#L159-L186) 的 `par()` 方法中：
```rust
fn par(&mut self, elem: &'a Packed<ParElem>, styles: StyleChain<'a>) -> SourceResult<()> {
    let lines = crate::inline::layout_par(
        elem,
        self.engine,
        self.locator.next(&elem.span()),
        styles,
        self.base,      // ← region Size 是 flow 的基宽度（列宽），不是 BlockElem.width
        self.expand,
        self.par_situation,
    )?;
    // ...
}
```

传入的 region 是 `self.base`，即 flow 的列宽度，**完全不读取 BlockElem.width**。

在行内布局链路 [crates/typst-layout/src/inline/mod.rs](crates/typst-layout/src/inline/mod.rs) 中：
- `configuration` 函数读取 `ParElem::justify`、`ParElem::linebreaks`、`ParElem::first_line_indent`、`ParElem::hanging_indent`、`TextElem::size`、`TextElem::dir`、`AlignElem::alignment` 等
- **没有任何地方读取 `BlockElem::*`**

**结论**：`show par: set block(...)` 中注入的 Block 属性全部无效，因为 ParElem 的布局代码不读取它们。这些属性只在 BlockElem 自身布局时才会被消费。

### 11.5 show par 搭配 set block 的属性有效性完整表

基于代码事实的完整分析：

| Block 属性 | 在 show par: set block(...) 中是否生效 | 原因 | 正确写法 |
|------------|--------------------------------------|------|----------|
| `above` / `below` / `spacing` | ❌ **无效**，有迁移告警 | Par 不再是 Block，不读 BlockElem.above/below | `set par(spacing: 2em)` |
| `width` | ❌ **无效**，无告警 | ParElem 布局用 flow 列宽，不读 BlockElem.width | `show par: it => block(width: 80%, it)` |
| `height` | ❌ **无效**，无告警 | 同上 | 同上，显式包 Block |
| `fill` / `stroke` / `radius` | ❌ **无效**，无告警 | 需要 BlockElem 包装器来消费这些属性 | 同上，显式包 Block |
| `inset` / `outset` | ❌ **无效**，无告警 | 同上 | 同上，显式包 Block |
| `clip` / `breakable` / `sticky` | ❌ **无效**，无告警 | 同上 | 同上，显式包 Block |
| `align`（BlockElem.align） | ❌ **无效**，无告警 | Par 布局读 AlignElem.alignment，不读 BlockElem.align | `set align(center)` 或 `#show par: set align(center)` |

**唯一能让 Block 属性生效的方式**：通过 transform show rule 显式把 ParElem 包装进 BlockElem：
```typst
// ✅ 正确：显式包 Block，Block 属性由 BlockElem 布局消费
#show par: it => block(
    width: 80%,
    fill: red,
    stroke: 1pt,
    inset: 1em,
    above: 1em,
    below: 1em,
    it
)

// ❌ 错误：show-set 注入的 Block 属性无人消费
#show par: set block(width: 80%, fill: red)
```

### 11.6 为什么 show par: set block(width) 看起来"好像"有效

在某些情况下用户可能误以为 `show par: set block(width: 80%)` 有效，实际上是其他机制在起作用：

1. **AlignedElem 或外层 Block 的 width**：如果外层有 `#set block(width: 80%)`，那是外层 Block 的 width，不是 show par 注入的
2. **Align 属性被误读**：`AlignElem::alignment` 是独立属性，通过 `styles.get(AlignElem::alignment)` 读取，容易和 BlockElem.align 混淆
3. **show rule 的 transform 形式**：如果用户实际写的是 `#show par: it => block(width: 80%, it)`（transform 形式），那是显式包装，当然有效

### 11.7 Par 分组与流式收集的完整链路

为了完整理解，Par 从文本到布局的完整链路：

```
文本输入（"Hello" + "World" + 空行）
    ↓ （PAR 分组规则）
[crates/typst-realize/src/lib.rs#L1045-L1073]
    ↓
finish_par 构造 ParElem::new(body)
[crates/typst-realize/src/lib.rs#L1203]
    ↓
visit(ParElem) → prepare → verdict
    ↓ （没有 builtin rule，step = None）
layout_flow → collect → run_block()
    ↓ （匹配 ParElem）
self.par(elem, styles)
[crates/typst-layout/src/flow/collect.rs#L83-L84]
    ↓
layout_par → layout_inline_impl → 行布局 → 生成 frames
    ↓
Child::Line 被推入 flow 的 output
    ↓
compose → distribute → 页面排版
```

在整个链路中，**没有任何环节会读取 BlockElem 的属性**。

---

## 十二、撤销规则（Revocation）与 Guard 机制

### 12.1 RecipeIndex 的计算方式

[crates/typst-library/src/foundations/styles.rs#L526-L528](crates/typst-library/src/foundations/styles.rs#L526-L528)

```rust
// "从链顶开始"计数的 Recipe 编号
pub struct RecipeIndex(pub usize);
```

在 verdict 中计算（[crates/typst-realize/src/lib.rs#L491-L493](crates/typst-realize/src/lib.rs#L491-L493)）：

```
StyleChain（从顶到底遍历 = 从内层到外层）
 r=0 (最先遇到的Recipe)
 r=1
 r=2
 ...
 r=N-1 (最后遇到的Recipe)
 depth = recipes().count() = N (总Recipe数)

 index = RecipeIndex(*depth - r)

 内层Recipe（优先级高）: r=0 → index = N   (最大)
 ...
 外层Recipe（优先级低）: r=N-1 → index = 1   (最小)
```

**RecipeIndex 越大 → 越靠近元素 → 优先级越高**

### 12.2 Guard 机制（普通 Show Rule）

普通 show rule 的防重复应用机制：

```rust
// 应用 Recipe 时：
output.into_owned().guarded(guard)  // 在 Content 上打 guard 标记

// 下次 verdict 时：
if elem.is_guarded(index) { continue; }  // 已 guard → 跳过此Recipe
```

Guard 标记存储在 `Content.guards: SmallBitSet` 上（[crates/typst-library/src/foundations/content/mod.rs](crates/typst-library/src/foundations/content/mod.rs)）。

特点：
- 每个 Recipe 只对**它自己产生的输出**打 guard
- 其他 Recipe 产出的新元素没有这个 guard，可以被正常匹配
- 这就是链式匹配（strong→emph→box）能工作的关键：每一步只 guard 自己

### 12.3 Revocation 机制（正则 Show Rule）

**为什么正则需要 Revocation 而不是 Guard？**

因为正则匹配是在 TEXTUAL 分组阶段进行的，操作的是**文本字符串**不是元素**，guard 打在元素上，但分组里跨元素操作，需要在 StyleChain 级别撤销。

[crates/typst-realize/src/lib.rs#L1465-L1470](crates/typst-realize/src/lib.rs#L1465-L1470)

```rust
let output = recipe.apply(s.engine, context.track(), matched_text)?;
// ★ 向 StyleChain 注入一个 Revocation 样式
let revocation = Style::Revocation(id).into();
let chained = outer.chain(s.arenas.styles.alloc(revocation));
visit(s, s.store(output), chained)?;  // 用带 Revocation 的链递归
```

在 `find_regex_match_in_str` 中检测（[crates/typst-realize/src/lib.rs#L1321-L1374](crates/typst-realize/src/lib.rs#L1321-L1374)）：

```rust
let mut revoked = SmallBitSet::new();
for entry in styles.entries() {
    let recipe = match &**entry {
        Style::Recipe(recipe) => recipe,
        Style::Property(_) => continue,
        Style::Revocation(index) => {
            revoked.insert(index.0);  // ★ 收集所有被撤销的 index
            continue;
        }
    };
    r += 1;
    // ... 找到匹配 regex ...
    let index = RecipeIndex(*depth - (r - 1));
    if revoked.contains(index.0) { continue; }  // ★ 被撤销 → 跳过
}
```

**Guard vs Revocation 对比**：

| 维度 | Guard | Revocation |
|------|-------|------------|
| 存储位置 | Content.guards 字段 | StyleChain 中的 Style::Revocation(_) 样式 |
| 作用对象 | 单个 Content 实例 | 整个 StyleChain 作用域内所有后续匹配 |
| 适用场景 | 普通 Elem/Label 选择器 Show Rule | 正则 Regex 选择器 Show Rule |
| 触发时机 | verdict 检测 elem.is_guarded(index) | 分组中遍历 entries 时检测 revoked.contains(index) |
| 撤销范围 | 只影响打过标记的单个元素 | 影响所有后续子树内的正则匹配 |

---

## 十三、跨文件规则边界：import 与 include

### 13.1 import：只导命名空间，不带规则

[crates/typst-eval/src/import.rs#L15-L181](crates/typst-eval/src/import.rs#L15-L181)

`#import "lib.typ"` 的行为：
1. 加载文件 → `import_file()` → 调用 `eval()` 完整求值被导入文件
2. 被导入文件的顶级代码全部执行完，包括其中的 set/show 规则作用于**被导入文件内部的内容
3. **只返回 Module（值绑定关系）**
4. 调用方 vm.scopes 中绑定 Module 的名字/导入项

**关键**：被导入文件里的顶级 set/show 规则**不会逃逸到导入方**：

```typst
// lib.typ:
#set text(red)
#let f = 42

// main.typ:
#import "lib.typ"  // text 不会变红！
Hello                    // 颜色还是默认的
```

原因：`set text(red)` 在 lib.typ 的求值中，作用于 lib.typ 内部剩余内容（如果有）。它被包在 lib.typ 内部 StyledElem 里。lib.typ 的 Module.value 不会携带这些 StyledElem。

### 13.2 include：插入内容，规则跟内容走

[crates/typst-eval/src/import.rs#L184-L209](crates/typst-eval/src/import.rs#L184-L209)

```rust
impl Eval for ast::ModuleInclude<'_> {
    type Output = Content;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let module = import(...)?;
        Ok(module.content())  // ★ 返回 module 的整个文档内容（含 StyledElem 树）
    }
}
```

`#include "chapter.typ"` 的行为：
1. 同样 import_file → eval 求值得 Module
2. 但取的是 `module.content()`，即被包含文件的**完整文档内容树**
3. 这棵 Content 树里包含了所有顶级 set/show 规则产生的 StyledElem 包装

**关键区别**：
- import：只拿 Module 的符号表（scope
- include：拿 Module 的 content()（Content 树，含所有 StyledElem）

所以被 include 文件中的顶级 `set page(...)` 会作为 StyledElem 形式**原封不动地插入到包含位置，**跟随插入点上下文生效。

### 13.3 跨文件模板的典型写法

正确的模板模式（常见做法）：

```typst
// template.typ:
#let conf(body) = {
    set page(paper: "a4")
    set text(font: "Linux Libertine")
    body  // ← body 被包在上面两个 set 的 StyledElem 内层
}

// main.typ:
#import "template.typ"
#show: doc => conf(doc)  // 导入函数，调用函数体内 set 对 doc 生效
```

这里的原理：`conf(doc)` 是用户函数调用。`set` 发生在**main.typ 的求值上下文中（不是 template.typ），所以产生的 StyledElem 在 main 的 Content 树中。

---

## 十四、边界条件汇总表

| 场景 | 结果 | 核心原因 |
|------|------|----------|
| `#show: template`（无选择器） | 立即对剩余内容调用 template(doc) | styled_with_recipe 中 selector.is_none() 时 eager apply |
| `#show heading: ...` 在 block() 内部 | ✅ 对 block 内 heading 仍然生效 | Recipe 在 StyleChain 中向下传递 |
| `#set page(...)` 在 show rule 内部 | ❌ 默认被"笼子"机制屏蔽，除非有另一个 `set page` 越狱 | s.outside &= content.is::<ContextElem>() 进入 show rule 时清零 |
| `#set page(...)` + HTML 导出 | ⚠️ 告警 B，规则被忽略 | visit_styled 中 Target::Html 时 emit warning |
| `#show page: ...`（任何导出目标） | ⚠️ 告警 A，规则永久无效 | check_show_page_rule() 在 Eval 阶段就告警 |
| `#set document(...)` 在 block() 内部 | ❌ 编译错误：not allowed inside of containers | RealizationKind != Document/Bundle |
| `#show par: set block(above: 2em)` | ❌ 迁移告警 + 实际无效 | Par 不再是 Block，不读 BlockElem.above/below |
| `#show par: set block(width: 80%)` + 外层 `#set block(width: 60%)` | ❌ 两者都无效，无告警 | ParElem 布局用 flow 列宽，不读 BlockElem.width |
| `#show par: set block(fill: red)` | ❌ 无效（无警告）| ParElem 不经过 BlockElem 布局，fill 等属性无人消费 |
| `#show heading` + import 中定义的规则在容器内 heading | ✅ 规则生效，但只有匹配 heading 时注入 | 容器内 StyleChain 仍包含外层 Recipe |
| `#show` 正则匹配后，递归内容中同样正则 | ✅ 通过 Revocation 机制跳过已应用规则 | Style::Revocation(index) 注入 StyleChain |
| `#show heading: box` 后，heading 产出 box 内 heading | ❌ 不会重复应用（guard 阻止） | Content.guards 打了该 RecipeIndex 的 guard |
| `#include "a.typ"`，a.typ 内顶级 set | ✅ set 生效在插入位置之后 | include 返回 content()，带 StyledElem 树 |
| `#import "a.typ"`，a.typ 内顶级 set | ❌ set 在 a.typ 内部，不影响导入方 | import 只绑定 Module 符号表不返回 Content 树 |
| 同一元素多个 show 选择器匹配 | 只应用优先级最高的一个 transform，show-set 全部收集 | verdict 中 step 是 Option<ShowStep>（单值） |

---

## 十五、新增关键代码索引（补充七章）

| 概念 | 文件 | 关键行 |
|------|------|--------|
| RealizationKind 定义 | [crates/typst-library/src/routines.rs](crates/typst-library/src/routines.rs) | L153-L179 |
| outside 初始化 + 越狱 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L66, L636-L641 |
| show rule 笼子继承 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L419-L420, L658-L662 |
| document/page 容器边界检查 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L605-L656 |
| Target 与 TargetElem | [crates/typst-library/src/foundations/target.rs](crates/typst-library/src/foundations/target.rs) | L65-L137 |
| Target 属性注入入口 | [crates/typst/src/lib.rs](crates/typst/src/lib.rs) | L111-L113 |
| 告警 A：show page 求值告警 | [crates/typst-eval/src/rules.rs](crates/typst-eval/src/rules.rs) | L59, L66-L77 |
| 告警 B：HTML 下 set page 告警 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L642-L646 |
| spacing 迁移提示（show par + set block）| [crates/typst-eval/src/rules.rs](crates/typst-eval/src/rules.rs) | L60, L79-L95 |
| ParElem 无内置 show rule（register 无注册） | [crates/typst-layout/src/rules.rs](crates/typst-layout/src/rules.rs) | L39-L112 |
| ParElem.spacing 属性定义 | [crates/typst-library/src/model/par.rs](crates/typst-library/src/model/par.rs) | L98-L99, L213-L225 |
| BlockElem 属性消费（layout_single_block） | [crates/typst-layout/src/flow/block.rs](crates/typst-layout/src/flow/block.rs) | L27-L28, L76-L95 |
| Flow 中 ParElem 直接调用 layout_par | [crates/typst-layout/src/flow/collect.rs](crates/typst-layout/src/flow/collect.rs) | L83-L84, L159-L186 |
| ParElem 行内布局不读 Block 属性 | [crates/typst-layout/src/inline/mod.rs](crates/typst-layout/src/inline/mod.rs) | L180-L242 |
| PAR 分组规则定义 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L1045-L1073 |
| finish_par 构造 ParElem | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L1190-L1205 |
| RecipeIndex 定义 | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L526-L528 |
| RecipeIndex 计算 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L491-L493 |
| Revocation 定义 | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L221-L226 |
| Revocation 注入 | [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L1465-L1470 |
| Revocation 检测（正则匹配）| [crates/typst-realize/src/lib.rs](crates/typst-realize/src/lib.rs) | L1321-L1374 |
| import vs ModuleImport | [crates/typst-eval/src/import.rs](crates/typst-eval/src/import.rs) | L15-L181 |
| include vs ModuleInclude | [crates/typst-eval/src/import.rs](crates/typst-eval/src/import.rs) | L184-L209 |
| Styles::outside() | [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L92-L102 |
| Style.liftable()/outside()| [crates/typst-library/src/foundations/styles.rs](crates/typst-library/src/foundations/styles.rs) | L268-L286 |
| check_show_par_set_block 触发点 | [crates/typst-eval/src/rules.rs](crates/typst-eval/src/rules.rs) | L60, L79-L95 |
