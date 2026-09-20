# Rust 学习笔记（20/21）：高级特征——unsafe、关联类型与宏的进阶工具箱

> 对应书源：《Rust 程序设计语言》第 19 章「高级特征」

## 开篇：Rust 的「地下室工具房」

前十八章学的东西覆盖了 Rust 的绝大多数日常，但总有一些时刻，你会翻进「地下室」找工具：直接跟操作系统打交道、给 `Vec` 实现 `Display`、看懂别人代码里诡异的 `<Dog as Animal>::baby_name()`。本章就是这间地下室——书里也明说了，这些功能「很少会碰到」，但被设计成了一份**遇到未知内容时的参考手册**。

用攀岩打个比方：安全 Rust 是有护栏的栈道，99% 的风景都能到达；`unsafe` 则是撤掉护栏的岩壁——有些山（比如与 C 代码交互）根本没有栈道可走，你必须自己系好绳子。本章四大板块：不安全 Rust、高级 trait、高级类型、宏。先看地图：

| 板块 | 核心内容 | 一句话定位 |
| --- | --- | --- |
| 不安全 Rust | `unsafe` 五种超能力 | 绕过编译器保护，但护栏只是变小、没被拆掉 |
| 高级 trait | 关联类型、默认泛型参数、完全限定语法、父 trait、newtype | 看懂标准库和第三方库的进阶签名 |
| 高级类型 | 类型别名、never type、动态大小类型 | 理解 `!`、`str` 和 `Sized` 这些「熟面孔」的另一面 |
| 宏 | 声明宏 `macro_rules!`、三种过程宏 | 为写代码而写代码（元编程） |

## 一、不安全 Rust：手动挡模式

`unsafe` 不是「关闭安全检查」，这个澄清很重要：**借用检查器照常工作，引用照常被检查**。它只是额外解锁了五种编译器无法验证内存安全的操作：

| 超能力 | 典型场景 |
| --- | --- |
| 解引用裸指针（`*const T` / `*mut T`） | 调用 C 接口、构建借用检查器理解不了的抽象 |
| 调用不安全函数或方法 | 函数文档里标明「调用者需满足某契约」 |
| 访问或修改可变静态变量（`static mut`） | 全局可变状态（有数据竞争风险，优先用第16章的并发原语） |
| 实现不安全 trait（`unsafe trait` + `unsafe impl`） | 如手动标记裸指针类型为 `Send`/`Sync` |
| 访问 union 字段 | 与 C 的联合体交互（Rust 无法保证当前存的是什么类型） |

裸指针有个容易忽略的细节：**创建裸指针不需要 `unsafe`，解引用才需要**。`let r1 = &num as *const i32;` 在安全代码里完全合法——创建指针本身无害，访问指向的值才可能出事。而且裸指针允许同一地址同时存在 `*const` 和 `*mut`，这在引用世界里是借用规则禁止的，也就埋下了数据竞争的种子。

本章最漂亮的例子是 `split_at_mut`：把一个 slice 从中间切成两半，各返回一个可变 slice。纯安全 Rust 写不出来（E0499：编译器只看到「同一个 slice 被可变借用两次」，理解不了「两个不重叠的区间」），但加上裸指针和 `unsafe` 块就能实现——而函数签名**不带** `unsafe`，调用者毫无感知。这就是「**不安全代码的安全抽象**」：`unsafe` 块保持尽可能小，风险被封装在 Reviewed 过的内部。

另一个高频场景是 FFI（外部函数接口），让 Rust 直接调 C 标准库：

```rust
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    unsafe {
        println!("Absolute value of -3 according to C: {}", abs(-3));
    }
}
```

`extern` 块里声明的函数**永远是不安全的**——其他语言不遵守 Rust 的规则，安全责任全在调用者身上。反过来，`#[no_mangle] pub extern "C" fn` 也能把 Rust 函数暴露给 C（这种 extern 不需要 `unsafe`）。

## 二、高级 trait：看懂标准库的进阶签名

### 关联类型 vs 泛型

`Iterator` trait 的定义用了一个占位类型：

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

为什么不直接写 `pub trait Iterator<T>`？区别在于：用泛型的话，同一个 `Counter` 可以实现 `Iterator<u32>`、`Iterator<String>` 等多个版本，每次调用 `next` 都得加类型标注来消歧义；用关联类型则**一个类型只能实现一次**，`Item` 选定 `u32` 后就定死，调用时无需任何标注。一句话总结：**泛型是「每次使用时选类型」，关联类型是「实现时选一次」**。

### 默认泛型参数与运算符重载

`Add` trait 的定义是 `trait Add<RHS=Self>`——`RHS=Self` 就是默认类型参数，不指定时右操作数就是自身。想实现 `Millimeters + Meters`？`impl Add<Meters> for Millimeters` 显式覆盖默认值，单位换算在 `add` 方法里完成。这个设计还有个妙用：给现有 trait 加新的类型参数时提供默认值，可以**不破坏任何已有实现**地扩展 trait。

### 完全限定语法

两个 trait 有同名方法，类型自己也有同名方法时，`person.fly()` 默认调用类型直接实现的方法；想调 trait 的用 `Pilot::fly(&person)`。但遇到**没有 `self` 参数的关联函数**，得动用终极形态：

```rust
<Dog as Animal>::baby_name()  // 明确指定：Dog 的 Animal 实现里的 baby_name
```

通式是 `<Type as Trait>::function(receiver_if_method, next_arg, ...)`。平时用不上，报 `E0283: type annotations required` 时它就是解药。

### 父 trait 与 newtype 模式

`trait OutlinePrint: fmt::Display` 声明「实现我之前必须先实现 `Display`」，于是 trait 方法内可以自由使用 `self.to_string()`。没实现 `Display` 就直接实现 `OutlinePrint`？E0277 拦下。

newtype 模式（用元组结构体包一层）则绕过孤儿规则——`Display` 和 `Vec` 都不归你管，但 `struct Wrapper(Vec<String>)` 归你管，给 `Wrapper` 实现 `Display` 即可，且**零运行时开销**（封装在编译期就被省略）。想透传内部类型的所有方法就实现 `Deref`，想收敛行为就手写需要的几个方法。

## 三、高级类型：`!`、别名与动态大小

### 类型别名：同义词不是新类型

`type Kilometers = i32;` 里的 `Kilometers` 只是 `i32` 的**另一个名字**，可以互相加、可以传给收 `i32` 的函数——和 `Millimeters` 那种独立新类型完全不同（后者能拦住单位混用）。别名的主战场是**消除重复**：

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;  // 一长串类型一个名字搞定

type Result<T> = std::result::Result<T, std::io::Error>;  // std::io 就这么干
```

`std::io::Result<T>` 让 `Write` trait 里满屏的 `Result<..., Error>` 变成 `Result<usize>`，且不损失 `?` 等语法支持——因为它本质还是那个 `Result`。

### never type：从不返回的 `!`

`!` 是一个没有任何值的类型，用于「发散函数」：`fn bar() -> !` 表示 `bar` 永不返回。它最有意思的用法藏在 `continue` 里：

```rust
let guess: u32 = match guess.trim().parse() {
    Ok(num) => num,
    Err(_) => continue,  // 一个分支 u32，一个分支 continue，为什么能编译？
};
```

因为 `continue` 的值就是 `!`，而 **`!` 可以强转为任何类型**——`Err` 分支实际不会产生值（控制权交回循环），Rust 于是推断整个 match 是 `u32`。`panic!`（如 `Option::unwrap` 的 `None` 分支）和无限 `loop` 也都是 `!`。

### 动态大小类型（DST）

`str`（不是 `&str`！）是一个 DST——长度运行时才确定，所以根本没法创建 `str` 类型的变量。解决办法你天天在用：把 DST 放到**指针后面**。`&str` 实际是「地址 + 长度」两个值，编译期大小恒为 `usize` 的两倍。这条**黄金规则**还解释了 trait 对象为什么必须写成 `&dyn Trait`、`Box<dyn Trait>`：每个 trait 也是 DST。

配套的 `Sized` trait 会被**隐式加到每个泛型参数上**（`fn generic<T>(t: T)` 实为 `fn generic<T: Sized>`）；用 `T: ?Sized`（只能用于 `Sized`）可放宽，此时参数要改成 `&T` 这类指针形式。

## 四、高级函数与闭包

函数可以当参数传——类型是 **`fn`（小写，函数指针）**，注意不是闭包 trait `Fn`：

```rust
fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}

do_twice(add_one, 5)  // 12
```

函数指针实现了全部三个闭包 trait，所以收闭包的函数也能收函数指针；反之，**只收 `fn` 不收闭包**的场景主要是与 C 代码交互——C 没有闭包。另外 `map(ToString::to_string)`、`map(Status::Value)`（元组结构体构造器也是函数指针！）这类写法之所以可行，靠的就是函数指针与完全限定语法。

返回闭包则不能直接写返回类型（闭包是 trait，大小未知），标准姿势是 trait 对象：

```rust
fn returns_closure() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

## 五、宏：为写代码而写代码

宏 = 元编程，与函数的关键区别一张表看清：

| 维度 | 函数 | 宏 |
| --- | --- | --- |
| 参数数量/类型 | 必须声明并检查 | 可接受任意数量（`println!`、`vec!`） |
| 展开时机 | 运行时调用 | **编译前展开**，可在类型上实现 trait |
| 可读性/维护成本 | 较低 | 更复杂（写生成代码的代码） |
| 定义/调用顺序 | 任意 | 调用前必须先定义或引入作用域 |

声明宏 `macro_rules!` 本质是「对 Rust 代码做 `match`」——匹配的是**代码结构**而非值。`vec!` 的简化版一窥究竟：

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $( temp_vec.push($x); )*
            temp_vec
        }
    };
}
```

`$x:expr` 捕获任意表达式，`$()*` 按匹配次数重复生成代码。`vec![1, 2, 3]` 展开后就是三行 `push`。

过程宏更像函数：吃进 `TokenStream`（token 序列），吐出 `TokenStream`，必须放在独立的 `proc-macro = true` crate 里。三种形态：

| 类型 | 触发方式 | 例子 |
| --- | --- | --- |
| 自定义 derive | `#[derive(HelloMacro)]` | 自动生成 trait 实现 |
| 类属性宏 | `#[route(GET, "/")]` | Web 框架的路由标注（可用于函数） |
| 类函数宏 | `sql!(SELECT * FROM posts ...)` | 解析并校验 SQL 语法 |

derive 宏的标准流水线：`syn` 把 `TokenStream` 解析成语法树（拿到 `ast.ident` 即结构体名）→ `quote!` 用模板（`#name` 占位符）生成 `impl HelloMacro for #name` 代码 → `into()` 转回 `TokenStream` 交给编译器。`#[derive(HelloMacro)] struct Pancakes;` 就能自动打印 `Hello, Macro! My name is Pancakes!`——这正是 Rust 没有反射却能做到「运行时打印类型名」的方式。

## 六、实践建议与总结

1. **`unsafe` 的三条纪律**：块保持尽可能小；优先封装成安全 API 对外暴露；可变静态变量能不用就不用，用第16章的并发原语替代。
2. **区分「新类型」和「别名」**：要类型安全拦住单位混用，用 newtype；只为少打字，用 `type` 别名。
3. **见到看不懂的签名别慌**：`T: ?Sized`、`Add<RHS=Self>`、`<Dog as Animal>::fn`、`-> Box<dyn Fn>` 都出自本章——把它当参考手册，下次在报错信息或他人代码里碰到，回来翻即可。

**总结**：本章把 Rust 的「地下室」逛了一遍——`unsafe` 撤掉护栏换取与硬件和 C 交互的能力；关联类型、默认泛型参数、完全限定语法、父 trait 和 newtype 模式撑起了标准库的进阶签名；`!` 与 DST 揭示了类型系统的两个隐藏角落；宏则让 Rust 拥有了编译期生成代码的元编程能力。下一章是最后一站：把全书所学倾注到一个多线程 Web 服务器项目里。
