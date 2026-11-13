---
title: 文档 CRUD 操作（mongosh）
tags: [nodejs, mongodb, crud, cursor]
course: "A Complete NODE JS Course (#54-#58)"
created: 2026-11-13
---

# 文档 CRUD 操作（mongosh）

> [!abstract] 本篇定位
> 覆盖课程 #54–#58：在 mongosh 里完成 **Create / Read / Update / Delete** 四类操作，并理解 `find()` 返回的**游标（Cursor）**。这五个操作和后面 [[04-Mongoose实战/02-MVC架构与Express实现CRUD|Mongoose 的 CRUD 方法]] 一一对应，先在 shell 里打熟手感。

## 一、Create：插入文档

![insertOne 演示](img/course-m01-insert.jpg)
*图 M3-01 · 视频 #54 · `db.customer.insertOne({...})` 及返回的 acknowledged/insertedId*

```javascript
use sampleDB

// 插入一条
db.customer.insertOne({ name: "John", age: 25, city: "New York" })
// → { acknowledged: true, insertedId: ObjectId("...") }

// 批量插入：数组包多个文档
db.customer.insertMany([
  { name: "Steve", age: 30, city: "London" },
  { name: "Amy",   age: 22, city: "Sydney"  }
])
// → { acknowledged: true, insertedIds: { '0': ObjectId(...), '1': ObjectId(...) } }
```

要点：
- `_id` 不写则自动生成 `ObjectId`；写了就用你的（但全集合必须唯一，重复报 `E11000`）
- 旧方法 `insert()` 已弃用，用 `insertOne` / `insertMany`
- 官方文档：[Insert Documents](https://www.mongodb.com/docs/manual/tutorial/insert-documents/)

![insertMany 演示](img/course-m01-insertmany.jpg)
*图 M3-02 · 视频 #52 · `insertMany` 一次写入多条*

## 二、Read：查询文档

![findOne 与 find 过滤](img/course-m01-find.jpg)
*图 M3-03 · 视频 #55 · `findOne({filter})` 与 `find({filter})` 的区别*

```javascript
db.customer.findOne()                    // 第一条文档（只返回一个对象）
db.customer.findOne({ name: "Steve" })   // 按过滤条件返回第一条匹配

db.customer.find()                       // 全部文档 → 游标
db.customer.find({ city: "London" })     // 全部匹配文档

// 带投影：只要 name 和 age（_id 默认总在，想排除写 0）
db.customer.find({}, { name: 1, age: 1, _id: 0 })
```

> [!tip] findOne vs find 的返回类型
> - `findOne` → **文档对象本身**（或 `null`）
> - `find` → **游标**（可迭代、可 `toArray()`、可链式 `.sort()/.limit()`）
> 后面在 Express 里写 `Model.find()` 时代码几乎一样，只是返回 Promise。

### 常用查询操作符

```javascript
db.customer.find({ age: { $gte: 25 } })            // 比较：$gte $lte $gt $lt $ne
db.customer.find({ city: { $in: ["London", "Sydney"] } })  // $in 在列表内
db.customer.find({ $and: [{ age: { $gte: 25 } }, { city: "London" }] })
db.customer.find({ $or:  [{ city: "London" }, { age: { $lt: 25 } }] })
```

官方文档：[Query Documents](https://www.mongodb.com/docs/manual/tutorial/query-documents/) · [操作符总表](https://www.mongodb.com/docs/manual/reference/operator/query/)

## 三、Update：更新文档

![updateOne 演示](img/course-m01-update.jpg)
*图 M3-04 · 视频 #56 · `updateOne` 配合更新操作符*

```javascript
// 只改匹配到的第一条；$set 是更新操作符
db.customer.updateOne({ name: "John" }, { $set: { age: 26 } })
// → { acknowledged: true, matchedCount: 1, modifiedCount: 1 }

// 改所有匹配的
db.customer.updateMany({ city: "London" }, { $set: { verified: true } })

// 整个替换文档（不常用，会丢掉未提及的字段）
db.customer.replaceOne({ name: "John" }, { name: "John", age: 26 })
```

> [!danger] 忘写 `$set` = 整文档被替换
> `db.customer.updateOne({name:"John"}, { age: 26 })` 会把文档替换成只有 `{_id, age}`——初学者第一大坑。更新操作符速查：
> - `$set` 设置字段值
> - `$inc` 数值自增/自减（`{ $inc: { age: -1 } }`）
> - `$push` / `$pull` 往数组追加/移除（见 [[03-MongoDB基础/04-数据类型与嵌入文档|数组篇]]）
> - `$unset` 删除字段

官方文档：[Update Documents](https://www.mongodb.com/docs/manual/tutorial/update-documents/)

## 四、Delete：删除文档

![deleteOne/deleteMany 演示](img/course-m01-delete.jpg)
*图 M3-05 · 视频 #57 · 删除操作与删除计数*

```javascript
db.customer.deleteOne({ name: "Amy" })          // 删第一条匹配
db.customer.deleteMany({ age: { $lt: 20 } })    // 删所有匹配
db.customer.deleteMany({})                      // 清空集合（集合本身还在）
```

> [!warning] 删除前先 `find` 一遍确认过滤条件
> 没有"回收站"。课程反复强调：把 `deleteMany` 的条件先放进 `find()` 跑一次，看返回的确实是你想删的，再换回 `deleteMany`。

官方文档：[Delete Documents](https://www.mongodb.com/docs/manual/tutorial/remove-documents/)

## 五、Cursor：`find()` 返回的游标

![游标遍历演示](img/course-m01-cursor.jpg)
*图 M3-06 · 视频 #58 · mongosh 里游标的批量返回与迭代*

`find()` 不立即取回所有结果，而是返回**游标**——一次拉一小批（shell 默认每批 20 条，敲 `it` 看下一批）：

```javascript
const cursor = db.customer.find()

cursor.forEach(doc => print(doc.name))   // 逐条迭代
cursor.toArray()                          // 一次性转数组

// 链式方法：都作用在游标上
db.customer.find().sort({ age: -1 }).limit(3)    // 年龄最大的 3 人
db.customer.find().skip(2).limit(2)              // 跳过前2条取2条（分页雏形）
db.customer.find().count()                       // 计数
```

> [!important] 游标是"分页/排序"的原点
> `sort / skip / limit` 这三件套就是 REST API 分页的底层逻辑，课程 #77–#79 会在 Express 里重演，见 [[04-Mongoose实战/03-高级查询与分页]]。
> 官方文档：[Iterate a Cursor](https://www.mongodb.com/docs/manual/tutorial/iterate-a-cursor/)

## 六、CRUD ↔ Mongoose ↔ REST 对照（先建立映射感）

| mongosh | Mongoose（后面学） | REST 语义 |
| :--- | :--- | :--- |
| `insertOne / insertMany` | `Model.create()` / `insertMany()` | `POST` |
| `findOne / find` | `Model.findOne() / Model.find()` | `GET` |
| `updateOne / updateMany` | `Model.findByIdAndUpdate()` | `PUT / PATCH` |
| `deleteOne / deleteMany` | `Model.findByIdAndDelete()` | `DELETE` |

## 相关笔记

- 上一篇：[[03-MongoDB基础/02-数据库与集合操作|数据库与集合操作]]
- 下一篇：[[03-MongoDB基础/04-数据类型与嵌入文档|数据类型与嵌入文档]]
- 前文回顾：[[02-Express/02-CRUD接口实现|Express 的 CRUD（内存版）]]

## 附：本篇课程索引

| 集数 | 主题 |
| :--- | :--- |
| #54 | 插入文档（insertOne / insertMany） |
| #55 | 查询文档（findOne / find + 过滤） |
| #56 | 更新文档（updateOne / updateMany + 操作符） |
| #57 | 删除文档 |
| #58 | 游标与迭代 |
