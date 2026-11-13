
> [!abstract] 一句话
> 启动一个 TypeScript 项目 = ==**装 Node → 建 npm 项目 → 装 `typescript` → 生成 `tsconfig.json` → 写 `.ts` → 编译/运行 → 配好编辑器**==，一共 6 步。
> 本文按**每一步做完后项目长什么样**来讲——每个步骤都配一张**项目结构树**，让你随时知道"现在文件夹里应该有什么、哪些是新增的"；至于终端里每条命令的输入顺序与回显细节，见 [[项目使用TS的终端输入流程]]。

---

## 一、全景：6 步流水线与结构演变

```text
空文件夹
   │
   │ ① 准备环境（Node.js + 编辑器）      ← 全局安装，项目文件夹不变
   │ ② npm init -y                      ← 新增 package.json
   │ ③ npm i -D typescript              ← 新增 node_modules/ + package-lock.json
   │ ④ npx tsc --init                   ← 新增 tsconfig.json
   │ ⑤ 写 src/index.ts                  ← 新增 src/
   │ ⑥ 编译运行（tsc / tsx / 打包器）    ← 新增 dist/（取决于路线）
   ▼
一个能跑 TypeScript 的项目
```

> [!tip] 阅读方式
> 每节的结构树中，==加粗 `←` 注释的行就是这一步"新增/变化"的内容==，其余行是前几步的产物。
> 对照着看，你永远不会迷失在"我的项目现在到底处于哪个阶段"。

> [!info] 已经用脚手架了呢？
> `npm create vite@latest`、Next、Angular CLI 等脚手架**替你做完了 ②③④ 步**——装好就自带 `package.json`、`typescript`、`tsconfig.json`。
> 此时直接跳到**第 5 步写代码、第 6 步运行**；本文其余部分用来理解"脚手架到底替我做了什么"。

---

## 二、第 0 步：准备环境（装 Node.js）

TypeScript 编译器 `tsc` 本身就是一个 **npm 包**，而 npm 随 Node.js 一起安装——所以==一切从 Node 开始==。

**做什么：**

1. 去 [nodejs.org](https://nodejs.org) 下载并安装 **LTS** 版；
2. 装完**重开终端**（安装程序改的是系统 PATH，已打开的终端读不到新值）；
3. 验证：

```bash
node -v      # 例：v20.11.0
npm -v       # 例：10.2.4
```

**这一步做完，项目结构（还是空的）：**

```text
my-ts-app/                 ← 你的项目文件夹，此刻是空的
（无任何文件）
```

> [!note] 编辑器也在这一步装好
> 用 **VS Code**（或 WebStorm 等）。VS Code 对 TS 开箱即用——它**内置了 TypeScript 语言服务**，装完编辑器不需要再装任何 TS 插件就能获得类型提示、错误波浪线与自动补全。编辑器的细配置见[[#九、第 6 步：配置编辑器|第 6 步]]。

> [!warning] 两个都出版本号才算就绪
> 报 `不是内部或外部命令` / `command not found` → 没装成功，或**没重开终端**。

---

## 三、哪些文件算 TypeScript 文件（完整清单）

在动手建项目之前，先弄清"什么文件会被 TS 工具链接管"。**后缀决定身份**：

| 后缀 | 身份 | 说明 |
| --- | --- | --- |
| `.ts` | ✅ 标准 TS 源文件 | 最常见；编译后产出 `.js` |
| `.tsx` | ✅ TS + JSX | React 等用；需 `"jsx"` 编译器选项配合 |
| `.mts` | ✅ ESM 版 TS | 强制按 ES Module 解析，产 `.mjs` |
| `.cts` | ✅ CJS 版 TS | 强制按 CommonJS 解析，产 `.cjs` |
| `.d.ts` | ✅ 类型声明文件 | **只描述类型、不产 JS**；描述已有 JS 库的类型 |
| `.d.mts` / `.d.cts` | ✅ 声明文件变体 | 与 `.mts` / `.cts` 配对的声明 |
| `.js` / `.jsx` / `.mjs` / `.cjs` | ⚠️ 可选纳入 | 默认**不管**；配 `"allowJs": true` 后 tsc 也检查/编译它们 |
| `.json` | ⚠️ 可选纳入 | 配 `"resolveJsonModule": true` 后可被 `import` |

> [!tip] 一个项目里哪些"身份"可以混装？
> **源码、配置、脚本、测试都能是 TS**——`vite.config.ts`、`jest.config.ts`、`*.test.ts` 都是常规操作。
> 编译范围由 `tsconfig.json` 的 `include` / `exclude` 圈定；字段详解见 [[tsconfig.json]]。

---

## 四、第 1 步：初始化 npm 项目

**做什么：** 在项目文件夹里敲

```bash
npm init -y        # -y = 全部接受默认值
```

**这一步做完的项目结构：**

```text
my-ts-app/
└── package.json        ← 【新增】npm 项目身份卡：名字、版本、依赖清单、scripts
```

`package.json` 此刻（节选）：

```json
{
  "name": "my-ts-app",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "license": "ISC"
}
```

> [!tip] 为什么必须先有这一步
> `package.json` 是**依赖清单的载体**——下一步 `npm i -D typescript` 就是把编译器记到它的 `devDependencies` 里。没有它，安装就成了"无账可记"。
> 已有 `package.json` 的项目（clone 下来 / 脚手架生成）**直接跳过本步**。

---

## 五、第 2 步：安装 TypeScript 编译器

**做什么：**

```bash
npm i -D typescript       # 等价于 npm install --save-dev typescript
npx tsc --version         # 验证 → Version 5.x.x
```

**这一步做完的项目结构：**

```text
my-ts-app/
├── node_modules/                ← 【新增】所有依赖的实际代码（体积大，别手动改）
│   └── typescript/
│       ├── bin/tsc              ← 编译器可执行入口（npx tsc 找的就是它）
│       └── lib/                 ← 编译器主体 + 内置 lib 类型定义（lib.es2022.d.ts 等）
├── package-lock.json            ← 【新增】依赖版本锁，保证团队装到一模一样的版本
└── package.json
    └── devDependencies: { "typescript": "^5.x.x" }   ← 【新增】记账条目
```

> [!tip] 为什么用 `-D`（devDependencies）
> TS 只在**开发和构建期**干活，产物是 `.js`，==运行时不需要它== → 放 `devDependencies` 而非 `dependencies`。

> [!warning] 装本地，别装全局
> `npm i -g typescript` 把 `tsc` 装进全局 PATH，看似方便，实则==每个项目被迫用同一个版本==，容易"我电脑能编译你电脑报错"。
> 本地装 + `npx tsc`（npx 会优先找 `node_modules/.bin` 里的本地版本）才是标准姿势。

---

## 六、第 3 步：生成 tsconfig.json

**做什么：**

```bash
npx tsc --init            # 生成一份带注释的配置骨架
npx tsc --showConfig      # 打印"实际生效"的最终配置（排查利器）
```

**这一步做完的项目结构：**

```text
my-ts-app/
├── node_modules/
├── package-lock.json
├── package.json
└── tsconfig.json        ← 【新增】TS 总配置：编译哪些文件 + 按什么规则编译
```

生成的 `tsconfig.json` 是一份**满是注释的模板**，起步时真正需要关心的核心字段：

```jsonc
{
  "compilerOptions": {
    "target": "ES2020",         // 产物 JS 版本（降级到多旧）
    "module": "commonjs",       // 产物模块格式
    "strict": true,             // 严格模式全家桶（强烈建议保持 true）
    "esModuleInterop": true,    // CJS/ESM 互操作兼容
    "skipLibCheck": true,       // 跳过 node_modules 内 .d.ts 检查（提速）
    "outDir": "./dist",         // 产物输出目录          ← 建议手动补上
    "rootDir": "./src"          // 源码根目录            ← 建议手动补上
  },
  "include": ["src"]            // 只编译 src 下的文件     ← 建议手动补上
}
```

> [!warning] `--init` 不会覆盖已存在的文件
> 目录里已有 `tsconfig.json` 时它会直接跳过；想重来就先删再 `--init`。
> 另外 `tsconfig.json` 实际是 **JSONC**——可以写注释和尾逗号，这正是模板满是注释的原因。

> [!question] 装完包，为什么没自动生成 `tsconfig.json`？
> =="装包"和"生成配置"是两件事==。`npm i typescript` 只装编译器，tsconfig 是**可选的**——没有它，`tsc` 也能编译，但只能一个文件一个文件地喂，且所有选项都得靠命令行参数。要它就得**显式**敲 `npx tsc --init`。

> [!info] 字段太多记不住？
> `compilerOptions` 上百个选项按七组分类的完整拆解，见 [[tsconfig.json]]；编译器命令行选项见 [[tsc]]。

---

## 七、第 4 步：写第一个 TS 文件

**做什么：** 建源码目录，写第一段 TS：

```bash
mkdir src
```

```ts
// src/index.ts
const greet = (name: string): string => `Hello, ${name}!`;
console.log(greet("TypeScript"));
```

**这一步做完的项目结构：**

```text
my-ts-app/
├── node_modules/
├── package.json
├── package-lock.json
├── tsconfig.json
└── src/                     ← 【新增】源码根（与 tsconfig 的 rootDir/include 对应）
    └── index.ts             ← 【新增】你的第一段 TypeScript
```

> [!tip] `src` 不是强制，但值得遵守
> `tsc --init` 生成的默认模板并不要求 `src`，但==把源码圈进 `src`、把产物圈进 `dist`== 是最干净的布局：`include: ["src"]` + `outDir: "./dist"`，源码与产物永不互相污染。

---

## 八、第 5 步：编译并运行——三条路线

> [!info] 核心前提
> **TS 不能被 Node / 浏览器直接运行**（浏览器只认 JS，Node 主流版本只认 JS），必须先"过一道"。区别只在==这一道由谁来做==，于是分三条路线，各自做完后的结构树也不同。

### 8.1 路线 A：`tsc` 编译 → 再运行（自己产 `.js`）

```bash
npx tsc                    # 按 tsconfig.json 编译
node dist/index.js         # 运行产物 → 输出 Hello, TypeScript!
```

**这条路线做完的项目结构：**

```text
my-ts-app/
├── node_modules/
├── package.json
├── package-lock.json
├── tsconfig.json
├── src/
│   └── index.ts
└── dist/                        ← 【新增】编译产物目录（tsconfig 的 outDir）
    ├── index.js                 ← 【新增】src/index.ts 的 JS 化身
    └── index.d.ts               ← 【新增】（若开 declaration）类型声明，供库消费方使用
```

```text
src/index.ts ──tsc──▶ dist/index.js ──node──▶ 运行
              └── 同时做类型检查，可产出 .js.map / .d.ts
```

适合**库、Node 服务、CI 构建**——自己掌控 `.ts → .js` 全过程，产物是真实文件、可发布。

> [!tip] 改一次编一次太累？
> `npx tsc --watch` 进入监听模式：保存 `.ts` 自动重编译，结构树里 `dist/` 持续刷新，终端不用反复敲。

### 8.2 路线 B：`tsx` 直接跑（不落盘）

```bash
npm i -D tsx               # 先装执行器（一次性）
npx tsx src/index.ts       # 直接运行 .ts
```

**这条路线做完的项目结构：**

```text
my-ts-app/
├── node_modules/
│   └── tsx/                     ← 【新增】"直跑"执行器（在内存里把 TS 转成 JS）
├── package.json                 ← devDependencies 多了 tsx
├── package-lock.json
├── tsconfig.json
└── src/
    └── index.ts                 ← 运行它，但磁盘上【没有】新增任何 .js
```

适合**本地脚本、开发调试、后端小服务**——省掉"先编译再运行"两步，文件系统始终干净。
注意：`tsx` 在**内存里转译**，==默认不做完整类型检查==；类型把关另跑 `npx tsc --noEmit`（它一个文件都不产出）。
同类工具 `ts-node` 的对比见 [[ts-node]]。

### 8.3 路线 C：交给打包器，`tsc` 只查类型

```bash
npm create vite@latest     # Vite 会替你建好整个项目
npx tsc --noEmit           # 只做"类型门禁"，不产文件
```

**这条路线的典型项目结构（Vite + React + TS 模板）：**

```text
my-vite-app/
├── node_modules/
├── package.json
├── package-lock.json
├── tsconfig.json            ← 通常含 "noEmit": true——产物由打包器负责
├── tsconfig.node.json       ← 【新增】给 vite.config.ts 单独的配置（项目引用）
├── index.html               ← 【新增】前端入口页
├── public/                  ← 【新增】静态资源
└── src/
    ├── main.tsx             ← 【新增】应用入口（.tsx：TS + JSX）
    ├── App.tsx              ← 【新增】根组件
    └── vite-env.d.ts        ← 【新增】Vite 提供的环境类型声明
```

> [!note] 这条路线里 `dist/`（或 `build/`）也会出现，但它是 **打包器**（esbuild / swc / webpack）产出的 bundle，不是 `tsc` 产的——`tsc` 全程只当类型检查器，`--noEmit` 确保两套输出不打架。

适合**前端应用（Vite / Next / webpack）**。

### 8.4 三条路线对照

| 路线 | 终端输入 | 产出 `.js` | 类型检查 | 典型场景 |
| --- | --- | --- | --- | --- |
| A `tsc` | `npx tsc` + `node dist/index.js` | ✅ 落盘到 `dist/` | ✅ 完整 | 库、Node、CI |
| B `tsx` | `npx tsx src/index.ts` | ❌ 不落盘 | ⚠️ 默认不做 | 脚本、开发调试 |
| C 打包器 | 打包器 + `npx tsc --noEmit` | ✅（bundle） | ✅ 靠 `--noEmit` | 前端应用 |

> [!success] 选型口诀
> **自己发产物 → `tsc`；只想直接跑 → `tsx`；前端打包 → 打包器 + `tsc --noEmit`。**

---

## 九、第 6 步：配置编辑器

**做什么：** 用 VS Code 打开项目文件夹即可。语言服务会自动读取 `tsconfig.json`，无需额外安装 TS 插件。可按需做两件事：

1. 装项目依赖后，VS Code 提示"是否使用工作区版本的 TypeScript"——**选工作区版本**，与 `npx tsc` 用同一个编译器；
2. 团队统一行为：项目根加 `.vscode/settings.json`。

**这一步做完的项目结构（可选产物）：**

```text
my-ts-app/
├── .vscode/                     ← 【新增·可选】编辑器级配置，随仓库共享
│   └── settings.json            ←   如 "typescript.tsdk": "node_modules/typescript/lib"
├── node_modules/
├── package.json
├── tsconfig.json
└── src/
    └── index.ts
```

```jsonc
// .vscode/settings.json 示例：强制 VS Code 用项目本地的 TypeScript 版本
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

> [!warning] 编辑器报错但终端能编译（或反过来）？
> 多半是编辑器缓存了旧配置或用了不同版本的 TS。VS Code 里用命令面板执行 ==`TypeScript: Restart TS Server`== 一键重载。

---

## 十、strict 到底打开了哪些开关（完整表格）

`tsconfig.json` 里 `"strict": true` 不是单个开关，而是==一键打开下面这一组严格检查==：

| 选项 | 作用（开启后） |
| --- | --- |
| `strictBindCallApply` | `bind` / `call` / `apply` 的参数与返回值严格匹配函数签名 |
| `strictFunctionTypes` | 函数参数双向协变 → 更严格的逆变检查 |
| `strictNullChecks` | `null` / `undefined` 不再能赋给任意类型（影响最大的一项） |
| `strictPropertyInitialization` | 类的实例属性必须在构造器里初始化，或显式标注可能 undefined |
| `alwaysStrict` | 产物顶层输出 `"use strict"` |
| `noImplicitAny` | 推断不出类型的地方不再默默当作 `any`，直接报错 |
| `noImplicitThis` | `this` 类型不明时报错 |
| `useUnknownInCatchVariables` | `catch (e)` 里的 `e` 是 `unknown` 而不是 `any` |

> [!tip] 间接收紧的关联项
> `strict` 还会影响 `suppressImplicitAnyIndexErrors`（废弃）等行为；比 `strict` 更激进的 `noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`、`noUnusedLocals` 等**不在 strict 管辖内**，需单独开启——详见 [[tsconfig.json]] 的类型检查分组。

---

## 十一、常见问题与报错速查

> [!question] `'tsc' 不是内部或外部命令`
> 只本地装了却敲了裸 `tsc`。改用 `npx tsc`，或写进 `package.json` 的 `scripts` 后 `npm run build`。

> [!question] 跑了 `npx tsc` 却没有输出文件？
> 配置里是 `"noEmit": true`（只查类型）。用 `npx tsc --showConfig` 确认；要产物就去掉它。另外检查 `outDir` 是否配置。

> [!question] `dist` 里多出来一堆没想编译的文件？
> `include` 没圈住范围，或 `outDir` 落进了源码目录导致"产物再被编译"。显式设 `"include": ["src"]`、`"outDir": "./dist"`，并把 `dist` 加进 `exclude`。

> [!question] `Cannot find module 'xxx' or its corresponding type declarations`
> 缺类型声明。装 `npm i -D @types/xxx`，或自己写 `xxx.d.ts`——详见 [[TS 与 JS 的互操作性]]。

> [!question] 装完 Node，终端还是找不到命令？
> 安装程序改的是 PATH，**已打开的终端读不到新值**。关掉重开终端（或重启编辑器）再试。

---

## 相关笔记

- [[项目使用TS的终端输入流程]] —— 本文的"终端视角"：每步命令的输入顺序、回显与分支
- [[tsconfig.json]] —— 配置文件完整字段、`extends` / `references` 与场景模板
- [[tsc]] —— 编译器命令行的完整选项与常见坑
- [[ts-node]] —— 另一种"不落盘直接跑"的执行器（与 `tsx` 对比）
- [[TypeScript vs JavaScript]] —— 为什么需要编译、类型擦除是什么
- [[TS 与 JS 的互操作性]] —— 混装项目、`@types` 与渐进迁移
- [[编译器选项]] —— compilerOptions 按组速查

---

参考：TypeScript 官方手册 —— TypeScript Tooling in 5 minutes / TSConfig Reference；npm 官方文档 —— npm-init / npm-install
