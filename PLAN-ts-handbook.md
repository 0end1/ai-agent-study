# TypeScript 官方 Handbook 学习笔记 · 大纲（16 篇）

- **书源**：[TypeScript Handbook (v2)](https://www.typescriptlang.org/docs/handbook/intro.html)
- **本地书源**：`book-src/ts-handbook/`（从 microsoft/TypeScript-Website 仓库 `v2` 分支拉取的源 Markdown）
- **编号**：047 - 062（接在 AI Agent 系列 001-013、企业 AI 系列 014-025、Rust 系列 026-046 之后）

| 序号 | 主题 | 书源文件 | 覆盖章节 |
|------|------|----------|----------|
| 047 | 开篇：类型是什么，静态检查到底在查什么 | The Handbook.md + Basics.md（前半） | About this Handbook / Static type-checking / Non-exception Failures / Types for Tooling |
| 048 | `tsc`、类型擦除、降级编译与严格性旋钮 | Basics.md（后半） | tsc / Emitting with Errors / Explicit Types / Erased Types / Downleveling / Strictness |
| 049 | 日常类型（上）：原始类型、数组、`any`、类型注解与函数类型 | Everyday Types.md（前半） | The primitives / Arrays / any / Type Annotations on Variables / Functions |
| 050 | 日常类型（下）：对象、联合、别名、接口、断言、字面量与枚举 | Everyday Types.md（后半） | Object Types / Union Types / Type Aliases / Interfaces / Type Assertions / Literal Types / Enums |
| 051 | 收窄（Narrowing）：让类型在分支中变具体 | Narrowing.md | typeof / 真值收窄 / 等值收窄 / in / instanceof / 控制流分析 / 判别联合 / never |
| 052 | 函数进阶：签名、上下文类型化、泛型函数与重载 | More on Functions.md | Function Type Expressions / Call Signatures / Construct Signatures / Generic Functions / Optional Parameters / Rest Params / Overloads |
| 053 | 对象类型：可选、只读、索引签名与泛型对象 | Object Types.md（前半） | Property Modifiers / Index Signatures / Excess Property Checks / Extending Types |
| 054 | 对象类型进阶：交叉、泛型对象与元组/数组只读 | Object Types.md（后半） + Everyday Types.md 元组部分 | Intersection Types / Generic Object Types / Array & Tuple Types / Readonly |
| 055 | 模块：ES Module、CommonJS 与 TS 的模块解析 | Modules.md | How JS Modules are Defined / ES Module Syntax / CommonJS Syntax / Module Resolution / Namespaces |
| 056 | 类（上）：字段、构造函数、`super`、可见性修饰符 | Classes.md（前半） | Class Members / Fields / Constructors / Super Calls / Methods / Getters / Setters |
| 057 | 类（下）：`implements`、`extends`、抽象类与继承陷阱 | Classes.md（后半） | Class Heritage / implements / extends / abstract / Visibility Modifiers / static |
| 058 | 泛型：类型层面的"参数" | Type Manipulation/Generics.md | Hello World of Generics / Generic Types / Generic Classes / Constraints |
| 059 | `keyof` 与 `typeof`：把值/键变成类型 | Type Manipulation/Keyof Type Operator.md + Typeof Type Operator.md | keyof / typeof / ReturnType |
| 060 | 索引访问类型与条件类型 | Type Manipulation/Indexed Access Types.md + Conditional Types.md | Indexed Access / Conditional Types / infer / Distributive Conditional Types |
| 061 | 映射类型与模板字面量类型 | Type Manipulation/Mapped Types.md + Template Literal Types.md | Mapped Types / Key Remapping / Template Literal Types / Intrinsic String Manipulation |
| 062 | 类型声明文件与读懂报错（收尾） | Type Declarations.md + Understanding Errors.md | .d.ts / DefinitelyTyped / Error categories / 收尾总结 |

## 维护约定

1. 每篇 1500-2500 字，结构：开篇引入 → 核心概念通俗解析 → 对比表格 → 代码示例 → 实践建议与总结。
2. 内容必须来自 `book-src/ts-handbook/` 的真实章节，禁止编造。
3. 每篇同步到 `0end1.github.io/_posts/`（Jekyll frontmatter：`layout: post`、`title`、`date`、`tags`）。
4. 单篇严格按上表顺序推进，一次只写一篇。
