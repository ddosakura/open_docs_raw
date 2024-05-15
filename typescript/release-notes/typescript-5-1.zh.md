---
source: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-1.html
---
# 文档 - TypeScript 5.1

## [](#更容易为返回 undefined 的函数实现隐式返回)更容易为返回 `undefined` 的函数实现隐式返回

在 JavaScript 中，如果一个函数在运行结束时没有遇到 `return` 语句，它将返回值 `undefined`。
```ts
function foo() {
  // no return
}
// x = undefined
let x = foo();
```
然而，在 TypeScript 的早期版本中，唯一不能有返回语句的函数是返回 `void` 和 `any` 类型的函数。这意味着，即使你明确声明“这个函数返回 `undefined`”，你仍然被迫至少有一个返回语句。
```ts
// ✅ fine - we inferred that 'f1' returns 'void'
function f1() {
  // no returns
}
// ✅ fine - 'void' doesn't need a return statement
function f2(): void {
  // no returns
}
// ✅ fine - 'any' doesn't need a return statement
function f3(): any {
  // no returns
}
// ❌ error!
// A function whose declared type is neither 'void' nor 'any' must return a value.
function f4(): undefined {
  // no returns
}
```
这可能会很麻烦，如果某个 API 期望一个返回 `undefined` 的函数 - 你需要至少有一个明确的返回 `undefined` 或者一个 `return` 语句 _和_ 一个明确的注解。
```ts
declare function takesFunction(f: () => undefined): undefined;
// ❌ error!
// Argument of type '() => void' is not assignable to parameter of type '() => undefined'.
takesFunction(() => {
  // no returns
});
// ❌ error!
// A function whose declared type is neither 'void' nor 'any' must return a value.
takesFunction((): undefined => {
  // no returns
});
// ❌ error!
// Argument of type '() => void' is not assignable to parameter of type '() => undefined'.
takesFunction(() => {
  return;
});
// ✅ works
takesFunction(() => {
  return undefined;
});
// ✅ works
takesFunction((): undefined => {
  return;
});
```
这种行为令人沮丧且令人困惑，尤其是在调用无法控制的函数时。理解推断`void`而非`undefined`之间的相互作用，以及一个返回`undefined`的函数是否需要`return`语句等，似乎分散了注意力。

首先，TypeScript 5.1现在允许返回`undefined`的函数没有返回语句。
```ts
// ✅ Works in TypeScript 5.1!
function f4(): undefined {
  // no returns
}
// ✅ Works in TypeScript 5.1!
takesFunction((): undefined => {
  // no returns
});
```
其次，如果一个函数没有返回表达式，并且被传递给一个期望返回 `undefined` 的函数，TypeScript 会推断该函数的返回类型为 `undefined`。
```ts
// ✅ Works in TypeScript 5.1!
takesFunction(function f() {
  //                 ^ return type is undefined
  // no returns
});
// ✅ Works in TypeScript 5.1!
takesFunction(function f() {
  //                 ^ return type is undefined
  return;
});
```
为了解决另一个类似的痛点，在 TypeScript 的 `--noImplicitReturns` 选项下，仅返回 `undefined` 的函数现在有一个类似于 `void` 的例外，即并非每个代码路径都必须以显式的 `return` 结尾。
```ts
// ✅ Works in TypeScript 5.1 under '--noImplicitReturns'!
function f(): undefined {
  if (Math.random()) {
    // do some stuff...
    return;
  }
}
```
有关更多信息，您可以阅读[原始问题](https://github.com/microsoft/TypeScript/issues/36288)和[实施拉取请求](https://github.com/microsoft/TypeScript/pull/53607)。

TypeScript 4.3 使得可以声明 `get` 和 `set` 访问器对可能指定两种不同的类型。
```ts
interface Serializer {
  set value(v: string | number | boolean);
  get value(): string;
}
declare let box: Serializer;
// Allows writing a 'boolean'
box.value = true;
// Comes out as a 'string'
console.log(box.value.toUpperCase());
```
最初，我们要求`get`类型必须是`set`类型的子类型。这意味着编写
```ts
box.value = box.value;
```
将始终有效。

然而，现有的和提议的 API 中有很多 getter 和 setter 之间的类型完全无关。例如，考虑最常见的例子之一 - DOM 中的 `style` 属性和 [`CSSStyleRule`](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleRule) API。每个样式规则都有一个 [a `style` 属性](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleRule/style)，它是一个 [`CSSStyleDeclaration`](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleDeclaration)；然而，如果你尝试写入该属性，它只会与字符串一起正确工作！

TypeScript 5.1 现在允许 `get` 和 `set` 访问器属性具有完全无关的类型，前提是有明确的类型注释。虽然这个版本的 TypeScript 还没有改变这些内置接口的类型，但 `CSSStyleRule` 现在可以按以下方式定义：
```ts

interface CSSStyleRule {
// ...
`/** Always reads as a `CSSStyleDeclaration` */`
get style(): CSSStyleDeclaration;
`/** Can only write a `string` here. */`
set style(newValue: string);
// ...
}
```
这也允许其他模式，例如要求`set`访问器仅接受“有效”数据，但指定如果某些底层状态尚未初始化，`get`访问器可能返回`undefined`。
```ts
class SafeBox {
  #value: string | undefined;
  // Only accepts strings!
  set value(newValue: string) {
  }
  // Must check for 'undefined'!
  get value(): string | undefined {
    return this.#value;
  }
}
```
实际上，这与在`--exactOptionalProperties`下检查可选属性的方式类似。

你可以阅读更多关于[实现拉取请求](https://github.com/microsoft/TypeScript/pull/53417)的信息。

## [](#decoupled-type-checking-between-jsx-elements-and-jsx-tag-types)JSX元素与JSX标签类型之间的解耦类型检查

TypeScript在处理JSX时的一个痛点是对每个JSX元素的标签类型的要求。

为了上下文，一个JSX元素要么是以下之一：

```tsx

// 一个自闭合的JSX标签
<Foo />
// 一个带有开闭标签的常规元素
<Bar></Bar>
```

在对`<Foo />`或`<Bar></Bar>`进行类型检查时，TypeScript总是查找一个名为`JSX`的命名空间，并从中获取一个名为`Element`的类型 - 或者更直接地说，它查找`JSX.Element`。

但是，为了检查`Foo`或`Bar`本身是否适合用作标签名，TypeScript大致只是获取或构造`Foo`或`Bar`返回的类型，并检查其与`JSX.Element`（或者如果该类型是可构造的，则检查与另一个名为`JSX.ElementClass`的类型）的兼容性。

这里的限制意味着，如果组件返回或“渲染”的类型比`JSX.Element`更广泛，那么这些组件就不能使用。例如，一个JSX库可能对组件返回`string`或`Promise`没有意见。

作为一个更具体的例子，[React正在考虑有限支持返回`Promise`的组件](https://github.com/acdlite/rfcs/blob/first-class-promises/text/0000-first-class-support-for-promises.md)，但现有版本的TypeScript无法在不大幅放宽`JSX.Element`类型的情况下表达这一点。

```tsx
import * as React from "react";
async function Foo() {
  return <div></div>;
}
let element = <Foo />;
//             ~~~
// 'Foo' 不能用作 JSX 组件。
//   它的返回类型 'Promise<Element>' 不是有效的 JSX 元素。
```

为了向库提供一种表达这一点的方法，TypeScript 5.1 现在查找一个名为 `JSX.ElementType` 的类型。`ElementType` 精确指定了在 JSX 元素中作为标签使用什么是有效的。所以今天它可能被类型化为类似以下内容

```tsx

namespace JSX {
export type ElementType =
// 所有有效的小写标签
keyof IntrinsicAttributes
// 函数组件
(props: any) => Element
// 类组件
new (props: any) => ElementClass;
export interface IntrinsicAttributes extends /*...*/ {}
export type Element = /*...*/;
export type ElementClass = /*...*/;
}
```

我们要感谢 [Sebastian Silbermann](https://github.com/eps1lon) 贡献了 [这一变更](https://github.com/microsoft/TypeScript/pull/51328)！

## [](#namespaced-jsx-attributes)命名空间 JSX 属性

TypeScript 现在在使用 JSX 时支持命名空间属性名称。

```tsx
import * as React from "react";
// 这两种写法都是等效的：
const x = <Foo a:b="hello" />;
const y = <Foo a:b="hello" />;
interface FooProps {
  "a:b": string;
}
function Foo(props: FooProps) {
  return <div>{props["a:b"]}</div>;
}
```

当名称的第一部分是小写名称时，命名空间标签名称在 `JSX.IntrinsicAttributes` 上以类似的方式查找。

```tsx
// 在某个库的代码中或对该库的增强中：
namespace JSX {
  interface IntrinsicElements {
    ["a:b"]: { prop: string };
  }
}
// 在我们的代码中：
let x = <a:b prop="hello!" />;
```

[这项贡献](https://github.com/microsoft/TypeScript/pull/53799) 感谢 [Oleksandr Tarasiuk](https://github.com/a-tarasyuk) 提供。

## [](#typeroots-are-consulted-in-module-resolution)`typeRoots` 在模块解析中被参考

当 TypeScript 指定的模块查找策略无法解析路径时，它现在将相对于指定的 `typeRoots` 解析包。

更多详情请参阅 [此拉取请求](https://github.com/microsoft/TypeScript/pull/51715)。

## [](#move-declarations-to-existing-files)将声明移动到现有文件

除了将声明移动到新文件之外，TypeScript 现在还推出了一个预览功能，用于将声明移动到现有文件。你可以在最新版本的 Visual Studio Code 中尝试此功能。

![Image 1: 将函数 'getThanks' 移动到工作区中的现有文件。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/05/moveToFile-5.1-preview.gif)

请注意，此功能目前处于预览状态，我们正在寻求进一步的反馈。

[https://github.com/microsoft/TypeScript/pull/53542](https://github.com/microsoft/TypeScript/pull/53542)

TypeScript 现在支持 JSX 标签名称的_链接编辑_。链接编辑（有时称为“镜像光标”）允许编辑器自动同时编辑多个位置。

![Image 2: 使用链接编辑修改 JSX 片段和 div 元素的 JSX 标签示例。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/04/linkedEditingJsx-5.1-1.gif)

这项新功能应该在 TypeScript 和 JavaScript 文件中都有效，并且可以在 Visual Studio Code Insiders 中启用。在 Visual Studio Code 中，你可以在设置 UI 中编辑 `Editor: Linked Editing` 选项：

![Image 3: Visual Studio Code 的 Editor: Linked Editing` 选项](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/04/linkedEditing-5.1-vscode-ui-1.png)

或者在 JSON 设置文件中配置 `editor.linkedEditing`：

```jsonc
{
  // ...
  "editor.linkedEditing": true
}
```

Visual Studio 17.7 Preview 1 也将支持此功能。

你可以在这里查看 [我们实现的链接编辑](https://github.com/microsoft/TypeScript/pull/53284)！

TypeScript 现在在 TypeScript 和 JavaScript 文件中输入 `@param` 标签时提供代码片段补全。这可以帮助减少一些打字和文本跳转，因为你在记录代码或在 JavaScript 中添加 JSDoc 类型时。

![Image 4: 在 'add' 函数上完成 JSDoc param 注释的示例。](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2023/04/paramTagSnippets-5-1-1.gif)

你可以在 [GitHub 上查看这个新功能是如何实现的](https://github.com/microsoft/TypeScript/pull/53260)。

## [](#optimizations)优化

### [](#avoiding-unnecessary-type-instantiation)避免不必要的类型实例化

TypeScript 5.1 现在避免了在已知不包含对外部类型参数引用的对象类型内执行类型实例化。这有可能减少许多不必要的计算，并将 [material-ui 文档目录](https://github.com/mui/material-ui/tree/b0351248fb396001a30330daac86d0e0794a0c1d/docs) 的类型检查时间减少了 50% 以上。

你可以在 [GitHub 上查看这个变更所涉及的内容](https://github.com/microsoft/TypeScript/pull/53246)。

### [](#negative-case-checks-for-union-literals)联合字面量的负面案例检查

在检查源类型是否是联合类型的一部分时，TypeScript 首先使用该源类型的内部类型标识符进行快速查找。如果该查找失败，则 TypeScript 检查与联合中的每个类型的兼容性。

在将字面量类型与纯字面量类型的联合相关联时，TypeScript 现在可以避免对联合中每个其他类型进行完整的遍历。这个假设是安全的，因为 TypeScript 总是内部化/缓存字面量类型 - 尽管有一些边缘情况需要处理，这些情况与“新鲜”的字面量类型有关。

[这项优化](https://github.com/microsoft/TypeScript/pull/53192) 将 [此问题中的代码](https://github.com/microsoft/TypeScript/issues/53191) 的类型检查时间从大约 45 秒减少到大约 0.4 秒。

### [](#reduced-calls-into-scanner-for-jsdoc-parsing)减少对 JSDoc 解析的扫描器调用

当旧版本的 TypeScript 解析 JSDoc 注释时，它们会使用扫描器/分词器将注释分解为细粒度标记，并将内容拼凑起来。这对于规范化注释文本可能很有帮助，这样多个空格就会折叠成一个；但它非常“啰嗦”，意味着解析器和扫描器会非常频繁地来回跳转，增加了 JSDoc 解析的开销。

TypeScript 5.1 将更多关于分解 JSDoc 注释的逻辑转移到扫描器/分词器中。扫描器现在直接将更大的内容块返回给解析器，由其根据需要进行处理。

[这些更改](https://github.com/microsoft/TypeScript/pull/53081) 将几个主要包含散文注释的 10Mb JavaScript 文件的解析时间减少了一半。对于一个更现实的例子，我们的性能套件中的 [xstate](https://github.com/statelyai/xstate) 快照将解析时间减少了约 300ms，使其加载和分析速度更快。

## [](#breaking-changes)破坏性变更

### [](#es2020-and-nodejs-1417-as-minimum-runtime-requirements)ES2020 和 Node.js 14.17 作为最低运行时要求

TypeScript 5.1 现在提供了在 ECMAScript 2020 中引入的 JavaScript 功能。因此，TypeScript 至少必须在相当现代的运行时中运行。对于大多数用户来说，这意味着 TypeScript 现在只运行在 Node.js 14.17 及更高版本上。

如果您尝试在较旧版本的 Node.js（如 Node 10 或 12）下运行 TypeScript 5.1，您可能会看到类似以下的错误，无论是运行 `tsc.js` 还是 `tsserver.js`：

`node_modules/typescript/lib/tsserver.js:2406`

`for (let i = startIndex ?? 0; i< array.length; i++) {`

`^`

`SyntaxError: Unexpected token '?'`

`at wrapSafe (internal/modules/cjs/loader.js:915:16)`

`at Module._compile (internal/modules/cjs/loader.js:963:27)`

`at Object.Module._extensions..js (internal/modules/cjs/loader.js:1027:10)`

`at Module.load (internal/modules/cjs/loader.js:863:32)`

`at Function.Module._load (internal/modules/cjs/loader.js:708:14)`

`at Function.executeUserEntryPoint [as runMain] (internal/modules/run_main.js:60:12)`

`at internal/main/run_main_module.js:17:47`

此外，如果您尝试安装 TypeScript，您将看到类似以下的 npm 错误消息：

`npm WARN EBADENGINE Unsupported engine {`

`npm WARN EBADENGINE   package: 'typescript@5.1.1-rc',`

`npm WARN EBADENGINE   required: { node: '>=14.17' },`

`npm WARN EBADENGINE   current: { node: 'v12.22.12', npm: '8.19.2' }`

`npm WARN EBADENGINE }`

来自 Yarn：

`error typescript@5.1.1-rc: The engine "node" is incompatible with this module. Expected version ">=14.17". Got "12.22.12"`

`error Found incompatible module.`

[请在此处查看有关此变更的更多信息](https://github.com/microsoft/TypeScript/pull/53291)。

### [](#explicit-typeroots-disables-upward-walks-for-node_modulestypes)显式 `typeRoots` 禁用 `node_modules/@types` 的向上遍历

以前，当在 `tsconfig.json` 中指定了 `typeRoots` 选项，但解析到任何 `typeRoots` 目录失败时，TypeScript 仍会继续向上遍历父目录，尝试解析每个父目录的 `node_modules/@types` 文件夹中的包。

这种行为可能导致过多的查找，并且已在 TypeScript 5.1 中禁用。因此，您可能会开始看到类似以下的错误，这些错误基于您的 `tsconfig.json` 中的 `types` 选项或 `///<reference >` 指令

`error TS2688: Cannot find type definition file for 'node'.`

`error TS2688: Cannot find type definition file for 'mocha'.`

`error TS2688: Cannot find type definition file for 'jasmine'.`

`error TS2688: Cannot find type definition file for 'chai-http'.`

`error TS2688: Cannot find type definition file for 'webpack-env"'.`

解决方案通常是在您的 `typeRoots` 中添加 `node_modules/@types` 的特定条目：

```jsonc
{
  "compilerOptions": {
    "types": [
      "node",
      "mocha"
    ],
    "typeRoots": [
      // 保留您之前的内容。
      "./some-custom-types/",
      // 您可能需要本地的 'node_modules/@types'。
      "./node_modules/@types",
      // 如果您使用的是 "monorepo" 布局，您可能还需要指定共享的 'node_modules/@types'
      "../../node_modules/@types"
    ]
  }
}
```

更多信息可在 [我们问题跟踪器上的原始变更](https://github.com/microsoft/TypeScript/pull/51715) 上查看。