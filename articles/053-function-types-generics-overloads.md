# TypeScript 学习笔记（7/17）：函数进阶——把"可调用"本身变成类型

> 对应书源：TypeScript Handbook「More on Functions」全篇（Function Type Expressions / Signatures / Generics / Overloads / this / void）｜系列第 7 篇

## 开篇：函数也是一种值

在 JavaScript 里，函数是一等公民：能赋值给变量、能当参数传、能挂属性、还能 `new`。既然函数是值，类型系统就该能**描述"一个函数长什么样"**。这一篇是函数类型的大全：从最简单的函数类型表达式，到泛型、重载，再到 `this` 与 `void` 这些容易被忽略的角落。

## 一、描述函数的三种姿势

**1. 函数类型表达式**——长得像箭头函数：

```ts
function greeter(fn: (a: string) => void) {
  fn("Hello, World");
}
```

`(a: string) => void` 表示"接收一个 string 参数、无返回值的函数"。两个细节：参数名**必须写**——`(string) => void` 的意思不是"参数是 string"，而是"参数名叫 string、类型是 any"！未标注的参数隐式 `any`。可以用类型别名给它起名：`type GreetFunction = (a: string) => void;`

**2. 调用签名（Call Signature）**——函数类型表达式没法声明属性，但 JS 的函数可以带属性。要描述"可调用且有属性的对象"，在对象类型里写调用签名：

```ts
type DescribableFunction = {
  description: string;
  (someArg: number): boolean; // 注意这里用 : 而非 =>
};
```

**3. 构造签名（Construct Signature）**——能被 `new` 调用的函数，在调用签名前加 `new`：

```ts
type SomeConstructor = {
  new (s: string): SomeObject;
};
```

像 `Date` 这样"可调用也可 new"的对象，两种签名可以混在同一个类型里。三者对比：

| 写法 | 语法特征 | 适用场景 |
|------|----------|----------|
| 函数类型表达式 | `(a: string) => void`，用 `=>` | 普通回调、函数参数 |
| 调用签名 | `{ description: string; (n: number): boolean }`，用 `:` | 可调用且带属性的函数 |
| 构造签名 | `{ new (s: string): Date }` | 可 `new` 的构造器（可两者混用） |

## 二、泛型函数：给类型之间"连线"

`firstElement(arr: any[])` 能用但返回 `any`——丢了类型信息。泛型通过**类型参数**把输入和输出的类型关联起来：

```ts
function firstElement<Type>(arr: Type[]): Type | undefined {
  return arr[0];
}
const s = firstElement(["a", "b", "c"]); // s: string
const n = firstElement([1, 2, 3]);       // n: number
```

不用手写 `Type`，编译器自动**推断**。多个类型参数也行：`map<Input, Output>` 同时从数组和回调返回值两处推断。

**约束（constraint）**用 `extends` 限定类型参数的能力范围：

```ts
function longest<Type extends { length: number }>(a: Type, b: Type) {
  if (a.length >= b.length) return a; // 有约束才能访问 .length
  return b;
}
longest([1, 2], [1, 2, 3]);   // number[]
longest("alice", "bob");      // "alice" | "bob"
longest(10, 100);             // ❌ number 没有 length
```

注意 `longest("alice", "bob")` 返回的是**字面量联合**——泛型的本质是"用同一个类型关联多个值"，推断结果精确到具体值。

**受约束值的经典陷阱**：返回 `{ length: minimum }` 看似满足约束，实则违反承诺——函数承诺返回**和传入同类的对象**，而不只是"满足约束的某个对象"。若允许，`minimumLength([1,2,3], 6)` 会返回 `{length: 6}`，随后 `arr.slice(0)` 直接崩溃。类型参数保持"同一个 Type"的同一性，正是泛型的保护伞。

**需要手动指定**类型参数的场景：`combine([1,2,3], ["hello"])` 推断失败（两个数组类型不一致），但若这是本意，可写 `combine<string | number>([1,2,3], ["hello"])`。

**写好泛型的三条官方规则**：

1. **能下推就下推**：`firstElement1<Type>(arr: Type[])` 返回 `Type`；`firstElement2<Type extends any[]>(arr: Type)` 返回 `any`——后者被迫用约束类型解析 `arr[0]`。尽量直接用类型参数本身，而非约束它。
2. **类型参数越少越好**：`filter2<Type, Func extends (arg: Type) => boolean>` 里的 `Func` 没有关联任何两个值，纯属累赘。
3. **类型参数应出现两次**：`greet<Str extends string>(s: Str)` 不如 `greet(s: string)`——类型参数是用来"关联"的，只出现一次就什么都没关联。

## 三、可选参数与回调陷阱

`f(x?: number)` 的 `x` 实际类型是 `number | undefined`（JS 中缺省参数就是 `undefined`）；给默认值 `f(x = 10)` 则函数体内 `x` 就是 `number`。

**回调里的可选参数是高频坑**：

```ts
function myForEach(arr: any[], callback: (arg: any, index?: number) => void) {
  // ...
}
myForEach([1, 2, 3], (a, i) => console.log(i.toFixed())); // ❌ i 可能是 undefined
```

写 `index?` 的本意是"回调**可以**接收第二个参数"，但类型上它表达的是"实现**可能不传**第二个参数"——所以回调内部拿到的 `i` 是 `number | undefined`。

> **规则**：给回调写函数类型时，除非你打算**调用**时不传该参数，否则永远别写 `?`。JS 里多传的实参本来就会被忽略，少参数的函数总能顶替多参数的函数。

## 四、函数重载：一张脸多个签名

一个函数想支持"1 个参数（时间戳）或 3 个参数（月日年）"两种调用方式，就写多个**重载签名** + 一个实现：

```ts
function makeDate(timestamp: number): Date;
function makeDate(m: number, d: number, y: number): Date;
function makeDate(mOrTimestamp: number, d?: number, y?: number): Date {
  if (d !== undefined && y !== undefined) return new Date(y, mOrTimestamp, d);
  return new Date(mOrTimestamp);
}
makeDate(12345678); // ✅
makeDate(5, 5, 5);  // ✅
makeDate(1, 3);     // ❌ 没有两个参数的重载！
```

最容易困惑的一点：**实现签名对外不可见**，它只服务于函数体；即便实现里写了 `d?` 可选，外部也只能按上面的重载签名调用。同时实现签名必须与所有重载兼容（参数类型、返回类型都对得上）。

**能用联合就用联合，别用重载**：

```ts
function len(x: any[] | string) { return x.length; } // ✅ 更好
// 而不是 len(s: string): number; len(arr: any[]): number;
```

后者连 `len(Math.random() > 0.5 ? "hello" : [0])` 都调不了——TS 只会把调用解析到**单个**重载上。

## 五、this 的显式声明

回调式 API 里，`this` 由调用方决定。TS 借用了"JS 禁止 this 作参数名"的语法空间来声明它的类型：

```ts
interface DB {
  filterUsers(filter: (this: User) => boolean): User[];
}
const admins = db.filterUsers(function (this: User) {
  return this.admin;
}); // 必须用 function，箭头函数没有自己的 this
```

## 六、函数语境下的特殊类型速览

| 类型 | 含义 | 关键提醒 |
|------|------|----------|
| `void` | 无返回值 | **不等于 `undefined`**；`undefined` 只是它携带的值 |
| `object` | 任何非原始类型 | `object` ≠ `Object` ≠ `{}`，**永远用小写 `object`** |
| `unknown` | 安全版 `any` | 接收任意值但用前必须收窄；`JSON.parse` 的理想返回类型 |
| `never` | 永不返回 / 联合收窄到空 | `throw` 或死循环的函数；穷举检查的基石 |
| `Function` | 全局函数类型 | 调用返回 `any`，尽量避开；只需"不调用的任意函数"时用 `() => void` |

## 七、rest 参数与参数解构

收集不定数量参数用 **rest 参数**（类型必须是 `T[]` 或元组），展开数组传参用 **rest 实参**：

```ts
function multiply(n: number, ...m: number[]) {
  return m.map((x) => n * x);
}
```

一个意外：`const args = [8, 5]; Math.atan2(...args)` 会报错——`args` 被推断为 `number[]`（零到多个数），而不是"恰好两个"。TS 不假设数组不可变，最直接的修法是 `const args = [8, 5] as const;` 把它锁成二元组（呼应 051 篇的 `as const`）。

参数解构的类型注解写在解构语法**之后**：`function sum({ a, b, c }: { a: number; b: number; c: number })`，用命名类型别名可免掉冗长。

## 八、void 的可赋值性：一个反直觉的设计

上下文为 `() => void` 的函数**可以返回任何值，只是返回值会被忽略**：

```ts
type voidFunc = () => void;
const f1: voidFunc = () => true; // 合法！
```

这个设计就是为了让下面的常见代码合法——`push` 返回新长度（number），`forEach` 只要求 `void` 回调：

```ts
src.forEach((el) => dst.push(el)); // ✅
```

但注意边界：**直接定义**返回类型为 `void` 的函数时，函数体里不能 `return true`——"上下文 void"宽容，"字面声明 void"严格。

## 实践建议

1. **回调函数类型里别写可选参数**——除非你实现时真的可能不传它。
2. **泛型宁少勿多**：类型参数出现不到两次的，先想想是否根本不需要泛型。
3. **联合类型优先于重载**：重留给参数个数或返回类型真正不同的情况。
4. **接收任意输入用 `unknown` 而非 `any`**，返回解析结果标 `unknown`，逼调用方收窄。
5. **别用全局 `Function`**：不调用就 `() => void`，要调用就写清楚签名。

## 小结

这一篇把"函数作为值"的类型语言补齐了：三种签名描述法覆盖了可调用对象的所有形态；泛型用"关联"思维让输入输出同呼吸；重载与联合的取舍、可选参数在回调中的语义反转、`void` 的双向宽容——都是实战里天天撞见的细节。下一篇进入对象类型：可选/只读修饰符、索引签名与多余属性检查。
