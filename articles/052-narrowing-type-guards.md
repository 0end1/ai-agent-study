# TypeScript 学习笔记（6/17）：Narrowing——教编译器读懂你的 if

> 对应书源：TypeScript Handbook「Narrowing」全篇（Type Guards / Truthiness / Discriminated Unions / never）｜系列第 6 篇

## 开篇：从"报错"到"没报错"，中间隔了一个 if

上一节的 `padLeft` 给我们留了个悬念：参数类型是 `number | string`，直接调 `" ".repeat(padding)` 编译器立刻报错——它可不敢赌你传进来的不是字符串。而解决办法朴素得像句废话：先 `if (typeof padding === "number")` 判断一下，错误就消失了。

```ts
function padLeft(padding: number | string, input: string): string {
  if (typeof padding === "number") {
    return " ".repeat(padding) + input; // padding: number
  }
  return padding + input; // padding: string
}
```

如果你把它当成普通 JavaScript 读，那就对了——这正是 TypeScript 的设计意图：**让你照常写 JS，类型安全是编译器"白送"的**。但表面平静，水面下藏着大工程：编译器把类型分析**叠加在 JS 的运行时控制流之上**（`if/else`、三元、循环、真值检查……），沿着每条可能的执行路径，把类型磨得越来越细。

这个"从宽类型磨到窄类型"的过程就叫 **Narrowing（收窄）**，那些让编译器识别分支类型的特殊检查被称为 **Type Guard（类型守卫）**。

## 一、typeof 守卫：最常用的那把尺子

`typeof` 的返回值被 TS 编码成了固定八种：`string`、`number`、`bigint`、`boolean`、`symbol`、`undefined`、`object`、`function`。在 TS 眼里，`typeof x === "..."` 就是标准的类型守卫。

但注意上面列表里**没有 `"null"`**——因为 JavaScript 的历史事故：`typeof null === "object"`。于是下面这段代码会踩坑：

```ts
function printAll(strs: string | string[] | null) {
  if (typeof strs === "object") {
    for (const s of strs) { // ❌ 报错：strs 仍是 string[] | null
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  }
}
```

编译器比人诚实：`typeof strs === "object"` 只能把类型收窄到 `string[] | null`，**收不掉 null**。它不会假装这个坑不存在，也不越权替你补 `null` 分支——这份"不过度热心"的分寸感，贯穿整个 Handbook。

## 二、真值收窄：好用，但有暗坑

JS 里 `if` 的条件可以是任何表达式，会先强制转成布尔再选分支。六个值转成 `false`：`0`、`NaN`、`""`、`0n`、`null`、`undefined`，其余全为 `true`。

利用这一点可以把 `printAll` 修好：

```ts
function printAll(strs: string | string[] | null) {
  if (strs && typeof strs === "object") { // && 先挡掉 null
    for (const s of strs) {
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  }
}
```

但换个写法就埋雷了——把整个函数体包进 `if (strs)`：

```ts
function printAll(strs: string | string[] | null) {
  if (strs) { // ⚠️ 别这么写！
    if (typeof strs === "object") { /* ... */ }
    else if (typeof strs === "string") { console.log(strs); }
  }
}
```

表面看没问题，实际上**空字符串 `""` 会被整体跳过**——而空字符串是合法的 `string`。TS 对此不报错（你只是"选择不处理"），但如果对原始类型做真值检查，务必想想 falsy 值表。

顺带一个细节：`Boolean("hello")` 返回类型是 `boolean`，而 `!!"hello"` 返回类型是字面量 `true`——后者的类型信息更精确。

## 三、等值收窄：`===` 是一次类型求交集

`===`、`!==`、`==`、`!=` 和 `switch` 都能收窄。最有意思的是两变量比较：

```ts
function example(x: string | number, y: string | boolean) {
  if (x === y) {
    x.toUpperCase(); // x: string —— 两个类型只有 string 是交集
  } else {
    console.log(y); // y: string | boolean（string 分支已被 else 剔除）
  }
}
```

`x === y` 意味着它们的值相同，类型自然也相同——只有 `string` 两边都有，于是收窄成 `string`。

另外 `== null`（及 `== undefined`）是个"双杀"检查：同时挡住 `null` 和 `undefined`，TS 能正确识别并把两者从类型中剔除。不想区分这两种"缺席"时，一个 `!= null` 就够。

## 四、in 与 instanceof：对象层面的守卫

`in` 检查属性是否存在，`instanceof` 检查原型链。两者都是类型守卫：

```ts
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) return animal.swim();
  return animal.fly();
}

function logValue(x: Date | string) {
  if (x instanceof Date) {
    console.log(x.toUTCString()); // Date
  } else {
    console.log(x.toUpperCase()); // string
  }
}
```

`in` 有个易被忽视的细节：**可选属性在 true/false 两个分支里都会出现**。若加上 `type Human = { swim?: () => void; fly?: () => void }`，`"swim" in animal` 的 true 分支是 `Fish | Human`，false 分支是 `Bird | Human`——毕竟人类穿上装备既能游泳也能飞。

## 五、赋值收窄与控制流分析

每次赋值，TS 会看右边的值收窄左边变量。但**可赋值性始终对照"声明类型"检查**：

```ts
let x = Math.random() < 0.5 ? 10 : "hello world!"; // 声明类型 string | number
x = 1;        // 此刻观察到 number
x = "goodbye!"; // 仍可赋 string —— 对照的是声明类型，不是上次的观察值
x = true;     // ❌ boolean 不在声明类型里，报错
```

更进一步，TS 做的是完整的**控制流分析**：`padLeft` 在 `if` 分支里已经 `return`，那么函数剩余部分在 `padding` 为 number 时不可达——于是 `number` 从后续类型中剔除。类型像水流一样随控制流分裂、合并，每个位置都可能有不同的观察类型。

## 六、类型谓词：自定义守卫

内置守卫不够用时，可以定义返回**类型谓词**（`paramName is Type`）的函数：

```ts
type Fish = { swim: () => void; name: string };
type Bird = { fly: () => void; name: string };
declare function getSmallPet(): Fish | Bird;

function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

let pet = getSmallPet();
if (isFish(pet)) {
  pet.swim(); // Fish
} else {
  pet.fly();  // else 分支也聪明：不是 Fish，那就是 Bird
}

// 还能直接喂给 filter 做类型过滤
const zoo: (Fish | Bird)[] = [getSmallPet(), getSmallPet()];
const underWater: Fish[] = zoo.filter(isFish);
```

注意谓词名必须与当前函数签名的参数名一致；`filter` 的回调若包了额外逻辑，需要写成 `(pet): pet is Fish => ...` 显式标注。

## 七、判别联合：本章的"大招"

收窄单个变量只是热身，实际代码里更多是**结构化数据的收窄**。先看一个失败的编码方式——用可选属性拼一个 `Shape`：

```ts
interface Shape {
  kind: "circle" | "square";
  radius?: number;      // 圆才有
  sideLength?: number;  // 方才有
}
```

`getArea(shape)` 里即使写了 `if (shape.kind === "circle")`，`shape.radius` 依旧报错——**编译器无法从 `kind` 的值推断出 `radius` 一定存在**，两者在类型定义上毫无关联。硬上只能 `shape.radius!` 非空断言，代码一挪动就脆。

正确做法是把"我们知道的信息"编码进类型——**联合 + 字面量 kind**：

```ts
interface Circle { kind: "circle"; radius: number }
interface Square { kind: "square"; sideLength: number }
type Shape = Circle | Square;

function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;  // shape: Circle ✅
    case "square":
      return shape.sideLength ** 2;        // shape: Square ✅
  }
}
```

当联合的每个成员都含一个**字面量类型的公共属性**时，TS 就视之为**判别联合（discriminated union）**，`kind` 即**判别属性**——检查 `kind === "circle"` 就能把联合里非此类型的成员全部剔除。这样连 `strictNullChecks` 关闭时也是安全的（前一种编码只在开启时才报错）。判别联合是表达网络消息协议、状态管理中 mutation 的标准姿势。

## 八、never 与穷举检查：让"漏加分支"变成编译错误

收窄的极限是把联合剔到空，这时 TS 用 `never` 表示"不该存在的状态"。`never` 可赋给任何类型，但没有任何类型可赋给它（除自身）——这个单向性正是穷举检查的机关：

```ts
type Shape = Circle | Square;

function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle": return Math.PI * shape.radius ** 2;
    case "square": return shape.sideLength ** 2;
    default:
      const _exhaustiveCheck: never = shape; // 所有情况已处理，shape 是 never
      return _exhaustiveCheck;
  }
}
```

一旦给 `Shape` 新增 `Triangle`，`default` 里的赋值立刻报错（`Triangle` 不能赋给 `never`）——**编译器逼你去补新分支**。这就是穷举检查（exhaustiveness checking）：加类型时漏改代码，不再靠运行时事故提醒你。

## 收窄手段速查表

| 手段 | 触发条件 | 典型场景 | 易错点 |
|------|----------|----------|--------|
| `typeof` | 与八种固定字符串比较 | 原始类型分流 | `typeof null === "object"`，收不掉 null |
| 真值 | `if(x)`、`&&`、`!` | 挡 null/undefined | `0`/`""`/`NaN` 也是 falsy，会误伤 |
| 等值 | `===`/`!==`/`==`/`switch` | 字面量、双变量交集 | `== null` 是"双杀"，别当只挡 null |
| `in` | 检查属性名 | 结构不同的对象联合 | 可选属性在两个分支都出现 |
| `instanceof` | 检查原型链 | Date/Error/自定义类 | 只适用于 class 实例 |
| 赋值 | 赋值语句 | 变量多次赋值 | 可赋值性对照声明类型 |
| 类型谓词 | `x is T` 返回类型 | 自定义守卫、filter 过滤 | 参数名须与签名一致 |
| 判别联合 | 检查公共字面量属性 | 消息协议、状态机 | 需先正确"分家"编码 |
| `never` 穷举 | default 中赋值给 never | switch 必须全覆盖 | 新增成员会编译报错 |

## 实践建议

1. **优先让代码"长得像 JS"**：别急着写类型体操，先用 `typeof`/真值/`in` 这些自然检查，让编译器自己收窄。
2. **对原始类型做真值检查前，过一遍 falsy 表**：`""`、`0`、`NaN` 都会掉进 else。
3. **数据建模首选判别联合**：用"联合 + 必填字面量 kind"取代"可选属性 + 非空断言"，把知识写进类型而不是写进 `!`。
4. **switch 处理联合时加 `never` default**：这是零成本的穷举保险，新增分支时编译器主动报警。
5. **过滤数组用类型谓词函数**：`zoo.filter(isFish)` 比 `filter(x => x.swim) as Fish[]` 更类型安全、可复用。

## 小结

Narrowing 是 TypeScript"静态类型叠加在动态语言之上"哲学的最佳展示：它不要求你换一种写法，只是**沿着你已经写下的 if/else/switch，把类型的可能性一条条收窄**。从 typeof、真值这些原生守卫，到类型谓词、判别联合这类主动建模手段，再到 never 穷举检查这道保险——下一篇我们进入函数的进阶话题：签名、上下文类型化、泛型函数与重载。
