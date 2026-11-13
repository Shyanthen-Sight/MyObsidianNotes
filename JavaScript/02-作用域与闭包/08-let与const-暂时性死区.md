---
title: "08 · let、const 与暂时性死区"
aliases:
  - let & const in JS, Temporal Dead Zone
tags:
  - JavaScript
  - Namaste-JavaScript
  - 变量提升
episode: 8
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 8 讲：let、const 与暂时性死区（let & const in JS, Temporal Dead Zone）

> [!abstract] 本讲要点
> - let 和 const 也会被提升，但与 var 不同：它们被存放在独立的 script 内存空间中。
> - 从 let 变量被提升到被初始化赋值之间的时间称为暂时性死区（Temporal Dead Zone，TDZ）。
> - 在暂时性死区中访问变量会抛出 ReferenceError；重复声明等语法问题会直接抛出 SyntaxError。
> - const 比 let 更严格：声明时必须初始化，且不能重新赋值（TypeError）。

## 一、let 与 const 的提升

- let 和 const 声明也会被提升，但与 **var** 不同。

  ```js
  console.log(a); // ReferenceError: Cannot access 'a' before initialization
  console.log(b); // 如预期打印 undefined
  let a = 10;
  console.log(a); // 10
  var b = 15;
  console.log(window.a); // undefined
  console.log(window.b); // 15
  ```

- 看起来 let 没有被提升，**但它确实被提升了**，下面来解释。

  - 在提升阶段，a 和 b 实际上都被初始化为 _undefined_。但 var 声明的 **b** 位于全局（GLOBAL）的存储空间中，而 **a** 位于一个名为 script 的独立内存对象中，只有先给它赋值才能访问它，也就是说，只有先给 a 赋值才能访问 a。因此会抛出错误。

## 二、暂时性死区

- **暂时性死区**（Temporal Dead Zone，TDZ）：从 let 变量被提升，到它被初始化某个值之间的这段时间。

  - 因此，在 `let a = 10` 之前的任何一行都属于 a 的暂时性死区。
  - 由于 a 在全局上不可访问，它在 _window/this_ 上同样不可访问。`window.b` 或 `this.b` -> 15；但 `window.a` 或 `this.a` -> undefined，就像 `window.x` -> undefined 一样（x 根本没有在任何地方声明）。

- 当变量处于暂时性死区时，会抛出 **ReferenceError**（引用错误）。

- **SyntaxError**（语法错误）甚至不会让我们运行任何一行代码。

```js
let a = 10;
let a = 100;  // 这段代码会作为 SyntaxError 被提前拒绝。（重复声明）
------------------
let a = 10;
var a = 100; // 这段代码同样会被提前拒绝为 SyntaxError。（同一作用域中不能使用相同名称）
```

## 三、let 与 const 的严格程度

- **let** 是 **var** 的更严格版本。而 **const** 比 **let** 还要更严格。

  ```js
  let a;
  a = 10;
  console.log(a) // 10。注意，a 的声明和赋值在不同行。
  ------------------
  const b;
  b = 10;
  console.log(b); // SyntaxError: Missing initializer in const declaration。（const 不能这样声明，只有 const b = 10 才行）
  ------------------
  const b = 100;
  b = 1000; // 这会产生 TypeError: Assignment to constant variable.
  ```

## 四、错误的类型

- **错误**的类型有：Syntax（语法）、Reference（引用）和 Type（类型）。

  - `Uncaught ReferenceError: x is not defined at ...`

    - 该错误表示 x 从未出现在程序的作用域中。也就是说，x 从未被定义或声明，却试图被访问。

  - `Uncaught ReferenceError: cannot access 'a' before initialization`

    - 该错误表示无法访问 a，因为它以 let 声明，而且尚未被赋值，正处于暂时性死区，因此会报错。

  - `Uncaught SyntaxError: Identifier 'a' has already been declared`

    - 该错误表示我们重复声明了一个用 let 声明的变量。代码不会执行。

  - `Uncaught SyntaxError: Missing initializer in const declaration`

    - 该错误表示我们没有为 const 声明初始化或赋值。

  - `Uncaught TypeError: Assignment to constant variable`

    - 该错误表示我们正在给 const 变量重新赋值。

## 五、var、let、const 对比

> [!abstract] 对比小结
> | 维度 | var | let | const |
> | --- | --- | --- | --- |
> | 提升 | 提升并初始化为 undefined | 提升但不初始化 | 提升但不初始化 |
> | 存储位置 | 全局（GLOBAL / window） | script 独立内存空间 | script 独立内存空间 |
> | 暂时性死区 | 无 | 有 | 有 |
> | 重复声明 | 允许 | SyntaxError | SyntaxError |
> | 声明时初始化 | 不要求 | 不要求 | 必须 |
> | 重新赋值 | 允许 | 允许 | TypeError |

## 六、一些良好实践

- 尽可能使用 const。
- 如果不行，使用 let，避免 var。
- 把所有变量都用 let 声明并初始化，放在最顶部，以避免错误，从而把暂时性死区的窗口缩小到零。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=BNC6slYCj50&ab_channel=AkshaySaini)
