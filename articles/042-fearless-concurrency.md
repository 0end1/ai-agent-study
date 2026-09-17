# Rust 学习笔记（17/21）：无畏并发——编译器替你排查数据竞争

> 对应书源：《Rust 程序设计语言》第 16 章「无畏并发」

## 开篇：并发编程的「噩梦」与 Rust 的解法

多线程编程是许多开发者的噩梦。在 C/C++ 中，一个数据竞争（data race）可能潜伏数月，直到某个特定时机才 crash；在 Java 里，虽然内存模型定义了 happen-before 规则，但死锁和活锁依然防不胜防。调试这些 bug 就像在黑暗中捉迷藏——你知道它在那里，但永远不确定何时会撞上。

Rust 对这个问题给出了一个激进的答案：**把并发错误变成编译错误**。通过所有权和类型系统，Rust 在编译期就拒绝那些有数据竞争风险的代码。这不是魔法，而是把第 4 章的所有权规则、第 15 章的智能指针，一并延伸到了多线程场景。官方给这个特性起了个响亮的名字——**无畏并发**（fearless concurrency）。

## 一、线程基础：spawn、join 与 move 闭包

### 1.1 创建线程与等待结束

Rust 标准库提供 1:1 线程模型（一个语言线程对应一个 OS 线程），通过 `thread::spawn` 创建：

```rust
use std::thread;
use std::time::Duration;

let handle = thread::spawn(|| {
    for i in 1..10 {
        println!("hi number {} from the spawned thread!", i);
        thread::sleep(Duration::from_millis(1));
    }
});

// 主线程继续执行自己的代码
for i in 1..5 {
    println!("hi number {} from the main thread!", i);
}

// 阻塞等待子线程结束
handle.join().unwrap();
```

`spawn` 返回一个 `JoinHandle`，调用其 `join` 方法会阻塞当前线程直到子线程结束。**`join` 放的位置很关键**：放在主线程的 for 循环之前，子线程会先跑完；放在之后，两者交替执行。

### 1.2 move 闭包：把所有权「搬」进线程

如果闭包要捕获主线程的数据，默认借用可能不安全——Rust 不知道子线程会运行多久。此时需要 `move` 关键字强制转移所有权：

```rust
let v = vec![1, 2, 3];

let handle = thread::spawn(move || {
    println!("Here's a vector: {:?}", v);
});
```

不加 `move` 时编译器报错 `E0373`：closure may outlive borrowed value。加了 `move` 后，`v` 的所有权归子线程所有，主线程不能再使用它——**所有权规则在线程间依然生效**。

## 二、消息传递：不要共享内存，要通过通讯共享内存

Go 语言的口号「不要通过共享内存来通讯；而是通过通讯来共享内存」在 Rust 中也有完美实现——**通道**（channel）。

### 2.1 mpsc 通道的基本用法

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel(); // multiple producer, single consumer

thread::spawn(move || {
    let val = String::from("hi");
    tx.send(val).unwrap(); // send 获取所有权并转移给接收端
});

let received = rx.recv().unwrap(); // 阻塞直到收到值
println!("Got: {}", received);
```

`mpsc::channel()` 返回发送端 `tx` 和接收端 `rx`。关键点：**`send` 会拿走值的所有权**，发送后发送方不能再使用该值，否则报 `E0382` use of moved value。

### 2.2 接收端的两种方式

| 方法 | 行为 | 适用场景 |
|------|------|----------|
| `recv()` | 阻塞直到收到值，通道关闭时返回 `Err` | 主线程无事可做，专心等待 |
| `try_recv()` | 立即返回，有值则 `Ok`，无值则 `Err` | 接收线程还要做其他工作，轮询检查 |

### 2.3 多生产者：克隆发送端

`mpsc` 允许多个发送端，通过 `clone` 实现：

```rust
let tx1 = tx.clone();
thread::spawn(move || { tx1.send(...).unwrap(); });
thread::spawn(move || { tx.send(...).unwrap(); });

for received in rx { // rx 可作为迭代器，通道关闭时结束
    println!("Got: {}", received);
}
```

多个线程并发发送，接收端按到达顺序处理。输出顺序每次可能不同，这是并发的本质特性。

## 三、共享状态：Mutex<T> + Arc<T>

通道类似单所有权，而共享内存需要多所有权。Rust 的方案是 **互斥器**（mutex）。

### 3.1 Mutex<T>：获取锁才能访问

```rust
use std::sync::Mutex;

let m = Mutex::new(5);
{
    let mut num = m.lock().unwrap(); // 获取锁，阻塞直到成功
    *num = 6; // num 是 MutexGuard，解引用后可变访问内部值
} // 离开作用域自动释放锁（Drop 实现）
```

`Mutex<T>` 是一个智能指针：`lock()` 返回 `MutexGuard`，它实现了 `Deref` 指向内部数据，并在离开作用域时自动释放锁。类型系统保证**不获取锁就无法访问内部数据**。

### 3.2 多线程共享：Rc<T> 不行，Arc<T> 才行

直觉上，用 `Rc<T>` 包一层 `Mutex<T>` 就可以多线程共享了：

```rust
let counter = Rc::new(Mutex::new(0));
// ...
let counter = Rc::clone(&counter);
let handle = thread::spawn(move || { ... });
```

但编译报错：`Rc<Mutex<i32>> cannot be sent between threads safely`。原因是 `Rc<T>` 的引用计数不是线程安全的，多个线程同时增减计数会数据竞争。

解决方案是 **`Arc<T>`**（atomically reference counted），API 与 `Rc<T>` 相同，但计数操作是原子的：

```rust
use std::sync::{Mutex, Arc};

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for h in handles { h.join().unwrap(); }
println!("Result: {}", *counter.lock().unwrap()); // 10
```

| 组合 | 用途 | 线程安全 |
|------|------|----------|
| `Rc<T>` | 单线程多所有权 | ❌ 非 Send |
| `Arc<T>` | 多线程多所有权 | ✅ Send + Sync |
| `RefCell<T>` | 单线程内部可变性 | ❌ 非 Sync |
| `Mutex<T>` | 多线程内部可变性 | ✅ Sync |
| `Rc<RefCell<T>>` | 单线程可变共享 | ❌ |
| `Arc<Mutex<T>>` | 多线程可变共享 | ✅ |

### 3.3 死锁的提醒

Rust 的类型系统能防止数据竞争，但**无法阻止逻辑层面的死锁**。如果线程 A 持有锁 X 等待锁 Y，同时线程 B 持有锁 Y 等待锁 X，就会永远阻塞。规避策略与其他语言相同：统一加锁顺序、使用超时等。

## 四、Send 与 Sync：并发的「类型护照」

Rust 并发模型的精髓在于两个**标记 trait**：

- **`Send`**：类型可以安全地**转移所有权**到另一个线程。几乎所有类型都是 `Send`，除了 `Rc<T>`（引用计数非原子）和裸指针。
- **`Sync`**：类型可以安全地**被多个线程同时引用**（即 `&T` 是 `Send`）。`Mutex<T>` 是 `Sync` 的，`RefCell<T>` 和 `Cell<T>` 不是（运行时借用检查非线程安全）。

**自动推导规则**：完全由 `Send`/`Sync` 类型组成的类型，自动也是 `Send`/`Sync`。几乎不需要手动实现——手动实现需要 `unsafe` 代码。

| trait | 含义 | 反例 |
|-------|------|------|
| `Send` | 所有权可跨线程转移 | `Rc<T>`、裸指针 |
| `Sync` | 可被多线程共享引用 | `RefCell<T>`、`Cell<T>`、`Rc<T>` |

`thread::spawn` 的签名中要求闭包实现 `Send`，这就是为什么 `Rc<T>` 无法直接传给 `spawn`。编译器在类型层面「发护照」：没 `Send`「签证」的就不能过境到别的线程。

## 五、实践建议

1. **优先用消息传递，次选共享状态**：通道让数据所有权流向清晰；共享状态只在性能必要或已有架构需要时才考虑。
2. **共享状态就用 `Arc<Mutex<T>>`**：这是 Rust 多线程可变共享的标准配方，别试图用 `Rc<T>` 绕过——编译器不答应。
3. **锁的粒度尽量小**：把 `lock()` 调用和后续操作限制在最小的作用域内，减少竞争窗口。别让锁跨过 `await` 或阻塞调用。

## 六、总结

| 并发模型 | Rust 工具 | 核心保障 |
|----------|-----------|----------|
| 创建线程 | `thread::spawn` + `JoinHandle` | 所有权规则在线程间生效 |
| 跨线程传数据 | `move` 闭包 | 编译期确保无悬垂引用 |
| 消息传递 | `mpsc::channel` | `send` 转移所有权，防止用后用 |
| 共享可变状态 | `Arc<Mutex<T>>` | 类型系统确保获取锁才能访问 |
| 线程安全标记 | `Send` / `Sync` | 自动推导，禁止不安全的跨线程传递 |

Rust 的并发不是通过增加运行时复杂度来换取安全，而是**把第 4 章的所有权、第 15 章的智能指针，自然延伸到了多线程**。你写的并发代码一旦通过编译，就从根本上排除了数据竞争和悬垂引用——这正是「无畏」二字的含义。

下一篇将进入第 17 章，聊聊 Rust 的面向对象特性——trait 与面向对象设计模式的碰撞。
