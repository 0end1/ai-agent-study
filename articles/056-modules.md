# TypeScript 学习笔记（10/17）：Modules——文件边界的沟通术

> 对应书源：TypeScript Handbook「Modules」全篇（JS 模块定义 / ES Module 语法 / CommonJS / 模块解析与输出 / Namespaces）｜系列第 10 篇

## 开篇：从"方言时代"到书同文

JavaScript 的模块化史是一部方言史：从 IIFE 到 AMD、CommonJS、UMD，各路方案各说各话。TypeScript 2012 年诞生时不得不兼容大半个清单；而社区与规范最终在 2015 年收敛到 **ES Modules**（ES6 模块）——也就是你熟悉的 `import`/`export`。到 2020 年，浏览器和 JS 运行时已广泛支持。本篇聚焦两条主线：ES Modules 与它的前身 CommonJS（npm 上多数包仍以此交付），以及 TS 在两者之间的桥接术。

## 一、什么才算"模块"？

TS 与 ECMAScript 2015 一致的判定标准简单粗暴：**文件里有顶层 `import` 或 `export`，它就是模块；没有，它就是脚本（script）**。两者的命运截然不同：

- **模块**在自身作用域内执行——变量、函数、类对外不可见，除非显式导出；要用别人的东西，必须显式导入。
- **脚本**的内容全部落在共享的**全局作用域**，通常靠 HTML 里多个 `<script>` 标签按正确顺序加载，或用 `outFile` 选项拼接。

由此产生一个高频小坑：一个暂时没有任何导入导出的文件，会被当脚本处理。想让它按模块对待，只需加一行"什么都不导出"的声明：

```ts
export {}; // 与模块目标无关，任何配置下都生效
```

## 二、写模块代码前先想三个问题

TS 里写模块代码有三件事要考虑：

1. **语法**：用什么语法导入导出？
2. **模块解析（Module Resolution）**：模块名/路径与磁盘上文件的对应关系？
3. **模块输出目标（Module Output Target）**：编译出的 JS 模块长什么样？

### ES Module 语法

`export default` 声明主导出，`export`（不带 default）声明若干具名导出：

```ts
// maths.ts
export const pi = 3.14;
export function absolute(num: number) { /* ... */ }

// app.ts
import { pi, phi, absolute } from "./maths.js";
```

导入语法全家桶：

| 写法 | 含义 |
|------|------|
| `import { old as new } from "..."` | 导入并重命名（`pi as π`） |
| `import Default, { named } from "..."` | 默认导出与具名导出混合 |
| `import * as math from "..."` | 全部导出收进一个命名空间对象 |
| `import "./file"` | **副作用导入**：不引入任何变量，只执行该文件 |

副作用导入是容易被误解的一个：它"什么都没拿"，但 `maths.ts` 里的代码会被完整求值——可能触发影响其他对象的副作用（注册 polyfill、给原型挂方法、初始化全局配置等），这是 CSS-in-JS 样式表与某些 SDK 的标准接入方式。

## 三、TS 专属：类型也走同一条门

类型可以和值用完全相同的语法导入导出：

```ts
// animal.ts
export interface Dog { breeds: string[]; yearOfBirth: number; }

// app.ts
import { Cat, Dog } from "./animal.js";
type Animals = Cat | Dog;
```

但 TS 额外扩展出两种"声明我只导类型"的方式：

- **`import type`**：整条语句**只能**导入类型——用它导入 `createCatName` 这种函数会直接报错（错误 1361）。
- **inline `type` 导入**（TS 4.5+）：在一条 import 里逐个标记——`import { createCatName, type Cat, type Dog } from "./animal.js"`。

这两者的存在意义在**类型擦除**层面：Babel、swc、esbuild 这类不懂 TS 的转译器，需要靠 `type` 标记知道哪些导入可以在编译时安全移除——否则类型导入会被当成运行时依赖保留，产物里出现不存在的引用。

## 四、CommonJS 与互操作

CommonJS 的世界观简单：导出就是给全局的 `module` 对象设置 `exports` 属性，导入就是 `require`：

```ts
// 导出
module.exports = { pi: 3.14, squareTwo: 1.41, phi: 1.61, absolute };

// 导入（可整体收，也可解构）
const maths = require("./maths");
const { squareTwo } = require("./maths");
```

TS 提供了一条**与 CommonJS/AMD 的 `require` 直接对应**的 ES 语法桥：

```ts
import fs = require("fs");
fs.readFileSync("hello.ts", "utf8");
```

它保证 TS 源文件里的导入与编译出的 CommonJS 输出**一一对应**——在 Node 项目里用这条语法写导入，编译产物不会有任何额外包装层，调试时栈信息与源码完全对得上。

两种体系的根本错位在于"默认导入"与"命名空间对象导入"的语义差异：ES Modules 里 `export default` 与具名导出泾渭分明，而 CommonJS 的 `module.exports` 只是一个普通对象，两种世界观映射到一起必然产生别扭。TS 提供 `esModuleInterop` 编译选项来降低这种摩擦（比如让无 `default` 的 CommonJS 模块也能用默认导入语法）。

## 五、模块解析与模块输出：两组旋钮

**模块解析**是把 import 字符串翻译成具体文件的过程——你写 `"./maths.js"`，解析器要回答"磁盘上到底哪个文件对应这个名字"。TS 内置两种策略：

- **Classic**：`module` 不是 `commonjs` 时的默认值，为向后兼容而保留；
- **Node**：复刻 Node.js 在 CommonJS 模式下的查找规则（找文件、找目录、找 package.json），外加对 `.ts` 和 `.d.ts` 的额外查找。

影响解析的 tsconfig 旗标有 `moduleResolution`、`baseUrl`、`paths`、`rootDirs`。

**模块输出**则由两个选项共同决定：

| 选项 | 职责 |
|------|------|
| `target` | 决定哪些 JS 特性被**降级**（downlevel）以适配旧运行时 |
| `module` | 决定模块之间**用什么代码互相通信**（即模块加载器） |

`target` 由运行环境（最老支持的浏览器/Node 版本/Electron 等）决定；`module` 决定加载器——运行时靠它先定位并执行所有依赖，再执行模块本身。同一份源码在不同 `module` 下的产出对比：

```
import { valueOfPi } from "./constants.js";
export const twoPi = valueOfPi * 2;
```

- `ES2020`：输出与源码几乎一致（import/export 原样保留）；
- `CommonJS`：`require("./constants")` + `exports.twoPi`；
- `UMD`：包一层兼容 AMD 与全局变量的外壳。

> ES2020 模式下输出"就是"源码——这就是为什么现代项目常直接选它。

## 六、Namespaces：TS 的前朝遗物

TS 还有一种自有格式——**命名空间（namespaces）**，诞生早于 ES Modules 标准。它在编写复杂声明文件时仍有实用功能，DefinitelyTyped 上依然活跃。虽然未废弃，但其大部分能力 ES Modules 都已覆盖，**官方建议直接用 ES Modules**以对齐 JS 的方向。

## 实践建议

1. **新文件一律当模块写**：哪怕暂时没有导出，也补一行 `export {}`，避免被当脚本污染全局。
2. **只用类型时写 `import type` 或 inline `type`**：给 esbuild/swc 这类转译器明确的擦除信号，产物才干净。
3. **理解 `module` 与 `target` 的分工**：前者管模块加载器，后者管语言特性降级；别用 `target` 去解决模块问题。
4. **消费 CommonJS 包遇到默认导入怪象，先想到 `esModuleInterop`**，而不是手写 `* as` 去绕。
5. **别再起新的 namespace**：除声明文件场景外，一律用 ES Modules。

## 小结

模块篇把视角从"类型"拉回到"文件与文件如何通信"：一行 `export {}` 决定文件的命运；import 的五种形态覆盖从取值到副作用的全谱；`import type` 让类型擦除有了契约；Classic/Node 两种解析策略与 target/module 两组输出旋钮则是编译配置层的地图。下一篇进入面向对象：类（上）——字段、构造函数与可见性修饰符。
