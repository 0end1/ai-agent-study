# Rust 学习笔记（14/21）：迭代器与闭包——Rust 的"零成本抽象"是怎么做到的

> 本系列基于官方《Rust 程序设计语言》（TRPL）逐章学习。前面我们一直在"写得对"的层面打磨：所有权、生命周期、测试。这一章转向"写得巧"——Rust 从函数式语言借来的两大件：**闭包**和**迭代器**。它们让代码看起来更高级，却几乎不付运行时性能代价。这一章还顺手把上一章的 `minigrep` 用迭代器重写了一遍，作为"抽象到底值不值"的实证。

## 开篇：一个健身 App 的两难

设想你在一个生成定制健身计划的初创公司，后端用 Rust 写。核心算法要考虑年龄、BMI、喜好、近期活动量……计算一次大约两秒，所以我们用一个 `simulated_expensive_calculation` 函数模拟它（打印 `calculating slowly...`、睡 2 秒、返回传入的数字）。

业务逻辑 `generate_workout(intensity, random_number)` 是：低强度（`intensity < 25`）建议做俯卧撑和仰卧起坐；高强度时随机数为 3 就休息，否则跑步若干分钟。慢计算因此被调用三处——第一个 `if` 分支调了**两次**（用户白等一倍），`else` 内的 `if` 分支**不该调**，最后再调**一次**。

第一次重构是把结果提取到变量 `expensive_result`：调用统一了，代价却是**所有情况都得先等两秒**，包括不需要结果的分支。我们要的是——**在一处定义代码，只在需要时才执行**。这正是闭包登场的地方：像把菜谱写进信封揣兜里，写下来不算做菜，饿了拆开照做，且只做一次。

## 闭包：能捕获环境的匿名函数

闭包是"可存进变量、或作为参数传给其他函数的匿名函数"。语法从一对**竖线**开始：`|num| { ... }`，多参数用逗号分隔，只有一行时大括号可省，体最后一行即返回值。这条 `let` 存的是**定义**而非调用结果——代码躺在变量里，等你用 `expensive_closure(intensity)` 触发。

### 类型推断与标注

闭包不要求标注参数与返回类型：它存在变量里、匿名、不供库用户调用，且通常很短、上下文很窄，编译器能可靠推断，强制标注只会重复编译器已知的信息。想写也行，四种等价写法：

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

代价是：**闭包的具体类型会被第一次调用锁定**。下面这段先传 `String` 再传整数，编译器会报 `E0308: mismatched types`，因为 `x` 与返回值已被推断并锁定为 `String`：

```rust
let example_closure = |x| x;
let s = example_closure(String::from("hello"));
let n = example_closure(5);   // error[E0308]: expected String, found integer
```

### 用 Fn trait 把闭包装进结构体

要解决"同一结果被算两次"，除了到处存变量，还有更优雅的方案：**用一个结构体存放闭包并缓存结果**——这个模式叫 *memoization* 或**惰性求值**（lazy evaluation）。难点在于结构体字段必须有类型，而每个闭包实例都有自己**独有的匿名类型**（即使签名相同也互不相同），所以要用泛型 + trait bound。标准库提供的三个 `Fn` 系列 trait 中，这里用 `Fn`：

```rust
struct Cacher<T>
where
    T: Fn(u32) -> u32,
{
    calculation: T,
    value: Option<u32>,
}

impl<T> Cacher<T>
where
    T: Fn(u32) -> u32,
{
    fn new(calculation: T) -> Cacher<T> {
        Cacher { calculation, value: None }
    }

    fn value(&mut self, arg: u32) -> u32 {
        match self.value {
            Some(v) => v,
            None => {
                let v = (self.calculation)(arg);
                self.value = Some(v);
                v
            }
        }
    }
}
```

字段私有，是为了让 `Cacher` 自己管理缓存而不被外部乱改。调用方把闭包交给 `Cacher::new`，之后一律走 `value(intensity)`：有缓存直接返回，没有才执行并存入。于是慢计算**最多只跑一次**，`generate_workout` 得以专注业务逻辑。（顺带一提：函数也实现了这三个 `Fn` trait，不需要捕获环境时直接传函数即可。）

但这个 `Cacher` 有两个限制，导致难以复用：

| 限制 | 现象 | 改进方向 |
|---|---|---|
| 假设"同一 arg 总返回同一值" | 测试 `call_with_different_values` 失败：先 `value(1)` 再 `value(2)`，缓存里是 `Some(1)`，断言报 `left: 1, right: 2` | 用 `HashMap` 以 `arg` 为 key 缓存多组结果 |
| 被写死为 `u32 -> u32` | 想缓存 `&str -> usize` 的闭包就不能用 | 引入更多泛型参数提高灵活性 |

### 捕获环境与三种 Fn trait

闭包还有一个函数没有的能力：**捕获其定义作用域中的变量**。

```rust
let x = 4;
let equal_to_x = |z| z == x;   // 闭包体内用了 x，尽管 x 不是参数
```

写成 `fn equal_to_x(z: i32) -> bool { z == x }` 会编译失败：`E0434: can't capture dynamic environment in a fn item`。代价是闭包要为捕获的变量占内存，函数则从不需要。捕获方式恰好对应函数取参的三种方式：

| trait | 捕获方式 | 对应函数参数 | 何时实现 |
|---|---|---|---|
| `FnOnce` | 取得所有权 | 按值传参 | **所有**闭包都实现（因为至少要被调用一次） |
| `FnMut` | 可变借用 | `&mut` | 没有把捕获变量所有权移进闭包的闭包 |
| `Fn` | 不可变借用 | `&` | 不需要对被捕获变量可变访问的闭包 |

`equal_to_x` 只读取 `x`，因此实现 `Fn`。若在参数列表前加 `move` 关键字，就强制闭包取得环境值的所有权——这在**把闭包传给新线程**时最实用（第 16 章详讲）。演示一下 `move` 的后果，这次用 `Vec` 而非整数（整数可拷贝、不会真的移走）：

```rust
let x = vec![1, 2, 3];
let equal_to_x = move |z| z == x;
println!("can't use x here: {:?}", x);   // error[E0382]: use of moved value: `x`
```

`x` 被移进闭包，`main` 里就不能再用它了。实用建议：trait bound **先从 `Fn` 开始**，编译器会告诉你何时该换成 `FnMut` 或 `FnOnce`。

## 迭代器：惰性的传送带

迭代器负责"遍历每一项 + 决定何时结束"，最大特点是**惰性**：`let v1_iter = v1.iter();` 本身什么都不发生，直到被真正消费。核心是标准库的 `Iterator` trait：

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // 此处省略了方法的默认实现
}
```

`type Item` 是**关联类型**（第 19 章详解），即迭代器产出元素的类型；`next` 是唯一必须实现的方法，一次返回一个项包在 `Some` 里，结束返回 `None`。直接调用 `next` 时迭代器变量必须 `mut`——它会改变记录位置的状态，也就是**消费**了迭代器（`for` 循环会自己取得所有权并内部处理可变性）。三个常用入口方法：

| 方法 | 产出的迭代器 | 适用场景 |
|---|---|---|
| `iter` | 元素的不可变引用（`&T`） | 只读遍历（如 `search`） |
| `iter_mut` | 元素的可变引用（`&mut T`） | 需要原地修改元素 |
| `into_iter` | 拥有所有权的元素（`T`） | 需要搬走元素（如 `shoes_in_my_size`） |

### 消费适配器 vs 迭代器适配器

`Iterator` trait 的众多方法分两类，区别十分关键：

| 类别 | 代表方法 | 是否取得所有权 | 是否立即执行 | 备注 |
|---|---|---|---|---|
| **消费适配器** | `sum`、`collect` | 是（消费后迭代器不可再用） | 是 | 内部反复调用 `next` 直到结束 |
| **迭代器适配器** | `map`、`filter`、`zip`、`skip` | 否 | 否，惰性 | 返回新迭代器，可链式调用 |

经典陷阱是只写 `v1.iter().map(|x| x + 1);` 而不消费它——编译器会警告 `unused std::iter::Map ... iterator adaptors are lazy and do nothing unless consumed`，那个闭包**从未被调用过**。补上 `collect` 才是完整用法：

```rust
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
assert_eq!(v2, vec![2, 3, 4]);
```

`filter` 则常与捕获环境的闭包搭配——下面这个函数只挑出指定尺码的鞋子，闭包捕获了 `shoe_size`：

```rust
fn shoes_in_my_size(shoes: Vec<Shoe>, shoe_size: u32) -> Vec<Shoe> {
    shoes.into_iter()
        .filter(|s| s.size == shoe_size)
        .collect()
}
```

因为 `map` 接受闭包，我们可以自定义"每个元素要做什么"，同时复用 `Iterator` 提供的迭代逻辑——这是闭包与迭代器配合的绝佳示例。

### 自定义迭代器：只需要实现 next

只要为类型实现 `Iterator`，就能免费获得标准库所有默认方法。下面这个 `Counter` 从 1 数到 5：

```rust
impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

有了 `next`，所有默认方法就都免费到手了。例如把两个 `Counter` 配对（第二个 `skip(1)` 跳过首值）、相乘、只留能被 3 整除者、再求和：

```rust
let sum: u32 = Counter::new()
    .zip(Counter::new().skip(1))
    .map(|(a, b)| a * b)
    .filter(|x| x % 3 == 0)
    .sum();
assert_eq!(18, sum);
```

注意 `zip` 只产生四对值：理论上第五对 `(5, None)` 从未产生，因为 `zip` 在任一输入迭代器返回 `None` 时就返回 `None`。

## 回头重写 minigrep：用迭代器去掉 clone 和可变状态

上一章我们留下两处"以后再说"的地方，现在可以收拾了。

### 一、`Config::new`：从索引 + clone 到移动所有权

原来 `new` 接收 `&[String]`，自身不拥有参数，为了返回拥有 `query`/`filename` 的 `Config`，只好 `clone` 两次。既然 `env::args` 本身就返回迭代器，我们干脆把**迭代器的所有权**交给 `Config::new`：

```rust
// main.rs：不再 collect 成 Vec，直接传迭代器
let config = Config::new(env::args()).unwrap_or_else(|err| {
    eprintln!("Problem parsing arguments: {}", err);
    process::exit(1);
});
```

签名相应改成 `pub fn new(mut args: std::env::Args) -> Result<Config, &'static str>`（`args` 要被迭代推进状态，故需 `mut`），函数体里的长度检查与索引访问全部换成 `next`：

```rust
impl Config {
    pub fn new(mut args: std::env::Args) -> Result<Config, &'static str> {
        args.next();                    // 第一个值是程序名，跳过

        let query = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a query string"),
        };
        let filename = match args.next() {
            Some(arg) => arg,
            None => return Err("Didn't get a file name"),
        };

        let case_sensitive = env::var("CASE_INSENSITIVE").is_err();
        Ok(Config { query, filename, case_sensitive })
    }
}
```

每次 `next()` 返回 `Some` 就 `match` 取出 `String`，返回 `None` 说明参数不够，直接 `Err` 提前返回。因为迭代器把值交给我们，`String` 是**移动**进 `Config` 的——不再需要 `clone` 分配新内存。

### 二、`search`：用适配器替代可变 vector

原版 `search` 里有个 `let mut results = Vec::new();` 加上 `for`/`push`。用 `filter` + `collect` 之后，整个函数的意图变成一句话：

```rust
pub fn search<'a>(query: &str, contents: &'a str) -> Vec<&'a str> {
    contents.lines()
        .filter(|line| line.contains(query))
        .collect()
}
```

| 维度 | for + push 版 | filter + collect 版 |
|---|---|---|
| 可变状态 | 有一个 `results` vector | 无 |
| 关注点 | 如何遍历、如何攒结果 | 只需表达"保留含 query 的行" |
| 未来并行化 | 需管理 `results` 的并发访问 | 无共享可变状态，更易改造 |

函数式风格倾向于最小化可变状态，这会让未来的并行搜索更易实现。多数 Rust 开发者更偏好迭代器版本：初看略绕，习惯后反而**更易看清代码目的**——样板循环被抽走，只剩业务特有的过滤条件。

## 性能对比：循环一定更快吗？

直觉上更"底层"的 `for` 循环应更快。用《福尔摩斯探案集》全文查找 "the" 做基准测试：

| 实现 | 耗时（ns/iter） |
|---|---|
| `bench_search_for` | 19,620,300（±915,700） |
| `bench_search_iter` | 19,234,900（±657,200） |

迭代器版本**还略快一点**。这正是 Rust 的**零成本抽象**（zero-cost abstractions）：抽象不引入运行时开销，与本贾尼·斯特劳斯特卢普在《Foundations of C++》中提出的零开销原则一致——"你不需要的，无需为其买单；你需要时，也找不到更好的代码了"。

书中还有一段音频解码器代码：用 `coefficients.iter().zip(&buffer[i - 12..i]).map(...).sum::<i64>()` 做线性预测。其汇编与手写版相同——迭代次数固定为 12，Rust 直接**展开**（unroll）循环，系数放进寄存器，也没有数组边界检查。放心用迭代器和闭包：它们让代码更高级，却不因此变慢。

## 实践建议与总结

1. **闭包用于"延迟执行 + 捕获上下文"**：要把行为存起来稍后执行、或让它记住几个外部变量时用它；一旦想复用"缓存结果"逻辑，就升级为 `Cacher` 式结构体（注意单值缓存、类型写死两个限制）。
2. **`Fn`/`FnMut`/`FnOnce` 先别背**：从 `Fn` 写起，让编译器告诉你是否需要更强的约束；需要把值搬进新线程时再考虑 `move`。
3. **优先迭代器风格，但先想清楚消费点**：漏掉 `collect`/`sum` 这类消费适配器，整条链式调用等于没写（编译器会警告）；`iter`/`iter_mut`/`into_iter` 决定你拿到的是引用还是所有权，这直接决定要不要 `clone`。

本章两个"意外收获"更值钱：一是抽象不必然付费——`filter().collect()` 与手写循环打平甚至更快；二是**抽象收益会累积**——上一章的 `clone` 与可变 `results` 能被干净消掉，正因它们被隔离在 `Config::new` 和 `search` 两个小函数里。下一章走出代码：`cargo` 的 profile 定制、文档注释与发布 crate。
