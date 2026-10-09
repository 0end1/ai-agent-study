# TypeScript 学习笔记（5/17）：日常类型（下）——断言、字面量与 null 的边界术

> 对应书源：TypeScript Handbook「Everyday Types」后半（Type Assertions / Literal Types / null & undefined / Enums / Less-Common Primitives）｜系列第 5 篇

## 开篇：地图之外的地带

前两篇我们把类型词汇表基本背完了。但类型系统终究是一张"编译器眼中的地图"——总有它不知道、而你知道的角落。这一篇讲的就是地图之外的三块地带：**类型断言**（"听我的，我知道它是什么"）、**字面量类型**（把"某个具体的值"本身变成类型），以及 **null 与 undefined**（"值的缺席"该如何在类型里安放）。

这三样东西有一个共同的气质：**都是你在和类型系统做"私下交易"**——用得对，代码更精准；用得滥，类型保护被一步步凿穿。

## 一、类型断言：对编译器说"这个我熟"

典型场景：`document.getElementById` 在 TypeScript 眼里只会返回"某种 `HTMLElement`"，但你清楚地知道页面上这个 ID 挂的是画布元素。这时用**类型断言（type assertion）**指定更具体的类型：

```ts
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```

也可以用尖括号写法 `<HTMLCanvasElement>getElementById(...)`——但 `.tsx` 文件里不能用（会和 JSX 混淆），所以实践中几乎都用 `as`。

三条铁律必须记牢：

1. **断言和类型注解一样会被编译器擦除，没有运行时检查**——断言错了，不会有异常也不会有 `null`，错误会一路潜伏到运行时才炸。它是"说服编译器"，不是"校验自己"。
2. **只允许转换到更具体或更宽泛的类型**，用来堵住"不可能的强转"：`const x = "hello" as number;` 会直接报错——`string` 与 `number` 毫无重叠。
3. 规则偶尔过于保守，挡住确实合法的复杂转换时，可以走**两段式断言**：先断言成 `any`（或后面会讲的 `unknown`），再断言到目标——`expr as any as T`。这等于亲手拆掉护栏，慎用。

## 二、字面量类型：把"值"变成类型

除了宽泛的 `string`，TypeScript 还允许在类型的位置直接写**具体的字符串或数字**。它和 `let`/`const` 的行为差异恰好是最佳入口：

```ts
let changingString = "Hello World";
changingString = "Olá Mundo"; // 可以改 → 类型是 string

const constantString = "Hello World";
// 不能改 → 类型是 "Hello World" 这个字面量本身
```

**`let` 变量可能装任何字符串，类型就是 `string`；`const` 只能是一个值，类型就是那个值。** 变量的可变性，直接反映在类型的宽窄上。

单独一个字面量类型没什么用——只能装一个值的变量毫无意义：

```ts
let x: "hello" = "hello";
x = "howdy"; // 报错
```

但它一旦**组合成联合类型**就大放异彩——表达"只接受这一组已知值"：

```ts
function printText(s: string, alignment: "left" | "right" | "center") {
  // ...
}
printText("Hello, world", "left");
printText("G'day, mate", "centre"); // 报错！拼错立刻现形
```

数字字面量同理：`function compare(a: string, b: string): -1 | 0 | 1`，把比较函数的三种返回约定写进了签名。字面量还能和非字面量类型混搭：`configure(x: Options | "auto")`。

顺带一个冷知识：**`boolean` 本身就是 `true | false` 的别名**——布尔类型只有两个成员，每个成员自己就是一个字面量类型。

## 三、字面量推断：为什么 req.method 不是 "GET"

字面量类型最经典的坑在这里：

```ts
declare function handleRequest(url: string, method: "GET" | "POST"): void;

const req = { url: "https://example.com", method: "GET" };
handleRequest(req.url, req.method); // 报错！
```

`req.method` 被推断成了 `string` 而非 `"GET"`。原因：TypeScript 假设**对象的属性之后可能会被改**——在创建 `req` 和调用函数之间，完全可能有人写 `req.method = "GUESS"`。类型要同时服务于**读和写**，所以只能放宽到 `string`。

两种解法，语义不同：

```ts
// 解法一：断言个别字段（或调用处断言 req.method as "GET"）
const req = { url: "https://example.com", method: "GET" as "GET" };

// 解法二：as const，把整个对象"锁死"成字面量
const req = { url: "https://example.com", method: "GET" } as const;
handleRequest(req.url, req.method); // OK
```

`as const` 就像 `const`，但作用在**类型系统**上：保证所有属性都取字面量类型，而不是 `string`/`number` 这样的宽泛版本。只需锁定个别字段用断言；想让整个对象不可放宽，用 `as const`。

## 四、null 与 undefined：值的缺席如何安放

JS 用 `null` 和 `undefined` 表示"没有值"，TypeScript 给了两个同名类型。它们的行为完全取决于 `strictNullChecks`：

| 设置 | 行为 | 风险 |
| --- | --- | --- |
| 关闭（默认） | `null`/`undefined` 可赋给任何类型，访问不设限 | 和 Java/C# 一样裸奔，漏判是重大 bug 源 |
| 开启（推荐） | 用之前**必须先检查** | 换来编译器把关 |

开启后，处理方式和收窄一样：

```ts
function doSomething(x: string | null) {
  if (x === null) {
    // 什么都不做
  } else {
    console.log("Hello, " + x.toUpperCase()); // 这个分支里 x 是 string
  }
}
```

还有一个"胆子大"的写法——**非空断言操作符（后缀 `!`）**：在表达式后加 `!`，等于声明"它绝不是 null/undefined"，直接把这两种值从类型里赶走：

```ts
function liveDangerously(x?: number | null) {
  console.log(x!.toFixed()); // 不报错——但请注意函数名
}
```

和所有断言一样，`!` 不改变运行时行为，也不会在值为 null 时抛异常——**只在百分之百确定非空时才用**。

## 五、枚举与罕见原始类型：知道就好

**枚举（enum）** 是 TypeScript 少数**不是类型层面的扩展、而是真正进入运行时**的功能（其他注解、断言、接口编译后全消失，enum 会生成真实代码）。正因如此，Handbook 的态度罕见地保守：

> 枚举让你能描述"一组命名常量中的一个值"。但它不是类型层面的附加物，而是加进了语言和运行时——**你应该知道它存在，但最好先别用**，除非确定需要。

最后是两个罕见原始类型的"点名"：`bigint`（ES2020 起，用于超大整数，`BigInt(100)` 或 `100n` 字面量）和 `symbol`（`Symbol("name")` 创建**全局唯一**引用——两个 `Symbol("name")` 永不相等，用 `===` 比较会直接被编译器判为不可达代码）。日常开发遇到的机会不多，认个脸即可。

## 实践建议

1. **`as` 三思而后用**：每次断言前问一句"我凭什么确定？"——来自接口文档、页面约定可以；来自"应该没问题"不行。能用收窄解决的，绝不用断言。
2. **常量配置对象默认 `as const`**：路由表、选项映射这类不会被改的对象，加 `as const` 一次性拿到最窄类型，后续传参不再踩推断放宽的坑。
3. **`strictNullChecks` 必开，`!` 少用**：开了它，`null` 检查从"线上事故"变成"编译错误"；`!` 是逃生舱里的逃生舱，用了就等于自己签了免责协议。
4. **枚举先放一放**：需要"一组固定值"时，字符串字面量联合（`"left" | "right" | "center"`）通常就够，而且零运行时成本。

## 小结

本篇收尾了《Everyday Types》：**类型断言**是你比编译器懂时的表态，编译期擦除、无运行时检查，只许更具体或更宽泛的两段式例外；**字面量类型**把具体值变成类型，`let`/`const` 的可变性决定推断宽窄，联合起来表达"枚举式"约束，`as const` 则是对象版的字面量锁；**null/undefined** 在 `strictNullChecks` 开启后必须显式处理，`!` 非空断言是最后的免责声明；**枚举**是唯一进入运行时的类型功能，官方建议慎用。至此，日常类型三部曲（上：原始与函数，中：对象与联合，下：断言与 null）全部完成。

下一篇进入独立的《Narrowing》章节：typeof 守卫、真值收窄、等值收窄、`in` 与 `instanceof`、控制流分析、判别联合，以及那个"永不到达"的 `never`——把本篇埋的"收窄"伏笔正式挖出来。
