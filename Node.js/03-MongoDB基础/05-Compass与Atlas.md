---
title: Compass 与 Atlas
tags: [nodejs, mongodb, compass, atlas]
course: "A Complete NODE JS Course (#62-#63)"
created: 2026-11-13
---

# MongoDB Compass 与 Atlas 云数据库

> [!abstract] 本篇定位
> 覆盖课程 #62–#63，第三板块收官。两件事：**Compass**——官方 GUI，肉眼检查数据的好帮手；**Atlas**——免费云托管 MongoDB，不需要本机装服务。之后 [[04-Mongoose实战/01-连接数据库与Schema与Model|Mongoose 篇]] 的连接串就来自这里。

## 一、MongoDB Compass：图形化客户端（#62）

![Compass 界面](img/course-m02-compass.jpg)
*图 M5-01 · 视频 #62 · Compass 连接本地实例并浏览集合文档*

Compass（装 server 时已随附）解决 mongosh 的两大痛点：**看不清结构、容易敲错命令**。课程演示的主线：

1. **连接**：打开 Compass → 默认连接串 `mongodb://localhost:27017` → Connect
2. **浏览**：左侧数据库列表 → 点 `sampleDB` → 点 `customer` 集合 → 表格/JSON 两种视图看文档
3. **查询**：顶部过滤栏直接填 mongosh 同款过滤条件（如 `{ "subscription.type": "monthly" }`）→ 点路径查询在这里尤其直观
4. **文档级操作**：点单条文档右上角铅笔图标可**可视化编辑**字段——比 shell 里拼 JSON 不容易错
5. **Schema / 索引 / 聚合**：左侧 Schema 标签能自动采样分析字段分布；Aggregations 标签是搭 [[04-Mongoose实战/06-聚合管道|聚合管道]] 的可视化面板

> [!tip] 什么时候用 Compass 什么时候用 shell
> - **探索、检查、改单条脏数据** → Compass
> - **批量操作、写脚本、要进 git 的操作记录** → mongosh
> 官方文档：[Compass Docs](https://www.mongodb.com/docs/compass/)

## 二、Atlas：免费云数据库（#63）

![Atlas 控制台](img/course-m02-atlas.jpg)
*图 M5-02 · 视频 #63 · Atlas 创建集群与连接信息页*

Atlas 是 MongoDB 官方的云托管服务，**M0 免费层**（512MB）学习完全够用，且省掉本机装服务的步骤——云服务器部署项目时尤其有用（Node 应用在哪，数据库连哪）。

课程流程（官方指引：[Create & Connect to a Cluster](https://www.mongodb.com/docs/atlas/getting-started/)）：

1. 注册 [cloud.mongodb.com](https://cloud.mongodb.com) → 创建免费 **M0 集群**（选离你近的 Region）
2. **数据库用户**：Security → Database Access → 建用户名 + 密码（这组凭据后面连接串要用）
3. **网络白名单**：Network Access → 添加 IP；开发期直接 `0.0.0.0/0`（允许所有，仅限学习环境）
4. **取连接串**：集群页 Connect → Drivers → 复制 `mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/...` 形式的 SRV 连接串
5. 用它替换本地串验证：
   ```bash
   mongosh "mongodb+srv://cluster0.xxxxx.mongodb.net/sampleDB" --username youruser
   ```
   `mongodb+srv://` 是 DNS 种子格式，客户端自动解析集群节点；**尾部的数据库名和参数（如 `retryWrites=true`）别丢**。

> [!warning] 连接串里的密码要 URL 编码
> 密码含 `@ : / ? #` 等字符时必须转义（如 `@` → `%40`），否则连接串解析失败——这是 Atlas 连不上的头号原因。

> [!important] 三种环境，一条连接串
> - 本机 server：`mongodb://127.0.0.1:27017/sampleDB`
> - Atlas：`mongodb+srv://user:pass@cluster0.xxx.mongodb.net/sampleDB`
> - 共同点：切换只是**换字符串**，代码零改动。[[02-Express/05-环境变量与配置|环境变量]] 里存 `MONGODB_URI`，开发和生产各配一份——课程 #64 正是这么把它接进 Express 的。

## 相关笔记

- 上一篇：[[03-MongoDB基础/04-数据类型与嵌入文档|数据类型与嵌入文档]]
- 下一板块开篇：[[04-Mongoose实战/01-连接数据库与Schema与Model|用 Mongoose 连接数据库]]
- 关联：[[02-Express/05-环境变量与配置|为什么连接串要放进环境变量]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #62 | Compass 安装使用：浏览/过滤/编辑 |
| #63 | Atlas 免费集群、用户与白名单、SRV 连接串 |
