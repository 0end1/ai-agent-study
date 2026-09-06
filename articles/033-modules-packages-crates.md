# Rust 学习笔记（8/21）：使用包、Crate 和模块——给代码"分房间"

> 本系列基于官方《Rust 程序设计语言》（The Rust Programming Language，TRPL）逐章学习。前几章都在"一张纸上"（单个文件单个模块）写代码：变量、结构体、枚举、函数。但真实程序动辄成千上万行——**当代码长到无法在脑海里通晓整个程序时，组织方式就成了生死线**。本章的主角，就是 Rust 的**模块系统（module system）**。

## 开篇：一家餐厅如何"分房间"

一家像样的餐厅不会让厨师、收银和洗碗工挤在同一张桌子上干活。它一定有**前台**（招待顾客：安排座位、接单、收银、调酒）和**后台**（厨房做菜、洗碗、经理办公）的明确分区——干不同活的人待在不同"房间"，顾客只看得见前台，看不见也不该看见后厨的流程。

Rust 的代码组织哲学和这几乎一模一样：**把相关功能分组进不同的"房间"（模块），并给每个房间一扇门（pub）控制谁能进去**。程序越大，这套"分房间 + 关门"的纪律越重要——它是让多人协作时不互相踩脚、让重构时不炸掉外部代码的根基。

## 包与 crate：两套"最小单元"

动手前先分清两个容易混的词：

- **crate**（箱）是编译的最小单元：一个**二进制项**（可运行的程序）或一个**库**（被别处复用的代码）。每个 crate 有一个 *crate root*——即 *src/main.rs*（二进制）或 *src/lib.rs*（库），编译器从它出发构建出整个 crate 的"根模块"。
- **package**（包）是提供功能的一组 crate + 一份 *Cargo.toml* 构建清单。规则简单：**至多一个库 crate，可以有任意多个二进制 crate，但至少有一个 crate**。

`cargo new my-project` 默认只建 *src/main.rs*，就是一个"单 crate 包"。若再放一个 *src/lib.rs*，包就同时拥有同名的库与二进制两个 crate；往 *src/bin/* 目录放文件，每个文件都会编译成一个独立二进制。

为什么要用 crate 圈住作用域？好处是**名字冲突天然免疫**：`rand` crate 有个 trait 叫 `Rng`，你也可以在自己的 crate 里定义 `struct Rng`——各自在家互不干扰，需要对方时用全路径 `rand::Rng` 指认即可。

## 模块与模块树：给 crate 内部再分区

用 `mod` 关键字定义模块，可以无限嵌套（示例 7-1 是"餐厅"版）：

```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
        fn seat_at_table() {}
    }
    mod serving {
        fn take_order() {}
        fn serve_order() {}
        fn take_payment() {}
    }
}
```

嵌套的模块天然形成一棵**模块树**，树根是隐式的 `crate`：

```
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```

书里点破一个绝佳类比：**模块树就是文件系统目录树**，而模块就是"文件夹"——你想访问 `add_to_waitlist`，就得先给出通往它的"路径"。

## 路径与私有性：通往目标的"门牌号"

### 两种路径

就像文件系统分"绝对路径 / 相对路径"：

- **绝对路径**：以 `crate` 关键字开头（相当于 shell 的 `/`）：`crate::front_of_house::hosting::add_to_waitlist();`
- **相对路径**：从当前模块开始（相当于 `front_of_house/hosting/...`）：`front_of_house::hosting::add_to_waitlist();`

还有从父模块出发的 **`super`**（相当于 `..`）：子模块里想调父模块的函数用 `super::serve_order()`。**本书惯例是优先绝对路径**——因为代码"定义处"和"调用处"往往各自独立搬迁，绝对路径对搬动的鲁棒性更好。

### 私有性边界：默认全关门，pub 逐层开

Rust 里**所有项（函数、结构体、枚举、模块、常量）默认私有**。而且模块天然是一道**私有性边界（privacy boundary）**：父模块不能用子模块的私有项，子模块却可以用父模块的项（就像顾客看不到后厨，但经理能管到全店）。

把示例中的函数从 `eat_at_restaurant` 调出来，编译器会毫不留情地报错（*E0603*），且**一级一级**地拦：

```rust
mod front_of_house {
    mod hosting {              // 第一层：hosting 私有，报错！
        fn add_to_waitlist() {} // 第二层：函数私有，报错！
    }
}
pub fn eat_at_restaurant() {
    crate::front_of_house::hosting::add_to_waitlist();
}
```

修复口诀：**路过的每一层都要 pub**——`pub mod hosting` + `pub fn add_to_waitlist` 全加上才能编译通过。注意：`pub` 只放行"让父模块能引用这一层"，它**不递归**；模块公有 ≠ 内容公有。

### pub struct 与 pub enum：暴露的"颗粒度"不同

给类型加 `pub` 有个反直觉细节：

```rust
mod back_of_house {
    pub struct Breakfast {
        pub toast: String,          // 字段级控制
        seasonal_fruit: String,     // 仍是私有
    }
    impl Breakfast {
        pub fn summer(toast: &str) -> Breakfast { /* 提供公开构造入口 */ }
    }
}
```

**`pub struct` 只公开类型本身，字段默认仍私有**——餐馆里顾客能自选面包（`toast`），却不能挑随季节变换的水果（`seasonal_fruit`）。也因为存在私有字段，外部无法直接字面量构造实例，必须由结构体提供一个公开的构造函数（如 `Breakfast::summer()`）。

**`pub enum` 则所有成员一并公开**（`pub enum Appetizer { Soup, Salad }` 的 `Soup`/`Salad` 立即可用）。原因很朴素：枚举靠"列出全部可能"才有意义，逐个成员加 `pub` 既恼火又没必要。

| 类型 | `pub` 后公开的范围 | 反例 |
|---|---|---|
| `pub struct` | 仅类型名 | 字段仍私有，需逐字段 `pub` + 提供构造入口 |
| `pub enum` | 类型名 + 全部成员 | —— |

## use：给长路径开"快捷方式"

路径虽精确，但每次写 `crate::front_of_house::hosting::add_to_waitlist()` 太长。`use` 可以把某条路径一次性"引进门"——**效果类似文件系统的软连接**：此后 `hosting` 在作用域内就像本地定义的名字一样好用。

```rust
use crate::front_of_house::hosting;   // 绝对路径引入
pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();       // 短名字直接调
}
```

`use` 引入同样受私有性检查，且引进来默认是**私有的**。

### 惯用姿势：函数引"父模块"，类型引"全路径"

```rust
use crate::front_of_house::hosting;          // 函数：引入其父模块，调用时 hosting::xxx
use std::collections::HashMap;               // 结构体：引入完整路径
```

为什么函数不直接 `use ...::add_to_waitlist`？因为调用时写着 `hosting::add_to_waitlist` 能时刻提醒读者"这个函数不是我本地定义的"。类型则相反，`HashMap` 直接裸用最干净。这不是硬性语法，而是社区默认的阅读惯例。

### 同名撞车三招

`use std::fmt::Result; use std::io::Result;` 会让 `Result` 撞名。三招任选：

1. **引父模块区分**：`fmt::Result` 与 `io::Result`，调用处带前缀；
2. **`as` 起别名**：`use std::io::Result as IoResult;`——一句话就能解决；
3. **`pub use` 重导出**：既把项引入作用域、又向外部代码公开这个路径。内部怎么组织（餐厅内部按"前台/后台"）与外部怎么用（顾客只想要一份早餐）可以完全分离——这是写库时对外**塑造 API 形状**的核心手段。

### 外部包与 std

猜数字游戏用的 `rand` 是外部包：先在 *Cargo.toml* 声明 `rand = "0.8.3"`，再用 `use rand::Rng;` 引入。注意：**标准库 `std` 对你而言也是"外部 crate"**——只是它随 Rust 分发、无需写进 Cargo.toml，但引用时同样要 `use std::collections::HashMap`。

### 压缩 use 行数

- **嵌套路径**：`use std::{cmp::Ordering, io};` 把同前缀的多行并一行；想连父模块一起引，用 `self`：`use std::io::{self, Write};`
- **glob 运算符**：`use std::collections::*;` 一次引入全部公有项。慎用——名字来源会变得难以推断；它最正经的用途在测试模块（第 11 章）与 prelude 模式。

## 把模块拆进文件：代码再大也放得下

模块多了，一个文件塞不下时，`mod` 声明后**不加代码块而加分号**，Rust 就去"同名文件"里找模块内容：

```rust
// src/lib.rs
mod front_of_house;                      // 声明：内容在 src/front_of_house.rs
pub use crate::front_of_house::hosting;
```

```rust
// src/front_of_house.rs
pub mod hosting;                          // 再拆一层 → src/front_of_house/hosting.rs
```

```rust
// src/front_of_house/hosting.rs
pub fn add_to_waitlist() {}
```

**模块树、`use`、调用代码一行都不用改**——Rust 在编译时按"文件路径 ≈ 模块路径"自动组装。这让代码增长期的拆分毫无心理负担。

## 实践建议与总结

1. **先目录化思维，再动手拆分**：模块树就是目录树，命名和嵌套都按"读代码的人怎么找"来设计；
2. **默认私有是最好的默认值**：所有项先私有、被外部真正需要时再 `pub`，这样你永远知道改哪些内部实现不会连累调用方；`pub struct` 别忘了提供构造入口，`pub enum` 则要接受"全公开"；
3. **对外 API 是设计出来的**：用 `pub use` 重导出把"内部结构"和"外部形状"解耦，调用方不该被迫理解你后厨的布局。

**一句话收束**：Rust 的组织体系是一套"洋葱"——**package 装 crate，crate 是编译单元，module 给 crate 分区，路径负责寻址，私有性负责保密，`use` 负责抄近道**。这套机制支撑起任何规模的工程而不塌方。下一站将进入日常编程最常用的**标准库集合**（`Vec`、`String`、`HashMap`）——正是从本章"图书馆"里借出的第一批工具。
