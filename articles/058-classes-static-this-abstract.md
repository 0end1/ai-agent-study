# TypeScript 学习笔记（12/17）：Classes（下）——静态、this 的多面性与抽象契约

> 对应书源：TypeScript Handbook「Classes」后半（Static Members / Generic Classes / this Types / Parameter Properties / abstract / 类间关系）｜系列第 12 篇

## 开篇：上半场介绍了这栋楼，下半场要聊它的公共管道与图纸规则

Classes 上半篇把类当成"带施工说明的图纸"——成员是房间、构造函数是开工、可见性是门禁卡。但真实项目里，类还有另一半人格：挂在类本身上的**静态成员**、开局意外流失的 **`this`**、随调用对象变形的 **`this` 类型**、一键合成属性的**参数属性**，以及只许当基类不得实例化的**抽象类**。这些就是本篇的菜单。

## 一、Static Members：属于"类"而不属于"实例"

静态成员不与某个具体实例关联，通过类构造器对象本身访问：

```ts
class MyClass {
  static x = 0;
  static printX() { console.log(MyClass.x); }
}
console.log(MyClass.x);
MyClass.printX();
```

要点三连：静态成员同样支持 `public`/`protected`/`private` 可见性修饰；静态成员**也会被继承**（`Derived.getGreeting()` 能拿到基类的静态方法）；但某些名字不能用——因为类本身是可被 `new` 调用的函数，覆盖 `Function.prototype` 上的属性不安全，`static name` 会直接报错 2699。

### 为什么没有 static class？

C# 那种"静态类"在 TS/JS 里不存在，原因是那些语言强制所有数据和函数必须待在类里才需要这么个构造；TS 没这个限制，**单例直接写成普通对象或顶层函数**即可：

| 写法 | 评价 |
|---|---|
| `class MyStaticClass { static doSomething() {} }` | 多余的"静态类" |
| `function doSomething() {}` | ✅ 推荐（纯函数） |
| `const MyHelperObject = { dosomething() {} }` | ✅ 推荐（工具对象） |

### static 块：带隐私权限的初始化手术台

`static {}` 块提供一段拥有独立作用域的语句序列，能访问类内部的私有字段——既具备语句级的全部能力，变量又不会泄漏到外面：

```ts
class Foo {
  static #count = 0;
  get count() { return Foo.#count; }
  static {
    try {
      const lastInstances = loadLastInstances();
      Foo.#count += lastInstances.length;
    } catch {}
  }
}
```

## 二、泛型类：类型会被抹掉的冷酷现实

类可以像接口一样泛型，实例化时类型参数与函数调用一样被推断（`new Box("hello!")` 得到 `Box<string>`），也支持约束与默认值。但有一条硬规则：

```ts
// @errors: 2302
class Box<Type> {
  static defaultValue: Type; // 不允许！
}
```

原因值得牢记——**类型永远被完全擦除**。运行时只有一个 `Box.defaultValue` 属性槽，`Box<string>.defaultValue` 若存在，就会与 `Box<number>.defaultValue` 共用同一个槽，这是灾难。结论：**泛型类的静态成员永远不能引用类的类型参数**。

## 三、运行时 this：JS 的著名脾气

TS 不改变 JS 的运行时行为，而 JS 的 `this` 恰好以古怪著称：

```ts
const c = new MyClass();
const obj = { name: "obj", getName: c.getName };
console.log(obj.getName()); // 打印 "obj"，不是 "MyClass"！
```

函数里的 `this` 取决于**函数怎么被调用**。通过 `obj` 引用调用，`this` 就是 `obj`——类实例早丢了。TS 提供两条互补的对策：

### 箭头函数 vs `this` 参数

| 方案 | 运行时安全 | 内存 | `super` 可用 | 对 JS 调用方有约束力 |
|---|---|---|---|---|
| 箭头函数属性 `getName = () => {}` | ✅ 绝对正确 | ❌ 每个实例一份拷贝 | ❌ 原型链找不到基类方法 | ✅ 有 |
| `this` 参数 `getName(this: MyClass)` | ❌ JS 调用方仍能用错 | ✅ 每类一份 | ✅ 可用 `super.` | ❌ 仅编译期检查 |

`this` 参数是被编译擦除的**假参数**：`function fn(this: SomeType, x: number)` 编译后就是 `function fn(x)`，但它让 TS 能在编译期校验调用上下文是否正确（错误 2684）。两者按"是否会被跨层传递、是否在意内存、是否要用 super"三问取舍。

## 四、`this` 类型：会随子类变形的活类型

类里有个特殊类型叫 `this`，它**动态指向当前类的类型**。看这个链式 API：

```ts
class Box {
  contents: string = "";
  set(value: string) {
    this.contents = value;
    return this;  // 返回类型被推断为 this，而非 Box
  }
}
class ClearableBox extends Box { clear() { this.contents = ""; } }

const b = new ClearableBox().set("hello");
//    ^? ClearableBox —— 链式调用不丢具体类型
```

这正是**流式 API（fluent API）**在继承体系里还能保有精确类型的秘密。同理，参数位置用 `other: this` 与写 `other: Box` 有本质区别：派生类的 `sameAs` 只接受同一派生类的实例（`derived.sameAs(base)` 报错 2345）。

### `this`-based 类型守卫

返回位置可以用 `this is Type`，配合 if 断言把目标对象收窄：

```ts
class FileSystemObject {
  isFile(): this is FileRep { return this instanceof FileRep; }
  isNetworked(): this is Networked & this { return this.networked; }
}
if (fso.isFile()) { fso.content; }       // 收窄为 FileRep
else if (fso.isNetworked()) { fso.host; } // 收窄为 Networked & FileSystemObject
```

另一个常见用法是懒验证：Box<T> 的 `hasValue(): this is { value: T }` 在检查通过后，就把 `value` 类型里的 `undefined` 精准剔除。

## 五、参数属性、类表达式与构造签名

**参数属性**是省字 syntactic sugar：构造函数参数前加 `public`/`private`/`protected`/`readonly`，自动变成同名同值的类属性，构造函数可以空着：

```ts
class Params {
  constructor(
    public readonly x: number,
    protected y: number,
    private z: number
  ) { /* No body necessary */ }
}
```

**类表达式**与类声明几乎一致，唯一差别是可以匿名（`const someClass = class<Type> { ... }`），通过绑定到的标识符引用。**构造签名**方面，`typeof Point` 是类本身的类型，要拿实例类型得用 `InstanceType<typeof Point>` 工具类型。

## 六、abstract：不许实例化的图纸罐

抽象方法/字段没有实现，只能存在于抽象类中；抽象类不能被直接 `new`（错误 2511），它的职责是充当基类，由子类补齐所有抽象成员——不含任何抽象成员的类称为**具体类（concrete）**。漏实现会报错 2515。

### 抽象构造签名

想写个函数"接收某个继承自抽象类的类构造器并实例化它"，直接写 `typeof Base` 是错的（错误 2511）——因为那样 `greet(Base)` 也合法，等于你允许构造抽象类。正确写法是**构造签名**：

```ts
function greet(ctor: new () => Base) {
  const instance = new ctor();
  instance.printName();
}
greet(Derived); // ✅ 具体类可实例化
greet(Base);    // ❌ 抽象类被拦住
```

## 七、类之间的关系：结构性比较的意外之喜

大多数情况下，TS 里的类是**按结构比较**的，与别的类型一视同仁：

```ts
class Point1 { x = 0; y = 0; }
class Point2 { x = 0; y = 0; }
const p: Point1 = new Point2(); // OK，结构相同
```

子类型关系甚至不需要显式继承——有 `name`/`age` 的 `Employee` 可赋给 `Person`。但有个必须警惕的边界：**空类没有成员，在结构类型系统里是万物的超类型**。

```ts
class Empty {}
function fn(x: Empty) {}
fn(window); fn({}); fn(fn); // 全都 OK！
```

所以书里的建议是干脆别写空类。顺带注意，private/protected 成员会打破这种结构性兼容（两侧的同名私有成员必须同源声明），这是"名义类型"味道的来源。

## 实践建议

1. **单例别用 static class**：顶层函数或普通对象更贴合语言习惯。
2. **泛型类里别让静态成员碰类型参数**——想想类型擦除后的单槽共享。
3. **会被当回调摘出去的方法用箭头函数**，普通方法优先保留原型链（省内存、能 super）。
4. **流式 API 返回 `this` 类型**，让子类链式调用不退化。
5. **需要强制子类实现某行为时用 abstract + 抽象构造签名**，而不是靠文档约定。

## 小结

类篇下半场覆盖了四类"类级"能力：静态侧（static 成员/块与泛型类的擦除禁令）、运行时 this 的两种防御姿势、**会随继承变形的 `this` 类型**与 this-based 守卫、以及抽象类对未来子类的契约约束。加上类之间的结构比较规则，Classes 章节至此走完。下一篇进入 More on Functions——函数签名的高级世界。
