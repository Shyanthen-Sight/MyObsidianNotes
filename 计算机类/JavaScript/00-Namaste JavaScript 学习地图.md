---
title: "Namaste JavaScript 学习地图"
aliases:
  - Namaste JavaScript Notes
  - Namaste JS 笔记
tags:
  - JavaScript
  - Namaste-JavaScript
  - MOC
source: https://github.com/alok722/namaste-javascript-notes
---

# Namaste JavaScript 学习地图

> [!info] 关于本套笔记
> 本套笔记译自 Alok Raj 的开源项目 [namaste-javascript-notes](https://github.com/alok722/namaste-javascript-notes)，内容整理自 Akshay Saini 的 YouTube 课程 [Namaste JavaScript](https://www.youtube.com/watch?v=pN6jk0uUrD8&list=PLlasXeu85E9cQ32gLCvAvr9vNaUccPVNP)。
>
> 全 25 讲已按主题重新组织为 6 个章节，并完整翻译为简体中文。原项目采用 MIT 协议。

## 课程简介

这套笔记覆盖 JavaScript 的核心运行机制与高频面试考点：从执行上下文、调用栈、变量提升，到作用域链、闭包、事件循环、Promise 与 `this`。适合作为系统复习 JavaScript 底层原理的路线图。

建议按章节顺序阅读；每篇笔记开头都有「本讲要点」，结尾附有原课程的 YouTube 视频链接。

## 目录

### 第一章 · JS 运行机制

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 01 | [[01-执行上下文\|执行上下文]] | 内存组件、代码组件、同步与单线程 |
| 02 | [[02-JS如何执行与调用栈\|JS 如何执行与调用栈]] | 内存创建阶段、代码执行阶段、调用栈 |
| 03 | [[03-变量提升\|变量提升]] | 变量与函数声明提升 |
| 04 | [[04-函数与变量环境\|函数与变量环境]] | 函数执行上下文、变量环境 |
| 05 | [[05-最短的JS程序-window与this\|最短的 JS 程序、window 与 this]] | 全局对象、`this` 初探 |
| 06 | [[06-undefined与not-defined\|undefined 与 not defined]] | 未赋值与未声明的区别 |

### 第二章 · 作用域与闭包

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 07 | [[07-作用域链与词法环境\|作用域链与词法环境]] | 作用域、词法环境、作用域链 |
| 08 | [[08-let与const-暂时性死区\|let、const 与暂时性死区]] | TDZ、重复声明、const 规则 |
| 09 | [[09-块级作用域与变量遮蔽\|块级作用域与变量遮蔽]] | 块级作用域、Shadowing、非法遮蔽 |
| 10 | [[10-闭包\|闭包]] | 闭包定义、形成过程、应用与陷阱 |
| 11 | [[11-setTimeout与闭包面试题\|setTimeout 与闭包面试题]] | `var` 循环陷阱、闭包修复 |
| 12 | [[12-闭包经典面试题\|闭包经典面试题]] | 多道闭包高频面试题 |

### 第三章 · 函数与异步

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 13 | [[13-一等函数与匿名函数\|一等函数与匿名函数]] | First-class Functions、函数表达式 |
| 14 | [[14-回调函数与事件监听\|回调函数与事件监听]] | 回调、事件监听器、闭包 |
| 15 | [[15-异步JavaScript与事件循环\|异步 JavaScript 与事件循环]] | 调用栈、Web API、任务队列、事件循环 |
| 16 | [[16-JS引擎与V8架构\|JS 引擎与 V8 架构]] | 解析、解释、JIT 编译、垃圾回收 |
| 17 | [[17-setTimeout的可信度问题\|setTimeout 的可信度问题]] | 定时器延迟、并发模型 |

### 第四章 · 高阶函数与数组

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 18 | [[18-高阶函数与函数式编程\|高阶函数与函数式编程]] | 高阶函数、函数式编程思想 |
| 19 | [[19-map-filter-reduce\|map、filter 与 reduce]] | 三大数组方法及手写实现 |

### 第五章 · Promise 与 async/await

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 20 | [[20-回调\|回调]] | 回调地狱及其问题 |
| 21 | [[21-Promise\|Promise]] | Promise 状态、`then`、解决回调地狱 |
| 22 | [[22-创建Promise-链式调用与错误处理\|创建 Promise、链式调用与错误处理]] | 创建、链式调用、错误处理 |
| 23 | [[23-async与await\|async 与 await]] | 语法糖、执行顺序、错误处理 |
| 24 | [[24-Promise-API\|Promise API]] | `all`、`allSettled`、`race`、`any` 及面试题 |

### 第六章 · this 关键字

| 讲次 | 笔记 | 核心内容 |
| :--: | :--- | :--- |
| 25 | [[25-this关键字\|this 关键字]] | 全局、函数、对象方法、箭头函数、DOM |

## 相关链接

- 🌐 [原项目笔记网站](https://alok722.github.io/namaste-javascript-notes/lectures.html)
- 📄 [原项目 PDF 版](https://github.com/alok722/namaste-javascript-notes/blob/main/dist/namaste-javascript-notes.pdf)
- 👤 作者：Alok Raj（[GitHub](https://github.com/alok722) · [LinkedIn](https://linkedin.com/in/alok722)）

---

> [!note] 归档说明
> 原 `JavaScript` 文件夹中已有的笔记已统一移动到 `others/` 文件夹，与本套课程笔记分开存放。
