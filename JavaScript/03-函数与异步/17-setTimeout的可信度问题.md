---
title: "17 · setTimeout 的可信度问题"
aliases:
  - Trust issues with setTimeout()
  - setTimeout Trust Issues
tags:
  - JavaScript
  - Namaste-JavaScript
  - 异步
episode: 17
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 17 讲：setTimeout 的可信度问题（Trust issues with setTimeout()）

> [!abstract] 本讲要点
> - 设为 5 秒的 `setTimeout` 并不能保证回调正好在 5 秒后执行。
> - 回调必须等调用栈清空后才能执行，因此实际延迟可能更长。
> - 这是由 JS 的并发模型（Concurrency Model）决定的。
> - 把 timeout 设为 0 可以把不重要的函数稍作延后。

## 一、setTimeout 并不保证精确

- 设为 5 秒的 setTimeout，有时并不能确切保证回调函数正好在 5 秒后执行。
- 让我们观察下面的代码及其解释。

```js
console.log("Start");
setTimeout(function cb() {
  console.log("Callback");
}, 5000);
console.log("End");
// 数百万行待执行的代码

// 输出：这里的 setTimeout 并不保证回调函数正好在 5 秒后被调用。也许是 6 秒、7 秒甚至 10 秒！这一切都取决于调用栈。为什么？
```

### 原因是什么？

- 首先创建 GEC（全局执行上下文）并推入调用栈。
- 控制台打印 Start。
- 当看到 setTimeout 时，回调函数被注册到 Web API 的环境中，并给它绑定计时器并启动。回调等待计时器到期后轮到它执行。但 JS 不会为谁等待，继续执行下一行。
- 控制台打印 End。
- 在 "End" 之后，我们有 100 万行代码，需要 10 秒（假设）执行完毕。因此 GEC 不会从栈中弹出，它要运行这 10 秒的所有代码。
- 但在后台，计时器会运行 5 秒。当调用栈还在运行这 100 万行代码时，计时器早已到期，回调函数已经被推入回调队列，等待被推入调用栈执行。
- 事件循环不断检查调用栈是否为空。但这里 GEC 仍在栈中，所以 cb 无法从回调队列弹出并推入调用栈。**尽管 setTimeout 只设了 5 秒，它仍要等到 10 秒、直到调用栈为空才能执行**（10 秒后 GEC 弹出，cb() 被推入调用栈并立即执行——推入调用栈的内容都会立刻执行）。
- 这就是 JS 的**[并发模型（Concurrency Model）](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)**。这就是 setTimeout 可信度问题背后的逻辑。

## 二、不要阻塞主线程

- JavaScript 的第一条规则：**不要阻塞主线程**（因为 JS 是单线程语言，只有一个调用栈）。
- 下面的例子中，我们阻塞了主线程。观察问题和输出。

![setTimeout 演示](settimeout1.jpg "setTimeout 演示")

- setTimeout 保证的是：执行代码至少需要等待给定的时间。

- JS 是一门同步、单线程的语言。仅凭 1 个线程，它就能运行所有代码。它有点像解释型语言，在浏览器中运行代码非常快（无需等待代码编译完成）（JIT，即时编译）。同时它仍然有多种方式完成异步操作。

## 三、如果 timeout = 0 呢？

```js
console.log("Start");
setTimeout(function cb() {
  console.log("Callback");
}, 0);
console.log("End");
// 即使 timer = 0s，cb() 也必须经过队列：在 Web API 环境中注册回调，移动到回调队列，并在调用栈为空时执行。
// 输出 - Start End Callback
// 把 timer 设为 0 的这种做法，可以把不太重要的函数稍作延后，让更重要的函数（这里打印 "End"）先执行。
```

> [!tip] 记忆法
> `setTimeout` 只保证「至少」等待给定时间；回调真正执行的时间取决于调用栈何时空闲。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=nqsPmuicJJc&ab_channel=AkshaySaini)
