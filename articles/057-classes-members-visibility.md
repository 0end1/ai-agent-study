# TypeScript 学习笔记（11/17）：Classes（上）——成员、继承契约与可见性

> 对应书源：TypeScript Handbook「Classes」前半（Class Members / Class Heritage / Member Visibility 至 private）｜系列第 11 篇

## 开篇：类是"带施工说明的图纸"

如果说对象类型是 describing 一栋房子的样子，那类就是把图纸和施工队绑在一起：字段是房间，构造函数是开工仪式，可见性修饰符则是门禁卡权限。TypeScript 对 ES2015 的 `class` 提供完整支持，它做的事不是发明新的类语法，而是在 JS 类的骨架上**补类型关系**。本篇走完类的前半程：成员、继承契约与可见性。

## 一、Class Members：从空类开始

最简单的类是空类，然后逐步添加成员。

### 字段（Fields）

字段声明创建一个公开可写的属性，类型注解可省（省了就是隐式 `any`），也可以带初始化器——实例化时自动执行，且**初始化器会推断字段类型**：

```ts
class Point {
  x = 0; // 推断为 number
  y = 0;
}
const pt = new Point();
pt.x = "0"; // 错误：不能把 string 赋给 number
```

`--strictPropertyInitialization` 选项控制字段是否必须在构造函数里初始化。未初始化的 `name: string` 会报错误 2564；注意字段必须在**构造函数本身**里赋值——TS 不会追踪你从构造函数里调用的方法，因为派生类可能重写那些方法而漏掉初始化。如果字段确定会被外部填充（比如某个库替你初始化），可用确定赋值断言 `name!: string` 跳过检查。

`readonly` 修饰符则禁止构造函数之外的一切赋值：构造函数里可以给 `readonly name` 赋初值，但方法里和外部的 `g.name = ...` 都会报错 2540。

### 构造函数

构造函数签名与函数几乎一样：类型注解、默认值、**重载**全都支持。仅有的两个限制：

- 不能有类型参数（泛型归外层类声明管）；
- 不能写返回类型注解（返回的永远是类实例类型）。

#### Super 调用

有基类时，必须在构造函数体内、使用任何 `this.` 成员**之前**调用 `super()`：

```ts
class Derived extends Base {
  constructor() {
    console.log(this.k); // 错误 17009：super 之前不能用 this
    super();
  }
}
```

JS 里忘调 `super` 是常见事故（ES6 下直接抛异常），TS 会提前告诉你。

### 方法与 `this.` 纪律

方法就是类上的函数属性，注解规则与函数相同。关键纪律：**方法体内访问字段必须写 `this.`**——不带限定的名字永远指向外层作用域：

```ts
let x: number = 0; // 全局
class C {
  x: string = "hello";
  m() {
    x = "world"; // 改的是外面那个 x，不是类属性！
  }
}
```

这类 bug 编译期就能抓到（把 string 赋给外层的 number 变量报错 2322），前提是你别用 `any` 糊过去。

### Getters / Setters

访问器的特殊推断规则：**只有 `get` 没有 `set` 时，属性自动变 readonly**；setter 参数类型未标注时从 getter 返回类型推断。TS 4.3 起还允许 get/set **类型不同**——`get size(): number` 配 `set size(value: string | number | boolean)`，写入端宽容、读取端严格。书里顺带提醒：没有额外逻辑的字段代理 get/set 在 JS 里几乎没意义，公开字段就好。

## 二、Class Heritage：implements 是安检，extends 是血缘

### `implements`：只查不改

`implements` 用于检查类是否满足某接口，**但它完全不改变类或其方法的类型**。最经典的误解：

```ts
interface Checkable {
  check(name: string): boolean;
}
class NameChecker implements Checkable {
  check(s) { // s 不会因此变成 string！
    return s.toLowerCase() === "ok";
  }
}
```

你可能以为参数 `s` 会被接口的 `name: string` 影响类型——不会。`implements` 只做"可当 XX 用"的资格审核，不参与类体内部的类型推断。同理，接口里的可选属性 `y?` 也不会因为 implements 就凭空出现在类上。

| | `implements` | `extends` |
|---|---|---|
| 对象 | 接口（可多个） | 基类（只能一个） |
| 效果 | 只检查，不改类型 | 真实继承属性与方法 |
| 常见误解 | 以为会改变成员类型 | —— |

### `extends` 与重写契约

派生类继承基类全部成员，可用 `super.` 访问基类方法（注意 JS 类只是查找链，**没有 super 字段**的概念）。TS 强制**派生类永远是基类的子类型**：

```ts
class Derived extends Base {
  greet(name: string) { ... } // 错误 2416：基类 greet() 无参
}
```

看似严格，实则必要：用基类引用持有派生实例（`const b: Base = new Derived()`）是合法且常见的写法。若 Derived 把参数从可选改成必填，`b.greet()` 调用就会拿到 undefined 的 `name` 直接崩溃。子类型约束守护的正是"基类引用永远可用"这一承诺。

三个进阶坑位：

- **Type-only 字段声明**：`target >= ES2022`（或 `useDefineForClassFields: true`）时，字段初始化发生在父类构造器之后，会**覆盖**父类设置的值。只想给继承字段收窄类型时，用 `declare resident: Dog` 声明零运行时效果。
- **初始化顺序**：基类字段 → 基类构造器 → 派生类字段 → 派生类构造器。所以基类构造器里打印 `this.name` 看到的是 `"base"` 而非派生类重定义的 `"derived"`——派生类字段那时还没初始化。
- **继承内置类型**（Error/Array/Map）在降级到 ES5 时会坏：`instanceof` 失灵、方法丢失。官方建议在 `super(...)` 后手动 `Object.setPrototypeOf(this, MsgError.prototype)`。

## 三、Member Visibility：三张门禁卡

| 修饰符 | 类外 | 子类 | 同类其他实例 |
|---|---|---|---|
| `public`（默认） | ✅ | ✅ | ✅ |
| `protected` | ❌ | ✅ | —— |
| `private` | ❌ | ❌ | ✅ |

`public` 是默认值，写出来只为了风格。`protected` 成员仅对子类可见——但子类可以"放大门禁"：重声明时不加修饰符就变 public。因为派生类本来就能自由读写它，这不算破坏安全，只是**没打算暴露时要记得重复写 `protected`**。跨层级访问则被禁止：Derived2 不能访问兄弟类 Derived1 实例上的 protected 成员——它只对 Derived2 的子类开放。

`private` 更严：连子类都不行，且子类不能提升其可见性（错误 2415）。有趣的是 TS 允许**同类的不同实例互访 private 成员**（Java/C#/C++ 同派，Ruby 派不许）：

```ts
class A {
  private x = 10;
  sameAs(other: A) {
    return other.x === this.x; // 合法
  }
}
```

### 重要警告：软私有 vs 硬私有

TS 的 `private`/`protected` **只在类型检查期生效**。编译成 JS 后，`s.secretKey` 照样能访问，甚至类型检查期内 `s["secretKey"]` 方括号写法也是放行的（方便单元测试）——这是**软私有**。而 JS 原生的 `#` 私有字段编译后依然私有、无任何逃生口，是**硬私有**（降级到 ES2021 及以下时 TS 用 WeakMap 模拟）。

## 实践建议

1. **初始化器推断 + `strictPropertyInitialization` 开满**：字段类型靠推断，未初始化靠报错兜底，别滥用 `!`。
2. **别指望 implements 帮你补类型注解**：它只验资格；参数类型要自己写。
3. **重写方法守子类型契约**：参数只能更宽松（可选项化），不能更严苛。
4. **暴露内部细节前想清楚 protected→public 的放大效应**，无意暴露就重复修饰符。
5. **真要藏数据用 `#` 硬私有**；`private` 只防队友手滑，不防运行时窥探。

## 小结

类篇上半场把三件事讲透：**成员**（字段初始化的严格性、super 时序、this. 纪律、访问器推断）是语法层；**继承契约**（implements 只查不改、extends 强制子类型、初始化顺序）是关系层；**可见性**（三张门禁卡 + 软私有的边界）是封装层。下半场见 Static 成员、泛型类与抽象类。
