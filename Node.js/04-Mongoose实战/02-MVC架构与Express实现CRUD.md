---
title: MVC 架构与 Express 实现 CRUD
tags: [nodejs, express, mongoose, mvc, crud]
course: "A Complete NODE JS Course (#67-#71)"
created: 2026-11-13
---

# MVC 架构与 Express 实现 CRUD

> [!abstract] 本篇定位
> 覆盖课程 #67–#71。把 [[02-Express/02-CRUD接口实现|Express CRUD（内存版）]] 和 [[04-Mongoose实战/01-连接数据库与Schema与Model|Mongoose Model]] 拼在一起，并按 **MVC** 把代码拆层——这是课程 `CINEFLIX PROJECT` 的正式工程结构，后面所有板块（查询/错误处理/认证）都在这个结构上生长。

## 一、为什么需要 MVC（#67）

![MVC 架构幻灯片](img/course-mg02-mvc-slide.jpg)
*图 G2-01 · 视频 #67 · APPLICATION LOGIC / BUSINESS LOGIC / PRESENTATION LAYER 分层图*

课程用的拆分逻辑（比经典 MVC 图更贴近实际 Node 项目）：

| 层 | 职责 | 课程里的文件 |
| :--- | :--- | :--- |
| **路由 Route** | 只做"URL → 处理函数"的映射薄层 | `routes/movieRoutes.js` |
| **控制器 Controller** | 应用逻辑：拿请求、调模型、发响应 | `controllers/movieController.js` |
| **模型 Model** | 业务逻辑 + 数据：Schema、验证、业务规则 | `models/movieModel.js` |
| **视图 View** | 这里是纯 API 项目，JSON 就是"视图" | 无（REST API） |

好处：路由文件不再塞满业务代码；同一份业务逻辑可以被不同路由复用（如 alias 路由 [[04-Mongoose实战/03-高级查询与分页]]）。

![控制器代码](img/course-mg02-controllers.jpg)
*图 G2-02 · 视频 #67 · movieController.js 导出的一组处理函数*

## 二、CRUD 五个接口完整实现（#68–#71）

### 1. Create —— POST /api/movies

![Postman 创建文档](img/course-mg02-postman-post.jpg)
*图 G2-03 · 视频 #68 · Postman 里 POST 新电影并收到 201 响应*

![createMovie 代码现场](img/course-mg02-create.jpg)
*图 G2-03b · 视频 #68 · `Movie.create(req.body)` 代码与响应*

```javascript
// controllers/movieController.js
const Movie = require('./../models/movieModel');

exports.createMovie = async (req, res) => {
  try {
    const movie = await Movie.create(req.body);       // Schema 自动过滤多余字段并校验
    res.status(201).json({                            // 创建成功约定用 201
      status: 'success',
      data: { movie },
    });
  } catch (err) {
    res.status(400).json({ status: 'fail', message: err.message });
  }
};
```

### 2. Read（全部 / 单条）—— GET

```javascript
exports.getAllMovies = async (req, res) => {
  try {
    const movies = await Movie.find();                // 条件对象为空 = 全部
    res.status(200).json({ status: 'success', data: { movies } });
  } catch (err) { res.status(400).json({ status: 'fail', message: err.message }); }
};

exports.getMovie = async (req, res) => {
  try {
    const movie = await Movie.findById(req.params.id);  // :id 来自路由参数
    res.status(200).json({ status: 'success', data: { movie } });
  } catch (err) { res.status(400).json({ status: 'fail', message: err.message }); }
};
```

![查询单条电影](img/course-mg02-get-one.jpg)
*图 G2-04 · 视频 #69 · `Movie.findById(req.params.id)` 的运行现场*

### 3. Update —— PATCH /api/movies/:id

![更新代码](img/course-mg02-update-code.jpg)
*图 G2-05 · 视频 #70 · `findByIdAndUpdate(id, req.body, options)` 与 options 对象*

```javascript
exports.updateMovie = async (req, res) => {
  try {
    const movie = await Movie.findByIdAndUpdate(req.params.id, req.body, {
      new: true,            // 返回【更新后】的文档（默认返回旧文档！）
      runValidators: true,  // 更新时也跑 Schema 验证（默认只在 create 时跑）
    });
    res.status(200).json({ status: 'success', data: { movie } });
  } catch (err) { res.status(400).json({ status: 'fail', message: err.message }); }
};
```

> [!important] 这两个 options 是课程强调的易错点
> 不写 `new: true`，响应里给的是**改之前**的数据；不写 `runValidators: true`，PATCH 可以把 `ratings` 改成 999 绕过验证。

### 4. Delete —— DELETE /api/movies/:id

![删除代码](img/course-mg02-delete-code.jpg)
*图 G2-06 · 视频 #71 · `findByIdAndDelete` 与 204 无内容响应*

```javascript
exports.deleteMovie = async (req, res) => {
  try {
    await Movie.findByIdAndDelete(req.params.id);
    res.status(204).json({           // 204 = 成功但无内容可返回
      status: 'success',
      data: null,
    });
  } catch (err) { res.status(400).json({ status: 'fail', message: err.message }); }
};
```

### 5. 路由层把一切接起来

```javascript
// routes/movieRoutes.js
const express = require('express');
const router = express.Router();
const movieController = require('./../controllers/movieController');

router.route('/')
  .get(movieController.getAllMovies)
  .post(movieController.createMovie);

router.route('/:id')
  .get(movieController.getMovie)
  .patch(movieController.updateMovie)
  .delete(movieController.deleteMovie);

module.exports = router;

// server.js
app.use('/api/v1/movies', require('./routes/movieRoutes'));
```

## 三、Mongoose Model 常用方法速查

![Model API 列表](img/course-mg02-model-api-list.jpg)
*图 G2-07 · 视频 #71 · 官方文档中 Model 类的全部静态方法*

| 方法 | 说明 |
| :--- | :--- |
| `Model.create(doc)` | 创建并保存 |
| `Model.find(filter)` | 查多条 → 游标包装成数组 |
| `Model.findOne(filter)` | 单条 |
| `Model.findById(id)` | `findOne({_id: id})` 的糖 |
| `Model.findByIdAndUpdate(id, update, opts)` | 查到即改 |
| `Model.findByIdAndDelete(id)` | 查到即删 |
| `doc.save()` | 实例方法：保存修改 |

完整列表：[Mongoose Models API](https://mongoosejs.com/docs/api/model.html)

> [!warning] 重复出现的错误处理样板
> 每个函数都写一遍 try/catch + 相同响应——课程 #90 起专门解决它，见 [[05-错误处理/01-默认路由与全局错误处理|全局错误处理中间件]]。

## 相关笔记

- 上一篇：[[04-Mongoose实战/01-连接数据库与Schema与Model|连接数据库与 Schema/Model]]
- 下一篇：[[04-Mongoose实战/03-高级查询与分页|高级查询与分页]]
- 回顾：[[02-Express/02-CRUD接口实现|Express CRUD（内存版）——本篇是它的"真数据库版"]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #67 | MVC 架构分层与控制器抽取 |
| #68 | 从 Express 创建文档（POST + 201） |
| #69 | 查询文档（GET 全部 / 单条） |
| #70 | 更新文档（findByIdAndUpdate + options） |
| #71 | 删除文档（204）与 Model API 总览 |
