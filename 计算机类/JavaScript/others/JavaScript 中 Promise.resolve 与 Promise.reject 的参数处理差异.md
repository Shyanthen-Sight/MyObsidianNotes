---
aliases:
  - Promise.resolve vs Promise.reject
tags:
  - JavaScript
  - Promise
  - 异步编程
---

# Promise.resolve 与 Promise.reject 的参数处理差异

> 一句话：`Promise.resolve(promise)` 会**采用**传入 Promise 的状态（穿透），而 `Promise.reject(reason)` **永远不解包**，传进去的 Promise 会被原样当作拒绝理由。

## 一、结论先行

| 调用 | 传入的是 Promise 时 | 最终状态 | `.catch()` 拿到的值 |
| --- | --- | --- | --- |
| `Promise.resolve(p)` | 不包裹，直接返回 `p`（采用其状态） | 与 `p` 相同 | `p` 的拒绝原因 |
| `Promise.reject(p)` | 不解析，`p` 整体作为 reason | 恒为 rejected | `p` 这个 Promise 对象本身 |

一句话记忆：**resolve 会拆包，reject 不拆包。**

## 二、示例 1：`Promise.resolve(Promise.reject(1))`

```js
Promise.resolve(Promise.reject(1))
  .then((value) => console.log("Resolved:", value))
  .catch((error) => console.error("Rejected:", error));
// Rejected: 1
```

执行过程：

1. `Promise.reject(1)` 创建一个 rejected 的 Promise，reason 为 `1`。
2. `Promise.resolve()` 收到的是 Promise（thenable），于是直接返回它，不新建包裹层。
3. 状态是 rejected，`.then` 被跳过，直接进入 `.catch`。
4. `error` 就是最初的值 `1`。

等价写法：`Promise.reject(1).then(...).catch(...)`。

## 三、示例 2：`Promise.reject(Promise.resolve(1))`

```js
Promise.reject(Promise.resolve(1))
  .then((value) => console.log("Resolved:", value))
  .catch((error) => console.error("Rejected:", error));
// Rejected: Promise { 1 }
// 浏览器控制台可能显示为 Promise {<fulfilled>: 1}
```

执行过程：

1. `Promise.resolve(1)` 是 fulfilled 的 Promise，值为 `1`。
2. `Promise.reject()` 无论收到什么，都新建一个 rejected 的 Promise，并且**不判断参数是不是 thenable**。
3. 于是整个 Promise 对象成了 reason；`.catch` 拿到的是一个 Promise，而不是 `1`。

想拿到里面的值，需要再 `await` 一次：

```js
Promise.reject(Promise.resolve(1)).catch(async (error) => {
  console.log(await error); // 1
});
```

## 四、为什么会这样

规范里两者并不对称：

- `Promise.resolve(x)`：当 `x` 是 Promise 且构造器一致时，直接**原样返回**（identity），不额外包一层；是 thenable 时则新建 Promise 并调用其 `then`，从而跟随它的状态。
- `Promise.reject(r)`：没有任何特殊处理，等价于 `new Promise((_, reject) => reject(r))`，`r` 被当作普通值存进 reason。

所以 `Promise.resolve` 的职责是「把值或承诺统一成 Promise」，`Promise.reject` 的职责只是「造一个以 `r` 为理由的失败承诺」。

## 五、延伸与常见坑

### 5.1 `Promise.resolve(Promise.resolve(1))`

fulfilled 的 Promise 再被 `Promise.resolve` 处理，仍然是同一个 Promise，结果 `1`。

```js
Promise.resolve(Promise.resolve(1)).then(console.log); // 1
```

### 5.2 thenable 也一样会被采用

任何带 `then` 方法的对象都会被 `Promise.resolve` 展开：

```js
const thenable = {
  then(resolve) {
    resolve("ok");
  }
};

Promise.resolve(thenable).then(console.log); // ok
```

### 5.3 用 `Promise.reject` 传递 Promise 通常是 bug

如果想把上游的失败继续往下传，直接 `return promise` 或 `throw promise` 即可；把 Promise 塞进 `Promise.reject()` 会让下游拿到一个 Promise 对象，后续 `error.message` 之类的判断全部失效。

### 5.4 拒绝理由里的 Promise 不会产生额外 unhandled rejection

示例 2 里内层 Promise 是 fulfilled 的，被当作 reason 时不构成未处理拒绝。但如果把 rejected 的 Promise 当作 reason，调试信息会变得很难读：

```js
// 不推荐：reason 本身是一个 rejected Promise
Promise.reject(Promise.reject(new Error("inner")));
```

## 六、速查表

| 场景 | 写法 | 结果 |
| --- | --- | --- |
| 普通值 → fulfilled | `Promise.resolve(1)` | fulfilled，值为 `1` |
| Promise → 采用其状态 | `Promise.resolve(p)` | 同 `p` |
| thenable → 采用其状态 | `Promise.resolve(thenable)` | 跟随 `then` |
| 任意值 → rejected | `Promise.reject(v)` | rejected，reason 为 `v` |
| Promise → rejected | `Promise.reject(p)` | rejected，reason 是 `p` 本身 |

> 口诀：**resolve 看状态，reject 只看参数。**

## 相关笔记

- [[计算机类/JavaScript/05-Promise与async-await/21-Promise.md|21 · Promise]]
- [[计算机类/JavaScript/05-Promise与async-await/22-创建Promise-链式调用与错误处理.md|22 · 创建 Promise、链式调用与错误处理]]
- [[计算机类/JavaScript/05-Promise与async-await/23-async与await.md|23 · async 与 await]]
- [[计算机类/JavaScript/05-Promise与async-await/24-Promise-API.md|24 · Promise API]]
