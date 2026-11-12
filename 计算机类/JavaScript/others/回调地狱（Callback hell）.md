---
title: 回调地狱（Callback Hell）
source: https://callbackhell.com/
author: Max Ogden
type: 技术文章
topic: JavaScript / 异步编程
tags:
  - JavaScript
  - 异步编程
  - Callback
  - Promise
created: 2026-11-10
---

# 回调地狱（Callback Hell）

> **一句话总结**：回调地狱不是 JavaScript 的缺陷，而是「用同步思维写异步代码」造成的坏习惯。只要做到 **函数不嵌套、逻辑模块化、处理每一个错误**，就能写出清晰可靠的异步代码。

> [!info] 原文信息
> - 标题：*Callback Hell: Taming JavaScript's Async Complexity*
> - 副标题：A guide to writing asynchronous JavaScript programs
> - 作者：Max Ogden
> - 来源：<https://callbackhell.com/>
> - 源码：<https://github.com/maxogden/callback-hell>

---

## 目录

- [[#什么是「回调地狱」？|什么是「回调地狱」？]]
- [[#什么是回调？|什么是回调？]]
- [[#如何解决回调地狱？|如何解决回调地狱？]]
  - [[#1. 保持代码「浅」|1. 保持代码「浅」]]
  - [[#2. 模块化|2. 模块化]]
  - [[#3. 处理每一个错误|3. 处理每一个错误]]
- [[#总结|总结]]
- [[#延伸阅读|延伸阅读]]
- [[#那 Promise / Generator / ES6 呢？|那 Promise / Generator / ES6 呢？]]
- [[#记住：只有你能阻止回调地狱|记住：只有你能阻止回调地狱]]

---

## 什么是「回调地狱」？

异步 JavaScript（也就是使用回调的 JavaScript）很难凭直觉写对。很多代码最终会写成这样：

```javascript
fs.readdir(source, function (err, files) {
  if (err) {
    console.log('Error finding files: ' + err)
  } else {
    files.forEach(function (filename, fileIndex) {
      console.log(filename)
      gm(source + filename).size(function (err, values) {
        if (err) {
          console.log('Error identifying file size: ' + err)
        } else {
          console.log(filename + ' : ' + values)
          aspect = (values.width / values.height)
          widths.forEach(function (width, widthIndex) {
            height = Math.round(width / aspect)
            console.log('resizing ' + filename + 'to ' + height + 'x' + height)
            this.resize(width, height).write(dest + 'w' + width + '_' + filename, function (err) {
              if (err) console.log('Error writing file: ' + err)
            })
          }.bind(this))
        }
      })
    })
  }
})
```

看到那层层递进的「金字塔」形状，以及结尾那一堆 `})` 了吗？这就是被大家戏称为 **回调地狱（callback hell）** 的东西。

回调地狱的成因，是人们试图用**从上到下的视觉顺序**来写 JavaScript。很多人都会犯这个错误！在 C、Ruby 或 Python 等语言中，人们习惯认为第 1 行的事情做完之后，第 2 行的代码才会开始运行，如此一路往下。但正如你将学到的，JavaScript 并非如此。

---

## 什么是回调？

回调（callback）只是使用 JavaScript 函数的一种**约定俗成的叫法**。JavaScript 语言里并没有一个叫「callback」的特殊东西，它只是一种约定。

大多数函数会立即返回结果；而使用回调的函数则需要**花一些时间**才能产生结果。「异步（asynchronous，简称 async）」这个词的意思就是「需要一些时间」或「将来才发生，而不是现在」。通常只有在做 I/O 时才会用到回调，例如下载东西、读取文件、与数据库通信等。

调用普通函数时，你可以直接使用它的返回值：

```javascript
var result = multiplyTwoNumbers(5, 10)
console.log(result)
// 50 gets printed out
```

然而，异步的、使用回调的函数不会立刻返回任何东西：

```javascript
var photo = downloadPhoto('http://coolcats.com/cat.gif')
// photo is 'undefined'!
```

在这个例子里，这张 gif 可能要下载很久，而你并不希望程序在等待下载完成时被暂停（也就是「阻塞」）。

于是，你把「下载完成后应该运行的代码」存进一个函数里。这就是回调！你把它交给 `downloadPhoto` 函数，它会在下载完成时运行你的回调（也就是「稍后回叫你」），并把照片传进去（如果出了问题，则传入一个错误）：

```javascript
downloadPhoto('http://coolcats.com/cat.gif', handlePhoto)

function handlePhoto (error, photo) {
  if (error) console.error('Download error!', error)
  else console.log('Download finished', photo)
}

console.log('Download started')
```

人们理解回调时最大的障碍，是理解**程序运行时各个事情的执行顺序**。在这个例子里，主要发生了三件事：

1. 先声明 `handlePhoto` 函数；
2. 然后调用 `downloadPhoto`，并把 `handlePhoto` 作为回调传给它；
3. 最后打印出 `'Download started'`。

注意，此时 `handlePhoto` **还没有被调用**，它只是被创建出来并作为回调传进了 `downloadPhoto`。它要等到 `downloadPhoto` 完成自己的任务后才会运行，而这个任务可能耗时很久，取决于网速。

这个例子旨在说明两个重要概念：

- `handlePhoto` 回调只是一种「把某些事情存起来、留到以后再做」的方式；
- 事情发生的顺序**并不是从上到下**的，而是根据各自何时完成来跳转。

---

## 如何解决回调地狱？

回调地狱是由不良的编码习惯造成的。幸运的是，写出更好的代码并不难！

你只需要遵循**三条规则**。

---

## 1. 保持代码「浅」

下面是一段凌乱的浏览器 JavaScript，它使用 [browser-request](https://github.com/iriscouch/browser-request) 向服务器发起 AJAX 请求：

```javascript
var form = document.querySelector('form')
form.onsubmit = function (submitEvent) {
  var name = document.querySelector('input').value
  request({
    uri: "http://example.com/upload",
    body: name,
    method: "POST"
  }, function (err, response, body) {
    var statusMessage = document.querySelector('.status')
    if (err) return statusMessage.value = err
    statusMessage.value = body
  })
}
```

这段代码里有两个匿名函数。让我们给它们起个名字！

```javascript
var form = document.querySelector('form')
form.onsubmit = function formSubmit (submitEvent) {
  var name = document.querySelector('input').value
  request({
    uri: "http://example.com/upload",
    body: name,
    method: "POST"
  }, function postResponse (err, response, body) {
    var statusMessage = document.querySelector('.status')
    if (err) return statusMessage.value = err
    statusMessage.value = body
  })
}
```

如你所见，给函数命名非常简单，而且能带来一些立竿见影的好处：

- 因为有描述性的函数名，代码更易读；
- 发生异常时，堆栈信息会指向真实的函数名，而不是「anonymous」；
- 允许你把函数移到别处，并通过名字来引用它们。

现在我们可以把这些函数移到程序的顶层：

```javascript
document.querySelector('form').onsubmit = formSubmit

function formSubmit (submitEvent) {
  var name = document.querySelector('input').value
  request({
    uri: "http://example.com/upload",
    body: name,
    method: "POST"
  }, postResponse)
}

function postResponse (err, response, body) {
  var statusMessage = document.querySelector('.status')
  if (err) return statusMessage.value = err
  statusMessage.value = body
}
```

注意，这里的 `function` 声明被定义在文件的底部。这得益于 [函数提升（function hoisting）](https://gist.github.com/maxogden/4bed247d9852de93c94c)。

---

## 2. 模块化

这是最重要的一部分：**任何人都能够创建模块**（也就是库）。引用 [Isaac Schlueter](http://twitter.com/izs)（node.js 项目成员）的话：

> 「写小的模块，每个只做一件事，再把它们组装成做更大事情的模块。如果你不去回调地狱，你就不会陷入其中。」

让我们把上面的样板代码拆分到几个文件里，改造成一个模块。我会展示一种同时适用于浏览器代码和服务端代码（或两端通用）的模块模式。

下面是一个新文件 `formuploader.js`，其中包含了之前的两个函数：

```javascript
module.exports.submit = formSubmit

function formSubmit (submitEvent) {
  var name = document.querySelector('input').value
  request({
    uri: "http://example.com/upload",
    body: name,
    method: "POST"
  }, postResponse)
}

function postResponse (err, response, body) {
  var statusMessage = document.querySelector('.status')
  if (err) return statusMessage.value = err
  statusMessage.value = body
}
```

`module.exports` 是 node.js 模块系统的一个例子，它可以在 node、Electron 以及浏览器中通过 [browserify](https://github.com/substack/node-browserify) 使用。我很喜欢这种模块风格，因为它到处都能用、非常简单易懂，也不需要复杂的配置文件或脚本。

现在我们已经有了 `formuploader.js`（经 browserify 处理后，以 script 标签加载到页面中），只需要 require 它并使用即可！下面是我们的应用代码现在的样子：

```javascript
var formUploader = require('formuploader')
document.querySelector('form').onsubmit = formUploader.submit
```

现在我们的应用只有两行代码，并且带来了以下好处：

- 新开发者更容易理解——他们不必被迫读完成堆的 `formuploader` 函数；
- `formuploader` 可以在其他地方复用，而不必复制代码，也很容易分享到 github 或 npm 上。

---

## 3. 处理每一个错误

错误有不同的类型：由程序员造成的**语法错误**（通常在你第一次尝试运行程序时就会被发现）、由程序员造成的**运行时错误**（代码运行了，但某个 bug 导致了混乱）、以及由诸如无效的文件权限、硬盘故障、网络断开等原因造成的**平台错误**。本节只讨论最后一类错误。

前两条规则主要是为了让代码**可读**，而这一条是为了让代码**稳定**。使用回调时，你本质上是在处理这样的任务：被派发出去、在后台做某件事，然后成功完成或因失败而中止。任何有经验的开发者都会告诉你，你永远无法预知这些错误何时发生，所以你必须**假设它们总是会发生**。

使用回调时，最流行的错误处理方式是 Node.js 风格：回调的第一个参数永远保留给错误。

```javascript
var fs = require('fs')

fs.readFile('/Does/not/exist', handleFile)

function handleFile (error, file) {
  if (error) return console.error('Uhoh, there was an error', error)
  // otherwise, continue on and use `file` in your code
}
```

把第一个参数设为 `error` 是一个简单的约定，它能促使你记得处理错误。如果它是第二个参数，你就可能写出 `function handleFile (file) { }` 这样的代码，从而更容易忽略错误。

代码检查工具（linter）也可以配置成帮你记得处理回调错误。最简单易用的是 [standard](http://standardjs.com/)。你只需在代码目录里运行 `$ standard`，它就会把代码中每一个未处理错误的回调都展示给你。

---

## 总结

1. **不要嵌套函数。** 给它们起名字，并把它们放在程序的顶层。
2. **利用[函数提升](https://gist.github.com/maxogden/4bed247d9852de93c94c)，** 把函数移到「折叠线以下」（文件底部）。
3. **在每一个回调中处理每一个错误。** 使用 [standard](http://standardjs.com/) 这样的 linter 来帮助你做到这一点。
4. **创建可复用的函数并把它们放进模块**，以降低理解代码所需的认知负担。像这样把代码拆成小块，还有助于处理错误、编写测试，迫使你为代码建立稳定且有文档的公共 API，并且有利于重构。

避免回调地狱最重要的方面，是**把函数移开**，这样程序的流程就能更容易被理解，新来者也不必在函数的各种细节中跋涉，才能看到程序真正想做什么的核心。

你可以先把函数移到文件底部，然后进阶到把它们移进另一个文件，并用相对路径 require 加载，例如 `require('./photo-helpers.js')`，最后再把它们移进一个独立的模块，例如 `require('image-resize')`。

创建模块时的一些经验法则：

- 先把重复使用的代码移进一个函数；
- 当你的函数（或一组同一主题的函数）变得足够大时，把它们移进另一个文件，并用 `module.exports` 暴露出来。你可以用相对路径 require 来加载它；
- 如果你有一些可以跨多个项目使用的代码，就给它配上自己的 readme、测试和 `package.json`，并发布到 github 和 npm 上。这种做法的好处多到无法在此一一列举！
- 好的模块很小，并且只专注于一个问题；
- 模块中的单个文件不应超过约 150 行 JavaScript；
- 一个模块的嵌套文件夹中包含 JavaScript 文件的层级不应超过一层。如果超过了，它可能做了太多事情；
- 多请有经验的程序员给你展示一些好模块的示例，直到你对好模块长什么样有清晰的概念。如果你需要花几分钟以上才能理解发生了什么，那它可能不是一个很好的模块。

---

## 延伸阅读

- 阅读我写的[更长的回调入门](https://github.com/maxogden/art-of-node#callbacks)
- 试试 [nodeschool](http://nodeschool.io/) 上的一些教程
- 查看 [browserify-handbook](https://github.com/substack/browserify-handbook)，了解编写模块化代码的示例

---

## 那 Promise / Generator / ES6 呢？

在考虑更高级的解决方案之前，请记住：回调是 JavaScript 的基础部分（因为它们就是函数），你应该先学会如何读写回调，再去接触更高级的语言特性，因为那些特性全都依赖于对回调的理解。如果你还写不出可维护的回调代码，就继续努力吧！

如果你*真的*想让异步代码自上而下地阅读，有一些花哨的办法可以尝试。但请注意，**这些办法可能引入性能问题和/或跨平台运行时兼容性问题**，所以务必先做好调研。

**Promise** 是一种编写异步代码的方式，它看起来仍然像是在自上而下地执行，并且由于鼓励使用 `try/catch` 风格的错误处理，能处理更多类型的错误。

**Generator** 让你可以「暂停」单个函数，而不必暂停整个程序的状态；代价是代码理解起来稍微复杂一些，但能让异步代码看起来是自上而下执行的。可以参考 [watt](https://github.com/mappum/watt) 了解这种方式的例子。

**Async functions** 是一个拟议中的 ES7 特性，它会在更高层的语法中进一步封装 generator 和 promise。如果你感兴趣，可以去了解一下。

就我个人而言，我写的异步代码中有 90% 都使用回调；当事情变得复杂时，我会引入 [run-parallel](https://github.com/feross/run-parallel) 或 [run-series](https://github.com/feross/run-series) 之类的东西。我不认为回调、Promise 或其他方案对我而言真的有本质区别——最大的影响来自**保持代码简单、不嵌套、并拆分成小模块**。

无论你选择哪种方式，永远要**处理每一个错误**，并且**保持代码简单**。

---

## 记住：只有你能阻止回调地狱

> [!quote] 原文结尾
> 「Remember, only *you* can prevent callback hell and forest fires.」

本文的源码可以在 [GitHub](http://github.com/maxogden/callback-hell) 上找到。

---

## 核心要点速查

| 规则 | 做法 | 收益 |
| --- | --- | --- |
| 保持代码浅 | 给匿名函数命名，移到顶层；利用函数提升 | 可读性提升，堆栈信息可定位 |
| 模块化 | 单个函数/文件拆分为模块，`module.exports` 导出 | 复用、易测试、低认知负担 |
| 处理所有错误 | Node 风格 `(err, result)` 回调；用 `standard` 检查 | 稳定性，避免静默失败 |

> [!tip] 判断标准
> 如果一段异步代码需要你「从上往下读才能看懂」，或者结尾出现一整排 `})`，那就说明它已经开始陷入回调地狱了。
