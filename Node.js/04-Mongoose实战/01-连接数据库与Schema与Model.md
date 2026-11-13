---
title: 连接数据库与 Schema/Model
tags: [nodejs, mongoose, schema, model]
course: "A Complete NODE JS Course (#64-#66)"
created: 2026-11-13
---

# 连接数据库与 Schema/Model

> [!abstract] 本篇定位
> 第四板块【Using MongoDB with Express】开篇（#64–#66）。原生 driver 够用但太底层，**Mongoose** 是事实标准的 ODM：把上一板块 [[03-MongoDB基础/04-数据类型与嵌入文档|灵活无 schema 的集合]] 收编成**有 schema 约束的模型**。

## 一、安装与连接（#64）

```bash
npm i mongoose
```

![连接代码与 .env](img/course-mg01-connect-code.jpg)
*图 G1-01 · 视频 #64 · `mongoose.connect(process.env.MONGO_URI)` 与 .env 中的连接串*

![连接事件监听](img/course-mg01-conn-events.jpg)
*图 G1-02 · 视频 #64 · connection 对象的事件回调与启动顺序*

```javascript
const mongoose = require('mongoose');
const config = require('config');          // 课程用 config 包管理环境配置

mongoose
  .connect(process.env.MONGO_URI)          // 连接串来自 [[02-Express/05-环境变量与配置|环境变量]]
  .then(() => console.log('DB Connection Successful'))
  .catch((err) => console.log(`Some error has occured: ${err}`));

const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Server has started...`));
```

连接串三种来源（对应 [[03-MongoDB基础/05-Compass与Atlas|上板块的 Compass/Atlas]]）：
- 本机：`mongodb://127.0.0.1:27017/cinefix`
- Atlas 云：`mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/cinefix`
- 课程示例用 `config` 包：`config.get('db.uri')`，生产配 `production.json`

> [!tip] 连接事件比 .then 更可靠
> 长连接会断线重连，`mongoose.connection` 是个 EventEmitter：
> ```javascript
> const db = mongoose.connection;
> db.on('error',   (err) => console.error(err));        // 持续监听错误
> db.once('open',  ()    => console.log('connected'));   // 首次连上
> db.on('disconnected', () => console.log('disconnected'));
> ```
> 官方文档：[Mongoose Connections](https://mongoosejs.com/docs/connections.html)

## 二、Schema：给集合"立规矩"（#65）

![Schema 定义代码](img/course-mg01-schema-code.jpg)
*图 G1-03 · 视频 #65 · MovieSchema：字段类型、required、trim、default*

原生 MongoDB 里同集合文档可以五花八门；Schema 把结构**写死在代码里**：

```javascript
const mongoose = require('mongoose');

const movieSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, "Name is required field!"],   // 自定义校验消息
    trim: true,                                     // 存前去首尾空格
  },
  description: { type: String },
  releaseDate: { type: Date },                      // BSON 的 Date 类型
  ratings: {
    type: Number,
    default: 5.0,                                   // 未提供时取默认值
    min: [1, 'Ratings must be above 1.0'],
    max: [1, 'Ratings must be below 1.0'],          // 课程示例里故意写的演示值
  },
  createdAt: { type: Date, default: Date.now },
});

const Movie = mongoose.model('Movie', movieSchema, 'movies'); // 第3参=实际集合名
```

三个参数的坑：`mongoose.model('Movie', schema)` 默认把集合名**小写复数化**为 `movies`；第三参数显式指定更保险。

SchemaType 支持的常用选项见官方 [Schematypes](https://mongoosejs.com/docs/schematypes.html)：`type / required / default / trim / lowercase / min / max / enum / validate`…（验证器专篇见 [[04-Mongoose实战/06-数据验证器]]）。

## 三、Model → Document：创建与保存（#66）

![create 与 save 运行输出](img/course-mg01-save-output.jpg)
*图 G1-04 · 视频 #66 · `movie.save()` 的 Promise 输出与校验失败报错*

Model 是"类"，`new` 出来的实例是"文档对象"：

```javascript
const testMovie = new Movie({
  name: 'Titanic',
  description: 'A movie about the sinking of the famous ship',
});

testMovie.save()                       // 异步写库，返回 Promise
  .then((doc) => console.log(doc))     // → 带上 _id / createdAt / ratings: 5 的完整文档
  .catch((err) => console.log(err));   // 校验失败 → ValidationError
```

| 方式 | 场景 |
| :--- | :--- |
| `new Model({...})` + `.save()` | 需要先在内存里改字段/挂方法，再落库 |
| `Model.create({...})` | 一步到位（内部就是 new + save） |

> [!important] Schema 的意义（承上启下）
> 上一板块说"无 schema 灵活"，这一板块又"立规矩"——并不矛盾：**数据库层面灵活（可以存任何文档），应用层面用 Mongoose 管住写入的口子**。`save()` 时自动跑 required/min/max/自定义验证，脏数据进不了库。

## 相关笔记

- 上一篇：[[03-MongoDB基础/05-Compass与Atlas|Compass 与 Atlas（连接串来源）]]
- 下一篇：[[04-Mongoose实战/02-MVC架构与Express实现CRUD|MVC 架构与 Express 实现 CRUD]]
- 关联：[[01-Node.js基础/06-模块化|模块化——MVC 各层靠 module.exports 拆分]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #64 | 安装 Mongoose、连接远程/本地库 |
| #65 | 创建 Schema 与 Model |
| #66 | 从 Model 创建文档并 save |
