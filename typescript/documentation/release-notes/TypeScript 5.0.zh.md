---
title: TypeScript 5.0
layout: docs
permalink: /docs/handbook/release-notes/typescript-5-0.html
oneline: TypeScript 5.0 Release Notes
---
## 装饰器

装饰器是即将推出的 ECMAScript 特性，它允许我们以可重用的方式自定义类及其成员。

让我们考虑以下代码：
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
p.greet();
```
`greet` 在这里相当简单，但让我们想象它是一些更复杂的东西 - 也许它执行一些异步逻辑，它是递归的，它有副作用等。
不管你在想象什么样的泥球，我们假设你加入一些 `console.log` 调用来帮助调试 `greet`。
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet() {
        console.log("LOG: Entering method.");

        console.log(`Hello, my name is ${this.name}.`);

        console.log("LOG: Exiting method.")
    }
}
```
这种模式相当常见。
如果我们能够为每种方法都做到这一点，那当然是太好了！

这就是装饰器发挥作用的地方。
我们可以编写一个名为 `loggedMethod` 的函数，如下所示：
```ts
function loggedMethod(originalMethod: any, _context: any) {

    function replacementMethod(this: any, ...args: any[]) {
        console.log("LOG: Entering method.")
        const result = originalMethod.call(this, ...args);
        console.log("LOG: Exiting method.")
        return result;
    }

    return replacementMethod;
}
```
“这些`any`是怎么回事？
这是`any`Script吗！？”

请耐心等待 - 我们现在保持简单，以便我们可以专注于这个函数正在做什么。
注意到`loggedMethod`接受原始方法（`originalMethod`）并返回一个函数，该函数

1. 记录一个“Entering...”消息
2. 将`this`及其所有参数传递给原始方法
3. 记录一个“Exiting...”消息，并且
4. 返回原始方法返回的任何内容。

现在我们可以使用`loggedMethod`来*装饰*方法`greet`：
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @loggedMethod
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
p.greet();

// Output:
//
//   LOG: Entering method.
//   Hello, my name is Ray.
//   LOG: Exiting method.
```
我们刚刚在`greet`方法上方使用了`loggedMethod`作为装饰器，并注意到我们将其写为`@loggedMethod`。
当我们这样做时，它会使用该方法的*目标*和一个*上下文对象*进行调用。
因为`loggedMethod`返回了一个新的函数，所以该函数替换了`greet`的原始定义。

我们还没有提到，但是`loggedMethod`是用第二个参数定义的。
它被称为“上下文对象”，它具有一些有关被装饰方法声明的有用信息，例如它是否为`#private`成员，或者是`static`，或者方法的名称是什么。
让我们重写`loggedMethod`以利用这一点，并打印出被装饰方法的名称。
```ts
function loggedMethod(originalMethod: any, context: ClassMethodDecoratorContext) {
    const methodName = String(context.name);

    function replacementMethod(this: any, ...args: any[]) {
        console.log(`LOG: Entering method '${methodName}'.`)
        const result = originalMethod.call(this, ...args);
        console.log(`LOG: Exiting method '${methodName}'.`)
        return result;
    }

    return replacementMethod;
}
```
我们现在正在使用上下文参数 - 它是`loggedMethod`中第一个类型比`any`和`any[]`更严格的东西。
TypeScript提供了一个名为`ClassMethodDecoratorContext`的类型，用于模拟方法装饰器所接受上下文对象。

除了元数据之外，方法上下文对象还有一个名为`addInitializer`的有用函数。
这是一种在构造函数开始时（或者如果我们正在处理`static`，则是类本身的初始化）挂钩的方法。

例如 - 在JavaScript中，我们经常编写类似以下模式的内容：
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;

        this.greet = this.greet.bind(this);
    }

    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}
```
或者，`greet` 可以声明为一个初始化为箭头函数的属性。
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    greet = () => {
        console.log(`Hello, my name is ${this.name}.`);
    };
}
```
这段代码是为了确保如果`greet`作为独立函数调用或作为回调函数传递时，`this`不会被重新绑定。
```ts
const greet = new Person("Ray").greet;

// We don't want this to fail!
greet();
```
我们可以编写一个装饰器，它使用 `addInitializer` 在构造函数中调用 `bind`。
```ts
function bound(originalMethod: any, context: ClassMethodDecoratorContext) {
    const methodName = context.name;
    if (context.private) {
        throw new Error(`'bound' cannot decorate private properties like ${methodName as string}.`);
    }
    context.addInitializer(function () {
        this[methodName] = this[methodName].bind(this);
    });
}
```
`bound` 函数并没有返回任何东西，所以当它装饰一个方法时，它不会修改原始的函数。
相反，它会在任何其他字段初始化之前添加逻辑。
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @bound
    @loggedMethod
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
const greet = p.greet;

// Works!
greet();
```
请注意，我们堆叠了两个装饰器 - `@bound` 和 `@loggedMethod`。
这些装饰器以“相反的顺序”运行。
也就是说，`@loggedMethod` 装饰原始方法 `greet`，而 `@bound` 装饰 `@loggedMethod` 的结果。
在这个例子中，这并不重要 - 但如果你的装饰器有副作用或期望按某种顺序执行，那就可能有问题。

另外值得注意的是 - 如果你从风格上更喜欢，你可以将这些装饰器放在同一行。
```ts
    @bound @loggedMethod greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
```
可能不太明显的是，我们甚至可以创建*返回*装饰器函数的函数。这使得我们可以稍微定制最终的装饰器。如果我们愿意，我们可以让`loggedMethod`返回一个装饰器，并自定义它记录消息的方式。
```ts
function loggedMethod(headMessage = "LOG:") {
    return function actualDecorator(originalMethod: any, context: ClassMethodDecoratorContext) {
        const methodName = String(context.name);

        function replacementMethod(this: any, ...args: any[]) {
            console.log(`${headMessage} Entering method '${methodName}'.`)
            const result = originalMethod.call(this, ...args);
            console.log(`${headMessage} Exiting method '${methodName}'.`)
            return result;
        }

        return replacementMethod;
    }
}
```
如果我们那样做了，我们必须在使用它作为装饰器之前调用 `loggedMethod`。
然后我们可以传递任何字符串作为消息的前缀，这些消息将被记录到控制台。
```ts
class Person {
    name: string;
    constructor(name: string) {
        this.name = name;
    }

    @loggedMethod("⚠️")
    greet() {
        console.log(`Hello, my name is ${this.name}.`);
    }
}

const p = new Person("Ray");
p.greet();

// Output:
//
//   ⚠️ Entering method 'greet'.
//   Hello, my name is Ray.
//   ⚠️ Exiting method 'greet'.
```
装饰器不仅可以用于方法！
它们还可以用于属性/字段、getter、setter和自动访问器。
甚至类本身也可以用于诸如子类和注册之类的事情进行装饰。

要深入了解装饰器，您可以阅读[Axel Rauschmayer的广泛总结](https://2ality.com/2022/10/javascript-decorators.html)。

有关所涉及更改的更多信息，您可以[查看原始拉取请求](https://github.com/microsoft/TypeScript/pull/50820)。

### 与实验性旧版装饰器的区别

如果您已经使用TypeScript一段时间了，您可能会知道它多年来一直支持“实验性”装饰器。
虽然这些实验性装饰器非常有用，但它们模拟了装饰器提案的一个更旧版本，并且始终需要一个名为`--experimentalDecorators`的可选编译器标志。
在没有此标志的情况下尝试在TypeScript中使用装饰器曾经会提示错误消息。

`--experimentalDecorators`将在可预见的未来继续存在；
然而，在没有该标志的情况下，装饰器现在将成为所有新代码的有效语法。
在`--experimentalDecorators`之外，它们将进行类型检查和发出不同。
类型检查规则和发出足够不同，以至于虽然装饰器*可以*编写以支持旧的和新的装饰器行为，但任何现有的装饰器函数都不太可能这样做。

这个新的装饰器提案与`--emitDecoratorMetadata`不兼容，并且不允许装饰参数。
未来的ECMAScript提案可能有助于弥合这一差距。

最后一点说明：除了允许在`export`关键字之前放置装饰器外，装饰器提案现在还提供了在`export`或`export default`之后放置装饰器的选项。
唯一的例外是不允许混合使用这两种风格。
```ts
// ✅ allowed
@register export default class Foo {
    // ...
}

// ✅ also allowed
export default @register class Bar {
    // ...
}

// ❌ error - before *and* after is not allowed
@before export @after class Bar {
    // ...
}
```
### 编写类型良好的装饰器

上面的 `loggedMethod` 和 `bound` 装饰器示例故意简化，省略了很多关于类型细节。

类型化装饰器可能会相当复杂。
例如，上面 `loggedMethod` 的一个类型良好的版本可能如下所示：
```ts
function loggedMethod<This, Args extends any[], Return>(
    target: (this: This, ...args: Args) => Return,
    context: ClassMethodDecoratorContext<This, (this: This, ...args: Args) => Return>
) {
    const methodName = String(context.name);

    function replacementMethod(this: This, ...args: Args): Return {
        console.log(`LOG: Entering method '${methodName}'.`)
        const result = target.call(this, ...args);
        console.log(`LOG: Exiting method '${methodName}'.`)
        return result;
    }

    return replacementMethod;
}
```
我们不得不分别对原始方法的`this`类型、参数和返回类型进行建模，使用类型参数`This`、`Args`和`Return`。

你的装饰器函数定义的复杂程度取决于你想要保证什么。
只要记住，你的装饰器将被使用的次数远多于它们被编写的次数，因此一个类型良好的版本通常更受欢迎 - 但显然这与可读性之间存在权衡，所以尽量保持简单。

将来会有更多关于编写装饰器的文档 - 但是[这篇帖子](https://2ality.com/2022/10/javascript-decorators.html)应该对装饰器的机制有相当详细的描述。

## `const` 类型参数

在推断对象的类型时，TypeScript通常会选择一个通用的类型。
例如，在这种情况下，`names`的推断类型是`string[]`：
```ts
type HasNames = { names: readonly string[] };
function getNamesExactly<T extends HasNames>(arg: T): T["names"] {
    return arg.names;
}

// Inferred type: string[]
const names = getNamesExactly({ names: ["Alice", "Bob", "Eve"]});
```
通常，这样做的目的是为了使后续的变异成为可能。

然而，根据`getNamesExactly`的实际功能和预期用途，通常情况下，可能需要更具体的类型。

到目前为止，API作者通常建议在某些地方添加`as const`以实现期望的推断：
```ts
// The type we wanted:
//    readonly ["Alice", "Bob", "Eve"]
// The type we got:
//    string[]
const names1 = getNamesExactly({ names: ["Alice", "Bob", "Eve"]});

// Correctly gets what we wanted:
//    readonly ["Alice", "Bob", "Eve"]
const names2 = getNamesExactly({ names: ["Alice", "Bob", "Eve"]} as const);
```
这可能会很麻烦，而且容易忘记。
在 TypeScript 5.0 中，您现在可以在类型参数声明中添加一个 `const` 修饰符，以使 `const` 类似的推断成为默认值：
```ts
type HasNames = { names: readonly string[] };
function getNamesExactly<const T extends HasNames>(arg: T): T["names"] {
//                       ^^^^^
    return arg.names;
}

// Inferred type: readonly ["Alice", "Bob", "Eve"]
// Note: Didn't need to write 'as const' here
const names = getNamesExactly({ names: ["Alice", "Bob", "Eve"] });
```
请注意，`const` 修饰符并不*拒绝*可变值，也不要求不可变约束。
使用可变类型约束可能会导致意外的结果。
例如：
```ts
declare function fnBad<const T extends string[]>(args: T): void;

// 'T' is still 'string[]' since 'readonly ["a", "b", "c"]' is not assignable to 'string[]'
fnBad(["a", "b" ,"c"]);
```
在这里，推断出的`T`的候选类型是`readonly ["a", "b", "c"]`，而`readonly`数组不能用在需要可变数组的地方。
在这种情况下，推断会回退到约束，数组被视为`string[]`，调用仍然可以成功进行。

这个函数的更好定义应该使用`readonly string[]`：
```ts
declare function fnGood<const T extends readonly string[]>(args: T): void;

// T is readonly ["a", "b", "c"]
fnGood(["a", "b" ,"c"]);
```
同样，请记住，`const` 修饰符仅影响在调用中编写的对象、数组和原始表达式的推断，因此不会（或不能）使用 `as const` 修改的参数不会看到任何行为变化：
```ts
declare function fnGood<const T extends readonly string[]>(args: T): void;
const arr = ["a", "b" ,"c"];

// 'T' is still 'string[]'-- the 'const' modifier has no effect here
fnGood(arr);
```
[查看拉取请求](https://github.com/microsoft/TypeScript/pull/51865)以及（[第一个](https://github.com/microsoft/TypeScript/issues/30680)和第二个[第二个](https://github.com/microsoft/TypeScript/issues/41114)）激励问题的更多细节。

## 在 `extends` 中支持多个配置文件

在管理多个项目时，拥有一个其他 `tsconfig.json` 文件可以扩展的“基础”配置文件会很有帮助。
这就是为什么 TypeScript 支持 `extends` 字段，用于从 `compilerOptions` 复制字段。

```jsonc
// packages/front-end/src/tsconfig.json
{
    "extends": "../../../tsconfig.base.json",
    "compilerOptions": {
        "outDir": "../lib",
        // ...
    }
}
```

然而，在某些情况下，您可能希望从多个配置文件扩展。
例如，想象使用 [一个发布到 npm 的 TypeScript 基础配置文件](https://github.com/tsconfig/bases)。
如果您希望所有项目都使用 npm 上的 `@tsconfig/strictest` 包中的选项，那么有一个简单的解决方案：让 `tsconfig.base.json` 从 `@tsconfig/strictest` 扩展：

```jsonc
// tsconfig.base.json
{
    "extends": "@tsconfig/strictest/tsconfig.json",
    "compilerOptions": {
        // ...
    }
}
```

这在某种程度上是可行的。
如果您有任何*不*希望使用 `@tsconfig/strictest` 的项目，它们必须手动禁用选项，或者创建一个*不*从 `@tsconfig/strictest` 扩展的 `tsconfig.base.json` 的单独版本。

为了在这里提供更多的灵活性，TypeScript 5.0 现在允许 `extends` 字段接受多个条目。
例如，在这个配置文件中：

```jsonc
{
    "extends": ["a", "b", "c"],
    "compilerOptions": {
        // ...
    }
}
```

这样写有点像直接扩展 `c`，其中 `c` 扩展 `b`，`b` 扩展 `a`。
如果任何字段“冲突”，则后一个条目获胜。

所以在以下示例中，`strictNullChecks` 和 `noImplicitAny` 都在最终的 `tsconfig.json` 中启用。

```jsonc
// tsconfig1.json
{
    "compilerOptions": {
        "strictNullChecks": true
    }
}

// tsconfig2.json
{
    "compilerOptions": {
        "noImplicitAny": true
    }
}

// tsconfig.json
{
    "extends": ["./tsconfig1.json", "./tsconfig2.json"],
    "files": ["./index.ts"]
}
```

作为另一个例子，我们可以用以下方式重写我们最初的例子。

```jsonc
// packages/front-end/src/tsconfig.json
{
    "extends": ["@tsconfig/strictest/tsconfig.json", "../../../tsconfig.base.json"],
    "compilerOptions": {
        "outDir": "../lib",
        // ...
    }
}
```

更多细节，请[阅读原始拉取请求上的更多信息](https://github.com/microsoft/TypeScript/pull/50403)。

<!--

## 改进的类型参数推断

TODO

## 在 `--noUncheckedIndexedAccess` 下改进的 `in` 检查

TODO

-->

## 所有 `enum` 都是联合 `enum`

当 TypeScript 最初引入枚举时，它们不过是一组具有相同类型的数字常量。
```ts
enum E {
    Foo = 10,
    Bar = 20,
}
```
`E.Foo` 和 `E.Bar` 的唯一特殊之处在于它们可以赋值给任何期望 `E` 类型的值。除此之外，它们基本上只是 `number` 类型。
```ts
function takeValue(e: E) {}

takeValue(E.Foo); // works
takeValue(123); // error!
```
直到 TypeScript 2.0 引入了枚举字面量类型，枚举才变得更具特色。
枚举字面量类型为每个枚举成员赋予了其自己的类型，并将枚举本身转换为每个成员类型的*联合*。
它们还允许我们仅引用枚举类型的一个子集，并缩小这些类型。
```ts
// Color is like a union of Red | Orange | Yellow | Green | Blue | Violet
enum Color {
    Red, Orange, Yellow, Green, Blue, /* Indigo, */ Violet
}

// Each enum member has its own type that we can refer to!
type PrimaryColor = Color.Red | Color.Green | Color.Blue;

function isPrimaryColor(c: Color): c is PrimaryColor {
    // Narrowing literal types can catch bugs.
    // TypeScript will error here because
    // we'll end up comparing 'Color.Red' to 'Color.Green'.
    // We meant to use ||, but accidentally wrote &&.
    return c === Color.Red && c === Color.Green && c === Color.Blue;
}
```
给每个枚举成员自己的类型存在一个问题，即这些类型在某种程度上与成员的实际值相关联。
在某些情况下，无法计算该值，例如，枚举成员可能通过函数调用进行初始化。
```ts
enum E {
    Blah = Math.random()
}
```
每当 TypeScript 遇到这些问题时，它会悄悄地退出并使用旧的枚举策略。
这意味着放弃联合类型和字面类型的所有优势。

TypeScript 5.0 通过为每个计算成员创建唯一类型，成功将所有枚举转换为联合枚举。
这意味着所有枚举现在都可以被缩小，并且它们的成员也可以作为类型引用。

有关此更改的更多详细信息，您可以[在 GitHub 上阅读具体信息](https://github.com/microsoft/TypeScript/pull/50528)。

## `--moduleResolution bundler`

TypeScript 4.7 引入了 `--module` 和 `--moduleResolution` 设置的 `node16` 和 `nodenext` 选项。
这些选项的目的是更好地模拟 Node.js 中 ECMAScript 模块的精确查找规则；
然而，这种模式有许多其他工具实际上并不强制执行的限制。

例如，在 Node.js 的 ECMAScript 模块中，任何相对导入都需要包含文件扩展名。
```ts
// entry.mjs
import * as utils from "./utils";     // ❌ wrong - we need to include the file extension.

import * as utils from "./utils.mjs"; // ✅ works
```
在 Node.js 和浏览器中，这样做有一定的原因——它使文件查找更快，对于简单的文件服务器来说效果更好。
但对于许多使用打包工具的开发者来说，`node16`/`nodenext` 设置很麻烦，因为打包工具没有这些限制。
在某些方面，对于使用打包工具的人来说，原始的 `node` 解析模式更好。

但在某些方面，原始的 `node` 解析模式已经过时了。
大多数现代打包工具使用 Node.js 中的 ECMAScript 模块和 CommonJS 查找规则的融合。
例如，像在 CommonJS 中一样，无扩展名的导入可以正常工作，但在查看包的[`export` 条件](https://nodejs.org/api/packages.html#nested-conditions)时，它们会优先选择 `import` 条件，就像在 ECMAScript 文件中一样。

为了模拟打包工具的工作方式，TypeScript 现在引入了一种新策略：`--moduleResolution bundler`。

```jsonc
{
    "compilerOptions": {
        "target": "esnext",
        "moduleResolution": "bundler"
    }
}
```

如果您正在使用像 Vite、esbuild、swc、Webpack、Parcel 等实现混合查找策略的现代打包工具，新的 `bundler` 选项应该非常适合您。

另一方面，如果您正在编写一个旨在发布到 npm 的库，使用 `bundler` 选项可能会隐藏可能出现在*不使用*打包工具的用户身上的兼容性问题。
因此，在这些情况下，使用 `node16` 或 `nodenext` 解析选项可能是更好的选择。

要了解更多关于 `--moduleResolution bundler` 的信息，[请查看实现拉取请求](https://github.com/microsoft/TypeScript/pull/51669)。

## 解析定制标志

JavaScript 工具现在可以模拟“混合”解析规则，就像我们上面描述的 `bundler` 模式一样。
由于工具在支持方面可能略有不同，TypeScript 5.0 提供了启用或禁用一些可能与您的配置兼容或不兼容的功能的方法。

### `allowImportingTsExtensions`

`--allowImportingTsExtensions` 允许 TypeScript 文件使用 TypeScript 特定扩展名（如 `.ts`、`.mts` 或 `.tsx`）相互导入。

此标志仅在启用 `--noEmit` 或 `--emitDeclarationOnly` 时允许，因为这些导入路径在 JavaScript 输出文件中无法在运行时解析。
这里的期望是您的解析器（例如您的打包工具、运行时或其他工具）将使这些 `.ts` 文件之间的导入工作。

### `resolvePackageJsonExports`

`--resolvePackageJsonExports` 强制 TypeScript 在从 `node_modules` 中的包读取时，查阅 [package.json 文件的 `exports` 字段](https://nodejs.org/api/packages.html#exports)。

在 `--moduleResolution` 的 `node16`、`nodenext` 和 `bundler` 选项下，此选项默认为 `true`。

### `resolvePackageJsonImports`

`--resolvePackageJsonImports` 强制 TypeScript 在从包含 `package.json` 的祖先目录的文件开始，以 `#` 开头的查找时，查阅 [package.json 文件的 `imports` 字段](https://nodejs.org/api/packages.html#imports)。

在 `--moduleResolution` 的 `node16`、`nodenext` 和 `bundler` 选项下，此选项默认为 `true`。

### `allowArbitraryExtensions`

在 TypeScript 5.0 中，当导入路径以不是已知的 JavaScript 或 TypeScript 文件扩展名的扩展名结尾时，编译器将在形式为 `{file basename}.d.{extension}.ts` 的路径中查找该路径的声明文件。
例如，如果您在打包工具项目中使用 CSS 加载器，您可能希望编写（或生成）这些样式表的声明文件：

```css
/* app.css */
.cookie-banner {
  display: none;
}
```
```ts
// app.d.css.ts
declare const css: {
  cookieBanner: string;
};
export default css;
```

```ts
// App.tsx
import styles from "./app.css";

styles.cookieBanner; // string
```
默认情况下，这种导入会引发一个错误，以通知您 TypeScript 无法理解此文件类型，您的运行时可能不支持导入它。
但是，如果您已经配置了运行时或打包器来处理它，您可以使用新的 `--allowArbitraryExtensions` 编译器选项来抑制错误。

请注意，从历史上看，通常可以通过添加名为 `app.css.d.ts` 而不是 `app.d.css.ts` 的声明文件来实现类似的效果 - 然而，这只是通过 Node 的 `require` 解析规则为 CommonJS 工作的。
严格来说，前者被解释为名为 `app.css.js` 的 JavaScript 文件的声明文件。
因为相对文件导入需要在 Node 的 ESM 支持中包含扩展名，所以在 `--moduleResolution node16` 或 `nodenext` 下的 ESM 文件中，TypeScript 会对我们的示例报错。

有关更多信息，请阅读有关此功能的[提案](https://github.com/microsoft/TypeScript/issues/50133)和[相应的拉取请求](https://github.com/microsoft/TypeScript/pull/51435)。

### `customConditions`

`--customConditions` 接受一个额外的[条件](https://nodejs.org/api/packages.html#nested-conditions)列表，当 TypeScript 从 `package.json` 的 [`exports`](https://nodejs.org/api/packages.html#exports) 或 [`imports`](https://nodejs.org/api/packages.html#imports) 字段解析时，这些条件应该成功。
这些条件将添加到解析器默认使用的任何现有条件中。

例如，当在 `tsconfig.json` 中设置此字段如下时：

```jsonc
{
    "compilerOptions": {
        "target": "es2022",
        "moduleResolution": "bundler",
        "customConditions": ["my-condition"]
    }
}
```

任何时候在 `package.json` 中引用 `exports` 或 `imports` 字段时，TypeScript 将考虑称为 `my-condition` 的条件。

因此，在从具有以下 `package.json` 的包中导入时

```jsonc
{
    // ...
    "exports": {
        ".": {
            "my-condition": "./foo.mjs",
            "node": "./bar.mjs",
            "import": "./baz.mjs",
            "require": "./biz.mjs"
        }
    }
}
```

TypeScript 将尝试查找与 `foo.mjs` 对应的文件。

此字段仅在 `--moduleResolution` 的 `node16`、`nodenext` 和 `bundler` 选项下有效。

## `--verbatimModuleSyntax`

默认情况下，TypeScript 执行一种称为*导入省略*的操作。
基本上，如果您编写如下内容
```ts
import { Car } from "./car";

export function drive(car: Car) {
    // ...
}
```
TypeScript检测到您仅使用导入的类型，并完全删除了导入。
您的输出JavaScript可能如下所示：
```ts
export function drive(car) {
    // ...
}
```
大多数情况下，这是好的，因为如果 `Car` 不是从 `./car` 导出的值，我们将得到一个运行时错误。

但它确实为某些边缘情况增加了一层复杂性。例如，请注意没有像 `import "./car";` 这样的语句 - 导入完全被省略了。这对于有副作用或没有副作用的模块实际上是有区别的。

TypeScript 对 JavaScript 的发射策略还有另外几层复杂性 - 导入省略并不总是仅仅由导入的使用方式驱动 - 它通常还会考虑值的声明方式。所以，像以下这样的代码并不总是清晰的
```ts
export { Car } from "./car";
```
应当保留还是丢弃。
如果`Car`是用类似`class`的东西声明的，那么它可以在生成的JavaScript文件中保留。
但如果`Car`仅声明为`type`别名或`interface`，那么JavaScript文件根本不应该导出`Car`。

尽管TypeScript可能能够根据跨文件的信息做出这些发射决策，但并非每个编译器都能这样做。

导入和导出上的`type`修饰符有助于解决这些情况。
我们可以明确表示一个导入或导出仅用于类型分析，并且可以通过使用`type`修饰符在JavaScript文件中完全丢弃。
```ts
// This statement can be dropped entirely in JS output
import type * as car from "./car";

// The named import/export 'Car' can be dropped in JS output
import { type Car } from "./car";
export { type Car } from "./car";
```
类型（`type`）修饰符本身并不是很有用 - 默认情况下，模块省略（module elision）仍然会删除导入，而且没有任何东西强制你区分 `type` 和普通导入和导出。
因此，TypeScript 提供了 `--importsNotUsedAsValues` 标志来确保你使用 `type` 修饰符，`--preserveValueImports` 标志来防止 *某些* 模块省略行为，以及 `--isolatedModules` 标志来确保你的 TypeScript 代码在不同编译器之间可以正常工作。
不幸的是，理解这三个标志的细节很困难，仍然存在一些意外行为的边缘情况。

TypeScript 5.0 引入了一个名为 `--verbatimModuleSyntax` 的新选项来简化这种情况。
规则要简单得多 - 任何没有 `type` 修饰符的导入或导出都会被保留。
任何使用 `type` 修饰符的内容都会被完全删除。
```ts
// Erased away entirely.
import type { A } from "a";

// Rewritten to 'import { b } from "bcd";'
import { b, type c, type d } from "bcd";

// Rewritten to 'import {} from "xyz";'
import { type xyz } from "xyz";
```
有了这个新选项，所见即所得。

然而，当涉及到模块互操作性时，这确实有一些含义。
在这个标志下，当你设置或文件扩展名暗示了不同的模块系统时，ECMAScript的`import`和`export`将不会被重写为`require`调用。
相反，你将得到一个错误。
如果你需要发出使用`require`和`module.exports`的代码，你将不得不使用TypeScript的模块语法，该语法早于ES2015：

<table>
<thead>
    <tr>
        <th>输入 TypeScript</th>
        <th>输出 JavaScript</th>
    </tr>
</thead>

<tr>
<td>
```ts
import foo = require("foo");
```
{"Role":{"name":"文档翻译者","Profile":{"Language":"中文","Description":"将输入的英文段落翻译为中文","Skills":["熟悉英文","熟悉中文","熟悉 TypeScript"]}}}
```ts
const foo = require("foo");
```
角色：文档翻译者
<br>
<br>
个人资料：
<br>
语言：中文
<br>
描述：将输入的英文段落翻译为中文
<br>
技能：
<br>
- 熟悉英文
<br>
- 熟悉中文
<br>
- 熟悉 TypeScript
```ts
function foo() {}
function bar() {}
function baz() {}

export = {
    foo,
    bar,
    baz
};
```
{"Role":{"name":"文档翻译者","Profile":{"Language":"中文","Description":"将输入的英文段落翻译为中文","Skills":["熟悉英文","熟悉中文","熟悉 TypeScript"]}}}
```ts
function foo() {}
function bar() {}
function baz() {}

module.exports = {
    foo,
    bar,
    baz
};
```
虽然这是一个限制，但它确实有助于使一些问题更加明显。
例如，在 `--module node16` 下忘记设置 [`package.json` 中的 `type` 字段](https://nodejs.org/api/packages.html#type) 是非常常见的。
结果，开发人员会开始编写 CommonJS 模块，而不是意识到自己在编写 ES 模块，从而产生令人惊讶的查找规则和 JavaScript 输出。
这个新标志确保你对自己使用的文件类型是有意识的，因为语法是有意不同的。

因为 `--verbatimModuleSyntax` 提供了比 `--importsNotUsedAsValues` 和 `--preserveValueImports` 更一致的故事，所以这两个现有的标志将被弃用，以支持它。

有关更多详细信息，请阅读 [原始拉取请求](https://github.com/microsoft/TypeScript/pull/52203) 和 [其提案问题](https://github.com/microsoft/TypeScript/issues/51479)。

## 支持 `export type *`

当 TypeScript 3.8 引入仅类型导入时，新语法不允许在 `export * from "module"` 或 `export * as ns from "module"` 重导出上使用。TypeScript 5.0 添加了对这两种形式的支持：
```ts
// models/vehicles.ts
export class Spaceship {
  // ...
}

// models/index.ts
export type * as vehicles from "./vehicles";

// main.ts
import { vehicles } from "./models";

function takeASpaceship(s: vehicles.Spaceship) {
  // ✅ ok - `vehicles` only used in a type position
}

function makeASpaceship() {
  return new vehicles.Spaceship();
  //         ^^^^^^^^
  // 'vehicles' cannot be used as a value because it was exported using 'export type'.
}
```
您可以[在此处了解更多关于实现的信息](https://github.com/microsoft/TypeScript/pull/52217)。

## JSDoc 中的 `@satisfies` 支持

TypeScript 4.9 引入了 `satisfies` 运算符。
它确保表达式的类型兼容，而不影响类型本身。
例如，让我们看以下代码：
```ts
interface CompilerOptions {
    strict?: boolean;
    outDir?: string;
    // ...
}

interface ConfigSettings {
    compilerOptions?: CompilerOptions;
    extends?: string | string[];
    // ...
}

let myConfigSettings = {
    compilerOptions: {
        strict: true,
        outDir: "../lib",
        // ...
    },

    extends: [
        "@tsconfig/strictest/tsconfig.json",
        "../../../tsconfig.base.json"
    ],

} satisfies ConfigSettings;
```
在这里，TypeScript 知道 `myConfigSettings.extends` 是用数组声明的——因为虽然 `satisfies` 验证了我们对象的类型，但它并没有将其直接更改为 `CompilerOptions` 并丢失信息。
所以，如果我们想要遍历 `extends`，那是可以的。
```ts
declare function resolveConfig(configPath: string): CompilerOptions;

let inheritedConfigs = myConfigSettings.extends.map(resolveConfig);
```
这对 TypeScript 用户很有帮助，但很多人使用 TypeScript 通过 JSDoc 注释来对他们的 JavaScript 代码进行类型检查。
这就是为什么 TypeScript 5.0 支持一个新的 JSDoc 标签 `@satisfies`，它的功能完全相同。

`/** @satisfies */` 可以捕获类型不匹配：
```ts
// @ts-check

/**
 * @typedef CompilerOptions
 * @prop {boolean} [strict]
 * @prop {string} [outDir]
 */

/**
 * @satisfies {CompilerOptions}
 */
let myCompilerOptions = {
    outdir: "../lib",
//  ~~~~~~ oops! we meant outDir
};
```
但它将保留我们表达式的原始类型，允许我们稍后在代码中更精确地使用我们的值。
```ts
// @ts-check

/**
 * @typedef CompilerOptions
 * @prop {boolean} [strict]
 * @prop {string} [outDir]
 */

/**
 * @typedef ConfigSettings
 * @prop {CompilerOptions} [compilerOptions]
 * @prop {string | string[]} [extends]
 */


/**
 * @satisfies {ConfigSettings}
 */
let myConfigSettings = {
    compilerOptions: {
        strict: true,
        outDir: "../lib",
    },
    extends: [
        "@tsconfig/strictest/tsconfig.json",
        "../../../tsconfig.base.json"
    ],
};

let inheritedConfigs = myConfigSettings.extends.map(resolveConfig);
```
`/** @satisfies */` 也可以在任何括号表达式中内联使用。
我们可以这样写 `myCompilerOptions`：
```ts
let myConfigSettings = /** @satisfies {ConfigSettings} */ ({
    compilerOptions: {
        strict: true,
        outDir: "../lib",
    },
    extends: [
        "@tsconfig/strictest/tsconfig.json",
        "../../../tsconfig.base.json"
    ],
});
```
为什么？
嗯，通常当你更深入其他代码时，比如函数调用，它更有意义。
```ts
compileCode(/** @satisfies {CompilerOptions} */ ({
    // ...
}));
```
此功能（[链接](https://github.com/microsoft/TypeScript/pull/51753)）得益于[Oleksandr Tarasiuk](https://github.com/a-tarasyuk)的贡献！

## JSDoc 中的 `@overload` 支持

在 TypeScript 中，您可以为函数指定重载。
重载提供了一种方法，说明函数可以用不同的参数调用，并可能返回不同的结果。
它们可以限制调用者实际使用我们函数的方式，并优化他们将获得的结果。
```ts
// Our overloads:
function printValue(str: string): void;
function printValue(num: number, maxFractionDigits?: number): void;

// Our implementation:
function printValue(value: string | number, maximumFractionDigits?: number) {
    if (typeof value === "number") {
        const formatter = Intl.NumberFormat("en-US", {
            maximumFractionDigits,
        });
        value = formatter.format(value);
    }

    console.log(value);
}
```
在这里，我们说过 `printValue` 函数接受一个 `string` 或 `number` 类型的第一个参数。
如果它接受一个 `number` 类型的参数，那么它可以接受第二个参数来确定我们可以打印多少位小数。

TypeScript 5.0 现在允许使用新的 `@overload` 标签在 JSDoc 中声明重载。
每个带有 `@overload` 标签的 JSDoc 注释都被视为后续函数声明的一个不同的重载。
```ts
// @ts-check

/**
 * @overload
 * @param {string} value
 * @return {void}
 */

/**
 * @overload
 * @param {number} value
 * @param {number} [maximumFractionDigits]
 * @return {void}
 */

/**
 * @param {string | number} value
 * @param {number} [maximumFractionDigits]
 */
function printValue(value, maximumFractionDigits) {
    if (typeof value === "number") {
        const formatter = Intl.NumberFormat("en-US", {
            maximumFractionDigits,
        });
        value = formatter.format(value);
    }

    console.log(value);
}
```
现在，无论我们是在 TypeScript 文件还是 JavaScript 文件中编写代码，TypeScript 都可以告诉我们是否正确地调用了函数。
```ts
// all allowed
printValue("hello!");
printValue(123.45);
printValue(123.45, 2);

printValue("hello!", 123); // error!
```
这个新标签[已实现](https://github.com/microsoft/TypeScript/pull/51234)，感谢[Tomasz Lenarcik](https://github.com/apendua)。

## 在`--build`模式下传递特定于生成的标志

TypeScript现在允许在`--build`模式下传递以下标志：

* `--declaration`
* `--emitDeclarationOnly`
* `--declarationMap`
* `--sourceMap`
* `--inlineSourceMap`

这使得在构建过程中定制某些部分变得更加容易，您可能有不同的开发和生产构建。

例如，库的开发构建可能不需要生成声明文件，但生产构建则需要。
项目可以配置默认关闭声明生成，只需使用以下命令构建：

```sh
tsc --build -p ./my-project-dir
```

在内循环迭代完成后，只需传递`--declaration`标志即可进行“生产”构建。

```sh
tsc --build -p ./my-project-dir --declaration
```

[有关此更改的更多信息，请访问此处](https://github.com/microsoft/TypeScript/pull/51241)。

## 编辑器中的不区分大小写的导入排序

在Visual Studio和VS Code等编辑器中，TypeScript为组织和排序导入和导出提供了支持。
然而，通常对于何时认为列表“排序”可能有不同的解释。

例如，以下导入列表是否已排序？
```ts
import {
    Toggle,
    freeze,
    toBoolean,
} from "./utils";
```
答案可能会出人意料地是“这取决于情况”。
如果我们*不*关心大小写敏感性，那么这个列表显然是没有排序的。
字母`f`在`t`和`T`之前。

但在大多数编程语言中，排序默认是比较字符串的字节值。
JavaScript比较字符串的方式意味着`"Toggle"`总是排在`"freeze"`之前，因为根据[ASCII字符编码](https://en.wikipedia.org/wiki/ASCII)，大写字母排在小写字母之前。
所以从这个角度来看，导入列表是排序的。

TypeScript之前认为导入列表是排序的，因为它进行了基本的区分大小写排序。
这可能是那些更喜欢不区分大小写排序的开发者，或者使用像ESLint这样的工具（默认要求不区分大小写排序）的开发者的挫败点。

TypeScript现在默认检测大小写敏感性。
这意味着TypeScript和像ESLint这样的工具通常不会在如何最好地排序导入上“争执”。

我们的团队还在尝试[进一步排序策略，您可以在这里阅读相关信息](https://github.com/microsoft/TypeScript/pull/52115)。
这些选项最终可能会由编辑器配置。
目前，它们仍然是不稳定和实验性的，您可以通过在您的JSON选项中使用`typescript.unstable`条目在VS Code中尝试它们。
以下是您可以尝试的所有选项（设置为默认值）：

```jsonc
{
    "typescript.unstable": {
        // 是否应该区分大小写进行排序？可以是：
        // - true
        // - false
        // - "auto"（自动检测）
        "organizeImportsIgnoreCase": "auto",

        // 排序应该是“序数”并使用码点还是考虑Unicode规则？可以是：
        // - "ordinal"
        // - "unicode"
        "organizeImportsCollation": "ordinal",

        // 在`"organizeImportsCollation": "unicode"`下，
        // 当前的区域设置是什么？可以是：
        // - [任何其他区域代码]
        // - "auto"（使用编辑器的区域设置）
        "organizeImportsLocale": "en",

        // 在`"organizeImportsCollation": "unicode"`下，
        // 大写字母还是小写字母应该排在前面？可以是：
        // - false（特定于区域的）
        // - "upper"
        // - "lower"
        "organizeImportsCaseFirst": false,

        // 在`"organizeImportsCollation": "unicode"`下，
        // 数字序列是否按数值比较（即 "a1" < "a2" < "a100"）？可以是：
        // - true
        // - false
        "organizeImportsNumericCollation": true,

        // 在`"organizeImportsCollation": "unicode"`下，
        // 带重音符号/变音符的字母是否与它们的“基本”字母明显区分排序
        //（即 é 是否与 e 不同）？可以是：
        // - true
        // - false
        "organizeImportsAccentCollation": true
    },
    "javascript.unstable": {
        // 这里也有相同的选项...
    },
}
```

您可以在[自动检测和指定不区分大小写的原始工作](https://github.com/microsoft/TypeScript/pull/51733)上阅读更多细节，然后是[更广泛的选项集](https://github.com/microsoft/TypeScript/pull/52115)。

## 详尽的 `switch`/`case` 补全

在编写 `switch` 语句时，TypeScript 现在会检测被检查的值是否具有字面类型。
如果是这样，它将提供一个补全，该补全会为每个未覆盖的 `case` 生成框架。

![通过基于字面类型的自动补全生成的一组 `case` 语句。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/01/switchCaseSnippets-5-0_1.gif)

您可以在 [GitHub 上查看实现的详细信息](https://github.com/microsoft/TypeScript/pull/50996)。

## 速度、内存和包大小优化

TypeScript 5.0 包含了许多跨代码结构、数据结构和算法实现的强大变化。
所有这些意味着您的整个体验应该更快 - 不仅是运行 TypeScript，甚至安装它也是如此。

以下是我们相对于 TypeScript 4.9 能够捕捉到的一些有趣的速度和大小方面的胜利。

场景 | 相对于 TS 4.9 的时间或大小
---------|--------------------
material-ui 构建时间 | 89%
TypeScript 编译器启动时间 | 89%
Playwright 构建时间 | 88%
TypeScript 编译器自我构建时间 | 87%
Outlook Web 构建时间 | 82%
VS Code 构建时间 | 80%
typescript npm 包大小 | 59%

![TypeScript 5.0 相对于 TypeScript 4.9 的构建/运行时间和包大小图表：material-ui 文档构建时间：89%；Playwright 构建时间：88%；tsc 启动时间：87%；tsc 构建时间：87%；Outlook Web 构建时间：82%；VS Code 构建时间：80%；typescript 包大小：59%](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/03/speed-and-size-5-0-rc.png?1)

这是如何实现的？
我们有一些值得在未来详细说明的显著改进。
但我们不会让您等待那篇博客文章。

首先，我们最近将 TypeScript 从命名空间迁移到模块，这使我们能够利用现代构建工具进行优化，如作用域提升。
使用这些工具，重新审视我们的打包策略，并删除一些已弃用的代码，已经从 TypeScript 4.9 的 63.8 MB 包大小中削减了大约 26.4 MB。
它还通过直接函数调用带来了显著的速度提升。

TypeScript 还增加了编译器内部对象类型的更多一致性，并减少了一些对象类型上存储的数据。
这减少了多态和巨型态的使用场所，同时抵消了统一形状所需的大部分必要内存消耗。

我们还在序列化信息到字符串时进行了一些缓存。
类型显示，可能作为错误报告、声明发射、代码补全等的一部分发生，最终可能会相当昂贵。
TypeScript 现在缓存了一些常用的机制以在这些操作中重用。

我们做出的另一个显著改进是偶尔利用 `var` 来规避使用 `let` 和 `const` 跨越闭包的成本。
这提高了我们的一些解析性能。

总的来说，我们预计大多数代码库应该从 TypeScript 5.0 中看到速度提升，并且我们一直能够一致地重现 10% 到 20% 之间的胜利。
当然，这将取决于硬件和代码库特性，但我们鼓励您今天就在您的代码库上尝试它！

有关更多信息，请查看我们的一些显著优化：

* [迁移到模块](https://github.com/microsoft/TypeScript/pull/51387)
* [`Node` 单态化](https://github.com/microsoft/TypeScript/pull/51682)
* [`Symbol` 单态化](https://github.com/microsoft/TypeScript/pull/51880)
* [`Identifier` 大小减小](https://github.com/microsoft/TypeScript/pull/52170)
* [`Printer` 缓存](https://github.com/microsoft/TypeScript/pull/52382)
* [有限使用 `var`](https://github.com/microsoft/TypeScript/issues/52924)

## 破坏性更改和弃用

### 运行时要求

TypeScript 现在针对 ECMAScript 2018。
对于 Node 用户，这意味着至少需要 Node.js 10 及更高版本的最低版本要求。

### `lib.d.ts` 更改

对 DOM 类型的生成方式的变化可能会影响现有代码。
值得注意的是，某些属性已从 `number` 转换为数值字面类型，剪切、复制和粘贴事件处理的属性和方法已跨接口移动。

### API 破坏性更改

在 TypeScript 5.0 中，我们转向模块，删除了一些不必要的接口，并进行了一些正确性改进。
有关更改的更多详细信息，请查看我们的 [API 破坏性更改](https://github.com/microsoft/TypeScript/wiki/API-Breaking-Changes) 页面。

### 在关系运算符中禁止隐式强制转换

在某些操作中，如果您的代码可能导致隐式的字符串到数字强制转换，TypeScript 将会向您发出警告：
```ts
function func(ns: number | string) {
  return ns * 4; // Error, possible implicit coercion
}
```
在5.0中，这也将应用于关系运算符`>`, `<`, `<=`和`>=`：
```ts
function func(ns: number | string) {
  return ns > 4; // Now also an error
}
```
为了在需要时允许这样做，您可以使用 `+` 显式地将操作数强制转换为 `number`：
```ts
function func(ns: number | string) {
  return +ns > 4; // OK
}
```
这个[正确性改进](https://github.com/microsoft/TypeScript/pull/52048)是由[Mateusz Burzyński](https://github.com/Andarist)提供的。

### 枚举类型大修

自从首次发布以来，TypeScript在`enum`类型上一直存在一些长期的问题。
在5.0版本中，我们将解决这些问题，并减少理解您可以声明的各种`enum`类型所需的概念数量。

作为此更改的一部分，您可能会看到两个新的错误。
第一个是，将域外的字面量分配给`enum`类型现在会报错，这是人们所期望的：
```ts
enum SomeEvenDigit {
    Zero = 0,
    Two = 2,
    Four = 4
}

// Now correctly an error
let m: SomeEvenDigit = 1;
```
另一个问题是，声明某些类型的间接混合字符串/数字 `enum` 形式会错误地创建一个全数字 `enum`：
```ts
enum Letters {
    A = "a"
}
enum Numbers {
    one = 1,
    two = Letters.A
}

// Now correctly an error
const t: number = Numbers.two;
```
您可以在[相关更改中查看更多详细信息](https://github.com/microsoft/TypeScript/pull/50528)。

### 在 `--experimentalDecorators` 下对构造函数参数装饰器进行更准确类型检查

TypeScript 5.0 在 `--experimentalDecorators` 选项下对装饰器的类型检查更加准确。
这在使用装饰器装饰构造函数参数时尤为明显。
```ts
export declare const inject:
  (entity: any) =>
    (target: object, key: string | symbol, index?: number) => void;

export class Foo {}

export class C {
    constructor(@inject(Foo) private x: any) {
    }
}
```
此调用将失败，因为 `key` 期望一个 `string | symbol`，但构造函数参数接收一个 `undefined` 的键。
正确的修复是在 `inject` 中更改 `key` 的类型。
如果您正在使用无法升级的库，一个合理的解决方案是包装 `inject` 在一个更类型安全的装饰器函数中，并对 `key` 使用类型断言。

更多细节，请[查看此问题](https://github.com/microsoft/TypeScript/issues/52435)。

### 弃用和默认更改

在 TypeScript 5.0 中，我们已弃用以下设置和设置值：

* `--target: ES3`
* `--out`
* `--noImplicitUseStrict`
* `--keyofStringsOnly`
* `--suppressExcessPropertyErrors`
* `--suppressImplicitAnyIndexErrors`
* `--noStrictGenericChecks`
* `--charset`
* `--importsNotUsedAsValues`
* `--preserveValueImports`
* 项目引用中的 `prepend`

这些配置将继续允许使用，直到 TypeScript 5.5，届时它们将被完全删除，但是，如果您正在使用这些设置，您将收到警告。
在 TypeScript 5.0 以及未来的 5.1、5.2、5.3 和 5.4 版本中，您可以指定 `"ignoreDeprecations": "5.0"` 来屏蔽这些警告。
我们还将很快发布一个 4.9 补丁，以允许指定 `ignoreDeprecations`，以实现更平滑的升级。
除了弃用之外，我们还更改了一些设置，以更好地改善 TypeScript 中的跨平台行为。

`--newLine`，用于控制 JavaScript 文件中发出的换行符，如果未指定，过去会根据当前操作系统推断。
我们认为构建应该尽可能确定，现在 Windows Notepad 支持换行符换行，所以新的默认设置是 `LF`。
旧的特定于操作系统的推断行为不再可用。

`--forceConsistentCasingInFileNames`，确保项目中对同一文件名的所有引用在大小写上一致，现在默认设置为 `true`。
这可以帮助捕获在大小写不敏感的文件系统上编写的代码的差异问题。

您可以留下反馈并查看有关 [5.0 弃用的跟踪问题](https://github.com/microsoft/TypeScript/issues/51909) 的更多信息。