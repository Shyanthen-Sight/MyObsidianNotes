---
title: "06 · undefined 与 not defined"
aliases:
  - undefined vs not defined in JS
tags:
  - JavaScript
  - Namaste-JavaScript
  - undefined
episode: 6
source: https://github.com/alok722/namaste-javascript-notes
---

# 第 6 讲：undefined 与 not defined（undefined vs not defined in JS）

> [!abstract] 本讲要点
> - 第一阶段（内存分配）中，JS 会为每个变量分配一个占位符，称为 undefined。
> - undefined：内存已分配，但尚未赋值。
> - not defined：变量在内存分配阶段根本没有被声明/找到，访问时才会出现。
> - 两者并不相等：Not Defined !== Undefined。

## 一、什么是 undefined

在第一个阶段（内存分配）中，JS 会为每个变量分配一个称为 **undefined** 的占位符。

**undefined** 指的是：已经为变量分配了内存，但还没有赋任何值。

## 二、什么是 not defined

如果一个对象/变量在内存分配阶段连声明/查找都没有出现，却试图访问它，那么它就是 **not defined（未定义）**。

`Not Defined !== Undefined`

> [!note] 核心结论
> 变量已声明但未赋值时，它的当前值是 **undefined**；变量本身没有声明却在代码中被调用时，它是 **not defined**。

## 三、两者的区别

```js
console.log(x); // undefined
var x = 25;
console.log(x); // 25
console.log(a); // Uncaught ReferenceError: a is not defined
```

| 对比项 | undefined | not defined |
| --- | --- | --- |
| 含义 | 变量已声明，内存已分配，但未赋值 | 变量根本没有声明 |
| 出现阶段 | 内存分配阶段就存在 | 内存分配阶段不存在 |
| 访问结果 | 得到 `undefined` | 抛出 `ReferenceError` |

## 四、JS 是弱类型语言

JS 是一门**松散类型（Loosely Typed）/ 弱类型（Weakly Typed）**语言。它不会把变量绑定到任何数据类型上。我们可以写 *var a = 5*，之后再把它改成布尔值 *a = true* 或字符串 *a = 'hello'*。

## 五、不要手动赋 undefined

**永远不要**手动把 *undefined* 赋给变量，让它自行发生即可。

> [!warning] 易错点
> 不要用 `var a = undefined;` 这样的写法来“重置”变量；undefined 应当由 JS 引擎在内存分配阶段自然产生。

---

> [!info] 🎬 配套视频
> [在 YouTube 观看本讲](https://www.youtube.com/watch?v=B7iF6G3EyIk&ab_channel=AkshaySaini)
