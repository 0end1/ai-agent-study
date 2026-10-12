# TypeScript 学习笔记（8/17）：对象类型——修饰符、索引签名与扩展的分家

> 对应书源：TypeScript Handbook「Object Types」前半（Property Modifiers / Index Signatures / Excess Property Checks / Extending Types / Intersections）｜系列第 8 篇

## 开篇：JS 的一切数据，都装在"袋子"里

JavaScript 组织数据的基本方式就是对象——从函数参数的"选项袋"（option bag）到 API 响应，无不是把一堆键值打包传递。TypeScript 用**对象类型**给这些袋子画"形状图"。画法有三种，效果等价：

```ts
function greet(person: { name: string; age: number }) { /* ... */ } // 匿名

interface Person { name: string; age: number }  // 接口

type Person = { name: string; age: number };    // 类型别名
```

形状定了，接下来是细节功夫：哪些属性可有可无？哪些只读？属性名事先不知道怎么办？传参时多塞了属性算不算错？本篇逐一拆解。

## 一、可选属性：`?` 只承诺"如果有"

给属性名加 `?` 标记可选：

```ts
interface PaintOptions {
  shape: Shape;
  xPos?: number;
  yPos?: number;
}
paintShape({ shape });                          // ✅ 四种传法都合法
paintShape({ shape, xPos: 100, yPos: 100 });
```

可选的含义很克制：**"如果设置了，类型必须是 number"**——它不承诺属性存在。所以 `strictNullChecks` 下读取得到 `number | undefined`，必须处理。最常见的处理是"缺省补默认值"，而 JS 的**解构默认值**语法为此而生：

```ts
function paintShape({ shape, xPos = 0, yPos = 0 }: PaintOptions) {
  // 调用方看：xPos/yPos 可选；函数体看：xPos/yPos 一定是 number
}
```

一个语法注意点：**解构模式里没法写类型注解**。`{ shape: Shape }` 在 JS 里意为"取 shape 属性并重命名为局部变量 Shape"，`{ xPos: number = 100 }` 会创建一个叫 `number` 的变量——类型注解只能写在解构模式整体之后。

## 二、readonly：防的是"重写"，不是"改内容"

`readonly` 让属性在**类型检查期**不可重新赋值，运行时行为完全不变：

```ts
interface Home {
  readonly resident: { name: string; age: number };
}
home.resident.age++;    // ✅ 修改内部内容没问题
home.resident = {};     // ❌ 不能替换整个属性
```

两个管理预期的重要提醒：

1. **readonly 不是深度不可变**：它只锁"属性本身的重写"，属性内部内容（对象字段、数组元素）照改不误。
2. **别名可以绕过它**：类型兼容性检查完全不考虑 readonly 修饰符。`WritablePerson` 赋给 `ReadonlyPerson` 后两者指向同一对象——通过可写引用 `age++`，"只读"引用读到的值也变了。

所以 readonly 的定位是**开发期的意图信号**，不是运行时安全护栏。这和很多语言里"常量即深冻结"的直觉不同——TypeScript 选择只承诺编译期能验证的部分，其余交给程序员自律。后续篇章的映射修饰符（`Readonly<T>` / `-readonly`）可以批量增删这个标记。

## 三、索引签名：属性名事先未知时

描述"字典/数组"这类结构，用**索引签名**声明"按某种键索引，返回什么类型"：

```ts
interface StringArray {
  [index: number]: string;
}
const secondItem = myArray[1]; // string
```

索引键只允许 `string`、`number`、`symbol`、模板字符串模式及其联合。两个细节值得记：

- **同时声明 number 和 string 索引器时，数字索引的返回类型必须是字符串索引返回类型的子类型**。因为 JS 里 `obj[100]` 实际等价于 `obj["100"]`——数字先转字符串再索引，两者必须自洽，否则 `NotOkay` 之类的接口会报错。
- **字符串索引签名会"管束"所有具名属性**：`obj.name` 就是 `obj["name"]`，所以 `NumberDictionary` 里写 `[index: string]: number` 再加 `name: string` 会报错；把索引类型放宽为 `number | string` 联合即可放行。索引签名本身也能加 `readonly`，禁止按下标赋值。

## 四、多余属性检查：字面量的"特殊安检"

对象类型检查默认是**结构化**的——多出来的属性本该无关紧要。但 TS 对**对象字面量**有一道特殊安检（excess property checking）：

```ts
interface SquareConfig {
  color?: string;
  width?: number;
}
createSquare({ colour: "red", width: 100 }); // ❌ colour 不存在于目标类型
```

在纯 JS 里这是**静默失败**的典型 bug（拼错属性名，逻辑悄悄走默认值）；TS 的立场是"这里大概率有 bug"。注意安检只针对**新建字面量直接赋值或传参**的场景，已存在的变量不走这道安检。

绕过检查有三种办法，但官方态度是"多数多余属性错误真的是 bug，别绕"：

| 绕过方式 | 写法要点 | 适用性 |
|----------|----------|--------|
| 类型断言 | `{ width: 100, opacity: 0.5 } as SquareConfig` | 快但凿穿保护 |
| 索引签名 | 接口加 `[propName: string]: unknown` | 适合确实要收任意附加属性的选项袋 |
| 中间变量 | 先赋给变量再传参 | 需与目标类型有共同属性，否则仍报错 |

反例提醒：`let squareOptions = { colour: "red" }`（与 SquareConfig 毫无共同属性）再传参依旧报错——绕过手段不是免死金牌。

## 五、Extending Types：extends 是"复制成员 + 声明亲缘"

```ts
interface BasicAddress {
  name?: string;
  street: string;
  city: string;
  country: string;
  postalCode: string;
}

interface AddressWithUnit extends BasicAddress {
  unit: string;
}
```

`extends` 把父类型的成员整体复制过来，只写新增字段——既省样板代码，又向读者声明"这两个类型的 street 是同一个东西"。接口还支持**多继承**：

```ts
interface ColorfulCircle extends Colorful, Circle {}
```

## 六、交叉类型：`&` 是另一条合并之路

```ts
type ColorfulCircle = Colorful & Circle;
// draw(circle: Colorful & Circle) 同时访问 color 和 radius
```

它和 `extends` 效果相似，但**冲突处理方式完全不同**——这往往就是二选一的决策依据：

| | interface extends / 重名合并 | 交叉类型 `&` |
|---|---|---|
| 同名属性类型冲突 | **直接报错**（如两个 Person 接口同名 name 类型不同） | 不报错，但属性变成 **never**（要求同时是 string 和 number，不可能成立） |
| 冲突发现时机 | 定义处立即发现 | 潜伏到使用处才现形 |

`Person1 & Person2` 里 `staffer.name` 的类型是 `never`——编译通过但成员废掉。**尽早报错的 interface extends 通常更安全**，这也是官方"先用 interface"建议的另一条注脚。

## 实践建议

1. **可选属性读取前先处理 undefined**，解构默认值是最优雅的收窄方式；别在解构模式里写类型注解。
2. **把 readonly 当意图标注而非安全机制**：它防重写不防改内容，别名一绕就失效；需要真正不可变时用映射修饰符体系（后续篇章）。
3. **索引签名是逃生舱不是首选**：它会强制所有具名属性服从其类型；能用具体属性就别全字典化。
4. **多余属性检查报错时先怀疑自己拼错了**，而不是急着 `as` 绕过；确实要收任意附加属性再加索引签名。
5. **类型合并优先用 interface extends**：冲突即时报错好过交叉类型里的 `never` 陷阱。

## 小结

对象类型的三种写法只是入口，真正的功力在修饰符与组合规则：`?` 与 readonly 管理属性的"存在性"和"可写性"，索引签名兜住未知属性名，多余属性检查替你拦截拼写事故，而 extends 与 `&` 给出了两种合并哲学——一个早失败，一个晚失败。下一篇继续对象类型的后半场：泛型对象（Box 体系）、元组类型与只读数组。
