---
title: CRUD 接口实现
tags: [NodeJS, Express, CRUD, Postman]
course: "A Complete NODE JS Course #34-#38"
created: 2026-11-13
官方文档:
  - "[Express Routing](https://expressjs.com/en/5x/api.html#routing)"
  - "[req.body 说明](https://expressjs.com/en/5x/api.html#req.body)"
---

# Express 02 - CRUD 接口实现

> [!info] 本篇覆盖
> 课程 #34 GET、#35 POST、#36 路由参数、#37 PATCH、#38 DELETE。
> 一句话目标：**把 [[01-Express入门与RESTful设计]] 的 REST 设计图变成真正能跑的电影 API。**
> （⚠️ 本章视频素材因 YouTube 限流未能完整截图，代码与流程依据字幕完整还原）

---

## 一、GET：读资源（#34）

### 1.1 API 端点规范

课程惯例：**API 路径带 `/api` 前缀和版本号**：

```js
app.get('/api/v1/movies', (request, response) => {   // 路由处理器 Route Handler
  response.json(movies);                              // 返回全部电影
});

app.get('/api/v1/movies/:id', (request, response) => {  // 单个，见第三节
  const movie = movies.find(el => el.id === Number(request.params.id));
  response.json(movie);
});
```

### 1.2 为什么要版本化

课程特别强调：**改 API 时不要动 V1，新开 V2**（`/api/v1/movies`、`/api/v2/movies`）——老客户继续用 V1 不会被打断，新客户用 V2。版本前缀是 REST API 的工程标配。

---

## 二、POST：建资源（#35）

### 2.1 关键坑：`request.body` 默认是 `undefined`

Express **开箱不会解析请求体**——`request.body` 是 `undefined`！这是本课最重要的一集知识点：

```js
app.use(express.json());    // ← 中间件：把 JSON 请求体挂到 request.body

app.post('/api/v1/movies', (request, response) => {
  console.log(request.body);        // 现在才有数据了
  movies.push(request.body);        // 写入数据源（课程暂用 JSON 文件）
  response.send('created');
});
```

> [!important] 中间件第一次登场
> `express.json()` 就是一个**中间件**——站在请求与响应之间"加工"请求的函数。它把请求体 JSON 解析后挂到 `req.body`。没有这一行，POST 永远拿不到数据。中间件机制详见 [[03-中间件机制]]。

### 2.2 用 Postman 测试

1. 方法选 POST，URL 同端点
2. Body → **raw → JSON**，注意 **不要带 id**（REST 约定 id 由数据源自动生成）
3. 发送后看服务端打印出 body 对象

课程示例请求体：

```json
{ "name": "test", "releaseYear": 2018 }
```

状态码最佳实践：创建成功应返回 `201 Created`（Express 4 课程用了 `send('created')`，标准写法见下）。

```js
response.status(201).json(newMovie);
```

---

## 三、路由参数（#36）

### 3.1 概念与语法

![[img/course-e04-param-middleware.jpg]]
*图 E2-1 · 路由参数与 param 中间件的实际代码（#44 的完整版）*

```js
app.get('/api/v1/movies/:id', (request, response) => { ... });
//                            ↑ 冒号声明路由参数（named URL segment）
```

**路由参数 = URL 中命名的可变段**。请求 `/api/v1/movies/4` 时，`4` 被捕获进：

```js
request.params        // { id: '4' }  ← 注意是字符串！
```

### 3.2 `:id` 是字符串的坑

```js
const id = Number(request.params.id);        // 必须转数字
const movie = movies.find(el => el.id === id);
```

Express 4 里不转数字直接 `el.id === request.params.id` 会永远 false（`1 === '1'` 为 false）。

### 3.3 顺手装 nodemon

课程在本集装了 `nodemon`（文件变动自动重启服务器，免去 Ctrl+C 循环）：

```bash
npm install nodemon      # 正确姿势应加 -D，见 [[08-npm与包管理]] 勘误
npx nodemon app.js
```

---

## 四、PATCH：改资源（#37）

> **PUT**：提交完整对象做整体替换；**PATCH**：只提交要改的字段做局部更新。课程选 PATCH。

```js
app.patch('/api/v1/movies/:id', (request, response) => {
  const id = Number(request.params.id);
  const movieToUpdate = movies.find(el => el.id === id);
  if (!movieToUpdate) {
    return response.status(404).json({ status: 'fail', message: 'No movie with ID' });
  }

  // 用请求体里的新值覆盖旧值
  const { name, releaseYear } = request.body;
  if (name !== undefined) movieToUpdate.name = name;
  if (releaseYear !== undefined) movieToUpdate.releaseYear = releaseYear;

  response.status(200).json({ status: 'success', data: { movie: movieToUpdate } });
});
```

课程演示：把 id=2 电影的 `releaseYear` 从 2014 改成 2018——请求体**只带 releaseYear 一个字段**，其余字段不动，这就是 PATCH 的语义。

> [!tip] 只复制 body 里存在的字段
> 上面 `if (xxx !== undefined)` 的模式很重要：POST 的 body 由客户端提供，**不可信任**——只更新明确传来的字段，防止把 `id` 或未定义字段灌进数据。更优雅的现代写法是 `Object.assign` / 解构过滤，Express 教程后续（[[04-Mongoose实战/02-MVC架构与Express实现CRUD]]）会复用这个思想。

---

## 五、DELETE：删资源（#38）

```js
app.delete('/api/v1/movies/:id', (request, response) => {
  const id = Number(request.params.id);

  const movieToDelete = movies.find(el => el.id === id);       // ① 找到对象
  if (!movieToDelete) return response.status(404).json({ ... });

  const index = movies.indexOf(movieToDelete);                 // ② 找到下标
  movies.splice(index, 1);                                     // ③ 原地删除 1 个元素

  response.status(204).json({ status: null, data: null });     // ④ 204 No Content
});
```

课程流程四步：find → indexOf → splice → 返回 **204 No Content**（删除成功不返回正文）。

> [!warning] splice 会修改原数组
> 课程强调 `splice` **原地变异** movies 数组。若是并发环境或想保留快照，用不可变写法 `movies.filter(el => el.id !== id)`。

---

## 六、CRUD 全景对照表

| 动作 | 方法 + URL | 状态码 | 请求体 |
|:---|:---|:---|:---|
| 列表 | `GET /api/v1/movies` | 200 | 无 |
| 单个 | `GET /api/v1/movies/:id` | 200 / 404 | 无 |
| 创建 | `POST /api/v1/movies` | 201 | 新对象（无 id） |
| 更新 | `PATCH /api/v1/movies/:id` | 200 / 404 | 变更字段 |
| 删除 | `DELETE /api/v1/movies/:id` | 204 | 无 |

> [!note] 数据源的演进
> 课程把数据存在 `movies.json` 里读写——只适合演示。真实应用要用数据库：Mongoose 版 CRUD 见 [[04-Mongoose实战/02-MVC架构与Express实现CRUD]]。

---

## 七、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| 忘写 `express.json()` | `req.body` 是 undefined | `app.use(express.json())` 放路由前 |
| `params.id` 不转数字 | find 永远匹配不到 | `Number(request.params.id)` |
| DELETE 返回 200+正文 | 不合 REST 语义 | 用 204 |
| POST body 里带 id | 数据冲突 | id 由数据源生成 |

> [!summary] 自查清单
> - [ ] 能解释为什么需要 express.json()
> - [ ] 会用 Postman 发五种请求并说对状态码
> - [ ] 理解路由参数与 request.params
> - [ ] 分清 PUT 与 PATCH

---
**相关笔记**：[[01-Express入门与RESTful设计]] · [[03-中间件机制]] · [[04-Mongoose实战/02-MVC架构与Express实现CRUD]] · [[README|课程导航]]
