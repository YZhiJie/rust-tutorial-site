# EP.11 异步与 await

> 对应正文：[guide/11-async.md](../docs/guide/11-async.md)（async/await 与 Future / tokio 运行时 / tokio::spawn 与独立任务 / 阻塞边界）

## 剧情回顾

- **第 1 格**：像素酱调用 `work()` 后一行输出都没有——`async fn` 调用后函数体根本不执行，只返回一个惰性的 `Future`。
- **第 2 格**：码叔揭秘：编译器把 `async` 代码转成状态机，每个 `.await` 是可能暂停的恢复点；`#[tokio::main]` 启动运行时来轮询驱动。
- **第 3 格**：齿轮君实测 `tokio::join!`：50/80/30ms 三个请求并发等待，总耗时约等于最慢的 80ms；顺序 `.await` 则是 160ms；`tokio::spawn` 要求 `Send + 'static`。
- **第 4 格**：码叔结案：不阻塞工作线程、不借用栈上局部变量、不让 std 锁守卫跨 `.await`。

## 知识点拆解

### 第 1 格 · Future 是惰性的

- `async fn` 调用后不会立即执行函数体，而是返回一个实现了 `Future` 的值；
- `Future` 只有被轮询（poll）时才向前推进；光创建不等待，会得到 `unused_must_use` 警告；
- 修复方式：`work().await`，或交给 `tokio::spawn` 变成独立任务。

```rust
async fn work() {
    println!("执行");
}

#[tokio::main]
async fn main() {
    work();        // 仅创建 Future，不会执行
    work().await;  // 这才真正执行
}
```

### 第 2 格 · 状态机与 tokio 运行时

- 编译器把 `async fn` 转换成状态机，每个 `.await` 是一个可能暂停的恢复点；
- `.await` 挂起当前任务，把控制权交还执行器，等 `Future` 就绪后再恢复；
- `#[tokio::main]` 宏把 `main` 包装成启动运行时的同步入口，并在其中阻塞等待异步主体完成；手写 `Future` 要处理 `Pin`、`Context` 与 `Waker`，日常几乎不需要。

### 第 3 格 · join! 与 spawn

- `tokio::join!` 同时等待多个 `Future`，全部完成后返回元组；它不创建新线程，在同一任务内并发推进，总耗时接近最慢的一个分支；
- 改成依次 `.await`，总耗时就是各分支之和（160ms）；
- `tokio::spawn` 把 `Future` 提交为独立任务，由运行时调度、可能在不同线程执行，因此要求 `Send + 'static`，不能借用栈上局部变量；`JoinHandle` 的 `.await` 返回 `Result`，任务 panic 会被包装成错误。

```rust
let (a, b, c) = tokio::join!(
    fetch("请求 A", 50),
    fetch("请求 B", 80),
    fetch("请求 C", 30)
); // 总耗时约 80 ms，而不是 160 ms
```

### 第 4 格 · 阻塞边界

- 异步运行时只有少量工作线程，`std::thread::sleep`、同步文件 I/O、长时间计算会占住整个 worker，同一线程上的其他任务全部停摆；
- 用 `tokio::time::sleep` 替代阻塞睡眠；无法改写的同步库调用放入 `tokio::task::spawn_blocking`，由专门的阻塞线程池执行；
- std 的 `Mutex` 守卫不要跨 `.await`：换 `tokio::sync` 里的锁，或缩短临界区让守卫在 `.await` 前释放。

## 坑点清单

1. 异步任务里做 CPU 密集或阻塞调用 → 占住工作线程拖垮其他任务，改用 `tokio::time::sleep` 或 `spawn_blocking`；
2. `tokio::spawn` 的闭包借用栈上局部变量 → `borrowed data escapes outside of closure`，加 `move` 或改用拥有所有权的类型；
3. std `Mutex` 守卫跨 `.await` → 可能死锁或占住 worker，用 tokio 的锁或缩短临界区。

## 自测 3 题（含答案）

1. 为什么调用了 `async fn` 却没任何输出？——**`async fn` 返回惰性的 `Future`，必须 `.await` 或交给执行器（如 `tokio::spawn`）驱动才会执行**。
2. `join!` 与顺序 `.await` 的耗时差别是什么？——**`join!` 在同一任务内并发推进，总耗时约等于最慢分支；顺序 `.await` 是各分支耗时之和**。
3. 异步代码里必须调用同步阻塞库时怎么办？——**放进 `tokio::task::spawn_blocking`，由运行时分配的专门阻塞线程池执行**。
