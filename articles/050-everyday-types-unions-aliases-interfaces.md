# TypeScript 学习笔记（4/17）：日常类型（中）——对象、联合、别名与接口的分野

> 对应书源：TypeScript Handbook「Everyday Types」中段（Object Types / Optional Properties / Union Types / Type Aliases / Interfaces）｜系列第 4 篇

## 开篇：从"单个值"到"组合零件"

上一篇我们攒齐了零散的词汇：string、number、boolean、数组。但真实程序里的数据从来不是散装零件——一个坐标是 `{ x: 3, y: 7 }`，一个 ID 可能是**数字也可能是字符串**。今天进入类型系统的真正魅力所在：**用已有类型，搭出新类型**。

Handbook 在讲完基础类型后有一句预告："类型远不止出现在类型注解里……我们会学到如何引用这些类型来构成新的结构。"这一篇就是这句话的兑现。

## 一、对象类型：把"有什么属性"写在脸上

除原始类型外，你最常打交道的就是**对象类型**——任何带属性的 JavaScript 值，几乎涵盖了一切。定义方式很直白：把属性和它们的类型列出来即可。

```ts
function printCoord(pt: { x: number; y: number }) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
printCoord({ x: 3, y: 7 });
```

三个细节：属性之间用 `,` 或 `;` 分隔都行（最后一个分隔符可省略）；**每个属性的类型部分是可选的，不写就默认是 `any`**——这是 `any` 又一个悄悄溜进来的入口，值得警惕。

### 可选属性：一个问号的分量

用一个 `?` 就能标记某属性可有可无：

```ts
function printName(obj: { first: string; last?: string }) {
  // ...
}
printName({ first: "Bob" });                     // OK
printName({ first: "Alice", last: "Alisson" });  // OK
```

但代价随之而来：JS 里访问不存在的属性只会得到 `undefined` 而非运行时错误，所以**读取可选属性前必须先处理 undefined**：

```ts
function printName(obj: { first: string; last?: string }) {
  console.log(obj.last.toUpperCase());   // 报错：'obj.last' 可能是 undefined
  if (obj.last !== undefined) {
    console.log(obj.last.toUpperCase()); // OK
  }
  console.log(obj.last?.toUpperCase());  // OK：可选链写法
}
```

记住这个连锁反应：**`?` 加在属性上，`undefined` 就长在使用方的路上**。它和 `strictNullChecks`（第 048 篇）是同一套思路的两个切面。

## 二、联合类型：允许"多选一"

光有单个类型不够表达现实世界，于是有了**联合类型（union type）**：由两个或更多类型组成的值，可以是其中**任意一种**，每个被联合的类型称为它的**成员（member）**。

```ts
function printId(id: number | string) {
  console.log("Your ID is: " + id);
}
printId(101);            // OK
printId("202");          // OK
printId({ myID: 22342 }); // 报错：对象不在成员列表里
```

成员多了可以换行写，`|` 也允许放在第一个元素前面：

```ts
function printTextOrNumberOrBool(
  textOrNumberOrBool:
    | string
    | number
    | boolean
) {
  console.log(textOrNumberOrBool);
}
```

### 关键规则：只允许对"全体成员都合法"的操作

这是新手最容易卡住的地方。你手握一个 `number | string` 时，**不能直接用只有 string 才有的方法**：

```ts
function printId(id: number | string) {
  console.log(id.toUpperCase()); // 报错：number 上没有 toUpperCase
}
```

解法是用代码把联合**收窄（narrow）**——和你在纯 JavaScript 里做判断一模一样，只不过 TypeScript 能看懂：

```ts
function printId(id: number | string) {
  if (typeof id === "string") {
    console.log(id.toUpperCase()); // 这个分支里 id 是 string
  } else {
    console.log(id);               // 这里 id 是 number
  }
}
```

注意 `else` 分支里我们**什么都没做**：既然不是 `string`，那它必然是 `number`。类似的还有 `Array.isArray`：

```ts
function welcomePeople(x: string[] | string) {
  if (Array.isArray(x)) {
    console.log("Hello, " + x.join(" and ")); // x: string[]
  } else {
    console.log("Welcome lone traveler " + x); // x: string
  }
}
```

还有个好消息：**如果联合的每个成员都有某个共同属性，不用收窄也能直接用**。`number[]` 和 `string` 都有 `slice`，于是：

```ts
function getFirstThree(x: number[] | string) {
  return x.slice(0, 3); // OK，返回值推断为 number[] | string
}
```

### 为什么"联合"却只能取"共同点"？

这一点确实反直觉，Handbook 专门用了一个精妙的类比解释：

> 一间屋子全是戴帽子的高个子，另一间屋子全是戴帽子的西班牙语使用者。把两间屋子的人合并到一间屋里，你唯一能确定适用于**每一个**人的事实，是**他们都戴着帽子**。

这就是类型理论里 "union" 的本意：**类型的联合，作用于"值"的集合取并集**，而关于它们的**事实（可用成员）却只能取交集**。读懂这段话，你就理解了为什么 `number | string` 上只剩两组大家都有的能力。

（收窄的完整方法——typeof 守卫、真值收窄、判别联合、`never` 等——是下一篇的主角。）

## 三、类型别名：给类型起个名字

一直把对象类型或联合类型直接写在注解里虽然方便，但同一个类型要复用多次时就该给它一个名字了。**类型别名（type alias）** 就是：**给任意类型起一个名字**。

```ts
type Point = {
  x: number;
  y: number;
};

// 和之前直接写匿名对象类型完全等价
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
```

它不止能给对象类型命名，联合类型同样可以：`type ID = number | string;`。

但要注意一条常被误解的规则：**别名就只是别名，它造不出"不同类型的版本"**：

```ts
type UserInputSanitizedString = string;

function sanitizeInput(str: string): UserInputSanitizedString {
  return sanitize(str);
}

let userInput = sanitizeInput(getInput());
userInput = "new input"; // 合法！因为它本质上就是 string
```

也就是说，**用了别名的地方，就等同于把被别名的类型原样抄了过去**——你不能用它来做类似 Go/Rust 里"新类型（newtype）"的品牌标记。

## 四、接口：另一种给对象类型命名的方式

**接口声明（interface declaration）** 也能命名对象类型：

```ts
interface Point {
  x: number;
  y: number;
}

function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
```

效果与用 `type` 完全一样。这里 Handbook 顺势点出了 TypeScript 的一个根本性格：**它只关心你传给 `printCoord` 的值的"结构"**——有没有我们要的那几个属性。至于它叫 `Point`、来自哪个库，一概不问。这种只看结构不看身份的机制，正是 TypeScript 被称为**结构化类型系统（structurally typed）**的原因。

## 五、type 与 interface：怎么选

两者高度相似，多数情况下可以随意选。但有一个**关键区别**：**接口可以被"重新打开"追加属性，`type` 一旦创建就不能再变**。

| 能力 | `interface` | `type` |
| --- | --- | --- |
| 扩展类型 | `interface Bear extends Animal { honey: boolean }` | `type Bear = Animal & { honey: boolean }`（交叉类型） |
| 给已有类型加字段 | 再声明一次同名 interface，自动**声明合并** | 同名重复声明 → **报错：Duplicate identifier** |
| 重命名原始类型 | 不可以 | 可以（`type ID = string`） |
| 错误信息中的呈现 | **总是**以原名出现 | TS 4.2 前可能显示为展开的匿名类型 |
| 编译器性能 | `extends` 通常优于交叉类型 | — |

Handbook 给的务实建议是：**先用 `interface`，等你需要 `type` 的独有能力时再用 `type`**。其余情况按个人或团队偏好来就行——真需要另一种声明时，TypeScript 会主动告诉你。

## 实践建议

1. **优先起名字**：对象类型/联合类型只要被复用两次以上就抽成 `type` 或 `interface`，别在多个函数签名里复制同一坨 `{ x: number; y: number }`。
2. **见到联合先想收窄**：写 `number | string` 时同步问自己"用它之前要不要判断"？记住"戴帽子的房间"——联合类型上只剩所有成员都有的能力。
3. **别把别名当新类型**：想要"品牌化"的类型（防止把两个语义不同的 string 混用），别名做不到，需要 branding 技巧或 `interface` + 独特属性。
4. **选 interface 起步**：对象形状用 `interface`；需要联合、交叉、重命名原始类型时再切换到 `type`。

## 小结

本篇完成了从"单个值"到"组合结构"的跨越：**对象类型**用 `{ x: number; y: number }` 描述形状，可选属性 `?` 带来必须处理 `undefined` 的义务；**联合类型** `number | string` 表达"多选一"，但使用时只能调用**所有成员共有的能力**——这正是"并集取值、交集取性质"的含义，解法是**收窄**（`typeof`、`Array.isArray`）；**类型别名**给任意类型起名字，但它只是"同义词"，造不出新类型；**接口**是另一种命名方式，可重复声明自动合并，这也是它和 `type` 的核心分歧——官方建议先用 `interface`，需要 `type` 的能力时再换。

下一篇继续《Everyday Types》收尾：类型断言 `as`、字面量类型与 `as const`、`null`/`undefined` 与 `strictNullChecks`、非空断言 `!`，以及那个"Know it exists, but maybe hold off"的枚举。
