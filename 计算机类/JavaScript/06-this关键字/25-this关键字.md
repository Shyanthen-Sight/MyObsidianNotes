---
title: "25 · this 关键字"
aliases:
  - This keyword in JavaScript
tags:
  - JavaScript
  - Namaste-JavaScript
  - this
episode: 25
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 25 讲:this 关键字(This Keyword in JavaScript)

> [!abstract] 本讲要点
> - `this` 指向哪个对象,取决于它被调用(使用)的方式,而不是定义的位置。
> - 全局空间中的 `this` 指向全局对象,浏览器中即 `window`。
> - 函数内部的 `this` 在严格模式下为 `undefined`,非严格模式下因「`this` 替换」而指向全局对象。
> - 对象方法中的 `this` 指向调用该方法的对象;可用 `call`、`apply`、`bind` 手动指定。
> - 箭头函数没有自己的 `this`,取外层词法上下文;DOM 中的 `this` 指向 HTML 元素。

> 在 JavaScript 中,`this` 关键字指向一个对象,而具体指向哪个对象,取决于 `this` 是如何被调用(使用或调用)的。

## 一、全局空间中的 this

任何在全局定义的内容,都被认为处于全局空间(global space)中。

```js
console.log(this); // 指向全局对象,在浏览器中即 window
// 💡 全局对象因运行环境而异
```

## 二、函数内部的 this

```js
function x() {
  // 下面的值取决于严格模式 / 非严格模式
  console.log(this);
  // 严格模式下 —— undefined
  // 非严格模式下 —— 指向全局的 window 对象
}
x();
// 💡 注意:

// 乍一看,全局空间中的 `this` 与函数内部的 `this` 表现相同,但实际上并非如此。

// 一旦在文件顶部使用 "use strict" 让 JS 进入严格模式,函数内部的 `this` 就会返回 `undefined`,而全局空间仍然指向全局的 window 对象
```

`this` 替换(this substitution):根据 `this` 替换规则,如果 `this` 关键字的值为 `null/undefined`,那么只有在非严格模式下才会被替换为 globalObject。这正是非严格模式下函数内部的 `this` 指向全局 window 对象的原因。

> [!note] 小结
> 函数内部 `this` 关键字的值本身是 `undefined`。由于非严格模式下存在 `this` 替换,`this` 会指向 `globalWindowObject`;而在严格模式下,它依然是 `undefined`。

`this` 关键字的值取决于 `function` 是如何被调用的。例如:
在严格模式下:

```js
x(); // undefined
window.x(); // 全局 window 对象
```

## 三、对象方法中的 this

```js
// 按术语来说,下面的 `x` 键就是一个方法
const obj = {
  a: 10,
  x: function () {
    console.log(this); // {a: 10, x: f()}
    console.log(this.a); // 10
  },
};
obj.x(); // `this` 的值指向当前对象,即 `obj`
```

## 四、call、apply 与 bind 方法

> 关于 call、apply 和 bind 方法的详细内容,请参阅[这里](https://www.youtube.com/watch?v=75W8UPQ5l7k&ab_channel=AkshaySaini)。

```js
const student = {
  name: "Alok",
  printName: function () {
    console.log(this.name);
  },
};
student.printName(); // Alok

const student2 = {
  name: "Kajal",
};
student2.printName(); // 抛出错误

// ❓ 如何复用 `student` 对象上的 printName 方法
student.printName.call(student2); // Kajal
// 上面的 `call` 方法接收的正是 `this` 关键字的值
// 因此在 `printName` 方法内部,`this` 的值现在就是 `student2` 对象

// 所以,call、bind 与 apply 都是用来设置 this 关键字的值的。
```

## 五、箭头函数中的 this

箭头函数没有自己的 `this` 值,它们从外层的词法上下文(lexical context)中取值。

```js
const obj = {
  a: 10,
  x: () => {
    console.log(this); // window 对象
    // 上面的 `this` 不再是 obj,而是外层词法上下文,在当前场景下即 window 对象。
  },
};
obj.x();

const obj2 = {
  a: 10,
  x: function () {
    const y = () => {
      console.log(this);
      // 上面的 `this` 会是 obj2,因为函数 y 的外层词法上下文是函数 `x`。
    };
    y();
  },
};
obj2.x();
```

## 六、DOM 中的 this

> 它指向 HTML 元素。

```html
<button onclick="alert(this)">Click Me</button>
<!-- [object HTMLButtonElement] Button 元素 -->
```

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=9T4z98JcHR0&list=PLlasXeu85E9eWOpw9jxHOQyGMRiBZ60aX&index=4&ab_channel=AkshaySaini)
