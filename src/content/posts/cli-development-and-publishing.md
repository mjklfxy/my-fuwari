---
title: 用 Node.js 开发和发布一个 CLI 工具
published: 2026-10-29
description: 以待办事项工具为例，串起命令行参数解析、输入校验、本地持久化，以及 npm 包的构建、安装和发布。
tags: [Node.js, TypeScript, CLI, npm]
category: 前端
draft: false
lang: zh_CN
---

CLI 是 Command-Line Interface，也就是命令行界面。对于一个简单的 Node.js CLI，主要工作是解析用户输入的参数，根据参数执行对应的业务逻辑，再把结果输出到终端。

不过，从“能用 Node.js 运行一个脚本”到“别人安装后能直接使用一个命令”，还需要处理几件事：参数是否有效，数据保存在哪里，源码怎样构建，以及 npm 怎样把命令名对应到程序入口。

下面用一个待办事项工具，把开发和发布的过程串起来。示例使用 TypeScript 编写，构建成 JavaScript 后运行；开发环境需要 Node.js 22 或更新版本及 npm。文中的 `@metheus/todo` 是沿用原稿的示例包名，实际发布时需要换成自己有权限使用且可以发布的名称。

## 一、先确定命令和数据的约定

这个工具提供五个命令：

```bash
todo add "学习 CLI 开发"
todo list
todo finish 1
todo delete 1
todo clear
```

`add` 添加事项，`list` 列出事项，`finish` 标记完成，`delete` 删除指定事项，`clear` 清空列表。添加、完成、删除和清空后，也会显示当前列表。

用户看到的编号从 `1` 开始，数组下标从 `0` 开始，所以编号必须先经过校验，再减去 `1`。删除事项后，列表会重新编号；这里的编号只是当前列表的位置，不是固定 ID。

为了保持示例简单，待办事项保存为字符串数组，完成时在末尾追加 `✅`。这种结构足够说明 CLI 的运行过程；如果后续需要完成时间、独立状态或固定 ID，再调整数据结构。

一次调用的处理顺序是：

```text
读取命令行参数 → 校验命令和参数 → 读取已有事项
→ 执行业务操作 → 有变更时保存 → 显示当前列表
```

## 二、参数由谁传入，程序怎样读取

开发阶段先通过 Node.js 执行构建产物：

```bash
node dist/todo.js add "学习 CLI 开发"
```

Node.js 会把启动参数放到 `process.argv` 中。上面这次调用大致对应：

```text
process.argv[0]  Node.js 可执行文件的路径
process.argv[1]  todo.js 的路径
process.argv[2]  add
process.argv[3]  学习 CLI 开发
```

前两项是运行程序所需的信息，业务参数从第三项开始，因此可以这样读取：

```ts
const [command, ...args] = process.argv.slice(2);
```

终端先按照自身的语法处理引号，再把参数交给程序。`"学习 CLI 开发"` 会作为一个参数传入，引号本身不会成为事项内容。本文约定 `add` 只接收一个内容参数，含空格的内容需要用引号包起来。

参数读取之后还要验证。比如 `finish 0`、`finish -1`、`delete abc`、`delete 1abc` 和超出列表范围的编号，都应该明确报错。不能只依赖 `parseInt()`：`parseInt("1abc")` 仍然会得到 `1`，而负数传给 `splice()` 还可能从数组末尾删除事项。

这里先验证参数是不是从 `1` 开始的十进制正整数，再检查是否属于 JavaScript 的安全整数范围，以及对应事项是否存在。失败时输出原因并以退出码 `1` 结束，方便用户和调用它的其他脚本判断执行结果。

## 三、数据应该保存在哪里

CLI 每次执行都会启动一个新进程。只把事项放在内存数组里，下次运行时就没有了，所以需要把它们保存到文件。

文件位置也要考虑安装后的场景。用户可能在任意目录执行 `todo`，如果直接写入当前工作目录，不同目录下就会出现不同的列表；如果写在包的安装目录，数据又会和程序文件混在一起，受到权限或卸载操作的影响。

这个示例把数据放在当前用户的 home 目录下：

```text
<用户 home 目录>/
└── .metheus-todo/
    └── todos.json
```

代码用 `homedir()` 获取用户目录，用 `join()` 拼接路径。Windows 下通常会落在 `C:\Users\用户名\.metheus-todo\todos.json`；以实际运行账户的 home 目录为准。

保存时通过 `mkdirSync(..., { recursive: true })` 创建目录，不要求用户提前准备。读取时，文件不存在表示尚未保存事项，可以从空列表开始；JSON 损坏、文件内容不是字符串数组或没有读取权限，则应该报告错误，不能把这些失败也当成空列表。

## 四、完整实现

项目目录如下，`dist` 由构建生成：

```text
todo-cli/
├── todo.ts
├── package.json
├── tsconfig.json
└── dist/
    └── todo.js
```

下面是完整的 `todo.ts`。业务操作直接作用于数组，没有额外拆出类；参数处理和文件读写仍然各自负责清楚的事情。

```ts
#!/usr/bin/env node

import { mkdirSync, readFileSync, writeFileSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";

const TODO_DIR = join(homedir(), ".metheus-todo");
const TODO_FILE = join(TODO_DIR, "todos.json");
const USAGE = '用法：todo add "事项" | list | finish <编号> | delete <编号> | clear';

/** 读取当前用户的事项；文件不存在时返回空列表，其他读取或格式错误向上传递。 */
function loadTodos(): string[] {
  try {
    const parsed: unknown = JSON.parse(readFileSync(TODO_FILE, "utf8"));
    if (!Array.isArray(parsed) || !parsed.every((todo) => typeof todo === "string")) {
      throw new Error("todos.json 的内容必须是字符串数组");
    }
    return parsed;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") {
      return [];
    }
    throw error;
  }
}

/** 首次保存时创建用户数据目录，将完整列表写入 JSON 文件。 */
function saveTodos(todos: string[]): void {
  mkdirSync(TODO_DIR, { recursive: true });
  writeFileSync(TODO_FILE, JSON.stringify(todos, null, 2) + "\n", "utf8");
}

/** 把用户输入的编号转换为数组下标，拒绝非法编号和不存在的事项。 */
function parseTodoIndex(input: string, count: number): number {
  if (!/^[1-9]\d*$/.test(input)) {
    throw new Error("编号必须是从 1 开始的正整数");
  }
  const number = Number(input);
  if (!Number.isSafeInteger(number) || number > count) {
    throw new Error("找不到这个编号对应的事项");
  }
  return number - 1;
}

/** 校验调用方式，再执行一次操作；只有列表内容改变时才保存。 */
function main(): void {
  const [command, ...args] = process.argv.slice(2);
  switch (command) {
    case "add":
      if (args.length !== 1 || !args[0].trim()) {
        throw new Error('请提供一个非空事项，例如：todo add "学习 CLI 开发"');
      }
      break;
    case "finish":
    case "delete":
      if (args.length !== 1) {
        throw new Error(`请提供一个事项编号，例如：todo ${command} 1`);
      }
      break;
    case "list":
    case "clear":
      if (args.length !== 0) {
        throw new Error(`${command} 不接收额外参数`);
      }
      break;
    default:
      throw new Error(USAGE);
  }

  const todos = loadTodos();
  const originalState = JSON.stringify(todos);

  switch (command) {
    case "add":
      todos.push(args[0]);
      break;
    case "finish": {
      const index = parseTodoIndex(args[0], todos.length);
      if (!todos[index].endsWith("✅")) {
        todos[index] += "✅";
      }
      break;
    }
    case "delete":
      todos.splice(parseTodoIndex(args[0], todos.length), 1);
      break;
    case "clear":
      todos.length = 0;
      break;
  }

  if (JSON.stringify(todos) !== originalState) {
    saveTodos(todos);
  }

  if (todos.length === 0) {
    console.log("暂无待办事项");
  } else {
    todos.forEach((todo, index) => console.log(`${index + 1}. ${todo}`));
  }
}

try {
  main();
} catch (error) {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
}
```

这里保留了“操作前后比较内容，有变化才保存”的思路。因此 `list` 不会写文件，对已经完成的事项再次执行 `finish` 也不会重复追加标记或保存。

文件读写使用同步方法，因为这个程序每次只处理一个很小的列表，执行完就退出。这个简单文件方案面向单进程顺序使用，没有处理多个进程同时修改造成的覆盖，也没有提供中断写入后的恢复机制；不能据此认为它已经适合并发写入或重要数据存储。

## 五、从 TypeScript 源码到命令入口

### 1. 构建配置

`todo.ts` 含有类型标注。本文统一先用 TypeScript 编译器 `tsc` 生成 JavaScript，再运行产物，不依赖 Node.js 直接执行 TypeScript 的能力。

完整的 `tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmitOnError": true,
    "outDir": "dist",
    "types": ["node"]
  },
  "include": ["todo.ts"]
}
```

`outDir` 决定产物放进 `dist`；`NodeNext` 让编译器按 Node.js 的模块规则处理代码，配合下一节的 `"type": "module"` 使用 ESM，也就是 JavaScript 标准模块体系。`strict` 开启严格类型检查，`noEmitOnError` 避免检查失败时仍然生成新的产物。

### 2. package.json 声明什么

下面是 `package.json` 的起始内容。先保存它，再在项目根目录执行后面的安装命令，由 npm 写入开发依赖及其版本。

```json
{
  "name": "@metheus/todo",
  "version": "1.0.0",
  "description": "一个简单的命令行待办事项应用",
  "type": "module",
  "engines": {
    "node": ">=22"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/todo.js",
    "prepack": "npm run build"
  },
  "bin": {
    "todo": "./dist/todo.js"
  },
  "files": ["dist"],
  "publishConfig": {
    "access": "public"
  }
}
```

在项目根目录执行：

```bash
npm install --save-dev typescript @types/node@22
npm run build
npm start -- add "学习 CLI 开发"
npm start -- list
```

`typescript` 提供编译器，`@types/node` 提供 Node.js API 的类型声明，它们用于开发和构建。用户执行已经构建好的 JavaScript 不需要安装 TypeScript。`npm start --` 后面的内容会传给脚本，因而最后两条命令相当于运行 `node dist/todo.js add ...` 和 `node dist/todo.js list`。

几个字段分别在不同阶段起作用：

| 声明                             | 谁读取、何时读取              | 实际作用                                                                         |
| -------------------------------- | ----------------------------- | -------------------------------------------------------------------------------- |
| `name`、`version`、`description` | npm 管理、展示和发布包时      | 标识包及其版本，说明用途                                                         |
| `type: "module"`                 | Node.js 加载包内 `.js` 文件时 | 将这些文件按 ESM 处理，使产物中的 `import` 可以按预期运行                        |
| `engines`                        | npm 安装包时                  | 声明支持的 Node.js 版本；默认配置下版本不符通常是警告，不能当作强制拦截          |
| `bin`                            | npm 安装或链接包时            | 把命令名 `todo` 对应到包内的 `dist/todo.js`                                      |
| `files`                          | npm 打包时                    | 将 `dist` 纳入发布内容；`package.json`、README 等部分文件仍有 npm 的自动包含规则 |
| `prepack`                        | npm 正常执行打包生命周期时    | 在打包前运行构建；`npm pack` 和从本地目录执行的常规 `npm publish` 都会触发它     |
| `publishConfig.access`           | npm 发布时                    | 为这个带作用域的包指定公开访问级别，不负责授予发布权限                           |

`@metheus/todo` 是包名，`todo` 是安装后使用的命令名，它们不必相同。`bin` 的键声明命令名，值声明相对于包根目录的入口路径。纯命令行工具不需要为了提供命令而再配置 `main`；`main` 是包被作为模块加载时使用的入口字段。

### 3. shebang 和命令安装怎样配合

源码第一行：

```text
#!/usr/bin/env node
```

叫作 shebang。它必须位于文件开头，TypeScript 构建时会保留到 JavaScript 产物中。在 Unix 类系统中直接执行这个脚本时，系统通过 `/usr/bin/env` 从 `PATH` 中找到 `node`，再用它执行脚本。`PATH` 是终端查找可执行命令的目录列表。

Windows 的命令安装方式不同：npm 通常生成 `.cmd` 等启动包装脚本，再由包装脚本调用 Node.js。因此，`bin` 负责声明命令与入口的关系，shebang 说明脚本使用的解释器，npm 则根据平台建立可执行的命令入口。

安装命令不会把 TypeScript 源码自动编译成 JavaScript。`bin` 指向 `dist/todo.js`，就需要确保发布包中确实存在这个文件。

## 六、先本地检查，再发布

### 1. 检查实际打包内容

在项目根目录执行：

```bash
npm pack
```

常规执行时，npm 会先运行 `prepack` 构建，再生成 `.tgz` 压缩包，并列出打包文件。应确认其中包含 `dist/todo.js`，且产物第一行仍然是 shebang。

按本文示例包名和版本，通常会生成 `metheus-todo-1.0.0.tgz`。以命令实际输出的文件名为准。`files` 控制的是发布包内容，不会删除源码目录，也不会决定用户数据保存位置。

### 2. 安装打包产物，验证命令

以下命令用于作者在发布前手动验证，需留意本机是否已有同名 `todo` 命令：

```bash
npm install --global ./metheus-todo-1.0.0.tgz
todo add "学习 CLI 开发"
todo list
todo finish 1
todo delete 1
```

首次使用且没有历史数据时，预期先显示 `1. 学习 CLI 开发`，完成后显示 `1. 学习 CLI 开发✅`，删除后显示“暂无待办事项”。这些命令会实际修改当前用户的待办文件，不要在需要保留的已有列表上随意做清空或删除测试。

安装打包产物可以检查发布包是否真的包含入口，以及安装后的命令能否启动。如果终端提示找不到命令，还需要检查 npm 的全局命令目录是否已加入 `PATH`。

发布前也要检查失败情况：缺少事项内容、编号为 `0` 或负数、编号夹杂字母、编号超出范围、未知命令、JSON 文件损坏。有效输入执行对应操作，无效输入报告原因并返回非零退出码，不能悄悄操作别的事项。

### 3. 发布到 npm

本地检查完成、包名和内容确认后，才进入发布步骤：

```bash
npm login
npm publish
```

这里按默认公共 npm registry 说明。发布需要拥有对应账号和作用域的权限，并满足账号及 registry 的认证要求；公开的作用域包由 `publishConfig.access: "public"` 声明访问级别。已经发布的同名同版本不能直接覆盖，后续发布需要更新版本号。

发布成功后，其他用户才能从 registry 安装：

```bash
npm install --global @metheus/todo
todo list
```

这时安装的是已发布包中的 JavaScript 产物。命令运行后，事项仍然保存到使用者自己的 home 目录中，而不是作者电脑中的文件，也不是 npm registry。

## 参考资料

- [Node.js：process.argv](https://nodejs.org/docs/latest-v22.x/api/process.html#processargv)
- [Node.js：os.homedir()](https://nodejs.org/docs/latest-v22.x/api/os.html#oshomedir)
- [npm：package.json 字段](https://docs.npmjs.com/cli/v11/configuring-npm/package-json)
- [npm：脚本生命周期](https://docs.npmjs.com/cli/v11/using-npm/scripts)
- [npm：npm pack](https://docs.npmjs.com/cli/v11/commands/npm-pack)
- [npm：npm publish](https://docs.npmjs.com/cli/v11/commands/npm-publish)
