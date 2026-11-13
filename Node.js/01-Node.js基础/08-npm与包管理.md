---
title: npm 与包管理
tags: [NodeJS, npm, package.json, 依赖]
course: "A Complete NODE JS Course #25-#26"
created: 2026-11-13
官方文档:
  - "[npm 官网](https://www.npmjs.com/)"
  - "[package.json 字段说明](https://docs.npmjs.com/cli/v10/configuring-npm/package-json)"
---

# 08 - npm 与包管理

> [!info] 本篇覆盖
> 课程 #25 什么是 npm、#26 包的类型与安装方式。
> 一句话目标：**理解 npm 生态、初始化 package.json、分清依赖类型与安装命令。**

---

## 一、npm 是什么（#25）

![[img/course-08-npmjs-website.jpg]]
*图 08-1 · 00:01:23 · npmjs.com——超过两百万个包的仓库*

**npm = Node Package Manager**，两件事：

1. **npmjs.com**：存放全世界开源包的中央仓库
2. **npm CLI**：随 Node 安装的命令行工具，负责装包、卸包、跑脚本

别人写好的轮子（处理日期、连接数据库、压缩图片……）一行命令就能装进项目。

---

## 二、初始化项目：package.json（#25）

### 2.1 npm init

![[img/course-08-npm-init.jpg]]
*图 08-2 · 00:06:32 · npm init 交互式问答与生成的 package.json*

```bash
$ npm init          # 交互式问答；npm init -y 直接接受全部默认值
```

生成的 **package.json 是项目的"身份证 + 依赖账本"**：

```json
{
  "name": "node-js-basic",
  "version": "1.0.0",
  "description": "The complete basics of node js",
  "main": "app.js",
  "scripts": { "start": "node app.js" },   // 自定义命令：npm start
  "keywords": [],
  "license": "ISC"
}
```

| 字段 | 说明 |
|:---|:---|
| `name` / `version` | 包名与版本（发布 npm 时唯一） |
| `main` | 入口文件（别人 `require` 你的包时加载谁） |
| `scripts` | 自定义命令，`npm run xxx` 执行 |
| `dependencies` | 生产依赖账本，见下节 |

### 2.2 scripts 的价值

```json
"scripts": { "dev": "nodemon app.js", "start": "node app.js" }
```

```bash
npm run dev      # 开发热重载
npm start        # 约定特例：start 可省略 run
```

---

## 三、依赖类型与安装方式（#26）

### 3.1 两类依赖

| 类型 | 定义 | 安装命令 | 存放字段 |
|:---|:---|:---|:---|
| **Regular（生产依赖）** | 应用运行必需 | `npm install express` | `dependencies` |
| **Development（开发依赖）** | 只在开发期用（热重载、测试、格式化） | `npm install --save-dev nodemon`（简写 `-D`） | `devDependencies` |

判断口诀：==打包部署到服务器之后还需要它吗？需要 → 生产依赖；只是开发时舒服 → 开发依赖。== 详见 [[开发依赖和生产依赖]]。

### 3.2 实际安装

![[img/course-08-package-json.jpg]]
*图 08-3 · 00:09:08 · package.json 中同时记录 dependencies 与 devDependencies*

```bash
$ npm install express          # 7 packages，加入 dependencies
$ npm install nodemon          # 30 packages，课程顺手装成生产依赖（见下方勘误）
```

安装后项目多出两样东西：

```text
├── node_modules/          # 所有包的实际代码（巨大，永不提交 git）
├── package-lock.json      # 锁定每个包的精确版本号
└── package.json           # 依赖账本
```

> [!warning] 课程勘误：nodemon 应该是开发依赖
> 视频里 nodemon 被装进了 `dependencies`——它只在开发时用，**正确做法是 `npm install -D nodemon`**。不影响学习，但这个习惯要摆正。

### 3.3 lock 文件与协作

- **package-lock.json** 锁定依赖树每一层的精确版本，保证"我电脑上能跑"在任何机器都一致
- 团队协作：**提交 package.json + package-lock.json，.gitignore 掉 node_modules**；别人拿到代码后 `npm install` 一键还原

### 3.4 常用命令全景

| 命令 | 作用 |
|:---|:---|
| `npm install <pkg>` | 装 + 写入 dependencies |
| `npm install -D <pkg>` | 装 + 写入 devDependencies |
| `npm uninstall <pkg>` | 卸载并从账本移除 |
| `npm update` | 更新到允许范围内的最新版 |
| `npm install` | 按 package.json 还原全部依赖 |
| `npm ls --depth=0` | 查看直接依赖树 |

版本号语义（`^1.2.3` / `~1.2.3` / `1.2.3`）详见 [[开发依赖和生产依赖]]。

---

## 四、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| node_modules 传上了 git | 仓库几个 GB | .gitignore 加 `node_modules/` |
| 只提交 package.json 不提交 lock | 队友装出不同版本 | 两个都提交 |
| 全局安装 `npm i -g` 却没配 PATH | 命令找不到 | 重开终端 / `npx` 替代，见 [[npm和npx有什么区别]] |
| 删了包但忘删账本 | require 报错 | 用 `npm uninstall` 而非手动删文件夹 |

> [!summary] 自查清单
> - [ ] 会 npm init 并说明 package.json 关键字段
> - [ ] 分得清两类依赖与对应安装命令
> - [ ] 知道 node_modules / package-lock 的角色
> - [ ] 会用 scripts 自定义命令

---
**相关笔记**：[[开发依赖和生产依赖]] · [[npm和npx有什么区别]] · [[01-Express入门与RESTful设计]] · [[README|课程导航]]
