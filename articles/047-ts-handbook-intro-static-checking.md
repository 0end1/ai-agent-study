# TypeScript 学习笔记（1/16）：开篇——类型是什么，静态检查到底在查什么

> 对应书源：TypeScript Handbook「About this Handbook」+「The Basics」前半（Static type-checking / Non-exception Failures / Types for Tooling）｜系列第 1 篇

## 开篇：给 JavaScript 装一盏「前置探照灯」

想象你在一条没有路灯的山路上夜跑。JavaScript 的开发体验就是这样——你可以跑得飞快，但脚下的坑只有踩上去才知道。脚本跑起来、用户点下去、线上崩了，控制台甩给你一句 `TypeError: xxx is not a function`，这时你才知道：哦，这里传错了。

TypeScript 做的事，是在你出发前先派一架无人机把这条路照一遍。它不替你跑，也不改变路线，只是提前告诉你「第 3 公里有个坑」。官方 Handbook 对它的定义非常克制：**TypeScript 是一个 JavaScript 程序的静态类型检查器（static typechecker）**——在你代码运行之前（static）运行，确保程序里的类型是正确的一（typechecked）。

这一篇是系列的开场，我们要回答三个问题：这本 Handbook 讲了什么、类型到底是什么、以及 TypeScript 究竟能在运行前抓住哪些错误。

## 一、Handbook 的定位：指南，不是规范

Handbook 开篇就把边界说清楚了，这比急着学语法更重要：

| 维度 | Handbook | Reference（参考页） |
| --- | --- | --- |
| 目标 | 面向日常开发者的完整指南 | 对单个概念的深入解释 |
| 读法 | 从左到右按顺序读 | 可随意跳读，不追求连续性 |
| 覆盖度 | 讲清主要特性与行为，略过边角 | 精确、形式化地描述行为 |

读完之后你应该能做到三件事：**读懂常见的 TypeScript 语法与模式**、**解释重要编译选项的作用**、**在大多数情况下正确预测类型系统的行为**。

同时它明确列出了「非目标」（Non-Goals），很有意思：

- 不从头教 JavaScript 基础（函数、类、闭包）——需要时会给外链；
- 不是语言规范——为了可读性会跳过形式化描述和边角案例；
- 不讲与构建工具的集成（webpack、babel、react、vue……）——那些在别处。

也就是说，这是一本「几小时能读完」的务实指南，而不是一本字典。

## 二、类型是什么：从 `message.toLowerCase()` 说起

Handbook 的第一课不是语法，而是一个思想实验。看这段 JavaScript：

```js
message.toLowerCase(); // 访问属性并调用
message();             // 直接调用
```

如果不知道 `message` 是什么值，没人能断言这两行的结果。它在运行时取决于四个问题：`message` 可调用吗？它有 `toLowerCase` 属性吗？该属性可调用吗？返回值又是什么？

```js
const message = "Hello World!";
message.toLowerCase(); // 正常，返回 "hello world!"
message();             // TypeError: message is not a function
```

关键点在于：**JavaScript 运行时是靠「值的类型」来决定行为的**。对 `string`、`number` 这类原始类型，我们还能用 `typeof` 在运行时问一句；但对函数这种东西，运行时根本没有对应的机制来告诉你「这个参数需要有 `flip` 方法」：

```js
function fn(x) {
  return x.flip();
}
```

这段代码读起来很清楚——`x` 必须是一个带 `flip` 方法的对象。但纯 JavaScript 里，唯一确认它能不能跑的办法就是**真的去调用它**。这就是**动态类型**：运行代码，然后看会发生什么。

TypeScript 给出的另一种选择是**静态类型系统**：在代码运行之前，就预测它应该做什么。把那句拗口的话翻译过来——**类型，就是描述「哪些值可以传给 `fn`、哪些会崩」的一套语言**。

## 三、静态类型检查：把 bug 挪到运行之前

为什么不早点发现？Handbook 给的理由很实在：就算你改完代码立刻重跑，也可能没测到那条分支；就算侥幸撞上了，中间可能已经堆了一大堆新代码，排查成本极高。

于是，`const message = "hello!"; message();` 这段，TypeScript 在你保存文件的那一刻就报 `This expression is not callable`（错误码 2349）——**代码还没跑，问题已经出现在编辑器里**。

## 四、真正的杀手锏：非异常失败（Non-exception Failures）

这是本篇最值得记住的一节。前面讲的都是「运行时会抛错」的场景，但 JavaScript 里有大量**不抛异常、却明显是 bug** 的代码。ECMAScript 规范规定：调用不可调用的东西要抛错；但访问对象上不存在的属性，返回的是 `undefined`。

```js
const user = { name: "Daniel", age: 26 };
user.location; // 不报错，返回 undefined
```

TypeScript 会明确报错：`Property 'location' does not exist`（2339）。**它把「合法但可疑」的 JavaScript 也纳入了检查范围**——这是一种取舍：牺牲一点表达自由度，换取大量真实 bug 被提前拦下。Handbook 举了三类典型：

| 错误类型 | 示例代码 | TypeScript 的判断 |
| --- | --- | --- |
| 拼写错误（typo） | `announcement.toLocalLowerCase()` | 方法不存在，直接标红（正确写法是 `toLocaleLowerCase`） |
| 忘记调用函数 | `Math.random < 0.5` | 比较一个函数与数字，无意义（应为 `Math.random()`） |
| 基础逻辑错误 | `value !== "a"` 与 `value === "b"` 分支 | 当 `value` 只能是 `"a" \| "b"` 时，后者不可达 |

第三类尤其惊艳：

```ts
const value = Math.random() < 0.5 ? "a" : "b";
if (value !== "a") {
  // ...
} else if (value === "b") {
  // Oops, unreachable
}
```

TypeScript 知道 `value` 的类型是 `"a" | "b"`，`else` 分支里它必然是 `"a"`，所以 `value === "b"` 永远不会成立——这种"死的分支"，纯靠肉眼 review 极容易漏掉。

## 五、类型不只是查错，还能让你少犯错

如果说查错是"事后拦截"，那 tooling 就是"事前预防"。类型检查器既然知道你正在操作的对象有哪些属性，它就能：

- **自动补全**：输入 `res.sen` 时提示 `send`；
- **快速修复（quick fixes）**：自动改正某些错误；
- **重构与导航**：重命名、整理代码、跳转到定义、查找所有引用。

```ts
import express from "express";
const app = express();

app.get("/", function (req, res) {
  res.sen // ← 编辑器在这里就能提示 send / sendFile / sendStatus
});
```

这些能力全部构建在同一个类型检查器之上，而且跨平台——你常用的编辑器基本都有 TypeScript 支持。**"类型"在这里从约束变成了生产力工具**：它不只是告诉你哪里错了，还在你写的时候就把正确选项递到手边。

## 实践建议

1. **先建立心智模型，再记语法**：把"类型 = 描述值能做什么"这句话刻在脑子里，后面学泛型、条件类型时都会轻松很多——它们只是把这句话参数化。
2. **优先解决"非异常失败"类报错**：`undefined` 属性、拼错的方法名、忘加括号的函数调用，这三类是 TypeScript 性价比最高的收益点，迁移老项目时先盯它们。
3. **让编辑器替你干活**：补全、跳转定义、查找引用、quick fix——这些不是"锦上添花"，而是你付了类型标注成本后应得的回报，值得花时间熟悉编辑器的 TS 快捷键。

## 小结

Handbook 的开篇没有急着教 `string` 和 `number`，而是先讲清 TypeScript 的立场：**它是一个在代码运行前工作的静态检查器，目标是抓住"类型用错了"这一类最常见的 bug**。它不仅能拦住会抛异常的错误，更能拦住那些 JavaScript 默许、但显然不对的写法（访问不存在的属性、拼错方法、函数忘了调用、不可达分支）；而它收集到的类型信息，又会立刻变成编辑器的补全与重构能力。

下一篇我们真正上手：`tsc` 编译器怎么用、类型注解写了之后去了哪（类型擦除）、模板字符串为什么被改写成 `concat`（降级编译），以及 `strict`、`noImplicitAny`、`strictNullChecks` 这三档"严格性旋钮"该怎么拧。
