# Rust 学习笔记（1/21）：Hello, Rust!——从安装到 Cargo

> 本系列基于官方《Rust 程序设计语言》（The Rust Programming Language，TRPL）逐章学习。

## 开篇：一次带"安检"的旅程

想象你搬进一个新城市：房东先检查你的行李（编译器验证），再给你钥匙（生成可执行文件），之后你随时可以进门——甚至不用告诉房东你还住在这儿。Rust 的学习之旅就是这样开始的：**先把代码编译成独立的可执行文件，别人拿走后无需安装 Rust 也能运行**。这就是 Rust 作为*预编译语言*和 Python、JavaScript 这类动态语言最根本的区别。

本章是全程 21 篇笔记的起点，我们要完成三件事：装好工具链（rustup）、用最原始的方式编译第一个程序（rustc）、然后认识真正的项目管理工具（Cargo）。

## 第一步：安装——rustup 是"版本管家"

Rust 的安装工具叫 `rustup`，它不只装一个编译器，还负责管理 Rust 的版本和相关工具（类似 Node.js 的 nvm）。

- **Linux / macOS**：一条命令搞定
  ```bash
  $ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
  ```
  看到 `Rust is installed now. Great!` 即安装成功。macOS 还需要一个链接器：`xcode-select --install`。
- **Windows**：从官网下载安装程序，过程中会提示安装 Visual Studio 的 "C++ build tools"（含 Windows 10 SDK），这是链接器依赖。
- **日常维护**：`rustup update` 升级到最新稳定版；`rustup self uninstall` 卸载。
- **验证**：`rustc --version` 应输出类似 `rustc x.y.z (abcabcabc yyyy-mm-dd)`；`rustup doc` 打开本地文档，离线也能查 API。

安装时顺带装好的还有 `rustfmt`（自动格式化，统一代码风格）和 `Cargo`。

## 第二步：Hello, World!——解剖第一个程序

创建 `projects/hello_world` 目录，写一个 `main.rs`：

```rust
fn main() {
    println!("Hello, world!");
}
```

编译并运行（分两步）：

```bash
$ rustc main.rs
$ ./main        # Windows 下是 .\main.exe
Hello, world!
```

这个四行程序里有四个值得记住的细节：

1. **`fn main()` 是入口**：每个可执行 Rust 程序都从 `main` 函数开始，参数放括号 `()` 里，函数体必须用大括号 `{}` 包住；
2. **`println!` 是宏不是函数**：看到名字带 `!`，就是调用宏。这解释了为什么它没有加 `!` 时（`println`）不存在；
3. **字符串参数**：`"Hello, world!"` 被传给 `println!` 打印出来；
4. **分号 `;` 结尾**：表示一个表达式结束，Rust 绝大多数行都以 `;` 收尾。

另外注意 Rust 的缩进风格是 4 个空格，不是制表符。

**编译与运行是独立的两步**——这点和 Ruby/Python/JavaScript 完全不同。`rustc main.rs` 之后会生成二进制文件 `main`（Windows 还有 `.pdb` 调试信息文件）。你可以把编译产物直接发给没有装 Rust 的人运行，而发 `.py` 文件给别人，对方还得先装 Python。这是语言设计上的权衡：编译一步换来了分发时的便利。

## 第三步：Hello, Cargo!——真正的项目管家

`rustc` 对付单文件足够，项目一复杂（多文件、依赖库）就抓瞎。**Cargo 是 Rust 的构建系统和包管理器**，绝大多数 Rustacean（Rust 用户的自我昵称）都用它开工。

```bash
$ cargo new hello_cargo
$ cd hello_cargo
```

Cargo 会生成一个标准项目骨架：

```
hello_cargo/
├── Cargo.toml   # 项目配置清单
├── .gitignore   # 自动初始化 Git 仓库
└── src/
    └── main.rs  # 源码必须放在 src/ 下
```

`Cargo.toml` 是 TOML 格式的配置文件：

```toml
[package]
name = "hello_cargo"
version = "0.1.0"
edition = "2021"

[dependencies]
```

- `[package]`：本项目的元信息（名称、版本、edition 大版本）；
- `[dependencies]`：依赖声明区，Rust 里代码包叫 **crate**，第 2 章猜数字游戏会用到。

三个日常命令是核心，它们的关系用一个表格就能看清：

| 命令 | 作用 | 产物 | 特点 |
|---|---|---|---|
| `cargo check` | 快速检查能否编译 | 无可执行文件 | **最快**，写代码时随时跑 |
| `cargo build` | 编译 | `target/debug/` 下可执行文件 | 首次会生成 `Cargo.lock`（记录依赖版本，交给 Cargo 管理，勿手改） |
| `cargo run` | 编译 + 运行 | 同上 | 文件没改就直接运行，改了先重新构建 |
| `cargo build --release` | 优化编译 | `target/release/` | 运行最快但编译更慢，基准测试时用它 |

**开发/发布双配置**：日常开发用 debug 构建（快、带调试信息），给用户交付或做基准测试时用 `--release`（优化充分、跑得快）。

## 实践建议与总结

给新手的三条建议：

1. **从 `cargo new` 开始**，别再用裸 `rustc`——Cargo 的项目结构（源码在 `src/`、配置在根目录 `Cargo.toml`）是 Rust 生态的通用约定，越早习惯越好；
2. **写代码时高频跑 `cargo check`**，确认能编译再 `cargo run`，节省等编译的时间；
3. **`git clone` 一个 Rust 项目后，进来直接 `cargo build`** 就能构建——这也是 Cargo 跨平台一致命令的便利之处（本书从这以后不再区分操作系统的命令差异）。

本章的收获可以浓缩成一句话：**rustup 装工具链，rustc 编译单文件，Cargo 管项目全流程**。第 2 章我们会用 Cargo 做一个真正的猜数字游戏，把变量、循环、match 这些概念串起来——那才是 Rust 之旅真正的第一座山。
