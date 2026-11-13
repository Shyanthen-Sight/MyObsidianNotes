---
title: "05 · 最短的 JS 程序、window 与 this"
aliases:
  - Shortest JS Program, window & this keyword
tags:
  - JavaScript
  - Namaste-JavaScript
  - 全局对象
episode: 5
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 5 讲：最短的 JS 程序、window 与 this 关键字（Shortest JS Program, window & this keyword）

> [!abstract] 本讲要点
> - 最短的 JS 程序是一个空文件，但 JS 引擎依然会创建全局执行上下文。
> - JS 引擎会在全局空间创建一个 window 对象和一个 this 关键字。
> - 在全局层级，this 指向 window（`this === window`）。
> - 在全局作用域声明的变量会挂到全局对象上。

## 一、最短的 JS 程序

最短的 JS 程序就是一个空文件。因为即便如此，JS 引擎仍然做了大量工作：和往常一样，它仍会创建 GEC（全局执行上下文），其中包含内存空间与执行上下文。

## 二、window 对象与 this 关键字

JS 引擎会创建一种称为 **window** 的东西。它是一个对象，创建在全局空间（Global Space）中，包含大量函数和变量，这些函数和变量可以从程序的任何地方访问。

JS 引擎还会创建一个 **this** 关键字，在全局层级它指向 **window 对象**。总结来说，除了 GEC 之外，还会创建一个全局对象（window）和一个 this 变量。

> [!note] 核心结论
> 全局执行上下文创建时，同时会创建全局对象 window 和 this，并且在全局层级 `this === window`。

## 三、不同引擎中的全局对象

不同引擎中全局对象的名称会有所不同：在浏览器中叫 window，而在 Node.js 中则叫别的名字。在全局层级，`this === window`。

## 四、全局变量与全局对象

如果在全局作用域中创建任何变量，这些变量会被挂到全局对象上。

例如：

```js
var x = 10;
console.log(x); // 10
console.log(this.x); // 10
console.log(window.x); // 10
```

> [!tip] 记忆要点
> 全局变量并不是“凭空存在”的，它会成为全局对象上的一个属性，所以 `x`、`this.x` 和 `window.x` 指向同一个值。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=QCRpVw2KXf8&ab_channel=AkshaySaini)
