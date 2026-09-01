# Rust 学习笔记（2/21）：猜数字游戏——动手认识 Rust 的第一座山

> 本系列基于官方《Rust 程序设计语言》（The Rust Programming Language，TRPL）逐章学习。

## 开篇：把抽象概念"跑"一遍

只看语法书永远学不会游泳。猜数字游戏就是 Rust 的"浅水区"：程序随机生成一个 1-100 的秘密数字，玩家输入猜测，程序提示"大了/小了"，猜对则恭喜退出。这个看起来简单的程序，几乎把 Rust 入门阶段最重要的概念全部串了起来：`let` 变量、`match` 表达式、方法调用、关联函数、`use` 引入作用域、外部 crate 依赖……本章的用意是让你**先看到全貌**，细节留给后面各章。

## 第一步：读取用户输入——认识变量与 Result

项目初始化沿用 Cargo：

```bash
$ cargo new guessing_game
$ cd guessing_game
```

处理输入的核心代码：

```rust
use std::io;

fn main() {
    println!("Guess the number!");
    println!("Please input your guess.");

    let mut guess = String::new();

    io::stdin()
        .read_line(&mut guess)
        .expect("Failed to read line");

    println!("You guessed: {}", guess);
}
```

一行行拆开看，信息量很大：

- **`use std::io;`**：标准库里的类型默认不会全部进作用域，只有少量基础项（叫 *prelude*）。需要输入输出，就得显式引入 `io` 库；
- **`let mut guess = String::new();`**：Rust 变量**默认不可变**，`mut` 才允许修改；`String::new()` 创建空字符串，其中 `::new` 是 `String` 的**关联函数**（隶属于类型的函数，而非实例），`new` 是创建实例的惯用名；
- **`io::stdin().read_line(&mut guess)`**：`&` 是**引用**——让多段代码访问同一份数据而不复制内存。引用默认也不可变，所以必须写 `&mut guess`（第 4 章所有权会彻底讲透）；
- **`.expect("Failed to read line")`**：`read_line` 返回 `Result` 枚举（成员为 `Ok`/`Err`）。`expect` 遇到 `Err` 就崩溃并打印信息，遇到 `Ok` 则取出里面的值。不处理 `Result` 的话，Rust 编译时会给出 `unused Result` 警告——错误处理被语言层面"逼"着面对；
- **`println!("You guessed: {}", guess)`**：`{}` 是占位符，像"小蟹钳"一样夹住后面的值；多个值就写多对 `{}`。

## 第二步：生成秘密数字——认识外部 crate

Rust 标准库**没有**随机数功能，于是第一次接触 Rust 生态的"外挂"方式——crate（代码包）。在 `Cargo.toml` 的 `[dependencies]` 下加一行：

```toml
rand = "0.8.3"
```

`0.8.3` 是 `^0.8.3` 的简写，表示"任何 ≥0.8.3 且 <0.9.0 的版本"，即语义化版本兼容区间。然后：

```rust
use rand::Rng;

let secret_number = rand::thread_rng().gen_range(1..101);
```

- `use rand::Rng;` 引入 trait（第 10 章详解），随机数方法都由它定义；
- `thread_rng()` 从操作系统取种子生成随机数生成器；
- `gen_range(1..101)` 生成 1-100（`start..end` 含头不含尾，等价写法 `1..=100`）。

Cargo 会自动下载 `rand` 及其依赖并编译。两个关键机制值得记住：

| 机制 | 作用 |
|---|---|
| `Cargo.lock` | 首次构建时锁定所有依赖的确切版本，**保证任何人在任何时候构建出相同结果**（可重现构建）；只有 `cargo update` 或改 `Cargo.toml` 才会升级 |
| `cargo update` | 在兼容区间内更新到最新版（如 0.8.3→0.8.4），要跨大版本则需手改 `Cargo.toml` |

## 第三步：比较大小——认识 match 与类型错误

```rust
use std::cmp::Ordering;

match guess.cmp(&secret_number) {
    Ordering::Less => println!("Too small!"),
    Ordering::Greater => println!("Too big!"),
    Ordering::Equal => println!("You win!"),
}
```

`Ordering` 是成员为 `Less`/`Greater`/`Equal` 的枚举；`cmp` 比较两个值返回其中一个成员；`match` 是 Rust 的条件利器，**按分支逐一检查模式**，命中就执行对应代码——没有 `if/else if` 链条的冗长感，且编译器能检查你是否覆盖了所有情况。

但这段代码**编译不过**——这是本书第一次故意展示错误：

```
error[E0308]: mismatched types
match guess.cmp(&secret_number) {
       ^^^^^^^^^^^^^^^^^^^^^^ expected struct `String`, found integer
```

Rust 是静态强类型语言：`guess` 是 `String`，`secret_number` 是数字（默认 `i32`），两者不能比较。解决之道是类型转换 + 变量遮蔽：

```rust
let guess: u32 = guess.trim().parse().expect("Please type a number!");
```

- **`trim()`**：去掉 `read_line` 附带的换行符（输入 `5` 回车后字符串实际是 `5\n`）；
- **`parse()`**：把字符串解析成数字，`u32` 是无符号 32 位整数；注释 `let guess: u32` 显式指定类型；
- **变量遮蔽（shadowing）**：允许复用 `guess` 这个名字，用新值"盖住"旧值——比发明 `guess_str`、`guess_num` 两个名字干净得多，常用于类型转换。

## 第四步：循环与处理无效输入

`loop` 创建无限循环，把"请猜测→读取→比较"整段包进去；猜对时 `break` 跳出。

但用户可能输入 `foo` 这种非数字，`expect` 会让程序崩溃。把错误处理从"崩溃"升级为"忽略并重来"：

```rust
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,
};
```

`_` 是通配符，匹配任何 `Err`；`continue` 直接进入下一轮循环。最终完整版代码：

```rust
use rand::Rng;
use std::cmp::Ordering;
use std::io;

fn main() {
    println!("Guess the number!");

    let secret_number = rand::thread_rng().gen_range(1..101);

    loop {
        println!("Please input your guess.");

        let mut guess = String::new();

        io::stdin()
            .read_line(&mut guess)
            .expect("Failed to read line");

        let guess: u32 = match guess.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };

        println!("You guessed: {}", guess);

        match guess.cmp(&secret_number) {
            Ordering::Less => println!("Too small!"),
            Ordering::Greater => println!("Too big!"),
            Ordering::Equal => {
                println!("You win!");
                break;
            }
        }
    }
}
```

两种错误处理方式的取舍可以总结为一张表：

| 场景 | 处理方式 | 行为 |
|---|---|---|
| 读取 stdin 失败 | `.expect()` | 罕见，直接崩溃并报错 |
| 输入非数字 | `match Ok/Err` + `continue` | 高频且可恢复，忽略后重来 |

## 实践建议与总结

1. **主动制造编译错误**：本书故意展示的 `E0308` 类型不匹配错误，是学习 Rust 最好的老师——错误信息会精确告诉你"期望什么、找到什么、怎么改"；
2. **小步迭代**：本章程序是分四步写完的（输入→随机数→比较→循环），每步都用 `cargo run` 验证，这正是 Rustacean 的日常节奏；
3. **`cargo doc --open` 是隐藏宝藏**：能打开所有本地依赖的文档，不确定某个 crate 的 API 时先查它。

这一章你其实学会了 Rust 的"四板斧"：`let` 声明变量（默认不可变）、`match` 处理枚举、方法/关联函数调用、用 Cargo 管理外部依赖。第 3 章会把变量、数据类型、函数、控制流这些基础概念展开讲透——从"会玩"走向"懂原理"。
