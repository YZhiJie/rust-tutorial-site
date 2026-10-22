# EP.07 trait 与泛型

> 对应正文：[guide/07-traits-generics.md](../docs/guide/07-traits-generics.md)（trait 定义与实现 / 泛型函数与泛型结构体 / impl Trait / trait 对象与动态分发）

## 剧情回顾

- **第 1 格**：像素酱在 `fn show<T>(x: T)` 里调用 `x.name()`，收获 `error[E0277]`——泛型参数 `T` 没有任何能力，编译器不认识这个方法。
- **第 2 格**：码叔揭秘：`T: Bound` 用 trait bound 声明能力；单态化在编译期为每个具体类型生成专用副本，静态分发零运行时开销。
- **第 3 格**：齿轮君实证：`Box<dyn Draw>` 把圆和方块装进同一个 `Vec`——异构集合只能靠 trait 对象，动态分发走 vtable。
- **第 4 格**：码叔结案——编译期能定的交给泛型，运行时才知的用 `dyn`。

## 知识点拆解

### 第 1 格 · E0277 与 trait bound

- `T` 只是占位符，编译器对它一无所知，调用任何方法都会被拒；
- 用 `T: TraitName`（或 `where T: ...`）声明能力后，对应方法才可用；
- 同一个泛型类型可以按 bound 做"条件实现"：只有满足约束的 `T` 才拥有某些方法。

### 第 2 格 · 单态化

```rust
fn largest<T: PartialOrd + Copy>(items: &[T]) -> T {
    let mut max = items[0];
    for &item in items {
        if item > max { max = item; }
    }
    max
}
```

- 编译器在调用点把 `T` 替换成具体类型，每个类型一份专用代码；
- 静态分发没有运行时类型检查开销，代价是二进制体积可能增大；
- `impl Trait` 写在参数位置等价于泛型加 bound；写在返回位置隐藏具体类型，但只能返回同一种类型。

### 第 3 格 · 静态 vs 动态分发

```rust
let boxes: Vec<Box<dyn Shape>> = vec![
    Box::new(Circle { radius: 1.0 }),
    Box::new(Square { side: 2.0 }),
];
```

- `&dyn Trait` 借用已有值，`Box<dyn Trait>` 拥有堆上的值；
- 方法调用经虚表（vtable）间接跳转一次，换来同一集合容纳不同具体类型；
- 对象安全（object safety）决定 trait 能否做成 `dyn`：方法不带自身泛型参数、不返回 `Self`、不要求 `Self: Sized`，违反则报 E0038。

### 第 4 格 · 选择标准

- 类型在编译期可知、追求极致性能 → 泛型与静态分发；
- 需要异构集合、插件式扩展 → trait 对象与动态分发；
- 泛型方法可以加 `where Self: Sized`，使其对 trait 对象不可用，从而保住对象安全。

## 坑点清单

1. 泛型里调方法却忘写 trait bound → `error[E0277]`；
2. 返回位置 `impl Trait` 在不同分支返回不同类型——它仍是静态分发，只能是一种类型，要换 `Box<dyn Trait>`；
3. 带泛型方法或返回 `Self` 的 trait 不是对象安全的，无法做成 `dyn Trait`（E0038）。

## 自测 3 题（含答案）

1. 什么是单态化？——**编译期把泛型参数替换为每个具体类型、各生成一份专用代码的过程，运行时没有分发开销**。
2. `&dyn Draw` 与 `Box<dyn Draw>` 的区别？——**前者借用已有值，后者拥有堆上的值；两者都经虚表动态分发**。
3. 为什么 `Vec<Box<dyn Shape>>` 能混放圆和方块，而 `Vec<T>` 不行？——**trait 对象擦除了具体类型，元素统一为 `Box<dyn Shape>`；泛型集合的每个实例只能是单一具体类型**。
