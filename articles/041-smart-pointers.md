# Rust 学习笔记（16/21）：智能指针——当编译期规则需要一点「弹性」

> 前几章我们反复看到 Rust 用所有权和借用规则在编译期杜绝内存问题：一个值只有一个所有者，引用要么多个不可变、要么一个可变。这些规则在绝大多数情况下非常清晰。但当数据结构本身需要"多个入口共享同一块内存"，或者"不可变外壳里藏着可变内核"时，编译器的严格有时会成为表达能力的障碍。智能指针就是 Rust 提供的标准答案：它们仍然遵守安全规则，只是把检查的场合从"编译时"挪到了"运行时"，或者通过额外的元数据（引用计数）扩展了所有权的语义。

## 开篇：引用只是「借」，智能指针是「拥有」

第4章的引用 `&T` 只借用数据，没有任何额外开销。智能指针则是一类**拥有**它们指向的数据，并附带额外元数据或功能的数据结构。`String` 和 `Vec<T>` 其实也算智能指针——它们拥有数据、管理容量、保证 UTF-8 或内存连续性。

标准库中最常用的三个是 `Box<T>`、`Rc<T>` 和 `RefCell<T>`，而理解它们的关键在于两个 trait：

- **`Deref`**：让智能指针可以像引用一样用 `*` 解引用
- **`Drop`**：让智能指针在离开作用域时自动执行清理代码

| 类型 | 核心能力 | 所有权模式 | 适用场景 |
|---|---|---|---|
| `Box<T>` | 堆分配 | 单一所有者 | 递归类型、大数据转移、trait 对象 |
| `Rc<T>` | 引用计数 | 多个所有者（只读） | 图结构多节点共享、树的多父引用 |
| `RefCell<T>` | 内部可变性 | 单一所有者（运行时借用检查） | 需要可变借用的 mock 对象、与 `Rc` 组合 |

## 一、Box<T>：把数据放到堆上

`Box<T>` 是最简单的智能指针，功能只有两个：在堆上分配数据，以及留在栈上的指针。没有额外开销。

```rust
let b = Box::new(5);
println!("b = {}", b);  // 像访问栈数据一样自然
```

`Box<T>` 真正的价值在于**递归类型**。Rust 需要在编译时知道类型大小，而递归类型理论上可以无限嵌套，大小未知。cons list（Lisp 风格链表）是典型的例子：

```rust
// 编译错误：recursive type `List` has infinite size
enum List {
    Cons(i32, List),
    Nil,
}
```

把 `List` 换成 `Box<List>`，问题就解决了——`Box` 是指针，大小固定（平台指针宽度），编译器能算出 `Cons` 需要 `i32 + box 指针` 的空间，递归链被打破：

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

let list = Cons(1,
    Box::new(Cons(2,
        Box::new(Cons(3,
            Box::new(Nil))))));
```

> 虽然函数式语言常用 cons list，Rust 里 `Vec<T>` 是更好的选择。这里用它只是因为概念简单，能清晰展示递归类型与 `Box` 的关系。

## 二、Deref trait：让自定义类型也能用 `*`

`Box<T>` 能像引用一样用 `*y` 解引用，是因为它实现了 `Deref` trait。我们自己也可以做到。

实现 `Deref` 只需要提供一个 `deref` 方法，返回内部数据的引用：

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &T {
        &self.0
    }
}
```

当写下 `*y` 时，Rust 实际上执行的是 `*(y.deref())`。`deref` 返回引用而非值，是为了不转移所有权。

### 解引用强制转换（Deref Coercions）

这是 `Deref` 带来的更实用的功能。当函数参数类型是 `&str` 而传入 `&MyBox<String>` 时，Rust 会自动链式调用 `deref`：

```rust
fn hello(name: &str) { ... }
let m = MyBox::new(String::from("Rust"));
hello(&m);  // 自动：&MyBox<String> → &String → &str
```

没有强制转换，你得写 `hello(&(*m)[..])`。强制转换发生在编译期，零运行时开销。

| 转换方向 | 条件 |
|---|---|
| `&T` → `&U` | `T: Deref<Target=U>` |
| `&mut T` → `&mut U` | `T: DerefMut<Target=U>` |
| `&mut T` → `&U` | `T: Deref<Target=U>`（可变→不可变永远安全） |

注意反向不可能：不可变引用不能强转为可变引用，因为借用规则无法保证唯一性。

## 三、Drop trait：离开作用域时自动「收尾」

`Drop` trait 让你定义值离开作用域时要执行的清理逻辑。`Box<T>` 用它来释放堆内存，你也可以用来关闭文件或网络连接。

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping `{}`", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("my stuff") };
    let d = CustomSmartPointer { data: String::from("other stuff") };
    println!("Created.");
}
// 输出顺序：Created. → Dropping `other stuff` → Dropping `my stuff`
```

变量以**创建顺序的相反顺序**被丢弃，所以 `d` 先于 `c`。

### 提前丢弃：std::mem::drop

不能手动调用 `c.drop()`——Rust 禁止显式析构，因为结束时还会自动再调用一次，导致 double free。如果确实需要提前清理，用标准库的 `std::mem::drop` 函数（注意是小写 `drop`，不是 `Drop` trait 的方法）：

```rust
drop(c);  // prelude 已导入，提前释放
println!("CustomSmartPointer dropped before end of main.");
```

## 四、Rc<T>：一个值，多个所有者

有些场景天然需要多所有权。比如图结构中多个边指向同一个节点，节点应该直到没有边指向它时才被释放。`Rc<T>`（reference counting）就是为此设计：

```rust
use std::rc::Rc;

enum List {
    Cons(i32, Rc<List>),
    Nil,
}

let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
let b = Cons(3, Rc::clone(&a));
let c = Cons(4, Rc::clone(&a));
```

`Rc::clone(&a)` 不会深拷贝数据，只是把引用计数加 1。标准库故意让 `Rc::clone` 与深拷贝的 `.clone()` 同名但语义不同，这是 Rust 社区的习惯：看到 `Rc::clone` 就知道"只是增计数，不用担心性能"。

用 `Rc::strong_count(&a)` 可以观察计数变化：创建 `a` 时为 1，克隆给 `b` 后为 2，给 `c` 后为 3，`c` 离开作用域后自动减回 2。

> ⚠️ `Rc<T>` **只能用于单线程**。多线程的引用计数在第16章讲。

## 五、RefCell<T>：编译器信不过你，你自己来保证

`Rc<T>` 解决了多所有权，但它只提供不可变访问。如果要修改共享数据怎么办？`RefCell<T>` 引入**内部可变性**（interior mutability）：外表不可变，内部可变。

`RefCell<T>` 的借用规则不在编译期检查，而在**运行时**检查。违反规则不会编译错误，而是直接 panic：

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1, 2, 3]);
cell.borrow_mut().push(4);     // 可变借用，OK
cell.borrow().len();           // 不可变借用，OK
// 同时两个 borrow_mut() → panic: already borrowed: BorrowMutError
```

`borrow()` 返回 `Ref<T>`，`borrow_mut()` 返回 `RefMut<T>`，两者都实现了 `Deref`，可以像普通引用一样用。`RefCell` 内部维护活跃借用的计数，规则与编译期完全一致：多个不可变 ✅，单个可变 ✅，多个可变 ❌，可变+不可变 ❌。

### Mock 对象的经典用例

这是书里最有说服力的例子。假设有个 `Messenger` trait，方法签名要求 `&self`（不可变），但测试时 mock 对象需要记录调用过的消息（需要可变内部状态）：

```rust
struct MockMessenger {
    sent_messages: RefCell<Vec<String>>,
}

impl Messenger for MockMessenger {
    fn send(&self, message: &str) {
        self.sent_messages.borrow_mut().push(String::from(message));
    }
}
```

外部调用者看到的 `send` 接收 `&self`，符合 trait 契约；内部通过 `RefCell` 完成可变操作。运行时如果出了错（比如两个 `borrow_mut` 重叠），测试会 panic，而不是带着 bug 上线。

### Rc<RefCell<T>>：多所有者 + 可变

把两者嵌套，就能得到"多个所有者且都能修改"的数据结构：

```rust
let value = Rc::new(RefCell::new(5));
let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
let b = Cons(Rc::new(RefCell::new(6)), Rc::clone(&a));

*value.borrow_mut() += 10;  // a 和 b 共享的 5 变成 15
```

## 六、引用循环与 Weak<T>：小心「互相指」带来的泄漏

Rust 不保证完全避免内存泄漏——如果 `Rc<T>` 之间形成循环引用，计数永远到不了 0，内存就不会释放：

```rust
let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));
// 让 a 指向 b，形成循环
if let Some(link) = a.tail() {
    *link.borrow_mut() = Rc::clone(&b);
}
// a 和 b 的 strong_count 都是 2，离开作用域后都剩 1，内存泄漏
```

解决方法是 **`Weak<T>` 弱引用**。`Rc::downgrade` 创建弱引用，它**不拥有**数据，只增加 `weak_count`。当 `strong_count` 降为 0 时，值就被释放，弱引用通过 `upgrade()` 返回 `Option<Rc<T>>` 来安全访问——如果值还在就返回 `Some`，已被释放就返回 `None`。

树结构的父节点引用是 `Weak<T>` 的典型场景：父节点拥有子节点（`Rc` 强引用），子节点知道父节点（`Weak` 弱引用）。父节点被丢弃时子节点跟着消失，子节点被丢弃时父节点不受影响。

```rust
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,      // 弱引用，不拥有父节点
    children: RefCell<Vec<Rc<Node>>>, // 强引用，拥有子节点
}
```

## 实践建议

1. **默认用 `Box<T>` 处理堆分配和递归类型**，它最简单、零开销、不会引入新的借用复杂度。
2. **需要共享读权限时选 `Rc<T>`，需要共享写权限时套 `RefCell<T>`**，但记住：
   - `Rc<T>` 只读、`RefCell<T>` 单线程——两者都不是万能药
   - `Rc<RefCell<T>>` 组合给了灵活性，也给了制造引用循环的能力
3. **有父子双向引用时，子→父用 `Weak<T>`**，这是避免循环泄漏的标准做法。始终问自己：这个方向的引用是"拥有"还是"知道"？拥有用 `Rc`，知道用 `Weak`。

智能指针的本质，是 Rust 在"编译期保证安全"这条主线上开出的几扇侧门：`Box` 解决大小未知，`Rc` 扩展所有权语义，`RefCell` 把检查推迟到运行时，`Weak` 打破循环。它们都封装在安全的 API 里，底线没有破——只是换了一种方式守住它。下一章，我们把这些工具带进并发世界，看看 Rust 如何用所有权系统实现"无畏并发"。
