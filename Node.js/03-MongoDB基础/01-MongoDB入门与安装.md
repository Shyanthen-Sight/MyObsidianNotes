---
title: MongoDB 入门与安装
tags: [nodejs, mongodb, nosql, 数据库]
course: "A Complete NODE JS Course (#49-#50)"
created: 2026-11-13
---

# MongoDB 入门与安装

> [!abstract] 本篇定位
> 课程第三板块【Getting Started with MongoDB】（#49–#63）的第一篇。搞清楚三件事：**MongoDB 是什么、和 MySQL 有什么本质区别、怎么装到本机**。下一篇进入 [[03-MongoDB基础/02-数据库与集合操作|数据库与集合的实操]]。

## 一、为什么课程要插一个 MongoDB 板块

前两个板块（[[01-Node.js概述与环境搭建|Node 基础]]、[[01-Express入门与RESTful设计|Express]]）里，数据都存在**内存变量**里——`node app.js` 一重启，`courses` 数组就没了。真正的应用需要一个**持久化存储**。课程选 MongoDB 而不是 MySQL，因为它是 Node 生态最常见的组合（MongoDB → Mongoose → Express），而且 JSON 文档和 JS 对象天然对应，学习曲线平滑。

## 二、MongoDB 是什么：与关系型数据库对照

MongoDB 是 **NoSQL 文档型数据库**：不存"表 + 行 + 列"，而是存"数据库 → 集合 → 文档"。讲师用 MySQL/MSSQL/Oracle 对照讲解，对应关系如下：

| 关系型数据库 (MySQL) | MongoDB | 说明 |
| :--- | :--- | :--- |
| Database | Database | 顶层容器，概念相同 |
| Table（表） | **Collection（集合）** | 存相关数据的地方 |
| Row（行） | **Document（文档）** | 一条记录，BSON/JSON 对象 |
| Column（列） | Field（字段） | 文档里的键 |
| Primary Key | `_id` 字段 | MongoDB 自动生成的唯一标识 |
| JOIN | 嵌入文档 / `$lookup` | 设计思路完全不同，见 [[03-MongoDB基础/04-数据类型与嵌入文档|嵌入文档]] |
> [!important] 核心差异一句话
> 关系型数据库要求**先定义表结构再写数据**（schema-on-write）；MongoDB 的集合**不强制字段结构**，同一集合里的文档可以有不同字段（schema-on-read）。灵活性高，但数据质量靠应用层把关——这正是后面 [[04-Mongoose实战/01-连接数据库与Schema与Model|Schema/Model 验证]] 存在的意义。

**课程给出的 MongoDB 特点**：
- 文档模型：数据以 JSON-like 文档存储，与 JS 对象互转零成本
- 无固定 schema：字段可动态增减
- 水平扩展友好：分片（sharding）内建
- 查询语言是**方法调用**（`db.col.find({...})`）而非 SQL 语句

## 三、安装 MongoDB Server（Windows）

> 官方安装文档：[Install MongoDB Community Edition on Windows](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-windows/)（macOS/Linux 各有对应页面，从文档首页左侧切换）。

![安装页选择部署方式](img/course-m01-deploy-choice.jpg)
*图 M1-01 · 视频 #50 · 官网下载页：Atlas（云托管）/ On-premises（本地安装）/ Tools 三类*

课程安装步骤（版本 6.0 时代流程，官网当前版本流程一致）：

1. 打开 [mongodb.com](https://www.mongodb.com/) → Products → **Community Edition**（社区版免费，学习/开发够用）
2. 选 **Community Server**，下载 MSI 安装包
3. 安装时勾选：
   - **MongoDB Server** —— 数据库本体
   - **MongoDB Compass** —— 官方 GUI 客户端（勾上，后面 [[03-MongoDB基础/05-Compass与Atlas|Compass 篇]] 直接用）
4. 安装完成后 MongoDB 作为 **Windows 服务**（服务名 `MongoDB`）自动在后台运行，默认端口 **27017**
5. 验证：`services.msc` 里看到 MongoDB 服务处于 Running

![Windows 服务列表中的 MongoDB](img/course-m01-windows-service.jpg)
*图 M1-02 · 视频 #50 · 服务面板：MongoDB Server (MongoDB) 正在运行*

> [!tip] shell 与 server 是两回事
> Server 是存数据的进程；**mongosh**（MongoDB Shell）是连接它干活用的客户端命令行。装完 server 后建议单独装/升级 [mongosh](https://www.mongodb.com/docs/mongodb-shell/install/)，老式的 `mongo` shell 已弃用。

## 四、客户端连接验证

打开 mongosh：

```bash
mongosh
# → 连接 mongodb://127.0.0.1:27017
# 提示符变为：
test>
```

看到 `test>` 提示符即安装成功。`test` 是**默认选中**的数据库（不存在也没关系，见下篇"隐式创建"）。

> [!note] 没装本机版也能学
> 后面 [[03-MongoDB基础/05-Compass与Atlas#二、Atlas：免费云数据库（#63）|Atlas 云数据库]] 提供免费集群，连接串一换、命令完全一样。课程 #63 会专门讲。

## 五、官方文档导航（本板块常用）

| 内容 | 链接 |
| :--- | :--- |
| MongoDB 手册（概念/操作总入口） | <https://www.mongodb.com/docs/manual/> |
| Windows 安装 | <https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-windows/> |
| mongosh 文档 | <https://www.mongodb.com/docs/mongodb-shell/> |
| CRUD 操作 | <https://www.mongodb.com/docs/manual/crud/> |
| BSON 类型总表 | <https://www.mongodb.com/docs/manual/reference/bson-types/> |

## 相关笔记

- 下一篇：[[03-MongoDB基础/02-数据库与集合操作|数据库与集合操作]]
- 回顾：[[01-Node.js基础/01-Node.js概述与环境搭建|为什么内存变量不能当数据库]]
- 后续：[[04-Mongoose实战/01-连接数据库与Schema与Model|用 Mongoose 把 MongoDB 接进 Express]]

## 附：本板块课程索引

| 集数 | 主题 | 对应笔记 |
| :--- | :--- | :--- |
| #49 | MongoDB 介绍 | 本篇 |
| #50 | Windows 安装 | 本篇 |
| #51–#53 | 数据库 / 集合 / BSON | [[03-MongoDB基础/02-数据库与集合操作]] |
| #54–#58 | 增删改查 + 游标 | [[03-MongoDB基础/03-文档CRUD操作]] |
| #59–#61 | 数据类型 / 嵌入 / 数组 | [[03-MongoDB基础/04-数据类型与嵌入文档]] |
| #62–#63 | Compass / Atlas | [[03-MongoDB基础/05-Compass与Atlas]] |
