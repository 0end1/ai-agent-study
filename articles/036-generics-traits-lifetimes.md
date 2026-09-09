# Rust 学习笔记（11/21）：泛型、trait 与生命周期——把"抽象"交给编译器把关

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。写代码到一定程度，"复制粘贴然后改改类型"的冲动就来了：两个函数体一模一样，只是一个是 `i32` 一个是 `char`，怎么办？本章给你三件武器——**泛型**让代码适配多种类型，**trait** 声明"我要求你具备什么行为"，**生命周期**约束"引用能活多久"。三者配合，把"灵活"和"安全"同时做到，且**全部在编译期完成，零运行时开销**。

## 开篇：给函数装上"可调模具"

想象一家糖果厂：同一台注糖机，换上不同的模具，就能压出小熊、爱心、星星形状的糖。模具是"占位"的形状，具体是哪种动物，由装上去那一刻决定。Rust 的泛型（`generics`）就是这台注糖机——**用 `T` 占住类型的位置，编译时再由编译器按实际类型"印"出专用版本**。

第 6 章的 `Option<T>`、第 8 章的 `Vec<T>`、`HashMap<K, V>`、第 9 章的 `Result<T, E>`，全都是泛型——你一直在用，这一章学习自己造。

## 泛型：把重复代码"参数化"

### 从提取函数到提取类型

找最大值的代码写两遍很蠢，于是提取成函数；但只提取函数还不够——找 `i32` slice 最大值和找 `char` slice 最大值的两个函数**函数体一字不差**，只差签名类型。此时把类型也"参数化"：在函数名后加尖括号声明类型参数 `T`，参数和返回值全换成 `T`。

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

第一次编译会报 `E0369`：二进制运算 `>` 不能用于类型 `T`。道理很直观——**不是所有类型都能比较大小**，编译器必须知道 `T` 具备"可比大小"这种能力，而这正是 trait 的用武之地（下文修复）。

### 结构体与枚举中的泛型

结构体也可泛型：`struct Point<T> { x: T, y: T }` 意味着 `x`、`y` **必须是同一种类型**，`Point { x: 5, y: 4.0 }` 会触发 `E0308`。想要两者不同，就引入两个参数：`struct Point<T, U> { x: T, y: U }`。

方法的 `impl` 也要跟着泛型走：

```rust
impl<T> Point<T> {                 // impl 后声明 T，让 Point<T> 是"泛型"
    fn x(&self) -> &T { &self.x }
}
impl Point<f32> {                  // 只给具体类型 Point<f32> 加方法
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

注意 `impl<T> Point<T>` 与 `impl Point<f32>` 的区别：前者对所有 `Point` 生效，后者只对 `f32` 版本生效——`distance_from_origin` 用到了浮点运算，只有 `f32` 有。此外，方法的泛型可以独立于结构体，例如 `mixup<V, W>(self, other: Point<V, W>) -> Point<T, W>`，把两个不同类型的点"杂交"成一个新点。

### 泛型真的零开销吗？——单态化

Rust 通过**单态化（monomorphization）**保证性能：编译时编译器找到泛型代码每一处调用，按具体类型生成专用版本。`Option::Some(5)` 和 `Option::Some(5.0)` 会被展开成 `Option_i32` 和 `Option_f64` 两个具体枚举。所以运行时的效率**等同手写每个具体版本**，没有虚函数调用、没有运行时类型判断——这也是 Rust 泛型"既灵活又快"的秘诀。

## trait：声明"你必须有这个本事"

`trait` 把一组方法签名打包成一个**行为契约**（类似其他语言的接口）。定义 `Summary` trait，只写签名不写实现：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

之后给任意类型"签约"：

```rust
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
impl Summary for Tweet {
    fn summarize(&self) -> String { format!("{}: {}", self.username, self.content) }
}
```

**孤儿规则（orphan rule）**：只有当 trait 或类型至少有一个定义在本地 crate 时才能为它实现 trait。你不能为标准库的 `Vec<T>` 实现标准库的 `Display`——否则两个 crate 各自实现同一对组合，编译器不知听谁的。

trait 还能提供**默认实现**，实现方可以留空 `impl Summary for NewsArticle {}` 直接继承；默认方法还可以调用**没有默认实现**的方法（如 `summarize` 的默认实现调用必须手写的 `summarize_author`），实现方只写最小集即可获得整套功能。注意：无法从重载的实现中反向调用默认实现。

### 作参数：impl Trait vs Trait Bound

函数想接受"任何实现了 `Summary` 的类型"，有两种等价写法：

```rust
pub fn notify(item: impl Summary) {}      // 语法糖，直观简短
pub fn notify<T: Summary>(item: T) {}     // trait bound，更灵活
```

区别在使用场景：若两个参数**允许不同类型**，`impl Summary` 更顺眼；若**强制同一类型**（两个参数必须是同一种实现了该 trait 的类型），只能用 trait bound `T: Summary`。多个约束用 `+`：`T: Summary + Display`；约束太多时挪到 `where` 从句让签名可读：

```rust
fn some_function<T, U>(t: T, u: U) -> i32
    where T: Display + Clone, U: Clone + Debug
{
    // ...
}
```

返回值也能用 `impl Summary`——调用方只知道"返回了能 summarize 的东西"，不用写出（可能长得吓人的）具体类型。但**这只适用于返回单一类型**：若按条件有时返回 `NewsArticle`、有时返回 `Tweet`，则编译不过（要等第 17 章的 trait 对象）。

### 修复 largest：加对约束

回到报 `E0369` 的 `largest`。`>` 运算符来自 `std::cmp::PartialOrd` trait，补上它：`fn largest<T: PartialOrd>(list: &[T]) -> T`。但新错误 `E0508 cannot move out of type [T]` 又冒出来——把 `list[0]` 移出来要求 `T` 可拷贝，于是再加 `Copy`：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T { /* ... */ }
```

不想限制 `Copy`？改用 `Clone`（逐个克隆，堆上数据有分配开销），或干脆返回引用 `&T`——零拷贝零堆分配，留作思考题。

**有条件地实现方法**：`impl<T: Display + PartialOrd> Pair<T> { fn cmp_display(&self) {...} }` 表示只有同时满足两个约束的 `Pair<T>` 才有 `cmp_display`。更进一步，"对一切满足某约束的类型实现某 trait"叫 **blanket implementation**——标准库靠它为所有 `Display` 类型实现了 `ToString`，于是 `3.to_string()` 直接可用。

## 生命周期：给引用划"保质期"

生命周期（lifetimes）其实也是一种泛型——只是它抽象的**不是类型，而是引用的有效范围**。目标是杜绝**悬垂引用**（引用一块已释放的内存）。

看这个经典错误：内部作用域里 `r = &x`，作用域一结束 `x` 被释放，外部还想打印 `r`，编译器报 `E0597: x does not live long enough`。借用检查器会把作用域比作带名字的区间——`r` 的区间 `'a` 比 `x` 的 `'b` 长，被引用者活不过引用者，拒绝编译。反过来 `x` 先声明、`r` 后声明，`'b` 长于 `'a`，放行。

### 函数的泛型生命周期

写"返回两个字符串 slice 中较长者"的函数，直接报 `E0106: missing lifetime specifier`——返回的是借用，Rust 不知道它来自 `x` 还是 `y`，也就无法验证安全。解法是声明泛型生命周期 `'a`，把三者的关系说清楚：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

含义：`x`、`y`、返回值共享生命周期 `'a`，具体值取 `x`、`y` 作用域重叠（较小）的那段。于是把 `result` 用到大括号外、而 `string2` 已提前释放的代码会被 `E0597` 拦下——即使人眼知道结果是 `string1` 更长，Rust 也按你声明的保守约束来把关。

语法速记：生命周期名以撇号开头小写，写在 `&` 后——`&i32`、`&'a i32`、`&'a mut i32`。**标注不改变任何引用的实际存活时间**，只是向借用检查器描述引用间的关联。

还有两条硬规则：返回的引用必须和某个参数生命周期关联（返回函数内部创建的值的引用必然悬垂，应改返回有所有权的类型）；若函数总是返回第一个参数，`y` 无需标注：`fn longest<'a>(x: &'a str, y: &str) -> &'a str`。

### 生命周期省略：编译器替你猜

早期 Rust 每个引用都要写生命周期，太啰嗦。Rust 团队把常见模式编码成**三条省略规则**，满足时无需标注：

| 规则 | 内容 |
|---|---|
| ① 输入规则 | 每个引用参数都有自己的生命周期参数（`x: &str, y: &str` → `'a`、`'b`） |
| ② 单输入规则 | 只有一个输入生命周期时，它赋给所有输出 |
| ③ 方法规则 | 有 `&self`/`&mut self` 参数时，所有输出生命周期取 `self` 的 |

对照一下：`fn first_word(s: &str) -> &str` 一个输入参数，规则②自动补全，无需标注；`longest(x: &str, y: &str) -> &str` 两个输入，规则②③都不适用，编译器算不出来 → 必须手写 `'a`。这就是省略规则"只覆盖可预测场景"的设计。

### 结构体与 'static

含引用的结构体必须标注每个引用字段：`struct ImportantExcerpt<'a> { part: &'a str }`，意思是实例不能活得比 `part` 引用的数据更久。为它写 `impl<'a> ImportantExcerpt<'a>`，字段生命周期声明在 impl 后；方法本身常靠规则③自动省略。

特殊的 `'static` 生命周期表示**整个程序期间都有效**——所有字符串字面量都是 `&'static str`（直接存进二进制）。但错误信息里建议你加 `'static` 时要三思：多数情况其实是悬垂引用或生命周期不匹配，别用 `'static` 掩盖真问题。

## 三合一：泛型 + trait + 生命周期同台

把它们写进同一个签名并不复杂——生命周期 `'a` 和泛型 `T` 都放尖括号里，trait 约束进 `where`：

```rust
fn longest_with_an_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
    where T: Display
{
    println!("Announcement! {}", ann);
    if x.len() > y.len() { x } else { y }
}
```

## 实践建议与总结

1. **看到重复先想"提取"**：先提取函数，再把类型差异也提取成泛型——识别重复代码的肌肉会越练越灵敏。
2. **约束往小处写**：`T: PartialOrd + Copy` 一次只加当前需要的 bound；能用 `impl Trait` 的地方别硬写复杂 trait bound，参数多、约束多就用 `where` 收尾。
3. **生命周期"能不写就不写，要写就写最小关联"**：单参数函数编译器自动处理；多参数才需要 `'a`，且只标注真正与返回有关联的参数。

本章结束时，泛型给你**类型层面的多态**，trait 给它套上**行为约束**，生命周期保证**引用永远有效**——而这一切都在编译时完成，不留运行时痕迹，这正是 Rust 敢说"既抽象又高效"的底气。第 17 章会登场另一种 trait 用法（trait 对象），第 19 章再深入生命周期的高级场景；下一章先学如何用测试确保代码真的按预期工作。
