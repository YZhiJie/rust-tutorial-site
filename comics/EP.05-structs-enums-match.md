# EP.05 结构体、枚举与 match

> 对应正文：[guide/05-structs-enums.md](../docs/guide/05-structs-enums.md)（三种结构体 / impl：方法与关联函数 / 枚举携带数据 / match、守卫与绑定）

## 剧情回顾

- **第 1 格**：齿轮君 `println!("{user:?}")` 报 `error[E0277]`——`User` 没有实现 `Debug`。
- **第 2 格**：码叔一行 `#[derive(Debug, Clone)]` 派生搞定打印，并区分 `impl` 块里的 `&self` 方法与 `Self::new()` 关联函数。
- **第 3 格**：像素酱见识枚举变体携带数据（`Message::Move { x: i32 }`），以及 match 必须穷尽——漏分支报 E0004。
- **第 4 格**：码叔结案：穷尽检查交给编译器，该 match 就 match。

## 知识点拆解

### 第 1 格 · {:?} 与 Debug

- `{:?}` 需要类型实现 `Debug` trait；自定义类型默认没有实现 → `error[E0277]`；
- 编译器会提示为类型添加 `#[derive(Debug)]`；
- 派生要求所有字段都实现对应 trait，有一个字段没实现就派生失败。

### 第 2 格 · derive 与 impl

```rust
#[derive(Debug, Clone)]
struct User { name: String }

impl User {
    fn new(name: String) -> Self {
        Self { name }          // 关联函数：User::new(...)
    }
    fn greet(&self) {}         // 方法：user.greet()
}
```

- `impl` 块中第一个参数为 `self` / `&self` / `&mut self` 的是方法，用 `value.method()` 调用；没有 `self` 的是关联函数，用 `Type::function()` 调用；`Self` 指当前类型；
- 选 `&self` 还是 `&mut self` 直接决定调用方是否需要可变绑定：对不可变绑定调用 `&mut self` 方法 → `error[E0596]`；
- `derive(Debug, Clone, PartialEq)` 分别支持 `{:?}` 输出、`clone()` 与 `==` 比较。

### 第 3 格 · 枚举、Option 与穷尽

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}
```

- 枚举变体可以是单元形态、结构体形态或元组形态，但都属于同一个类型；任一时刻只处于一个变体，读取内部数据必须通过模式匹配；
- `Option<T>` 只有 `Some(T)` 与 `None` 两个变体，Rust 没有空指针，"缺失"语义一律用 `Option`；
- `match` 必须覆盖所有可能情况：漏掉分支（哪怕是 `None`）→ `error[E0004]: non-exhaustive patterns`，可补 `None` 或 `_` 兜底。

### 第 4 格 · match 家族怎么选

- 只关心一个分支用 `if let`；模式持续匹配用 `while let`；"匹配失败就提前退出"用 `let else`（其 `else` 分支必须发散：`return`、`break` 或 `panic!`）；
- 但 `if let` 不做穷尽检查——需要编译器帮你数清全部情形时，应当用 `match`；
- 守卫（guard）用 `if` 追加条件；绑定用 `value @ 1..=9` 同时匹配范围并绑定值。

## 坑点清单

> 🔗 正文对照：[《结构体、枚举与模式匹配 · 常见错误》](../docs/guide/05-structs-enums.md)

1. 用 `{:?}` 打印自定义类型却忘 `derive(Debug)` → `error[E0277]`；
2. `match` 分支不穷尽 → `error[E0004]`，即使写了 `Some` 也必须补 `None` 或 `_`；
3. 处处用 `if let` 图省事，丧失编译器的穷尽检查；新增变体时没有任何报错提醒你回来补分支。

## 自测 3 题（含答案）

1. 为什么 `println!("{user:?}")` 报 `error[E0277]`？——**`{:?}` 需要 `Debug` trait，自定义类型默认未实现；加 `#[derive(Debug)]` 即可**。
2. 方法和关联函数怎么区分？——**`impl` 中第一个参数是 `self` / `&self` / `&mut self` 的是方法（`value.method()`）；没有 `self` 的是关联函数（`Type::function()`）**。
3. `match` 漏写 `None` 分支会怎样？——**`error[E0004]: non-exhaustive patterns`，必须补 `None` 或 `_` 让分支穷尽**。
