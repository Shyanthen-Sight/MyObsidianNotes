---
title: "11 · setTimeout 与闭包面试题"
aliases:
  - setTimeout + Closures Interview Question
tags:
  - JavaScript
  - Namaste-JavaScript
  - 闭包
episode: 11
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 11 讲：setTimeout 与闭包面试题（setTimeout + Closures Interview Question）

> [!abstract] 本讲要点
> - JavaScript 不会等待 setTimeout，而是立即执行后面的代码。
> - setTimeout 中的回调函数与变量形成闭包，记住的是变量的引用而不是值。
> - 用 var 在循环中配合 setTimeout，会导致 5 次都打印循环结束后的最终值 6。
> - 解决办法是改用 let，或者用一个闭包函数为每次迭代创建新的变量副本。

> **时间、潮汐和 JavaScript 都不会等待任何人。**

## 一、setTimeout 的执行顺序

- ```js
  function x() {
    var i = 1;
    setTimeout(function () {
      console.log(i);
    }, 3000);
    console.log("Namaste Javascript");
  }
  x();
  // 输出：
  // Namaste Javascript
  // 1 // 等待 3 秒后
  ```

  - 我们期望 JS 等待 3 秒、打印 1，然后继续往下打印字符串。但 JS 会立刻打印字符串，等待 3 秒后才打印 1。
  - setTimeout 内部的函数形成了闭包（记住了对 i 的引用）。因此无论函数走到哪里，都会带着这个引用。
  - setTimeout 接收这个回调函数，为它附加 3000ms 的定时器并保存起来，然后不等待就继续执行下一行，打印字符串。
  - 3000ms 结束后，JS 取出这个函数，放入调用栈并执行。

## 二、经典面试题：按秒递增打印

- 问：1 秒后打印 1，2 秒后打印 2，直到 5 —— 一道棘手的面试题。

  我们以为这有一个简单的写法，如下所示：

  ```js
  function x() {
    for (var i = 1; i <= 5; i++) {
      setTimeout(function () {
        console.log(i);
      }, i * 1000);
    }
    console.log("Namaste Javascript");
  }
  x();
  // 输出：
  // Namaste Javascript
  // 6
  // 6
  // 6
  // 6
  // 6
  ```

  - 原因？

    - 这是由闭包造成的。当 setTimeout 把函数保存到某处并为其附加定时器时，函数记住的是对 i 的**引用**，而不是 i 的**值**。这 5 份函数都指向同一个 i 引用。JS 保存这 5 个函数，打印字符串，然后回头执行这些函数。此时定时器已经到点。而由于循环，i 的值变成了 6。当回调函数运行时，变量 i = 6，所以每次日志打印的都是同一个 6。

    - 要避免这个问题，可以用 **let** 代替 **var**，因为 let 具有块作用域。每次迭代中，i 都是一个全新的变量（i 的新副本）。每次运行 setTimeout 时，内部函数都会与新的变量 i 形成闭包。

  - 但如果面试官要求只能用 **var** 实现呢？

    ```js
    function x() {
      for (var i = 1; i <= 5; i++) {
        function close(i) {
          setTimeout(function () {
            console.log(i);
          }, i * 1000);
          // 把 setT 函数放进新函数 close() 中
        }
        close(i); // 每次调用 close(i) 都会创建 i 的新副本。只是这一次，用的仍然是 var！
      }
      console.log("Namaste Javascript");
    }
    x();
    ```

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=eBTBG4nda2A&ab_channel=AkshaySaini)
