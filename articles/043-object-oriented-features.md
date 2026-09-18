# Rust 学习笔记（18/21）：Rust 的面向对象特性——trait 对象与状态模式

> 对应书源：《Rust 程序设计语言》第 17 章「Rust 的面向对象特性」

## 开篇：「Rust 到底算不算面向对象语言？」

番茄是水果还是蔬菜？植物学说它是水果，厨房里说它是蔬菜——答案取决于你采用哪套标准。「Rust 是不是面向对象」也是同一类问题，编程社区至今没有统一定义，在某些定义下 Rust 是，在另一些定义下不是。

所以本章不急着下结论，而是把大家普遍认可的面向对象特征拆成三块——**对象、封装、继承**——逐一对照 Rust 的实际做法。接着再动手实现一个经典设计模式（状态模式），最后讨论一个关键问题：照搬面向对象模式，是不是发挥了 Rust 的全部优势？

## 一、对象包含数据和行为：这一条 Rust 达标

《设计模式》（GoF，"四人帮"）对面向对象编程的定义是：

> 面向对象的程序由对象组成。一个对象包含数据和操作这些数据的过程，这些过程通常被称为方法或操作。

按这个标准，Rust 完全达标：`struct` 和 `enum` 装数据，`impl` 块提供方法。唯一的差别在于，Rust 刻意**不**把结构体和枚举叫作「对象」，以便和其他语言里的对象概念区分开。功能上，它们提供的是同一件事。

## 二、封装：`pub` 就是那道门

封装的意思是：对象的实现细节对使用它的代码不可见，唯一交互方式是公有 API。这样重构内部时不必改动外部代码。

Rust 的做法你已经熟悉——默认私有，用 `pub` 逐项开放。书里给了个漂亮的例子 `AveragedCollection`，它维护一个整型列表与列表平均值，并把平均值缓存下来：

```rust
pub struct AveragedCollection {
    list: Vec<i32>,
    average: f64,
}

impl AveragedCollection {
    pub fn add(&mut self, value: i32) {
        self.list.push(value);
        self.update_average();
    }

    pub fn remove(&mut self) -> Option<i32> {
        let result = self.list.pop();
        match result {
            Some(value) => {
                self.update_average();
                Some(value)
            }
            None => None,
        }
    }

    pub fn average(&self) -> f64 {
        self.average
    }

    fn update_average(&mut self) {
        let total: i32 = self.list.iter().sum();
        self.average = total as f64 / self.list.len() as f64;
    }
}
```

结构体本身是 `pub`，但 `list` 和 `average` 字段仍是私有的。这一点至关重要：外部代码无法直接增删 `list`，因此「平均值与列表不同步」这种 bug 在结构上就不可能发生。而好处随之而来——将来想把 `Vec<i32>` 换成 `HashSet<i32>`，只要 `add`/`remove`/`average` 的签名不变，调用方一行都不用改；反之如果把 `list` 设为 `pub`，外部直接操作它，换数据结构就会引发连锁修改。

## 三、继承：Rust 明确没有，但给了两条替代路径

如果「必须有继承才算面向对象」，那 Rust 就不算。你无法定义一个结构体继承另一个结构体的字段和方法。但人们用继承通常出于两个原因，Rust 各有对应方案：

| 使用继承的原因 | Rust 的替代方案 |
| --- | --- |
| **代码复用**：一个类型实现的行为，想给另一个类型直接用 | trait 的默认方法实现（如 `Summary` 的 `summarize`），实现该 trait 即可拥有，也可覆盖 |
| **多态**：子类型可以用在父类型出现的地方 | 泛型 + trait bound（有界参数化多态），或 trait 对象 |

近年来继承在很多语言里「失宠」，因为它常常强迫子类共享并不需要的父类特性，设计变僵硬，还可能出现「子类调用了其实不适用于它的方法」。有些语言还只允许单继承，进一步限制了灵活性。出于这些原因，Rust 选了另一条路：**用 trait 对象实现多态**。

## 四、trait 对象：让不同类型的值共处一个集合

第 8 章说过，`Vec` 只能存同种元素，用 `SpreadsheetCell` 枚举可以绕开——但那要求类型集合在编译期就固定。如果写库的人无法预知使用者会新增什么类型呢？

设想一个 GUI 库：它遍历组件列表，对每个组件调用 `draw`。库作者知道 `Button`、`TextField`，但使用者可能想加 `Image`、`SelectBox`。做法是定义一个 `Draw` trait，让 `Screen` 持有 trait 对象的 vector：

```rust
pub trait Draw {
    fn draw(&self);
}

pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

`Box<dyn Draw>` 就是 trait 对象：它是「任何实现了 `Draw` 的类型」的替身。使用者只需为自己的类型实现 `Draw`，就能塞进 `Screen`：

```rust
use gui::{Screen, Button};

fn main() {
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No"),
                ],
            }),
            Box::new(Button {
                width: 50,
                height: 10,
                label: String::from("OK"),
            }),
        ],
    };

    screen.run();
}
```

如果换成泛型写法 `Screen<T: Draw>`，`components` 就只能是**清一色** `Button` 或清一色 `TextField`。两种写法该怎么选？

| 维度 | 泛型 + trait bound | trait 对象 `Box<dyn Draw>` |
| --- | --- | --- |
| 元素类型 | 必须同质 | 可异质混装 |
| 类型集合 | 编译期已知 | 库使用者可自行扩展 |
| 分发方式 | 静态分发（单态化，编译期确定调用哪个方法） | 动态分发（运行时通过指针查表） |
| 性能 | 可内联，有优化空间 | 无法内联，丧失部分优化 |
| 适用场景 | 同质集合 | 需要运行时灵活扩展 |

`run` 不需要知道组件到底是 `Button` 还是 `SelectBox`，只关心「能调用 `draw`」——这很像动态类型语言里的**鸭子类型**（走起来像鸭子、叫起来像鸭子，那它就是鸭子）。区别在于：动态语言要到运行时才发现「这个对象没有 draw 方法」，而 Rust 在编译期就拦下，比如往 `Screen` 里塞一个 `String`，会直接报 `error[E0277]: the trait bound 'String: Draw' is not satisfied`。

代价是**动态分发**：编译器不知道 trait 对象的具体类型，只能在运行时查方法表，因此无法内联优化。

还有一条硬规则：只有**对象安全**的 trait 才能做成 trait 对象。实践中最常遇到的是——方法不能返回 `Self`、不能带泛型参数。因为 trait 对象已经「忘记」了具体类型。典型反例是 `Clone`（`fn clone(&self) -> Self`），写 `Vec<Box<dyn Clone>>` 会报 `error[E0038]: the trait 'Clone' cannot be made into an object`。

## 五、状态模式实战：一篇博客的发布工作流

**状态模式**的要点是：一个值有内部状态，行为随状态而变；每个状态各自负责自己的行为以及何时转移到下一个状态；持有状态的值对这些细节毫不知情。

需求是这样的：新建的博文是草案 → 请求审核 → 审核通过 → 发布。其他一切非法操作（比如在审核前直接发布）都**不产生效果**。用状态模式实现时，`Post` 内部持有一个 trait 对象：

```rust
pub struct Post {
    state: Option<Box<dyn State>>,
    content: String,
}

impl Post {
    pub fn new() -> Post {
        Post {
            state: Some(Box::new(Draft {})),
            content: String::new(),
        }
    }

    pub fn request_review(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.request_review())
        }
    }

    pub fn approve(&mut self) {
        if let Some(s) = self.state.take() {
            self.state = Some(s.approve())
        }
    }

    pub fn content(&self) -> &str {
        self.state.as_ref().unwrap().content(self)
    }
}

trait State {
    fn request_review(self: Box<Self>) -> Box<dyn State>;
    fn approve(self: Box<Self>) -> Box<dyn State>;
    fn content<'a>(&self, post: &'a Post) -> &'a str {
        ""
    }
}
```

几个设计细节值得单独拎出来：

| 手法 | 为什么这样写 |
| --- | --- |
| `state` 用 `Option<Box<dyn State>>` | 要取走旧状态的所有权，而 Rust 不允许结构体里有「空字段」，于是用 `take()` 取出 `Some` 留下 `None` |
| `self: Box<Self>` | 只能在这个类型的 `Box` 上调用，消费旧状态使其失效，返回新状态 |
| `as_ref().unwrap()` | 需要的是 `Option` 里值的**引用**而非所有权（不能把 `state` 移出 `&self`）；所有方法都保证返回时是 `Some`，所以 `unwrap` 不会 panic |
| `content` 给默认实现 `""` | `Draft`、`PendingReview` 直接继承即可，只有 `Published` 覆盖成 `&post.content`；需要生命周期标注，因为返回值借自参数 `post` |

三个状态各自实现转移规则：`Draft::request_review` 返回 `PendingReview`，`PendingReview::approve` 返回 `Published`，而 `Draft::approve` 和 `Published::*` 都返回 `self`（非法操作静默无效）。于是「未审核的博文不能显示内容」这条规则，全部集中在状态对象里，`Post` 自身一无所知。

**权衡取舍**：优点显而易见——要看「已发布博文有哪些行为」，只需看 `Published` 一处；新增状态只需加一个 struct 并为其实现 trait，不用到处改 `match`。缺点也有三个：状态之间互相耦合（想在 `PendingReview` 和 `Published` 之间插入 `Scheduled`，就得改 `PendingReview` 的代码）；存在重复逻辑（想给返回 `self` 的方法加默认实现会破坏对象安全）；`Post` 里 `request_review`/`approve` 的模板化代码，方法多了可以考虑用宏消除。

## 六、更好的一种可能：把状态编码进类型

既然目标是「非法状态不可达」，那不如让类型系统直接拦住它。换个思路：`Post::new()` 返回的不是 `Post` 而是 `DraftPost`，而 **`DraftPost` 根本没有 `content` 方法**：

```rust
pub struct Post {
    content: String,
}

pub struct DraftPost {
    content: String,
}

impl Post {
    pub fn new() -> DraftPost {
        DraftPost { content: String::new() }
    }

    pub fn content(&self) -> &str {
        &self.content
    }
}

impl DraftPost {
    pub fn add_text(&mut self, text: &str) {
        self.content.push_str(text);
    }

    pub fn request_review(self) -> PendingReviewPost {
        PendingReviewPost { content: self.content }
    }
}

pub struct PendingReviewPost {
    content: String,
}

impl PendingReviewPost {
    pub fn approve(self) -> Post {
        Post { content: self.content }
    }
}
```

`request_review` 和 `approve` 都获取 `self` 的所有权，消费掉旧实例、返回新类型。于是「拿到能读 `content` 的 `Post`」的唯一路径，就是 `DraftPost → request_review → PendingReviewPost → approve → Post`。想显示草案内容？那行代码压根编译不过。调用方相应要重新绑定变量：

```rust
let mut post = Post::new();
post.add_text("I ate a salad for lunch today");
let post = post.request_review();
let post = post.approve();
assert_eq!("I ate a salad for lunch today", post.content());
```

| 维度 | 状态模式（trait 对象） | 状态编码为类型 |
| --- | --- | --- |
| 非法状态 | 运行时静默无效 | 编译期错误，根本无法表示 |
| 转移封装 | 完全封装在 `Post` 内部 | 暴露给用户，需 `let post = ...` 重新绑定 |
| 扩展方式 | 新增一个状态 struct | 新增类型并调整转换链 |
| 代价 | 状态间耦合、部分重复代码 | 严格来说不再是「状态模式」 |

## 实践建议

1. **同质用泛型，异质用 trait 对象**。集合元素类型一致且编译期已知，就用泛型 + trait bound 享受静态分发；需要让使用者自行扩展类型集合，再用 `Box<dyn Trait>`，并坦然接受动态分发的那点开销。
2. **让非法状态无法被表示**。能用类型系统在编译期拦住的 bug，就不要留到运行时去「返回空字符串」。这是 Rust 相比传统 OOP 的额外红利。
3. **别生搬设计模式**。Rust 拥有所有权、trait、泛型这些面向对象语言没有的武器，照搬 OOP 模式未必是最优解——先问一句「这个约束能不能交给编译器」。

## 小结

trait 对象是 Rust 获取部分面向对象能力的手段：用少量运行时性能换取灵活性，进而支撑状态模式这类可维护性模式。但同时，Rust 也提供了「把状态编码进类型」这种更彻底的方案——它不是面向对象模式，却能把 bug 挡在部署之前。这正是本章的落点：**面向对象模式在 Rust 中始终可用，但并不总是最佳选择**。

下一篇进入第 18 章「模式和匹配」，那些散落全书的 `match`、`if let`，终于要被系统地讲一次了。
