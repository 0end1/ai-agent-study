# Rust 学习笔记（7/21）：枚举与模式匹配——把"可能的情况"变成类型

> 本系列基于官方《Rust 程序设计语言》（The Rust Programming Language，TRPL）逐章学习。上一篇用 struct 给数据加了"表头"，本篇再进一步：**当"可能性"本身需要建模时**——一个值只能是 A 或 B 或 C——就该 `enum` 出场了。

## 开篇：一张只能勾一个的选项表

填表时总有这种字段：性别只能是"男/女/保密"，IP 地址只能是 IPv4 或 IPv6，消息只能是"退出/移动/发送/改色"。这类字段的本质是——**取值来自一个有限集合，且一次只能选一个**。用字符串硬存？拼写错误、语义混淆接踵而至。Rust 的**枚举（enum）**把这套"有限选项"变成了编译器可检查的类型：你**枚举**出所有可能的值，这正是它名字的由来。在函数式语言里，它叫**代数数据类型（ADT）**。

## 定义枚举与枚举值：先列出所有成员

`struct` 用 `{}` 定义字段，枚举用 `{}` 列出**成员（variant）**，关键字从 `struct` 换成 `enum`：

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

创建成员实例要用 `::`（成员位于其标识符的命名空间中）：

```rust
let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

`IpAddrKind::V4` 与 `IpAddrKind::V6` 都是 `IpAddrKind` 类型，于是 `fn route(ip_type: IpAddrKind) {}` 可以接受**任一**成员——这正是"这些变体本质上是同一类"的建模方式。

### 把数据直接"长"在成员上

如果只是标记类型，用 `struct { kind, address }` 也能存数据，但枚举更胜一筹——**每个成员可以携带不同类型、不同数量的数据**：

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),   // IPv4 用四个 0-255 的数字
    V6(String),           // IPv6 用一个字符串
}
let home = IpAddr::V4(127, 0, 0, 1);
```

同样的需求用结构体根本写不出来（不同成员的数据形态不同）。标准库的 `IpAddr` 就是类似设计。再看一个"整包多功能"的例子——`Message` 枚举的四个成员形态各异：

```rust
enum Message {
    Quit,                             // 无数据（类单元）
    Move { x: i32, y: i32 },          // 具名结构体式
    Write(String),                    // 单个 String（元组式）
    ChangeColor(i32, i32, i32),       // 三个 i32
}
```

要写出能统一处理四种不同"消息形状"的代码，用四个 struct 是不行的——它们的类型各不相同；而 `Message` 是**单独一个类型**，任何成员都算"一条 Message"。与 struct 一样，枚举也能配 `impl` 块定义方法：

```rust
impl Message {
    fn call(&self) { /* 方法体 */ }
}
let m = Message::Write(String::from("hello"));
m.call();
```

## Option<T>：不引入 null，却更安全

讨论到最常用的枚举前，先听一段名场面。null（空值）的发明者 Tony Hoare 在 2009 年演讲中称之为 **"十亿美元的错误"**（The Billion Dollar Mistake）：四十多年来它引发了无数错误、漏洞与崩溃。null 的问题在于——**当你把它当非空值使用时才出错**，而这在代码里防不胜防。

空值的"概念"其实有用（表示"当前缺失/无效"），问题在"实现"。Rust 的选择是：**没有 null**，但有一个表达"有或没有"的枚举 `Option<T>`：

```rust
enum Option<T> {
    Some(T),    // 有值，T 是任意类型（泛型，第 10 章细讲）
    None,       // 没值
}
```

它太常用，已被纳入 **prelude**（预导入），写 `Some`/`None` 都不用加 `Option::` 前缀：

```rust
let some_number = Some(5);              // 类型能推断
let some_string = Some("a string");
let absent_number: Option<i32> = None;  // None 无法推断 T，须显式标注
```

为什么比 null 安全？**`Option<T>` 和 `T` 是两种类型**，编译器不允许把它们混用：

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);
let sum = x + y;   // error[E0277]：i8 与 Option<i8> 不能相加
```

看到这你可能觉得"烦人"，但这正是重点：一个普通 `i8` 编译器保证它**必有有效值**，可以放心运算；一旦遇到 `Option<i8>`，编译器就逼你先处理"为空"的可能——**假设非空却实际为空的 bug 在编译期被拦截**，而不是运行时崩溃。只要类型不是 `Option<T>`，你就可以大胆假设它非空。

## match：一台优雅的"硬币分类器"

从 `Option<T>` 里取出值、或按枚举成员分流，靠的是 Rust 最强的控制流运算符 **`match`**。它的机制可以想象成**硬币分类器**：硬币（值）滑过一排孔径（模式），掉进第一个"合身"的孔（分支）。

```rust
enum Coin { Penny, Nickel, Dime, Quarter }

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,   // 每个分支：模式 => 代码，逗号分隔
    }
}
```

每个分支的代码是**表达式**，其值就是整个 `match` 的返回值。多行逻辑用大括号包住（最后一行仍作为分支的值）。与 `if` 不同，`match` 后的表达式可以是任意类型，不限于布尔。

### 绑定值的模式：把数据从成员里"接"出来

分支模式可以绑定匹配到的那部分值。若 `Quarter` 里藏着一个州：

```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Quarter(state) => {   // state 绑定 Quarter 携带的 UsState
            println!("State quarter from {:?}!", state);
            25
        }
        _ => 25,   // 其余币种略...
    }
}
```

再以 `Option<i32>` 为例，写个 `plus_one`：

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,          // None 直接走这里
        Some(i) => Some(i + 1),// Some(5) 时 i 绑定 5，返回 Some(6)
    }
}
```

`match` 的一个隐藏福利：**分支代码是一个作用域**，绑定的 `i`、`state` 只在对应分支内有效，用完即弃，绝不会"泄漏"到别处污染逻辑。

### 穷尽性检查：编译器强迫你"列全"

试着只写 `Some` 分支、漏掉 `None` 试试：

```
error[E0004]: non-exhaustive patterns: `None` not covered
```

Rust 的匹配是**穷举式的**：必须覆盖所有可能，否则拒绝编译——它甚至知道**你漏掉了哪个模式**。这个设计让"忘了处理空值"这种亿万级错误在编译期灰飞烟灭。

### 通配模式与 `_` 占位符

穷尽 ≠ 必须逐个列出。用**通配分支**兜底其余所有情况：掷骰子游戏，3 戴帽子、7 丢帽子，其他数值走格子：

```rust
match dice_roll {
    3 => add_fancy_hat(),
    7 => remove_fancy_hat(),
    other => move_player(other),   // other 绑定未匹配到的任何值
}
```

如果兜底分支根本不用那个值，就用 **`_`**：匹配任意值但**不绑定**，也不触发"未使用变量"警告——掷到非 3/7 只想重掷或什么都不做：

```rust
_ => reroll(),   // 或 _ => ()：明确"不执行任何代码"
```

通配分支必须放**最后**——match 按顺序比对，放前面会让后续分支永远匹配不到（Rust 会警告）。

## if let：只想处理一种情况的语法糖

只关心 `Some(3)`，其余一概不管？完整 `match` 得写 `_ => ()`，样板略烦：

```rust
let some_u8_value = Some(0u8);
match some_u8_value {
    Some(3) => println!("three"),
    _ => (),
}
```

`if let` 是 `match` 的**语法糖**：等号左边是模式、右边是表达式，只匹配单一模式，忽略其他：

```rust
if let Some(3) = some_u8_value {
    println!("three");
}
```

还能配 `else`——等价于 match 的 `_` 分支：

```rust
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1;   // 非 Quarter 的硬币
}
```

| 手段 | 场景 | 代价 |
|---|---|---|
| `match` | 需处理多个成员、每支要绑定数据 | 必须写全所有分支（穷尽性） |
| `if let` | 只关心单一模式，其余忽略 | 失去穷尽性检查 |

一句话：`match` 追求"一个不漏"，`if let` 追求"只取一瓢"。

## 实践建议与总结

1. **"要么 A 要么 B"的领域概念，先想到 enum 而不是字符串或布尔**：状态机、消息协议、错误类型（第 9 章的 `Result` 就是枚举）都是它的主场；
2. **能进 `Option<T>` 就别发明"魔法空值"**：`Some(x)`/`None` 把"可能有也可能没有"写进类型，让编译器替你做空值检查；需要默认值时，标准库还备着 `unwrap_or`、`map` 等一整套工具；
3. **多分支用 `match` 保穷尽、单分支用 `if let` 求简洁**：每次 match 报"非穷尽"，别嫌烦——那是编译器在替你把关。

本章收尾后，你的"类型工具箱"已具雏形：**struct 组装数据，enum 建模可能性，`Option<T>` 驯服"空"概念，`match`/`if let` 拆开每一种情况并绑定其中的值。** 这类"先让类型描述清楚问题，编译器随即帮你堵住所有漏洞"的思路，正是 Rust 与 C 系语言最根本的气质差异。下一站进入**模块系统**——当程序长大，如何把代码组织成"别人能看懂、自己找得到"的形态。
