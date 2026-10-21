---
title: npm 包与 package.json：从作者声明到工具解析
published: 2026-10-21
description: 从包作者、工具和使用者三个视角，系统说明 package.json 的字段、入口解析、依赖、构建和发布，并完整拆解 @vue/reactivity。
tags: [npm, JavaScript, Vue, 源码阅读]
category: 前端
draft: false
lang: zh_CN
---

一份 `package.json` 里，可能同时出现包名、依赖、脚本、模块入口和构建选项。它们放在同一个文件里，却不由同一个工具处理，也不在同一个阶段生效。

读懂它，需要回答四个问题：作者声明了什么，谁读取这项声明，使用者怎样使用，最后发生什么。

本文以可安装的 JavaScript 库为主，先建立包和模块的基本概念，再说明常用字段，最后把 Vue 的 `@vue/reactivity` 3.5.44 配置完整串起来。示例中的包名仅用于说明，本地验证不需要注册或发布 npm 包。

## 一、先区分包、模块和包管理器

### 1. 包：安装和分发的单位

一个 npm 包通常是带有 `package.json` 的目录，发布后以压缩包的形式分发。它可以包含代码、类型声明、样式、命令行程序和其他资源。

```text
my-utils/
├── package.json
├── index.js
├── README.md
└── LICENSE
```

这个目录已经可以是一个包。`src`、`dist`、测试目录和构建配置按需要添加，并非固定格式。

应用项目同样可以有 `package.json`。有这个文件，不代表项目必须发布给别人安装；网站通常也用它声明依赖和开发命令。

### 2. 模块：代码加载和组织的单位

本文中的 JavaScript 模块通常对应一个代码文件。一个包可以包含多个模块，并指定其中哪些供外部使用。

JavaScript 生态中常见两种模块体系：

| 体系 | 全称与含义 | 导出写法 | 导入写法 |
| --- | --- | --- | --- |
| ESM | ECMAScript Modules，JavaScript 标准模块体系 | `export` | `import` |
| CommonJS，简称 CJS | Node.js 传统模块体系 | `module.exports` | `require()` |

ESM 文件可以这样写：

```js
/** 计算两个数的和，作为工具包的公开 API。 */
export function sum(a, b) {
  return a + b;
}
```

CommonJS 文件可以这样写：

```js
/** 计算两个数的和，作为工具包的公开 API。 */
function sum(a, b) {
  return a + b;
}

module.exports = { sum };
```

API 指调用方可以使用的函数、对象等接口。上面两个模块提供同样的 `sum` 功能，但表达和加载方式不同。

两套体系有互操作能力，不过支持范围取决于运行时版本、模块内容和工具配置。不能据此认为任意 ESM 都能被任意版本的 `require()` 加载。

ESM 中还要区分具名导出与默认导出。上面的 `export function sum` 是具名导出，使用方写 `import { sum } from "my-utils"`；如果模块写的是 `export default sum`，使用方则写 `import sum from "my-utils"`。这里的 `default` 是模块的默认导出，与后文 `exports` 配置中的 `default` 兜底条件含义不同。

入口文件也可以用 `export { sum } from "./math.js"` 转发另一个模块的 API。使用者仍从包入口导入，实现不必全部放在同一个文件里。

### 3. npm：管理包的工具

npm 既是一个包管理命令行工具的名字，也常用于指它所连接的公共包仓库。仓库也叫 registry，用于保存和分发各版本的包。

```bash
npm install my-utils
```

这条命令负责解析依赖、取得包并安装到项目中。之后代码里的：

```js
import { sum } from "my-utils";
```

则需要模块解析器把 `my-utils` 对应到具体文件。直接用 Node.js 运行时，由 Node.js 处理；通过 Vite 等工具运行或构建时，由相应工具及其解析逻辑处理。

**包是分发单位，模块是代码单位，包管理器负责安装，模块解析器负责把导入请求对应到模块。**

## 二、package.json 是谁与谁之间的约定

### 1. 作者怎样表达

`package.json` 是包根目录中的 JSON 文件。键是字段名，值是声明内容：

```json
{
  "name": "my-utils",
  "version": "1.0.0",
  "type": "module",
  "exports": "./index.js"
}
```

它必须使用合法 JSON：字符串用双引号，不能写注释，最后一项后面不能有多余逗号。本文的 JSON 片段如未注明完整文件，只展示所讨论的字段。

### 2. 工具怎样理解

每个工具按自己的规则读取字段。未知字段可能被忽略，也可能由插件或项目脚本处理。

| 读取者 | 关心的典型字段 | 处理后的结果 |
| --- | --- | --- |
| npm 等包管理器 | `name`、`version`、各类依赖、`scripts`、`files` | 安装依赖、运行命令、打包和发布 |
| Node.js | `main`、`exports`、`imports`、`type` | 选择模块入口、解释模块格式 |
| Vite、Webpack、Rollup、esbuild 及解析插件 | `exports`、`module`、`sideEffects` 等 | 解析依赖、选择构建版本、优化产物 |
| TypeScript 及编辑器类型服务 | `types`、条件导出中的 `types` 等 | 找到声明文件，检查 API 的类型 |
| CDN 服务 | `unpkg`、`jsdelivr` 等服务约定 | 为相应请求选择分发文件 |
| 项目自有脚本 | 例如 Vue 的 `buildOptions` | 按项目规则生成构建产物 |

CDN 是通过网络分发静态文件的服务。构建工具则把源码及依赖转换、组合为可运行或可分发的文件；生成的文件称为构建产物。

### 3. 使用者怎样接触它

使用者不一定直接打开这个文件，但安装包、导入代码、获得类型提示、运行命令时，都可能间接使用其中的声明。

**`package.json` 是包的描述与工具配置文件。字段的效果取决于哪个工具在什么阶段按什么规则读取它。**

## 三、身份信息：这个包叫什么，属于哪个版本

```json
{
  "name": "@example/my-utils",
  "version": "1.2.3",
  "description": "提供数字计算工具",
  "keywords": ["utils", "math"],
  "author": "Example Author",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/example/my-utils.git",
    "directory": "packages/my-utils"
  },
  "homepage": "https://github.com/example/my-utils#readme",
  "bugs": {
    "url": "https://github.com/example/my-utils/issues"
  }
}
```

| 字段 | 作者表达什么 | 工具或使用者怎样用 |
| --- | --- | --- |
| `name` | 包的名称 | 安装、依赖声明、按包名导入 |
| `version` | 当前发布版本 | 区分版本、匹配依赖范围 |
| `description`、`keywords` | 简介与关键词 | 包页面展示和搜索 |
| `author` | 作者信息 | 展示维护者信息 |
| `license` | 许可证标识 | 了解使用和分发许可，还需与许可证正文一致 |
| `repository` | 源码位置 | 找到仓库；`directory` 标明包在仓库内的位置 |
| `homepage`、`bugs` | 主页与问题反馈位置 | 阅读说明、反馈问题 |

`@example/my-utils` 中，`@example` 是 scope，即命名空间，`my-utils` 是其中的包名。scope 可以用于区分个人或组织名下的包，不表示包一定是私有的。

版本通常采用 `主版本.次版本.修订版本`：例如 `1.2.3`。按语义化版本约定，修复问题增加修订版本，兼容地增加功能增加次版本，不兼容变更增加主版本。`0.x` 仍处于初始开发阶段，兼容性需要更谨慎判断；版本号本身不能保证作者遵守了约定。

发布到 npm 时，`name` 和 `version` 是必需信息。仅在本地使用、不会发布的项目，不必为了满足发布要求填写所有元信息。

## 四、入口与模块格式：从包名走到代码

### 1. type：怎样解释 .js 文件

```json
{
  "type": "module"
}
```

`type` 解决文件格式问题：

| 文件 | Node.js 中的含义 |
| --- | --- |
| `.mjs` | 明确作为 ESM |
| `.cjs` | 明确作为 CommonJS |
| `.js`，所属包范围内 `type: "module"` | 作为 ESM |
| `.js`，所属包范围内 `type: "commonjs"` | 作为 CommonJS |

所属包范围通常由最近一层父目录的 `package.json` 确定，嵌套包有自己的范围，依赖包也不会直接继承应用的 `type`。

不写 `type` 时存在历史默认行为，较新的 Node.js 还可能对模糊的 `.js` 输入做语法检测。自己写包时明确填写，可以减少版本差异。

`type` 不会把 ESM 转换成 CommonJS，也不会生成代码。它声明的是解释方式。

### 2. main：传统的包根入口

```json
{
  "name": "my-utils",
  "type": "module",
  "main": "./index.js"
}
```

作者声明入口在 `index.js`。支持这一机制的解析器遇到包名导入时，去加载该文件：

```js
import { sum } from "my-utils";
```

`main` 不只属于 CommonJS，目标文件的模块格式还取决于扩展名和 `type`。它也不会决定文件提供哪些函数。

没有 `exports`，依然可以通过 `main` 导入包。连 `main` 也没有时，Node.js 传统 CommonJS 解析还可以回退查找根目录 `index.js` 等文件；这类兼容规则不宜当作所有工具和所有导入形式的统一规则。

### 3. exports：哪些导入请求对应哪些文件

```json
{
  "name": "my-utils",
  "type": "module",
  "main": "./index.js",
  "exports": {
    ".": "./dist/index.js",
    "./utils": "./dist/utils.js"
  }
}
```

从作者视角，左边声明公开的包子路径，右边指定目标文件。这里的目标路径以 `./` 开头，相对于包根目录。

从使用者视角：

| 导入请求 | 去掉包名后的子路径键 | 目标 |
| --- | --- | --- |
| `my-utils` | `"."` | 包内 `dist/index.js` |
| `my-utils/utils` | `"./utils"` | 包内 `dist/utils.js` |

从解析器视角，先区分包名和子路径，再在 `exports` 中匹配对应键。对于 scoped 包，`@example/my-utils` 整体才是包名，后面的 `/utils` 才是子路径。

因此，**`"."` 是 `exports` 中表示包根入口的匹配键，对应使用者只写包名的导入请求。最终文件由它对应的值决定。** 它不指调用者的当前目录，也不要求入口文件放在包根目录。

只有根入口时，还可以简写：

```json
{
  "exports": "./dist/index.js"
}
```

这等价于 `"exports": { ".": "./dist/index.js" }`。

### 4. exports 与 main 同时存在时

支持 `exports` 的解析器优先使用 `exports`。上例导入 `my-utils` 时，选择 `dist/index.js`，不会改用 `main` 的 `index.js`。

如果请求的路径没有在 `exports` 中公开，Node.js 会报 `ERR_PACKAGE_PATH_NOT_EXPORTED`，意思是“包子路径未导出”。目标文件不存在时，也不会因为有 `main` 就自动改走 `main`。

例如上面的映射没有公开 `my-utils/src/internal.js`，因此不能通过这个包子路径导入它，即使该文件确实存在。

这是正常包导入的公开边界，不能把它当作文件访问权限或安全隔离：已知文件的绝对路径仍可能直接访问。

### 5. 通配符和禁止访问

```json
{
  "exports": {
    ".": "./dist/index.js",
    "./utils/*": "./dist/utils/*.js",
    "./utils/internal/*": null
  }
}
```

`my-utils/utils/math` 中的 `math` 替换右侧的 `*`，对应 `dist/utils/math.js`。这里是字符串替换规则，不是扫描目录后自动发现入口；目标文件必须实际存在。

更具体的 `./utils/internal/*` 映射为 `null`，表示不公开这一组路径。精确子路径优先于通配规则，通配规则之间按具体程度匹配，不能把下面的条件对象顺序规则机械套到子路径映射上。

### 6. 条件导出：同一个入口按条件选文件

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  }
}
```

这里需要区分两层：`"."` 选择使用者请求的包入口；它右边的条件对象，再选择这个入口在当前环境下对应的文件。

| 条件 | 含义 |
| --- | --- |
| `import` | 通过 ESM 导入机制加载，包括静态导入和动态 `import()` |
| `require` | 通过 `require()` 加载 |
| `node` | 面向 Node.js 环境 |
| `default` | 通用兜底条件，应放在后面 |
| `types` | 供支持它的类型解析器选声明文件，通常放在相关条件对象前面 |
| `browser`、`module`、`development`、`production` | 是否启用取决于工具及其配置，不能假定所有解析器都自动启用 |

条件可能同时成立。例如 Node.js 执行 ESM 导入时，`node` 和 `import` 可以同时成立。**条件对象按键的书写顺序尝试匹配，较早成功匹配到目标的分支优先。** 嵌套条件继续在内部匹配；内部找不到匹配时，还可能继续尝试外层的后续条件。

条件叫 `import`，并不能保证右边的文件就是 ESM；名称决定什么时候选，扩展名与 `type` 决定文件格式，文件内容必须与之配合。

### 7. module、browser 和 imports 各管什么

顶层 `module` 是部分构建工具采用的 ESM 入口约定，不是 Node.js 原生入口字段：

```json
{
  "main": "./dist/index.cjs",
  "module": "./dist/index.js"
}
```

有 `exports` 时，具体工具往往先处理 `exports`；不能写一个脱离工具配置的“`exports > module > main`”万能顺序。顶层 `browser` 也属于工具约定，可以指浏览器入口或替换映射，与 `exports` 内名为 `browser` 的条件是两种机制。

`imports` 则为包内部声明以 `#` 开头的导入映射：

```json
{
  "imports": {
    "#utils": "./src/utils.js"
  }
}
```

包内代码可以写 `import ... from "#utils"`。它不自动公开 `my-utils/utils` 给外部使用，也不等于任意构建工具的路径别名功能。

## 五、类型与优化：代码之外还提供什么信息

### 1. types：让 TypeScript 知道 API 怎么调用

JavaScript 文件负责执行，类型声明文件描述参数、返回值和对象结构。例如：

```ts
/** 计算两个数的和，参数和返回值均为数字。 */
export declare function sum(a: number, b: number): number;
```

将这段内容放在 `index.d.ts`，再声明：

```json
{
  "type": "module",
  "types": "./index.d.ts",
  "exports": {
    ".": {
      "types": "./index.d.ts",
      "default": "./index.js"
    }
  }
}
```

类型工具由此知道 `sum` 接受两个数字，可以检查错误调用并提供编辑器提示；运行时仍执行 `index.js`。

TypeScript 的 `moduleResolution` 配置决定怎样寻找模块和类型。`node16`、`nodenext` 用于模拟相应 Node.js 解析规则，`bundler` 面向构建工具环境，这些模式支持处理 `exports`、`imports`。更旧的 `node10` 模式不处理条件导出，可能依赖顶层 `types` 等传统机制。因此类型入口要与使用者的解析方式相适应。

声明文件也有模块格式：`.d.mts` 明确对应 ESM，`.d.cts` 明确对应 CommonJS，`.d.ts` 受所属包的 `type` 影响。双格式包需要让声明与各分支的实际 API、模块格式一致，不能只复制同一个路径就假定两边都正确。

`types` 是类型入口声明，不会自动为实现生成类型文件，也不会修复声明与实现之间的不一致。

### 2. sideEffects：允许怎样删除未使用代码

“副作用”指模块执行时，除了提供导出，还发生需要保留的外部行为，例如注册全局功能、修改全局对象、加载样式。

```json
{
  "sideEffects": false
}
```

作者这样声明，是告诉支持该字段的打包工具：包中模块没有必须保留的顶层副作用，可以在适当条件下移除未使用模块。

这参与 Tree Shaking，即构建工具分析哪些导出或模块未被使用，并移除相关代码的优化。实际删除多少，还取决于模块结构、静态分析和构建配置。

如果样式必须在导入时保留，可以列出有副作用的文件模式：

```json
{
  "sideEffects": ["**/*.css"]
}
```

这声明 CSS 文件需要保留副作用，其他文件仍要符合无副作用的承诺。有全局注册模块时也应列入。

`sideEffects: false` 不表示导出的每个函数都是纯函数，更不表示调用 `reactive()` 不会改变状态。它描述的是模块被导入和执行时的行为，错误声明可能导致必要代码被优化掉。

## 六、依赖：开发者需要什么，使用者又需要什么

### 1. 四类依赖的边界

| 字段 | 作者声明什么 | 对安装和使用的影响 |
| --- | --- | --- |
| `dependencies` | 本包运行时仍需要的其他包 | 正常安装本包时，包管理器同时解析其运行依赖 |
| `devDependencies` | 开发、测试、构建本项目需要的工具 | 开发本项目时安装；使用者安装已发布的本包时，不会因此安装本包的开发工具 |
| `peerDependencies` | 与宿主项目共同使用的依赖及兼容范围 | 约束宿主组合，例如 Vue 插件支持的 Vue 版本；安装处理随包管理器版本和配置变化 |
| `optionalDependencies` | 可选的运行依赖 | 安装失败通常不阻止整体安装，代码必须能处理它不可用的情况 |

例如，一个 Vue 组件库可以声明：

```json
{
  "peerDependencies": {
    "vue": "^3.5.0"
  },
  "devDependencies": {
    "vue": "^3.5.0",
    "typescript": "^5.9.0"
  }
}
```

`peerDependencies` 表达与使用者项目中 Vue 的兼容关系；`devDependencies` 里的 Vue 则供库作者本地开发、测试使用。npm 7 及之后通常会尝试自动安装 peer 依赖，因此不能简单解释为“peer 永远不会自动安装”。

构建工具属于开发依赖；如果发布的 JavaScript 仍然导入某个包，该包通常需要放在运行依赖中，或有明确的 peer 关系。如果依赖代码已经完整包含在构建产物中，运行时是否还需要安装它，要看实际输出。

### 2. 版本范围与锁文件

依赖字段的值通常是版本范围：

| 写法 | 通常接受的稳定版本范围 |
| --- | --- |
| `"1.2.3"` | 仅 `1.2.3` |
| `"^1.2.3"` | `>=1.2.3 <2.0.0` |
| `"~1.2.3"` | `>=1.2.3 <1.3.0` |
| `"^0.2.3"` | `>=0.2.3 <0.3.0` |
| `"^0.0.3"` | `>=0.0.3 <0.0.4` |
| `"*"` | 允许任意稳定版本，范围很宽 |

预发布版本如 `1.3.0-beta.1` 还有专门的匹配规则，不应假定普通范围会自动接受所有预发布版。

`package.json` 声明允许范围；`package-lock.json` 或 `pnpm-lock.yaml` 等锁文件记录具体解析结果，帮助重复安装得到一致的依赖树。使用者安装你的库时，不会直接用你开发仓库的普通锁文件来替代他自己的依赖解析。

### 3. workspaces 与 workspace:*

Monorepo 指一个仓库管理多个包；workspace 指包管理器识别的一组本地包。例如：

```text
repo/
├── packages/
│   ├── shared/package.json
│   └── reactivity/package.json
└── pnpm-workspace.yaml
```

npm 可以通过根 `package.json` 的 `workspaces` 声明成员；pnpm 通常使用 `pnpm-workspace.yaml`。这些配置属于包管理器工作区机制。

```json
{
  "dependencies": {
    "@vue/shared": "workspace:*"
  }
}
```

在支持该协议的工具中，这表示使用工作区内的 `@vue/shared`，而非任意从仓库下载一个同名包。以 pnpm 为例，打包或发布时，会把 `workspace:*` 转成该本地包的实际版本：本地为 `3.5.44`，发布声明就转换成 `"3.5.44"`。

普通 npm 项目不能假定直接安装带有 `workspace:*` 的源码声明就会成功，必须使用项目要求的工具和发布流程。

### 4. overrides：项目怎样干预间接依赖

间接依赖指依赖的依赖。npm 的根项目可以用 `overrides` 调整依赖树中的版本选择：

```json
{
  "overrides": {
    "some-transitive-package": "1.2.3"
  }
}
```

这常用于修复或统一间接依赖版本，需要确认兼容性。npm 主要读取根项目的这项配置，库发布后的 `overrides` 不能替所有使用者控制依赖树。其他包管理器可能有不同的配置位置与限制。

## 七、命令、环境和命令行入口

### 1. scripts：给开发命令命名

```json
{
  "scripts": {
    "build": "tsc",
    "test": "node --test",
    "prepack": "npm run build"
  }
}
```

`npm run build` 运行 `tsc`，也就是 TypeScript 编译器命令。该命令是否能构建成功，仍取决于工具已安装、源码存在，以及 `tsconfig.json` 中的编译配置。

`npm test` 可以作为 `npm run test` 的简写。运行脚本时，本地依赖提供的命令会加入可执行路径，因此可以直接写 `tsc`，不必拼出 `node_modules` 内的完整路径。

普通脚本需显式调用，部分名称属于生命周期钩子：`prepack` 在打包前运行，`prepublishOnly` 只在发布前运行；`prepare` 可在本地安装以及打包、发布等相关流程中运行，Git 依赖场景也可能触发。不要把 `prepare` 理解成“只在发布时执行”。

脚本只是命令配置。`npm install` 不会因为有个叫 `build` 的普通脚本就自动执行它；是否执行，要看生命周期及配置中是否有对应调用。

### 2. bin：让安装后的包提供命令

```json
{
  "bin": {
    "my-tool": "./bin/cli.js"
  }
}
```

作者声明命令名 `my-tool` 对应包内的 `bin/cli.js`。包管理器在安装时建立命令入口，本地安装后可在 npm 脚本中调用，也可通过 `npm exec -- my-tool` 使用已安装的命令。

Node.js 命令行文件通常以 `#!/usr/bin/env node` 开头，告诉相关环境使用 Node.js 执行。文件必须随包发布，并满足相应平台的执行要求。

`bin` 面向终端命令，`exports` 面向代码模块导入，两者可以出现在同一个包中。

### 3. engines 与 packageManager：声明工具环境

```json
{
  "engines": {
    "node": ">=20"
  },
  "packageManager": "pnpm@9.14.4"
}
```

`engines` 声明支持的运行时版本范围。是否拒绝不符合版本的安装，还取决于包管理器及严格检查配置，不能默认认为它一定会阻止安装。

`packageManager` 声明预期的包管理器和版本，供支持它的工具使用。它本身不会在任何机器上自动安装 pnpm 或切换版本。

## 八、打包与发布：仓库里有，不等于安装包里有

### 1. files：选择分发内容

```json
{
  "files": ["dist", "bin"]
}
```

这声明要包含的发布文件范围。npm 对 `package.json`、README、许可证等有默认包含规则，对部分文件也有固定排除规则，因此最终清单应以实际打包结果为准。

`.npmignore` 也参与文件排除；没有它时，npm 在相应情况下会使用 `.gitignore`。它与 `files` 及子目录规则的组合存在细节，不宜仅凭 Git 跟踪状态猜测发布内容。

在包目录运行：

```bash
npm pack --dry-run
```

可以查看预计打包文件，不会向 npm 仓库发布。正常生命周期配置仍可能执行脚本，所以这不是保证无副作用的命令。确认入口、类型、样式等必要文件都在清单里，同时检查没有无关文件。

`files` 决定文件有没有进入包，`exports` 决定正常包导入允许怎样访问，文件内的 `export` 决定提供哪些 API。这三层缺一层，都可能导致使用失败。

### 2. private 与 publishConfig：控制发布行为

```json
{
  "private": true
}
```

`private: true` 会让 npm 拒绝发布该项目，常用于应用项目或工作区根目录。它不表示代码仓库自动变成私有，也不表示已发布的包被设成私有。

准备发布公共 scoped 包时，可以在符合仓库权限和账号条件的前提下声明：

```json
{
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

它表达发布时使用的配置，不会自动触发发布，也不能绕过 `private: true`。

`npm pack` 生成本地压缩包，`npm publish` 把包上传到指定 registry。构建、打包、发布是三个不同动作；发布仍需要有效身份、包名权限和适用的认证步骤。

## 九、一份可以在本地使用的完整小包

前面的字段不用一次全部填上。下面是一个直接分发 JavaScript 的 ESM 工具包，不需要构建工具。

目录如下：

```text
my-utils/
├── package.json
├── index.js
└── index.d.ts
```

完整的 `package.json`：

```json
{
  "name": "my-utils",
  "version": "1.0.0",
  "description": "一个数字求和工具",
  "type": "module",
  "main": "./index.js",
  "types": "./index.d.ts",
  "exports": {
    ".": {
      "types": "./index.d.ts",
      "import": "./index.js",
      "default": "./index.js"
    }
  },
  "files": ["index.js", "index.d.ts"],
  "sideEffects": false,
  "private": true
}
```

`index.js`：

```js
/** 计算两个数的和；调用者应传入数字。 */
export function sum(a, b) {
  return a + b;
}
```

`index.d.ts`：

```ts
/** 计算两个数的和，参数和返回值均为数字。 */
export declare function sum(a: number, b: number): number;
```

另建一个同级使用者目录：

```text
workspace/
├── my-utils/
└── consumer/
```

在 `consumer` 目录执行：

```bash
npm init -y
npm install ../my-utils
```

创建 `consumer/main.mjs`：

```js
import { sum } from "my-utils";

console.log(sum(1, 2)); // 3
```

运行：

```bash
node main.mjs
```

这个过程不需要 npm 账号，也不会发布。`private: true` 保护的是发布动作，本地安装仍然可用。

作者提供了入口映射、实现和声明。npm 安装本地包；Node.js 处理 `my-utils` 的根入口请求，跳过不认识的 `types` 条件，在 ESM 导入场景匹配 `import`，加载 `index.js`；最后调用文件导出的 `sum`，得到 `3`。TypeScript 使用者则可由对应声明获得数字参数与返回值的类型信息。

## 十、完整拆解 @vue/reactivity

`@vue/reactivity` 是 Vue 响应式系统的独立包，提供 `reactive`、`ref`、`effect` 等 API。下面按 3.5.44 源码中的声明分析。

### 1. 为什么要有多份构建产物

同一个功能面对不同使用方式，可能需要不同文件：

| Vue 的格式名 | 面向谁 | 使用方式 |
| --- | --- | --- |
| `esm-bundler` | 前端构建工具 | 保留 ESM 结构，由工具处理依赖和环境标记等 |
| `esm-browser` | 浏览器原生模块环境 | 通过 `<script type="module">` 加载适合浏览器的 ESM 文件 |
| `cjs` | CommonJS 使用场景 | 用 `require()` 等相应机制加载 |
| `global` | 普通浏览器脚本环境 | 用 `<script>` 加载，通过全局变量取得 API |

例如全局构建加载成功后，可以通过 `VueReactivity.reactive()` 调用；ESM 构建则通过 `import { reactive }` 取得函数。二者提供相近的功能，使用者拿到 API 的方式不同。

浏览器原生环境不会直接照搬 Node.js 的 `node_modules` 和 `package.json` 解析规则。使用包名导入时，需要构建工具、导入映射或相应服务把包名转换成可加载的地址。

`esm-bundler` 与 `esm-browser` 虽然都是 ESM，但前者可能保留等待工具处理的内容，不能仅凭扩展名就当成可直接放到浏览器运行的版本。

### 2. 顶层入口的含义

```json
{
  "name": "@vue/reactivity",
  "version": "3.5.44",
  "description": "@vue/reactivity",
  "main": "index.js",
  "module": "dist/reactivity.esm-bundler.js",
  "types": "dist/reactivity.d.ts",
  "unpkg": "dist/reactivity.global.js",
  "jsdelivr": "dist/reactivity.global.js"
}
```

`main` 提供传统入口，`module` 给支持该约定的构建工具提供 ESM 版本，`types` 提供声明入口。`unpkg` 和 `jsdelivr` 指定相应 CDN 使用的入口，这里选择全局构建；显式请求 CDN 上某个文件时，还要遵循该服务的 URL 规则。

顶层入口便于不同工具使用，并不意味着每次导入都同时加载这些文件。

### 3. 条件导出：同一句导入在不同环境中的结果

```json
{
  "exports": {
    ".": {
      "types": "./dist/reactivity.d.ts",
      "node": {
        "production": "./dist/reactivity.cjs.prod.js",
        "development": "./dist/reactivity.cjs.js",
        "default": "./index.js"
      },
      "module": "./dist/reactivity.esm-bundler.js",
      "import": "./dist/reactivity.esm-bundler.js",
      "require": "./index.js"
    },
    "./*": "./*"
  }
}
```

使用者写：

```js
import { reactive } from "@vue/reactivity";
```

解析过程可以按环境对照：

| 解析场景 | 如何匹配 | 本配置中的目标 |
| --- | --- | --- |
| 原生 Node.js 的 ESM 导入，未额外启用开发或生产条件 | 根入口 `.` → 跳过 `types` → `node` → `default` | `index.js` |
| 原生 Node.js 的 `require()`，未额外启用开发或生产条件 | 根入口 `.` → `node` → `default` | `index.js` |
| Node.js 显式启用 `production` 条件 | 根入口 `.` → `node` → `production` | `reactivity.cjs.prod.js` |
| 构建工具启用 `module`，没有启用 `node` 或更早的匹配条件 | 根入口 `.` → `module` | `reactivity.esm-bundler.js` |
| 解析器启用 `import`，没有更早成功匹配的条件 | 根入口 `.` → `import` | `reactivity.esm-bundler.js` |
| 支持条件导出的 TypeScript 类型解析 | 根入口 `.` → `types` | `reactivity.d.ts` |

关键在于 `node` 排在 `import` 之前。在 Node.js 里使用 `import`，并不意味着一定选择名为 `import` 的分支；这里 `node` 内部有兜底，已经可以完成匹配。

`index.js` 还要继续做运行时选择。源码如下：

```js
'use strict'

if (process.env.NODE_ENV === 'production') {
  module.exports = require('./dist/reactivity.cjs.prod.js')
} else {
  module.exports = require('./dist/reactivity.cjs.js')
}
```

`process.env.NODE_ENV` 是 Node.js 进程中的一个环境变量值，许多项目约定用它区分生产和开发行为。这里等于 `production` 时加载生产版，否则加载开发版。

它与条件导出属于两个阶段：

```text
没有额外的导出条件
  包名请求 → exports 的 node.default → index.js 执行
  → 检查 NODE_ENV → 选择开发或生产 CJS 文件

显式启用 production 条件
  包名请求 → exports 的 node.production → 生产 CJS 文件
```

例如 `node --conditions=production app.mjs` 给解析器增加 `production` 条件。仅设置 `NODE_ENV=production` 不会替 Node.js 启用该条件；只是本例的 `index.js` 也读取了环境变量，两条路径可以最终选到同一份生产文件。

`"./*": "./*"` 则开放包子路径，将请求直接映射到相应包内路径。它不生成文件，也不保证绕过正常入口后，所选构建适合当前环境。

### 4. 文件、依赖和构建配置

```json
{
  "files": ["index.js", "dist"],
  "sideEffects": false,
  "buildOptions": {
    "name": "VueReactivity",
    "formats": ["esm-bundler", "esm-browser", "cjs", "global"]
  },
  "dependencies": {
    "@vue/shared": "workspace:*"
  }
}
```

`files` 包含入口包装文件和构建目录；`sideEffects` 为构建工具提供移除未使用模块的依据；`@vue/shared` 是工作区内共享工具包的依赖声明。

`buildOptions` 属于 Vue 仓库自己的构建约定。仓库的 `rollup.config.js` 读取它，按 `formats` 生成相应构建；`name` 用于全局构建等需要名称的场景。npm 与 Node.js 不会因为这个字段存在就替 Vue 构建文件。

其余 `repository`、`keywords`、`author`、`license`、`bugs`、`homepage` 分别描述源码位置、检索关键词、作者、许可证、问题反馈和主页。它们帮助人和平台理解包，不参与上述运行时入口选择。

从作者视角，Vue 提供了多种文件和一套选择规则；从工具视角，各自读取适用的字段与条件；从使用者视角，可以通过包名取得当前工具链所选模块提供的响应式 API。

## 十一、遇到问题时按哪一层查

| 现象 | 先检查什么 | 为什么 |
| --- | --- | --- |
| 安装时版本或 peer 依赖冲突 | 依赖范围、宿主版本、锁文件和包管理器配置 | 尚未进入代码加载阶段 |
| 找不到包 | 包名拼写、安装位置、依赖是否安装成功 | 先确认解析器能定位包 |
| `ERR_PACKAGE_PATH_NOT_EXPORTED` | `exports` 是否公开了请求的子路径 | 文件存在不代表允许通过该包路径导入 |
| 找到包却找不到目标文件 | 入口路径、构建是否完成、实际打包清单 | 映射可能指向没有生成或没有发布的文件 |
| `require`、`module` 或 `import` 相关格式错误 | 扩展名、`type`、条件选择和实际语法 | 加载方式与文件格式可能不一致 |
| 运行正常，但 TypeScript 找不到声明 | `types`、条件声明、声明文件是否发布、`moduleResolution` | 类型解析与运行时代码解析需要分别检查 |
| 构建后样式或初始化行为消失 | `sideEffects` 声明及导入链 | 有副作用的模块可能被错误移除 |
| 修改配置却没有效果 | 当前执行的工具、读取位置、配置支持情况 | 可能改了当前工具根本不读取的字段 |

可以用下面几个定义回查全文：

- **包**是安装和分发的单位；**模块**是代码加载和组织的单位。
- **`package.json`** 是包的描述与工具配置文件，其声明由具体工具解释。
- **`type`** 声明 `.js` 的模块解释方式；**`main`** 声明传统根入口。
- **`exports`** 声明公开包子路径及其目标，可以按条件选择目标。
- **`"."`** 是包根入口的匹配键，对应只写包名的导入请求。
- **代码中的 `export` 或 `module.exports`** 提供模块的实际 API。
- **`types`** 声明类型入口；**`files`** 声明分发范围；**`scripts`** 声明可运行的命令。

## 参考资料

本文已在对应章节说明理解示例所需的概念。下面保留官方规范与源码位置，供核对版本差异和进一步查阅。

- [Node.js：包、模块格式与入口解析](https://nodejs.org/api/packages.html)
- [npm：package.json 字段](https://docs.npmjs.com/cli/v11/configuring-npm/package-json/)
- [npm：脚本与生命周期](https://docs.npmjs.com/cli/v11/using-npm/scripts/)
- [TypeScript：模块与类型解析](https://www.typescriptlang.org/docs/handbook/modules/reference.html)
- [pnpm：工作区协议与发布转换](https://pnpm.io/workspaces)
- [Vue 3.5.44：reactivity 包配置](https://github.com/vuejs/core/blob/v3.5.44/packages/reactivity/package.json)
- [Vue 3.5.44：reactivity 的 index.js](https://github.com/vuejs/core/blob/v3.5.44/packages/reactivity/index.js)
- [Vue 3.5.44：Rollup 构建配置](https://github.com/vuejs/core/blob/v3.5.44/rollup.config.js)
