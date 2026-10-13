# TypeScript 学习笔记（9/17）：泛型对象与元组——容器类型的完成态

> 对应书源：TypeScript Handbook「Object Types」后半（Generic Object Types / Array / ReadonlyArray / Tuple Types）｜系列第 9 篇

## 开篇：一个盒子的四代进化史

假设要设计一个"什么都能装"的 `Box` 类型。最朴素的第一代用 `any`：

```ts
interface Box {
  contents: any;
}
```

能用，但 `box.contents.toLowerCase()` 这类危险调用编译器一概放行——事故埋在将来。第二代换成 `unknown`：

```ts
interface Box { contents: unknown; }
if (typeof x.contents === "string") { x.contents.toLowerCase(); } // 每次都要收窄
console.log((x.contents as string).toLowerCase());                 // 或者每次断言
```

安全了，但明知是字符串还得层层检查，写起来很累。第三代追求精确——为每种内容造一个专用盒子：

```ts
interface NumberBox  { contents: number;  }
interface StringBox  { contents: string;  }
interface BooleanBox { contents: boolean; }

function setContents(box: StringBox, newContents: string): void;
function setContents(box: NumberBox, newContents: number): void;
function setContents(box: BooleanBox, newContents: boolean): void;
function setContents(box: { contents: any }, newContents: any) { /* ... */ }
```

类型安全了，代价是**样板代码爆炸**：每来一种新类型就要加一个接口加一条重载。而它们本质上明明是同一个东西。

第四代，也就是正解——**泛型对象类型**，给类型声明一个"占位符"：

```ts
interface Box<Type> {
  contents: Type;
}

let box: Box<string>;       // Box<string> ≡ { contents: string }
type AppleBox = Box<Apple>; // 新类型？不用新接口，换个实参就行
```

读作"装 `Type` 的盒子，其 contents 是 `Type` 类型"。`Type` 是占位符，TS 看到 `Box<string>` 就把它替换进模板——泛型类型是"真实类型的模板"，实参替换是它唯一的工作方式。原来三条重载也随之浓缩成一个泛型函数：

```ts
function setContents<Type>(box: Box<Type>, newContents: Type) {
  box.contents = newContents;
}
```

`Type` 同时出现在参数和返回位置——又见"类型参数出现两次"的法则。顺带一提，**类型别名同样能带参数**，而且因为别名能描述非对象类型，还能写出这样的泛型工具：

```ts
type OrNull<Type> = Type | null;
type OneOrMany<Type> = Type | Type[];
type OneOrManyOrNull<Type> = OrNull<OneOrMany<Type>>; // 可以像函数一样嵌套调用
type OneOrManyOrNullStrings = OneOrManyOrNull<string>; // string | null | string[]
```

## 二、原来 Array 一直是个泛型

容器类型天然适合泛型——它们的行为与装的东西无关。全 Handbook 里我们其实早就在用一个这样的类型：**`Array`**。`number[]`、`string[]` 只是 `Array<number>`、`Array<string>` 的**语法糖**：

```ts
interface Array<Type> {
  length: number;
  pop(): Type | undefined;
  push(...items: Type[]): number;
  // ...
}
```

`pop()` 返回 `Type | undefined`（数组可能为空）、`push` 接受 `Type[]`——每一处 `Type` 都和容器装的东西联动。同理，现代 JS 的 `Map<K, V>`、`Set<T>`、`Promise<T>` 也都是泛型容器，两个类型参数的 `Map` 说明泛型并不局限于一个占位符。

## 三、ReadonlyArray：用类型声明"只读意图"

`ReadonlyArray` 描述不应被修改的数组：可以读（索引、`slice`），不能改（`push`/`pop`/赋值全部报错）：

```ts
function doStuff(values: readonly string[]) { // ReadonlyArray<string> 的语法糖
  const copy = values.slice(); // ✅ 读没问题
  values.push("hello!");       // ❌ 不能变
}
```

它和属性的 `readonly` 一样是**意图工具**：函数返回 `ReadonlyArray` 意味着"别改我的返回值"；参数声明为 `ReadonlyArray` 意味着"放心传，我不会动你的数组"。比起靠口头约定或注释，类型直接写进了函数签名，IDE 悬浮提示一眼可见。三个实用细节：

1. **没有构造器**：不能 `new ReadonlyArray(...)`，但普通数组可以直接赋给它。
2. **可赋值性是单向的**：`string[]` 可以赋给 `readonly string[]`（缩小能力是安全的），**反过来不行**——把只读数组赋给可变数组，等于承诺"我不会改"却拿不到保证。注意这一点和 readonly **属性**不同（属性修饰符连单向性都没有，别名双向互穿）。
3. 与 `Array` 不同，它是纯类型概念，运行时无对应物。

## 四、元组类型：知道"每个位置是什么"的数组

普通数组只知道"装的都是什么"，**元组**（tuple）进一步知道"有几个、每个位置是什么"：

```ts
type StringNumberPair = [string, number];

function doSomething(pair: [string, number]) {
  const a = pair[0]; // string
  const b = pair[1]; // number
  //  const c = pair[2]; // ❌ 越界直接报错
  const [inputString, hash] = pair; // 支持数组解构
}
```

对类型系统而言，`[string, number]` 约等于一个这样的对象：

```ts
interface StringNumberPair {
  length: 2;      // length 是数字字面量类型！
  0: string;      // 索引即属性
  1: number;
  slice(start?: number, end?: number): Array<string | number>; // 其余成员同 Array
}
```

元组适合**约定驱动**的 API——每个位置的含义"不言自明"（如 `[经度, 纬度]`）。但书里也提醒：不是所有人对"显然"的理解一致，若含义不直观，**用带描述性属性名的对象往往更好**。权衡的关键在于解构时的命名自由度：元组让调用方自己给 `pair[0]`、`pair[1]` 起语义化名字，代价是放弃了属性名自带的文档性。

元组还能带**可选元素**（只能放尾部，且会影响 `length` 的类型）和 **rest 元素**（必须是数组/元组类型）：

```ts
type Either2dOr3d = [number, number, number?];       // length: 2 | 3
type StringNumberBooleans = [string, number, ...boolean[]];      // 前两个固定，后面任意个 boolean
type BooleansStringNumber = [...boolean[], string, number];      // 前面任意个 boolean，末尾固定
const a: StringNumberBooleans = ["hello", 1];        // rest 允许零个
const c: StringNumberBooleans = ["world", 3, true, false];
```

可选与 rest 元素的真正威力在于：**让元组和参数列表对应起来**——

```ts
function readButtonInput(...args: [string, number, ...boolean[]]) {
  const [name, version, ...input] = args;
}
// 基本等价于：
function readButtonInput(name: string, version: number, ...input: boolean[]) {}
```

当你想用 rest 参数接收"至少 N 个"参数、又不想引入中间变量时，元组类型是最短路径。

## 五、readonly 元组与 as const 的化学反应

元组同样有 `readonly` 变体：`readonly [string, number]`，任何位置写入都报错。由于元组大多"创建后即不再修改"，**给元组加 readonly 是个好的默认习惯**。这不只是风格问题——`as const` 断言会把数组字面量推断为 **readonly 元组**：

```ts
let point = [3, 4] as const;         // 类型是 readonly [3, 4]，不是 number[]！

function distanceFromOrigin([x, y]: [number, number]) {
  return Math.sqrt(x ** 2 + y ** 2);
}
distanceFromOrigin(point); // ❌ readonly [3, 4] 不能赋给 [number, number]
```

`distanceFromOrigin` 并没有修改元素，但它声明的 `[number, number]` **保留修改的可能性**，而 readonly 元组无法给出这个保证——于是被拒。呼应 053 篇的 `Math.atan2(...args)`：解法都是 `as const`，坑源都是"可变数组与只读元组之间不可逆的可赋值方向"。

## 四类数组型类型速查

| 类型 | 语法 | 长度 | 可变 | 典型用途 |
|------|------|------|------|----------|
| `Array<Type>` / `Type[]` | `string[]` | 任意 | 可变 | 通用列表 |
| `ReadonlyArray<Type>` / `readonly Type[]` | `readonly string[]` | 任意 | 只读 | 只读集合、防误改参数 |
| 元组 | `[string, number]` | 固定 | 可变 | 定长结构、对应参数列表 |
| readonly 元组 | `readonly [string, number]` | 固定 | 只读 | `as const` 的产物、约定数据 |

## 实践建议

1. **容器类型一律泛型化**：别再为每种内容类型复制接口 + 重载，一个 `Box<Type>` 一劳永逸。
2. **参数是"只读使用"就声明 `readonly Type[]`**：调用方随便传，你这边也不会误改——双赢。
3. **注意可赋值性的方向**：可变 → 只读 ✅，只读 → 可变 ❌；readonly 属性连单向保证都没有，别混淆。
4. **元组适合"含义显然"的定长结构**，否则用带属性名的对象更可读；越界访问在元组上是编译错误，这是它相对普通数组的最大红利。
5. **默认给元组加 `readonly`**，尤其当数据来自 `as const`；需要可变版本时再放宽。

## 小结

对象类型这条线至此走完：泛型对象给了容器"装什么都能保住类型"的能力，`Array`/`Map`/`Promise` 不过是它的现成范例；`ReadonlyArray` 与元组把"只读意图"和"定长定形"补进类型词汇。下一篇离开类型世界，进入工程视角——TS 的模块体系：ES Module 语法、CommonJS 兼容与模块解析。
