---
title: 异步错误处理与 NotFound
tags: [nodejs, express, async, catchasync]
course: "A Complete NODE JS Course (#93-#94)"
created: 2026-11-13
---

# 异步错误处理与 Not Found

> [!abstract] 本篇定位
> 覆盖课程 #93–#94。解决两个高频问题：**async 控制器里 Promise reject 后 Express 不知道**；**"ID 存在但查不到"** 该返回什么。

## 一、异步函数的错误去哪了（#93）

Express 4 只自动捕获**同步**错误。`async` 函数里 reject 的 Promise 没人接：

```javascript
exports.updateMovie = async (req, res, next) => {
  const movie = await Movie.findByIdAndUpdate(req.params.id, req.body, {...});
  // 如果 Promise reject：没人 catch → UnhandledPromiseRejection → 严重时整个进程崩掉
};
```

### 解法 1：手动 try/catch（啰嗦，不推荐）

```javascript
exports.createMovie = async (req, res, next) => {
  try {
    const movie = await Movie.create(req.body);
    res.status(201).json({ status: 'success', data: { movie } });
  } catch (err) {
    next(err);                                   // 转交给全局错误中间件
    // 或抛得更具体：next(new AppError(err.message, 400))
  }
};
```

![AppError 与 next(err)](img/course-err02-apperror.jpg)
*图 E2-01 · 视频 #93 · 控制器里 `next(new CustomError(...))` 的现场*

### 解法 2：`catchAsync` 包装器（课程采用的方案）

![catchAsync 包装器](img/course-err02-catchasync.jpg)
*图 E2-02 · 视频 #93 · utils/catchAsync.js：三层嵌套箭头函数逐层讲解*

```javascript
// utils/catchAsync.js —— 高阶函数：吃进控制器，吐出"带错误兜底"的控制器
module.exports = (fn) => {
  return (req, res, next) => {
    fn(req, res, next).catch(next);   // Promise reject 时自动 next(err)
  };
};

// 路由层使用：一行包装，全控制器免 try/catch
router.route('/:id')
  .patch(catchAsync(movieController.updateMovie))
  .delete(catchAsync(movieController.deleteMovie));
```

> [!success] 对比记忆
> | 方案 | 每个控制器要写 | 优点 |
> | :--- | :--- | :--- |
> | try/catch | 5-8 行样板 | 直白 |
> | `catchAsync(fn)` | 1 个包裹词 | 业务代码干净 |
> Express 5 已内建 Promise 自动转发（[官方文档](https://expressjs.com/en/guide/error-handling.html)），但课程项目和多数现存代码库仍是 Express 4 风格。

## 二、"查不到"与"查坏了"是两种错误（#94）

![NotFound 错误处理](img/course-err02-notfound.jpg)
*图 E2-03 · 视频 #94 · getMovie：`!movie` 时 `next(new AppError(..., 404))`*

```javascript
exports.getMovie = asyncError(async (req, res, next) => {
  const movie = await Movie.findById(req.params.id);

  if (!movie) {                                    // ID 合法但记录不存在 → 404
    return next(new AppError('Movie with that ID is not found!', 404));
  }
  res.status(200).json({ status: 'success', data: { movie } });
});
// 注意：不存在的 ID 若格式非法（如 12 位非 ObjectId），Mongoose 直接抛 CastError → 500
// 那类错误的统一处理在 [[05-错误处理/03-开发与生产错误与常见错误类型]]
```

> [!warning] `if (!movie)` 检查为什么不写在 Promise 后面也行？
> `findById` 正常 resolve 时 resolve 值就是 `null`（没查到），所以能同步判断；只有**查询本身失败**（连接断、CastError）才走 catch 通道。两种失败路径必须分开处理。

## 三、完整错误流（本板块的逻辑地图）

```text
async 控制器 ─(reject)─► catchAsync → next(err) ─┐
同步 throw ──────────────────────────────────────┤
兜底路由 next(AppError) ─────────────────────────┤
                                                  ▼
                                    全局错误中间件（后续几篇改造它：
                                    dev/prod 分支 + Cast/Dup/Validation 分类）
```

## 相关笔记

- 上一篇：[[05-错误处理/01-默认路由与全局错误处理|默认路由与全局错误处理]]
- 下一篇：[[05-错误处理/03-开发与生产错误与常见错误类型|开发与生产错误与常见错误类型]]
- 关联：[[01-Node.js基础/02-文件系统与异步IO|异步回调的 err-first 约定——错误处理思想一脉相承]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #93 | async 错误：try/catch vs catchAsync 包装器 |
| #94 | !movie 检查与 404 响应 |
