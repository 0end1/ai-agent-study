# Rust 学习笔记（10/21）：错误处理——是优雅降级，还是果断停摆？

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。真实程序没有"不出错"的：文件可能找不到、网络可能超时、用户可能乱输入。Rust 对可靠性的执着同样延伸到错误处理上——它要求你**在编译代码之前就承认出错的可能性并采取行动**。这一章登场两个核心工具：不可恢复错误的 `panic!`，和可恢复错误的 `Result<T, E>`。

## 开篇：错误只有两类，别一刀切

试想你是潜水教练：学员在水下呼吸器出了小故障，你有两个选择——**浮回水面喊停**，或**换用备用气源继续调整**。Rust 把错误按此分成两类：

- **不可恢复错误**：通常是 bug 的同义词（比如越界访问数组），继续跑下去没有意义 → `panic!`，程序停止执行。
- **可恢复错误**：值得向用户报告并重试（比如找不到文件，可以改去创建它）→ 返回 `Result<T, E>` 让调用者决策。

大部分语言不区分这两类，统一用"异常（exception）"一把抓；**Rust 根本没有异常**，只有这两个明确分工的工具。有意思的是，这个"二分法"本身不是重点，**重点是你写每个可能失败的函数时，都得先想清楚走哪条路**——这份"想清楚"正是健壮性的来源。

## panic!：果断停摆的艺术

### 停摆的方式：展开 vs 终止

`panic!("crash and burn")` 执行时，程序打印错误信息后默认**展开（unwinding）**——回溯栈、逐个清理沿途函数的数据，然后退出。若希望最终二进制尽量小、不要这份清理开销，可以在 `Cargo.toml` 里切换为**终止（abort）**——不清理数据直接退出，内存交给操作系统回收：

```toml
[profile.release]
panic = 'abort'
```

| | 展开 unwinding | 终止 abort |
|---|---|---|
| 行为 | 回溯并清理每个函数数据 | 不清理，直接退出 |
| 代价 | 清理有额外开销 | 内存交给 OS 回收 |
| 启用 | 默认 | `panic = 'abort'` 配置 |

### 用 backtrace 追凶

`panic!` 报错会指出触发位置：`thread 'main' panicked at 'crash and burn', src/main.rs:2:5`（文件第 2 行第 5 个字符）。但很多 panic 发生在你调用的库内部——比如 `v[99]` 越界时，真正的 `panic!` 在标准库的 `libcore/slice/mod.rs` 里。这时设置环境变量 `RUST_BACKTRACE=1` 运行，会打印**到 panic 为止所有被调用的函数列表**。阅读窍门：**从头往下找，第一个出现你编写的文件的那一行，就是问题发源地**。

顺带一提：同样的越界在 C 里会**缓冲区溢出（buffer overread）**——返回内存中不属于你的数据，可能被攻击者利用读取越权信息。Rust 选择直接停止执行，把这类安全漏洞掐死在编译后第一时间。

## Result：优雅降级的正式通道

### 从 match 到按错误类型分支

`Result<T, E>` 是个泛型枚举：成功返回带数据的 `Ok(T)`，失败返回带错误详情的 `Err(E)`。怎么知道函数会返回它？直接问编译器——故意把返回值标注成错误类型 `let f: u32 = File::open(...)`，E0308 报错会贴心告诉你真实类型是 `std::result::Result<std::fs::File, std::io::Error>`。

用第 6 章的 `match` 就能优雅处理：

```rust
let f = File::open("hello.txt");
let f = match f {
    Ok(file) => file,
    Err(error) => panic!("Problem opening the file: {:?}", error),
};
```

更进一步，不同失败原因该有不同对策。`io::Error` 的 `kind()` 方法返回 `io::ErrorKind` 枚举，用嵌套 `match` 就能区分"文件不存在→创建它"和"无权限→直接 panic"：

```rust
Err(error) => match error.kind() {
    ErrorKind::NotFound => match File::create("hello.txt") {
        Ok(fc) => fc,
        Err(e) => panic!("Problem creating the file: {:?}", e),
    },
    other_error => panic!("Problem opening the file: {:?}", other_error),
},
```

这样一层层 `match` 显然啰嗦——第 13 章会学到用闭包 `unwrap_or_else` 把这些压缩成可读性高得多的写法。

### unwrap 与 expect：panic 的"速效简写"

`unwrap()` 等价于"是 Ok 就取值，是 Err 就 panic"。`expect("Failed to open hello.txt")` 几乎一样，但 panic 信息以你给的文本开头——**多个 unwrap 打印相同默认信息，panic 后难以分辨凶手；expect 的自定义信息直接点名**。

```rust
let f = File::open("hello.txt").unwrap();                          // 默认信息
let f = File::open("hello.txt").expect("Failed to open hello.txt"); // 自定义信息
```

## 错误传播：把难题留给更有信息的人

函数内部处理不了（或不该处理）的错误，可以**传播（propagate）**给调用者——调用者掌握更多上下文，更知道怎么决定。实现"读用户名"函数，写法从"笨"到"巧"有四步演进：

```rust
// ① 手写 match 逐层返回（啰嗦）
fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),        // 显式提前返回
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}

// ② ? 运算符：Err 自动作为函数返回值传播
fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}

// ③ 链式调用进一步压缩
fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    Ok(s)
}

// ④ 标准库直接给了一行版本
fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("hello.txt")   // 打开+建 String+读取+返回，一气呵成
}
```

`?` 的本质就是那个 `match`：**`Ok` 就取出值继续；`Err` 就当作整个函数的结果返回**。它还悄悄做了一件事——把错误值经过 `From` trait 的 `from` 函数转换成当前函数声明的错误类型，让你可以用单一错误类型代表多种失败来源。

**限制**：`?` 只能用在返回 `Result`（或 `Option`，及实现 `Try` 的类型）的函数里，因为 Err 分支要靠 `return` 传播。想在 `main` 里用 `?`，可以让 `main` 返回 `Result`：

```rust
fn main() -> Result<(), Box<dyn Error>> {   // Box<dyn Error> ≈ "任意类型的错误"
    let f = File::open("hello.txt")?;       // 出错时 main 直接返回 Err，程序以失败码退出
    Ok(())
}
```

## panic 还是不 panic：设计的岔路口

代码一旦 panic 就没有恢复可能。**返回 `Result` 等于把决定权交给调用者**，因此它是定义"可能失败函数"的好默认。以下场景 panic 反而合理：

| 适合 panic | 理由 |
|---|---|
| 示例、代码原型 | `unwrap` 是"将来该处理成什么样"的清晰占位标记 |
| 测试 | 测试目标失败就应让整个测试失败 |
| 比编译器更懂 | 硬编码 `"127.0.0.1".parse().unwrap()`——逻辑上不可能失败 |
| 违反契约（bad input） | 传入值破坏函数的前提，说明是**调用方的 bug** |

**有害状态（invalid state）**是指"假设、保证、协议或不可变性被打破"，如收到无效/矛盾/缺失的值。当有害状态出现时 panic 合理（库要及早警告使用者的 bug）；而当**失败本身是预期内**的（解析器遇到格式错误、HTTP 限流）就该返回 `Result`。注意区分：panic 惩罚的是"调用方的 bug"，而这类 bug 该让**开发者**修，不该让库来兜底。函数契约及其 panic 条件应写进 API 文档。

### 把规则写进类型：Guess 模式

与其在几十个函数里重复"检查 1-100"的 if，不如**把有效性的约束封装成类型本身**：

```rust
pub struct Guess { value: i32 }               // value 私有，外界无法直接赋值

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {}.", value);
        }
        Guess { value }
    }
    pub fn value(&self) -> i32 { self.value }  // getter 只读
}
```

从此任何接收（或返回）1-100 数字的函数都声明成接收 `Guess` 而非 `i32`——**不合法的值根本造不出来**，函数体内无需再检查。这正是 Rust 类型系统的威力：把"运行时到处 if 判空"变成"编译期类型就排除非法状态"（非 `Option` 类型保证有值、`u32` 保证非负，同理）。

## 实践建议与总结

1. **默认返回 `Result`**：把错误处理的选择权交给调用者，这是设计 API 的默认姿势；panic 是少数场景的刻意选择。
2. **错误传播用 `?`**：别手写 `match` 样板；普通代码里它几乎总是够用，也让错误类型能统一转换。
3. **趁早把规则做成类型**：一个 `Guess` 式的封装 + 一次 `panic!`，胜过一百次分散的 if 检查——让非法值连出生的机会都没有。

`panic!` 代表"无法处理的状态"——停下来，绝不用无效值继续；`Result` 代表"可以恢复的失败"——把选择权交出去。二者配合，配合第 6 章的 `Option`，Rust 程序面对不可避免的错误时依然可靠。标准库的两大泛型枚举都已见识过——下一章正是泛型（以及 trait 与生命周期）如何在你的代码里工作。
