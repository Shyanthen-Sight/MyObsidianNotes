---
title: "24 · Promise API"
aliases:
  - Promise APIs
  - Promise.all
  - Promise.allSettled
  - Promise.race
  - Promise.any
tags:
  - JavaScript
  - Namaste-JavaScript
  - 异步编程
episode: 24
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 24 讲：Promise API（all、allSettled、race、any）与面试题（Promise APIs）

> [!abstract] 本讲要点
> - 最常用的 4 个 Promise API：`Promise.all()`、`Promise.allSettled()`、`Promise.race()`、`Promise.any()`。
> - `all` 是快速失败：全部成功才成功，任一失败立即失败。
> - `allSettled` 会等全部 settle，并给出累积结果。
> - `race` 取第一个 settle 的结果（成功或失败都算）。
> - `any` 取第一个成功的结果，全部失败时抛出 `AggregateError`。

最常用的 4 个 Promise API：

- Promise.all()
- Promise.allSettled()
- Promise.race()
- Promise.any()

> [!tip] 不懂 Promise，就用不好 async/await
> 没有人可以在不了解 promise 的情况下用好 async/await。

## 一、Promise.all()

> Promise 是一个占位符，代表某个稍后才会可用的值。promise 帮助处理异步操作。JavaScript 提供了辅助函数 `Promise.all(promisesArrayOrIterable)`，用于并行地一次处理多个 promise，并以单个聚合数组的形式拿到结果。

问：在什么场景下会用到这个 API？
答：假设你需要并行发起 API 调用并拿到结果，该怎么做？这时就可以用 Promise.all。它用于把多个 promise 一起处理。

`Promise.all([p1, p2, p3])` -> 假设我们发起 3 个 API 调用来取数据。同时假设 **p1** 需要 **3 秒**，**p2** 需要 **1 秒**，**p3** 需要 **2 秒**。

第一种场景，假设 3 个 promise 都成功。那么 Promise.all 会花 **3 秒**，并给出结果 promise，值为 `[val1, val2, val3]`。它会等所有 promise 都完成，然后收集结果，以数组形式输出。

如果其中某个 promise 被 reject 会怎样？例如 `Promise.all([p1, p2, p3])`，这次 p2 在 1 秒后被 reject。于是 Promise.all 会在错误发生的瞬间立即抛出与 p2 相同的错误。它不会等其他 promise 变成成功或失败。此外，p1 和 p3 不会被取消，因为它们已经被触发，所以会按各自的命运走向成功或失败，但 Promise.all 不会关心。所以这是一种「或/空」的局面。

💡 总结来说，`Promise.all()` 会等待所有输入 promise resolve，并返回一个新的 promise，其 resolve 值是包含各输入 promise 结果的数组。如果其中一个输入 promise 被 reject，`Promise.all()` 会立即返回一个被 reject 的 promise，错误来自第一个被 reject 的 promise。

## 二、Promise.allSettled()

> `Promise.allSettled()` 方法接受一个 Promise 列表，并返回一个新的 promise，该 promise 在所有输入 promise 都 settle（无论是 resolved 还是 rejected）之后才 resolve。

`Promise.allSettled([p1, p2, p3])` -> 假设我们发起 3 个 API 调用来取数据。同时假设 **p1** 需要 **3 秒**，**p2** 需要 **1 秒**，**p3** 需要 **2 秒**。

第一种场景，假设 3 个 promise 都成功。那么 Promise.allSettled 会花 **3 秒**，并给出结果 promise，值为 `[val1, val2, val3]`。它会等所有 promise 都完成，然后收集结果，以数组形式输出。

如果其中某个 promise 被 reject 会怎样？例如 `Promise.all([p1, p2, p3])`，这次 p2 在 1 秒后被 reject。于是 Promise.allSettled 仍会等待所有 promise 都 settle。所以在 3 秒后，结果是 `[val1, err, val3]`。

- 💡 `Promise.all()` -> 快速失败（Fail Fast）
- 💡 `Promise.allSettled()` -> 会等待，并给出累积结果

## 三、Promise.race()

> `Promise.race()` 静态方法接受一个 promise 列表作为可迭代对象，并返回一个新的 promise：只要其中有一个 promise fulfill 或 reject，它就立即以那个 promise 的值或原因 fulfill 或 reject。`Promise.race()` 这个名字意味着所有 promise 彼此赛跑，只有一个赢家，无论是 resolved 还是 rejected。

`Promise.race([p1, p2, p3])` -> 假设我们发起 3 个 API 调用来取数据。同时假设 **p1** 需要 **3 秒**，**p2** 需要 **1 秒**，**p3** 需要 **2 秒**。所以只要第一个 promise resolve 或 reject，它就会给出输出。

因此在顺利的场景下，Promise.race 会在 1 秒后输出 `val2`，因为 p2 最早 resolve。而如果它是失败的，Promise.race 同样会在 1 秒后给出输出，只不过这次是错误。

## 四、Promise.any()

> `Promise.any()` 方法接受一个 Promise 对象列表作为可迭代对象。如果可迭代对象中有一个 promise 被 fulfill，`Promise.any()` 就返回一个 promise，其 resolve 值是那个 fulfilled promise 的结果。

`Promise.any([p1, p2, p3])` -> 假设我们发起 3 个 API 调用来取数据。同时假设 **p1** 需要 **3 秒**，**p2** 需要 **1 秒**，**p3** 需要 **2 秒**。所以只要第一个 promise 成功，它就会给出输出。

在上面的情形中，如果 p2 被 reject 了，什么也不会发生，因为 Promise.any 只寻找成功，所以第一个成功出现时，它就是结果。

❓ 但如果所有 promise 都失败了会怎样？那么返回的结果会是聚合错误，即 `[err1, err2, err3]`。

## 五、代码示例

### Promise.all()

```js
// 📌 场景一

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 1000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P3 Success");
  }, 2000);
});

Promise.all([p1, p2, p3]).then((results) => {
  console.log(results); // ['P1 Success', 'P2 Success', 'P3 Success'] -> 耗时 3 秒
});
```

```js
// 📌 场景二

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P2 Fail");
  }, 1000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P3 Success");
  }, 2000);
});

Promise.all([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err)); // 1 秒后抛出错误，即 'P2 Fails'
```

### Promise.allSettled()

💡 这是所有 Promise API 中最安全的一个。

```js
const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 1000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.allSettled([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err));

// 这里它会等所有 promise 都被 resolve 或 reject，然后返回，
/*
    [
      {status: 'fulfilled', value: 'P1 Success'},
      {status: 'fulfilled', value: 'P2 Success'},
      {status: 'rejected', reason: 'P3 Fail'}
    ]
  */
```

### Promise.race()

```js
// 📌 场景一

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 1000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.race([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err));

// 只要第一个 promise 被 resolve 或 reject，它就会返回。
// 上例输出："P2 Success"
```

```js
// 📌 场景二

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 5000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.race([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err));

// 2 秒后输出："P3 Fail"
```

> [!note] settled 的含义
> 一旦 promise 被 settle，意味着 -> 拿到了结果。而且 settled 大体上分为两类：
> 1. resolve、success、fulfilled
> 2. reject、failure、rejected

### Promise.any()

```js
// 📌 场景一

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P1 Success");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 5000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.any([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err));

// 它会等待第一个 settle 的**成功**
// 上例中 p3 会最先 settle，但它是 rejected 的，所以它会继续等待，于是在第 3 秒打印 "P1 Success"
```

```js
// 📌 场景二

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P1 Fail");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("P2 Success");
  }, 5000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.any([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => console.error(err));

// 5 秒后：'P2 Success'
```

```js
// 📌 场景三

const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P1 Fail");
  }, 3000);
});
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P2 Fail");
  }, 5000);
});
const p3 = new Promise((resolve, reject) => {
  setTimeout(() => {
    reject("P3 Fail");
  }, 2000);
});

Promise.any([p1, p2, p3])
  .then((results) => console.log(results))
  .catch((err) => {
    console.error(err);
    console.error(err.errors); // ['P1 Fail', 'P2 Fail', 'P3 Fail']
  });

// 因为全部都被 reject，所以会输出「聚合错误」
// AggregateError: All promises were rejected
// 要拿到 AggregateError 数组，需要写 "err.errors"
```

## 六、四个 Promise API 对比

| API | 返回时机 | 成功场景 | 失败场景 |
| --- | --- | --- | --- |
| `Promise.all()` | 全部成功，或任一失败时立即返回（快速失败） | 3 秒后得到 `[val1, val2, val3]` | p2 在 1 秒后 reject，1 秒后立即抛出与 p2 相同的错误，不等其他 promise；p1、p3 不会被取消 |
| `Promise.allSettled()` | 等全部 settle（3 秒） | `[val1, val2, val3]` | `[val1, err, val3]` |
| `Promise.race()` | 第一个 promise settle 时 | 1 秒后得到 `val2` | 若首个是失败，1 秒后得到该错误 |
| `Promise.any()` | 第一个 promise fulfilled 时 | p2 即使 reject 也不影响，3 秒后得到 `val1` | 全部失败时抛出 `AggregateError`，含 `[err1, err2, err3]` |

注意：`Promise.allSettled()` 实际返回的是对象数组，形如 `{status: 'fulfilled', value: ...}`，详见上面的代码示例。

## 七、总结：Promise 的 6 个静态方法

Promise 类共有 6 个静态方法：

- **Promise.all(promises)** – 等待所有 promise resolve，并返回它们结果的数组。如果其中任意一个 promise reject，它就变成 Promise.all 的错误，其他所有结果都会被忽略。
- **Promise.allSettled(promises)**（较新加入的方法）– 等待所有 promise settle，并返回它们的结果，即一个对象数组，每个对象包含：
  - `status`："fulfilled" 或 "rejected"
  - `value`（如果 fulfilled）或 `reason`（如果 rejected）
- **Promise.race(promises)** – 等待第一个 promise settle，它的结果/错误就成为最终结果。
- **Promise.any(promises)**（较新加入的方法）– 等待第一个 promise fulfill，它的结果成为最终结果。如果所有给定 promise 都被 reject，则 `AggregateError` 成为 Promise.any 的错误。
- **Promise.resolve(value)** – 用给定值创建一个已 resolve 的 promise。
- **Promise.reject(error)** – 用给定错误创建一个已 reject 的 promise。
- 在所有这些当中，`Promise.all` 大概是实践中最常用的。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=DlTVt1rZjIo&list=PLlasXeu85E9eWOpw9jxHOQyGMRiBZ60aX&index=4&ab_channel=AkshaySaini)
