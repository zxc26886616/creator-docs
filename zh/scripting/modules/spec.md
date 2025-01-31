# 模块规范

在 Cocos Creator 中，所有 JavaScript 代码都以模块的形式组织，这包括：

- 项目中创建的代码；

- 引擎提供的功能；

- 非项目中创建也非引擎提供，但是被项目引用到的代码。

Cocos Creator 原生支持并推荐使用 ECMAScript 模块格式（以下简称 ESM 模块格式）。为了支持对外部模块的使用，Cocos Creator 也在某种限度上支持了 CommonJS 模块格式。

## 模块格式

本节介绍了 Cocos Creator 如何决定一个模块的格式。

Cocos Creator 引擎提供的所有功能都以 ESM 模块的形式存在，见 [引擎模块](./index.md)。

项目资源目录下以 `.ts` 作为后缀的文件。例如 `assets/scripts/foo.ts`。

对于任何其它模块格式，Cocos Creator 选择与 Node.js 类似的规则来 [鉴别](https://nodejs.org/api/packages.html#packages_determining_module_system)。具体地，以下文件将被视为 ESM 格式：

- 以 `.mjs` 为后缀的文件；

- 以 `.js` 为后缀的文件，并且与其最相近的父级 `package.json` 文件中包含一个顶级的 `"type"` 字段，其值为 `"module"`。

其余的文件将被视为 CommonJS 模块格式，这包括：

- 以 `.cjs` 为后缀的文件；

- 以 `.js` 为后缀的文件，并且与其最相近的父级 `package.json` 文件中包含一个顶级的 `"type"` 字段，其值为 `"commonjs"`。

- 不在上述条件下的以 `.js` 为后缀的文件。

## 模块说明符与模块解析

在 ESM 模块中，通过标准的导入导出语句与目标模块进行交互，例如：

```ts
import { Foo } from './foo';
export { Bar } from './bar';
```

导入导出语句中关键字 `from` 后的字符串称为 **模块说明符**。模块说明符也可作为参数出现在动态导入表达式 `import()` 中。

模块说明符用于指定目标模块，从模块说明符中解析出目标模块 URL 的过程称为 **模块解析**。

Cocos Creator 支持三种模块说明符：

- **相对说明符** 像 `'./foo'`、`'../bar'` 这样以 `'./'` 和 `'../'` 开头的说明符。

- **绝对说明符** 指定了一个 URL 的说明符。例如：`foo:/bar`。

- **裸说明符**（Bare specifier） 像 `foo`、`foo/bar` 这样既不是 URL 又不是相对说明符的说明符。

### 相对说明符

相对说明符以当前模块的 URL 为基础 URL，以相对说明符为输入来解析目标模块的 URL。

例如，对于模块 `项目路径/assets/scripts/utils/foo` 来说，`'./bar'` 将解析为同目录下的 `项目路径/assets/scripts/utils/bar`；`'../baz'` 将解析为上层目录中的 `项目路径/assets/scripts/baz`。

### 绝对说明符

绝对说明符直接指定了目标模块的 URL。

Cocos Creator 目前仅支持文件协议的 URL。但由于文件 URL 中指定的文件路径是绝对路径，因此很少使用。

> 值得注意的是，在 Node.js 中，一种访问 Node.js 内置模块的方法是通过 `node:` 协议的 URL，例如：`node:fs`。Cocos Creator 会将所有对 Node.js 内置模块的访问请求解析为 `node:` URL 请求，例如 `import fs from 'fs'` 中的 `'fs'` 将解析为 `node:fs`。但 Cocos Creator 并不支持 Node.js 内置模块，也就是说并不支持 `node:` 协议。因此会产生加载错误。当使用 npm 中的模块时，可能会遇到该错误。

### 裸说明符

目前为止，对于裸说明符，Cocos Creator 将应用 Node.js 模块解析算法。

> 这就包括了对 npm 模块的解析。

一般来说，裸说明符具有以下两种形式：

- `'foo'` 解析为 npm 包 `foo` 的入口模块。

- `'foo/bar'` 将解析为 npm 包 `foo` 中子路径 `./bar` 下的模块。

裸说明符的具体解析规则可参考 [Node.js 模块解析算法](https://nodejs.org/api/esm.html#esm_resolver_algorithm_specification)。

> 在后续，Cocos Creator 可能将支持导入映射（import maps），见 [导入映射](https://github.com/WICG/import-maps)。

### 后缀与目录导入

Cocos Creator 对模块说明符中模块的后缀要求更偏向于 Web —— 必须指定后缀并且不支持 Node.js 式的目录导入。然而，基于历史原因和现行的一些限制，TypeScript 模块不允许给出后缀并支持 Node.js 式的目录导入。具体来说：

当目标模块文件的后缀是 `.js`、`.mjs` 时，模块说明符中 **必须指定** 后缀：

```ts
import './foo.mjs'; // 正确
import './foo'; // 错误：无法找到指定模块
```

Node.js 式的目录导入是不支持的：

```ts
import './foo/index.mjs'; // 正确
import './foo'; // 错误：无法找到模块。
```

> 这种后缀要求与对目录导入的限制同时应用到了相对说明符和绝对说明符。对于在裸说明符中的要求可参考 Node.js 模块解析算法。

但当目标模块文件的后缀是 `.ts` 时，模块说明符中 **不允许指定** 后缀：

```ts
import './foo'; // 正确：解析为同目录下的 `foo.ts` 模块
import './foo.ts'; // 错误：无法找到指定模块
```

另一方面，支持 Node.js 式的目录导入：

```ts
import './foo'; // 正确：解析为 `foo/index.ts` 模块
```

> Cocos Creator 支持 Web 平台。在 Web 平台上实现 Node.js 那样复杂的模块解析算法成本是昂贵的，客户端和服务端之间无法通过频繁的通讯来尝试不同的后缀和文件路径。
> 即使通过一些后处理工具可以实现在构建阶段完成这样的复杂解析，但会造成静态导入解析（通过 `import` 语句）和动态导入解析（通过 `import()` 表达式）算法的不一致。因此在模块解析算法的选择上，我们更偏向于在代码中指定完整的文件路径。
> 但我们却无法完全限制这一点，因为就目前来说，TypeScript 中不允许在说明符中指定后缀为 `.ts`。并且 TypeScript 尚且不支持自动补全特定的目标后缀。在这些限制下，我们很难做到两全其美，但我们仍在观测这些条件在未来是否有好转。

### 未支持 `browser` 字段

有些 npm 包的清单文件 `package.json` 中记录了 `browser` 字段，例如 [JSZip](https://github.com/Stuk/jszip)。`browser` 字段用于指定当该包在非 Node.js 环境下特有的模块解析方法，它可使得包中的某些专用于 Node.js 的模块被替换为能够在 Web 中使用的模块。虽然 Cocos Creator **不支持该字段**，但如果对 npm 包有编辑的能力，Cocos Creator 推荐使用 [条件化导出](https://nodejs.org/api/packages.html#packages_conditional_exports) 和 [子路径导入](https://nodejs.org/api/packages.html#packages_subpath_imports) 来代替 `browser` 字段。

否则，可以以非 npm 的方式使用目标库。例如，将目标库中专为非 Node.js 环境制定的模块复制至项目中，再通过相对路径来导入。

## CommonJS 模块解析

在 CommonJS 模块中，Cocos Creator 应用的是 [Node.js CommonJS 模块解析算法](https://nodejs.org/api/modules.html#modules_all_together)。

## 模块格式交互

Cocos Creator 允许在 ESM 模块中导入 CommonJS 模块。

当从 ESM 模块中导入 CommonJS 模块时，CommonJS 模块的 `module.exports` 对象将作为 ESM 模块的默认导出：

```ts
import { log } from 'cc';

import { default as cjs } from 'cjs';

// 上面导入语句的另一种写法：
import cjsSugar from 'cjs';

log(cjs);
log(cjs === cjsSugar);
// 打印：
//   <module.exports>
//   true

```

CommonJS 模块的 [ECMAScript 模块命名空间](https://tc39.es/ecma262/#sec-module-namespace-objects) 表示，是含有一个 `default` 导出的命名空间，其中的 `default` 导出就指向了 CommonJS 模块的 `module.exports` 的值。

该 [模块命名空间外来对象](https://tc39.es/ecma262/#module-namespace-exotic-object) 可以通过 `import * as m from 'cjs'` 来观察：

```ts
import * as m from 'cjs';
console.log(m);
// 打印：
//   [Module] { default: <module.exports> }
```

<!-- TODO：guess named exports -->

## Cocos Creator ESM 解析算法公示

Cocos Creator 用于解析 ESM 模块说明符的算法由以下的 `CREATOR_ESM_RESOLVE` 方法给出。它返回从当前 URL 解析模块说明符得到的 URL 结果。

在解析算法规范中，引用了[外部的算法](https://nodejs.org/api/esm.html#esm_resolution_algorithm)。

### 解析算法规范

`CREATOR_ESM_RESOLVE(specifier, parentURL)`
  1. Let `resolved` be the result of `ESM_RESOLVE(specifier, parentURL)`.
  2. If both `parentURL` and `resolved` are under project assets directory, then
     1. Let `extensionLessResolved` be the result of `TRY_EXTENSION_LESS_RESOLVE(resolved)`.
         1. If `extensionLessResolved` is not `undefined`, return `extensionLessResolved`.
  3. Return `resolved`.

`TRY_EXTENSION_LESS_RESOLVE(url)`
  1. If the file at `url` exists, then
     1. Return `url`.
  2. Let `baseName` be the portion after the last "/" in pathname of `url`, or whole pathname if it does not contain a "/".
  3. If `baseName` is empty, then
     1. Return `undefined`.
  4. Let `resolved` be the result URL resolution of "./" concatenated with `baseName` and `.ts`, relative to parentURL.
     1. If the file at `resolved` exists, then
     2. Return `resolved`.
  5. Let `resolved` be the result URL resolution of "./" concatenated with `baseName` and `/index.ts`, relative to parentURL.
     1. If the file at `resolved` exists, then
     2. Return `resolved`.
  6. Return `undefined`.
