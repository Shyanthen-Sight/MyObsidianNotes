---
title: await 运算符
aliases:
  - await
  - await 操作符
tags:
  - JavaScript
  - 异步编程
  - Promise
  - MDN精读
source: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await
author: MDN Web Docs
created: 2026-11-16
---

# await 运算符（MDN 精读）

> [!info] 资料来源
> 本笔记基于 [MDN：await - JavaScript](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await) 完整精读整理，所有示例代码均来自原文。
> 通俗解读部分用 💡 标注，帮助理解，非 MDN 原文。

## 一句话总结

> `await` = **等一个 Promise 出结果，然后把结果"拆箱"直接给你**。它只能用在 `async` 函数里或模块顶层。

---

## 1. 它是什么（MDN 定义）

> `await` 操作符用于等待一个 Promise 兑现并获取它兑现之后的值。它只能在**异步函数**或者**模块顶层**中使用。

💡 三个关键词：
- **等待**：停在原地，直到 Promise 出结果
- **拆箱**：拿到的是"兑现后的值"，不再是 Promise
- **使用限制**：必须在 `async` 函数内，或 ES 模块顶层（Top-level await）

### 语法

```js-nolint
await expression;
```

### 参数

- `expression`：要等待的 **Promise 实例、Thenable 对象，或任意类型的值**

### 返回值

返回从 Promise 实例或 thenable 对象取得的处理结果；**如果等待的值不符合 thenable，则返回表达式本身的值**。

### 异常

**拒绝（reject）的原因会被作为异常抛出**——就像同步代码里 `throw` 一样，可以用 `try...catch` 接住。

---

## 2. 核心行为（描述章节精读）

> 使用 `await` 总会**暂停当前异步函数的执行**，在该 Promise **敲定（settled，指兑现或拒绝）后继续执行**。函数恢复执行时，`await` 表达式的值已经变成了 Promise 兑现的值。

💡 注意"暂停"只暂停**这个 async 函数自己**，主线程马上被释放去干别的——这就是"异步不阻塞"的本质。

若 Promise 被拒绝，`await` 会把拒绝原因抛出，并且**当前函数会出现在错误的栈追踪里**（细节见第 8 节）。

### ⭐ 三种值的处理规则（与 `Promise.resolve()` 行为一致）

`await` 总会**同步地**对表达式求值并处理，规则如下：

| 表达式是什么 | await 怎么处理 |
|---|---|
| **原生 Promise**（`expression.constructor === Promise`） | 直接用于等待，由原生代码实现，**该对象的 `then()` 不会被调用** |
| **thenable 对象**（非原生 Promise、polyfill、Proxy、派生类等） | 会**构造一个新 Promise** 用于等待，构造时**会调用该对象的 `then()` 方法** |
| **不是 thenable**（数字、字符串、普通对象等） | 包装进一个**已兑现的 Promise**，结果就是表达式本身的值 |

💡 面试考点：`await new Promise(...)` 与 `await fakeThenable(...)` 走的是完全不同的路径——后者会额外创建一个 Promise 并调用 `then()`。

---

## 3. 示例一：等待 Promise 兑现

```js
function resolveAfter2Seconds(x) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(x);
    }, 2000);
  });
}

async function f1() {
  let x = await resolveAfter2Seconds(10);
  console.log(x); // 10
}

f1();
```

💡 2 秒后 Promise 兑现为 10，`await` 把 10 "拆箱"给 `x`——写法上像同步代码，这就是 async/await 的意义。

---

## 4. 示例二：等待非 Promise 值（自动转换）

若表达式的值不是 Promise，`await` 会把该值转换为**已兑现的 Promise**，然后返回其结果。

```js
async function f3() {
  const y = await 20;
  console.log(y); // 20

  const obj = {};
  console.log((await obj) === obj); // true
}

f3();
```

💡 `await 20` → 20 被包成"已兑现的 Promise"再拆出来，值不变；`await {}` → 原样返回，还是同一个对象（`===` 为 true）。

---

## 5. 示例三：Promise 被拒绝 → 抛异常

如果 Promise 被拒绝，则**抛出拒绝的原因**。

```js
async function f4() {
  try {
    const z = await Promise.reject(30);
  } catch (e) {
    console.error(e); // 30
  }
}

f4();
```

💡 拒绝值 30 被 `catch` 接住——`await` 让异步错误可以用同步的 `try...catch` 处理。

---

## 6. 示例四：用 `.catch()` 提前拦截拒绝

你可以**链式调用 `catch()`**（而不是使用 `try`），在等待 promise 兑现之前处理被拒绝的 promise。

```js
const response = await promisedFunction().catch((err) => {
  console.error(err);
  return "default response";
});
// response will be "default response" if the promise is rejected
```

💡 技巧：先 `.catch()` 给个兜底值，`await` 拿到的就永远是正常值，外层不需要 try/catch。适合"失败就给默认值"的场景。

---

## 7. 示例五：模块顶层 await（Top-level await）

在**模块的顶层**，你可以单独使用 `await`（异步函数的外面）。也就是说：一个模块如果包含用了 `await` 的子模块，该模块就会**等待该子模块**，**这一过程并不会阻塞其他子模块**。

```js
// fetch request
const colors = fetch("../data/colors.json").then((response) => response.json());

export default await colors;
```

💡 任何文件只要导入这个模块，后面的代码就会等待，直到 fetch 完成。适合：加载配置文件、连接数据库后再导出。注意只在 **ES 模块**中可用（普通 `<script>` 不行）。

---

## 8. ⭐⭐ 最重要的一节：await 对执行过程的影响

> 当函数执行到 `await` 时，被等待的表达式**会立即执行**，所有依赖该表达式的值的代码会被**暂停**，并**推送进微任务队列**。然后主线程被释放出来，用于事件循环中的下一个任务。**即使等待的值是已经敲定的 promise 或不是 promise，也会发生这种情况。**

### 8.1 没有 await：整体同步

```js
async function foo(name) {
  console.log(name, "start");
  console.log(name, "middle");
  console.log(name, "end");
}

foo("First");
foo("Second");

// First start
// First middle
// First end
// Second start
// Second middle
// Second end
```

### 8.2 有 await：函数被"切"成两段

```js
async function foo(name) {
  console.log(name, "start");
  await console.log(name, "middle");
  console.log(name, "end");
}

foo("First");
foo("Second");

// First start
// First middle
// Second start
// Second middle
// First end
// Second end
```

💡 **划重点**：执行到 `await` 时，**后面的代码整体被安排进一个新的微任务**，此后的函数体变为异步执行。所以两个 `foo` 交替输出——这就是输出顺序题的原理。

### 8.3 对应的 Promise 写法（帮助理解本质）

```js
function foo(name) {
  return new Promise((resolve) => {
    console.log(name, "start");
    resolve(console.log(name, "middle"));
  }).then(() => {
    console.log(name, "end");
  });
}
```

> [!important] 原文的核心提醒
> 虽然这里的 `then()` 看起来很多余（代码完全可以合并到构造器回调里），但**不管该 Promise 的状态如何，`then()` 的回调总会被异步执行**，`await` 的行为也一样。因此，**只要情况不是必须或可能需要等待 Promise 的结果，就应该避免使用 `await`**。

💡 这解释了一个反直觉现象：`await null`（等一个普通值）也会让后续代码变成微任务！所以滥用 `await` 会带来不必要的调度开销。

### 8.4 微任务的轮流调度

其他微任务能在异步函数恢复之前执行：

```js
let i = 0;

queueMicrotask(function test() {
  i++;
  console.log("microtask", i);
  if (i < 3) {
    queueMicrotask(test);
  }
});

(async () => {
  console.log("async function start");
  for (let i = 1; i < 3; i++) {
    await null;
    console.log("async function resume", i);
  }
  await null;
  console.log("async function end");
})();

queueMicrotask(() => {
  console.log("queueMicrotask() after calling async function");
});

console.log("script sync part end");

// Logs:
// async function start
// script sync part end
// microtask 1
// async function resume 1
// queueMicrotask() after calling async function
// microtask 2
// async function resume 2
// microtask 3
// async function end
```

💡 解读：`test()` 总会在异步函数恢复执行前被调用，呈现**轮流的调度**。微任务按**入队先后顺序**执行——`queueMicrotask(...) after calling async function` 比第一次 `await` 的恢复**晚入队**，所以它排在该次恢复之后输出。

---

## 9. 改善栈追踪（省略 await 的代价）

有时当异步函数直接返回 Promise 时，我们会省略 `await`：

```js
async function noAwait() {
  // Some actions...

  return /* await */ lastAsyncTask();
}
```

但如果这个 Promise 来自异步函数、且其异步部分抛错：

```js
async function lastAsyncTask() {
  await null;
  throw new Error("failed");
}

async function noAwait() {
  return lastAsyncTask();
}

noAwait();

// Error: failed
//    at lastAsyncTask
```

💡 栈追踪里**只有 `lastAsyncTask`**——因为抛错时 `noAwait` 早已返回，这个 Promise 在某种意义上已经和它无关了。排查问题时会丢关键线索。

加上 `return await` 后：

```js
async function lastAsyncTask() {
  await null;
  throw new Error("failed");
}

async function withAwait() {
  return await lastAsyncTask();
}

withAwait();

// Error: failed
//    at lastAsyncTask
//    at async withAwait
```

💡 错误在 `withAwait` 函数体结束**前**抛出并被包装进新 Promise，所以主调函数出现在栈追踪里——**调试信息完整了**。

> [!note] 代价
> 这样会有一点性能牺牲——毕竟 Promise 会被**拆装了又再次包装**。
> 实践建议：正常代码用 `return await` 优先保证可调试性；性能敏感且确认无碍时才省略。

---

## 10. 快速记忆卡

| 问题 | 答案 |
|---|---|
| `await` 后面跟普通值会怎样？ | 包装成已兑现的 Promise，值原样返回，**但后续代码仍变成微任务** |
| `await` 拒绝的 Promise 会怎样？ | 把拒绝原因当异常抛出，可 `try...catch` 或前置 `.catch()` |
| `await` 能在哪里用？ | `async` 函数内、ES 模块顶层 |
| `await fakePromise`（非原生）会怎样？ | 新建 Promise 并调用其 `then()` |
| 为什么 `await null` 也影响输出顺序？ | 恢复执行的代码被排进微任务队列，永远异步 |
| `return promise` vs `return await promise`？ | 后者栈追踪更完整，前者少一层包装略快 |

### 关联笔记

- [[事件循环]]
- [[05-Promise与async-await]]
