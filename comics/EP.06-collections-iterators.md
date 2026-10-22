# EP.06 集合与迭代器

> 对应正文：[guide/06-collections.md](../docs/guide/06-collections.md)（Vec：可变长序列 / String 与 &str / HashMap、BTreeMap 与 HashSet / 集合中的所有权与三种迭代）

## 剧情回顾

- **第 1 格**：齿轮君用 `v[10]` 取下标，程序直接 panic——下标访问越界即崩，连个招呼都不打。
- **第 2 格**：码叔揭秘：`v.get(i)` 返回 `Option<&T>`，越界只是 `None`；顺带盘点 Vec、String、HashMap 三剑客的分工。
- **第 3 格**：像素酱写完 `map` 却看不到输出——迭代器适配器是惰性的，没有 `collect` 就不执行；`iter`/`iter_mut`/`into_iter` 三种迭代的所有权语义也一并演示。
- **第 4 格**：码叔结案三条军规——下标赌命、`get` 保命，迭代器不见 `collect` 不动身。

## 知识点拆解

### 第 1 格 · 下标与 panic

- `v[i]` 在越界时直接 panic，只适合"下标一定合法"的场景；
- `v.get(i)` 返回 `Option<&T>`：越界得到 `None`，用 `match` 或 `if let` 处理；
- 来自用户输入、外部数据的下标都不可信，永远选 `get`。

### 第 2 格 · 三剑客与 Option

- `Vec<T>` 是有序可变长序列，`push` 追加、`insert` 插入、`pop` 弹出末尾；
- `String` 拥有堆上的 UTF-8 缓冲区，`&str` 是借用切片，函数参数优先用 `&str`；
- `HashMap<K, V>` 要求键实现 `Eq + Hash`，查找平均 O(1)，`entry` API 只查找一次；
- `HashMap` 的 `map[key]` 在键不存在时同样 panic，不确定就用 `get`。

```rust
match v.get(10) {
    Some(n) => println!("{n}"),
    None => println!("索引越界，get 返回 None"),
}
```

### 第 3 格 · 三种迭代与惰性求值

- `iter()` 产出 `&T`、`iter_mut()` 产出 `&mut T`（可原地修改）、`into_iter()` 产出 `T` 并消耗集合；
- `map`、`filter`、`take` 是惰性适配器：只搭好流水线，遇到 `collect`、`sum`、`fold` 或 `for` 才真正执行；
- 惰性带来零成本：链式调用不产生中间集合，编译器会生成与手写循环相当的紧密代码。

```rust
let doubled: Vec<i32> = numbers.iter().map(|n| n * 2).collect();
let even_sum: i32 = numbers.iter().filter(|n| *n % 2 == 0).sum();
```

### 第 4 格 · 军规

- 访问先想失败模式：下标赌不越界，`get` 把失败变成 `None`；
- 遍历期间不能修改同一集合（借用冲突 E0502），要改先收集再批量应用；
- 写完适配器链条后，自问一句：谁来消费它？

## 坑点清单

1. 下标访问越界直接 panic；不可信下标一律改用 `get`；
2. 遍历中对同一集合 `push`/`remove` → `error[E0502]` 可变借用冲突；
3. 以为 `map` 写完就执行了——适配器惰性求值，必须有 `collect`/`sum` 等消费端触发。

## 自测 3 题（含答案）

1. `v[10]` 与 `v.get(10)` 在越界时分别怎样？——**前者 panic；后者返回 `None`，调用方可以安全处理**。
2. `iter`、`iter_mut`、`into_iter` 的元素类型与集合命运分别是什么？——**依次产出 `&T`、`&mut T`、`T`；前两者借用集合、之后仍可用，后者消耗集合**。
3. 为什么 `v.iter().map(|n| n * 2);` 单独一行没有任何效果？——**`map` 是惰性适配器，缺少 `collect`/`sum`/`for` 等消费端就不会执行**。
