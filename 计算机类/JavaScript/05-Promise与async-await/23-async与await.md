---
title: "23 · async 与 await"
aliases:
  - async await
  - async-await
tags:
  - JavaScript
  - Namaste-JavaScript
  - 异步编程
episode: 23
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 23 讲：async 与 await（async await）

> [!abstract] 本讲要点
> - `async` 关键字用于声明异步函数，async 函数总是返回 Promise。
> - `await` 只能用在 async 函数内部，用于等待 Promise 的结果。
> - `await` 会暂停 async 函数的执行，但不会阻塞调用栈。
> - Promise 在创建时就开始执行，`await` 只是等待一个已经在运行的 Promise。
> - 用 `try-catch` 处理 async-await 中的错误；它本质上是 Promise 的语法糖。

## 一、本讲内容

- 什么是 async？
- 什么是 await？
- async 与 await 在背后是如何工作的？
- 使用 async/await 的示例
- 错误处理
- 面试题
- async await 与 Promise.then/.catch 的对比

## 二、什么是 async？

问：什么是 async？
答：async 是一个关键字，用在函数前面，用来创建异步函数。

问：什么是 async 函数，它和普通函数有什么区别？

```js
// 💡 async 函数总是返回一个 promise，即使我从下面的函数返回一个普通字符串，async 关键字也会用 Promise 把它包起来再返回。
async function getData() {
  return "Namaste JavaScript";
}
const dataPromise = getData();
console.log(dataPromise); // Promise {<fulfilled>: 'Namaste JavaScript'}

//❓如何从上面的 promise 中取出数据？一种方式是使用 promise 的 .then
dataPromise.then((res) => console.log(res)); // Namaste JavaScript
```

另一个例子：`async` 函数返回一个 Promise。

```js
const p = new Promise((resolve, reject) => {
  resolve("Promise resolved value!!");
});

async function getData() {
  return p;
}
// 在上面的情况下，因为我们已经返回了一个 promise，async 函数会直接返回它，而不会再用新的 Promise 包装一次。
const dataPromise = getData();
console.log(dataPromise); // Promise {<fulfilled>: 'Promise resolved value!!'}
dataPromise.then((res) => console.log(res)); // Promise resolved value!!
```

## 三、什么是 await？

问：如何在 async 函数中使用 `await`？
答：`async` 与 `await` 组合起来用于处理 promise。

但问题是：以前我们是怎么处理 promise 的，为什么还需要 async/await？

```js
const p = new Promise((resolve, reject) => {
  resolve("Promise resolved value!!");
});

function getData() {
  p.then((res) => console.log(res));
}

getData(); // Promise resolved value!!

//📌 到目前为止，我们一直在用 Promise.then/.catch 处理 promise。
// 现在来看看 async await 能帮我们做什么，它有什么不同

// 规则是：我们必须在 promise 前面使用 await 关键字。
async function handlePromise() {
  const val = await p;
  console.log(val);
}
handlePromise(); // Promise resolved value!!
```

📌 `await` 是一个只能在 `async` 函数内部使用的关键字。

```js
await function () {}; // 语法错误：await 只在 async 函数下有效。
```

> [!warning] await 的使用范围
> `await` 只能写在 `async` 函数内。在普通函数或全局作用域中直接使用会报语法错误。

## 四、async-await 特殊在哪里？

问：`async`-`await` 有什么特别之处？
答：我们用一个例子来对比：分别用 async-await 和旧的 .then/.catch 方式解决同一个 promise。为此我们会修改一下 promise `p`。

```js
const p = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value!!");
  }, 3000);
});

// 现在做一些修改再对比：

// 📌 Promise.then/.catch 方式
function getData() {
  // JS 引擎不会等待 promise 被 resolve
  p.then((res) => console.log(res));
  console.log("Hello There!");
}

getData(); // 先打印 `Hello There!`，3 秒后再打印 'Promise resolved value!!'
// 上面之所以如此，是因为 JavaScript 不等任何人：它会注册这个 promise，把回调函数单独注册，然后继续执行下面的 console，等 promise 被 resolve 后，再打印后续的 console。

//❓ 问题：人们通常会误以为 JS 会等 promise 被 resolve 后才执行后面的行。

// 📌 async-await 方式：
async function handlePromise() {
  // JS 引擎会等待 promise 被 resolve。
  const val = await p;
  console.log("Hello There!");
  console.log(val);
}
handlePromise(); // 这一次 `Hello There!` 不会立刻打印，而是 3 秒后先打印 `Hello There!`，再打印 'Promise resolved value!!'
// 💡 也就是说，代码在 `await` 那一行等待 promise resolve，然后才继续下一行。

// 以上是 Promise.then/.catch 与 async-await 的主要区别

//🤓 再深入想一想 async-await
async function handlePromise() {
  console.log("Hi");
  const val = await p;
  console.log("Hello There!");
  console.log(val);

  const val2 = await p;
  console.log("Hello There! 2");
  console.log(val2);
}
handlePromise();
// 在上面的代码中，程序会等两次，还是会并行执行？
//📌 `Hi` 立即打印 -> 然后代码等待 3 秒 -> 3 秒后两个 promise 都已 resolve，于是（'Hello There!' 'Promise resolved value!!' 'Hello There! 2' 'Promise resolved value!!'）会立即一起打印。

// 再创建两个不同的 promise，然后依次 resolve。
const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value by p2!!");
  }, 2000);
});

async function handlePromise() {
  console.log("Hi");
  const val = await p;
  console.log("Hello There!");
  console.log(val);

  const val2 = await p2;
  console.log("Hello There! 2");
  console.log(val2);
}
handlePromise();
// 📌 `Hi` 立即打印 -> 代码等待 3 秒 -> 3 秒后两个 promise 都已 resolve，于是（'Hello There!' 'Promise resolved value!!' 'Hello There! 2' 'Promise resolved value by p2!!'）会立即一起打印。所以尽管 `p2` 在 2 秒后就 resolve 了，它仍然必须等 `p` 先 resolve。

// 现在把 promise 的执行顺序反过来，观察结果。
async function handlePromise() {
  console.log("Hi");
  const val = await p2;
  console.log("Hello There!");
  console.log(val);

  const val2 = await p;
  console.log("Hello There! 2");
  console.log(val2);
}
handlePromise();
// 📌 `Hi` 立即打印 -> 代码等待 2 秒 -> 2 秒后（'Hello There!' 'Promise resolved value by p2!!'）打印；随后的一秒，也就是第 3 秒，（'Hello There! 2' 'Promise resolved value!!'）打印。
```

## 五、背后发生了什么？

问：程序是真的在等待吗，背后到底发生了什么？
答：我们知道，时间、潮水与 JS 都不等任何人，这是真的。这里看起来像是 JS 引擎在等待，但实际上 JS 引擎并没有在等。它没有占用调用栈——如果占用了，页面可能就卡死了。所以 JS 引擎并没有在等。那么如果它没在等，背后在做什么？我们用下面的代码片段来理解。

```js
const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value by p1!!");
  }, 5000);
});

const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value by p2!!");
  }, 10000);
});

async function handlePromise() {
  console.log("Hi");
  debugger;
  const val = await p1;
  console.log("Hello There!");
  debugger;
  console.log(val);

  const val2 = await p2;
  console.log("Hello There! 2");
  debugger;
  console.log(val2);
}
handlePromise();
// 当这个函数执行时，它会一行一行往下走，因为 JS 是同步单线程语言。让我们观察调用栈里发生了什么。上面我们设置了断点。

// 调用栈流程 -> handlePromise() 被压入栈 -> 把 `Hi` 打印到控制台 -> 接着它看到 await，这里 promise 应该被 resolve -> 那么它会等待 promise resolve 并阻塞调用栈吗？不会 -> 于是 handlePromise() 的执行被挂起并移出调用栈 -> 所以当 JS 看到 await 关键字时，它会暂停函数执行，直到 promise 被 resolve -> `p1` 会在 5 秒后 resolve，所以 5 秒后 handlePromise() 会再次被压入调用栈 -> 但这次它会从上次中断的地方继续执行 -> 现在它打印 'Hello There!' 和 'Promise resolved value by p1!!' -> 然后检查 `p2` 是否已经 resolve -> 它发现 `p2` 需要 10 秒才能 resolve，所以上述过程会重复 -> 执行会被挂起，直到 promise 被 resolve。

// 📌 因此 JS 并没有在等待，调用栈也没有被阻塞。

// 另外，在上面的场景中，如果 p1 需要 10 秒、p2 需要 5 秒会怎样？即使 p2 更早 resolve，但 JS 是同步单线程语言，所以它会先等 p1 resolve，然后立刻执行完剩下的全部。
```

## 六、面试输出题

问：解释代码的输出及其原因。

```js
const p1 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value by p1!!");
  }, 10000);
});

const p2 = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve("Promise resolved value by p2!!");
  }, 5000);
});

async function handlePromise() {
  console.log("Hi");
  debugger;
  const val = await p1;
  console.log("Hello There!");
  debugger;
  console.log(val);

  const val2 = await p2;
  console.log("Hello There! 2");
  debugger;
  console.log(val2);
}
handlePromise();

// 先打印 Hi，然后因为 await p1，会等 10 秒并打印 `Promise resolved value by p1!!`，紧接着打印 `Hello There!`，然后立刻打印 `Promise resolved value by p2!!`，紧接着打印 `Hello There! 2`

// ❓ 为什么 p2 没有再多等 5 秒，而是立刻就执行了？
```

答：

- Promise 在被创建时就开始执行，而不是在被 await 时才开始执行。
- p1 的 `setTimeout(10s)` 立刻开始计时。
- p2 的 `setTimeout(5s)` 也立刻开始计时。
- 两个定时器在文件执行时就开始计时了，甚至在 `handlePromise()` 遇到第一个 `await` 之前就已经开始。
- `await` 不会启动一个 promise，它只是等待一个已经在运行的 promise。

## 七、如何让 p2 在 p1 之后执行

问：如何让 p2 在 p1 之后再开始？
答：

```js
function createP1() {
  return new Promise((resolve) =>
    setTimeout(() => resolve("p1 resolved"), 10000),
  );
}

function createP2() {
  return new Promise((resolve) =>
    setTimeout(() => resolve("p2 resolved"), 5000),
  );
}

async function handlePromise() {
  console.log("Hi");

  const val1 = await createP1();
  console.log(val1);

  const val2 = await createP2();
  console.log(val2);
}
```

## 八、真实场景示例

```js
async function handlePromise() {
  // fetch() => Response 对象，其 body 是 Readable stream => Response.json() 也是一个 promise，resolve 后得到值
  const data = await fetch("https://api.github.com/users/alok722");
  const res = await data.json();
  console.log(res);
}
handlePromise();
```

## 九、错误处理

使用普通 Promise 时，我们用 .catch 处理错误；而在 `async-await` 中，我们使用 `try-catch` 代码块来处理错误。

```js
async function handlePromise() {
  try {
    const data = await fetch("https://api.github.com/users/alok722");
    const res = await data.json();
    console.log(res);
  } catch (err) {
    console.log(err);
  }
}
handlePromise();

// 在上面代码中，只要发生错误，执行就会进入 catch 块。你可以用错误的 URL 试一试，会得到错误。

// 另一种处理错误的方式：
handlePromise().catch((err) => console.log(err)); // 这样也可以，因为失败时 handlePromise 会返回一个 reject 的 promise。
```

## 十、async-await 与 Promise.then/.catch 的对比

应该用哪一个？`async-await` 只是 promise 的语法糖。在背后，`async-await` 就是 promise。所以两者本质相同，只是 `async-await` 是更新的写法。`async-await` 解决了 Promise 的一些短板，比如 `Promise 链`。`async-await` 还提升了可读性。所以就某种程度上说，始终建议使用 `async-await`。

> [!abstract] 本讲小结
> - async 函数总是返回 Promise；await 只能出现在 async 函数内。
> - await 会挂起函数执行、把函数移出调用栈，而不是阻塞调用栈。
> - Promise 在创建时即开始执行，await 只是等待已有 promise 的结果。
> - 错误处理用 try-catch，或在外层调用 `.catch`。
> - async-await 是 Promise 的语法糖，可读性更好。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=6nv3qy3oNkc&list=PLlasXeu85E9eWOpw9jxHOQyGMRiBZ60aX&index=4&ab_channel=AkshaySaini)
