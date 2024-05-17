---
title: TypeScript 5.4
layout: docs
permalink: /docs/handbook/release-notes/typescript-5-4.html
oneline: TypeScript 5.4 Release Notes
---
## 在最后一次赋值后的闭包中保留缩窄

TypeScript 通常可以根据您可能执行的检查来推断变量的更具体类型。
这个过程称为缩窄。
```ts
function uppercaseStrings(x: string | number) {
    if (typeof x === "string") {
        // TypeScript knows 'x' is a 'string' here.
        return x.toUpperCase();
    }
}
```
一个常见的痛点是，这些缩小的类型并不总是在函数闭包中保留。
```ts
function getUrls(url: string | URL, names: string[]) {
    if (typeof url === "string") {
        url = new URL(url);
    }

    return names.map(name => {
        url.searchParams.set("name", name)
        //  ~~~~~~~~~~~~
        // error!
        // Property 'searchParams' does not exist on type 'string | URL'.

        return url.toString();
    });
}
```
在这里，TypeScript 认为假设 `url` 在我们的回调函数中*实际上*是一个 `URL` 对象是不“安全”的，因为它在其他地方被修改了；
然而，在这个例子中，那个箭头函数*总是在*对 `url` 的赋值之后创建的，并且它也是最后一次对 `url` 的赋值。

TypeScript 5.4 利用了这一点，使类型缩小变得更聪明。
当参数和 `let` 变量在非[提升](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)函数中使用时，类型检查器会寻找最后一个赋值点。
如果找到了一个，TypeScript 就可以从包含函数的外部安全地缩小范围。
这意味着上面的例子现在就可以正常工作。

请注意，如果变量在嵌套函数中的任何地方被赋值，缩小分析就不会启动。
这是因为没有确切的方法知道该函数是否会在后面被调用。
```ts
function printValueLater(value: string | undefined) {
    if (value === undefined) {
        value = "missing!";
    }

    setTimeout(() => {
        // Modifying 'value', even in a way that shouldn't affect
        // its type, will invalidate type refinements in closures.
        value = value;
    }, 500);

    setTimeout(() => {
        console.log(value.toUpperCase());
        //          ~~~~~
        // error! 'value' is possibly 'undefined'.
    }, 1000);
}
```
这应该会使许多典型的 JavaScript 代码更容易表达。
你可以在 [GitHub 上阅读更多关于这个变更的信息](https://github.com/microsoft/TypeScript/pull/56908)。

## `NoInfer` 实用类型

在调用泛型函数时，TypeScript 能够根据你传入的任何内容推断类型参数。
```ts
function doSomething<T>(arg: T) {
    // ...
}


// We can explicitly say that 'T' should be 'string'.
doSomething<string>("hello!");

// We can also just let the type of 'T' get inferred.
doSomething("hello!");
```
然而，一个挑战是并不总是清楚推断出的“最佳”类型是什么。
这可能导致 TypeScript 拒绝有效的调用，接受可疑的调用，或者在捕获到错误时报告更糟糕的错误信息。

例如，让我们想象一个 `createStreetLight` 函数，它接受一个颜色名称列表，以及一个可选的默认颜色。
```ts
function createStreetLight<C extends string>(colors: C[], defaultColor?: C) {
    // ...
}

createStreetLight(["red", "yellow", "green"], "red");
```
当我们传入一个不在原始`colors`数组中的`defaultColor`时会发生什么？
在这个函数中，`colors`应该是“真理之源”，描述了可以传递给`defaultColor`的内容。
```ts
// Oops! This undesirable, but is allowed!
createStreetLight(["red", "yellow", "green"], "blue");
```
在这次调用中，类型推断决定了`"blue"`与`"red"`、`"yellow"`或`"green"`一样，都是一个有效的类型。
因此，TypeScript并没有拒绝这个调用，而是将`C`的类型推断为`"red" | "yellow" | "green" | "blue"`。
你可能会说，推断就这样突然出现在我们面前！

目前人们处理这个问题的一种方法是添加一个额外的类型参数，该参数受到现有类型参数的约束。
```ts
function createStreetLight<C extends string, D extends C>(colors: C[], defaultColor?: D) {
}

createStreetLight(["red", "yellow", "green"], "blue");
//                                            ~~~~~~
// error!
// Argument of type '"blue"' is not assignable to parameter of type '"red" | "yellow" | "green" | undefined'.
```
这个方法是可行的，但有点尴尬，因为`D`可能不会在`createStreetLight`的签名中的其他地方使用。
虽然在这种情况下*还不错*，但在签名中只使用一次类型参数通常是一个代码异味。

这就是为什么TypeScript 5.4引入了新的`NoInfer<T>`实用类型。
将类型包裹在`NoInfer<...>`中，向TypeScript发出信号，不要深入挖掘并匹配内部类型以寻找类型推断的候选者。

使用`NoInfer`，我们可以将`createStreetLight`重写为类似以下内容：
```ts
function createStreetLight<C extends string>(colors: C[], defaultColor?: NoInfer<C>) {
    // ...
}

createStreetLight(["red", "yellow", "green"], "blue");
//                                            ~~~~~~
// error!
// Argument of type '"blue"' is not assignable to parameter of type '"red" | "yellow" | "green" | undefined'.
```
排除`defaultColor`类型被用于推断意味着`"blue"`永远不会成为推断候选，类型检查器可以拒绝它。

你可以在[实现拉取请求](https://github.com/microsoft/TypeScript/pull/56794)中看到具体的变化，以及由[Mateusz Burzyński](https://github.com/Andarist)提供的[初始实现](https://github.com/microsoft/TypeScript/pull/52968)！

## `Object.groupBy` 和 `Map.groupBy`

TypeScript 5.4 为 JavaScript 的新 `Object.groupBy` 和 `Map.groupBy` 静态方法添加了声明。

`Object.groupBy` 接受一个可迭代对象，以及一个决定每个元素应该放在哪个“组”的函数。
该函数需要为每个不同的组创建一个“键”，`Object.groupBy` 使用该键创建一个对象，其中每个键都映射到一个包含原始元素的数组。

所以以下的 JavaScript 代码：
```ts
const array = [0, 1, 2, 3, 4, 5];

const myObj = Object.groupBy(array, (num, index) => {
    return num % 2 === 0 ? "even": "odd";
});
```
基本上相当于写这个：
```ts
const myObj = {
    even: [0, 2, 4],
    odd: [1, 3, 5],
};
```
`Map.groupBy` 类似于 `Object.groupBy`，但它产生一个 `Map` 而不是普通对象。
如果你需要 `Map` 的保证，你正在处理期望 `Map` 的 API，或者你需要使用任何类型的键进行分组——不仅仅是可以在 JavaScript 中用作属性名的键，这可能更受欢迎。
```ts
const myObj = Map.groupBy(array, (num, index) => {
    return num % 2 === 0 ? "even" : "odd";
});
```
正如之前一样，你可以用等价的方式创建 `myObj`：
```ts
const myObj = new Map();

myObj.set("even", [0, 2, 4]);
myObj.set("odd", [1, 3, 5]);
```
请注意，在上面的`Object.groupBy`示例中，生成的对象使用了所有可选属性。
```ts
interface EvenOdds {
    even?: number[];
    odd?: number[];
}

const myObj: EvenOdds = Object.groupBy(...);

myObj.even;
//    ~~~~
// Error to access this under 'strictNullChecks'.
```
这是因为无法以一般方式保证*所有*键都是由`groupBy`生成的。

还要注意，这些方法只能通过将`target`配置为`esnext`或调整`lib`设置来访问。
我们预计它们最终将在稳定的`es2024`目标下可用。

我们要感谢[Kevin Gibbons](https://github.com/bakkot)为这些`groupBy`方法[添加声明](https://github.com/microsoft/TypeScript/pull/56805)。

## 支持`--moduleResolution bundler`和`--module preserve`中的`require()`调用

TypeScript有一个名为`bundler`的`moduleResolution`选项，旨在模拟现代打包器确定导入路径引用哪个文件的方式。
该选项的一个限制是它必须与`--module esnext`一起使用，这使得无法使用`import ... = require(...)`语法。
```ts
// previously errored
import myModule = require("module/path");
```
如果你打算只编写标准的 ECMAScript `import`，这看起来可能不是什么大问题，但在使用具有[条件导出](https://nodejs.org/api/packages.html#conditional-exports)的包时会有所不同。

在 TypeScript 5.4 中，当将 `module` 设置为名为 `preserve` 的新选项时，现在可以使用 `require()`。

在 `--module preserve` 和 `--moduleResolution bundler` 之间，这两个选项更准确地模拟了像 Bun 这样的打包器和运行时允许的内容，以及它们将如何执行模块查找。实际上，在使用 `--module preserve` 时，`bundler` 选项将隐式设置为 `--moduleResolution`（以及 `--esModuleInterop` 和 `--resolveJsonModule`）

```json5
{
    "compilerOptions": {
        "module": "preserve",
        // ^ 还隐含：
        // "moduleResolution": "bundler",
        // "esModuleInterop": true,
        // "resolveJsonModule": true,

        // ...
    }
}
```

在 `--module preserve` 下，ECMAScript `import` 将始终按原样发出，`import ... = require(...)` 将作为 `require()` 调用发出（尽管实际上您可能甚至不会使用 TypeScript 进行 emit，因为您可能会使用打包器来处理代码）。
无论包含文件的文件扩展名如何，这一点都成立。
所以这段代码的输出是：
```ts
import * as foo from "some-package/foo";
import bar = require("some-package/bar");
```
应该看起来像这样：
```ts
import * as foo from "some-package/foo";
var bar = require("some-package/bar");
```
这也就意味着，你选择的语法将指导[条件导出](https://nodejs.org/api/packages.html#conditional-exports)如何匹配。
所以在上面的例子中，如果`some-package`的`package.json`如下所示：

```json5
{
  "name": "some-package",
  "version": "0.0.1",
  "exports": {
    "./foo": {
        "import": "./esm/foo-from-import.mjs",
        "require": "./cjs/foo-from-require.cjs"
    },
    "./bar": {
        "import": "./esm/bar-from-import.mjs",
        "require": "./cjs/bar-from-require.cjs"
    }
  }
}
```

TypeScript将会解析这些路径到`[...]/some-package/esm/foo-from-import.mjs`和`[...]/some-package/cjs/bar-from-require.cjs`。

更多信息，你可以[在这里阅读这些新设置的详细介绍](https://github.com/microsoft/TypeScript/pull/56785)。

## 检查导入属性和断言

导入属性和断言现在会根据全局`ImportAttributes`类型进行检查。
这意味着运行时现在可以更准确地描述导入属性
```ts
// In some global file.
interface ImportAttributes {
    type: "json";
}

// In some other module
import * as ns from "foo" with { type: "not-json" };
//                                     ~~~~~~~~~~
// error!
//
// Type '{ type: "not-json"; }' is not assignable to type 'ImportAttributes'.
//  Types of property 'type' are incompatible.
//    Type '"not-json"' is not assignable to type '"json"'.
```
此变更得益于[Oleksandr Tarasiuk](https://github.com/a-tarasyuk)。

## 添加缺失参数的快速修复

TypeScript 现在有一个快速修复功能，用于向调用时参数过多的函数添加新参数。

![当 someFunction 调用 someHelperFunction 时，提供的参数比预期的多 2 个，此时会提供快速修复。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2024/01/add-missing-params-5-4-beta-before.png)

![应用快速修复后，缺失的参数已添加到 someHelperFunction。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2024/01/add-missing-params-5-4-beta-after.png)

当需要通过多个现有函数传递一个新参数时，这非常有用，这在现在可能会很麻烦。

此快速修复由[Oleksandr Tarasiuk](https://github.com/a-tarasyuk)提供。

## TypeScript 5.0 弃用的即将发生的变化

TypeScript 5.0 弃用了以下选项和行为：

 * `charset`
 * `target: ES3`
 * `importsNotUsedAsValues`
 * `noImplicitUseStrict`
 * `noStrictGenericChecks`
 * `keyofStringsOnly`
 * `suppressExcessPropertyErrors`
 * `suppressImplicitAnyIndexErrors`
 * `out`
 * `preserveValueImports`
 * 项目引用中的 `prepend`
 * 隐式特定于操作系统的 `newLine`

要继续使用它们，使用 TypeScript 5.0 及更高版本的开发者需要指定一个名为 `ignoreDeprecations` 的新选项，其值为 `"5.0"`。

然而，TypScript 5.4 将是这些选项继续正常工作的最后一个版本。
到了 TypeScript 5.5（可能在 2024 年 6 月），这些选项将成为硬错误，使用它们的代码将需要迁移。

要了解更多信息，您可以[在 GitHub 上阅读有关此计划的详细信息](https://github.com/microsoft/TypeScript/issues/51909)，其中包含如何最好地适应您的代码库的建议。

## 值得注意的行为变化

本节重点介绍了一系列值得注意的变化，作为任何升级的一部分，应该承认和理解这些变化。
有时它会强调弃用、删除和新限制。
它还可以包含在功能上改进的 bug 修复，但这也可能通过引入新错误影响现有构建。

### `lib.d.ts` 变化

为 DOM 生成的类型可能会影响您的代码库的类型检查。
有关更多信息，请[查看 TypeScript 5.4 的 DOM 更新](https://github.com/microsoft/TypeScript/pull/57027)。

### 更准确的条件类型约束

以下代码不再允许在函数 `foo` 中声明第二个变量。
```ts
type IsArray<T> = T extends any[] ? true : false;

function foo<U extends object>(x: IsArray<U>) {
    let first: true = x;    // Error
    let second: false = x;  // Error, but previously wasn't
}
```
以前，当 TypeScript 检查 `second` 的初始化表达式时，它需要确定 `IsArray<U>` 是否可以分配给单元类型 `false`。
虽然 `IsArray<U>` 没有明显的兼容方式，但 TypeScript 还会查看该类型的*约束*。
在条件类型 `T extends Foo ? TrueBranch : FalseBranch` 中，其中 `T` 是泛型，类型系统会查看 `T` 的约束，将其替换为 `T` 本身，并决定是选择真分支还是假分支。

但这种行为是不准确的，因为它过于急切。
即使 `T` 的约束不能分配给 `Foo`，这并不意味着它不会用可以分配给 `Foo` 的东西实例化。
因此，更正确的行为是在无法证明 `T` *从不* 或 *总是* 扩展 `Foo` 的情况下，为条件类型的约束生成一个联合类型。

TypeScript 5.4 采用了这种更准确的行为。
这在实践中意味着，您可能会发现某些条件类型实例不再与它们的支兼容。

[您可以在此处阅读有关具体更改的信息](https://github.com/microsoft/TypeScript/pull/56004)。

### 更积极地减少类型变量与原始类型之间的交集

TypeScript 现在根据类型变量的约束与这些原始类型的重叠程度，更积极地减少与类型变量和原始类型的交集。
```ts
declare function intersect<T, U>(x: T, y: U): T & U;

function foo<T extends "abc" | "def">(x: T, str: string, num: number) {

    // Was 'T & string', now is just 'T'
    let a = intersect(x, str);

    // Was 'T & number', now is just 'never'
    let b = intersect(x, num)

    // Was '(T & "abc") | (T & "def")', now is just 'T'
    let c = Math.random() < 0.5 ?
        intersect(x, "abc") :
        intersect(x, "def");
}
```
更多信息，请[查看此处的更改](https://github.com/microsoft/TypeScript/pull/56515)。

### 改进了针对具有插值的模板字符串的检查

TypeScript 现在更准确地检查字符串是否可以分配给模板字符串类型的占位符插槽。
```ts
function a<T extends {id: string}>() {
    let x: `-${keyof T & string}`;
    
    // Used to error, now doesn't.
    x = "-id";
}
```
这种行为更可取，但可能会导致在使用条件类型等构造时代码中出现中断，这些规则更改很容易观察到。

[查看此更改](https://github.com/microsoft/TypeScript/pull/56598)以了解更多详细信息。

### 类型导入仅与本地值冲突时的错误

以前，如果对`Something`的导入仅引用类型，TypeScript将在`isolatedModules`下允许以下代码。
```ts
import { Something } from "./some/path";

let Something = 123;
```
然而，对于单文件编译器来说，假设删除`import`是否“安全”是不安全的，即使代码在运行时肯定会失败。
在 TypeScript 5.4 中，这段代码将触发类似以下的错误：

```
导入 'Something' 与本地值冲突，因此在启用 'isolatedModules' 时必须使用仅类型导入声明。
```

修复方法应该是进行本地重命名，或者按照错误所述，在导入时添加 `type` 修饰符：
```ts
import type { Something } from "./some/path";

// or

import { type Something } from "./some/path";
```
[查看更多关于变更本身的信息](https://github.com/microsoft/TypeScript/pull/56354)。

### 新的枚举类型赋值限制

当两个枚举具有相同的声明名称和枚举成员名称时，它们之前总是被认为是兼容的；
然而，当值已知时，TypeScript 会默默地允许它们具有不同的值。

TypeScript 5.4 通过要求在值已知时它们必须完全相同来加强这一限制。
```ts
namespace First {
    export enum SomeEnum {
        A = 0,
        B = 1,
    }
}

namespace Second {
    export enum SomeEnum {
        A = 0,
        B = 2,
    }
}

function foo(x: First.SomeEnum, y: Second.SomeEnum) {
    // Both used to be compatible - no longer the case,
    // TypeScript errors with something like:
    //
    //  Each declaration of 'SomeEnum.B' differs in its value, where '1' was expected but '2' was given.
    x = y;
    y = x;
}
```
此外，当枚举成员之一没有静态已知的值时，还有新的限制。
在这些情况下，另一个枚举必须至少是隐式数值型的（例如，它没有静态解析的初始化表达式），或者是显式数值型的（意味着 TypeScript 可以将值解析为数值型）。
实际上，这意味着字符串枚举成员只与具有相同值的其他字符串枚举兼容。
```ts
namespace First {
    export declare enum SomeEnum {
        A,
        B,
    }
}

namespace Second {
    export declare enum SomeEnum {
        A,
        B = "some known string",
    }
}

function foo(x: First.SomeEnum, y: Second.SomeEnum) {
    // Both used to be compatible - no longer the case,
    // TypeScript errors with something like:
    //
    //  One value of 'SomeEnum.B' is the string '"some known string"', and the other is assumed to be an unknown numeric value.
    x = y;
    y = x;
}
```
更多信息，请[查看引入此更改的拉取请求](https://github.com/microsoft/TypeScript/pull/55924)。

### 枚举成员的命名限制

TypeScript 不再允许枚举成员使用名称 `Infinity`、`-Infinity` 或 `NaN`。
```ts
// Errors on all of these:
//
//  An enum member cannot have a numeric name.
enum E {
    Infinity = 0,
    "-Infinity" = 1,
    NaN = 2,
}
```
[查看更多细节请点这里](https://github.com/microsoft/TypeScript/pull/56161)。

### 更好的映射类型保留，通过带有 `any` Rest 元素的元组

之前，将带有 `any` 的映射类型应用到元组中会创建一个 `any` 元素类型。
这是不希望的，现在已经修复。
```ts
Promise.all(["", ...([] as any)])
    .then((result) => {
        const head = result[0];       // 5.3: any, 5.4: string
        const tail = result.slice(1); // 5.3 any, 5.4: any[]
    });
```
更多信息，请参见[修复](https://github.com/microsoft/TypeScript/pull/57031)以及[关于行为变更的后续讨论](https://github.com/microsoft/TypeScript/issues/57389)和[进一步的调整](https://github.com/microsoft/TypeScript/issues/57389)。

### 输出变更

虽然这些变更本身并不是破坏性的，但开发人员可能已经隐含地依赖于TypeScript的JavaScript或声明输出。
以下是一些值得注意的变更。

* [在类型参数名称被遮蔽时更频繁地保留类型参数名称](https://github.com/microsoft/TypeScript/pull/55820)
* [将异步函数的复杂参数列表移动到降级生成器体中](https://github.com/microsoft/TypeScript/pull/56296)
* [不要在函数声明中删除绑定别名](https://github.com/microsoft/TypeScript/pull/57020)
* [当ImportAttributes位于ImportTypeNode中时，应通过相同的输出阶段](https://github.com/microsoft/TypeScript/pull/56395)