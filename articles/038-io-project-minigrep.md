# Rust 学习笔记（13/21）：I/O 项目 minigrep——把散装知识组装成一台真机器

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。前 11 章我们一直在"造零件"：所有权是螺丝、生命周期是卡扣、trait 是接口标准、测试是质检线。这一章终于要**把它们装成一台真正能用的机器**——一个迷你版 `grep`：`minigrep`。更重要的是，你会看到"先让它跑起来，再拆开重装"的工程节奏，以及 TDD 如何在真实项目里落地。

## 开篇：为什么"能跑"不等于"做完了"

设想你刚组装完一张书桌：四条腿都拧上了，桌面也放上去了，摇一摇不倒——**它"能跑"了**。但接线槽里的线是随便塞的、螺丝规格混用、说明书找不到、下次搬家人家根本不知道怎么拆。

初学者写命令行程序也常常如此：一个 `main` 函数从解析参数、读文件到打印结果全包圆，60 行代码看着也能用。但真正的工程习惯是——**趁代码还小的时候重构**。本章最有价值的部分不是"怎么写 grep"，而是"怎么把一个能跑的 `main` 拆成可测试的模块结构"。

`grep` 是 "**G**lobally search a **R**egular **E**xpression and **P**rint" 的缩写：给它一个字符串和一个文件，它打印出文件中所有包含该字符串的行。本章的 `minigrep` 是它的极简版（真实的 `ripgrep` 功能完备得多）。整个项目会串起第 7 章的模块、第 8 章的集合、第 9 章的错误处理、第 10 章的生命周期与 trait、第 11 章的测试，并顺带预览闭包、迭代器和 trait 对象（第 13、17 章详讲）。

```bash
$ cargo new minigrep
     Created binary (application) `minigrep` project
$ cd minigrep
```

## 第一步：接受命令行参数

标准库的 `std::env::args` 返回一个**迭代器**，生成程序收到的命令行参数。迭代器的两个关键点是：它会按序产出一系列值，且可以调用 `collect` 把它们收集进集合（比如 `Vec<String>`）。

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    println!("{:?}", args);
}
```

几个容易被忽略的细节：

| 细节 | 说明 |
|---|---|
| 为何 `use std::env` 而非 `use std::env::args` | 函数嵌套超过一层模块时，惯用做法是引入**父模块**，代码里写 `env::args()` 更明确，也方便用到 `std::env` 的其他函数 |
| 为何必须写类型标注 | `collect` 能创建很多种集合，Rust 无法推断你要哪种，所以需要 `Vec<String>` |
| `args[0]` 是什么 | 程序自身名称（如 `target/debug/minigrep`），与 C 的参数列表行为一致；本章忽略它 |
| 无效 Unicode 怎么办 | `env::args` 遇到无效 Unicode 会 panic；需要接受时改用 `env::args_os`（返回 `OsString`，跨平台处理更复杂） |

运行 `cargo run needle haystack`，输出是 `["target/debug/minigrep", "needle", "haystack"]`——印证了 `args[0]` 是程序名，所以我们从索引 1 开始取 `query`，索引 2 取 `filename`。

## 第二步：读取文件

引入 `std::fs`，用 `fs::read_to_string` 打开文件并返回 `Result<String>`：

```rust
let contents = fs::read_to_string(filename)
    .expect("Something went wrong reading the file");
```

测试用例用艾米莉·狄金森的短诗 "I'm nobody! Who are you?"（多行、有重复单词，非常适合验证行匹配），存为项目根目录的 `poem.txt`。运行 `cargo run the poem.txt` 能看到文件内容被完整打印——**程序能跑了**。但正如开篇所说，此刻正是重构的时机。

## 第三步：重构——四个问题与"关注分离"

代码虽小，却暴露了四个与"组织方式 + 错误处理"相关的问题：

| 问题 | 症状 | 目标 |
|---|---|---|
| 1. `main` 职责过多 | 既解析参数又读文件，将来还搜索、打印 | 每个函数只负责一件事 |
| 2. 配置变量散落 | `query`、`filename` 与 `contents` 混在一起，难以追踪用途 | 把配置项收进一个结构体 |
| 3. 错误信息不具体 | `expect("Something went wrong...")` 无法区分文件不存在与权限不足 | 错误信息要能指导用户 |
| 4. 错误处理分散且帮不上用户 | 参数不足时抛出 `index out of bounds: the len is 1 but the index is 1` | 错误处理集中到一处，输出用户能懂的话 |

**Rust 社区的"二进制项目关注分离"指导流程**给了一套标准解法：当 `main` 开始膨胀，就把逻辑拆到 `lib.rs`，最终 `main` 只剩两件事——**处理程序运行**（拿参数、调 `run`、处理错误），而**所有真正的业务逻辑**都在 `lib.rs` 里。这样做的关键收益是：**`main` 无法被直接测试，而 `lib.rs` 里的函数可以**。

### 3.1 从 `parse_config` 到 `Config::new`

先把参数解析提取成函数：

```rust
let (query, filename) = parse_config(&args);
```

调用返回元组再立刻拆开，这其实是"抽象还不到位"的信号；而 `config` 这个词也暗示两个值同属一个配置体。于是把它们收进结构体：

```rust
struct Config {
    query: String,
    filename: String,
}

fn parse_config(args: &[String]) -> Config {
    let query = args[1].clone();
    let filename = args[2].clone();
    Config { query, filename }
}
```

这里有个必须理解的取舍：`args` 是 `main` 的所有者、只允许被借用，所以 `Config` **不能持有 `args` 中的引用**，只能持有拥有所有权的 `String`，于是 `clone` 做一次完整拷贝——这是"用一点性能换代码直白（无需管理生命周期）"的典型权衡，在数据短、只拷贝一次的场景完全可接受。而"该用复杂类型时硬用基本类型"的反模式，社区称为**基本类型偏执**（primitive obsession）。

最后一步是把它改成关联函数，更符合习惯（就像 `String::new`）：

```rust
impl Config {
    fn new(args: &[String]) -> Config { /* ... */ }
}
// 调用处
let config = Config::new(&args);
```

### 3.2 错误处理演进：从 panic 到 Result 再到统一出口

参数不足时直接访问 `args[2]` 会 panic 并抛出面向开发者的越界信息。加个长度检查能改善信息，但 `panic!` 仍会附带 `thread 'main'` 与 `RUST_BACKTRACE` 的噪声——按第 9 章的判断标准，**"用户没传够参数"是使用问题，不是程序 bug，应该返回 `Result` 而非 panic**：

```rust
impl Config {
    fn new(args: &[String]) -> Result<Config, &'static str> {
        if args.len() < 3 {
            return Err("not enough arguments");
        }
        let query = args[1].clone();
        let filename = args[2].clone();
        Ok(Config { query, filename })
    }
}
```

调用方用 `unwrap_or_else` 处理 `Err`——它接受一个**闭包**，`Err` 内部的值会传进闭包的参数：

```rust
use std::process;

let config = Config::new(&args).unwrap_or_else(|err| {
    println!("Problem parsing arguments: {}", err);
    process::exit(1);   // 非零退出码告诉调用方"程序以错误状态结束"
});
```

### 3.3 提取 `run`，让错误统一走 `Result`

把业务逻辑移到 `run(config: Config)` 中，并让它返回 `Result<(), Box<dyn Error>>`：

```rust
use std::error::Error;

pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;
    println!("With text:\n{}", contents);
    Ok(())
}
```

三处修改各有讲究：错误类型用 **trait 对象** `Box<dyn Error>`（`dyn` 是 dynamic 的缩写），表示"任何实现了 `Error` trait 的类型"，无需写死具体类型，灵活性最高（第 17 章详讲）；`expect` 换成 `?`，把错误交给调用者而不是就地 panic；成功时返回 `Ok(())`——用 `()` 表明"这个函数的意义在于副作用，没有有意义的值要返回"。

`main` 这边如果只写 `run(config);` 会得到 `unused_must_use` 警告：编译器在提醒你"这个 `Result` 可能是 `Err`，你却没处理"。修正方式是用 `if let`，因为 `run` 成功时返回 `()`，无需 `unwrap`：

```rust
if let Err(e) = run(config) {
    println!("Application error: {}", e);
    process::exit(1);
}
```

### 3.4 拆分到库 crate

把所有非 `main` 的代码（`Config`、`Config::new`、`run` 及相关 `use`）搬进 `src/lib.rs`，并给它们加 `pub`：

```rust
// src/lib.rs
pub struct Config {
    pub query: String,
    pub filename: String,
}

impl Config {
    pub fn new(args: &[String]) -> Result<Config, &'static str> { /* ... */ }
}

pub fn run(config: Config) -> Result<(), Box<dyn Error>> { /* ... */ }
```

`main.rs` 则通过 crate 名引用它们（`use minigrep::Config;`、`minigrep::run(config)`）。至此我们有了一个**带公有 API 的库 crate**，可以正经写测试了。这也是上一章提到的"二进制 crate 无法被集成测试 `use`"的实战解法。

## 第四步：TDD 实现 search 函数

测试驱动开发（TDD）遵循"写失败测试 → 让测试通过 → 重构"的循环。先写测试，等于强迫自己先想清楚**函数签名长什么样**：

```rust
#[test]
fn one_result() {
    let query = "duct";
    let contents = "\
Rust:
safe, fast, productive.
Pick three.";

    assert_eq!(
        vec!["safe, fast, productive."],
        search(query, contents)
    );
}
```

此时 `search` 还不存在，测试编译不过。先写一个"刚好能编译"的骨架让它**失败**：

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    vec![]
}
```

签名里的生命周期**必不可少**：返回值引用的到底是 `query` 还是 `contents` 的 slice，编译器无从判断，会报 `E0106: missing lifetime specifier`。我们明确告诉它——返回的每一行都来自 `contents`：

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.contains(query) {
            results.push(line);
        }
    }

    results
}
```

三步到位：`lines()` 按行迭代 → `contains()` 判断是否包含查询串 → `Vec::push` 收集。测试随即变绿（`test tests::one_result ... ok`）。

接着在 `run` 里真正用上它：

```rust
pub fn run(config: Config) -> Result<(), Box<dyn Error>> {
    let contents = fs::read_to_string(config.filename)?;

    for line in search(&config.query, &contents) {
        println!("{}", line);
    }

    Ok(())
}
```

命令行验证：`cargo run frog poem.txt` 只输出一行（`How public, like a frog`）；`cargo run body poem.txt` 输出三行；而搜索诗中不存在的 `monomorphization` 则什么都不输出——**零匹配是正常结果，不是错误**。

## 第五步：用环境变量控制大小写敏感

新需求：通过环境变量让搜索可切换大小写敏感。依旧先写失败测试，把老测试从 `one_result` 改名为 `case_sensitive`，并为新函数写好期望（注意老测试的文本里特意加了 `"Duct tape."`，确保大写 D 不会被敏感搜索误匹配）：

```rust
pub fn search_case_insensitive<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    let query = query.to_lowercase();   // 变成 String（新分配的数据）
    let mut results = Vec::new();

    for line in contents.lines() {
        if line.to_lowercase().contains(&query) {
            results.push(line);
        }
    }

    results
}
```

注意 `to_lowercase` 会**新建字符串**（`"rUsT"` 里没有现成的小写 `u`），所以 `query` 从 `&str` 变成了 `String`，传给 `contains` 时要写 `&query`。关键设计是**返回的仍是 `contents` 里的行**（`&'a str`），而不是小写后的副本——搜索结果要保留原文原样。

在 `Config` 中加入 `case_sensitive: bool`，`run` 里按它二选一；而 `new` 里读取环境变量：

```rust
let case_sensitive = env::var("CASE_INSENSITIVE").is_err();
```

`env::var` 返回 `Result`：变量已设置为 `Ok`、未设置为 `Err`。我们**只关心"有没有设置"，不关心值是什么**，所以判断 `is_err()`——一旦设置了 `CASE_INSENSITIVE`，就走大小写不敏感搜索：

```bash
$ CASE_INSENSITIVE=1 cargo run to poem.txt
Are you nobody, too?
How dreary to be somebody!
To tell your name the livelong day
To an admiring bog!
```

换成小写的 `to` 查询时只匹配前两行，而 `CASE_INSENSITIVE=1` 后连 `To` 开头的行也被匹配——环境变量生效了。

## 第六步：错误信息写到标准错误

终端提供两条输出流，用途截然不同：

| 流 | 用途 | 打印宏 | 典型场景 |
|---|---|---|---|
| 标准输出 `stdout` | 程序的一般（成功）输出 | `println!` | 搜索结果、正常数据 |
| 标准错误 `stderr` | 错误信息 | `eprintln!` | 参数错误、运行失败 |

这个区分让用户可以**把正常输出重定向到文件，同时错误信息仍然显示在屏幕上**。改造前，`cargo run > output.txt` 会把错误信息也写进文件，屏幕上什么都没有——不符合命令行程序的惯例。把两处打印错误的 `println!` 换成 `eprintln!` 后：

```bash
$ cargo run > output.txt
Problem parsing arguments: not enough arguments   # 错误显示在屏幕，output.txt 为空
$ cargo run to poem.txt > output.txt              # 终端无声，结果写进文件
```

正是期望的行为。改动之所以只有两行，正是因为前面已经把错误处理**收敛到了 `main` 一处**——重构的回报在此刻兑现。

## 实践建议与总结

1. **先跑通，再重构，别怕拆**：`clone`、临时 `println!`、丑陋的 `main` 都可以接受；关键是**趁代码还小的时候**做小步重构，每改一步就跑一次验证（本章的口诀是"经常验证你的进展，出问题时才好定位"）。
2. **用类型表达意图**：`(query, filename)` 元组 → `Config` 结构体，`&'static str` 错误 → `Box<dyn Error>`，`bool` 字段 → 分支选择。类型的形状就是设计的形状，清晰的结构让维护者不必猜。
3. **错误分两类，出口只留一个**：程序 bug 用 `panic!`，用户可预期的失败（参数不足、文件缺失）返回 `Result` 并最终在 `main` 用 `eprintln!` + `process::exit(1)` 统一收口；TDD 则保证每次改动都有测试兜底，`search` 与 `search_case_insensitive` 的并存正是靠测试锁住行为不回归。

本章我们第一次走完了"需求 → 可跑原型 → 重构分层 → TDD 实现 → 环境变量 → 输出流规范"的完整闭环。下一章进入 Rust 的函数式特性：**闭包与迭代器**——`unwrap_or_else(|err| ...)` 里那个竖线包起来的匿名函数究竟怎么工作、编译器如何把它优化到零开销，我们届时一探究竟。
