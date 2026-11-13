

> 来源：[编程狮（w3cschool.cn）笔记](https://www.w3cschool.cn/article/67311146.html) 2025-08-14

一文搞懂 npx 和 npm：零基础也能选对 JavaScript 包管理工具

> 如果你刚学 JavaScript，看到 `npm install` 和 `npx create-react-app` 就头大，那么这篇文章就是为你写的。跟着编程狮一步步来，5 分钟就能分清什么时候用 npm，什么时候用 npx。

## 一、npm 是谁？npx 又是谁？（大白话版）

| 工具      | 中文解释                              | 类比生活场景               |
| ------- | --------------------------------- | -------------------- |
| **npm** | Node 包管理器：帮你把别人写好的代码"下载"到本地，再统一管理 | 像超市购物车：先拿（下载）再结账（保存） |
| **npx** | Node 包执行器：临时"借"一个包来用，用完就还，不占地方    | 像点外卖：只在需要时下单，吃完不用洗碗  |


**一句话记忆**

- 想**长期用** → 用 npm
- 想**临时跑** → 用 npx

## 二、核心区别：3 张图秒懂

### ① 安装 vs 不安装

```bash
npm install create-react-app   # 把包搬进家（node_modules）
npx create-react-app my-app    # 只在门口租 1 分钟
```

### ② 执行方式

| 场景 | npm 写法 | npx 写法 |
| --- | --- | --- |
| 本地脚本 | `npm run dev` | — |
| 全局命令 | 需 `npm i -g xxx` | 直接 `npx xxx` |
| 指定版本 | 改 package.json | `npx xxx@1.2.3` |

### ③ 项目目录对比

```text
npm 安装后
└── 项目
    └── node_modules（体积可能几百 MB）

npx 运行后
└── 项目
    （无残留，node_modules 不会变大）
```

## 三、实战：跟着编程狮敲 3 个例子

### 1）用 npm 长期依赖

```bash
# 1. 进入你的练习文件夹
cd D:\编程狮练习\demo1

# 2. 把 lodash 这个工具库"搬进家"
npm install lodash
```

```js
// 3. 在 index.js 中使用
const _ = require('lodash');
console.log(_.chunk([1, 2, 3, 4], 2)); // 输出: [[1,2],[3,4]]
```

### 2）用 npx 临时脚手架

```bash
# 1. 不安装 create-next-app，直接生成项目
npx create-next-app@latest 我的第一个Next项目 --typescript

# 2. 进入目录并启动
cd 我的第一个Next项目
npm run dev
```

### 3）版本测试（不污染项目）

```bash
# 同时跑两个不同版本的 ESLint
npx eslint@7.32.0 app.js
npx eslint@8.57.0 app.js
```

## 四、什么时候选 npm？什么时候选 npx？

| 需求 | 推荐 | 原因 |
| --- | --- | --- |
| 项目核心依赖（如 Vue、React） | npm | 长期存在，版本锁定 |
| 构建工具（如 webpack） | npm | 需多次复用 |
| 一次性脚手架（如 create-react-app） | npx | 用完即走 |
| 体验最新功能而不升级 | npx | 不污染 node_modules |
| CI/CD 脚本 | npx | 减少镜像体积 |

## 五、常见坑 & 解决方案（来自编程狮问答区）

| 问题 | 原因 | 解决 |
| --- | --- | --- |
| `npx command not found` | Node < 8.2 | 升级 Node 或 `npm i -g npx` |
| npx 每次都重新下载 | 网络慢 | 配淘宝镜像 `npx --registry=https://registry.npmmirror.com xxx` |
| npm 全局包卸载不干净 | 路径混乱 | 用 `npm root -g` 找到目录手动删 |

## 六、3 条最佳实践（建议收藏）

1. **项目依赖写在 package.json**：用 `npm install xxx --save` 或 `--save-dev`，方便团队共享。
2. **脚手架一律 npx**：保持 `node_modules` 干净，减少磁盘占用。
3. **统一版本管理**：使用 `.nvmrc` 锁定 Node 版本，避免"我电脑能跑，你电脑报错"的尴尬。

## 七、总结（一句话背诵）

> **npm 是仓库管理员，npx 是快递员；仓库管理员长期保管，快递员随叫随到。**
