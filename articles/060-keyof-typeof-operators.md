# TypeScript 学习笔记（14/17）：keyof 与 typeof——从值和键里"长"出类型

> 对应书源：TypeScript Handbook「Creating Types from Types」总览 +「Keyof Type Operator」+「Typeof Type Operator」｜系列第 14 篇，Type Manipulation 系列开篇

## 开篇：类型系统也有"复印机"

前 13 篇我们一直在**声明**类型：这里一个 interface、那里一个 type alias。但 TS 类型系统的真正威力在于**用已有类型表达新类型**——类型可以由其他类型推导而来，甚至由**值**推导而来。TS 官方把这套能力称为 Type Manipulation，工具箱包括：泛型（带参数的类型）、keyof、typeof、索引访问、条件类型、映射类型、模板字面量类型。本篇先拿下最基础的两个操作符：**keyof** 与 **typeof**——一个从类型里抽出键，一个从值里抽出类型。

## 一、keyof：把对象的键变成类型

`keyof` 接受一个对象类型，产出其**键名字面量的联合类型**：

```ts
type Point = { x: number; y: number };
type P = keyof Point; // 等价于 "x" | "y"
```

从"对象长什么样"到"对象有哪些键"，一个操作符完成。它对索引签名同样友好——有索引签名时返回的是**索引类型本身**：

```ts
type Arrayish = { [n: number]: unknown };
type A = keyof Arrayish; // number

type Mapish = { [k: string]: boolean };
type M = keyof Mapish;   // string | number
```

`M` 为什么是 `string | number` 而不只是 `string`？因为 **JS 对象的键总是被强制转成字符串**——`obj[0]` 与 `obj["0"]` 永远是同一个属性，所以数字键也算合法键名。这个细节在后面的映射类型篇会大放异彩：`keyof` + 映射类型正是 `Partial`、`Readonly` 等内置工具类型的实现原理。

## 二、typeof：把值的类型搬进类型世界

JS 早已有一个表达式上下文的 `typeof`（`console.log(typeof "Hello world")` 打印 "string"）。TS 在**类型上下文**新增了一个同名操作符——引用**变量或属性的类型**：

```ts
let s = "hello";
let n: typeof s; // 类型为 string
```

对基础类型这么写意义不大，但与其他类型操作符组合后威力显现。看预置类型 `ReturnType<T>`：它接收**函数类型**并产出返回类型：

```ts
type Predicate = (x: unknown) => boolean;
type K = ReturnType<Predicate>; // boolean
```

然后是最具教学价值的一幕——把 `ReturnType` 用在函数名上会得到一条很有启发性的报错：

```ts
function f() {
  return { x: 10, y: 3 };
}
type P = ReturnType<f>; // 错误 2749！
```

错误原因一针见血：**值和类型不是一回事**。`f` 是值（函数本身），`ReturnType` 要的是类型。要引用"值 `f` 所具有的类型"，就得请出 `typeof`：

```ts
type P = ReturnType<typeof f>; // { x: number; y: number }
```

这一组三连（`f` ✗ → `typeof f` ✓ → 得到返回类型）是 TS 最常用的惯用法之一——从现成的函数定义里提取返回类型，避免手写重复的对象类型。更重要的是，这个错误本身就是一次关于**两个世界边界**的提醒：函数声明 `function f()` 同时创造了一个值（可调用的 `f`）与一种类型意识，但类型位置里只能放类型，而 `typeof` 正是那张过桥通行证。理解这一点后，`Parameters<typeof f>`、`InstanceType<typeof Cls>` 这类写法也就顺理成章了——凡是要拿"某个具名变量/函数/类的类型"的场合，前缀 `typeof` 几乎是必然的。

## 三、值世界与类型世界的"双通道"

把两个操作符放到一起看，TS 的两个世界各有一条搬运通道：

| 操作符 | 上下文 | 作用 | 方向 |
|---|---|---|---|
| `keyof` | 类型上下文 | 对象类型 → 键名联合类型 | 类型 → 类型 |
| `typeof`（TS） | 类型上下文 | 值（变量/属性）→ 它的类型 | 值 → 类型 |
| `typeof`（JS） | 表达式上下文 | 值 → 运行时类型字符串 | 值 → 字符串 |

记忆技巧：`typeof` 是**值世界通往类型世界的桥**，`keyof` 是**类型世界内部的取键器**。两者还能串联——`keyof typeof obj` 先把对象值升格为类型、再抽出键联合，这是配置常量建模（主题色表、路由表、权限字典）的标准起手式。

为什么要如此强调"值"与"类型"的分野？因为 TS 里大量标识符在两个世界各有身份：`f` 既是可调用的函数值，又是某个类型世界缺席的东西；`Point` 既可作为类型使用，也可经 `typeof` 指向类构造器本身。混淆二者最常见的后果，就是写出 `ReturnType<f>` 这类"把值当类型传"的 2749 报错，或者反过来在类型位置调用值。把两个操作符当作两条单向通道记住，绝大多数类型体操的困惑都会少一半。

## 四、边界：typeof 的刻意限制

TS 故意限制 `typeof` 能作用的表达式——**只允许用于标识符（变量名）或它们的属性**。这避免了"你以为代码在执行，其实它只在类型层面被解析"的混淆陷阱：

```ts
declare const msgbox: (prompt: string) => boolean;
// 本想写 = ReturnType<typeof msgbox>
let shouldContinue: typeof msgbox("Are you sure you want to continue?");
// 错误 1005！typeof 后面不能跟调用
```

`typeof msgbox(...)` 试图"在类型位置调用函数"，TS 直接拒绝。想要函数返回类型，正确姿势永远是 `ReturnType<typeof msgbox>`。这条限制看似死板，实则守住了类型世界的纯粹性——类型查询不产生任何运行时副作用。

## 五、实操小组合拳

三个由本篇概念自然延伸出的常用写法：

```ts
// 1. 从配置对象提取键联合
const config = { host: "localhost", port: 8080 };
type ConfigKey = keyof typeof config; // "host" | "port"

// 2. 提取函数返回类型（配合索引访问拿到数组元素）
function loadUsers() { return [] as { id: number; name: string }[]; }
type User = ReturnType<typeof loadUsers>[number]; // { id: number; name: string }

// 3. 限定函数只能访问对象的合法键
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

第 3 例的 `K extends keyof T` 是本篇与后续两篇（索引访问 061、映射类型 062）的交汇点，也是类型安全属性访问器的通用写法——键名写错编译期就报错，而不是运行时拿到 `undefined`。

## 实践建议

1. **对象常量建模优先用 `keyof typeof`**：别手写一遍 `"host" | "port"`，改键时类型自动跟随，永不脱节。
2. **提取函数返回类型用 `ReturnType<typeof fn>`**：记住值是值、类型是类型，`ReturnType<fn>` 必然报错 2749。
3. **索引签名类型上取 keyof 记住会得到 `string | number`**：因为数字键在 JS 里会被强制转成字符串。
4. **`typeof` 只作用于标识符或属性**：别在类型位置写代码调用，那不属于类型查询的合法范围。
5. **属性访问器约束写成 `K extends keyof T`**：把"键必须存在"的保证从运行时挪到编译期。

## 小结

Type Manipulation 系列的两个入门操作符各司其职：`keyof` 在类型世界内取键，`typeof` 负责把值世界的东西搬进类型世界，二者的串联 `keyof typeof` 则让"配置常量的类型"无需重复声明。真正让它们威力倍增的是后面三篇——索引访问类型（取出值的类型）、条件类型（类型系统的 if）与映射类型（批量改造属性）。下一篇先拿下索引访问与条件类型。
