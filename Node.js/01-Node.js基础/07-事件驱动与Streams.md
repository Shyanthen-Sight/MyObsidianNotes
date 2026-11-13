---
title: 事件驱动与 Streams
tags: [NodeJS, EventEmitter, Stream, pipe, backpressure]
course: "A Complete NODE JS Course #20-#24"
created: 2026-11-13
官方文档:
  - "[events 模块](https://nodejs.org/docs/latest/api/events.html)"
  - "[stream 模块](https://nodejs.org/docs/latest/api/stream.html)"
---

# 07 - 事件驱动与 Streams

> [!info] 本篇覆盖
> 课程 #20 事件驱动架构、#21 自定义事件、#22 流的概念、#23 流的实践、#24 pipe。
> 一句话目标：**掌握 Node 的两大核心机制——事件驱动（一切交互的基础）与流（处理大数据的利器）。**

---

## 一、事件驱动架构（#20）

### 1.1 核心模型：观察者模式

![[img/course-07-event-flow.jpg]]
*图 07-1 · 00:01:38 · Event Emitter → Emit → Listener → Handler*

事件驱动由四个角色构成：

| 角色 | 通俗理解 | Node 中的对应 |
|:---|:---|:---|
| Event Emitter | 广播站 | `EventEmitter` 类的实例 |
| Emit event | 发广播 | `emitter.emit('事件名', 参数)` |
| Event Listener | 收音机 | `emitter.on('事件名', 处理函数)` |
| Event Handler | 收到广播后做的事 | 那个回调函数 |

> [!tip] 顺序很重要
> **必须先 `on`（订阅）再 `emit`（发布）**——先发广播，收音机还没开机，什么也收不到。`emit` 是同步调用的：触发时立即依次执行所有监听器。

### 1.2 Node 里到处都是 EventEmitter

![[img/course-03-createserver.jpg]]
*图 03-1 · `server.on('request')` 与 `rl.on('close')` 你早已用过*

- `http.Server` 的 `request` 事件
- `readline` 接口的 `close` / `line` 事件
- `fs.ReadStream` 的 `data` / `end` 事件（马上见到）
- Express 的 `app.listen` 返回值同样继承自 EventEmitter

可以说：**学会了 events 模块，就学会了 Node 一半的 API 设计语言**。

### 1.3 自定义事件（#21）

![[img/course-07-emit-args.jpg]]
*图 07-2 · 00:07:57 · 自定义 userCreated 事件并携带参数*

```js
const EventEmitter = require('events');
const myEmitter = new EventEmitter();

// 订阅：监听 userCreated 事件
myEmitter.on('userCreated', (name, id) => {
  console.log(`A new user ${name} with ID ${id} is created!`);
});

myEmitter.on('userCreated', () => {
  console.log('A new user is created in database!');
});

// 发布：可以携带任意参数
myEmitter.emit('userCreated', 'Leo', 15);
```

```text
A new user Leo with ID 15 is created!
A new user is created in database!
```

课程里还演示了实战组合：**把"写文件到 db.txt"封装进监听器**——业务逻辑被拆成一个个事件处理器，主流程只负责 emit：

```js
server.on('request', (req, res) => {
  res.end('Hello');
  myEmitter.emit('userCreated', 'Leo');   // 顺带触发日志写入等副作用
});
```

这种"主流程只发事件、细节交给监听器"正是解耦的关键。

---

## 二、Streams：为什么要流（#22）

### 2.1 传统读法的死穴

`fs.readFile` 是**一次性读完整个文件**再给你数据。读 1GB 的视频文件：

1. 内存先吃 1GB
2. 磁盘读完整段才开始处理
3. 用户全程等待

### 2.2 流 = 分块处理

![[img/course-07-stream-flow.jpg]]
*图 07-3 · 00:01:50 · 数据从源文件分块流向目标文件*

流把数据切成小块（chunk）**边读边处理边写**：内存峰值只有一块的大小，且读和写可以并行。

### 2.3 四种流类型

![[img/course-07-readable-stream.jpg]]
*图 07-4 · Readable Stream：事件 data & end，方法 read & pipe*

![[img/course-07-writable-stream.jpg]]
*图 07-5 · Writable Stream：事件 drain & finish，方法 write & end*

| 类型 | 方向 | 典型例子 | 关键事件 | 关键方法 |
|:---|:---|:---|:---|:---|
| **Readable** | 读 | `fs.createReadStream`、`request` | `data`、`end` | `read`、`pipe` |
| **Writable** | 写 | `fs.createWriteStream`、`response` | `drain`、`finish` | `write`、`end` |
| **Duplex** | 双向 | TCP socket | 上述事件兼有 | 上述方法兼有 |
| **Transform** | 双向+变换 | `zlib.createGzip()` 压缩流 | 同 Duplex | `_transform` |

完整 API 见[官方 Stream 文档](https://nodejs.org/docs/latest/api/stream.html)。

---

## 三、流的实践（#23）

课程场景：**读一个超大文件，边读边写回响应**。

![[img/course-07-stream-data-event.jpg]]
*图 07-6 · 00:09:02 · createReadStream + data 事件*

```js
const rs = fs.createReadStream('./files/large-file.txt', 'utf-8');

rs.on('data', (chunk) => {
  console.log(chunk);              // 每一块数据到达时触发
  response.write(chunk);           // 边收边发给客户端
});

rs.on('end', () => {
  response.end();                  // 读完，结束响应
});
```

对比 `readFileSync`：**第一个 chunk 到达时客户端就开始收到数据了**，内存只占用一个 chunk。

---

## 四、背压与 pipe（#24）

### 4.1 背压（Backpressure）问题

`response.write(chunk)` 有可能比 `rs` 读得**慢**（网络慢时尤其明显）——读进来的 chunk 堆积在内存里等写完，**背压**就出现了。

手动处理：

![[img/course-07-stream-backpressure.jpg]]
*图 07-7 · 00:01:22 · 手动处理背压：drain 事件*

```js
const ws = fs.createWriteStream('./files/target-file.txt');

rs.on('data', (chunk) => {
  if (!ws.write(chunk)) {          // write 返回 false = 写缓冲满了
    rs.pause();                    // 暂停读
    ws.once('drain', () => {       // 缓冲排空后
      rs.resume();                 // 恢复读
    });
  }
});
rs.on('end', () => ws.end());
```

### 4.2 pipe：一行搞定

```js
rs.pipe(ws);    // 自动处理 data / end / drain，还有错误传播
```

`pipe` 把可读流接到可写流，背压自动处理。HTTP 场景：

```js
const rs = fs.createReadStream('./files/large-file.txt');
rs.pipe(response);        // 一行替代上一章的 data/end 监听
```

> [!tip] 现代写法补充
> Node 17+ 推荐 `stream.pipeline()` 或 `Readable.fromX().pipe()`，能正确处理**错误传播**（`pipe` 遇错不会自动销毁另一端）。Web Streams 标准 API 也已进入 Node，见[官方文档](https://nodejs.org/docs/latest/api/stream.html#api-for-stream-implementers)。原理与课程相同，工程上优先 `pipeline`。

---

## 五、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| 先 emit 后 on | 监听器永远不执行 | 先订阅再发布 |
| 大文件用 readFile | 内存暴涨、响应慢 | createReadStream + pipe |
| 手写背压处理 | 容易漏掉 drain | 直接 pipe / pipeline |
| 流出错没监听 `error` | 进程崩溃 | 所有流都加 `on('error')` |

> [!summary] 自查清单
> - [ ] 能画出发射-订阅模型的四个角色
> - [ ] 会自定义事件并携带参数
> - [ ] 说出四种流类型与各自的关键事件/方法
> - [ ] 能解释背压和 pipe 为什么存在

---
**相关笔记**：[[06-模块化]] · [[09-Node架构与事件循环]] · [[04-Mongoose实战/02-MVC架构与Express实现CRUD]] · [[README|课程导航]]
