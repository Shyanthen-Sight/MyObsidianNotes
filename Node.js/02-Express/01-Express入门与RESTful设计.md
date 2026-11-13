---
title: Express 入门与 RESTful 设计
tags: [NodeJS, Express, REST, API]
course: "A Complete NODE JS Course #30-#33"
created: 2026-11-13
官方文档:
  - "[Express 入门](https://expressjs.com/en/starter/hello-world.html)"
  - "[Express 5 API](https://expressjs.com/en/5x/api.html)"
---

# Express 01 - Express 入门与 RESTful 设计

> [!info] 本篇覆盖
> 课程 #30 Express 简介、#31 前端 vs 后端、#32 Web API、#33 REST 架构。
> 一句话目标：**明白为什么需要框架、写出第一个 Express 应用、会用 REST 的世界观设计 URL。**

---

## 一、为什么需要 Express（#30）

回看 [[03-构建HTTP服务器]] 的原生写法：路由靠 if/else 链、读请求体要手动收集 chunk、Content-Type 全靠手写、加一个中间件就要改一大段逻辑。**Express = 把这些模板代码收敛成几行声明式 API**：

![[img/course-e01-express-hello.jpg]]
*图 E1-1 · 00:15:13 · 同样的服务器，Express 只要几行*

```js
const express = require('express');       // 第三方包（[[08-npm与包管理]] 装的）

const app = express();

app.get('/', (request, response) => {     // 路由：方法 + 路径 + 处理器
  response
    .status(200)                          // 状态码
    .send('Hello from express server!')   // 自动设置 Content-Type 并 end()
    .json({ app: 'express', number: 1 })  // 或直接发 JSON
});

app.use((request, response) => {          // 兜底 404
  response.status(404).send("Sorry, can't find that!");
});

app.listen(3000, () => console.log('Server has started!'));
```

对比原生 http 模块：

| 任务 | 原生 http | Express |
|:---|:---|:---|
| 路由 | 手写 if/else 判断 `request.url` | `app.get('/path', handler)` |
| 响应头 | 手动 `writeHead` | `res.send()`/`res.json()` 自动推断 |
| 解析参数 | `url.parse` / 手动处理 | `request.query` / `request.params` 直接用 |
| 错误处理 | 自己 try-catch 每处 | 全局错误中间件统一处理（见 [[05-错误处理/01-默认路由与全局错误处理]]） |

> [!note] 2026 年的版本差异（课程用的 Express 4，如今是 5.x）
> 写作时 Express 已发布 **5.3.0**。与本课程相关的重要变化：
> 1. **Promise 拒绝自动进入错误处理**——`app.get()` 的 async 处理器 throw 出错会自动传给错误中间件，Express 4 需要 `express-async-errors` 或手写 catch
> 2. 通配符路由 `app.get('*')` 改为 `app.get('*splat')` 或 `app.get('/{*splat}')`，正则语法收紧
> 3. `res.send(status)` 这种把状态码当 body 的旧写法被移除，必须 `res.status(x).send(y)`
> 其余 99% 的课程代码在 5.x 下原样可用。

---

## 二、前端 vs 后端（#31）

![[img/course-e01-fe-vs-be.jpg]]
*图 E1-2 · 00:05:35 · 前端（浏览器代码）与后端（Web Server）*

| | 前端（Front End） | 后端（Back End） |
|:---|:---|:---|
| 运行在哪 | 客户端浏览器 | 服务器 |
| 技术 | HTML / CSS / JS | 任意：PHP+MySQL、Python+PostgreSQL、**Node.js+MongoDB** |
| 职责 | 界面渲染、交互 | 业务逻辑、数据存取 |

![[img/course-e01-dynamic-vs-api.jpg]]
*图 E1-3 · 00:06:18 · 动态网站 vs API 驱动网站*

- **动态网站**：后端渲染好整个 HTML 发给浏览器（[[04-路由与响应]] 的模板做法）
- **API 驱动网站**：后端只发 JSON，页面由前端框架在浏览器里构建——这就是"前后端分离"，也是本课程后面做 Movie API 的形态

---

## 三、什么是 Web API（#32）

> **API = Application Programming Interface**——但在这个语境里，"Application" 不是狭义的应用：课程特意指出 ==`fs` 和 `http` 模块本身就是 Node 提供的 API==。Web API 特指**通过 HTTP 暴露、供程序调用**的接口。

Web API 的消费者不是人而是**程序**：Web 前端、手机 App、第三方服务（支付回调、天气数据）。特点：

- 输入输出结构化（通常是 JSON）
- 不关心调用方是谁、界面长什么样
- 由文档约定输入输出

---

## 四、REST：API 设计的世界观（#33）

![[img/course-e01-rest-rules.jpg]]
*图 E1-4 · 00:10:11 · REST 架构五原则*

REST（REpresentational State Transfer）是一套 **URL 设计规范**，课程总结为五条：

1. **Separate APIs into logical resources** —— 按资源划分
2. **Expose structured, resource-based URLs** —— URL 指向资源，不是动作
3. **Use HTTP methods** —— 用方法表达动作
4. **Send JSON data in response**
5. **API must be stateless** —— ==无状态==：服务器不记住上一次请求，每次请求必须自带全部信息（会话信息放 token，见 [[06-认证与授权/02-JWT原理与登录注册]]）

### 动词映射：一个 URL 处理全部 CRUD

![[img/course-e01-rest-mapping.jpg]]
*图 E1-5 · 00:06:35 · 动词驱动的电影 API*

| URL（资源） | 方法 | 动作 |
|:---|:---|:---|
| `/movies` | GET | 读列表 |
| `/movies/21` | GET | 读单个 |
| `/movies` | POST | 创建 |
| `/movies/21` | PUT | 整体更新 |
| `/movies/21` | PATCH | 部分更新 |
| `/movies/21` | DELETE | 删除 |

反例对照（**不要这样设计**）：

```
/addMovie ❌  →  POST /movies ✅
/deleteMovie?id=21 ❌  →  DELETE /movies/21 ✅
```

> [!tip] PUT vs PATCH 的区别（面试常问）
> **PUT** 提交完整资源做整体替换；**PATCH** 只提交要改的字段做局部更新。课程在 [[02-CRUD接口实现]] 里用 `PATCH /movies/:id` 实现改价。

---

## 五、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| 在 URL 里写动词 | `/getUser` / `/deleteUser` | 资源 URL + HTTP 方法 |
| 忘记 404 兜底 | 未匹配路径挂起 | `app.use()` 放在所有路由之后 |
| Express 5 用 `'*'` 通配符 | `PathError: Unexpected * at 1` | 改 `'*splat'` |
| API 记录会话状态 | 服务器扩容后状态丢失 | 无状态设计 + token |

> [!summary] 自查清单
> - [ ] 能说出 Express 相对原生 http 解决了什么
> - [ ] 能画出前后端分离的数据流
> - [ ] 背出 REST 五原则
> - [ ] 会用资源 URL + HTTP 方法设计 CRUD

---
**相关笔记**：[[03-构建HTTP服务器]] · [[08-npm与包管理]] · [[02-CRUD接口实现]] · [[README|课程导航]]
