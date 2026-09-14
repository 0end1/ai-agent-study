# Rust 学习笔记（15/21）：Cargo 进阶与 Crates.io——从"能跑"到"能被别人用"

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。前几章我们一直在写代码本身：所有权、测试、迭代器。这一章换个视角，把 Cargo 当作真正的工程工具来用——定制构建配置、写文档注释、把 crate 发布到 crates.io，以及用工作空间管理一组协同开发的包。一句话：从"我自己能跑"走向"别人能用、团队能一起维护"。

## 开篇：同一份代码，两副面孔

`cargo build` 和 `cargo build --release` 的输出你多半见过：

```
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.0 secs
$ cargo build --release
    Finished release [optimized] target(s) in 0.0 secs
```

方括号里的 `unoptimized + debuginfo` 与 `optimized`，就是两套**发布配置**（release profiles）的差异。它们不是"两个 Cargo"，而是同一条编译流水线的两组参数：调试时希望编译快、能打断点；发布时希望程序快、体积小，编译久一点无所谓。理解了这一点，本章几乎所有"发布相关"的设计就都好懂了——**在不同阶段付出不同的代价**。

## 一、发布配置：把编译参数写进 Cargo.toml

Cargo 有 `dev` 与 `release` 两个主要配置，分别对应 `cargo build` 与 `cargo build --release`。项目里没有 `[profile.*]` 时全部用默认值；一旦写上，只覆盖你写的那部分：

```toml
[profile.dev]
opt-level = 0

[profile.release]
opt-level = 3
```

`opt-level` 取 0~3，越高优化越多、编译越慢。开发期频繁编译，所以 `dev` 默认 0；发布只编译一次却要运行很多次，所以 `release` 默认 3。想在开发时多要一点性能，改成 1 即可——不必自己发明一套构建流程。

| 配置 | 触发方式 | opt-level 默认 | 取舍 |
|---|---|---|---|
| `dev` | `cargo build` | 0 | 编译快、便于调试 |
| `release` | `cargo build --release` | 3 | 运行快、编译慢 |

## 二、文档注释：写给"使用者"而非编译器

`//` 是给读源码的人看的，`///` 则是**文档注释**（documentation comments）：支持 Markdown，由 `rustdoc` 渲染成 HTML，用来告诉别人如何**使用**你的 crate，而不是它如何被**实现**。

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

运行 `cargo doc` 会在 `target/doc` 生成 HTML；`cargo doc --open` 直接构建并打开浏览器（连同所有依赖的文档）。

除了 `# Examples`，社区常用的小节还有三个，本质是一份"提醒你去检查"的清单：

| 小节 | 何时写 |
|---|---|
| `# Examples` | 几乎总是值得写，最直观 |
| `# Panics` | 函数可能 panic 的场景 |
| `# Errors` | 返回 `Result` 时，何种情况返回 `Err` |
| `# Safety` | `unsafe` 函数对调用者的前置要求 |

文档注释最妙的一点是**文档即测试**：`cargo test` 会把文档里的示例代码当测试运行：

```
   Doc-tests my_crate
running 1 test
test src/lib.rs - add_one (line 5) ... ok
```

如果你改了函数却忘了改例子，`assert_eq!` 就会失败——文档从此不会"过期"。

## 三、`//!` 与 `pub use`：让使用者少走几步

`///` 描述它**之后**的项；`//!` 描述**包含它的项**，因此常写在 `src/lib.rs` 或模块根部，为整个 crate 或模块写总览：

```rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.
```

它必须放在文件最顶部（"最后一行之后没有任何代码"恰恰说明它属于这个文件本身）。

`pub use` 解决另一个问题：**你的文件结构往往不等于用户想要的 API 结构**。假设 `art` 库把内容拆成 `kinds` 与 `utils` 两个模块，用户必须写：

```rust
use art::kinds::PrimaryColor;
use art::utils::mix;
```

作者只需在 `lib.rs` 顶部加三行重导出（re-export）：

```rust
pub use self::kinds::PrimaryColor;
pub use self::kinds::SecondaryColor;
pub use self::utils::mix;
```

用户就能写成 `use art::PrimaryColor;`。`pub use` 让**内部组织**与**对外 API** 解耦：内部维持清晰分层，对外提供扁平入口，`cargo doc` 首页也会直接列出这些重导出项。原有的深层路径依然可用，只是不再强迫用户走那么深。

## 四、发布到 crates.io：把代码交到别人手里

发布不是一条命令，而是一串有先后顺序的准备：

| 步骤 | 命令/动作 | 要点 |
|---|---|---|
| 1. 注册账号 | 用 GitHub 登录 crates.io，在账户页获取 API token | token 是秘密，泄露要立刻重新生成 |
| 2. 登录本地 Cargo | `cargo login <token>` | 存入 `~/.cargo/credentials` |
| 3. 补全元信息 | 编辑 `Cargo.toml` 的 `[package]` | 名称唯一（先到先得）、`description`、`license` |
| 4. 发布 | `cargo publish` | 先在本地打包并编译验证 |

缺 `description` 或 `license` 时，`cargo publish` 会直接报 `error: api errors: missing or empty metadata fields: description, license.`。补齐后大致是这样：

```toml
[package]
name = "guessing_game"
version = "0.1.0"
authors = ["Your Name <you@example.com>"]
edition = "2018"
description = "A fun game where you guess what number the computer has chosen."
license = "MIT OR Apache-2.0"
```

`license` 用 SPDX 标识符；找不到合适的就用 `license-file` 指向许可证文件。许多 Rust 项目选择 `MIT OR Apache-2.0` 双许可（`OR` 分隔多个标识符）。

两条必须记住的规则：

- **发布是永久性的**：版本号一旦发布，不能覆盖、不能删除代码。crates.io 要做"永久文档服务器"，好让所有依赖者的构建永远可复现；想改就升版本号（遵循语义化版本）再发布一次。
- **`cargo yank` 只是撤回，不是删除**：`cargo yank --vers 1.0.1` 让新项目不再选中该版本，但已有 `Cargo.lock` 的项目照旧能下载，不会被"断供"，加 `--undo` 可撤销撤回。它**不是**用来删除误传密钥的——那种情况请立刻重置密钥。

## 五、工作空间：让一组 crate 一起长大

项目变大时，"一个大库"往往该拆成几个协同的小 crate。Cargo 的**工作空间**（workspaces）正是为这种场景准备：一组共享同一个 `Cargo.lock` 和同一个 `target` 目录的包。

```
├── Cargo.toml      ← 只写 [workspace]，没有 [package]
├── Cargo.lock      ← 全工作空间唯一
├── add-one/        ← 库 crate
│   ├── Cargo.toml
│   └── src/lib.rs
├── adder/          ← 二进制 crate
│   ├── Cargo.toml
│   └── src/main.rs
└── target/         ← 全工作空间唯一
```

根 `Cargo.toml` 只负责登记成员：

```toml
[workspace]

members = [
    "adder",
    "add-one",
]
```

`adder` 要用 `add-one`，需显式声明**路径依赖**（Cargo 不假定工作空间内的 crate 互相依赖）：

```toml
[dependencies]
add-one = { path = "../add-one" }
```

| 关注点 | 工作空间下的行为 |
|---|---|
| 构建产物 | 统一放在根 `target/`，避免各 crate 重复编译 |
| 依赖版本 | 唯一的 `Cargo.lock`，保证所有成员用同一版本、彼此兼容 |
| 使用依赖 | 每个要用的 crate 仍需在自己 `Cargo.toml` 中声明（但不会重复下载） |
| 运行/测试 | `cargo run -p adder`、`cargo test`（全部）、`cargo test -p add-one`（单个） |
| 发布 | 没有 `--all`/`-p`，只能进入每个 crate 目录逐个 `cargo publish` |

一个容易踩的坑是**外部依赖**：假设 `add-one` 在 `Cargo.toml` 里加了 `rand = "0.5.5"`，根目录 `Cargo.lock` 会记录它；但若 `adder` 也写 `use rand;`，编译就会报错——`rand` 并未成为 `adder` 的依赖。Cargo 不做"依赖传染"：每个 crate 想用什么，都要在自己的 `Cargo.toml` 里声明。好在声明之后不会下载第二份拷贝，所有成员共享同一版本。测试同理：根目录 `cargo test` 跑遍所有成员，`cargo test -p add-one` 只跑指定 crate。

共享 `target` 与 `Cargo.lock` 是最实际的两个收益：既省磁盘、省编译时间，也从根上消除了"同一依赖装了两个版本"的不兼容麻烦。

## 六、装别人的工具、扩展自己的能力

`cargo install` 用来在本地安装**二进制** crate（只有含 `src/main.rs` 这类二进制目标的包能装），比如第 12 章提到的 `ripgrep`：

```
$ cargo install ripgrep
  Installing ~/.cargo/bin/rg
```

它装到 Rust 根目录的 `bin`（rustup 默认是 `$HOME/.cargo/bin`），把这个目录放进 `$PATH` 就能直接用。其定位是"方便 Rust 开发者安装社区工具"，而非替代系统包管理器。

Cargo 还能被**扩展**：只要 `$PATH` 里有名为 `cargo-something` 的可执行文件，就能用 `cargo something` 调用它，甚至会被 `cargo --list` 列出。于是"安装第三方 cargo 子命令"的体验与内建命令几乎一致。

## 实践建议与总结

1. **配置是分阶段的**：`dev`/`release` 的差异提醒我们，开发期优先反馈速度、发布期优先运行质量；`opt-level` 是唯一常见的手调旋钮，别急着全局优化。
2. **文档要写成"可执行的承诺"**：`///` + `# Examples` 让 `cargo test` 替你检查示例是否过期，`//!` 给出 crate 总览，`pub use` 把好用的入口顶到台前——三者合起来才是"别人能用"的文档。
3. **先想清楚公开边界再发布**：名称唯一、版本永久、依赖锁版本是硬约束；发布前补齐 `description`/`license`，出事后记住 `yank` 是"止损"而非"删除"。

回到开头那句：Cargo 的进阶功能几乎都在回答同一个问题——**这份代码要交给别人（或未来的自己）吗？** 要交给别人，就该有文档、有明确的公开 API、有可复现的依赖与版本。下一章我们重新回到语言本身，看 `Box<T>`、`Rc<T>`、`RefCell<T>` 这些"智能指针"如何在编译期规则之外，为数据所有权提供更多可能。
