---
title: Node.js 概述与环境搭建
tags: [NodeJS, 环境搭建, REPL]
course: "A Complete NODE JS Course #01-#04"
created: 2026-11-13
官方文档:
  - "[REPL](https://nodejs.org/docs/latest/api/repl.html)"
  - "[命令行选项](https://nodejs.org/docs/latest/api/cli.html)"
  - "[Globals](https://nodejs.org/docs/latest/api/globals.html)"
---

# 01 - Node.js 概述与环境搭建

> [!info] 本篇覆盖
> 课程 #01 Introduction、#02 Installing、#03 REPL、#04 First Project。
> 一句话目标：**搞清楚 Node 到底是什么、装好它、会打开 REPL、能跑第一个交互式程序。**

---

## 一、Node 到底是什么（#01）

### 1.1 官方定义的正确理解

Node.js® is a **JavaScript runtime** built on Chrome's V8 JavaScript engine.

三个词要抠清楚：

| 说法 | 对不对 | 原因 |
|:---|:---|:---|
| Node 是一门语言 | ❌ | 语言是 JavaScript，Node 只是一个**运行环境** |
| Node 是一个框架 | ❌ | 框架（如 Express）是构建在 Node **之上**的库 |
| Node 是 JS 的运行时 | ✅ | V8 负责**执行代码**，Node 额外提供**文件、网络、系统**等能力 |

### 1.2 浏览器 JS 与 Node JS 的能力差异

同一个 V8 引擎，但环境提供的东西不同：

![[img/course-01-browser-vs-node.jpg]]
*图 01-1 · 00:04:26 · 浏览器 vs Node：各自独有 API*

| 环境 | 拥有的 API | 没有 |
|:---|:---|:---|
| 浏览器 | `DOM`、`fetch`、`getElementById`、`localStorage`… | 文件系统、直接监听端口 |
| Node | `fs`、`http`、`readline`、`console.log`、`process`… | `window`、`document` |

> [!tip] 一句话记住
> **浏览器管的是"页面"，Node 管的是"机器"。** 所以前端能在 Node 里学后端，却碰不到 DOM。

### 1.3 为什么选 Node（课程给的三个卖点）

![[img/course-01-node-features.jpg]]
*图 01-2 · 00:06:52 · Node 的特性与使用它的公司*

1. **Single threaded, event driven & non-blocking** —— 单线程、事件驱动、非阻塞 I/O（[[09-Node架构与事件循环]] 详解）
2. **适合数据密集、高并发的快速可扩展应用** —— I/O 多、计算少的场景
3. **全栈同一种语言** —— Same technology across entire stack
4. Netflix、Uber、PayPal、eBay 都在生产环境使用

---

## 二、安装 Node（#02）

### 2.1 LTS 还是 Current？

![[img/course-01-nodejs-download.jpg]]
*图 01-3 · nodejs.org 下载页（课程录制时为 16.17 LTS / 18.8 Current）*

| 版本线 | 含义 | 适合谁 |
|:---|:---|:---|
| **LTS**（偶数版本，如 22、24） | 长期支持，稳定 | ==生产环境和学习者都用它== |
| **Current**（奇数版本，如 25、26） | 每年 10 月发布，尝鲜新特性 | 库作者、尝鲜者 |

> [!note] 2026 年的现状
> 课程录制时 Current 是 18.x；写作本文时 Current 已是 **v26**，LTS 为 **v24**。安装方式完全一样，去 [nodejs.org](https://nodejs.org/) 下 LTS 即可。版本号不影响本课程的知识点——**核心模块 API 十年间基本没变**。

### 2.2 验证安装

![[img/course-01-version-check.jpg]]
*图 01-4 · 00:05:35 · 终端里验证 node 与 npm 版本*

```bash
node -v      # 输出如 v24.x.x
npm -v       # npm 随 Node 一起安装
```

npm 是什么、怎么用，见 [[08-npm与包管理]]。

---

## 三、REPL：Node 的"命令行实验台"（#03）

REPL = **R**ead **E**val **P**rint **L**oop（读取-求值-打印-循环），在终端输入 `node` 直接进入，**不写文件就能执行 JS**。

```bash
$ node
Welcome to Node.js v24.x.x.
Type ".help" for more information.
> 1 + 2
3
> 'abc'.toUpperCase()
'ABC'
> const os = require('os')
undefined
> os.totalmem()
17179869184
```

常用点命令（完整列表见[官方 REPL 文档](https://nodejs.org/docs/latest/api/repl.html#commands-and-special-keys)）：

| 命令 | 作用 |
|:---|:---|
| `.help` | 列出所有点命令 |
| `.exit` | 退出（或两次 `Ctrl+C`） |
| `.break` | 跳出未写完的多行表达式 |
| `.editor` | 进入多行编辑模式 |
| `Tab` | 自动补全 / 列出对象属性 |

> [!tip] REPL 里隐藏彩蛋
> 上一条表达式的返回值存在 **`_`** 变量里：`> 3*4` → `12`，再输 `> _ * 2` → `24`。快速算数很好用。

REPL 适合**验证语法和小片段**；超过 10 行就该写成文件用 `node app.js` 跑。

---

## 四、第一个真正的项目（#04）

课程用一段"提问-回答"的交互程序，一次展示了四个核心概念：

![[img/course-01-readline.jpg]]
*图 01-5 · 00:04:45 · readline 交互程序的雏形*

```js
// app.js
const fs = require('fs');                    // ① CommonJS 导入内置模块
const readline = require('readline');        // ② 同样是内置模块

const rl = readline.createInterface({
  input: process.stdin,                      // ③ 标准输入流
  output: process.stdout,                    // ④ 标准输出流
});

rl.question("Please enter your name: ", (name) => {
  console.log(`Your entered: ${name}`);      // 模板字符串
  rl.close();
});

rl.on('close', () => {
  console.log('Interface closed');
  process.exit(0);                           // ⑤ 显式结束进程
});
```

```bash
$ node app.js
Please enter your name: Leo
Your entered: Leo
Interface closed
```

**四个概念对应的知识分支：**

| 概念 | 深入阅读 |
|:---|:---|
| `require` 与模块化 | [[06-模块化]] |
| `process.stdin/stdout` 是流 | [[07-事件驱动与Streams]] |
| `rl.on('close', …)` 事件模型 | [[07-事件驱动与Streams]] |
| 为什么回调写法是这样的 | [[02-文件系统与异步IO]] |

---

## 五、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| 装完 Node 不重开终端 | `node 不是内部或外部命令` | 重开终端或重配 PATH |
| 在 REPL 里写多行函数出错 | `...` 提示卡住 | 输入 `.break` 跳出 |
| 程序跑完不退出 | readline 接口还开着 | `rl.close()` + `process.exit(0)` |
| `window` / `document` 报错 | 在 Node 里用了浏览器 API | Node 没有这些全局对象 |

> [!summary] 自查清单
> - [x] 能用一句话说明 Node 和 JavaScript 的区别
> - [x] 知道 LTS / Current 怎么选
> - [x] 会用 REPL 快速验证代码
> - [x] 能独立跑通 readline 交互程序

---
**相关笔记**：[[09-Node架构与事件循环]] · [[08-npm与包管理]] · [[README|课程导航]]
