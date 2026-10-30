# TypeScript 学习笔记（15/17）：索引访问与条件类型——类型世界的下标与 if

> 对应书源：TypeScript Handbook「Indexed Access Types」+「Conditional Types」全篇｜系列第 15 篇

## 开篇：类型体操的"取值"与"分支"

上一篇的 `keyof` 和 `typeof` 打开了 Type Manipulation 的大门，但光会"取键"还不够——类型体操需要两件更日常的兵器：**下标**（从类型里取出某个属性的类型）和**分支**（根据类型关系做出选择）。它们分别是索引访问类型与条件类型，组合起来威力远大于各自为战。

## 一、Indexed Access：`Type["a"]`，类型的下标运算

索引访问类型长得就像取对象属性，只是发生在类型世界：

```ts
type Person = { age: number; name: string; alive: boolean };
type Age = Person["age"]; // number
```

下标位置**本身也是类型**，所以联合、`keyof`、类型别名全都能放：

```ts
type I1 = Person["age" | "name"]; // number | string
type I2 = Person[keyof Person];   // number | string | boolean
type I3 = Person[AliveOrName];    // "alive" | "name" 是类型别名也行
```

要点速记：

| 写法 | 结果 |
|---|---|
| `Person["age"]` | 单个属性的类型 |
| `Person["age" \| "name"]` | 联合键 → 属性类型联合 |
| `Person[keyof Person]` | 全部属性类型联合 |
| `Person["alve"]`（拼错） | 错误 2339——不存在的键编译期就报 |

与 `typeof` 组合可以优雅捕获数组字面量的元素类型——`number` 作为下标取出数组元素类型：

```ts
const MyArray = [{ name: "Alice", age: 15 }, { name: "Bob", age: 23 }];
type Person = typeof MyArray[number];     // { name: string; age: number }
type Age = typeof MyArray[number]["age"]; // number
```

两个限制要分清：**下标只能是类型**——`const key = "age"; Person[key]` 报错 2538/2749；但类型别名可以模拟同样的重构姿势：`type key = "age"; type Age = Person[key]` ✅。

这组限制再次印证了 060 篇的"两个世界"框架：索引访问是**类型世界内部的运算**，值世界的变量（哪怕内容是字符串字面量）无法直接参与，必须先经 `typeof` 或类型别名"过桥"。初学者最常见的困惑就在这里——`const key = "age"` 看起来明明就是那个字符串，为什么不能用？因为类型位置需要的是类型层面的字面量 `"age"`，而非常量引用。理解了通道方向，这类报错就不再是玄学。

## 二、Conditional Types：类型系统的 if 语句

条件类型形式与 JS 三元表达式一致：`SomeType extends OtherType ? TrueType : FalseType`。左侧可赋值给右侧就走真分支，否则走假分支：

```ts
type Example1 = Dog extends Animal ? number : string;    // number
type Example2 = RegExp extends Animal ? number : string; // string
```

单看例子似乎没什么用——`Dog extends Animal` 我们自己就能判断。**条件类型的威力来自泛型**：把选择逻辑编码进类型参数。

教科书案例是压扁函数重载。原来的 `createLabel` 需要三条重载（string 一条、number 一条、联合一条），且每新增一种可处理类型，重载数量**指数增长**。条件类型一招制敌：

```ts
type NameOrId<T extends number | string> = T extends number ? IdLabel : NameLabel;

function createLabel<T extends number | string>(idOrName: T): NameOrId<T> {
  throw "unimplemented";
}
const a = createLabel("typescript"); // NameLabel
const b = createLabel(2.8);          // IdLabel
const c = createLabel(Math.random() ? "hello" : 42); // NameLabel | IdLabel
```

### 条件类型约束：真分支自带 narrowing

条件类型的真分支会进一步约束泛型——就像类型守卫收窄值一样。`MessageOf<T>` 想取 `T["message"]`，但 `T` 未必有 message 属性，硬取报错 2536。两种解法对比颇有味道：

```ts
// 解法一：直接约束 T——调用方传无 message 的类型直接报错
type MessageOf<T extends { message: unknown }> = T["message"];

// 解法二：放宽约束 + 条件兜底——没有 message 就给 never
type MessageOf<T> = T extends { message: unknown } ? T["message"] : never;
```

解法二里，**真分支中 TS 知道 `T` 一定有 message**，索引访问因此合法。同样的思路写出 `Flatten`：是数组就取元素类型，否则原样返回：

```ts
type Flatten<T> = T extends any[] ? T[number] : T;
type Str = Flatten<string[]>; // string
type Num = Flatten<number>;   // number
```

注意 `T[number]` 正是上一篇的索引访问——两个工具在此会师。

## 三、infer：让编译器替你"挖"类型

"加约束再手动提取"的模式太常见，于是有了 `infer`——在真分支里**声明式地**引入新类型变量，让 TS 自动推断匹配位置：

```ts
type Flatten<Type> = Type extends Array<infer Item> ? Item : Type;
```

不再需要思考"如何拆解类型的结构"，只声明"这里应该有个 Item"。用 `infer` 写一个自己的返回类型提取器：

```ts
type GetReturnType<Type> = Type extends (...args: never[]) => infer Return ? Return : never;
type Num = GetReturnType<() => number>;                            // number
type Bools = GetReturnType<(a: boolean, b: boolean) => boolean[]>; // boolean[]
```

一个冷知识：从带多个调用签名的类型（重载函数）做推断时，**只会从最后一个签名推断**——它通常是最宽容的兜底签名，TS 无法根据实参列表做重载决议，`ReturnType<typeof stringOrNum>` 得到的是最后一条签名的 `string | number`。

`infer` 的能力远不止提取返回类型：它可以出现在函数参数位置（实现 `Parameters`）、Promise 的泛型槽位（实现 `Awaited` 思路）、元组的任意位置（取首元素/去尾元素）——凡是你能"描述出形状"的地方，都能挖出那个未知位。它与 `keyof`/索引访问构成互补：后者是"已知键取已知值"，`infer` 则是"给未知留占位、由匹配反推"。工具类型库（type-fest 等）的绝大多数成员，骨架都是一条 `extends ... infer ... ? ... : ...`。

## 四、Distributive Conditional Types：联合会被"拆开发货"

条件类型作用于泛型时，若喂进去的是联合类型，会**分布到每个成员**分别计算再合并：

```ts
type ToArray<Type> = Type extends any ? Type[] : never;
type StrArrOrNumArr = ToArray<string | number>; // string[] | number[]
```

实际执行的是 `ToArray<string> | ToArray<number>`。分布性通常是期望行为——它让"逐成员转换联合"变得自然；想关掉它，给 extends 两侧套方括号：

```ts
type ToArrayNonDist<Type> = [Type] extends [any] ? Type[] : never;
type ArrOfStrOrNum = ToArrayNonDist<string | number>; // (string | number)[]，不再是联合的联合
```

| | 分布式（默认） | 非分布式 `[T] extends [U]` |
|---|---|---|
| 输入 `string \| number` | 逐成员计算 → `string[] \| number[]` | 整体计算 → `(string \| number)[]` |
| 典型用途 | 逐成员转换/过滤联合 | 整体判断（如排除 any、判 never） |

## 实践建议

1. **取属性类型用索引访问，别重写一遍类型**：`Person["age"]` 永远与源类型同步。
2. **数组元素类型用 `typeof arr[number]`**：从数组字面量一键提取，配合下标链 `["age"]` 深挖。
3. **重载爆炸时改写条件类型**：一个泛型签名替代 N 条重载，可扩展性天差地别。
4. **真分支里放心使用被检查的能力**：`T extends { message: unknown } ? T["message"] : never` 的 narrowing 是编译器背书的。
5. **理解默认分布，需要整体判断时套方括号**：这是排除 any/never 等整体判断的唯一正确姿势。

另一个值得内化的心智模型：条件类型的执行时机是**惰性的**——只有拿到具体类型实参时才会真正求值，所以未实例化的泛型条件类型在编辑器里显示为原样表达式而非结果。这也解释了为什么条件类型与泛型函数返回类型搭配时，IDE 能给出精确到分支的提示。

## 小结

本篇把类型体操的两个核心动作连成一线：索引访问负责"取"（属性、元素、与 keyof/typeof 组合成任意深度挖掘），条件类型负责"选"（泛型化的分支、真分支自动收窄），`infer` 把"取"自动化，分布式规则决定联合如何流经分支。至此，Partial/Readonly 等工具类型的实现原理已呼之欲出——下一篇映射类型正式揭晓。
