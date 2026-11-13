# JD商城项目完整分析

> [!abstract] 一句话
> 这是一个用 ==Vue 3 + Vite + Vue Router + Element Plus + Swiper== 实现的京东风格商城前端。项目没有后端，商品与评论来自 `database/*.json`，购物车用「模块级 `reactive` + `localStorage`」共享并持久化，完整覆盖了「商品列表 → 搜索 → 分页 → 商品详情 → 加入购物车 → 结算」的购物链路。

---

## 一、项目概况

- **项目位置**：`C:\Users\Shyanthen\Desktop\vue_lab2`
- **项目类型**：Vue 3 前端实验 / 教学项目（`package.json` 中的包名为 `vue_lab2`）
- **页面数量**：3 个路由页面
  - `/goods`：商品列表页
  - `/goods/:skuId`：商品详情页
  - `/cart`：购物车页
- **数据规模**：
  - `database/products.json`：61 条商品
  - `database/comments.json`：600 条评论，覆盖全部 61 个商品
- **数据来源**：本地 JSON 文件，`src/api/goods.js` 把它们包装成带 `setTimeout` 的模拟异步接口
- **持久化**：购物车写入 `localStorage`，键名为 `jd_cart_v1`

### 1.1 已实现的功能

| 功能 | 所在位置 | 说明 |
| --- | --- | --- |
| 商品列表 | `views/goods-list.vue` | 每页 12 条，共 6 页 |
| 关键字搜索 | `components/tab-header.vue` | 回车后写入 URL 的 `?keyword=` |
| 商品分页 | `views/goods-list.vue` | 上一页 / 页码 / 下一页 |
| 商品详情 | `views/goods-detail.vue` | 动态路由参数 `skuId` |
| 图片轮播 | `components/detail/goods-gallery.vue` | Swiper + 自动播放 |
| 商品参数 | `components/detail/showcase-board.vue` | `dl / dt / dd` 参数表 |
| 商品详情长图 | `components/detail/goods-scroll-img.vue` | 展示全部 detail 图片 |
| 用户评价 | `components/detail/comment-list.vue` | 头像、昵称、时间、追评、晒图 |
| 加入购物车 | `components/detail/goods-info-panel.vue` | 支持先选数量 |
| 购物车增删改 | `components/cart-area/*` | 数量、小计、删除、空状态 |
| 购物车持久化 | `stores/cart.js` | 刷新页面不丢失 |
| 模拟结算 | `components/cart-area/footer-fixed.vue` | 确认后清空购物车并提示 |

> [!info] 项目边界
> 没有真实后端、没有登录与用户系统、没有订单页和支付页。「去结算」只是 `ElMessageBox` 确认 + 清空购物车 + `ElMessage.success('下单成功')` 的演示逻辑。

---

## 二、技术栈

| 技术 | 版本 | 用途 |
| --- | --- | --- |
| Vue | `^3.5.42` | 组件与响应式系统，全部使用 `<script setup>` |
| Vite | `^8.2.2` | 开发服务器与生产构建 |
| Vue Router | `^4.6.4` | 三个页面的 SPA 路由 |
| Element Plus | `^2.14.8` | 图标、`ElMessage`、`ElMessageBox` |
| Swiper | `^14.4.0` | 详情页主图轮播 |
| `@element-plus/icons-vue` | 由 Element Plus 依赖提供 | `Search`、`ShoppingCartFull`、`House`、`Top` |
| JavaScript | — | 未使用 TypeScript |
| CSS | — | 全局变量 + 组件级 `scoped` 样式 |
| JSON | — | 本地商品与评论数据 |
| localStorage | — | 购物车持久化 |

> [!note] 没有使用的常见依赖
> 项目没有 Pinia、没有 Axios、没有 ESLint 配置、没有测试框架。状态管理由 `src/stores/cart.js` 自己实现了一个轻量的组合式 store。

---

## 三、运行与构建

```bash
cd C:\Users\Shyanthen\Desktop\vue_lab2

npm install       # 安装依赖
npm run dev       # 启动开发服务器
npm run build     # 生产构建
npm run preview   # 预览构建产物
```

`package.json` 的 `engines` 要求：

```json
"node": "^22.18.0 || >=24.12.0"
```

本次分析实际执行了 `npm run build`，结果：

```text
✓ 1674 modules transformed
dist/index.html                   0.48 kB │ gzip:   0.32 kB
dist/assets/index-*.css         385.36 kB │ gzip:  52.57 kB
dist/assets/index-*.js        1,918.74 kB │ gzip: 549.07 kB
✓ built in 652ms
```

构建成功，但 Vite 给出了「chunk 大于 500 kB」的警告。主要原因是 Element Plus 被完整注册，且路由页面没有做懒加载，详见 [[可访问性与性能]] 和 [[代码质量与改进]]。

---

## 四、笔记目录

### 架构与路由

- [[项目概况]] —— 项目定位、功能清单、目录结构、数据字段与技术栈
- [[启动与路由]] —— `index.html → main.js → App.vue` 启动链路、路由表与跳转方式
- [[数据层与模拟接口]] —— JSON 导入、字段归一化、列表查询、详情与评论接口、价格格式化

### 组件与通信

- [[组件树与职责]] —— 完整组件树、每个组件的 props / emits / 职责边界
- [[商品列表与卡片]] —— 列表页状态、加载流程、分页、卡片组件与网格布局
- [[商品详情组件]] —— 详情页编排、轮播、信息面板、参数表、长图、评论列表
- [[购物车组件]] —— 购物车页、表头与行网格、`NumAddDel` 自定义 `v-model`、固定结算栏

### 状态与功能流程

- [[购物车状态与持久化]] —— 模块级单例、`reactive`、`computed`、`localStorage` 读写与校验
- [[搜索分页与详情流程]] —— 搜索、分页、详情加载三条端到端链路与边界情况
- [[加入购物车与结算]] —— 加购、改数量、删除、结算四条交互链路

### 样式与体验

- [[布局与响应式]] —— CSS 变量、Grid / Flex、断点设计、购物车全宽布局
- [[可访问性与性能]] —— 语义化与键盘可访问性、构建体积、图片策略、健壮性风险

### 总结与改进

- [[数据流总图]] —— 从 JSON 到页面、从用户操作到 localStorage 的全站数据流
- [[代码质量与改进]] —— 优点盘点、问题分级、可落地的改进代码与优先级路线
- [[学习路线与总结]] —— 建议阅读顺序、Vue 知识点对照表、练习与扩展方向

---

## 五、阅读顺序建议

> [!tip] 推荐路线
> 1. 先读 [[项目概况]]，建立「有哪些页面、数据从哪来」的整体印象。
> 2. 再读 [[启动与路由]] 和 [[数据层与模拟接口]]，理解应用如何启动、数据如何被查询和转换。
> 3. 接着读 [[组件树与职责]]，把组件之间的关系画出来。
> 4. 然后按页面读 [[商品列表与卡片]] → [[商品详情组件]] → [[购物车组件]]。
> 5. 用 [[购物车状态与持久化]]、[[搜索分页与详情流程]]、[[加入购物车与结算]] 串起状态与流程。
> 6. 最后用 [[数据流总图]]、[[代码质量与改进]]、[[学习路线与总结]] 收尾。

---

## 六、核心一句话

> [!important] 这个项目最值得掌握的主线
> **用户操作 → 组件事件 / 路由变化 → 页面方法或 store action → 响应式状态变化 → `computed` / `watch` 重新计算 → Vue 更新 DOM。**
>
> 商品数据是「模块加载时一次性归一化」的只读数组；购物车是「全局共享的可变状态」，并通过深度 `watch` 自动落盘。抓住这两条数据线，就抓住了整个项目。

---

> [!success] 分析说明
> 本文档基于对项目全部源码（`src/` 下 20 个文件）、`package.json`、`vite.config.js`、`index.html` 与 `database/` 两份 JSON 的逐文件阅读，并实际执行生产构建验证。分析日期：2026-11-09。
