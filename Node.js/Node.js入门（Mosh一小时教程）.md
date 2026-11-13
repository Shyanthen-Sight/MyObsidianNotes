---
title: Node.js 入门（Mosh 一小时教程）
aliases:
  - Node.js 入门
  - Learn Node in 1 Hour
tags:
  - Node.js
  - 后端
  - JavaScript
  - 视频笔记
source: https://youtu.be/TlB_eWDSMt4
created: 2026-11-12
---

# Node.js 入门（Programming with Mosh《Learn Node in 1 Hour》）

> [!abstract] 一句话
> 这门课用 1 小时把 Node 讲透：==Node 是什么 → 为什么快（非阻塞）→ 怎么装 → 模块系统怎么运作 → 五个核心内置模块（path / os / fs / events / http）==。
> 全文按视频 18 章的时间顺序整理，**30 张视频截图**已插入对应位置，代码全部来自现场演示。

**视频** · [Node.js Tutorial for Beginners](https://youtu.be/TlB_eWDSMt4) ｜ **讲师** · Mosh Hamedani ｜ **时长** · 1h10m ｜ **前置** · 会一点 JavaScript

> [!info]- 章节目录（18 章 · 点击展开）
> | # | 章节 | 时间 | # | 章节 | 时间 |
> | :-- | :--- | :--- | :-- | :--- | :--- |
> | 1 | What is Node | `00:00` | 10 | Loading a Module | `27:35` |
> | 2 | Node Architecture | `03:01` | 11 | Module Wrapper Function | `32:59` |
> | 3 | How Node Works | `06:04` | 12 | Path Module | `39:53` |
> | 4 | Installing Node | `10:29` | 13 | OS Module | `44:03` |
> | 5 | Your First Node Program | `13:01` | 14 | File System Module | `48:22` |
> | 6 | Node Module System | `15:22` | 15 | Events Module | `53:14` |
> | 7 | Global Object | `15:52` | 16 | Event Arguments | `59:33` |
> | 8 | Modules | `19:14` | 17 | Extending EventEmitter | `01:02:43` |
> | 9 | Creating a Module | `22:51` | 18 | HTTP Module | `01:10:46` |

---

## 一、Node 是什么（00:00）

Node（Node.js）是==**开源、跨平台的运行时环境，让 JavaScript 能在浏览器之外执行**==。它最常见的用途是构建**后端服务**（API），为浏览器里的 Web 应用、手机上的 App 提供能力：存数据、发邮件、推通知、启动工作流。

![560](Node-01-runtime-environment.jpg)
*图 01 · 00:04 · Node 是 "a runtime environment for executing JavaScript code"*

![560](Node-01-clients-backend.jpg)
*图 02 · 00:40 · Web App / Mobile App 都是"壳"，背后都要连一个 Back-end Service*

### 为什么选 Node

| #   | 理由           | 说明                          |
| :-- | :----------- | :-------------------------- |
| 1   | **上手快**      | 适合原型设计与敏捷开发，同时也能扛住超高并发      |
| 2   | **大厂已用于生产**  | PayPal、Uber、Netflix、Walmart |
| 3   | **复用 JS 技能** | 不必学新语言就能转全栈；前后端同语言，规范/工具统一  |
| 4   | **生态最大**     | npm 上几乎任何功能都有现成库，可专注业务核心    |

> [!quote] PayPal 的迁移数据（视频引用）
> 用 Node 重写一个 Java + Spring 应用后：==开发速度翻倍、人更少、代码行数减少 33%、文件数减少 40%、每秒请求数翻倍、平均响应时间降低 35%==。

> [!warning] 两个常见的错误类比
> - **Node 不是编程语言**——把它和 C#、Ruby 比，就像把汽车和苹果比。
> - **Node 不是框架**——它不同于 .NET、Rails、Django。那些是"用来建 Web 应用的框架"，Node 是"执行 JavaScript 的运行时环境"。

---

## 二、Node 的架构（03:01）

要理解 Node，先理解什么叫"运行时环境"。

- **浏览器时代**：JavaScript 只能在浏览器里跑。每个浏览器内置一个 **JavaScript 引擎**，把 JS 翻译成机器能懂的代码——Edge 用 Chakra、Firefox 用 SpiderMonkey、Chrome 用 ==V8==。引擎各不相同，所以同一段代码在不同浏览器里行为可能有差异。
- **浏览器的运行时环境**还额外提供了 `window`、`document`：让代码能与环境交互（操作 DOM、发请求、用 localStorage）。
- **2009 年的那一步**：Ryan Dahl 想到"为什么不能在浏览器之外跑 JS"，于是把市面上最快的引擎——Google 的 **V8**——嵌进一个 C++ 程序，这个程序就是 Node。

```text
浏览器                          Node
┌───────────────────────┐      ┌───────────────────────┐
│  V8 引擎（执行 JS）    │      │  V8 引擎（执行 JS）    │  ← 同一个引擎
├───────────────────────┤      ├───────────────────────┤
│ window / document     │      │ fs / http / os / path │  ← 不同的环境对象
│ DOM、BOM、localStorage │      │ 文件、网络、进程        │
└───────────────────────┘      └───────────────────────┘
        在浏览器里跑                    在服务器上跑
```

![560](Node-02-chrome-vs-node-v8.jpg)
*图 03 · 05:25 · 左：Chrome（浏览器运行时）；右：Node.exe。共用同一个 v8 引擎，只是外围环境不同*

> [!tip] 一句话概括
> **Node = V8 引擎 + 一组额外模块**。Chrome 和 Node 共用同一个引擎，但提供给 JS 的**运行时环境完全不同**——所以 Node 里 `console.log(window)` 会直接报 `window is not defined`。

---

## 三、Node 如何工作：非阻塞 vs 阻塞（06:04）

### 3.1 餐厅比喻

> **非阻塞（异步）**：服务员把你的订单交给厨房，**立刻去服务下一桌**；厨师做菜时他不干等。一个人能服务很多桌。
> **阻塞（同步）**：服务员把订单交给厨房后，**坐在厨房门口一直等**，直到菜做好才去接下一桌。

**服务员 = 线程；桌 = 请求。**

![560](Node-03-blocking-synchronous.jpg)
*图 04 · 07:00 · Blocking = Synchronous：同步即阻塞*

### 3.2 阻塞架构的问题（.NET / Rails 默认）

```text
请求到达 → 分配一个线程 → 线程去查数据库 → 【线程原地等待】
                                            ↓
                              等待期间无法服务其他客户端
                                            ↓
                              并发一高 → 线程池耗尽 → 新客户端排队
                                            ↓
                              要么让用户等，要么加硬件（浪费资源）
```

![560](Node-03-threads-wait.jpg)
*图 05 · 08:15 · 每个请求各占一条线程；请求 3 已无线程可用，只能 WAIT*

### 3.3 Node 的做法：单线程 + 事件队列

```text
请求到达 → 唯一的主线程处理 → 需要查数据库 → 【不等】立刻去服务下一个客户端
                                        ↓
                        数据库好了 → 把"事件"放进事件队列
                                        ↓
                        Node 后台持续监控队列 → 取出事件 → 执行对应的回调
```

![560](Node-03-single-thread.jpg)
*图 06 · 09:05 · 只有一条线程，就能同时服务多个请求*

> [!success] 结论
> Node 应用**默认就是异步的**，不需要你额外做什么（.NET 也能异步，但得自己写）。
> 因此 Node 特别适合 ==**I/O 密集**==（大量磁盘/网络访问）的应用——同样的硬件能服务更多客户端，这就是"高可扩展"的来源。

![480](Node-03-io-intensive.jpg)
*图 07 · 09:30 · Node is ideal for I/O-intensive apps*

![480](Node-03-cpu-intensive.jpg)
*图 08 · 09:55 · 反面结论：Do not use Node for CPU-intensive apps*

> [!danger] 不要用 Node 做 CPU 密集型应用
> 视频编码、图像处理这类任务几乎全是 CPU 计算、极少碰 I/O。而 Node 是**单线程**——一个客户端占着 CPU 算，==其他所有客户端只能等==。
> 一句话：**Node 适合 I/O 密集和实时应用，不适合 CPU 密集应用。**

---

## 四、安装 Node（10:29）

```bash
node --version      # 检查是否已安装；视频里是 v6.11.0，最新稳定版当时是 v8.9.1
```

去 [nodejs.org](https://nodejs.org) 下载，主页给两个版本：

| 版本 | 说明 |
| :--- | :--- |
| **LTS**（推荐大多数用户） | 最新稳定版，生产首选 |
| **Current** | 更新的实验版本，可能不稳定 |

装最新稳定版（一路 next / 输入密码），装完**重开终端**再验一次版本。

![520](Node-04-nodejs-download.jpg)
*图 09 · 11:09 · 当年的下载页：8.9.1 LTS 与 9.2.0 Current 两个版本*

> [!tip] 版本会过时，基础不会
> Mosh 特意说明：不用担心课程随版本过时，因为本课只用**核心模块**（`path` / `os` / `fs` / `events` / `http`），这些 API 稳定存在很多年，代码在未来版本里照样能跑。先吃透基础，新特性以后看 changelog 就行。

> [!note] 2026 年的补充
> Node 已到 v22 / v24 一线，LTS 才是生产选择。Windows 可用 `winget install OpenJS.NodeJS.LTS` 安装；多版本切换用 **nvm / fnm / volta**（`nvm install --lts`）。

---

## 五、第一个 Node 程序（13:01）

```bash
mkdir first-app
cd first-app
code .                        # 用 VS Code 打开（也可用任何编辑器）
```

新建 `app.js`：

```js
// app.js
function sayHello(name) {
  console.log('Hello ' + name);
}

sayHello('Mosh');
```

```bash
node app.js
# → Hello Mosh
```

![520](Node-05-hello-mosh.jpg)
*图 10 · 14:35 · 终端里跑出第一个 Node 程序的输出*

**背后发生了什么**：`node` 是一个 C++ 程序，内含 V8。它把 `app.js` 交给 V8 执行，仅此而已——**浏览器里的 JS 怎么写，这里基本就怎么写**。

### 验证"没有浏览器环境"

```js
// app.js
console.log(window);
```

```bash
node app.js
# → ReferenceError: window is not defined
```

![560](Node-05-window-not-defined.jpg)
*图 11 · 15:05 · Node 里访问 `window` 直接报 ReferenceError*

```text
first-app/
└── app.js          ← 主模块（入口文件）
```

> [!info] 为什么没有 window / document
> 这两个对象是**浏览器**提供的运行时环境的一部分。Node 提供的是另一套对象：文件系统、操作系统、网络——也就是接下来的五个核心模块。

---

## 六、全局对象：`global` vs `window`（15:52）

`console.log` 里的 `console` 是**全局对象**——全局作用域的一部分，任何文件任何位置都能直接用。

**标准 JS 自带的全局函数**（浏览器和 Node 都有）：

| 函数 | 用途 |
| :--- | :--- |
| `setTimeout(fn, ms)` | 延迟执行一次 |
| `clearTimeout(id)` | 取消上面的定时器 |
| `setInterval(fn, ms)` | 每隔一段时间重复执行 |
| `clearInterval(id)` | 停止重复执行 |

**区别在"全局作用域"由谁代表：**

```js
// 浏览器：全局作用域 = window 对象
console.log === window.console        // true
setTimeout === window.setTimeout      // true

var message = 'hi';
window.message                        // 'hi'  ← 顶层变量挂到了 window 上
```

```js
// Node：全局作用域 = global 对象
global.console.log('hi');             // 可以，等价于 console.log('hi')
global.setTimeout(...);               // 可以

const message = 'hi';
console.log(global.message);          // → undefined  ❗
```

> [!tip] 关键差异（也是模块系统的伏笔）
> 浏览器里顶层声明的变量/函数会挂到 `window` 上；**Node 里不会挂到 `global` 上**——它们只在这个文件内部有效。
> 这种"不污染全局"的行为，正是下一节模块系统要做的事。

---

## 七、模块（19:14）：为什么需要模块

**浏览器全局作用域的坑**：真实应用会把代码拆成多个文件。如果两个文件都定义了 `sayHello`，因为都挂在全局，==后加载的会覆盖先加载的==。

```text
file1.js:  function sayHello() {}
file2.js:  function sayHello() {}     ← 同名 → 互相覆盖，Bug 极难查
```

**解法：模块化**——把变量和函数封装在小容器（模块）里，容器内的定义**默认私有**，要用就显式导出（OOP 里叫 private / public）。

**Node 的模块规则：**

1. ==**每个文件就是一个模块**==；
2. 文件里定义的变量、函数**作用域限于该文件**，外部不可见；
3. 一个 Node 应用至少有一个文件，称为**主模块**（这里是 `app.js`）；
4. 要对外暴露，必须**显式导出**。

![520](Node-08-module-diagram.jpg)
*图 12 · 21:00 · 每个文件（如 app.js）都是一个 MODULE，内部变量/函数默认私有*

### 看一眼 `module` 对象

```js
console.log(module);
```

```text
Module {
  id: '.',                                  // 模块唯一标识（主模块是 '.'）
  path: 'D:\\...\\first-app',
  exports: {},                              // ← 导出的东西都放这里
  parent: null,                             // 谁 require 了我（主模块为 null）
  filename: 'D:\\...\\first-app\\app.js',   // 完整路径
  loaded: false,                            // 是否已加载完成
  children: [],                             // 我 require 了谁
  paths: [ 'D:\\...\\node_modules', ... ]   // 模块查找路径
}
```

![560](Node-08-module-object.jpg)
*图 13 · 25:10 · 终端里打印出的 module 对象：id / exports / filename / loaded / children / paths*

> [!warning] `module` 看起来像全局变量，但不是
> 它**不是** `global` 的属性——它是"模块包装函数"的一个参数（见[[#十、模块包装函数（32:59）：Node 的封装魔法|第十节]]）。这也是为什么每个文件的 `module` 互不相同。

---

## 八、创建模块（22:51）：`module.exports`

场景：做一个可复用的日志模块 `logger.js`，内部往远程日志服务发 HTTP 请求。

```js
// logger.js
var url = 'http://mylogger.io/log';         // 实现细节，不导出

function log(message) {
  // 真实实现：向 url 发 HTTP 请求
  console.log(message);
}

module.exports.log = log;                   // 导出：外部可用的只有 log
// module.exports.url = url;                // 想导出也行，但没必要
```

![560](Node-09-logger-exports.jpg)
*图 14 · 26:10 · `module.exports.log = log` 把函数导出，url 留作内部实现*

> [!tip] DVD 播放器的比喻
> 播放器外壳上只有几个按钮——那是它的**公共接口**；盒子内部的电路是**实现细节**，不同型号差异巨大，用户不需要知道。
> 所以 `url` 是内部电路，**不导出**；`log` 是面板按钮，**导出**。一个模块往往有多个成员，但只暴露必要的那几个。

> [!note] 导出时可以改名
> `module.exports.endpoint = url;` —— 内部叫 `url`，外部看到的是 `endpoint`。名字可以按"使用者视角"重新取。

---

## 九、加载模块（27:35）：`require`

```js
// app.js
const logger = require('./logger');

console.log(logger);        // → { log: [Function] }
logger.log('message');      // → message
```

![560](Node-10-require-logger.jpg)
*图 15 · 30:10 · 左：require('./logger') 加载模块；右：logger.log('message') 调用*

### `require` 的解析规则

| 写法 | Node 的理解 |
| :--- | :--- |
| `require('path')` | 不带 `./` `../` → 先找**内置模块**，再找 `node_modules` 里的包 |
| `require('./logger')` | 带 `./` → 当前目录下的文件，**自动补 `.js`** |
| `require('./lib/logger')` | 子目录 |
| `require('../logger')` | 父目录 |

`require` **返回目标模块的 `module.exports` 对象**——所以拿到什么，完全取决于对方导出了什么。

> [!tip] 用 `const` 而不是 `var`
> ```js
> var logger  = require('./logger');
> logger = 1;            // 不报错 → 后面 logger.log() 才炸：logger.log is not a function
>
> const logger = require('./logger');
> logger = 1;            // 立刻报：Assignment to constant variable
> ```
> 用 `const` 让错误**提前暴露**。视频还演示了 **JSHint**：`jshint app.js` 能静态扫出 `Attempting to overwrite logger which is a constant`——在运行前就拦住这类手误。

### 也可以只导出一个函数

```js
// logger.js —— 只有一个功能时，不必包成对象
module.exports = log;
```

```js
// app.js
const log = require('./logger');   // 现在它就是一个函数
log('message');
```

> [!success] 两种导出风格
> - **导出对象**：`module.exports = { log, other }`——有多个成员时用；
> - **导出单个函数/类**：`module.exports = log`——只有一个能力时更简洁。

---

## 十、模块包装函数（32:59）：Node 的封装魔法

在 `logger.js` **第一行**故意写错语法：

```js
var x = ;              // 语法错误，放第一行！
```

```bash
node app.js
```

报错信息里除了 `SyntaxError: Unexpected token ';'`，还露出了一段**你没写过的函数头**：

```js
(function (exports, require, module, __filename, __dirname) {
  // ← 你的代码被塞进这里
});
```

![620](Node-11-wrapper-function.jpg)
*图 16 · 35:30 · 故意制造语法错误，让 Node 的报错暴露出"模块包装函数"*

> [!info] 真相
> **Node 从不直接执行你的文件**，而是把每个模块的代码包进这个"**模块包装函数**"（module wrapper function）里再执行——本质是一个 **IIFE**（立即调用函数表达式）。
> 这就是"每个文件的变量都是私有的"的实现机制：你的代码只是函数体，变量自然成了函数内的局部变量。

**五个参数**（那个"看起来是全局、其实不是"的五件套）：

| 参数 | 含义 |
| :--- | :--- |
| `exports` | `module.exports` 的**快捷引用** |
| `require` | 加载模块的函数（每个模块私有，不是全局的） |
| `module` | 当前模块对象 |
| `__filename` | 当前文件的**完整路径** |
| `__dirname` | 当前文件**所在目录**的路径 |

```js
console.log(__filename);   // D:\...\first-app\logger.js
console.log(__dirname);    // D:\...\first-app
```

> [!danger] `exports = log` 为什么不行
> `exports` 只是 `module.exports` 的**引用**，重新赋值只是把局部变量指向别处，`module.exports` 本身没变。
> - ✅ `exports.log = log;` 等价于 `module.exports.log = log;`
> - ✅ `module.exports = log;` 换掉整个导出对象
> - ❌ `exports = log;` 什么都不导出（外部拿到空对象 `{}`）

---

## 十一、Path 模块（39:53）

```js
// app.js
const path = require('path');

const pathObj = path.parse(__filename);
console.log(pathObj);
```

```text
{
  root: 'D:\\',
  dir: 'D:\\projects\\first-app',   // 所在目录
  base: 'app.js',                   // 文件名（含扩展名）
  ext: '.js',                       // 扩展名
  name: 'app'                       // 文件名（不含扩展名）
}
```

![600](Node-12-path-parse.jpg)
*图 17 · 43:10 · 编辑器里敲 path.parse(__filename)，悬浮提示注明返回 ParsedPath 对象*

> [!tip] 为什么用 `path` 而不是字符串拼接
> 拼接 `dir + '/' + file` 在 Windows（`\`）和 Linux/macOS（`/`）上不一样，还要处理多余分隔符、`..` 归一化。
> `path` 模块（`join` / `resolve` / `parse` / `basename` / `extname`）把这些都处理好了，==跨平台安全==。

---

## 十二、OS 模块（44:03）

```js
// app.js
const os = require('os');

const totalMemory = os.totalmem();
const freeMemory = os.freemem();

console.log(`Total memory: ${totalMemory}`);
console.log(`Free memory: ${freeMemory}`);
```

![600](Node-13-os-code.jpg)
*图 18 · 46:10 · 取总内存 / 空闲内存，并用模板字符串拼接输出*

![520](Node-13-os-output.jpg)
*图 19 · 48:10 · 终端输出 Total Memory / Free Memory*

其他可用方法：`os.userInfo()`（当前用户）、`os.uptime()`（开机时长）、`os.cpus()`、`os.platform()`……

> [!tip] 模板字符串（ES6 / ES2015）
> 用**反引号**（键盘数字 1 左边那个键）包裹，`${变量}` 插值，不用 `+` 拼接。
> ECMA 每年给 JS 加新特性，浏览器实现进度不一，但 **Node 用的 V8 很新**，所以 ES6+ 特性在 Node 里基本都能直接用。

> [!info] 这件事在"纯浏览器时代"做不到
> 以前 JS 只能在浏览器里跑，只有 `window` / `document`，拿不到操作系统信息、读不了文件。
> 现在代码跑在浏览器之外（服务器上），==文件系统、网络、进程信息全部对 JS 开放==。

---

## 十三、File System 模块（48:22）

`fs` 里**几乎每个操作都有两种形态**：同步（`xxxSync`，阻塞）与异步（`xxx`，非阻塞）。

```js
const fs = require('fs');

// ① 同步（仅用于理解，不要在生产用）
const files = fs.readdirSync('./');
console.log(files);            // [ 'app.js', 'logger.js' ]

// ② 异步（推荐）：最后一个参数是回调，err-first 风格
fs.readdir('./', function (err, files) {
  if (err) console.log('Error', err);
  else console.log('Result', files);
});
```

![620](Node-14-fs-async.jpg)
*图 20 · 53:10 · fs.readdir 的异步写法：err-first 回调，err 和 files 二选一*

> [!danger] 生产环境一律用异步方法
> Node 是**单线程**。后端可能同时有几百上千个客户端连着，一旦你用同步方法把这条唯一的主线程卡住（读大文件、查磁盘），==所有客户端都得排队==。
> 同步方法（`readdirSync`、`readFileSync`……）**只为脚本和简单场景存在**。

### err-first 回调约定

回调的第一个参数永远是 `error`，第二个才是结果；==两者只有一个有值，另一个是 `null`==。

```js
fs.readdir('./nonexist', function (err, files) {
  if (err) console.log('Error', err);   // Error: ENOENT: no such file or directory
  else console.log('Result', files);
});
```

> [!note] 本节演示了"错误处理"的雏形
> `console.log(err)` 只是演示，不是正经的错误处理。生产里该记日志、该抛出、该返回 500——Mosh 说明完整错误处理在后面单独的章节讲。

---

## 十四、Events 模块（53:14）：EventEmitter

**事件 = 应用里"某件事发生了"的信号。** Node 的核心功能大量建立在事件之上——比如 HTTP 服务器：每收到一个请求，`http.Server` 就**触发一个事件**，你的任务就是响应它。

```js
const EventEmitter = require('events');   // 拿到的是【类】
const emitter = new EventEmitter();       // 创建【实例】

// 注册监听器
emitter.on('messageLogged', function () {
  console.log('Listener called');
});

// 触发事件
emitter.emit('messageLogged');
```

```bash
node app.js
# → Listener called
```

![600](Node-15-events-on-emit.jpg)
*图 21 · 58:50 · 完整流程：require('events') → new EventEmitter → on 注册 → emit 触发*

> [!tip] 类 vs 对象（人 vs 具体的人）
> `EventEmitter` 是**类**——一张蓝图，定义"一个事件发射器能做什么"；`emitter` 是**对象/实例**——蓝图造出来的具体东西。
> 就像 `Person` 是类，而 John、Mary、Bob 是对象。
> 命名约定：==类用 PascalCase（`EventEmitter`），实例用 camelCase（`emitter`）==。

> [!warning] 顺序很关键
> 如果在 `emit()` **之后**才 `on()`，监听器永远不会被调用——`emit` 会**同步地**遍历当下已注册的监听器并立即调用。

**两个核心方法**（其余十来个很少用）：

| 方法 | 作用 |
| :--- | :--- |
| `emitter.on(name, listener)` | 注册监听器（`addListener` 的别名） |
| `emitter.emit(name, ...args)` | 触发事件（emit ≈ 发出声音、产生信号） |

> [!question] 只 emit 不 on，会发生什么？
> 什么都不会发生，也不报错。==事件是"广播"，没人订阅就等于白喊==。

---

## 十五、事件参数（59:33）

触发事件时，可以附带数据（比如刚记录的那条日志的 id、访问 URL）：

```js
// 反例：一堆魔法值，看不懂谁是谁
emitter.emit('messageLogged', 1, 'http://');

// 正解：封装成一个对象（event argument）
emitter.emit('messageLogged', { id: 1, url: 'http://' });
```

监听器接收这个参数：

```js
emitter.on('messageLogged', (arg) => {
  console.log('Listener called', arg);       // { id: 1, url: 'http://' }
});
```

![600](Node-16-event-args.jpg)
*图 22 · 61:00 · emit 时附带一个对象参数，监听器用 (arg) 接收*

> [!tip] 箭头函数（ES6）
> `function () { ... }` → `() => { ... }`，去掉了 `function` 关键字，用 `=>` 分隔参数和函数体。语法更短，社区更偏好。
> 参数名随意：`arg`、`e`、`eventArgs` 都行，团队统一即可。

---

## 十六、继承 EventEmitter（01:02:43）

### 16.1 先看清陷阱：两个 emitter 互不相干

把 `emit` 写进 `logger.js`，把 `on` 留在 `app.js`：

```js
// logger.js
const EventEmitter = require('events');
const emitter = new EventEmitter();

function log(message) {
  console.log(message);
  emitter.emit('messageLogged', { id: 1, url: 'http://' });
}

module.exports = log;
```

```js
// app.js
const log = require('./logger');

log('message');                                // 打印了 message
// → 但监听器没被调用！
```

> [!danger] 原因
> `logger.js` 里有一个 `emitter`，`app.js` 里注册监听器的是**另一个** `emitter` 对象。
> 类相同 ≠ 实例相同。==事件是"每个实例各自一套"==——在 A 对象上 emit，B 对象的监听器当然收不到。
> 这就是为什么**真实项目里几乎不直接 `new EventEmitter()`**。

### 16.2 正解：让自定义类继承 EventEmitter

```js
// logger.js
const EventEmitter = require('events');

class Logger extends EventEmitter {
  log(message) {                    // 类里的函数叫"方法"，不需要 function 关键字
    console.log(message);
    this.emit('messageLogged', { id: 1, url: 'http://' });   // this = 当前实例
  }
}

module.exports = Logger;            // 导出的是【类】
```

![620](Node-17-logger-class.jpg)
*图 23 · 67:10 · class Logger extends EventEmitter，方法内部 this.emit 触发事件*

```js
// app.js
const Logger = require('./logger');   // 拿到类，首字母大写

const logger = new Logger();          // 创建实例

logger.on('messageLogged', (arg) => {
  console.log('Listener called', arg);
});

logger.log('message');                // → message / Listener called { id: 1, ... }
```

![620](Node-17-logger-usage.jpg)
*图 24 · 68:30 · app.js：require 类 → new 实例 → 在同一实例上注册监听器并调用*

![480](Node-17-logger-output.jpg)
*图 25 · 69:50 · 终端输出：message 与 Listener called { id: 1, url: 'http://' }*

> [!success] 核心套路（背下来）
> 1. 定义一个类，`extends EventEmitter`；
> 2. 类里该触发事件时用 `this.emit(...)`（`this` 就是继承来的 emitter 本身）；
> 3. 想加业务能力就加自己的方法（这里是 `log`）；
> 4. `module.exports = 类名`；
> 5. 使用方 `new` 出来，==在**同一个对象**上 `on` 和 `log`==——监听器和触发器同源，事件才通。

```text
first-app/
├── app.js          ← 主模块：require 类、new 实例、注册监听、调用方法
└── logger.js       ← 可复用模块：class Logger extends EventEmitter
```

---

## 十七、HTTP 模块（01:10:46）：起一个 Web 服务器

```js
const http = require('http');

const server = http.createServer((req, res) => {
  if (req.url === '/') {
    res.write('Hello World');
    res.end();
  }

  if (req.url === '/api/courses') {
    res.write(JSON.stringify([1, 2, 3]));      // 对象 → JSON 字符串
    res.end();
  }
});

server.listen(3000);
console.log('Listening on port 3000...');
```

```bash
node app.js
# Listening on port 3000...
# 浏览器访问 http://localhost:3000            → Hello World
# 浏览器访问 http://localhost:3000/api/courses → [1,2,3]
```

![520](Node-18-listening-terminal.jpg)
*图 26 · 74:40 · 终端显示 Listening on port 3000...，浏览器每次刷新都会触发 New connection...*

![520](Node-18-hello-world.jpg)
*图 27 · 77:27 · 浏览器访问 localhost:3000 得到 Hello World*

### 17.1 底层一瞥：`server` 本身就是一个 EventEmitter

```js
server.on('connection', (socket) => console.log('New connection'));
```

每来一个新连接就触发 `connection` 事件——证明 ==`http.Server` 继承自 `net.Server`，而 `net.Server` 是一个 `EventEmitter`==。**Node 的很多核心功能都长在 EventEmitter 上**，这就是为什么第 14 节要单独讲事件。

![440](Node-18-http-event-diagram.jpg)
*图 28 · 54:10 · HTTP 服务器收到新请求时触发 "Event: New Request"，你的任务就是响应它*

![440](Node-18-http-docs-events.jpg)
*图 29 · 72:26 · 官方文档里 http.Server 的事件列表：request / connection / close……它是 EventEmitter 的子类*

> [!note] 为什么平时不这么写
> 监听 `connection` 太底层了——你拿到的是裸 socket，要自己解析 HTTP 报文。日常直接用 `createServer` 的回调，它已经把 `request` / `response` 对象准备好了。

### 17.2 路由：`req.url` + if

```js
if (req.url === '/') { ... }
if (req.url === '/api/courses') { ... }
```

```bash
Ctrl + C          # 关掉旧进程
node app.js
```

![620](Node-18-http-code.jpg)
*图 30 · 76:07 · 最终版 app.js：按 req.url 分流的两个路由（/ 与 /api/courses）*

> [!warning] 每改一次代码都要手动重启？
> 视频这里也吐槽了这点（说后面会讲自动化）。
> **当时的方案**是 `nodemon app.js`；**现在**可以直接用 Node 内置的 `node --watch app.js`（Node 18.11+），不用装任何东西。

### 17.3 为什么真实项目不用裸 `http`

路由一多，`if` 就一层层堆在同一个回调里，代码变成"线性堆叠"，越来越难维护。

> [!success] 所以用 **Express**
> Express 内部就建在 Node 的 `http` 模块之上，但提供了干净的路由结构（`app.get('/api/courses', handler)`）。
> 这也解释了为什么学 Node **必须**先懂 `http`——框架只是薄薄一层封装。

---

## 十八、总结：一条主线串完 18 章

```text
Node 是什么 ── V8 + 额外模块，浏览器之外的 JS 运行时
     │
为什么快 ──── 单线程 + 非阻塞 + 事件队列（I/O 密集友好，CPU 密集勿用）
     │
怎么写代码 ── 每个文件是一个模块，默认私有
     │          导出 module.exports，加载 require
     │          底层：代码被包进 (exports, require, module, __filename, __dirname) 函数
     │
能用什么 ──── path（路径）· os（系统）· fs（文件，异步优先）
     │        events（EventEmitter：on / emit，自定义类 extends EventEmitter）
     │        http（createServer + req.url 路由 → 引出 Express）
     ▼
一个能跑的 Node 后端
```

> [!success] 五条最该记住的
> 1. **Node 不是语言、不是框架**，是运行时环境（V8 + 模块）。
> 2. **单线程 + 非阻塞**：I/O 密集是主场，CPU 密集是雷区。
> 3. **每个文件即模块**，靠 `module.exports` / `require` 连接；`exports` 只是引用，不能整体重新赋值。
> 4. **`fs` 永远用异步版本**，回调第一个参数是 `err`。
> 5. **要发事件就 `extends EventEmitter`**，别 `new` 两个实例指望它们互通。

> [!info] 视频录制于 2018（Node 8），今天可以补的课
> - 版本：LTS 已到 v22 / v24，安装用 nvm / fnm 管多版本；
> - 模块：除 CommonJS 外还有 **ESM**（`import` / `export`，后缀 `.mjs` 或 `"type": "module"`），新项目可优先考虑；
> - 内置模块可加 `node:` 前缀：`require('node:fs')`，语义更清晰；
> - `fs/promises` + `async/await` 比回调更顺手：`const fs = require('fs/promises'); const files = await fs.readdir('./');`
> - 热重载：`node --watch` 内置，不必再装 nodemon；
> - 裸 `http` → 生产用 **Express / Fastify / NestJS**；CPU 密集场景可用 `worker_threads` 补齐。

---

## 相关笔记

| 笔记 | 关联点 |
| :--- | :--- |
| [[npm和npx有什么区别]] | 装包（npm）与临时执行（npx） |
| [[开发依赖和生产依赖]] | `package.json` 的两本账，`dependencies` vs `devDependencies` |
| [[如何启动TS]] | Node 环境下的 TypeScript 工程启动流程 |
| [[沙箱]] | 进程 / 权限隔离相关概念 |

---

*视频来源：[Programming with Mosh — Node.js Tutorial for Beginners](https://youtu.be/TlB_eWDSMt4)（2018-02-21，18 章）· 参考：Node.js 官方文档 Modules / fs / path / os / events / http*
