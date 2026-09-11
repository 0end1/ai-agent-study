# Rust 学习笔记（12/21）：编写自动化测试——给代码装上"自动质检线"

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。Dijkstra 有句名言："软件测试是证明 bug 存在的有效方法，而证明其不存在时则显得令人绝望的不足。"——但这不代表我们不该尽可能测试。Rust 的类型系统能拦下类型错误，却拦不下"参数本该加 2 结果加了 3"这类逻辑 bug。本章教你用 `#[test]`、断言宏与 `cargo test`，给代码装一条**每次改动都能自动重跑的质检线**。

## 开篇：为什么需要"质检线"

工厂里没人相信"工人手艺好，不用质检"——因为手艺再好也会疲劳、会疏忽、会被临时改动带偏。软件更是如此：你今天把 `can_hold` 里的 `>` 手滑写成 `<`，代码照样编译通过、照样能跑，但行为已经错了。

Rust 的测试就是这条自动化质检线：**每次修改代码后运行 `cargo test`，几秒钟内告诉你"原有正确行为有没有被改坏"**。这是重构时最大的底气来源——没有测试，你不敢改老代码；有了测试，改动立刻有反馈。

## 如何编写测试

一个测试函数本质上只做三件事：**准备数据 → 运行被测代码 → 断言结果符合预期**。Rust 提供的专用工具是 `#[test]` 属性、几个断言宏，以及 `should_panic` 属性。

### 测试函数剖析

`#[test]` 是属性（attribute），加在 `fn` 行之前，标记"这是一个测试"。`cargo test` 会构建测试执行程序，调用所有带 `#[test]` 的函数并报告通过/失败。用 `cargo new adder --lib` 新建库项目，Cargo 自动生成：

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

运行 `cargo test` 的输出分三部分：**单元测试**（`running 1 test` / `test tests::it_works ... ok` / 摘要 `test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out`）、**集成测试**、以及 **Doc-tests**——Rust 会编译 API 文档中的代码示例，让文档与代码保持同步（详见第 14 章）。

有意思的是，**测试函数中出现 panic 就算失败**：每个测试跑在独立线程里，主线程发现某个测试线程异常了就把它标记为失败。所以最粗暴的失败方式就是 `panic!("Make this test fail")`。

### 断言宏工具箱

| 宏 | 作用 | 失败时输出 |
|---|---|---|
| `assert!(expr)` | 断言布尔表达式为 `true` | 只显示断言失败与行号 |
| `assert_eq!(a, b)` | 断言两值相等（`==`） | 打印 `left` / `right` 具体值 |
| `assert_ne!(a, b)` | 断言两值不等（`!=`） | 同上 |
| 自定义信息 | 三者都可追加 `"..{}..", value` | 附加你的说明与上下文 |

以第 5 章的 `Rectangle::can_hold` 为例（返回 `bool`，天然适合 `assert!`）：

```rust
#[cfg(test)]
mod tests {
    use super::*;   // 把外部模块的项引入测试模块作用域

    #[test]
    fn larger_can_hold_smaller() {
        let larger = Rectangle { width: 8, height: 7 };
        let smaller = Rectangle { width: 5, height: 1 };
        assert!(larger.can_hold(&smaller));
    }

    #[test]
    fn smaller_cannot_hold_larger() {
        let larger = Rectangle { width: 8, height: 7 };
        let smaller = Rectangle { width: 5, height: 1 };
        assert!(!smaller.can_hold(&larger));   // 期望 false，取反后传入
    }
}
```

`tests` 只是普通模块，遵循第 7 章的可见性规则，所以要 `use super::*;` 把外部内容引入。一旦把 `can_hold` 里的 `>` 误改成 `<`，`larger_can_hold_smaller` 立刻 FAILED——测试抓到了 bug。

比较值相等用 `assert_eq!` / `assert_ne!` 更省事，因为失败时能打印双方的值：

```rust
pub fn add_two(a: i32) -> i32 { a + 2 }

#[test]
fn it_adds_two() { assert_eq!(4, add_two(2)); }
```

注意 Rust 里两个参数叫 `left` / `right`，**不区分 expected/actual 顺序**；把实现改成 `a + 3` 后，失败信息会告诉你 `left: 4, right: 5`。另外这两个宏底层用 `==`/`!=` 并需要调试格式打印，因此被比较的类型**必须实现 `PartialEq` 和 `Debug`**——自定义结构体/枚举通常加 `#[derive(PartialEq, Debug)]` 即可。

`assert_ne!` 的适用场景是"不确定值会是什么，但确定它绝不会是什么"（比如函数保证输出会被改变，但改变方式取决于当天星期几）。

### 自定义失败信息与 should_panic

当断言只报"失败+行号"不够用时，追加格式字符串。比如测试"问候语包含人名"，需求还没定死 `Hello` 前缀：

```rust
assert!(
    result.contains("Carol"),
    "Greeting did not contain name, value was `{}`", result
);
```

失败时就变成 `Greeting did not contain name, value was 'Hello!'`——一眼看出实际值与期望差在哪。

有时反而要**验证代码会 panic**，比如 `Guess::new` 传入越界值应当崩溃（第 9 章的 `Guess` 类型）：

```rust
#[test]
#[should_panic(expected = "Guess value must be less than or equal to 100")]
fn greater_than_100() { Guess::new(200); }
```

`should_panic` 只要发生 panic 就通过，太含糊——**可能因错误的原因 panic**。加上 `expected` 参数后，测试工具会校验 panic 信息包含指定文本（可以是子串）。示例中故意把两个 panic 分支对调，失败信息会明确指出"panic 了，但信息里没有我期望的字符串"。

### 用 Result<T, E> 写测试

测试也可以返回 `Result<(), E>` 而不是 panic：

```rust
#[test]
fn it_works() -> Result<(), String> {
    if 2 + 2 == 4 { Ok(()) }
    else { Err(String::from("two plus two does not equal four")) }
}
```

好处是**能在测试体内用 `?` 运算符**：任何操作返回 `Err` 测试即失败，写多步操作很顺手。注意两点限制：带 `Result` 的测试**不能同时用 `#[should_panic]`**；要断言"操作会返回 Err"，别用 `?`（那会直接把测试变成功/失败），改用 `assert!(value.is_err())`。

## 控制测试如何运行

`cargo test` 的参数分两段：`--` 之前给 cargo，之后给测试二进制。`cargo test --help` 看前者，`cargo test -- --help` 看后者。

| 需求 | 命令 | 说明 |
|---|---|---|
| 不要并行（避免共享状态互相干扰） | `cargo test -- --test-threads=1` | 默认多线程并行，更快但要求测试互不依赖 |
| 显示通过测试的输出 | `cargo test -- --show-output` | 默认会截获 `println!`，只在失败时显示 |
| 只跑某个测试 | `cargo test one_hundred` | 只使用第一个参数；摘要显示 `2 filtered out` |
| 过滤跑多个测试 | `cargo test add` | 名称包含 `add` 的都跑；**模块名也算名称的一部分** |
| 只跑被标记忽略的 | `cargo test -- --ignored` | 配合 `#[ignore]` 跳过耗时测试 |

几点细节值得记牢：

- **并行是默认**，所以测试之间不能互相依赖（尤其不能共享文件、环境变量、当前工作目录）。经典坑：多个测试同时读写同一个 `test-output.txt`，彼此干扰导致假失败——解决办法是各用不同文件，或直接 `--test-threads=1`。
- **通过的测试看不到 `println!` 输出**，因为被截获了；想看就加 `--show-output`。
- 被 `#[ignore]` 标记的测试默认不跑，输出里显示为 `ignored`，需要时用 `cargo test -- --ignored` 单独运行。这是"耗时一小时的测试"的标准处理方式。

## 测试的组织结构

Rust 社区把测试分成两大类，二者互补，都要写：

| | 单元测试（unit tests） | 集成测试（integration tests） |
|---|---|---|
| 位置 | 与源码同文件，`tests` 模块内 | 项目根目录 `tests/` 下 |
| 视角 | 小而集中，隔离测试单个模块 | 完全外部，像其他使用者一样调用库 |
| 可测范围 | 公开接口 **+ 私有函数** | 仅公有 API |
| 是否需要 `#[cfg(test)]` | 需要（同文件，`cargo build` 时不编译） | 不需要（`tests/` 是特殊目录，只在 `cargo test` 时编译） |
| 编译单元 | 与所在源码同 crate | **每个文件都是独立 crate** |

### 单元测试：同文件 + #[cfg(test)]

规范做法是在每个源文件里建 `tests` 模块并用 `#[cfg(test)]` 标注。`cfg` 代表 configuration，表示"只在 `cargo test` 时才编译这段代码"——这样 `cargo build` 不会把测试编进产物，加快编译、减小体积。

**Rust 允许测试私有函数**（这点在其他语言里往往做不到）：

```rust
pub fn add_two(a: i32) -> i32 { internal_adder(a, 2) }

fn internal_adder(a: i32, b: i32) -> i32 { a + b }   // 私有

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn internal() { assert_eq!(4, internal_adder(2, 2)); }
}
```

因为 `tests` 也只是模块树中的一个子模块，`use super::*;` 之后自然能访问父模块的私有项。

### 集成测试：tests 目录

在项目根目录建 `tests/` 目录（与 `src/` 同级），其中每个 `.rs` 文件都是**独立的 crate**，因此必须显式 `use adder;` 导入被测库：

```rust
// tests/integration_test.rs
use adder;

#[test]
fn it_adds_two() { assert_eq!(4, adder::add_two(2)); }
```

运行 `cargo test` 会看到三段输出：单元测试、集成测试、Doc-tests。集成测试部分每个文件一个结果段。可以按函数名运行指定测试，也可以用 `cargo test --test integration_test` 跑指定文件的全部测试。

**共享辅助函数的坑**：若建 `tests/common.rs` 放 `setup()`，它会被当成一个集成测试文件，输出里多出一段 `running 0 tests`。正确做法是建 `tests/common/mod.rs`——目录形式告诉 Cargo "这不是测试文件"，然后在测试文件里 `mod common;` + `common::setup()` 使用。

**二进制 crate 的限制**：若项目只有 `src/main.rs` 而没有 `src/lib.rs`，`tests/` 里无法 `use` 其中的函数——只有库 crate 才对外暴露函数。这就是 Rust 二进制项目常见"`src/main.rs` 只调用 `src/lib.rs` 中的逻辑"的原因：核心逻辑放库 crate 里可被集成测试覆盖，`main.rs` 里只剩少量无需测试的胶水代码。

## 实践建议与总结

1. **测试命名即文档**：用 `larger_can_hold_smaller`、`greater_than_100` 这种"描述行为"的名字，失败输出一眼看懂；配合自定义失败信息，调试成本直接砍半。
2. **断言选对工具**：布尔判断用 `assert!`，比相等用 `assert_eq!`/`assert_ne!`（能打印双方值），"必须 panic"用 `#[should_panic(expected = "..")]`，多步操作可用返回 `Result` 的测试 + `?`。
3. **单元 + 集成两层别偏废**：单元测试盯紧每个模块的细节（还能测私有函数），集成测试验证多个部分拼起来是否正常——尤其记得把核心逻辑放进库 crate，否则集成测试根本够不着它。

Rust 的类型系统与所有权规则能挡掉大量 bug，但"逻辑不符合预期"这类问题只有测试能抓。更重要的是，测试给你**改代码的勇气**：无论重构还是加功能，一条绿色的质检线就是最好的安全感。下一章，我们用前面学到的文件、命令行、错误处理、测试等所有技能，一起动手写一个真正的命令行程序——`minigrep`。
