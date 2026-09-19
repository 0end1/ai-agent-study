# Rust 学习笔记（19/21）：模式和匹配——你天天在用，却没发现它无处不在

> 对应书源：《Rust 程序设计语言》第 18 章「模式和匹配」

## 开篇：`let x = 5;` 里藏着一个秘密

先看一行你写过不下百次的代码：

```rust
let x = 5;
```

问你这里用了什么语法特性？「变量绑定」，对吧。但书里揭了个底：**`x` 本身就是一个模式**。`let` 语句的完整形态其实是 `let PATTERN = EXPRESSION;`——变量名只是「形式最朴素的模式」，它的含义是「把任何值绑定到变量 `x`，不管值是什么」。

这就像收快递：快递单上的收件格式（模式）和实际包裹（值）对上了，你就能「拆开取件」——把里面的东西一件件拿出来用。Rust 里拆这个「包裹」的动作无处不在，`match`、`if let`、`for`、`let`、甚至函数参数，全都是同一个机制的不同入口。本章就把「包裹拆解说明书」完整过一遍：模式能用在哪、什么模式必须百发百中、以及九种模式语法的正确用法。

## 一、模式的六个使用位置

你已经在不经意间用过很多模式了。先来一张全局地图：

| 使用位置 | 接受的模式类型 | 典型场景 |
| --- | --- | --- |
| `match` 分支 | 前面的分支可反驳 + 最后兜底不可反驳 | 穷尽地分派所有情况 |
| `if let` / `else if let` | 只接受**可反驳**模式 | 只关心某一种情况时的简写 |
| `while let` | 只接受**可反驳**模式 | 「取到就继续」的循环，如弹栈 |
| `for` 循环 | 只接受**不可反驳**模式 | 遍历时解构，如 `(index, value)` |
| `let` 语句 | 只接受**不可反驳**模式 | 解构元组一次创建多个变量 |
| 函数参数 | 只接受**不可反驳**模式 | 参数位置直接解构 |

两个容易忽略的位置：

**`let` 解构元组**。`let (x, y, z) = (1, 2, 3);` 会把 `1`、`2`、`3` 分别绑定到三个变量。但元素数量必须严格对上，否则直接编译错误：

```rust
let (x, y) = (1, 2, 3);
// error[E0308]: mismatched types
// expected a tuple with 3 elements, found one with 2 elements
```

**`while let` 弹栈**是它最经典的用武之地——`stack.pop()` 返回 `Option`，取到 `Some` 就循环，取到 `None` 就自然停止：

```rust
let mut stack = Vec::new();
stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top); // 依次打印 3、2、1
}
```

**函数参数**也能是模式，这是最少人知道的一个：

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}
```

值 `&(3, 5)` 匹配模式 `&(x, y)`，`x` 得到 `3`，`y` 得到 `5`。闭包参数同理。

## 二、可反驳性：模式会不会「失手」

注意上面表格里反复出现的一对词。模式分两种：

- **不可反驳（irrefutable）**：能匹配任何传进来的值，永远不会失败。`let x = 5;` 里的 `x` 就是——任何值它都接得住。
- **可反驳（refutable）**：对某些可能的值会匹配失败。`if let Some(x) = a_value` 里的 `Some(x)` 就是——如果 `a_value` 是 `None`，匹配就失败了。

这条区分决定了各位置「选人标准」，也是新手最常撞上的编译错误来源：

| 位置 | 为什么 |
| --- | --- |
| `let` / `for` / 函数参数 | 只接受不可反驳模式——匹配不上就没有有意义的后续可做 |
| `if let` / `while let` | 只接受可反驳模式——它们的存在意义就是「处理可能失败」 |
| `match` 分支 | 前面的分支用可反驳模式，最后一个兜底分支用不可反驳模式 |

两个方向的反例都值得看一眼。在 `let` 里用可反驳模式：

```rust
let Some(x) = some_option_value;
// error[E0005]: refutable pattern in local binding: `None` not covered
```

编译器的态度很明确：万一进来的是 `None`，这行代码没法有意义地继续，所以不放行。修复方式是改用 `if let`，给代码一个「匹配不上就跳过」的出路。

反过来，在 `if let` 里用不可反驳模式也不行——它会**警告**（不是报错）`irrefutable if-let pattern`：一个永远匹配的条件写在 `if let` 里毫无意义，不如直接写 `let`。

## 三、模式语法工具箱

### 基础四式：字面量、命名变量、或、范围

```rust
match x {
    1 | 2 => println!("one or two"),   // | 表示「或」
    1..=5 => println!("one through five"), // 闭区间范围，比 1|2|3|4|5 省事得多
    _ => println!("anything"),
}
```

范围 `..=` 只允许用于数字和 `char`，因为只有这两种类型编译器能在编译期判断范围是否为空。

### 命名变量的「遮蔽」陷阱

这是本章最值得多看两眼的地方。猜猜下面的代码打印什么？

```rust
let x = Some(5);
let y = 10;

match x {
    Some(50) => println!("Got 50"),
    Some(y) => println!("Matched, y = {:?}", y),
    _ => println!("Default case, x = {:?}", x),
}
println!("at the end: x = {:?}, y = {:?}", x, y);
```

答案是 `Matched, y = 5`——**不是** `10`。`match` 会开启新作用域，第二个分支的 `Some(y)` 引入了一个**新的**变量 `y`（遮蔽外部那个 `10`），它匹配任何 `Some` 里的值，于是绑定了 `x` 里的 `5`。循环结束后，外部的 `y` 依然是 `10`。

如果你的本意是「比较 `Some` 里的值和外部 `y` 是否相等」，答案是**匹配守卫**（后面讲到）：把模式改名为 `Some(n)`，再写 `Some(n) if n == y`。

### 解构：拆结构体、枚举和嵌套

解构结构体时，变量名不必与字段名一致（`let Point { x: a, y: b } = p;`），但更常用的是字段简写 `let Point { x, y } = p;`。还可以把**字面量混进模式**，用来「测一部分、绑另一部分」：

```rust
match p {
    Point { x, y: 0 } => println!("On the x axis at {}", x),
    Point { x: 0, y } => println!("On the y axis at {}", y),
    Point { x, y } => println!("On neither axis: ({}, {})", x, y),
}
```

解构枚举的模式必须对应枚举定义数据的方式：无数据的 `Quit` 只能匹配字面量；类结构体成员 `Move { x, y }` 像结构体；类元组成员 `Write(text)`、`ChangeColor(r, g, b)` 像元组。而且**嵌套随便拆**：

```rust
match msg {
    Message::ChangeColor(Color::Rgb(r, g, b)) => { /* ... */ }
    Message::ChangeColor(Color::Hsv(h, s, v)) => { /* ... */ }
    _ => ()
}
```

两个枚举的嵌套，一个模式里一次拆穿。结构体和元组还能再混搭：`let ((feet, inches), Point {x, y}) = ((3, 10), Point { x: 3, y: -10 });`。

### 忽略值的四种姿势

| 写法 | 效果 | 注意 |
| --- | --- | --- |
| `_` | 匹配但不绑定任何值 | 不会移动所有权 |
| 模式内嵌 `_` | 忽略部分值，如 `(first, _, third, _, fifth)` | 只测形状不看内容 |
| `_x` 前缀 | **仍然绑定**，只是压制未使用警告 | 会移动所有权！ |
| `..` | 忽略剩余所有部分 | 每个模式只能用一次 |

`_` 和 `_x` 的区别很微妙但重要：

```rust
let s = Some(String::from("Hello!"));

if let Some(_s) = s { println!("found a string"); }
println!("{:?}", s); // 错误！s 已被移动进 _s

if let Some(_) = s { println!("found a string"); }
println!("{:?}", s); // 正常。_ 不绑定值，s 没有被移动
```

`..` 的使用必须无歧义，`(first, .., last)` 合法，但 `(.., second, ..)` 会报错 `` `..` can only be used once per tuple or tuple struct pattern ``——编译器无法确定 `second` 到底是哪个位置。

### 匹配守卫：给模式追加 if 条件

模式表达不了「`Some` 里的值小于 5」这种条件，匹配守卫可以：

```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("less than five: {}", x),
    Some(x) => println!("{}", x),
    None => (),
}
```

守卫的条件可以使用模式中创建的变量（甚至外部的变量），这也是修复上面遮蔽陷阱的标准姿势：`Some(n) if n == y` 里的 `y` 是外部的 `y`，因为 `if n == y` 不是模式、不引入新变量。

一个优先级细节：`4 | 5 | 6 if y` 的语义是 **`(4 | 5 | 6) if y`**——守卫作用于整个「或」组合，而不只是最后一个 `6`。

### @ 绑定：一边测试一边抓住

最后一件法宝。想测「id 在 3 到 7 之间」，又想在分支里用上这个值？`@` 运算符两件事一起干：

```rust
match msg {
    Message::Hello { id: id_variable @ 3..=7 } => {
        println!("Found an id in range: {}", id_variable)
    }
    Message::Hello { id: 10..=12 } => {
        println!("Found an id in another range") // 测了范围但拿不到值
    }
    Message::Hello { id } => {
        println!("Found some other id: {}", id) // 拿得到值但没测范围
    }
}
```

`id_variable @ 3..=7` = 「测试它在这个范围 + 把它存进变量」。中间分支只测不存，最后分支只存不测——`@` 让你两个都要。

## 四、实践建议与总结

1. **记住「选人标准」**：`let`/`for`/函数参数要百发百中（不可反驳），`if let`/`while let` 专为「可能失手」而生。看到 E0005 报错，第一反应是「这该用 `if let` 而不是 `let`」。
2. **警惕 `match` 里的变量遮蔽**：分支模式里的名字是新变量，不是外部同名变量。想比较外部值，用匹配守卫 `Some(n) if n == y`。
3. **忽略值想清楚要不要「接住」**：`_` 不绑定不移动所有权，`_x` 会真的把值拿走。碰到所有权被意外移走的编译错误，检查是不是用了 `_x`。

**总结**：模式是 Rust 中区分数据形状的通用语法，`match` 靠它实现穷尽性检查，`let` 和函数参数靠它实现优雅解构。九种语法——字面量、命名变量、`|`、`..=`、解构、`_`/`_x`/`..`、匹配守卫、`@` 绑定——组合起来能写出既简洁又安全的分派逻辑。下一章我们进入「高级特征」：unsafe、trait 高阶玩法等特定场景武器。
