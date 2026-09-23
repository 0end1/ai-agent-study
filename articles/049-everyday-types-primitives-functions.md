# TypeScript 学习笔记（3/16）：日常类型（上）——string、any 与"少写注解"的智慧

> 对应书源：TypeScript Handbook「Everyday Types」前半（The primitives / Arrays / any / Type Annotations on Variables / Functions）｜系列第 3 篇

## 开篇：先背常用词，别急着啃词典

学英语的人都知道，最常用的 2000 个单词能覆盖 80% 的日常对话。类型系统也一样——概念图再宏伟，日常开发打交道的就是那一小撮：字符串、数字、布尔、数组、函数。Handbook 这一章的标题起得很诚实：**Everyday Types（日常类型）**，讲的就是你每天要摸的词汇表。

不过开篇先立一个原则，它会贯穿全篇：**"少写注解"比"多写注解"更接近 TypeScript 的正确用法**。本章所有小节都会反复印证这一点。

## 一、三块基石：string、number、boolean

JavaScript 最常用的三个原始类型（primitives），在 TypeScript 里各有一个同名类型，名字正好与 `typeof` 运算符的返回值一致：

| 类型 | 表示的值 | 注意点 |
| --- | --- | --- |
| `string` | `"Hello, world"` 等字符串 | — |
| `number` | `42` 等所有数字 | JS 没有独立的整数运行时值，**没有 `int`/`float` 之分**，一切皆 `number` |
| `boolean` | `true` / `false` | — |

这里埋着一个经典陷阱：**`String`、`Number`、`Boolean`（大写开头）虽然合法，但它们是极罕见的特殊内置类型，永远应该用小写的 `string`、`number`、`boolean`**。这条规则能帮你避开一大类莫名其妙的类型报错。

另外，`number` 的设计提醒我们：TypeScript 描述的是 **JavaScript 运行时的真实样子**，而不是 C/Java 式的想象。JS 里没有整数类型，类型系统就不假装有。

## 二、数组：两种写法，一个意思

给 `[1, 2, 3]` 这样的数组标类型，用 `number[]`；任何类型都适用（`string[]` 就是字符串数组）。等价写法是 `Array<number>`——这种 `T<U>` 形式就是泛型语法，后面第 58 篇会专门讲。

```ts
const nums: number[] = [1, 2, 3];
const names: Array<string> = ["Alice", "Bob"]; // 与 string[] 等价
```

注意一个细节：**`[number]` 是完全不同的东西**——那是元组（tuple），表示"固定长度、每个位置类型确定"的数组，后续讲对象类型时会展开。少一个方括号的差别，语义天壤之别。

## 三、any：逃生舱，不是邀请函

`any` 是 TypeScript 的"免检通道"：一个值一旦被标记为 `any`，它就退出了类型检查——访问任意属性、当函数调用、赋给任何类型，只要语法合法统统放行：

```ts
let obj: any = { x: 0 };
// 下面这些全都不会报编译错误：
obj.foo();             // 调用不存在的方法
obj();                 // 把对象当函数调用
obj.bar = 100;
obj = "hello";         // 随便改类型
const n: number = obj; // 随便赋值
```

Handbook 说得公道：`any` 的用途是"你不想为了说服 TypeScript 而写一大长串类型"的场景。但它的代价在第 048 篇已经预告过——**对 `any` 的每次使用，都是类型系统的一次失明**：检查失效、补全失效，等于暂时退回纯 JavaScript。

与之配套的是 `noImplicitAny`：当你没写类型、TypeScript 又推不出来时，编译器默认把该变量当作 `any`。开启这个 flag 后，**所有"隐式 any"都会被标记为错误**——它防的不是你显式写的逃生舱，而是那些你没意识到、悄悄漏进来的失明。

## 四、变量注解：能不写就不写

`const`/`var`/`let` 声明变量时都可以加类型注解：

```ts
let myName: string = "Alice";
//        ^^^^^^^^ 类型注解
```

两条规则值得记下：

1. **注解永远写在被标注对象的后面**——TypeScript 不用 `int x = 0;` 这种"类型在左"的声明风格；
2. **绝大多数情况下不需要写**。TypeScript 会根据初始化值自动推断，`let myName = "Alice"` 里的 `myName` 已经被推断为 `string`，注解是冗余的。

Handbook 给初学者的建议非常反直觉：**试着比你以为需要的写更少的注解**——你可能会惊讶，TypeScript 靠这么少的标注就能完全理解你的代码。推断规则不需要专门背，写多了自然有体感。

## 五、函数：类型的主战场

函数是 JavaScript 传递数据的主要手段，也是类型注解投入产出比最高的地方——因为它描述的是**代码边界的契约**。

**参数注解**写在参数名后面：

```ts
function greet(name: string) {
  console.log("Hello, " + name.toUpperCase() + "!!");
}

greet(42); // 报错：实参 number 不能赋给 string 参数
```

一个贴心细节：**就算参数不写注解，TypeScript 仍会检查实参数量对不对**——检查能力不全靠注解。

**返回值注解**写在参数列表之后：

```ts
function getFavoriteNumber(): number {
  return 26;
}
```

但和变量注解一样，返回值通常也不用写：TypeScript 会根据 `return` 语句推断返回类型，上面的 `: number` 什么也没改变。那什么时候写？Handbook 点了三种动机：**当文档用**（让读者一眼看懂契约）、**防止意外改动**（改坏了返回类型立刻报错）、或者纯粹的个人/团队偏好。

异步函数返回 Promise 时，用 `Promise` 类型包裹：

```ts
async function getFavoriteNumber(): Promise<number> {
  return 26;
}
```

## 六、匿名函数与上下文类型化：最聪明的"免费午餐"

匿名函数的行为和函数声明有点不一样，也是本章最精彩的设计：**当函数出现在 TypeScript 能确定它将如何被调用的位置时，参数会自动获得类型**——

```ts
const names = ["Alice", "Bob", "Eve"];

// 参数 s 没有注解，但被推断为 string
names.forEach(function (s) {
  console.log(s.toUpperCase());
});

// 箭头函数同样适用
names.forEach((s) => {
  console.log(s.toUpperCase());
});
```

`forEach` 的签名告诉 TypeScript"回调会收到一个字符串"，再结合 `names` 被推断为 `string[]`，`s` 的类型就不写自明了。这个过程叫**上下文类型化（contextual typing）**——函数所处的**上下文**决定了它的参数该是什么类型。

你不需要弄懂它的实现机制，但必须知道它存在：**下次想给回调参数写注解前，先停一停——上下文大概率已经替你写好了**。

## 三种"注解策略"速查

| 位置 | 建议写注解吗 | 原因 |
| --- | --- | --- |
| 局部变量（有初始化值） | 不写 | 从初始化值推断，写了是冗余 |
| 函数参数 | 写 | 调用方在别处，推断不了，这是契约 |
| 函数返回值 | 视情况 | 能推断；当文档、防意外改动时写 |
| 回调/匿名函数参数 | 不写 | 上下文类型化自动搞定 |

可以看到一条清晰的规律：**注解是写给"推断够不着的地方"的**——变量和回调离上下文近，推断够得着；函数参数是边界，必须靠人来说明。

## 实践建议

1. **从小写类型开始洁癖**：`string`/`number`/`boolean` 永远小写，见到 `String` 直接怀疑是 bug 或旧代码；也别在类型层面纠结"整数"——JS 里没有。
2. **把 `any` 当逃生舱而非默认选项**：只有当写出完整类型成本明显过高时才用，且优先考虑能否用 `unknown` + 收窄（后面会讲）；同时确保 `noImplicitAny` 开着，堵住隐式失明。
3. **注解三问**：写注解前问自己——推断得出吗（不写）？是代码边界吗（写）？写了对读者有价值吗（写）？

## 小结

本篇认识了类型词汇表的前半部分：`string`/`number`/`boolean` 三块基石（永远小写）、`number[]` 与 `Array<number>` 两种数组写法（`[number]` 是元组，别混）、以及需要敬而远之的 `any`（免检通道，隐式的更危险）。更重要的是确立了一个心法：**TypeScript 的类型系统以推断为先，注解只该出现在"函数参数"这类推断够不着的边界上**——匿名函数的上下文类型化更是把这一点做到了极致，回调参数连写都不用写。

下一篇继续《Everyday Types》后半：对象类型与可选属性（`last?: string` 与 `undefined` 检查）、联合类型（`number | string` 与"取交集的性质"）、类型别名、接口，以及 type 与 interface 的分野。
