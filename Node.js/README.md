---
title: Node.js 完全课程导航
tags: [NodeJS, MOC, 课程笔记]
source: https://www.youtube.com/playlist?list=PL1BztTYDF-QPdTvgsjf8HOwO4ZVl_LhxS
created: 2026-11-13
---

# Node.js 完全课程导航

> [!info] 课程来源
> procademy《A Complete NODE JS Course》，共 **122 集 / 约 27 小时**，分 6 大板块。
> 本目录的笔记树与课程板块一一对应：**文件夹 = 课程板块，笔记 = 章节，`##` = 单集知识点**。
> 每篇笔记均附带 [Node.js 官方文档](https://nodejs.org/docs/latest/api/) 对应章节的链接。

## 学习路线图

```text
01-Node.js基础 ──► 02-Express ──► 03-MongoDB基础 ──► 04-Mongoose实战 ──► 05-错误处理 ──► 06-认证与授权
   (核心机制)        (Web框架)       (数据库入门)        (ODM建模)          (工程健壮性)      (安全闭环)
```

## 01-Node.js基础（#01–#29）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[01-Node.js概述与环境搭建]] | #01–#04 | Node 是什么、安装、REPL、第一个项目 |
| [[02-文件系统与异步IO]] | #05–#07 | 同步读写、异步本质、异步读写 |
| [[03-构建HTTP服务器]] | #08–#10 | createServer、Web 工作原理、请求响应模型 |
| [[04-路由与响应]] | #11–#14 | 路由概念、URL 路由、HTML 响应、响应头 |
| [[05-JSON与查询字符串]] | #15–#18 | JSON 处理、模板渲染、URL 解析、可复用函数 |
| [[06-模块化]] | #19 | CommonJS 模块、exports 与 require |
| [[07-事件驱动与Streams]] | #20–#24 | EventEmitter、可读/可写流、pipe |
| [[08-npm与包管理]] | #25–#26 | npm 基础、依赖类型与安装方式 |
| [[09-Node架构与事件循环]] | #27–#29 | V8/libuv、事件循环六阶段、实践验证 |

## 02-Express（#30–#48）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[01-Express入门与RESTful设计]] | #30–#33 | Express 基本流程、前后端分离、Web API、REST |
| [[02-CRUD接口实现]] | #34–#38 | GET / POST / 路由参数 / PATCH / DELETE |
| [[03-中间件机制]] | #39–#41 | 中间件概念、自定义中间件、第三方中间件 |
| [[04-路由模块化与静态文件]] | #42–#46 | Router 挂载、路由模块、param 中间件、静态资源 |
| [[05-环境变量与配置]] | #47–#48 | process.env、dotenv 配置文件 |

## 03-MongoDB基础（#49–#63）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[03-MongoDB基础/01-MongoDB入门与安装]] | #49–#50 | NoSQL 对比关系库、安装 MongoDB Server |
| [[03-MongoDB基础/02-数据库与集合操作]] | #51–#53 | 隐式创建、BSON vs JSON |
| [[03-MongoDB基础/03-文档CRUD操作]] | #54–#58 | insert/find/update/delete、游标 |
| [[03-MongoDB基础/04-数据类型与嵌入文档]] | #59–#61 | BSON 类型、嵌入文档、数组 |
| [[03-MongoDB基础/05-Compass与Atlas]] | #62–#63 | 图形化管理、免费云数据库 |

## 04-Mongoose实战（#64–#89）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[04-Mongoose实战/01-连接数据库与Schema与Model]] | #64–#66 | 连接库、Schema/Model、save |
| [[04-Mongoose实战/02-MVC架构与Express实现CRUD]] | #67–#71 | 分层、POST/GET/PATCH/DELETE 完整版 |
| [[04-Mongoose实战/03-高级查询与分页]] | #72–#81 | 过滤、排序、字段裁剪、分页、APIFeatures |
| [[04-Mongoose实战/04-聚合管道]] | #82–#83 | $match/$group、$unwind/$project |
| [[04-Mongoose实战/05-虚拟属性与中间件]] | #84–#87 | virtual、Document/Query/Aggregate 钩子 |
| [[04-Mongoose实战/06-数据验证器]] | #88–#89 | 内建验证器、自定义验证器 |

## 05-错误处理（#90–#101）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[05-错误处理/01-默认路由与全局错误处理]] | #90–#92 | 兜底路由、AppError、四参数中间件 |
| [[05-错误处理/02-异步错误处理与NotFound]] | #93–#94 | catchAsync 包装器、404 判断 |
| [[05-错误处理/03-开发与生产错误与常见错误类型]] | #95–#98 | dev/prod 响应、Cast/Duplicate/Validation |
| [[05-错误处理/04-全局异常捕获与调试]] | #99–#101 | unhandledRejection、uncaughtException、断点调试 |

## 06-认证与授权（#102–#122）

| 笔记 | 覆盖课程 | 主题 |
|:---|:---|:---|
| [[06-认证与授权/01-用户模型与密码加密]] | #102–#104 | User Schema、bcrypt 哈希与比对 |
| [[06-认证与授权/02-JWT原理与登录注册]] | #105–#107 | JWT 三段结构、signup/login |
| [[06-认证与授权/03-路由保护与权限]] | #108–#111 | protect 四步、restrictTo 角色权限 |
| [[06-认证与授权/04-密码重置与邮件发送]] | #112–#114 | resetToken、Nodemailer、重置流程 |
| [[06-认证与授权/05-账户自助管理]] | #115–#117 | updatePassword/Me、软删除 |
| [[06-认证与授权/06-安全加固]] | #118–#122 | Cookie、限流、helmet、消毒、hpp |

> [!note] 关于截图覆盖
> 板块 01–05 与板块 06 的 #102–#110 均配了视频截图（存于 `img/`）；#111–#122 的原始视频未成功获取，对应三篇笔记依据课程字幕 + 官方文档整理，无截图。

## 配套资料

- 官方文档：[Node.js API](https://nodejs.org/docs/latest/api/) · [Express](https://expressjs.com/) · [Mongoose](https://mongoosejs.com/docs/) · [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [[Node.js入门（Mosh一小时教程）]] —— 入门向快速版，可与本课程互为补充
