---
title: Node 架构与事件循环
tags: [NodeJS, V8, libuv, 事件循环, 微任务]
course: "A Complete NODE JS Course #27-#29"
created: 2026-11-13
官方文档:
  - "[事件循环官方指南](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)"
  - "[process 模块](https://nodejs.org/docs/latest/api/process.html)"
  - "[Timers](https://nodejs.org/docs/latest/api/timers.html)"
---

# 09 - Node 架构与事件循环

> [!info] 本篇覆盖
> 课程 #27 架构、#28 事件循环、#29 事件循环实践。
> 一句话目标：**搞懂 V8 / libuv / 线程池的分工，画出事件循环六个阶段，能预判一段代码的输出顺序。**这是整个板块的理论收官战。

---

## 一、Node 架构全景（#27）

![[img/course-09-architecture.jpg]]
*图 09-1 · 00:06:50 · V8 + libuv：主线程、事件循环与线程池*

| 组件 | 角色 | 负责什么 |
|:---|:---|:---|
| **V8** (C++) | JS 引擎 | 编译执行你的 JS 代码（出自 Chrome 项目） |
| **libuv** (C++) | 异步 I/O 库 | **事件循环**（Event Loop）+ **线程池**（Thread Pool） |
| **Main Thread** | 主线程 | 唯一执行你 JS 代码的地方 |
| **Event Loop** | 调度器 | 决定"下一个执行哪段回调" |
| **Thread Pool** | 后台线程（默认 4 条） | 干重活：文件 I/O、DNS、加密等 |

### 2.2 架构背后的分工逻辑

```js
const fs = require('fs');

function someTask() {
  console.log('some task');
}

fs.readFile('./input.txt', 'utf-8', (err, data) => {   // ① libuv 线程池干活
  console.log(data);                                    // ③ 完成后回调回主线程
});

someTask();                                             // ② 主线程立刻执行
```

- 主线程把文件读取**委托**给线程池，自己继续往下跑
- 线程池读完后把回调放进队列
- 事件循环在合适的阶段取出回调，回到主线程执行

> [!tip] 一图流记忆
> **主线程只跑 JS，重活给线程池，调度靠事件循环。** 这就是"单线程非阻塞"的全部秘密。

更多底层细节：[官方事件循环指南](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)。

---

## 二、事件循环：六个阶段（#28）

![[img/course-09-event-loop-phases.jpg]]
*图 09-2 · 00:16:11 · 事件循环阶段 + 微任务队列（process.nextTick）*

每转一圈（一个 tick）依次经过：

| 阶段 | 干什么 | 谁的回调 |
|:---|:---|:---|
| ① **Expired Timers** | 执行到期的定时器 | `setTimeout` / `setInterval` |
| ② **IO Polling** | 收集 I/O 完成事件，执行其回调 | 文件、网络的回调 |
| ③ **setImmediate** | 紧随 IO 阶段执行 | `setImmediate(cb)` |
| ④ **Close Callbacks** | 关闭事件 | `socket.on('close')` |
| ↻ | 转完一圈再来一圈 | …… |

**两个特殊队列（每个阶段之间插队清空）：**

| 队列 | API | 优先级 |
|:---|:---|:---|
| Microtask Queue | Promise 回调（`.then`） | 高 |
| **NextTick Queue** | `process.nextTick(cb)` | ==最高，比 Promise 还先== |

执行优先级总结：**当前同步代码 → process.nextTick → Promise 微任务 → 当前阶段的定时器/IO 回调 → 下一阶段**。

> [!warning] 为什么 node --check 检测不到这些坑
> 阶段顺序是运行时行为，语法检查器管不了。判断执行顺序只能靠下图这套模型推演——这正是 #29 实践题的意义。

---

## 三、实践：预判输出顺序（#29）

![[img/course-09-event-loop-practice.jpg]]
*图 09-3 · 00:22:20 · 混合定时器与 IO 的实测输出*

### 例题 1：定时器三兄弟

```js
setTimeout(() => console.log('Timer callback executed'), 0);
setImmediate(() => console.log('Immediate callback executed'));
console.log('Program has completed');
```

```text
Program has completed
Timer callback executed
Immediate callback executed
```

同步代码最先跑完；Timers 阶段在 setImmediate 阶段之前，所以 `setTimeout(0)` 先输出。

### 例题 2：混入 IO 与 nextTick

```js
fs.readFile('./input.txt', () => {
  console.log('IO callback');
  setImmediate(() => console.log('Immediate callback executed'));
  setTimeout(() => console.log('Timer callback executed'), 0);
  process.nextTick(() => console.log('nextTick callback executed'));
});
```

在 **IO 回调内部**，顺序反过来了：`nextTick → Immediate → Timer`——因为 IO 阶段之后紧跟 setImmediate 阶段，而 Timers 要等下一圈。

> [!tip] 记忆锚点
> ==**主线程里 Timer 先，IO 回调里 Immediate 先。**== 面试高频题。

### 例题 3：阻塞主线程的代价

```js
const start = Date.now();
setTimeout(() => console.log('Delay:', Date.now() - start), 100);

// 同步死循环 500ms
while (Date.now() - start < 500) {}
```

输出 `Delay: ≈500`——事件循环被同步代码卡住，定时器只能排队。**永远不要在主线程做重计算**（CPU 密集场景用 worker_threads，见[官方文档](https://nodejs.org/docs/latest/api/worker_threads.html)）。

---

## 四、理论回望：前面每一集都发生在架构的哪里

| 已学内容 | 架构角色 |
|:---|:---|
| `readFileSync`（[[02-文件系统与异步IO]]） | 主线程被阻塞——线程池闲置 |
| `fs.readFile` 异步回调 | 线程池执行，回调经事件循环回到主线程 |
| `server.on('request')`（[[03-构建HTTP服务器]]） | libuv 监听端口，请求到达即触发 IO Polling 阶段 |
| `rs.on('data')`（[[07-事件驱动与Streams]]） | 流的 chunk 事件由 libuv 泵出 |
| `rl.on('close')`（[[01-Node.js概述与环境搭建]]） | EventEmitter 只是事件循环上的回调约定 |

---

## 五、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| `setTimeout(fn, 0)` 想"立即"执行 | 总是晚于同步代码 | 同步逻辑不要依赖 0 延迟 |
| `process.nextTick` 递归 | 事件循环**饿死**，永远转不到下一阶段 | 换 `setImmediate` |
| 主线程 while 大循环 | 所有回调、定时器全部延迟 | worker_threads / 拆任务 |
| Promise 与 setTimeout 混用顺序记不住 | 输出与预期不符 | 回看图 09-2 的队列优先级 |

> [!summary] 自查清单
> - [ ] 能画出 V8 / libuv / 主线程 / 事件循环 / 线程池的关系
> - [ ] 背出事件循环六阶段与两个微任务队列
> - [ ] 能预判"主线程 Timer 先、IO 回调里 Immediate 先"
> - [ ] 理解同步代码阻塞事件循环的后果

---
**相关笔记**：[[02-文件系统与异步IO]] · [[07-事件驱动与Streams]] · [[01-Express入门与RESTful设计]] · [[README|课程导航]]

*01-Node.js基础 板块完结 ✅ 下一板块：[[01-Express入门与RESTful设计]]*
