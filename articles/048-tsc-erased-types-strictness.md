# TypeScript 学习笔记（2/16）：tsc、类型擦除与严格性旋钮

> 对应书源：TypeScript Handbook「The Basics」后半（tsc / Emitting with Errors / Explicit Types / Erased Types / Downleveling / Strictness）｜系列第 2 篇

## 开篇：一位身兼两职的翻译官

上一篇我们搞清了 TypeScript 的立场：一个在代码运行前工作的静态检查器。但有个问题悬而未决——浏览器根本不认识 `.ts` 文件，你的类型注解要送到哪里去？

答案是 `tsc`，TypeScript 编译器。它像一位身兼两职的翻译官：**第一职是安检**——在翻译前把你的稿子从头到尾审一遍，发现"类型用错了"就当面指出；**第二职才是翻译**——把 TypeScript 代码转写（transform）成等价的 JavaScript，让你能真正跑起来。有意思的是，这两项职责是**解耦**的：安检发现了问题，翻译照做不误；你还可以指定"翻译成哪个年代的 JavaScript"。本篇就把这两条线都走一遍。

## 一、初识 tsc：一场"什么都没发生"的仪式

先安装再开跑：

```sh
npm install -g typescript
```

进入空文件夹，写下第一个 TypeScript 程序 `hello.ts`：

```ts
// Greets the world.
console.log("Hello world!");
```

没错——它和 JavaScript 写法**一模一样**，没有任何花哨的东西。运行 `tsc hello.ts`，控制台一片寂静，什么也没输出。这是因为它没发现类型错误，自然无话可说；但回头一看，目录里多了个 `hello.js`——这就是编译产物，内容几乎和源文件相同。

这里藏着一个容易被忽略的设计目标：**tsc 努力输出"看起来像人写的"干净代码**——缩进一致、尊重换行、尽量保留注释。它不是把你的代码搅碎重排，而是做一次礼貌的转写。

那么，如果代码真有问题呢？把 `hello.ts` 改成：

```ts
// This is an industrial-grade general-purpose greeter function:
function greet(person, date) {
  console.log(`Hello ${person}, today is ${date}!`);
}

greet("Brendan");
```

再跑 `tsc hello.ts`，这次命令行报错了：

```txt
Expected 2 arguments, but got 1.
```

注意一个细节：到目前为止我们写的**全是标准 JavaScript**，一个类型注解都没加，但类型检查照样抓出了漏传参数的 bug。这就是上一说的延续——TypeScript 的检查能力从 JavaScript 代码本身的形状开始生效。

## 二、带错也照常输出：TypeScript 的核心价值观

刚才的例子还有个更微妙的现象：虽然报了错，`hello.js` **依然被更新了**。查错归查错，翻译照做。

这不是 bug，而是 TypeScript 的一条核心价值观：**大多数时候，你比 TypeScript 更清楚自己在干什么**。最典型的场景是从 JavaScript 迁移到 TypeScript：代码本来跑得好好的，一转成 `.ts` 冒出一堆类型错误，难道迁移就该让程序停下来吗？不该。所以 tsc 默认"报错但不拦路"。

当你想更严格时，有一条开关可用：

```sh
tsc --noEmitOnError hello.ts
```

加上这个标志后，只要存在类型错误，`hello.js` 就**永远不会被更新**。两种策略的取舍如下：

| 策略 | 行为 | 适用场景 |
| --- | --- | --- |
| 默认 | 报错，但照常输出 JS | 渐进迁移、原型探索 |
| `--noEmitOnError` | 有错就不输出 | 严谨项目、CI 构建 |

## 三、显式类型与推断：什么时候该写注解

回到 `greet`，给它补上类型：

```ts
function greet(person: string, date: Date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}
```

`: string` 和 `: Date` 叫**类型注解（type annotations）**，读作"greet 接收一个 string 类型的 person 和一个 Date 类型的 date"。有了它，TypeScript 立刻能发现另一处调用错误：

```ts
greet("Maddison", Date()); // 报错！
```

这个报错相当反直觉——`Date()` 不就是造日期吗？还真不是。**不带 `new` 直接调用 `Date()` 返回的是字符串**，要拿到 Date 对象必须写 `new Date()`。没有类型系统时，这个坑要等运行时才炸；现在编辑器当场标红。

不过 Handbook 紧接着泼了盆"别过度"的冷水：

```ts
let msg = "hello there!";
//  ^? 鼠标悬停可见类型是 string
```

即使不写注解，TypeScript 也能**推断（infer）**出 `msg` 是 string。它的建议很明确：**如果类型系统推出来的结果和你打算写的注解一模一样，那就别写了**——注解应该用在你需要"说出意图"的地方（比如函数参数），而不是给每个变量都贴标签。

## 四、类型擦除与降级编译：注解去了哪

把带注解的 `greet` 交给 tsc，输出是：

```js
function greet(person, date) {
  console.log("Hello ".concat(person, ", today is ").concat(date.toDateString(), "!"));
}
```

和源码相比有两处变化，各代表一条重要机制：

**变化一：注解全部消失**。`person: string` 变回了 `person`。因为类型注解不是 JavaScript（严谨地说是 ECMAScript）的一部分，没有任何浏览器能直接运行它——这正是 TypeScript 需要编译器的根本原因：**把 TypeScript 特有的代码剥掉或转写掉**。大多数 TS 特有代码都会被这样"擦除"（erased）。Handbook 为此给出了一句值得抄在显眼处的备忘：

> **记住：类型注解永远不会改变程序的运行时行为。**

这句话是理解 TypeScript 性质的钥匙——它加的是"安检"，不是"发动机"。指望用类型做运行时校验（比如校验接口返回值）是常见误区。

**变化二：模板字符串被改写成 `.concat()` 拼接**。这是**降级编译（downleveling）**：把新版 ECMAScript 的语法改写成旧版（如 ES3/ES5）能跑的形式。默认 target 是 ES5——一个极其古老的版本。可以用 `--target` 指定：

```sh
tsc --target es2015 hello.ts
```

输出就保留了原生模板字符串。Handbook 顺带提醒：绝大多数现代浏览器都支持 ES2015，**除非要兼容古老浏览器，否则放心把 target 设为 ES2015 或更高**。

## 五、严格性旋钮：从"开关"到"旋钮"

最后一个话题是所有 TypeScript 项目的灵魂设置。不同用户对检查器的期待不同：

- 有人要**宽松**：类型可选、推断取最宽松的类型、不检查 `null`/`undefined`——这是默认体验，专为"别挡我的路"设计，迁移老项目时是理想的起点；
- 有人要**严格**：让 TypeScript 一开始就尽力验证一切。

关键洞察在于：**严格性不是一个开关，而是一个旋钮**。拧得越紧，TypeScript 替你检查得越多，代价是需要额外的工作——但长期看是划算的，还能换来更精确的工具支持。新代码库应当**始终把严格检查打开**。

`tsconfig.json` 里的一句 `"strict": true`（或命令行的 `--strict`）能同时拧开所有严格选项，也可以逐个单独关闭。其中最重要的两个：

| 选项 | 解决什么问题 |
| --- | --- |
| `noImplicitAny` | 有些地方 TS 推断不出类型会退回最宽松的 `any`（等于回到纯 JavaScript 体验）；开启后，任何被**隐式**推断为 `any` 的变量都会报错 |
| `strictNullChecks` | 默认 `null`/`undefined` 可以赋给任何类型，方便但危险——忘了处理它们是无数线上事故的根源，被称为"十亿美元的错误"（billion dollar mistake）；开启后必须显式处理这两种值 |

`any` 的问题说得再直白些：用了它，TypeScript 对那个值就"失明"了——检查和补全都失效，用得越多，用 TypeScript 的意义越少。

## 实践建议

1. **新项目一律 `"strict": true`**：这是 Handbook 的原话级建议（"新代码库应当始终开启严格检查"）。迁移老项目可以先默认宽松跑通，再逐步拧紧旋钮。
2. **注解写在"刀刃"上**：函数参数、公开 API 边界处写注解；局部变量交给推断。判断标准很简单——推出来的类型和你要写的一样吗？一样就删掉注解。
3. **记牢两个"反直觉"**：`Date()` 返回字符串、类型注解不影响运行时行为。前者是日常 bug 源，后者决定了"运行时校验请用 zod 之类的库，别指望类型系统"。

## 小结

本篇揭开了编译器的面纱：`tsc` 是安检员兼翻译官——检查代码类型，并把 `.ts` 转写成 JS；默认"报错也照常输出"体现的是"你比它更懂"的价值观，`--noEmitOnError` 可以关掉这份宽容。类型注解在编译时被**擦除**，永远不改变运行时行为；旧语法会被**降级编译**成 target 版本可跑的形式。最后，严格性是旋钮不是开关，`strict` 一键全开，`noImplicitAny` 堵住 `any` 的暗门，`strictNullChecks` 管住"十亿美元的错误"。

下一篇进入干货最密的《Everyday Types》上篇：string/number/boolean、数组、声名狼藉的 `any`、变量注解，以及函数类型的写法。
