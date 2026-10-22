# EP.14 性能优化方法论

> 对应正文：[guide/14-performance.md](../docs/guide/14-performance.md)（release 与 dev 的差异 / 先测量，再优化 / 分配与克隆 / 基准测试与剖析）

## 剧情回顾

- **第 1 格**：像素酱嫌程序慢，拍脑袋就要加缓存，还在 debug 模式下用 `Instant` 计时——被码叔拦下：没测量过的优化都是瞎忙。
- **第 2 格**：码叔立下工作流：测量、剖析、优化、验证四步闭环；`cargo build --release` 才开优化，criterion 做基准，flamegraph 找热点。
- **第 3 格**：齿轮君对号入座"浪费三连"——无脑 `clone`、循环里 `format!` 反复分配、`Vec` 不预分配；好在迭代器链在 release 下是零成本抽象。
- **第 4 格**：码叔结案——没有测量的优化是瞎忙，先 profile、再动手、后验证。

## 知识点拆解

### 第 1 格 · "感觉慢"不是结论

- 性能问题常藏在与直觉相反的位置：一次多余的 `clone`、循环里反复扩容的 `Vec`、被调试构建放大的迭代器链，都可能是真凶；
- 凭直觉改写后的代码也可能没有变快，却损失了可读性；
- 正确顺序永远是：先测量、再优化、后验证。

### 第 2 格 · 结论只认 release

dev 与 release 是两种生物：

| 配置 | `opt-level` | 调试断言 | 整数溢出检查 |
| --- | --- | --- | --- |
| `dev`（默认） | 0 | 开启 | 开启 |
| `release` | 3 | 关闭 | 关闭 |

- `cargo run` 是 dev 构建，用它比较写法，测到的主要是"未优化代码的执行成本"；
- `Instant::now()` 只能粗略计时；可信结论来自 release 构建、多轮采样与 criterion：

```toml
[dev-dependencies]
criterion = "0.5"

[[bench]]
name = "sum"
harness = false
```

- 基准回答"哪个更快"，剖析回答"时间花在哪里"：Linux 用 `perf`，跨平台可 `cargo flamegraph`；微基准里用 `std::hint::black_box` 防止编译器把未使用的计算整个删掉。

### 第 3 格 · 分配与克隆是真实开销

```rust
let mut v = Vec::with_capacity(items.len()); // 预分配，避免反复扩容
let total: u64 = values.iter().map(|v| v * 2).filter(|v| *v > 10).sum();
```

- `clone()` = 堆分配 + 逐元素复制；只有需要独立所有权时才克隆，函数参数优先借用 `&str`、`&[T]`；
- 热路径避免 `to_string()`、`format!` 反复分配；可在循环外建 `String`，每轮 `clear()` 复用；
- 迭代器链在 release 下通常被内联成与手写循环等价的机器码——零成本抽象；但 `collect()` 会创建新集合，是真实开销。

### 第 4 格 · 用工程化固化结论

```toml
[profile.release]
lto = "thin"
codegen-units = 1
panic = "abort"
```

- `lto`、`codegen-units`、`panic` 可进一步调优产物，但并非越多越好，应结合基准与体积逐项验证；
- CI 至少跑 `cargo fmt --check`、`cargo clippy -D warnings` 与 `cargo test`，正确性永远排在性能前面。

## 坑点清单

1. 在 debug 模式下测性能下结论——dev 与 release 优化级别悬殊，结论可能完全相反；
2. 用 `clone` 消编译错误而不是理清所有权——错误没了，分配与复制开销留下了；
3. 不懂 `lto`/`codegen-units` 就乱调 profile——用数倍编译时间换无法测量的收益。

## 自测 3 题（含答案）

1. 为什么性能结论只能来自 release 构建？——**dev 的 `opt-level = 0`，不做内联与向量化，还保留调试断言与溢出检查，测到的是未优化成本而非算法差异**。
2. criterion 基准里为什么要用 `black_box`？——**防止编译器发现结果未被使用而把整段计算优化掉**。
3. 函数参数需要字符串时，为什么优先 `&str` 而非 `String`？——**借用可以避免调用方为传参而克隆，且 `&str` 能同时接受 `&String` 与字符串字面量**。
