---
layout: post
title: 关于rust中shadowing的一些实现原理 
date: 2025-11-05 03:24:00 +0800
categories: [学习思考]
---
## shadowing in rust

# 何为 shadowing

Shadowing（变量遮蔽）在 Rust 中指的是：在同一或内层作用域里用 `let` 再次声明同名标识符，从而引入一个全新的“变量绑定”，让外层（或先前）同名绑定在当前作用域中“不可见”。被遮蔽（shadowed）的旧绑定并不会立刻被 drop；它会按照它自己的作用域规则在适当的时机被 drop（通常是所在作用域结束，或编译器证明不再使用后提前结束其存储期）。

示例：

```rust
let x = String::from("hi"); // x1
let x = x.len();            // x2，遮蔽 x1；这里发生 move: String -> usize
// 到这里为止，x1 在名字上不可访问；其值已被移动给 x2
println!("{}", x);          // 使用的是 x2
```

对比“重新赋值”（`x = ...`）：
- 重新赋值要求 `x` 可变（`mut`），且不能改变类型。
- shadowing 不需要 `mut`，允许改变类型、可变性、甚至发生所有权 move，因为它创建的是“新绑定”。

---

# 编译器对 shadowing 的实现（名称解析与作用域）

从宏观到微观，编译期大致是这样工作的：

- 解析得到 AST/HIR 后，名称解析器维护“作用域栈”，每一层称为一个 Rib（肋骨）。
- 每个 `let` 绑定把名字插入当前栈顶 Rib 的 `bindings`。
- 查找名字时，从“栈顶向外”逐层查找，先命中者为准，于是新绑定遮蔽旧绑定。
- 一些作用域层（RibKind）会作为“屏障”，限制继续向外查找或进行额外校验。
- 到 MIR 阶段，每个 `let` 都对应不同的本地槽位（如 `_1`、`_2`），这进一步说明 shadowing 是“新变量”，不是就地改写。

## 1. 作用域栈与 Rib 结构

在 rustc 的名称解析（`rustc_resolve`）里，单层作用域用 `Rib` 表示。简化理解如下：

```rust
// 作用域层（简化示意）
#[derive(Debug)]
pub(crate) struct Rib<'ra, R = Res> {
    pub bindings: FxIndexMap<Ident, R>, // 同名插入即实现 shadowing：新绑定会“盖住”旧绑定
    pub patterns_with_skipped_bindings: UnordMap<DefId, Vec<(Span, Result<(), ErrorGuaranteed>)>>,
    pub kind: RibKind<'ra>,             // 这一层是什么作用域（Block、Item、Fn、Module…）
}

// 编译器中的简化示意
pub struct Resolver<'a> {
    // 实际上按命名空间分多套栈（ValueNS/TypeNS/MacroNS）
    ribs: Vec<Rib<'a>>,  // 作用域栈，从内到外
}
```

进入一个新的语法范围（如块 `{}`、函数体、某些宏/常量上下文等），就会 `push` 一个新的 `Rib`；离开则 `pop`。遇到 `let`（含各种模式绑定）时，将新名字写入当前栈顶 `Rib.bindings`。

## 2. 名称解析（从内向外查找）

当编译器遇到一个标识符时，核心的思路就是“从当前作用域自内向外找，遇到屏障停”。大致的实现：

```rust
impl<'a> Resolver<'a> {
    fn resolve_name_in_ribs(&self, ident: Ident) -> Option<&R> {
        // 从内层作用域向外查找
        for rib in self.ribs.iter().rev() {
            if let Some(binding) = rib.bindings.get(&ident) {
                // 命中最近的绑定，完成遮蔽
                return Some(binding);
            }
            // 某些作用域层可能限制继续向外查找
            if rib.kind.stops_searches() {
                break;
            }
        }
        None
    }
}
```

真实编译器还会：
- 按命名空间分别维护作用域栈（值/类型/宏/label/生命周期各有一套），宏卫生（SyntaxContext）匹配，模块/前奏（prelude）解析等；
- 命中后做一轮“跨 rib 校验”（例如禁止从某些上下文捕获外层局部）；
- 若局部未命中，才会走模块路径解析。

## 3. 遮蔽的发生与 drop 时机

- 何时发生遮蔽：当第二次 `let` 同名出现时，新绑定立即生效；后续对该名字的解析将命中新绑定。
- 旧绑定何时 drop：
  - 旧绑定的值若被 move（如示例），则不会再 drop（因所有权已转移）。
  - 若未 move，旧绑定通常在其作用域末尾 drop；编译器也可能将“存储生存期”提早到“最后使用点”之后（在 MIR 中体现为 `StorageDead` 的插入，有时恰好就位于新的 `let` 之前）。
- 同一作用域内的 shadowing 与“进入内层块”的 shadowing：
  - 同一作用域再次 `let`：旧绑定在名字上“立即不可见”，其真正 drop 点取决于是否 move 以及编译器的生存期收缩分析。
  - 内层块里 `let` 同名：外层绑定在内层块中被遮蔽，但其作用域不变；内层块结束后，名字又“看见”外层绑定；外层绑定在外层作用域结束时（或更早的“最后使用点”后）drop。

---

# 与“重新赋值”的区别

- 重新赋值（`x = expr;`）：要求 `x` 为 `mut`，且不能改变类型；对所有权与借用有既定约束。
- shadowing（`let x = expr;` 覆盖前一个 `x`）：
  - 不需要 `mut`；
  - 可以改变类型与可变性；
  - 可以发生 move，让所有权从旧绑定转移到新绑定；
  - 因为是“新绑定”，调试时看到的是两个不同的变量，只是同名。

示例：

```rust
let s = String::from("hi"); // &str -> String
let s = s.as_str();         // 现在 s 的类型变成 &str
```

---

# 常见用法与注意事项

- 常见用法
  - 链式转换时复用变量名，逐步“收窄/转化”类型（如 `String -> &str -> usize`）；
  - 控制可见性：在内层作用域里引入同名临时绑定，内层结束后回到外层绑定；
  - 从拥有所有权的值派生只读视图或计算结果并继续使用同名标识符。

- 注意事项
  - 过度使用会降低可读性，尤其是长函数中频繁改变含义/类型；
  - 与借用/所有权交互时要留意 move/borrow 规则，避免混淆；
  - 调试时同名但不同绑定不易分辨，可适当改名或缩小作用域。

---
# 小结

- shadowing 的本质是“新绑定遮蔽旧绑定”：每个 `let` 创建独立绑定，名称解析从“最近作用域”优先命中。
- 旧绑定不会因被遮蔽而立刻 drop；它的析构时机由作用域与生存期分析决定（未 move 通常在作用域末尾，编译器可在不再使用后提前结束存储期）。
- 与重新赋值不同，shadowing 允许改变类型与可变性，并可发生所有权移动，因而在表达“阶段性转换”时非常实用，但需权衡可读性。

---

# 参考
    rust源码：https://github.com/rust-lang/rust
