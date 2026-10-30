# TypeScript 学习笔记（16/17）：映射类型与模板字面量类型——批量改造与字符串魔术

> 对应书源：TypeScript Handbook「Mapped Types」+「Template Literal Types」全篇｜系列第 16 篇

## 开篇：从"一个个改"到"整批改造"

前两篇学会了取键（keyof）、取值（索引访问）与做选择（条件类型）。现在的问题是：如果我想把某个类型**所有属性**都加 `readonly`、都变可选、都套一层 Promise，难道要手写一遍？映射类型（Mapped Types）就是答案——它建立在索引签名语法之上，用 `keyof` 生成的键联合**遍历键**，批量产出新类型。而模板字面量类型则让这个遍历过程顺带能**改写字符串键名**，组合起来便是 TS 类型系统最花哨也最实用的一幕。

## 一、映射类型：`[P in keyof T]`

映射类型是一种泛型类型，用 `PropertyKey` 联合（通常由 `keyof` 生成）遍历键来创建类型：

```ts
type OptionsFlags<Type> = {
  [Property in keyof Type]: boolean;
};

type Features = { darkMode: () => void; newUserProfile: () => void };
type FeatureOptions = OptionsFlags<Features>;
// { darkMode: boolean; newUserProfile: boolean }
```

它把 `Features` 的全部属性**值类型统一改写成 boolean**，键名原样保留。这就是 Partial、Readonly、Record 等内置工具类型的骨架——官方文档里 Partial 的实现即为 `[P in keyof T]?: T[P]`，Readonly 为 `readonly [P in keyof T]: T[P]`。

映射类型与索引签名的区别在于：索引签名（`[key: string]: boolean`）声明的是"未知键的通配规则"，而映射类型遍历的是**已知且穷尽的键联合**。前者用于字典式结构，后者用于改造既有类型——这个分野决定了各自的使用场景。也正因为遍历的是穷尽联合，映射类型产出的类型是"闭合"的：源类型加一个新属性，产物自动跟随，无需任何手工维护，这正是 DRY 原则在类型层面的体现。

## 二、Mapping Modifiers：`+` / `-` 增删修饰符

映射时可施加两个修饰符：`readonly`（可变性）与 `?`（可选性），并可加前缀 `-` 或 `+` 表示删除或添加——**不写前缀则默认 `+`**：

```ts
// 去掉所有 readonly
type CreateMutable<Type> = {
  -readonly [Property in keyof Type]: Type[Property];
};

// 去掉所有可选
type Concrete<Type> = {
  [Property in keyof Type]-?: Type[Property];
};

type MaybeUser = { id: string; name?: string; age?: number };
type User = Concrete<MaybeUser>; // { id: string; name: string; age: number }
```

| 写法 | 效果 |
|---|---|
| `readonly [P in keyof T]` | 全加只读（即 Readonly） |
| `-readonly [P in keyof T]` | 全去只读（即 Mutable） |
| `[P in keyof T]?` | 全加可选（即 Partial） |
| `[P in keyof T]-?` | 全去可选（即 Required） |

这也解释了 054 篇"readonly 被兼容性检查忽略"的实用解法：真要强制可变，用 `-readonly` 造一个新类型，而不是指望类型系统替你拦住。

## 三、Key Remapping：用 `as` 改写键名

TS 4.1 起，映射类型支持 `as` 子句**重映射键名**：

```ts
type MappedTypeWithNewProperties<Type> = {
  [Properties in keyof Type as NewKeyType]: Type[Properties];
};
```

配合模板字面量类型，可以基于旧键名造新键名——自动生成 getter 家族：

```ts
type Getters<Type> = {
  [Property in keyof Type as `get${Capitalize<string & Property>}`]: () => Type[Property];
};
// LazyPerson: { getName: () => string; getAge: () => number; getLocation: () => string }
```

注意 `string & Property` 这一步：`keyof` 产出的可能是 `string | number | symbol`，而模板字面量只接受字符串类型，交叉一下即可收窄。

`as` 的另一个杀手级用法是**过滤键**——让不想要的键产出 `never`（属性即被剔除）：

```ts
type RemoveKindField<Type> = {
  [Property in keyof Type as Exclude<Property, "kind">]: Type[Property];
};
type KindlessCircle = RemoveKindField<Circle>; // 剔除 kind 属性
```

而且映射的遍历对象不限于 `keyof` 产生的键联合——**任意联合类型都可以**：

```ts
type EventConfig<Events extends { kind: string }> = {
  [E in Events as E["kind"]]: (event: E) => void;
};
// Config: { square: (e: SquareEvent) => void; circle: (e: CircleEvent) => void }
```

这里用成员的 `kind` 字段值当键，是"判别联合 → 事件处理表"的经典建模手法。

## 四、与条件类型合体：ExtractPII

映射类型常与其他类型操作组合。例如给每个字段判断"是否含 PII"，产出一张布尔图：

```ts
type ExtractPII<Type> = {
  [Property in keyof Type]: Type[Property] extends { pii: true } ? true : false;
};
// DBFields → { id: false; name: true }
```

遍历（映射）+ 分支（条件）+ 取值（索引访问）三种能力在此彻底融合。

## 五、模板字面量类型：类型系统的字符串拼接

模板字面量类型建立在字符串字面量类型之上，语法与 JS 模板字符串一致，但用在**类型位置**：

```ts
type World = "world";
type Greeting = `hello ${World}`; // "hello world"
```

真正的威力在于：插值位置放联合时，会展开成**所有可能字符串的集合**，且多个插值位置之间做**交叉相乘**：

```ts
type EmailLocaleIDs = "welcome_email" | "email_heading";
type FooterLocaleIDs = "footer_title" | "footer_sendoff";
type AllLocaleIDs = `${EmailLocaleIDs | FooterLocaleIDs}_id`; // 4 个
type Lang = "en" | "ja" | "pt";
type LocaleMessageIDs = `${Lang}_${AllLocaleIDs}`;            // 3 × 4 = 12 个
```

官方提醒：这种组合会指数膨胀，大型字符串联合建议**提前生成**（ahead-of-time generation），小场景才用类型计算。

### 实战：约束事件名

`makeWatchedObject` 给对象加一个 `on(eventName, callback)` 方法，事件名形如 `属性名 + "Changed"`。朴素的 `on(eventName: string, ...)` 毫无保护，模板字面量类型把约束搬进类型系统：

```ts
type PropEventSource<Type> = {
  on(eventName: `${string & keyof Type}Changed`, callback: (newValue: any) => void): void;
};
declare function makeWatchedObject<Type>(obj: Type): Type & PropEventSource<Type>;

person.on("firstNameChanged", () => {});  // ✅
person.on("firstName", () => {});         // ❌ 少了 Changed
person.on("frstNameChanged", () => {});   // ❌ 拼错也拦得住
```

### 再进一步：让回调参数也类型安全

上面回调参数是 `any`——没用上"firstName 是 string、age 是 number"的信息。把 `on` 改成泛型方法即可让 TS 反推属性名：

```ts
type PropEventSource<Type> = {
  on<Key extends string & keyof Type>(
    eventName: `${Key}Changed`,
    callback: (newValue: Type[Key]) => void
  ): void;
};

person.on("firstNameChanged", (newName) => {
  console.log(newName.toUpperCase()); // newName: string
});
person.on("ageChanged", (newAge) => {
  if (newAge < 0) console.warn("warning!"); // newAge: number
});
```

原理四步：字面量被捕获为字面量类型 → 校验它落在合法属性联合内 → 用索引访问 `Type[Key]` 取回属性类型 → 应用到回调参数。这是模板字面量 + 索引访问 + 泛型推断的三重奏。

把这三个例子串起来看，模板字面量在类型系统里其实提供了一对互逆能力：**构造**（从属性名联合拼出事件名联合 `firstNameChanged | ageChanged | ...`）与**解构**（从 `"firstNameChanged"` 反推出 `Key = "firstName"`）。前者用于约束入参，后者用于推导上下文，两者共用同一套 `${}` 语法，只是泛型推断的方向不同——理解这一点，就能读懂绝大多数以模板字面量为核心的类型工具。

## 六、内置字符串操作类型

TS 内置四个字符串操作类型，由编译器直接实现（不在 `.d.ts` 里，为性能考虑）：

| 类型 | 作用 | 例子 |
|---|---|---|
| `Uppercase<S>` | 全大写 | `Uppercase<"my_app">` → `MY_APP` |
| `Lowercase<S>` | 全小写 | `Lowercase<"Hello, world">` → `hello, world` |
| `Capitalize<S>` | 首字母大写 | `Capitalize<"hello">` → `Hello` |
| `Uncapitalize<S>` | 首字母小写 | `Uncapitalize<"HELLO">` → `hELLO` |

它们与模板字面量组合常用于缓存键、事件名、CSS 变量名的规范化：`` type ASCIICacheKey<Str extends string> = `ID-${Uppercase<Str>}` ``。技术细节上，这些实现直接调用 JS 运行时字符串方法，**不考虑语言区域（not locale aware）**。

## 实践建议

1. **批量改造属性一律用映射类型**，别手写第二份类型；需要反向操作就加 `-readonly` / `-?`。
2. **`as` 重映射是改名与过滤的双刃剑**：造 getter/setter 家族、剔除字段都靠它。
3. **模板字面量的联合是交叉相乘的**：注意规模爆炸，大联合改用代码生成。
4. **`string & Property` 收窄**是模板拼接前的必备动作，别忘。
5. **事件名/回调联动用泛型方法 + `${Key}Changed`**：一处定义，键名与参数类型同时受保护。

## 小结

映射类型让"对类型做批量手术"成为一行表达式，模板字面量类型则补上了字符串键名的生成与推断能力，二者通过 `as` 无缝衔接 ——TS 里最华丽的类型体操，落点却非常实用：类型安全的事件系统、自动生成的 getter、可推导的配置表。系列只剩最后一篇：类型声明文件（.d.ts）与读懂报错。
