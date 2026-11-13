---
title: JSON 与查询字符串
tags: [NodeJS, JSON, URL, 重构]
course: "A Complete NODE JS Course #15-#18"
created: 2026-11-13
官方文档:
  - "[URL API](https://nodejs.org/docs/latest/api/url.html)"
  - "[querystring 模块](https://nodejs.org/docs/latest/api/querystring.html)"
---

# 05 - JSON 与查询字符串

> [!info] 本篇覆盖
> 课程 #15 JSON 数据、#16 JSON→HTML、#17 解析查询字符串、#18 提取可复用函数。
> 一句话目标：**让同一份 JSON 数据既能喂浏览器（HTML）也能喂前端应用（JSON API），并学会从 URL 里取参数。**

---

## 一、JSON 是 API 的通用语言（#15）

![[img/course-05-json-data.jpg]]
*图 05-1 · 00:02:01 · 课程使用的 iPhone 商品 JSON 数据*

```js
// data/product.json —— 标准的 JSON 文件
{
  "id": "1",
  "name": "iPhone SE",
  "image": "…png",
  "link": "https://…",
  "price": "$1990",
  "status": "In Stock",
  "camera": "…",
  "size": "5.45-inches (4.7-inch Display)"
}
```

JSON 三条铁律：

1. **键必须是双引号**（JS 对象字面量的单引号/无引号都不合法）
2. 不允许注释、不允许尾随逗号
3. JS ↔ JSON 转换只有两个方法：

```js
JSON.parse(jsonString)    // 字符串 → JS 对象
JSON.stringify(jsObject)  // JS 对象 → 字符串
```

### 提供两种响应：API 与页面

![[img/course-02-callback-nesting.jpg]]
*图 05-1b · 异步读 JSON 文件*

```js
fs.readFile('./data/product.json', 'utf-8', (err, data) => {
  if (path === '/api') {
    response.writeHead(200, { 'Content-Type': 'application/json' });
    response.end(data);                        // 原样返回 JSON 字符串
  } else {
    response.writeHead(200, { 'Content-Type': 'text/html' });
    response.end(renderHtml(JSON.parse(data))); // 渲染成 HTML
  }
});
```

> [!tip] 前后端分离的核心思想
> **同一份数据，两种格式**。Web/App 前端拿 JSON 自己渲染，传统浏览器直接拿 HTML——服务器只是"数据 + 表示"的分发者。

---

## 二、把 JSON 渲染成 HTML（#16）

模板里用 `{{字段名}}` 占位，JS 里逐字段替换：

![[img/course-05-json-to-html.jpg]]
*图 05-2 · 00:08:11 · 对模板做逐字段替换*

```js
let output = productListHtml
  .replace('{{%IMAGE%}}', product.productImage)
  .replace('{{%NAME%}}', product.name)
  .replace('{{%MODEL%}}', product.model)
  .replace('{{%STATUS%}}', product.status)
  .replace('{{%PRICE%}}', product.price)
  .replace('{{%LINK%}}', product.link)
  .replace('{{%ID%}}', product.id);
```

一页要渲染**多个商品**时，对 JSON 数组循环拼接：

```js
const cards = products.map(p => renderCard(p)).join('');
response.end(html.replace('{{%PRODUCTS%}}', cards));
```

> [!note] 课程用 `.replace()` 链，工业界用什么？
> Express 生态用模板引擎（EJS、Pug）或前后端分离 + 前端框架。`replace` 链的教训是：**拼字符串维护成本极高**——这正是后面引入 Express 的动机之一。

---

## 三、解析查询字符串（#17）

### 3.1 什么是查询字符串

![[img/course-05-query-parse.jpg]]
*图 05-3 · 视频 #17 · URL 中 `?key=value&...` 的结构示意*


```
www.nodeapp.com/books?author=john&sort=price
                 ──────┬───────────────────
                 查询字符串（query string）
```

`?` 后面的 `key=value` 用 `&` 连接，是**向服务器传参数的最常见方式**。

### 3.2 老方法：`url.parse`

![[img/course-05-url-parse.jpg]]
*图 05-4 · 00:09:23 · url.parse(request.url, true).query*

```js
const url = require('url');

const server = http.createServer((request, response) => {
  const { query, pathname } = url.parse(request.url, true);   // true → query 解析成对象

  if (pathname === '/product') {
    const product = products[query.id];       // /product?id=1
    response.end(renderCard(product));
  }
});
```

第二个参数 `true` 让 `query` 直接变成对象：`?id=1` → `{ id: '1' }`。

### 3.3 新方法：`URL` 类（现代标准）

`url.parse` 在官方文档中已标记为 **Legacy**（见[官方 URL 文档](https://nodejs.org/docs/latest/api/url.html#urlurlparseurl-resolveotext-true)），现代代码用 WHATWG `URL` 类：

```js
const u = new URL(request.url, 'http://localhost:8000');
u.pathname;             // '/product'
u.searchParams.get('id');     // '1'
u.searchParams.get('sort');   // 'price'
```

两者对照：

| 需求 | url.parse（旧） | URL 类（新） |
|:---|:---|:---|
| 路径 | `parse(url).pathname` | `new URL(url, base).pathname` |
| 取参数 | `.query.id` | `.searchParams.get('id')` |
| 维护状态 | Legacy，勿在新代码使用 | 标准推荐 |

> [!tip] Express 里更简单
> Express 已把解析结果挂在 `request.query` / `request.params` 上，根本不用自己解析，见 [[02-CRUD接口实现]]。

---

## 四、提取可复用函数（#18）

![[img/course-05-reusable-fn.jpg]]
*图 05-5 · 00:06:04 · 渲染逻辑封装成 replaceProducts 函数*

```js
function replaceProducts(template, product) {
  let output = template.replace('{{%IMAGE%}}', product.productImage);
  output = output.replace('{{%NAME%}}', product.name);
  // …其余字段
  return output;
}
```

然后路由里按 ID 取商品渲染详情页：

```js
if (pathname === '/product') {
  const product = products[query.id];
  const prodDetailHtml = replaceProducts(productDetailHtml, product);
  response.end(prodDetailHtml);
}
```

**重构心法**（这一集表面讲封装，实际在教三件事）：

1. 相同逻辑出现两次就该抽函数（DRY 原则）
2. 函数应该**只做一件事**：`replaceProducts` 只管"对象→卡片 HTML"
3. 下一集（[[06-模块化]]）会把这类工具函数**搬到独立文件**里

---

## 五、常见坑速查

| 坑 | 现象 | 解决 |
|:---|:---|:---|
| JSON 文件有注释/单引号 | `JSON.parse` 直接抛异常 | 检查 JSON 格式合法性 |
| `query.id` 是字符串 | `products[1]` 写成 `products['1']` 侥幸可用，但比较时 `1 === '1'` 为 false | 显式 `Number(query.id)` |
| URL 带查询串路由失配 | `/product?id=1` 进不了 `/product` 分支 | 先分离 pathname 再判断 |
| 中文/特殊字符参数乱码 | `%E6%89%8B…` | `searchParams.get` 自动解码；旧方法需 `decodeURIComponent` |

> [!summary] 自查清单
> - [ ] 会 JSON.parse / stringify，知道 JSON 语法三铁律
> - [ ] 能对同一份数据返回 JSON 或 HTML
> - [ ] 会用新 URL 类取查询参数
> - [ ] 知道为什么要抽可复用函数

---
**相关笔记**：[[04-路由与响应]] · [[06-模块化]] · [[02-CRUD接口实现]] · [[README|课程导航]]
