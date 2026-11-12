---
title: "22 · 创建 Promise、链式调用与错误处理"
aliases:
  - Creating a Promise
  - Promise Chaining
  - Error Handling
tags:
  - JavaScript
  - Namaste-JavaScript
  - 异步编程
episode: 22
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 22 讲：创建 Promise、链式调用与错误处理（Creating a Promise, Chaining & Error Handling）

> [!abstract] 本讲要点
> - Promise 分为消费者（Consumer）与生产者（Producer）两部分。
> - 用 `new Promise(function (resolve, reject) {...})` 创建 Promise，`resolve` 处理成功、`reject` 处理失败。
> - 尚未完成的 Promise 打印出来是 `Promise {<pending>}`，完成后 `.then` 才会执行。
> - 用 `.catch` 捕获 rejection（失败）。
> - Promise 链中，上一个 `.then` 的返回值成为下一个 `.then` 的数据；链中任一环节被 reject，执行就会跳到 `.catch`。

## 一、Promise 的消费者与生产者

```js
const cart = ["shoes", "pants", "kurta"];

// Promise 的消费者部分
const promise = createOrder(cart); // orderId
// 我们的预期是：上面的函数会返回一个 promise。

promise.then(function (orderId) {
  proceedToPayment(orderId);
});

// 上面这段代码我们在上一讲已经见过。
// 现在我们来看 `createOrder` 是如何实现的，从而让它返回一个 promise
// 简而言之，我们要看「如何创建一个 Promise」，然后把它返回。

// Promise 的生产者部分
function createOrder(cart) {
  // JS 提供了 Promise 构造函数，通过它可以创建 promise
  // 它接受一个回调函数，回调函数有两个参数 `resolve` 和 `reject`
  const promise = new Promise(function (resolve, reject) {
    // 这个 `resolve` 和 `reject` 是什么？
    // 它们是 JavaScript 传给我们的函数，用来处理函数调用的成功与失败。
    // 现在我们来实现 `createOrder` 的逻辑
    /** 模拟的步骤
     * 1. 校验购物车
     * 2. 写入数据库并拿到 orderId
     */
    // 我们假设在真实场景中，`validateCart` 是已经定义好的
    if (!validateCart(cart)) {
      // 如果购物车无效，就 reject 这个 promise
      const err = new Error("Cart is not Valid");
      reject(err);
    }
    const orderId = "12345"; // 我们通过调用数据库拿到了这个 id（假设）
    if (orderId) {
      // 成功场景
      resolve(orderId);
    }
  });
  return promise;
}
```

## 二、pending 状态

在上面的代码中，如果你的 `validateCart` 返回 true，那么上面这个 promise 就会被 resolve（成功）。

```js
const cart = ["shoes", "pants", "kurta"];

const promise = createOrder(cart); // orderId
// ❓ 下面这行会打印什么？
// 它会打印 Promise {<pending>}，为什么？
// 因为上面的 createOrder 需要一些时间才会被 resolve，所以处于 pending 状态。但一旦 promise 被 resolve，`.then` 就会作为回调被执行。
console.log(promise);

promise.then(function (orderId) {
  proceedToPayment(orderId);
});

function createOrder(cart) {
  const promise = new Promise(function (resolve, reject) {
    if (!validateCart(cart)) {
      const err = new Error("Cart is not Valid");
      reject(err);
    }
    const orderId = "12345";
    if (orderId) {
      resolve(orderId);
    }
  });
  return promise;
}
```

> [!note] 为什么打印的是 pending
> `createOrder` 内部的异步/延迟逻辑需要一点时间才完成。在 promise 被 resolve 之前，它的状态就是 `pending`；一旦 resolve，注册在 `.then` 里的回调才会执行。

## 三、用 catch 捕获错误

现在来看看，如果出现错误、我们 reject 了 promise，该如何捕获？

答案是使用 `.catch`。

```js
const cart = ["shoes", "pants", "kurta"];

const promise = createOrder(cart); // orderId

// 这里我们在消费 Promise，并尝试捕获 promise 的错误
promise
  .then(function (orderId) {
    // ✅ 成功，即 resolved promise 的处理
    proceedToPayment(orderId);
  })
  .catch(function (err) {
    // ⚠️ 失败，即 reject 的处理
    console.log(err);
  });

// 这里我们在创建 Promise
function createOrder(cart) {
  const promise = new Promise(function (resolve, reject) {
    // 假设下面的 `validateCart` 返回 false，那么 promise 就会被 reject
    // 然后浏览器就会抛出错误。
    if (!validateCart(cart)) {
      const err = new Error("Cart is not Valid");
      reject(err);
    }
    const orderId = "12345";
    if (orderId) {
      resolve(orderId);
    }
  });
  return promise;
}
```

## 四、Promise 链式调用（Promise Chaining）

现在来理解 Promise 链式调用的概念。

- 为此我们假设 `createOrder` 之后还需要调用 `proceedToPayment`。
- 在 Promise 链中，第一个 `.then` 返回的任何内容，都会成为下一个 `.then` 的数据，依此类推……
- 在 Promise 链的任意一点，如果 promise 被 reject，执行都会回退到 `.catch`，其他 promise 不会执行。

```js
const cart = ["shoes", "pants", "kurta"];

createOrder(cart)
  .then(function (orderId) {
    // ✅ 成功，即 resolved promise 的处理
    // 💡 我们返回数据或 promise，以便继续链式调用，这里返回的是数据
    console.log(orderId);
    return orderId;
  })
  .then(function (orderId) {
    // Promise 链
    // 💡 我们要确保 `proceedToPayment` 也返回一个 promise
    return proceedToPayment(orderId);
  })
  .then(function (paymentInfo) {
    // 由于上面 `proceedToPayment` 返回了一个 promise，所以可以用 `.then` 来消费
    console.log(paymentInfo);
  })
  .catch(function (err) {
    // ⚠️ 失败，即 reject 的处理
    console.log(err);
  });

// 这里我们在创建 Promise
function createOrder(cart) {
  const promise = new Promise(function (resolve, reject) {
    // 假设下面的 `validateCart` 返回 false，那么 promise 就会被 reject
    // 然后浏览器就会抛出错误。
    if (!validateCart(cart)) {
      const err = new Error("Cart is not Valid");
      reject(err);
    }
    const orderId = "12345";
    if (orderId) {
      resolve(orderId);
    }
  });
  return promise;
}

function proceedToPayment(cart) {
  return new Promise(function (resolve, reject) {
    // 眼下我们只是简单地 `resolve` 这个 promise
    resolve("Payment Successful");
  });
}
```

## 五、在链中的任意位置捕获错误

问：如果我希望即使某个 promise 失败，也继续执行后面的代码，该怎么做？

- 把 `.catch` 放在某个层级上，在这个层级之后我们就不再关心失败了。
- 也可以有多个 `.catch`。

例如：

```js
createOrder(cart)
  .then(function (orderId) {
    // ✅ 成功，即 resolved promise 的处理
    // 💡 我们返回数据或 promise，以便继续链式调用，这里返回的是数据
    console.log(orderId);
    return orderId;
  })
    .catch(function (err) {
    // ⚠️ 它下面无论什么失败，catch 都不关心
    // 这个代码块只对它上面的代码块负责
    console.log(err);
  });
  .then(function (orderId) {
    // Promise 链
    // 💡 我们要确保 `proceedToPayment` 也返回一个 promise
    return proceedToPayment(orderId);
  })
  .then(function (paymentInfo) {
    // 由于上面 `proceedToPayment` 返回了一个 promise，所以可以用 `.then` 来消费
    console.log(paymentInfo);
  })
```

> [!warning] catch 的位置决定了它能接住谁
> `.catch` 只负责它上层链条中的失败。把 `.catch` 放在某个位置后，它下面新接的 `.then` 不会受到前面失败的影响，代码可以继续往下执行。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=U74BJcr8NeQ&list=PLlasXeu85E9eWOpw9jxHOQyGMRiBZ60aX&index=4&ab_channel=AkshaySaini)
