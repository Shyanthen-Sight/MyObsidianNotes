---
title: "15 · 异步 JavaScript 与事件循环"
aliases:
  - Asynchronous JavaScript & EVENT LOOP from scratch
  - Event Loop
tags:
  - JavaScript
  - Namaste-JavaScript
  - 事件循环
episode: 15
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 15 讲：异步 JavaScript 与事件循环（Asynchronous JavaScript & EVENT LOOP from scratch）

> [!abstract] 本讲要点
> - 浏览器通过 Web API 为 JS 调用栈提供定时器、DOM、网络请求等能力。
> - 定时器到期后，回调进入回调队列（Callback Queue），由事件循环推入调用栈执行。
> - Promise 与 Mutation Observer 产生的回调进入微任务队列（Microtask Queue），优先级高于回调队列。
> - 微任务不断产生新任务时，回调队列中的任务会一直得不到执行，称为「饥饿」（starvation）。
> - `setTimeout` 的延迟并不精确，回调必须等调用栈为空才能执行。

> [!note] 注意
> 调用栈会执行任何进入其中的执行上下文。时间与潮汐不等人，JS 也不会为谁停留。简而言之：**调用栈没有计时器。**

## 一、浏览器、JS 引擎与调用栈

- 浏览器中有 JS 引擎，JS 引擎中有调用栈，调用栈中有全局执行上下文（Global Execution Context）、局部执行上下文等。
  - 但浏览器还有许多其他「超能力」——本地存储空间、计时器、输入 URL 的地方、蓝牙访问、地理定位访问等等。
  - 现在，JS 需要某种方式把调用栈与这些超能力连接起来。这通过 Web API 完成。

![事件循环示意图 1](eventloop1.jpg "事件循环示意图 1")

## 二、Web API

以下内容没有一个是 JavaScript 的一部分！它们是浏览器拥有的额外超能力。浏览器把这些能力的访问权限交给 JS 调用栈使用。

![事件循环示意图 2](eventloop2.jpg "事件循环示意图 2")

- `setTimeout()`、DOM API、`fetch()`、localStorage、console（没错，连 `console.log` 都不是 JS！）、location 以及更多其他内容。
  - setTimeout()：定时器函数。
  - DOM API：例如 Document.xxxx；用于访问 HTML DOM 树。（Document Object Manipulation，文档对象操作）
  - fetch()：用于与外部服务器建立连接，例如 Netflix 服务器等。
- 我们是通过全局对象 `window` 在调用栈中拿到这些能力的。
  - 像这样使用 `window` 关键字：`window.setTimeout()`、`window.localstorage`、`window.console.log()` 来在控制台中输出内容。
  - 由于 window 是全局对象，而上面这些函数都存在于全局对象上，我们不必显式写 `window`，但它是隐含的。

- 让我们理解下面的代码图和它的解释：

  ![事件循环示意图 3](eventloop3.jpg "事件循环示意图 3")

- 代码如下：

```js
console.log("start");
setTimeout(function cb() {
  console.log("timer");
}, 5000);
console.log("end");
// start end timer
```

- 首先创建一个 GEC（全局执行上下文），并把它放入调用栈。
- `console.log("Start");`：这会调用 console Web API（通过 window），而它实际上修改了控制台中的值。
- `setTimeout(function cb() { ... }, 5000);`：这会调用 setTimeout Web API，从而获得计时器能力。它保存回调 `cb()` 并启动计时器。
- `console.log("End");`：调用 console API 并在控制台窗口中输出。此后 GEC 从调用栈中弹出。
- 在这一切发生的同时，计时器一直在走。当它归零后，回调 `cb()` 就需要运行了。
- 现在我们需要让这个 cb 进入调用栈，只有这样它才会被执行。为此我们需要**事件循环（Event Loop）**和**回调队列（Callback Queue）**。

## 三、事件循环与回调队列

**问：5 秒后计时器是如何输出到控制台的？**

- cb() 不能直接进入调用栈被执行。当计时器到期时，它要先经过回调队列。
- 事件循环不断检查回调队列，看其中是否有元素需要放进调用栈。它就像一个守门人。
- 一旦 cb() 进入回调队列，事件循环就会把它推入调用栈运行。此时使用 console API，输出被打印出来。

![事件循环示意图 4](eventloop4.jpg "事件循环示意图 4")

**问：另一个理解事件循环与回调队列的例子。**

看下面的图片和代码，尝试理解其中的原因：

![事件循环示意图 5](eventloop5.jpg "事件循环示意图 5")

解释？

```js
console.log("Start");
document.getElementById("btn").addEventListener("click", function cb() {
  // cb() 注册在 Web API 环境中，并绑定了事件（click）。即：注册回调并给它绑定事件。
  console.log("Callback");
});
console.log("End"); // 调用 console API 并在控制台窗口中输出。此后 GEC 从调用栈中移除。
// 在上面的代码中，即使控制台已经打印了 "Start" 和 "End" 并且 GEC 已弹出，事件监听器仍留在 Web API 环境中（期待用户某天会点击它），直到被显式移除或浏览器关闭。
```

- 事件循环只有一个职责：不断检查回调队列，一旦发现内容就把它推入调用栈，并从回调队列中删除。

**问：为什么需要回调队列？**

**答**：假设用户点击按钮 6 次。于是 6 个 cb() 被放入回调队列。事件循环检查调用栈是否为空/是否有空间，以及回调队列是否非空（这里有 6 个元素）。回调队列中的元素被弹出、放入调用栈、执行，然后从调用栈中弹出。

## 四、fetch 的行为（**微任务队列？**）

让我们观察下面的代码并尝试理解。

```js
console.log("Start"); // 这会调用 console Web API（通过 window），进而真正修改控制台中的值。
setTimeout(function cbT() {
  console.log("CB Timeout");
}, 5000);
fetch("https://api.netflix.com").then(function cbF() {
    console.log("CB Netflix");
}); // 需要 2 秒来返回响应
// 数百万行代码
console.log("End");
```

### 代码解释

- 上述代码中 fetch() 之前的所有步骤与之前相同。
- fetch 会把 cbF 与已有的 cbT 一起注册到 Web API 环境中。
- cbT 在等待 5000 毫秒结束，以便能进入回调队列；cbF 在等待 Netflix 服务器返回数据，大约需要 2 秒。
- 之后这数百万行代码开始运行。等这数百万行代码执行完时，5 秒已经过去，此时计时器到期，Netflix 服务器的响应也已就绪。
- cbF 拿到的数据已准备好执行，它会被存入一个叫**微任务队列（Microtask Queue）**的地方。
- 同时，计时器到期后，cbT 也已准备好在回调队列中执行。
- 微任务队列与回调队列完全相同，但优先级更高。微任务队列中的函数会比回调队列中的函数更早执行。
- 在控制台中，首先打印 Start 和 End。然后 cbF 进入调用栈，打印 "CB Netflix"，cbF 从调用栈弹出。接着 cbT 从回调队列中取出，放入调用栈，打印 "CB Timeout"，cbT 从调用栈移除。
- 更多理解见下图。

![事件循环示意图 6](eventloop6.jpg "事件循环示意图 6")

微任务优先级可视化

![微任务优先级可视化](microtask.gif "微任务优先级可视化")

### 什么会进入微任务队列？

- 所有通过 Promise 产生的回调函数都会进入微任务队列。
- **Mutation Observer（变更观察器）**：不断检查 DOM 树中是否有变更，如果有，就执行某个回调函数。
- 通过 Promise 和 Mutation Observer 产生的回调函数进入**微任务队列**。
- 其余所有内容进入**回调队列（Callback Queue），又称任务队列（Task Queue）**。
- 如果微任务队列中的任务不断在队列中创建新任务，回调队列中的元素就永远没有机会执行。这被称为**饥饿（starvation）**。

## 五、几个重要问题

1. **事件循环究竟何时开始？** 顾名思义，事件循环是一个单线程的循环，几乎是*无限的*。它一直在运行并执行自己的职责。
2. **是否只有异步 Web API 回调才会注册到 Web API 环境中？** 是的。像传给 map、filter、reduce 的那种同步回调函数不会注册到 Web API 环境中。只有那些异步回调函数才会经历这一整套流程。
3. **Web API 环境只存储回调函数，并把同一个回调推入队列/微任务队列吗？** 是的，回调函数会被存储，同时在队列中安排一个引用。此外，对于事件监听器（例如点击处理器），原始回调会永远留在 Web API 环境中，这就是为什么建议在不再使用时显式移除监听器，好让垃圾回收器完成工作。
4. **如果 setTimeout 的延迟是 0ms 会怎样？回调是不是不用等待就能进入队列？** 不是，`setTimeout()` 存在信任问题 😅。回调函数必须等待调用栈为空。因此，即使回调只有 0ms，如果调用栈忙碌，也可能要等上 100ms。

## 六、事件循环、回调队列与微任务队列的观察（GIF）

![微任务观察 1](microtask1.gif "微任务观察 1")

![微任务观察 2](microtask2.gif "微任务观察 2")

![微任务观察 3](microtask3.gif "微任务观察 3")

![微任务观察 4](microtask4.gif "微任务观察 4")

![微任务观察 5](microtask5.gif "微任务观察 5")

![微任务观察 6](microtask6.gif "微任务观察 6")

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=8zKuNo4ay8E&ab_channel=AkshaySaini)
