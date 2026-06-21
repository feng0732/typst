# Typst 表达式求值：从解析到求值的完整路径

本文追踪 Typst 脚本表达式从「源代码字符串」经「语法树」到「运行时值」的完整路径，重点剖析作用域（scope）和闭包（closure）的行为。

---

## 1. 总体架构：两阶段管线

```
源代码字符串
  │
  ▼  Parser (typst-syntax)
CST / AST（SyntaxNode 为骨架，ast::* 为类型化视图）
  │
  ▼  Eval trait (typst-eval)
Value（运行时值）
```

核心 crate 划分：

| Crate | 职责 |
|---|---|
| [typst-syntax](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax) | 词法分析、语法解析、CST/AST 定义 |
| [typst-eval](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval) | 表达式求值、VM、作用域管理、闭包捕获 |
| [typst-library](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library) | `Scope`、`Scopes`、`Binding`、`Closure`、`Func`、`Value` 等运行时基础类型 |

---

## 2. 解析阶段：源代码 → CST/AST

### 2.1 入口函数

三个模式对应三个入口（见 [parser.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/parser.rs#L16-L37)）：

```rust
pub fn parse(text: &str) -> SyntaxNode          // Markup 模式
pub fn parse_code(text: &str) -> SyntaxNode     // Code 模式
pub fn parse_math(text: &str) -> SyntaxNode     // Math 模式
```

它们返回一棵以 `SyntaxKind::Markup` / `SyntaxKind::Code` / `SyntaxKind::Math` 为根的 **CST**（Concrete Syntax Tree）。

### 2.2 CST 与 AST 的关系

CST 节点是 `SyntaxNode`，包含 `SyntaxKind` 和子节点列表，**保留全部源文本细节**（空格、注释、分隔符等）。

AST 是 CST 上的**惰性类型化视图**（见 [ast.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/ast.rs#L1-L78) 头部文档）：

- 每个 AST 节点本质上是一个 `&'a SyntaxNode` 指针（如 `struct Ident<'a>(&'a SyntaxNode)`）
- 通过 `AstNode::from_untyped()` 将 CST 节点转换为具体类型，转换时会检查 `SyntaxKind` 是否匹配
- 只有在**遍历时**才做转换，未访问的分支不付出构造开销

`Expr` 枚举是 AST 的核心，覆盖了所有表达式类型（见 [ast.rs#L249-L375](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-syntax/src/ast.rs#L249-L375)）：

```rust
pub enum Expr<'a> {
    Text(Text<'a>), Ident(Ident<'a>), Bool(Bool<'a>),
    CodeBlock(CodeBlock<'a>), ContentBlock(ContentBlock<'a>),
    Closure(Closure<'a>), FuncCall(FuncCall<'a>),
    LetBinding(LetBinding<'a>), ForLoop(ForLoop<'a>),
    Conditional(Conditional<'a>), ...
}
```

---

## 3. 求值阶段：AST → Value

### 3.1 `Eval` trait

定义在 [lib.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L178-L184)：

```rust
pub trait Eval {
    type Output;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output>;
}
```

为每种 AST 节点实现了 `Eval`，分布在多个文件中：

| 文件 | 覆盖的 `Eval` 实现 |
|---|---|
| [code.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs) | `Code`, `Expr`, `Ident`, 字面量, `Array`, `Dict`, `CodeBlock`, `ContentBlock`, `Parenthesized`, `FieldAccess`, `Contextual` |
| [call.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs) | `FuncCall`, `Closure`, `Args`, `CapturesVisitor` |
| [binding.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/binding.rs) | `LetBinding`, `DestructAssignment` |
| [flow.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs) | `Conditional`, `WhileLoop`, `ForLoop`, `LoopBreak`, `LoopContinue`, `FuncReturn` |
| [markup.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/markup.rs) | `Markup`, `Text`, `Space`, `Strong`, `Emph`, `Heading` 等 |
| [import.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/import.rs) | `ModuleImport`, `ModuleInclude` |
| [math.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/math.rs) | 数学模式求值 |

### 3.2 虚拟机 Vm

`Vm` 是求值的核心状态容器（见 [vm.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L16-L28)）：

```rust
pub struct Vm<'a> {
    pub engine: Engine<'a>,        // 世界访问、诊断、路由
    pub flow: Option<FlowEvent>,   // 控制流事件（break/continue/return）
    pub scopes: Scopes<'a>,        // 作用域栈
    pub inspected: Option<Span>,   // IDE 追踪的 span
    pub context: Tracked<'a, Context<'a>>,  // 上下文样式数据
}
```

关键方法：
- `Vm::define(var, value)` — 在当前作用域顶层绑定变量（见 [vm.rs#L50-L52](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L50-L52)）
- `Vm::bind(var, binding)` — 更底层的绑定，同时触发 IDE 追踪（见 [vm.rs#L58-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/vm.rs#L58-L73)）

### 3.3 求值入口

**模块级求值** — [eval()](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L40-L97)：

```
1. 检查循环求值（route.contains）
2. 构建 Engine 和 Vm（Scopes::new(Some(library))，即以标准库为 base）
3. 检查语法错误
4. root.cast::<ast::Markup>() → markup.eval(&mut vm)
5. 组装 Module（把 vm.scopes.top 作为模块导出的作用域）
```

**字符串求值** — [eval_string()](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/lib.rs#L101-L175)：

用于 `eval()` 函数等运行时求值场景，支持三种 `SyntaxMode`，还会额外 `push` 一个调用方传入的 `scope`。

---

## 4. 作用域系统：Scopes → Scope → Binding

### 4.1 三层结构

```
Scopes (作用域栈)
  ├── top: Scope (当前最内层作用域)
  ├── scopes: Vec<Scope> (外层作用域，从近到远)
  └── base: Option<&'a Library> (标准库全局作用域)
```

`Scopes` 定义在 [scope.rs](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L17-L25)。

### 4.2 作用域进入与退出

```rust
impl Scopes {
    pub fn enter(&mut self) {
        // 把当前 top 保存到栈中，创建新的空 top
        self.scopes.push(std::mem::take(&mut self.top));
    }
    pub fn exit(&mut self) {
        // 弹出之前保存的 top，丢弃当前层
        self.top = self.scopes.pop().expect("no pushed scope");
    }
}
```

**谁触发 `enter`/`exit`？**

| 场景 | 代码位置 |
|---|---|
| `CodeBlock` 求值 | [code.rs#L320-L326](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L320-L326) |
| `ContentBlock` 求值 | [code.rs#L331-L337](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L331-L337) |
| `ForLoop` 循环体 | [flow.rs#L124-L146](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs#L124-L146) |

注意 `WhileLoop` **没有** enter/exit —— 其循环变量不在新作用域中（因为 while 没有绑定变量的语法）。

### 4.3 变量查找：`Scopes::get()`

```rust
pub fn get(&self, var: &str) -> HintedStrResult<&Binding> {
    std::iter::once(&self.top)
        .chain(self.scopes.iter().rev())
        .find_map(|scope| scope.get(var))
        .or_else(|| {
            self.base.and_then(|base| match base.global.scope().get(var) {
                Some(binding) => Some(binding),
                None if var == "std" => Some(&base.std),
                None => None,
            })
        })
        .ok_or_else(|| unknown_variable(var))
}
```

查找顺序：**当前 top → 栈中外层（由近及远）→ 标准库全局作用域**。这是典型的词法作用域链查找。

数学模式有单独的 `get_in_math()`，它会查找 `base.math.scope()` 而非 `base.global.scope()`。

### 4.4 Binding：绑定值的元数据

```rust
pub struct Binding {
    value: Value,                    // 绑定的值
    kind: BindingKind,               // Normal 或 Captured(Capturer)
    span: Span,                      // 定义位置的 span
    category: Option<Category>,      // 分类
    deprecation: Option<Box<Deprecation>>,  // 弃用信息
}
```

关键区分（见 [scope.rs#L262-L268](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L262-L268)）：

- `BindingKind::Normal` — 普通绑定，**可读写**
- `BindingKind::Captured(Capturer)` — 闭包/上下文捕获的副本，**只读**

`Binding::write()` 方法在遇到 `Captured` 时会报错（见 [scope.rs#L313-L325](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-library/src/foundations/scope.rs#L313-L325)）：

```rust
pub fn write(&mut self) -> StrResult<&mut Value> {
    match self.kind {
        BindingKind::Normal => Ok(&mut self.value),
        BindingKind::Captured(capturer) => bail!(
            "variables from outside the {} are read-only and cannot be modified",
            match capturer {
                Capturer::Function => "function",
                Capturer::Context => "context expression",
            },
        ),
    }
}
```

这就是**闭包捕获的变量不可修改**的根源。

---

## 5. 闭包：从定义到调用

### 5.1 闭包定义阶段

`ast::Closure` 的 `Eval` 实现（见 [call.rs#L562-L595](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L562-L595)）：

```rust
impl Eval for ast::Closure<'_> {
    type Output = Value;
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        // 1. 求值命名参数的默认值
        let mut defaults = Vec::new();
        for param in self.params().children() {
            if let ast::Param::Named(named) = param {
                defaults.push(named.expr().eval(vm)?);
            }
        }

        // 2. 收集捕获的变量
        let captured = {
            let mut visitor = CapturesVisitor::new(Some(&vm.scopes), Capturer::Function);
            visitor.visit(self.to_untyped());
            visitor.finish()
        };

        // 3. 构建 Closure 对象
        let closure = Closure {
            node: ClosureNode::Closure(self.to_untyped().clone()),
            defaults,
            captured,
            num_pos_params: ...,
        };

        Ok(Value::Func(Func::from(closure).spanned(self.params().span())))
    }
}
```

**关键步骤解析**：

1. **默认值在定义时求值**：命名参数的默认值表达式在闭包定义时就被求值了，不是在调用时。
2. **CapturesVisitor 静态分析捕获**：遍历 AST，确定哪些自由变量需要从外部作用域捕获。
3. **闭包体（AST 节点）被保存但不在定义时求值**。

### 5.2 CapturesVisitor：闭包捕获分析

`CapturesVisitor` 是理解作用域和闭包行为的核心（见 [call.rs#L709-L882](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L709-L882)）。

它维护两组作用域：
- `external: Option<&'a Scopes<'a>>` — 引用当前 VM 的真实作用域
- `internal: Scopes<'a>` — 模拟的内部作用域（只用于跟踪哪些名字被绑定了，值无关紧要）
- `captures: Scope` — 收集到的捕获

**核心逻辑 `visit()` 方法**，对每种 AST 节点做不同处理：

| 节点类型 | 行为 |
|---|---|
| `Ident` / `MathIdent` | 尝试捕获（若不在 internal 中，则从 external 查找并捕获） |
| `CodeBlock` / `ContentBlock` | `internal.enter()`，遍历子节点，`internal.exit()` |
| `Closure` | 先访问命名参数默认值，再 `internal.enter()` 并 bind 参数名和函数名，然后访问函数体 |
| `LetBinding` | 先访问 init 表达式，再 bind 绑定名（绑定在 init 之后） |
| `ForLoop` | 先访问 iterable，再 `internal.enter()` + bind 模式变量 + 访问循环体 |
| `FieldAccess` | 只访问 target，不捕获 field 名 |
| `Named` | 只访问 expr，不捕获 name |

**捕获时做两件事**：
1. 调用 `binding.capture(capturer)` 创建一个 `BindingKind::Captured` 的副本
2. 将副本存入 `captures` Scope

这意味着**捕获发生在定义时，捕获的是值的快照**（clone），而非引用。

### 5.3 闭包调用阶段

`eval_closure()` 是闭包调用的核心（见 [call.rs#L600-L707](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/call.rs#L600-L707)）：

```rust
pub fn eval_closure(...) -> SourceResult<Value> {
    // 1. 从 AST 节点提取 name, params, body
    // 2. 构建全新的 Scopes，不继承调用方的
    let mut scopes = Scopes::new(None);  // 注意：base = None
    scopes.top = closure.captured.clone();  // 用捕获的变量作为顶层作用域

    // 3. 创建新的 Vm
    let mut vm = Vm::new(engine, context, scopes, body.span());

    // 4. 如果有函数名，绑定为递归调用
    if let Some(name) = name {
        vm.define(name, func.clone());
    }

    // 5. 绑定参数
    // 6. 求值函数体
    let output = body.eval(&mut vm)?;

    // 7. 处理 return 控制流
}
```

**关键行为**：
- **不继承调用方的任何作用域**：`Scopes::new(None)` 且 `scopes.top = closure.captured.clone()`
- 闭包只能访问**捕获的变量 + 参数 + 递归自引用**
- 这就是词法闭包的本质：函数体中的自由变量在**定义时**就已经确定了绑定

### 5.4 context 表达式的闭包

`context expr` 也创建闭包（见 [code.rs#L386-L410](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/code.rs#L386-L410)）：

```rust
impl Eval for ast::Contextual<'_> {
    fn eval(self, vm: &mut Vm) -> SourceResult<Self::Output> {
        let captured = {
            let mut visitor = CapturesVisitor::new(Some(&vm.scopes), Capturer::Context);
            visitor.visit(body.to_untyped());
            visitor.finish()
        };
        let closure = Closure {
            node: ClosureNode::Context(self.body().to_untyped().clone()),
            defaults: vec![],
            captured,
            num_pos_params: 0,
        };
        Ok(ContextElem::new(Func::from(closure)).pack())
    }
}
```

与函数闭包的区别：
- `Capturer::Context` 而非 `Capturer::Function`
- `ClosureNode::Context` 存储的是 Markup 节点而非 Closure 节点
- 无参数

---

## 6. 典型求值路径详解

### 6.1 简单变量引用

```
源码: #x
  │
  ▼ 解析
Expr::Ident(Ident)
  │
  ▼ Eval for Ident (code.rs#L156-L168)
vm.scopes.get(&self)        // 从作用域链查找
  .at(span)?                // 找不到则报错
  .read_checked(...)        // 读取值（检查弃用）
  .clone()                  // 克隆返回
```

### 6.2 let 绑定

```
源码: #let x = 1 + 2
  │
  ▼ 解析
Expr::LetBinding(LetBinding)
  │
  ▼ Eval for LetBinding (binding.rs#L9-L28)
1. eval init expr → Value::Int(3)
2. match kind:
   - Normal(pattern) → destructure(vm, pattern, value)
   - Closure(ident) → vm.define(ident, value)
3. 返回 Value::None
```

`vm.define()` 调用 `vm.scopes.top.bind(name, Binding::new(value, span))`，将变量绑定到**当前最内层作用域**。

### 6.3 代码块

```
源码: #{ let y = 1; y + 2 }
  │
  ▼ 解析
Expr::CodeBlock(CodeBlock { body: Code })
  │
  ▼ Eval for CodeBlock (code.rs#L317-L326)
1. vm.scopes.enter()   ← 进入新作用域
2. self.body().eval(vm)  → eval Code → eval_code()
   - eval "let y = 1" → 在新 top 中绑定 y
   - eval "y + 2"     → 从作用域查找 y，得到 1，计算 3
3. vm.scopes.exit()    ← 退出作用域，y 不再可见
4. 返回 Value::Int(3)
```

### 6.4 闭包定义与调用

```
源码:
#let outer = 10
#let f = (x) => x + outer
#f(5)  // 结果: 15
```

**定义 `f` 时**：

```
1. eval "10" → Value::Int(10)
2. vm.define("outer", Value::Int(10))
3. eval Closure "(x) => x + outer"
   a. 无命名参数默认值
   b. CapturesVisitor 分析:
      - "x" → internal 中有绑定（参数），不捕获
      - "outer" → internal 中没有 → 从 external 查找 → 捕获
      - captured = Scope { "outer": Binding { value: Int(10), kind: Captured(Function) } }
   c. Closure { node, defaults: [], captured, num_pos_params: 1 }
4. vm.define("f", Func::from(closure))
```

**调用 `f(5)` 时**：

```
1. eval "f" → Func (从 scopes 查找)
2. eval Args "(5)" → Args { items: [Arg { name: None, value: Int(5) }] }
3. call_func → Func::call → eval_closure
   a. scopes = Scopes::new(None)  // 不继承调用方
   b. scopes.top = closure.captured.clone()  // { "outer": Int(10) }
   c. vm.define("x", Int(5))  // 绑定参数
   d. eval "x + outer"
      - 查找 x → 当前 top → Int(5)
      - 查找 outer → 当前 top（captured）→ Int(10)
      - 5 + 10 = 15
4. 返回 Value::Int(15)
```

### 6.5 闭包中的变量遮蔽与不可修改

```
源码:
#let x = 1
#let f = () => {
  x = 2  // 错误！
}
```

调用 `f` 时：
- `x` 被捕获为 `BindingKind::Captured(Function)`
- `x = 2` 尝试 `Binding::write()` → 失败，因为 Captured 绑定是只读的

### 6.6 闭包修改可变引用（数组/字典的 in-place 方法）

```
源码:
#let arr = (1, 2, 3)
#let f = () => { arr.push(4) }
#f()
```

这里 `arr` 被捕获的是**值的 clone**，所以 `arr.push(4)` 修改的是捕获的副本，不影响外部的 `arr`。Typst 的闭包是**值捕获**（capture by value），不是引用捕获。

### 6.7 for 循环中的作用域

```
源码:
#for x in (1, 2, 3) {
  let y = x * 2
}
// y 不可见
```

求值流程（见 [flow.rs#L114-L189](file:///d:/fz/0601-2/solo-dogfeeding/code/121-typst/crates/typst-eval/src/flow.rs#L114-L189)）：

```
1. eval iterable → (1, 2, 3)
2. vm.scopes.enter()   ← 进入新作用域
3. 对每个元素:
   a. destructure(vm, pattern, value) → vm.define("x", 1)
   b. eval body → eval CodeBlock
      - vm.scopes.enter()  ← 代码块又进入一层
      - vm.define("y", 2)
      - vm.scopes.exit()
   c. 下一轮: x 被重新绑定
4. vm.scopes.exit()    ← 退出，x 和 y 都不可见
```

---

## 7. 完整路径总结图

```
┌──────────────────────────────────────────────────────────────────┐
│                        源代码字符串                               │
│                 "#let f = (x) => x + outer"                      │
└──────────────────────┬───────────────────────────────────────────┘
                       │ parse() / parse_code()
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    CST (SyntaxNode 树)                            │
│  root: SyntaxKind::Markup                                        │
│    └─ SyntaxKind::Hash                                           │
│    └─ SyntaxKind::LetBinding                                     │
│        ├─ SyntaxKind::Ident "f"                                  │
│        ├─ SyntaxKind::Eq                                         │
│        └─ SyntaxKind::Closure                                    │
│            ├─ SyntaxKind::Params "(x)"                           │
│            └─ SyntaxKind::Body "=> x + outer"                    │
└──────────────────────┬───────────────────────────────────────────┘
                       │ cast::<ast::Markup>()
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│               AST (类型化视图)                                     │
│  Markup.exprs() → [Expr::LetBinding(LetBinding)]                 │
│    .kind() = Closure(ident="f")                                  │
│    .init() = Expr::Closure(Closure)                              │
│      .params() = [Param::Pos(Pattern::Normal(Expr::Ident("x")))]│
│      .body()  = Expr::Binary(x + outer)                          │
└──────────────────────┬───────────────────────────────────────────┘
                       │ .eval(&mut vm)
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    运行时求值                                      │
│                                                                  │
│  Vm { scopes: Scopes { top, scopes, base: Library } }           │
│                                                                  │
│  1. eval LetBinding:                                             │
│     eval init → eval Closure                                     │
│       a. defaults = []                                           │
│       b. CapturesVisitor:                                        │
│          internal: [x]  (参数绑定)                                │
│          external → vm.scopes                                    │
│          "x"     → internal 有，不捕获                            │
│          "outer" → internal 无，从 external 获取并捕获             │
│          captured = Scope { "outer": Binding::Captured(...) }    │
│       c. Closure { node, captured, ... }                         │
│     vm.define("f", Func::from(closure))                          │
│                                                                  │
│  2. 调用 f(5) 时:                                                │
│     eval_closure():                                              │
│       scopes = Scopes::new(None)                                 │
│       scopes.top = closure.captured  // { "outer": ... }         │
│       vm.define("x", 5)                                          │
│       eval body: x + outer → 5 + <captured> → 返回值             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. 关键设计要点总结

1. **词法作用域**：变量查找沿作用域链从内到外，最远到标准库全局。不存在动态作用域。

2. **值捕获，非引用捕获**：闭包通过 `CapturesVisitor` 在**定义时**静态分析需要捕获的变量，然后 clone 值存入 `captured` Scope。调用时完全不继承调用方的作用域。

3. **捕获变量只读**：被捕获的变量标记为 `BindingKind::Captured`，`write()` 会失败。这是"闭包捕获变量不可修改"的实现根源。

4. **代码块/内容块自动创建作用域**：`CodeBlock.eval()` 和 `ContentBlock.eval()` 会 `enter`/`exit` 作用域，`ForLoop` 也为循环体创建新作用域。

5. **let 绑定在 init 之后生效**：`CapturesVisitor` 处理 `LetBinding` 时先访问 `init` 再 `bind` 名称，因此 `#let x = x` 中的右侧 `x` 引用的是外层的 `x`。

6. **闭包默认值在定义时求值**：命名参数的默认值在 `Closure.eval()` 中立即求值，存入 `closure.defaults`，调用时通过 `args.named::<Value>(&name)?.unwrap_or_else(|| default.clone())` 使用。

7. **递归自引用**：`eval_closure()` 中，如果闭包有名字，会把自身绑定到新 Vm 的作用域中，支持递归调用。

8. **模块求值产生独立作用域**：`eval()` 函数求值一个文件后，`vm.scopes.top` 被提取为模块的导出作用域（`Module::new(name, vm.scopes.top)`），外部只能通过 import 访问。
