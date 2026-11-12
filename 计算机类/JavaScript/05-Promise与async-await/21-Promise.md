---
title: "21 · Promise"
aliases:
  - Promises
  - Promise
tags:
  - JavaScript
  - Namaste-JavaScript
  - 异步编程
episode: 21
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 21 讲：Promise（Promises）

> [!abstract] 本讲要点
> - Promise 用于处理 JavaScript 中的异步操作。
> - 可以把 Promise 看作一个占位符：内部数据初始为 `undefined`，异步操作完成后会自动填入。
> - 通过 `then` 把回调「附加」到 Promise 对象上；Promise 保证只回调一次。
> - 真实的 Promise 对象有 `prototype`、`promiseState`、`promiseResult`，状态分为 pending、fulfilled、rejected。
> - Promise 对象是不可变的；Promise 链可以解决回调地狱。

> Promise 用于处理 JavaScript 中的异步操作。

我们会用代码示例来讨论：在 `Promise` 之前事情是怎么做的，有了 `Promise` 之后又是怎么做的。

假设以电商为例：

```js
const cart = ["shoes", "pants", "kurta"];

// 下面两个函数是异步的，并且彼此依赖
const orderId = createOrder(cart);
proceedToPayment(orderId);

// 使用回调（Promise 之前）
// 这里由 createOrder 函数负责先创建订单，然后调用回调函数
createOrder(cart, function () {
  proceedToPayment(orderId);
});
// 上面存在「控制反转」的问题
```

问：如何解决上面的问题？
_答：使用 Promise。_

## 一、Promise 的本质

现在，我们让 `createOrder` 函数返回一个 promise，并把这个 `promise` 保存到一个 `变量` 里。

Promise 没什么神奇的，可以把它想象成一个内部带有某个数据值的空对象，而这个数据值会保存 `createOrder` 函数最终返回的内容。

由于 `createOrder` 是异步函数，我们不知道它需要多久才能执行完。

所以 `createOrder` 一执行，就会返回给你一个 `undefined` 值。假设 5 秒后执行完成，此时 `orderId` 就绪了，于是它会用 `orderId` 填上那个 `undefined` 值。

简而言之，当 `createOrder` 执行时，它会立刻返回一个值为 `undefined` 的 `promise 对象`，然后 JavaScript 继续执行后面的代码。过一段时间，`createOrder` 执行完毕、`orderId` 就绪，这个值就会`自动`赋给我们返回的那个、之前还是 `undefined` 的 `promise`。

问：问题是我们怎么知道 `response` 已经就绪了？
_答：我们会用 `then` 给 `promise 对象`附加一个 `回调` 函数，当 `result` 就绪时它会自动触发。_

```js
const cart = ["shoes", "pants", "kurta"];

const promiseRef = createOrder(cart);
// 这个 promiseRef 拥有 `then` 方法

// {data: undefined}
// 一开始它是 undefined，所以下面这段代码不会触发
// 过一段时间，执行完成、promiseRef 有了数据之后，下面这行会自动触发。

promiseRef.then(function () {
  proceedToPayment(orderId);
});
```

## 二、传递函数与附加函数

问：它比回调方式好在哪里？

在之前的方案里，我们习惯把函数传进去，然后信任那个函数会去执行回调。

但使用 promise 时，我们是在给一个 promise 对象附加回调函数。

「传递一个函数」和「附加一个函数」，这两种说法是有区别的。

Promise 保证：一旦它有了已兑现（fulfilled）的数据，就会回调那个被附加的函数。而且它只会调用一次。就一次。

## 三、真实的 Promise 对象

前面我们说过 promise 是一个数据为空的对象，但这并不完全正确，`Promise` 远比这复杂。

现在我们来看看一个真实的 promise 对象。

fetch 是一个 Web API，用来发起 API 调用，它返回一个 promise。

我们会调用公开的 GitHub API 来获取数据：
https://api.github.com/users/alok722

```js
// 我们会调用公开的 GitHub API 来获取数据
const URL = "https://api.github.com/users/alok722";
const user = fetch(URL);
// 上面的 user 会是一个 promise。
console.log(user); // Promise {<Pending>}

/** 观察：
 * 如果深入查看，这个 `promise` 对象包含 3 样东西：
 * `prototype`、`promiseState` 和 `promiseResult`
 * 而这个 `promiseResult` 就是我们之前所说的 data
 * 一开始 `promiseResult` 是 `undefined`
 *
 * `promiseResult` 会保存 API 调用返回的数据
 * `promiseState` 会告诉你 promise 当前处于哪个状态，一开始是 `pending` 状态，之后会变成 `fulfilled`
 */

/**
 * 上面那行执行时，`fetch` 发起 API 调用并立刻返回一个处于 `Pending` 状态的 `promise`，JavaScript 不会等它变成 `fulfilled`
 * 然后在下一行把 `pending 状态的 promise` 打印出来。
 * 注意：Chrome 浏览器有一些不一致的行为——打印的那一刻它显示为 pending，但如果你展开它，会看到 fulfilled，因为 promise 兑现时 Chrome 更新了那条日志。
 * 一旦兑现，数据就在 promiseResult 中，它位于 body 内、是 ReadableStream 格式，有办法把数据提取出来。
 */
```

现在我们可以给上面的响应附加回调吗？

使用 `.then`：

```js
const URL = "https://api.github.com/users/alok722";
const user = fetch(URL);

user.then(function (data) {
  console.log(data);
});
// 这就是 Promise 的用法。
// 它保证只会被解决（resolved）一次，要么 `成功`，要么 `失败`
/**
    Promise 处于以下状态之一：

    pending：初始状态，既未 fulfilled 也未 rejected。
    fulfilled：表示操作成功完成。
    rejected：表示操作失败。
 */
```

## 四、Promise 的不可变性

💡 Promise 对象是不可变的。

因此，一旦 promise 兑现、我们拿到了数据，就可以把这个数据传来传去，而不必担心有人会修改它。所以在上面的例子中，我们不能直接修改 `user` 这个 promise 对象，必须通过 `.then` 来使用它。

## 五、面试指南：什么是 Promise

> [!question] 什么是 Promise？
> - Promise 对象是在我们从异步操作拿到值之前的一段时间里的占位符。
> - 一个未来值的容器。
> - **Promise 是一个对象，表示一个异步操作的最终完成或失败。**

## 六、用 Promise 链解决回调地狱

我们现在已经解决了回调的一个问题，也就是控制反转。

但还有一个问题：回调地狱……

```js
// 回调地狱示例
createOrder(cart, function (orderId) {
  proceedToPayment(orderId, function (paymentInf) {
    showOrderSummary(paymentInf, function (balance) {
      updateWalletBalance(balance);
    });
  });
});
// 上面这段代码横向不断扩张，这就是所谓的厄运金字塔。
// 回调地狱既不美观，也难以维护。

// 💡 Promise 用 `Promise 链` 也解决了这个问题
// 下面的例子就是 Promise 链
createOrder(cart)
  .then(function (orderId) {
    proceedToPayment(orderId);
  })
  .then(function (paymentInf) {
    showOrderSummary(paymentInf);
  })
  .then(function (balance) {
    updateWalletBalance(balance);
  });

// ⚠️ 常见坑
// 我们在 Promise 链中忘记 return promise
// 关键在于：一个 .then 返回的 promise/数据，会成为下一个 .then 的数据
// 所以应该这样写：
createOrder(cart)
  .then(function (orderId) {
    return proceedToPayment(orderId);
  })
  .then(function (paymentInf) {
    return showOrderSummary(paymentInf);
  })
  .then(function (balance) {
    return updateWalletBalance(balance);
  });

// 为了提高可读性，可以用箭头函数代替普通函数
```

> [!warning] 忘记 return 就会断链
> 在 Promise 链中，上一个 `.then` 里返回的 promise 或数据，才会成为下一个 `.then` 的输入。如果忘了 `return`，下一个 `.then` 拿到的是 `undefined`，链就断了。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=ap-6PPAuK1Y&list=PLlasXeu85E9eWOpw9jxHOQyGMRiBZ60aX&index=3&ab_channel=AkshaySaini)
