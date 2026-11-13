---
title: 构建 HTTP 服务器
tags: [NodeJS, http, 网络, DNS]
course: "A Complete NODE JS Course #08-#10"
created: 2026-11-13
官方文档:
  - "[http 模块](https://nodejs.org/docs/latest/api/http.html)"
  - "[net 模块](https://nodejs.org/docs/latest/api/net.html)"
---

# 03 - 构建 HTTP 服务器

> [!info] 本篇覆盖
> 课程 #08 创建简单 Web 服务器、#09 Web 工作原理、#10 请求与响应机制。
> 一句话目标：**会用 `http.createServer` 起一个服务器，并真正理解"浏览器输入 URL 之后发生了什么"。**

---

## 一、最小可用服务器（#08）

![[img/course-03-createserver.jpg]]
*图 03-1 · 00:04:54 · createServer + listen 及终端输出*

```js
const http = require('http');          // 内置 http 模块

const server = http.createServer((request, response) => {
  console.log('A new request received');           // 每次浏览器请求都会打印
  response.end('Hello from the server!');          // 给浏览器返回内容
});

server.listen(8000, '127.0.0.1', () => {
  console.log('Server has started!');
});
```

```bash
$ node app.js
Server has started!
A new request received        ← 浏览器访问 http://127.0.0.1:8000 时出现
A new request received        ← favicon.ico 也会发一次请求！
```

三要素拆解：

| 组件 | 角色 | 类比 |
|:---|:---|:---|
| `http.createServer(callback)` | 创建服务器实例，回调处理每个请求 | 前台接待员 |
| `server.listen(port, host, cb)` | 监听端口，进程常驻 | 占下一个门牌号 |
| `(request, response)` | 请求对象（进来什么）、响应对象（回去什么） | 信件与回信 |

> [!warning] 为什么控制台打印了两条请求？
> 浏览器会自动请求 `/favicon.ico`。从现在起每个请求都要想清楚"这是页面请求还是资源请求"。

---

## 二、Web 的工作原理（#09）

### 2.1 Client-Server 架构

![[img/course-03-client-server.jpg]]
*图 03-2 · 00:02:23 · 客户端-服务器架构*

- **Client**：发起请求的一方（浏览器、App、Postman）
- **Server**：7×24 运行、监听端口、等待请求的机器
- 通信协议：**HTTP**（应用层协议），跑在 **TCP/IP** 之上

### 2.2 从输入 URL 到看到页面：完整链路

![[img/course-03-dns-tcp-request.jpg]]
*图 03-3 · 00:07:19 · DNS 解析 → TCP 连接 → 请求响应全链路*

1. **DNS 解析**：浏览器只知道域名（`www.nodeapp.com`），问 DNS 服务器要 **IP 地址**（`192.168.20.134`）。课程中 `127.0.0.1` 与 `localhost` 是特例——指本机自己，不经过 DNS。
2. **建立 TCP 连接**：与目标 IP 进行 TCP 三次握手（HTTP 底层传输）。
3. **发送 HTTP 请求**：浏览器构造请求报文——

```text
GET /index.html HTTP/1.1
Host: www.nodeapp.com
Accept-Language: en-US
User-Agent: Mozilla/5.0
BODY: null
```

4. **服务器响应**：返回状态行 + 响应头 + 响应体——

```text
HTTP/1.1 200 OK
Date: Tue, 06 Sep 2022
Content-Type: text/html; charset=utf-8
BODY: <!DOCTYPE html>...
```

5. 浏览器解析 HTML，遇到 `<link>` `<script>` `<img>` 再发起**更多请求**（所以 Network 面板里不止一条）。

> [!tip] 端口的意义
> 一台服务器只有一个 IP，却可以跑多个服务：`8000` 给 Node、`3306` 给 MySQL……端口就是"门牌号"。浏览器默认 80（HTTP）/ 443（HTTPS），所以平时不用写端口。

---

## 三、Request 与 Response 对象（#10）

### 3.1 request：请求的一切都在里面

```js
http.createServer((request, response) => {
  console.log(request.url);          // /about
  console.log(request.method);       // GET
  console.log(request.headers);      // 全部请求头
  response.end('Hello');
});
```

`request` 常用属性（完整列表见[官方文档](https://nodejs.org/docs/latest/api/http.html#class-httpincomingmessage)）：

| 属性 | 说明 |
|:---|:---|
| `request.url` | 请求路径及查询字符串，如 `/books?author=john` |
| `request.method` | `GET` / `POST` / `PATCH` / `DELETE`… |
| `request.headers` | 请求头对象（小写键名） |

### 3.2 response：控制"回去的内容"

```js
response.statusCode = 200;                     // 状态码
response.setHeader('Content-Type', 'text/html');
response.write('<h1>Hello</h1>');              // 可多次写
response.end('done');                          // 结束响应（必须调用）
```

### 3.3 用 DevTools 验证

课程 #10 打开 Chrome DevTools 的 Network 面板，观察一次真实请求的 Headers 标签页——**Request Headers** 与 **Response Headers** 与上面的报文一一对应。养成"用 DevTools 看真实报文"的习惯，比背概念有效得多。

---

## 四、这个服务器还缺什么？

当前服务器不管请求什么都回 `Hello from the server!`。真实应用至少需要：

1. **根据 URL 区分响应** → 路由，见 [[04-路由与响应]]
2. **设置正确的响应头** → `Content-Type` 决定浏览器怎么解析，见 [[04-路由与响应]]
3. **返回 JSON** → API 的核心，见 [[05-JSON与查询字符串]]
4. **更简洁的写法** → Express 框架，见 [[01-Express入门与RESTful设计]]

> [!summary] 自查清单
> - [ ] 能手写 createServer + listen 五行最小服务器
> - [ ] 能完整讲述"输入 URL 后发生了什么"（DNS→TCP→请求→响应）
> - [ ] 知道 request 三大常用属性
> - [ ] 知道 favicon 会产生额外请求

---
**相关笔记**：[[04-路由与响应]] · [[09-Node架构与事件循环]] · [[README|课程导航]]
