# Rust 学习笔记（6/21）：结构体——给数据起名字的艺术

> 本系列基于官方《Rust 程序设计语言》（The Rust Programming Language，TRPL）逐章学习。前两篇啃完了 Rust 最难的所有权与借用，本篇回到熟悉的领域：**如何把散落的数据组织成有意义的整体**——`struct`。

## 开篇：一份没有表头的报表

想象财务同事甩给你一份报表：`(30, 50, 2, true, "zhang@example.com", "zhangsan", 1)`。元组诚实地保存了所有数据，但没人看得懂第 0 位是宽还是高、第 4 位是邮箱还是用户名——所有信息都**依赖顺序**，阅读全靠猜。

struct 就是"给列加表头"：把相关的值**命名并打包**成一个自定义类型。如果你熟悉面向对象语言，struct 就是"对象里的数据属性"那一半。这一章会看到三种结构体变体、一个经典的"重构三连"，以及把函数"贴"到类型上的方法语法。

## 定义与实例化：先画模板，再填数据

用 `struct` 关键字定义一个类型模板，大括号里列**字段**（field）的名字和类型：

```rust
struct User {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```

创建**实例**时，用 `结构体名 { 字段: 值 }` 填数据——注意字段顺序无需与声明一致：

```rust
let user1 = User {
    email: String::from("zhang@example.com"),
    username: String::from("zhangsan"),
    active: true,
    sign_in_count: 1,
};
```

用**点号**读写字段：`user1.email` 取值；实例整体声明为 `mut` 后，`user1.email = ...` 改值。Rust **不允许只把某个字段标成可变**——要么整个实例可变，要么都不可变。

结构体定义是"通用模板"，实例是"填入特定数据后产生的类型值"。函数最后一个表达式若构造结构体，会被隐式返回：

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email: email,        // 啰嗦：字段名与变量名重复
        username: username,
        active: true,
        sign_in_count: 1,
    }
}
```

### 两个省事的语法糖

**字段初始化简写**：当参数名与字段名完全相同时，只写一次名字即可——`email` 等价于 `email: email`：

```rust
User { email, username, active: true, sign_in_count: 1 }
```

**结构体更新语法**：想"复制旧实例的大部分字段、只改一两个"，用 `..user1` 收尾，其余字段自动继承：

```rust
let user2 = User {
    email: String::from("li@example.com"),
    ..user1          // username/active/sign_in_count 从 user1 取；必须放最后
};
```

> 注意：更新语法像 `=` 赋值一样会**移动数据**。上面把 `user1.username`（String，堆类型）移进了 `user2`，之后 `user1` 整个就不可再用了。若 `user2` 把 `email`、`username` 都换成新 String，只继承 `active`、`sign_in_count`（两者都是 `Copy` 类型），那 `user1` 仍可继续使用。

## 三种结构体：字段命名度各不相同

| 类型 | 语法 | 字段名 | 典型用途 |
|---|---|---|---|
| 普通结构体 | `struct User { email: String, ... }` | ✅ 有名字 | 领域模型（用户、订单等） |
| 元组结构体 | `struct Color(i32, i32, i32);` | ❌ 只有类型 | 给整个元组命名、与其他元组区分 |
| 类单元结构体 | `struct AlwaysEqual;` | —（无字段） | 只需类型概念、无需存数据（trait 载体） |

**元组结构体**：有类型名无字段名。`Color(i32,i32,i32)` 与 `Point(i32,i32,i32)` 是**两个不同类型**——接收 `Color` 的函数绝不能传 `Point`，即便底层都是三个 `i32`。这就是"新类型"的价值：编译器帮你区分本不该混淆的概念。访问方式同元组：解构、或用 `.0`/`.1` 索引。

**类单元结构体**：零字段，类型即一切。声明只要 `struct AlwaysEqual;`（分号结束），实例化就是 `let subject = AlwaysEqual;`。常用于"我要在这个类型上实现 trait，但不需要存数据"——比如给测试实现一个"永远等于任何值"的类型。

**所有权小坑**：结构体通常存**拥有所有权的 `String`** 而非 `&str` 引用，这样"结构体有效，数据就有效"。想在结构体里存引用？编译器会要你写生命周期标注（E0106），那是第 10 章的活，现在先老老实实存 `String`。

## 经典重构三连：Rectangle 面积计算

书里用一个求矩形面积的例子，展示了三种写法如何一步步"让意义浮现"：

| 方案 | 代码特征 | 问题 |
|---|---|---|
| 两个独立变量 | `area(width1, height1)` | 宽高明明相关，函数却收两个孤立参数，看不出"它们属于一个矩形" |
| 元组 | `area((30, 50))`，`dimensions.0 * dimensions.1` | 只传一个参数了，但要靠索引 `0`/`1` 记位置，换个人看代码全靠脑补 |
| 结构体 | `area(&rect1)`，`rectangle.width * rectangle.height` | 数据命名、整体打包，签名直接表达意图 |

结构体版的关键在借用：`fn area(rectangle: &Rectangle) -> u32`——借用而非取得所有权，`main` 才能继续用 `rect1`。**"方法签名表达意图 + 借用保持所有权"，是 Rust 组织代码的日常姿势。**

### 打印结构体：Display 与 Debug

直接 `println!("rect1 is {}", rect1)` 会报错：`Rectangle doesn't implement std::fmt::Display`。原因很合理——基本类型怎么显示给用户是唯一的（数字就是数字），但结构体的显示方式**有无限种可能**（要不要逗号？打不打大括号？显示哪些字段？），Rust 不猜你的心思，所以默认不实现面向用户的 `Display`。

调试时想看看字段值？用 `{:?}` 走 **`Debug`** 格式，但得先显式声明支持——在结构体定义前加属性：

```rust
#[derive(Debug)]                    // 派生宏：让编译器自动实现 Debug trait
struct Rectangle {
    width: u32,
    height: u32,
}

println!("rect1 is {:?}", rect1);   // rect1 is Rectangle { width: 30, height: 50 }
println!("rect1 is {:#?}", rect1);  // {:#?} 漂亮打印，多行缩进，大结构体更易读
```

还有 `dbg!` 宏：它接收表达式的**所有权**，打印"文件:行号 + 表达式 + 结果值"，再把所有权返回——想看又不愿让它吞掉结构体，就传引用 `dbg!(&rect1)`。输出走 stderr（标准错误）而非 stdout：

```
[src/main.rs:10] 30 * scale = 60
[src/main.rs:14] &rect1 = Rectangle { width: 60, height: 50 }
```

## 方法语法：把行为"装"进类型

`area` 这函数只服务 `Rectangle`，与其在文件里到处找，不如把它定义进 `Rectangle` 的上下文——这就是**方法（method）**。用 `impl` 块（implementation 的缩写）包裹：

```rust
impl Rectangle {
    fn area(&self) -> u32 {          // &self 是 self: &Self 的缩写
        self.width * self.height
    }
}
// main 里调用：rect1.area()
```

方法本质是函数，区别在于**定义在类型的上下文中**，且第一个参数永远是 `self`（代表调用它的实例）。`self` 前面加不加 `&` 决定了借用方式：`&self` 只读借用（最常见）、`&mut self` 要改字段、`self` 拿走所有权（少见，用于"把自己变成别的实例"的场景）。

**Rust 没有 `->` 运算符**：C++ 里对象用 `.`、指针用 `->`，Rust 用**自动引用/解引用**统一了——调用 `object.something()` 时编译器自动补 `&`、`&mut` 或 `*` 来匹配方法签名。所以 `p1.distance(&p2)` 和 `(&p1).distance(&p2)` 完全等价，你永远只写简洁的那个。

方法可以带更多参数（第二个及以后的参数和普通函数一样）：实现一个 `can_hold`，判断 `self` 能否完全包含另一个矩形：

```rust
impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
// rect1.can_hold(&rect2)  // 传 &rect2 不可变借用，main 保留 rect2 所有权
```

方法还能与字段同名——`rect1.width`（无括号，字段）与 `rect1.width()`（有括号，方法），Rust 靠括号区分。同名方法常当 **getter** 用：字段私有、方法公有，对外只暴露只读访问（第 7 章讲可见性）。

### 关联函数与多个 impl 块

`impl` 块里的所有函数统称**关联函数**。不以 `self` 开头、不作用于某个实例的，常用来当**构造函数**——你已经见过的 `String::from` 就是关联函数：

```rust
impl Rectangle {
    fn square(size: u32) -> Self {    // 构造函数惯例
        Self { width: size, height: size }
    }
}
// 调用用 ::（类型命名空间）：let sq = Rectangle::square(3);
```

`::` 语法用于关联函数与模块创建的命名空间。另外，每个结构体可以有**多个 `impl` 块**——现在看不出好处，等第 10 章泛型与 trait 时就会见到"按 trait 拆 impl 块"的实用场景。

## 实践建议与总结

1. **优先普通结构体，元组结构体只用于"要给元组起名"**：字段名是代码自文档化的第一道防线；`Color` vs `Point` 的例子说明"同构不同名"本身就是一种类型安全；
2. **打印自定义类型一律先 `#[derive(Debug)]`**：这是最常用的派生 trait；想给用户看漂亮的 `Display` 输出，得自己 `impl`（第 10 章学 trait 后即可）；
3. **行为跟随数据**：函数如果只服务一种类型，就写成方法放进 `impl` 块；`&self` 优先、需要改动再升级 `&mut self`，别随手 `self` 拿走所有权。

这一章没有新的"内存哲学"，只有组织学：**struct 把字段命名打包成领域类型（模板→实例），方法把行为收编进类型（impl→self），关联函数提供构造入口（类型::函数）。** 至此你的自定义类型已能承载真实业务，而 Rust 的编译期类型检查——那个 "Color 不是 Point" 的看门人——开始为你把关。下一章引入更灵活的**枚举**：一个值只能属于某个"成员"集合，配合 `match` 模式匹配，将开启 Rust 编程风格的下一个纪元。
