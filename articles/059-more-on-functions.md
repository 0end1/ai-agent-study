# TypeScript 学习笔记（13/17）：More on Functions——把"怎么调"说清楚

> 对应书源：TypeScript Handbook「More on Functions」全篇｜系列第 13 篇

## 开篇：函数是"会跑的值"

函数是任何应用的基本构件——本地函数、跨模块导入、类的方法都是它。但函数还有一个身份常被忽略：**函数也是值**。既然是值，就得有类型来描述"它接受什么、返回什么、能不能带属性、能不能被 new"。本篇就是 TS 描述函数类型的完整工具箱。

## 一、三种描述"可调用物"的语法

**函数类型表达式**最简单，形似箭头函数：

```ts
function greeter(fn: (a: string) => void) { fn("Hello, World"); }
```

注意两个易踩的坑：参数类型不写就是隐式 `any`；**参数名是必需的**——`(string) => void` 不是"接收 string 的函数"，而是"接收一个名叫 `string`、类型为 `any` 的参数的函数"！

**调用签名**用于描述"既可调用又带属性"的函数（JS 函数可以有属性），写法在对象类型里，且分隔符从 `=>` 换成 `:`：

```ts
type DescribableFunction = {
  description: string;
  (someArg: number): boolean; // 注意是冒号
};
```

**构造签名**就是在调用签名前加 `new`。像 `Date` 这种"既能 `new` 也能直接调"的对象，两种签名可以任意混在同一个类型里：

| 语法 | 适用场景 | 分隔符 |
|---|---|---|
| `(a: string) => void` | 纯函数类型表达式 | `=>` |
| `{ (n?: number): string; new (s: string): Date; }` | 可调用 / 可构造 / 带属性 | `:` |

## 二、泛型函数：用类型参数"连线"

泛型解决的是**描述多个值之间的类型对应关系**。返回数组首元素，`any[]` 版本丢失了类型信息，泛型版本则把输入与输出连起来：

```ts
function firstElement<Type>(arr: Type[]): Type | undefined {
  return arr[0];
}
const s = firstElement(["a", "b", "c"]); // string
const u = firstElement([]);              // undefined
```

类型参数通常被自动推断（独立版 `map<Input, Output>` 甚至能从回调的返回值推断 `Output`）；推断不灵时手动指定——`combine<string | number>([1,2,3], ["hello"])`。

**约束（extends）** 则给连线加限定：`longest<Type extends { length: number }>` 保证函数体内能访问 `.length`，同时 `longest(10, 100)` 会被拒（number 没有 length）。

### 约束的经典陷阱：minimumLength

```ts
function minimumLength<Type extends { length: number }>(obj: Type, minimum: number): Type {
  if (obj.length >= minimum) return obj;
  return { length: minimum }; // 错误 2322！
}
```

看起来返回值满足约束就够了？不。函数承诺返回的是**传入的那个 Type**，不是"某个满足约束的东西"——否则调用方拿到的 `{ length: 6 }` 调 `.slice()` 会直接崩溃。

### 写好泛型函数的三条官方规则

1. **尽量用类型参数本身，而不是约束它**：`firstElement1<Type>(arr: Type[])` 推断出 `Type`，而 `firstElement2<Type extends any[]>(arr: Type)` 只能得出 `any`（TS 被迫用约束类型解析 `arr[0]`）。
2. **类型参数能少则少**：`filter2<Type, Func extends ...>` 里的 `Func` 没有联系任何两个值，纯属给调用方添堵。
3. **类型参数应出现两次**：它存在的意义就是"关联"，只出现一次就重新考虑是否真的需要泛型（`greet<Str extends string>(s: Str)` 应该直接写成 `(s: string)`）。

## 三、可选参数、默认值与回调签名陷阱

`x?: number` 标可选，但函数体内 `x` 实际是 `number | undefined`；给默认值 `x = 10` 后体内就是纯 `number`。调用方永远可以显式传 `undefined` 模拟"缺参"。

**回调签名的可选参数是重灾区**：

```ts
function myForEach(arr: any[], callback: (arg: any, index?: number) => void) { ... }
myForEach([1,2,3], (a, i) => console.log(i.toFixed())); // 错误 2532！
```

写 `index?` 的本意是"调用方可以不接收 index"，但它声明的是"**我的实现可能不传这个参数**"——于是 TS 强制你检查 `i` 可能是 undefined。而 JS 里多传的实参本来就会被忽略、少参函数总能顶替多参函数，所以规则是：**给回调写函数类型时，除非你真的打算不传那个参数，否则永远别写可选参数**（直接 `(arg: any, index: number) => void`）。

## 四、函数重载：多副面孔，一个实现

重载签名写在实现之前，实现签名对外**不可见也不可直接调用**：

```ts
function makeDate(timestamp: number): Date;
function makeDate(m: number, d: number, y: number): Date;
function makeDate(mOrTimestamp: number, d?: number, y?: number): Date { ... }
makeDate(1, 3); // 错误 2575——没有两参的重载！
```

实现签名必须与所有重载兼容（参数类型、返回类型都要对得上，否则错误 2394）。而写好重载的头号原则是——**能用联合类型就别用重载**：

```ts
// 两个重载：len(Math.random() > 0.5 ? "hello" : [0]) 会报错——调用只能解析到单个重载
// 联合类型版：一个签名通吃
function len(x: any[] | string) { return x.length; }
```

## 五、this 声明与函数家族速查表

回调式 API（调用时机被别人掌控）用 `this` 参数声明上下文，且**必须用 `function`，箭头函数拿不到这种行为**：`db.filterUsers(function (this: User) { return this.admin; })`。

| 类型 | 一句话要点 |
|---|---|
| `void` | 不返回值的函数的推断类型；**`void` ≠ `undefined`** |
| `object` | 任何非原始类型；**`object` ≠ `Object`，永远用小写** |
| `unknown` | 同 `any` 一样接受一切，但使用前必须收窄（安全的 `any`） |
| `never` | 函数抛异常/不终止时；联合穷尽后剩余分支 |
| `Function` | 描述 bind/call/apply，但调用返回 `any`——**避免**，不打算调用就写 `() => void` |

## 六、Rest 参数、解构与 void 的可赋值性特例

**rest 参数**（`...m: number[]`）接收无界实参，注解必须是 `T[]`/`Array<T>` 或元组。与它对称的是 **rest 实参**——用展开语法从可迭代对象（比如数组）**提供**一批参数，例如 `arr1.push(...arr2)`。注意 TS 不假设数组不可变：`const args = [8, 5]` 推断为 `number[]`（"零个或多个数"），而非"恰好两个数"，喂给 `Math.atan2(...args)` 会报错 2556——`as const` 化为二元组即可（与 055 篇 as const 化学反应一脉相承）；目标运行时较旧时可能还需开启 `downlevelIteration`。**参数解构**的类型注解写在解构模式之后，嫌啰嗦就提一个具名类型：`function sum({ a, b, c }: ABC) {}`。

**void 返回类型的可赋值性**是本章最精妙的设计：上下文类型为 `() => void` 时，实现**可以返回任何值**，只是返回值会被忽略：

```ts
const src = [1, 2, 3], dst = [0];
src.forEach((el) => dst.push(el)); // push 返回 number，forEach 要 void——完全合法
```

这条规则存在的意义就是让上面这种代码成立。但注意边界：**直接标注** `function f2(): void { return true; }` 的函数体里必须真的不返回值（@ts-expect-error 标记处报错）——"上下文 void 宽容，显式 void 严格"。

## 实践建议

1. **函数类型表达式必写参数名**，别让 `(string) => void` 变成事故。
2. **泛型三问**：类型参数用了几次？没关联两个值就删；能用类型参数本身别加约束。
3. **回调签名不写可选参数**，除非实现真的可能不传。
4. **重载之前先想联合类型**；真用重载，记住实现签名对外不可见。
5. **接收任意函数用 `() => void` 而非 `Function`**；描述"可能是任何东西"的返回值用 `unknown` 而非 `any`。

## 小结

本篇把函数类型的工具箱一次装满：三种可调用物语法（表达式/调用签名/构造签名）、泛型的连线思维与三条官方规则、回调可选参数的陷阱、重载的"实现不可见"与"联合优先"、this 参数的回调场景、void/object/unknown/never/Function 五兄弟的边界，以及 void 可赋值性特例。下一篇进入泛型的系统性专题。
