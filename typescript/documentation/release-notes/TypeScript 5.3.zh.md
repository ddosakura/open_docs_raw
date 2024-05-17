---
title: TypeScript 5.3
layout: docs
permalink: /docs/handbook/release-notes/typescript-5-3.html
oneline: TypeScript 5.3 Release Notes
---
## 导入属性

TypeScript 5.3 支持最新的 [导入属性](https://github.com/tc39/proposal-import-attributes) 提案更新。

导入属性的一个用例是向运行时提供有关模块预期格式的信息。
```ts
// We only want this to be interpreted as JSON,
// not a runnable/malicious JavaScript file with a `.json` extension.
import obj from "./something.json" with { type: "json" };
```
这些属性的内容不会被 TypeScript 检查，因为它们是特定于宿主的，并且只是保留原样，以便浏览器和运行时可以处理它们（可能会出错）。
```ts
// TypeScript is fine with this.
// But your browser? Probably not.
import * as foo from "./foo.js" with { type: "fluffy bunny" };
```
动态 `import()` 调用也可以通过第二个参数使用导入属性。
```ts
const obj = await import("./something.json", {
    with: { type: "json" }
});
```
第二个参数的预期类型由一个名为`ImportCallOptions`的类型定义，默认情况下只期望一个名为`with`的属性。

请注意，导入属性是早期提案["导入断言"](https://devblogs.microsoft.com/typescript/announcing-typescript-4-5/#import-assertions)的演变，该提案已在TypeScript 4.5中实现。最明显的区别是使用`with`关键字而不是`assert`关键字。但不太明显的区别是，现在运行时可以自由使用属性来指导导入路径的解析和解释，而导入断言只能在加载模块后断言某些特征。

随着时间的推移，TypeScript将弃用旧的导入断言语法，以支持提议的导入属性语法。使用`assert`的现有代码应迁移到`with`关键字。需要导入属性的新代码应专门使用`with`。

我们要感谢[Oleksandr Tarasiuk](https://github.com/a-tarasyuk)实现了这个提案！我们还要表扬[Wenlu Wang](https://github.com/Kingwl)实现了[导入断言](https://github.com/microsoft/TypeScript/pull/40698)！

## 稳定支持`resolution-mode`在导入类型中

在TypeScript 4.7中，TypeScript在`///<reference types="..." />`中添加了对`resolution-mode`属性的支持，以控制指定符是通过`import`还是`require`语义进行解析。
```ts
/// <reference types="pkg" resolution-mode="require" />

// or

/// <reference types="pkg" resolution-mode="import" />
```
相应的字段也被添加到仅类型导入的导入断言中；
然而，它只在 TypeScript 的夜间版本中得到支持。
其理由是，从精神上讲，导入*断言*并不是为了指导模块解析。
因此，这个功能以仅在夜间模式下实验性地发布，以获得更多反馈。

但鉴于*[导入属性](#import-attributes)*可以指导解析，并且我们已经看到了合理的用例，TypeScript 5.3 现在支持 `import type` 的 `resolution-mode` 属性。
```ts
// Resolve `pkg` as if we were importing with a `require()`
import type { TypeFromRequire } from "pkg" with {
    "resolution-mode": "require"
};

// Resolve `pkg` as if we were importing with an `import`
import type { TypeFromImport } from "pkg" with {
    "resolution-mode": "import"
};

export interface MergedType extends TypeFromRequire, TypeFromImport {}
```
这些导入属性也可以用在 `import()` 类型上。
```ts
export type TypeFromRequire =
    import("pkg", { with: { "resolution-mode": "require" } }).TypeFromRequire;

export type TypeFromImport =
    import("pkg", { with: { "resolution-mode": "import" } }).TypeFromImport;

export interface MergedType extends TypeFromRequire, TypeFromImport {}
```
更多信息，请[查看此处的更改](https://github.com/microsoft/TypeScript/pull/55725)。

## 所有模块模式均支持 `resolution-mode`

之前，`resolution-mode` 只能在 `moduleResolution` 选项 `node16` 和 `nodenext` 下使用。
为了更容易地查找特定于类型的模块，`resolution-mode` 现在在所有其他 `moduleResolution` 选项（如 `bundler`、`node10`）下都能正常工作，并且在 `classic` 模式下不会报错。

更多信息，请[查看实现此更改的拉取请求](https://github.com/microsoft/TypeScript/pull/55725)。

## `switch (true)` 缩小范围

TypeScript 5.3 现在可以根据 `switch (true)` 中每个 `case` 子句的条件执行缩小范围操作。
```ts
function f(x: unknown) {
    switch (true) {
        case typeof x === "string":
            // 'x' is a 'string' here
            console.log(x.toUpperCase());
            // falls through...

        case Array.isArray(x):
            // 'x' is a 'string | any[]' here.
            console.log(x.length);
            // falls through...

        default:
          // 'x' is 'unknown' here.
          // ...
    }
}
```
[此功能](https://github.com/microsoft/TypeScript/pull/55991)是在[Mateusz Burzyński](https://github.com/Andarist)的[初步工作](https://github.com/microsoft/TypeScript/pull/53681)基础上发起的。我们要对这项贡献表示衷心的感谢！

## 在与布尔值的比较中进行缩小

有时，您可能会在条件中直接与`true`或`false`进行比较。通常，这些比较是不必要的，但您可能更喜欢这种风格，或者为了避免JavaScript真值周围的某些问题。无论如何，之前TypeScript在执行缩小时并不识别这种形式。

TypeScript 5.3现在跟上了，并在缩小变量时理解这些表达式。
```ts
interface A {
    a: string;
}

interface B {
    b: string;
}

type MyType = A | B;

function isA(x: MyType): x is A {
    return "a" in x;
}

function someFn(x: MyType) {
    if (isA(x) === true) {
        console.log(x.a); // works!
    }
}
```
我们要感谢[Mateusz Burzyński](https://github.com/Andarist)实现了这个功能的[pull request](https://github.com/microsoft/TypeScript/pull/53681)。

## 通过`Symbol.hasInstance`细化`instanceof`

JavaScript中有一个稍微深奥的特性，那就是可以覆盖`instanceof`操作符的行为。
为此，`instanceof`操作符右侧的值需要有一个特定的方法，该方法由`Symbol.hasInstance`命名。
```ts
class Weirdo {
    static [Symbol.hasInstance](testedValue) {
        // wait, what?
        return testedValue === undefined;
    }
}

// false
console.log(new Thing() instanceof Weirdo);

// true
console.log(undefined instanceof Weirdo);
```
为了更好地在 `instanceof` 中模拟这种行为，TypeScript 现在会检查是否存在这样的 `[Symbol.hasInstance]` 方法，并且是否声明为类型谓词函数。
如果存在，那么 `instanceof` 运算符左侧的测试值将被该类型谓词适当缩小范围。
```ts
interface PointLike {
    x: number;
    y: number;
}

class Point implements PointLike {
    x: number;
    y: number;

    constructor(x: number, y: number) {
        this.x = x;
        this.y = y;
    }

    distanceFromOrigin() {
        return Math.sqrt(this.x ** 2 + this.y ** 2);
    }

    static [Symbol.hasInstance](val: unknown): val is PointLike {
        return !!val && typeof val === "object" &&
            "x" in val && "y" in val &&
            typeof val.x === "number" &&
            typeof val.y === "number";
    }
}


function f(value: unknown) {
    if (value instanceof Point) {
        // Can access both of these - correct!
        value.x;
        value.y;

        // Can't access this - we have a 'PointLike',
        // but we don't *actually* have a 'Point'.
        value.distanceFromOrigin();
    }
}
```
正如你在这个例子中看到的，`Point` 定义了自己的 `[Symbol.hasInstance]` 方法。
它实际上作为一个自定义类型保护器，覆盖了一个名为 `PointLike` 的独立类型。
在函数 `f` 中，我们可以通过 `instanceof` 将 `value` 缩小为 `PointLike` 类型，但 *不是* `Point` 类型。
这意味着我们可以访问属性 `x` 和 `y`，但不能访问方法 `distanceFromOrigin`。

要了解更多信息，你可以[在这里阅读关于这个变更的信息](https://github.com/microsoft/TypeScript/pull/55052)。

## 检查实例字段上的 `super` 属性访问

在 JavaScript 中，可以通过 `super` 关键字访问基类中的声明。
```ts
class Base {
    someMethod() {
        console.log("Base method called!");
    }
}

class Derived extends Base {
    someMethod() {
        console.log("Derived method called!");
        super.someMethod();
    }
}

new Derived().someMethod();
// Prints:
//   Derived method called!
//   Base method called!
```
这与编写类似 `this.someMethod()` 的内容不同，因为那可能会调用一个被覆盖的方法。
这是一个微妙的区别，由于通常如果一个声明根本没有被覆盖，那么两者可以互换使用，这使得这种区别更加微妙。
```ts
class Base {
    someMethod() {
        console.log("someMethod called!");
    }
}

class Derived extends Base {
    someOtherMethod() {
        // These act identically.
        this.someMethod();
        super.someMethod();
    }
}

new Derived().someOtherMethod();
// Prints:
//   someMethod called!
//   someMethod called!
```
问题在于，`super` 只能用于在原型上声明的成员，而不能用于实例属性。这意味着，如果你写了 `super.someMethod()`，但 `someMethod` 被定义为一个字段，你将会在运行时遇到错误！
```ts
class Base {
    someMethod = () => {
        console.log("someMethod called!");
    }
}

class Derived extends Base {
    someOtherMethod() {
        super.someMethod();
    }
}

new Derived().someOtherMethod();
// 💥
// Doesn't work because 'super.someMethod' is 'undefined'.
```
TypeScript 5.3 现在更加严格地检查 `super` 属性访问/方法调用，以查看它们是否对应于类字段。
如果是这样，我们现在将收到一个类型检查错误。

[此检查](https://github.com/microsoft/TypeScript/pull/54056) 的贡献要归功于 [Jack Works](https://github.com/Jack-Works)！

## 类型的交互式内联提示

TypeScript 的内联提示现在支持跳转到类型的定义！
这使得随意浏览代码变得更加容易。

![Ctrl-clicking an inlay hint to jump to the definition of a parameter type.](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/10/clickable-inlay-hints-for-types-5-3-beta.gif)

更多内容请参见 [这里的实现](https://github.com/microsoft/TypeScript/pull/55141)。

## 首选 `type` 自动导入的设置

以前，当 TypeScript 为类型位置中的内容生成自动导入时，它会根据您的设置添加 `type` 修饰符。
例如，在以下代码中自动导入 `Person` 时：
```ts
export let p: Person
```
TypeScript 的编辑体验通常会为 `Person` 添加如下导入：
```ts
import { Person } from "./types";

export let p: Person
```
在某些设置下，例如 `verbatimModuleSyntax`，它会添加 `type` 修饰符：
```ts
import { type Person } from "./types";

export let p: Person
```
然而，也许您的代码库无法使用这些选项中的一些；或者您只是更喜欢在可能的情况下显式地使用`type`导入。

[最近的一个更改](https://github.com/microsoft/TypeScript/pull/56090)使得TypeScript现在可以将此设置为编辑器特定的选项。
在Visual Studio Code中，您可以在“TypeScript › Preferences: Prefer Type Only Auto Imports”下的UI中启用它，或者将其作为JSON配置选项`typescript.preferences.preferTypeOnlyAutoImports`。

## 通过跳过JSDoc解析进行优化

通过`tsc`运行TypeScript时，编译器现在将避免解析JSDoc。
这本身就降低了解析时间，而且还减少了存储注释的内存使用以及垃圾回收所花费的时间。
总的来说，您应该会看到编译速度略有提升，以及在`--watch`模式下更快的反馈。

[具体更改可以在这里查看](https://github.com/microsoft/TypeScript/pull/52921)。

因为并非所有使用TypeScript的工具都需要存储JSDoc（例如typescript-eslint和Prettier），这种解析策略已经作为API本身的一部分呈现出来。
这使得这些工具能够获得我们为TypeScript编译器带来的相同的内存和速度改进。
新的注释解析策略选项在`JSDocParsingMode`中描述。
更多信息可在[此拉取请求](https://github.com/microsoft/TypeScript/pull/55739)上查看。

## 通过比较非规范化交叉类型进行优化

在TypeScript中，联合类型和交叉类型总是遵循特定的形式，其中交叉类型不能包含联合类型。这意味着当我们创建一个联合类型上的交叉类型，如`A & (B | C)`时，该交叉类型将被规范化为`(A & B) | (A & C)`。然而，在某些情况下，类型系统为了显示目的会保持原始形式。

事实证明，原始形式可以用于一些巧妙的快速路径类型比较。

例如，假设我们有`SomeType & (Type1 | Type2 | ... | Type99999NINE)`，我们想看看这是否可以赋值给`SomeType`。回想一下，我们的源类型实际上并不是一个交叉类型——我们有一个看起来像`(SomeType & Type1) | (SomeType & Type2) | ... |(SomeType & Type99999NINE)`的联合类型。在检查一个联合类型是否可以赋值给某个目标类型时，我们必须检查联合类型的*每个*成员是否可以赋值给目标类型，这可能会非常慢。

在TypeScript 5.3中，我们窥探了我们能够藏起来的原始交叉形式。在比较类型时，我们快速检查目标是否存在于源交叉的任何组成部分中。

更多信息，请[参见此拉取请求](https://github.com/microsoft/TypeScript/pull/55851)。

## `tsserverlibrary.js` 和 `typescript.js` 之间的整合

TypeScript 本身提供了两个库文件：`tsserverlibrary.js` 和 `typescript.js`。`tsserverlibrary.js` 中提供了一些特定的 API（例如 `ProjectService` API），这对某些导入者可能很有用。尽管如此，这两个文件是不同的包，它们之间有很多重叠，导致在包中重复代码。更重要的是，由于自动导入或肌肉记忆，始终如一地使用其中一个而不是另一个可能具有挑战性。意外地加载两个模块太容易了，代码可能无法在 API 的不同实例上正常工作。即使它确实有效，加载第二个包也会增加资源使用。

鉴于此，我们决定将两者整合。`typescript.js` 现在包含了 `tsserverlibrary.js` 过去包含的内容，而 `tsserverlibrary.js` 现在只是重新导出 `typescript.js`。在整合前后进行比较，我们看到包大小的以下减少：

|  | 之前 | 之后 | 差异 | 差异（百分比） |
| - | - | - | - | - |
| 打包 | 6.90 MiB | 5.48 MiB | -1.42 MiB | -20.61% |
| 解包 | 38.74 MiB | 30.41 MiB | -8.33 MiB | -21.50% |

|  | 之前 | 之后 | 差异 | 差异（百分比） |
| - | - | - | - | - |
| `lib/tsserverlibrary.d.ts` | 570.95 KiB | 865.00 B | -570.10 KiB | -99.85% |
| `lib/tsserverlibrary.js` | 8.57 MiB | 1012.00 B | -8.57 MiB | -99.99% |
| `lib/typescript.d.ts` | 396.27 KiB | 570.95 KiB | +174.68 KiB | +44.08% |
| `lib/typescript.js` | 7.95 MiB | 8.57 MiB | +637.53 KiB | +7.84% |

换句话说，这是超过 20.5% 的包大小减少。

更多信息，您可以 [查看此处涉及的工作](https://github.com/microsoft/TypeScript/pull/55273)。

## 重大更改和正确性改进

### `lib.d.ts` 更改

为 DOM 生成的类型可能会影响您的代码库。
更多信息，请 [查看 TypeScript 5.3 的 DOM 更新](https://github.com/microsoft/TypeScript/pull/55798)。

### 对实例属性上的 `super` 访问的检查

TypeScript 5.3 现在检测由 `super.` 属性访问引用的声明是否为类字段，并在此情况下发出错误。这可以防止在运行时可能发生的错误。

[在此处了解更多关于此更改的信息](https://github.com/microsoft/TypeScript/pull/54056)。