# EP.10 无畏并发

> 对应正文：[guide/10-concurrency.md](../docs/guide/10-concurrency.md)（线程、move 闭包与 join / 消息传递：mpsc 通道 / 共享状态：Arc 与 Mutex / Send 与 Sync）

## 剧情回顾

- **第 1 格**：齿轮君让子线程帮忙打印 `data`，却收到 `error[E0373]: closure may outlive the current function`——闭包默认借用捕获的变量，可能比 `data` 活得久。
- **第 2 格**：码叔揭秘：`move` 把捕获变量的所有权交进线程以满足 `'static`，`join` 阻塞等待线程收尾；mpsc 通道践行"不要靠共享内存来通信，要靠通信来共享内存"。
- **第 3 格**：像素酱实证 `Arc<Mutex<T>>`：`Arc::clone` 提供跨线程共享所有权，`lock()` 拿到互斥守卫再修改；`Send`/`Sync` 把数据竞争挡在编译期。
- **第 4 格**：码叔结案：能通信就别共享；非共享不可，就把锁的粒度做到最小、顺序固定。

## 知识点拆解

### 第 1 格 · 闭包为什么活不过线程

- `thread::spawn` 要求闭包满足 `FnOnce() + Send + 'static`；
- 闭包默认按借用捕获变量，借来的引用不是 `'static`，编译器无法证明它比线程活得久，于是报 `error[E0373]`；
- 编译器的 `help` 会提示添加 `move`，把捕获变量的所有权移交给闭包。

```rust
use std::thread;

fn main() {
    let data = vec![1, 2, 3];
    let handle = thread::spawn(|| {
        println!("{data:?}");
    });
    handle.join().unwrap();
}
// error[E0373]: closure may outlive the current function
// help: to force the closure to take ownership of `data`, use the `move` keyword
```

### 第 2 格 · move、join 与 mpsc 通道

- `move` 让闭包取得捕获变量的所有权，从而满足 `'static`；`join` 返回 `Result`，子线程 panic 会在这里被观察到；不 `join` 的话主线程退出时进程直接结束，子线程可能来不及完成；
- `std::sync::mpsc` 是多生产者、单消费者通道：`Sender<T>` 可克隆，`Receiver<T>` 独占；
- `for message in rx` 持续阻塞接收，直到所有 `Sender` 被丢弃、通道关闭后自动退出。

```rust
let (tx, rx) = mpsc::channel();
let producer = thread::spawn(move || {
    for i in 1..=3 {
        tx.send(format!("消息 {i}")).unwrap();
    }
});
for message in rx {
    println!("收到: {message}");
}
producer.join().unwrap();
```

### 第 3 格 · Arc<Mutex<T>> 与 Send/Sync

- `Arc<T>` 提供跨线程的共享所有权，克隆只增加原子引用计数；`Mutex<T>` 保证同一时刻只有一个线程能访问内部值；
- `lock()` 返回 `MutexGuard`，通过 `DerefMut` 直接修改内部值，离开作用域自动解锁；持锁时 panic 会导致互斥锁中毒，后续 `lock` 返回 `Err`；
- `Send` 表示所有权可安全移动到另一线程，`Sync` 表示 `&T` 可被多线程共享；`Rc<T>` 没有 `Send`，`RefCell<T>` 没有 `Sync`。

```rust
let counter = Arc::new(Mutex::new(0));
let counter2 = Arc::clone(&counter);
thread::spawn(move || {
    let mut value = counter2.lock().unwrap();
    *value += 1;
});
```

### 第 4 格 · 锁的纪律

- 锁守卫持有时间越短越好，异步任务里尤其不能跨 `.await` 持锁；
- 多把锁必须固定获取顺序，否则两个线程各持一把互等对方就是死锁；
- 标准库 `Mutex` 非重入，同一线程重复 `lock` 同一把锁会永久阻塞。

## 坑点清单

1. 闭包忘写 `move` → `error[E0373]`，按 `help` 加 `move`；想借用栈上数据可改用 `thread::scope` 作用域线程；
2. 锁守卫持有太久（甚至跨 `.await`）→ 并发退化为串行甚至阻塞，先缩短临界区；
3. 多把锁获取顺序不一致 → AB-BA 死锁；同一线程重复 `lock` 同一把非重入锁同样死锁。

## 自测 3 题（含答案）

1. 为什么 `thread::spawn` 的闭包常常要加 `move`？——**闭包需满足 `Send + 'static`，`move` 把捕获变量的所有权移交进去，避免闭包比被借用的局部变量活得久（`error[E0373]`）**。
2. mpsc 的接收循环什么时候结束？——**所有 `Sender` 被丢弃、通道关闭之后，`for message in rx` 自动退出**。
3. `Arc<Mutex<T>>` 里两个类型各管什么？——**`Arc` 管跨线程共享所有权，`Mutex` 管同一时刻只允许一个线程访问内部值**。
