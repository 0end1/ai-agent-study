# Rust 学习笔记（21/21 完结）：最后的项目——亲手撸一个多线程 Web 服务器

> 对应书源：《Rust 程序设计语言》第 20 章「最后的项目：构建多线程 Web 服务器」｜系列完结篇

## 开篇：毕业考试

这是本系列的最后一篇，也是《Rust 程序设计语言》的最后一章。前二十一篇笔记里，所有权、借用、trait、生命周期、并发、模式匹配轮番登场，但都停留在「知识点」层面——而第 20 章是毕业考试：**只靠标准库，手写一个多线程 Web 服务器**。

书里特意交代了一个细节：crates.io 上有大量生产级 Web 框架，我们这样做**不是最佳实践，而是学习**。因为 Rust 是系统编程语言，你可以选择抽象层次——自己写 HTTP server 和线程池，才能看懂将来用的 crate 背后的通用理念。就像练厨艺先不点外卖：外卖能吃饱，但只有亲手颠勺才知道火候是什么。

整个项目分三阶段演进，每一步都踩在前面的知识点上：

| 阶段 | 核心问题 | 用到的知识 |
| --- | --- | --- |
| 单线程服务器 | 听连接、读请求、写响应 | TCP/HTTP 协议、I/O、字符串 |
| 多线程化 | 慢请求阻塞一切 → 线程池 | 通道、`Arc<Mutex<T>>`、闭包、trait 对象 |
| 优雅停机 | ctrl-c 暴力退出 → 主动清理 | `Drop`、`Option::take`、枚举消息 |

## 一、单线程服务器：40 行代码看懂 HTTP

Web 服务器的两个主角协议：**TCP**（底层，描述信息如何从一台机器到另一台）和 **HTTP**（构建于 TCP 之上，定义请求和响应的内容）。两者都是请求-响应协议——客户端发起，服务端监听并回应。我们要做的，就是处理这些「原始字节数据」。

第一步只需要标准库的 `std::net`：

```rust
use std::net::TcpListener;

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    for stream in listener.incoming() {
        let stream = stream.unwrap();
        handle_connection(stream);
    }
}
```

几个易忽略的细节：`7878` 在九宫格电话键盘上打出来就是 "rust"（这是选端口的彩蛋）；`incoming` 迭代的其实是**连接尝试**而非连接本身（系统可能限制同时打开的连接数）；`handle_connection` 里的 `stream` 必须是 `mut`——`TcpStream` 内部会缓存读取的多余数据，**读操作也需要可变性**，这点很反直觉。

`handle_connection` 用 1024 字节的缓冲区读入请求后打印，就能看到浏览器发来的原始 HTTP 请求：请求行 `GET / HTTP/1.1`（方法 + URI + 版本，以 CRLF 即 `\r\n` 结尾）+ 一堆 header。手写响应同样简单——`"HTTP/1.1 200 OK\r\n\r\n"` 这个微型字符串写回流里，浏览器就不再报错；再配合 `fs::read_to_string` 把 `hello.html` 塞进 body，用 `Content-Length` 标头声明长度，一个真正的 HTML 页面就渲染出来了。

最后用 `buffer.starts_with(b"GET / HTTP/1.1\r\n")` 区分 `/` 与其他请求（后者返回 404），并做一次经典重构：把 `if/else` 收缩为只返回**元组** `("HTTP/1.1 200 OK", "hello.html")`，再用模式解构赋值，读文件和写响应的代码只保留一份。重复代码消失了，两种情况的差异一目了然。

## 二、线程池：编译器驱动的开发

单线程服务器的致命弱点用一个 `/sleep` 请求（休眠 5 秒）就能验证：先请求 `/sleep` 再请求 `/`，后者要干等 5 秒——因为服务器**串行处理连接**，慢请求把队伍全堵死了。

最朴素的修复是每个连接 `thread::spawn` 一个新线程，但这会无限制创建线程——千万级请求打过来直接耗尽资源，这就是拒绝服务（DoS）攻击。所以答案是**线程池**：固定数量的线程预先待命，新请求进入队列，空闲线程认领任务，处理完继续待命。可以并发处理 N 个请求（N 为线程数）。

有意思的是开发方式：书里先写出**假想的调用方代码**（`ThreadPool::new(4)` + `pool.execute(闭包)`），再让 `cargo check` 的编译错误一步步指导实现——作者称之为**编译器驱动开发**。因为线程池与 Web 服务器业务无关，还被拆进了独立的库 crate（`src/lib.rs`），主程序挪到 `src/bin/main.rs`。

实现中藏着三个关键决策：

1. **`execute` 的闭包 bound 抄谁？** 抄 `thread::spawn` 的签名：`F: FnOnce() + Send + 'static`——任务只执行一次所以 `FnOnce`，要跨线程转移所以 `Send`，不知道线程跑多久所以 `'static`。
2. **谁来领任务？** `ThreadPool` 不直接存线程，而是存 `Worker`（每个 Worker 包着一个 `JoinHandle`）——类比餐馆厨房的员工：先上岗，等订单来了再做菜。因为标准库只能「创建线程即给代码」，而我们需要「先创建线程，后发代码」，这层封装必须自己做。
3. **任务怎么发？** 用第 16 章的 `mpsc` 通道充当任务队列。但直接把 receiver 传给多个 Worker 会报 E0382——通道是**多生产者、单消费者**的。解法是 `Arc::new(Mutex::new(receiver))`：`Arc` 让多个 Worker 共享 receiver 的所有权，`Mutex` 保证一次只有一个 Worker 取任务。

`Job` 的定义是第 19 章类型别名的实战应用：`type Job = Box<dyn FnOnce() + Send + 'static>;`——把闭包装箱成 trait 对象，再取个短名字在通道里传递。

### 一个精妙的陷阱：`while let` vs `loop`

第 18 章学过 `while let`，很自然会想这样写 Worker：

```rust
while let Ok(job) = receiver.lock().unwrap().recv() {
    job();  // ⚠️ 锁还握着！
}
```

能编译、能运行，但**并发性悄悄失效**：`Mutex` 没有公有的 `unlock` 方法，锁的释放靠 `lock` 返回的 `MutexGuard` 被丢弃——而 `while` 条件表达式的值在整个循环体中都活着，`job()` 执行期间锁一直被持有，其他 Worker 全在干等。改成 `loop` + 循环体内的 `let job = receiver.lock().unwrap().recv().unwrap();`，`MutexGuard` 在这条 `let` 语句结束（`job()` 执行前）就被丢弃，锁立刻释放。**一个变量的生命周期细节，决定了服务器是「假并发」还是真并发**。

## 三、优雅停机：Drop、Terminate 与两段循环

ctrl-C 终止主线程时，所有 Worker 会被立刻掐断，哪怕正在处理请求——这不体面。真正的服务器要做到**优雅停机**（graceful shutdown）：处理完手头的活，再干净地退出。

**第一步：实现 `Drop`**。直觉写法是在 `drop` 里对每个 Worker 调用 `thread.join()`，但编译器报 E0507：我们手里只有 `&mut Worker`，而 `join` 要**拿走所有权**。解法正是第 17 章的模式——把字段类型改成 `Option<thread::JoinHandle<()>>`，清理时用 `take()` 把值移出来：

```rust
if let Some(thread) = worker.thread.take() {
    thread.join().unwrap();
}
```

**第二步：让线程停下来**。光 `join` 没用——Worker 还在 `loop` 里等任务，主线程会永远阻塞。于是通道不再只发任务，而是发一个枚举：`enum Message { NewJob(Job), Terminate }`。Worker 收到 `NewJob` 就干活，收到 `Terminate` 就 `break` 退出循环。

**第三步：两段循环，避免死锁**。`Drop` 里先向通道发 `size` 条 `Terminate`，再在**另一个循环**里逐个 `join`。为什么不能合成一个循环？书里的推演很精彩：假设只有两个 Worker，若边发终止消息边 join，第一个 Worker 正忙着处理请求时，`Terminate` 可能被第二个 Worker 抢走了——于是主线程永远等第一个 Worker 结束，而它永远等不到自己的终止消息。**死锁**。先全员广播、再统一收队，才保证每个线程都能收到自己的「下班通知」。

在 `main` 里用 `listener.incoming().take(2)` 只处理两个请求验证整套机制：第三个请求失败后打印 `Shutting down.`，随后是 `Terminate` 广播与逐个 worker 关闭的日志——服务器体面谢幕。

## 四、全书总结：这套项目浓缩了什么

| 项目组件 | 对应章节知识 |
| --- | --- |
| `TcpListener`/`TcpStream` 读写 | 第 12 章 I/O、第 3 章类型 |
| `if buffer.starts_with(...)` + 元组解构 | 第 18 章模式匹配 |
| `execute` 的 `FnOnce + Send + 'static` | 第 13 章闭包、第 19 章 trait bound |
| `Box<dyn FnOnce()>` 任务装箱 | 第 17 章 trait 对象、第 19 章类型别名 |
| `mpsc::channel` + `Arc<Mutex<Receiver>>` | 第 16 章并发与共享状态 |
| `Option::take` 移出所有权 | 第 17 章模式、第 6 章 `Option` |
| `impl Drop for ThreadPool` | 第 15 章智能指针与 RAII |
| 编译器驱动开发 + `assert!`/文档注释 | 第 9 章错误处理、第 14 章文档 |

**实践建议**：

1. **`while let` 陷阱要刻进肌肉记忆**：凡是用 `MutexGuard`，警惕它被 `while` 条件、临时变量拖着「多活」几行——锁的持有时间越短越好。
2. **API 先行，实现随后**：先写出期望的调用代码再动手实现，编译器的报错序列就是你的任务清单；通用逻辑（线程池）与业务（服务器）拆成独立 crate。
3. **优雅停机是生产素养**：`Drop` 清理 + 显式退出信号是通配模式，不只是 Web 服务器专用——任何「长期运行的 worker 集合」都适用。

**完结感言**：从 `cargo new` 到一个能优雅停机的多线程 Web 服务器，21 篇笔记走完了《Rust 程序设计语言》全程。Rust 的学习曲线陡在前期——所有权、借用、生命周期在第一周就能劝退——但收益同样前置：一旦编译通过，运行时的心智负担骤降。书末作者说得好：你已经准备好实现自己的项目了，也别忘了社区里乐于助人的 Rustaceans。下一次遇到问题，去翻文档、读源码、提问吧——这门语言的下一个六年，有你一份。
