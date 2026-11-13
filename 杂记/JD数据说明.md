

本文说明 `data/products.jsonl` 与 `data/comments.jsonl` 的内容格式、字段含义和数据特征。

- **数据来源**：`jd_spider` 浏览器模式（`source_mode = "browser"`）被动采集。
- **文件格式**：JSON Lines（JSONL），**每行一个完整 JSON 对象**，行间无逗号、无数组包裹，编码为 UTF-8。
- **统计口径**：基于根目录当前快照——`products.jsonl` 61 条、`comments.jsonl` 600 条（`crawled_at` 覆盖 2026-11-02 至 2026-11-04，UTC）。
- **字段定义**：以 [jd_spider/models.py](jd_spider/models.py) 为准；本文额外标注了实际数据中的空值情况。

---

## 1. 文件概览

| 文件               | 记录数 | 唯一 SKU 数 |       大小 | 粒度                      |
| ---------------- | --: | -------: | -------: | ----------------------- |
| `products.jsonl` |  61 |       61 | 约 190 KB | 一行 = 一个商品，每个 SKU 只出现一次  |
| `comments.jsonl` | 600 |       61 | 约 594 KB | 一行 = 一条带图评论，同一 SKU 可有多条 |

两个文件通过 `sku_id` 关联，关系为 **1 个商品 : N 条评论**。当前快照中 61 个商品全部有评论，没有孤立记录。

---

## 2. `products.jsonl` — 商品

每行是一条商品记录。商品对象在写入前会经过 `_clean_null()` 兜底处理，因此**正常记录中不会出现 `null`**，缺失值表现为空字符串或空数组（见 [models.py:14](jd_spider/models.py#L14)）。

| 字段                  | 类型              | 本快照出现率 |         取值范围 | 说明                        | 示例                                              |
| ------------------- | --------------- | ------ | -----------: | ------------------------- | ----------------------------------------------- |
| `sku_id`            | string          | 61/61  |     6–14 位数字 | 京东商品编号，与评论表关联的外键          | `"100288035468"`                                |
| `title`             | string          | 61/61  |     28–98 字符 | 商品完整标题                    | `"摩托罗拉 联想moto g100 ..."`                        |
| `price`             | string          | 61/61  |       2–7 字符 | 页面展示价；**是字符串不是数字**        | `"1319"`、`"30.64"`                              |
| `brand`             | string          | 61/61  |      2–23 字符 | 品牌名，常带英文括注                | `"摩托罗拉（Motorola）"`                              |
| `shop_name`         | string          | 61/61  |      5–24 字符 | 店铺名称                      | `"联想moto手机京东自营旗舰店"`                             |
| `params`            | object          | 61/61  |      3–34 个键 | 规格参数表，键随类目变化，值均为字符串       | 见 2.1                                           |
| `main_image_urls`   | array\<string\> | 61/61  |       恒为 1 个 | 商品主图 URL                  | `"https://img10.360buyimg.com/pcpubliccms/..."` |
| `detail_image_urls` | array\<string\> | 61/61  | 5–48 个，均值 21 | 商品详情长图 URL，全表共 1284 个     | `"https://img30.360buyimg.com/sku/..."`         |
| `product_url`       | string          | 61/61  |     31–39 字符 | 商品页地址                     | `"https://item.jd.com/100288035468.html"`       |
| `crawled_at`        | string          | 61/61  |        25 字符 | 采集时间，UTC ISO-8601         | `"2026-11-02T04:30:48+00:00"`                   |
| `discovery_keyword` | string          | 61/61  |       2–5 字符 | 该商品来自哪个搜索关键词；固定 SKU 模式为空串 | `"手机"`                                          |

### 2.1 `params` 规格参数

`params` 是京东商品详情页的「规格参数 / 商品介绍」键值表。

- **不是固定 schema**：键名由商品类目决定，不同类目字段完全不同。全表累计出现过 190+ 个不同键。
- **只有两个键始终存在**：`品牌`（61/61）和 `商品编号`（61/61），可用于快速取品牌/编号。
- 所有值都是字符串，单位直接写在值里（如 `"210g"`、`"120Hz"`）。
- 每个商品有 3–34 个键，常见键的出现次数如下。

| 常见键 | 出现次数 | 示例值 |
|---|---:|---|
| `品牌` | 61 | `"摩托罗拉（Motorola）"` |
| `商品编号` | 61 | `"100288035468"` |
| `货号` | 30 | `"WXY-K262C"` |
| `系统` | 12 | `"Android"`、`"Windows 11"` |
| `风格` | 12 | `"简约风 轻奢 时尚"` |
| `能效网规格型号` | 12 | `"BCD-466WGHTDEDC9"` |
| `屏幕刷新率` | 11 | `"120Hz"` |
| `电池容量` | 11 | `"7000mAh(typ)"` |
| `国补备案型号` | 10 | `"moto g100"` |
| `类别` | 10 | `"抽取式"` |
| `店铺` | 10 | `"心相印亿晗专卖店"` |
| `认证型号` | 9 | `"TPN-Q304"` |
| `能效等级` | 9 | `"一级能效"` |
| `适用场景` | 9 | `"日常 职场"` |
| `适用人群` | 9 | `"中年 青年"` |

不同类目的典型键：

| 类目 | 典型键 |
|---|---|
| 手机 | `入网型号`、`机型`、`屏幕刷新率`、`系统`、`机身内存`、`5G网络`、`充电功率`、`电池容量`、`生物识别`、`后摄主像素` |
| 笔记本 | `内存类型`、`插槽数量`、`产品净重`、`屏幕尺寸`、`屏幕色域`、`内存容量`、`硬盘容量`、`处理器(CPU)`、`显卡`、`线程数` |
| 冰箱 | `气候类型`、`综合耗电量`、`制冷剂`、`冷冻能力`、`产品尺寸`、`容积`、`门款式`、`能效等级` |
| 运动鞋 | `货号`、`鞋帮高度`、`鞋底材质`、`闭合方式`、`适用性别`、`鞋跟高度` |
| 洗护/化妆 | `批准文号/备案编号`、`产品（注册/备案）名称`、`是否特殊化妆品`、`保质期`、`总净含量` |
| 服饰 | `面料`、`领型`、`衣长`、`裤长`、`版型`、`尺码`、`适用季节` |
| 纸尿裤 | `适用月龄`、`单包抽数`、`纸张尺寸`、`是否一次性` |
| 厨电 | `出风口径`、`拢烟腔材质`、`工作噪声`、`额定功率`、`安装方式` |

> [!warning] 下游程序不要假设固定键名
> 读取规格时应遍历 `params` 的键，或按类目白名单取值。

### 2.2 商品示例（URL 已截断）

```json
{
  "sku_id": "100288035468",
  "title": "摩托罗拉 联想moto g100 【凑单价更低】 四年质保 LCD护眼屏 ...",
  "price": "1319",
  "brand": "摩托罗拉（Motorola）",
  "shop_name": "联想moto手机京东自营旗舰店",
  "params": {
    "品牌": "摩托罗拉（Motorola）",
    "商品编号": "100288035468",
    "机型": "moto g100",
    "屏幕刷新率": "120Hz",
    "系统": "Android",
    "电池容量": "7000mAh(typ)",
    "机身颜色": "曜石黑"
  },
  "main_image_urls": ["https://img10.360buyimg.com/pcpubliccms/..."],
  "detail_image_urls": ["https://img30.360buyimg.com/sku/..."],
  "product_url": "https://item.jd.com/100288035468.html",
  "crawled_at": "2026-11-02T04:30:48+00:00",
  "discovery_keyword": "手机"
}
```

---

## 3. `comments.jsonl` — 评论

每行是一条评论。当前流程**只写入带买家晒图的评论**：`images_urls` 为空的评论不会落盘，也不占用每个 SKU 的 `max_comments` 额度。字段定义见 [models.py:49](jd_spider/models.py#L49)。

| 字段 | 类型 | 本快照出现率 | 取值范围 | 说明 | 示例 |
|---|---|---|---:|---|---|
| `sku_id` | string | 600/600 | 6–14 位数字 | 所属商品编号，关联 `products.jsonl` | `"100288035468"` |
| `comment_id` | string | 600/600 | 29 字符，600 个全部唯一 | `auto_` + 24 位十六进制，程序生成 | `"auto_2d1ba9eb502335805563b191"` |
| `user_nickname` | string | 600/600 | 2–13 字符 | 京东脱敏昵称，中间用 `***` 遮挡 | `"云***节"` |
| `user_avatar_url` | string | 600/600 | 88–116 字符 | 用户头像 URL；277 条为平台默认头像 | `"https://storage.360buyimg.com/..."` |
| `user_avatar_local_path` | null | 600/600 | 恒为 `null` | 仅兼容旧 schema，当前不下载头像 | `null` |
| `content` | string | 600/600 | 10–250 字符 | 评论正文 | `"京东物流超快，隔天就收到..."` |
| `after_content` | string | 600/600 | 0–65 字符；592 条为空 | 追评正文，绝大多数评论没有追评 | `""` |
| `score` | null | 600/600 | 恒为 `null` | 评分；当前浏览器采集路径未解析该值 | `null` |
| `comment_time` | string | 588/600 | 10 字符或空串 | 评论日期，格式 `月-日`（如 `10-05`），**不含年份**；12 条为空 | `"10-05"` |
| `images_urls` | array\<string\> | 600/600 | 2–10 个，均值 2.8 | 买家晒图 URL，全表共 1691 个 | `"https://img30.360buyimg.com/shaidan/..."` |
| `images_local_paths` | array | 600/600 | 恒为空数组 | 仅兼容旧 schema，当前不下载图片 | `[]` |
| `source_mode` | string | 600/600 | 恒为 `"browser"` | 采集模式，另一个可能值为 `"api"` | `"browser"` |
| `crawled_at` | string | 600/600 | 25 字符 | 采集时间，UTC ISO-8601 | `"2026-11-02T04:31:03+00:00"` |

> [!note] 空值属于正常现象
> 评论对象**不做** `null` 兜底，所以 `user_avatar_local_path` 和 `score` 会显式出现为 `null`。

### 3.1 评论示例（URL 已截断）

```json
{
  "sku_id": "100288035468",
  "comment_id": "auto_2d1ba9eb502335805563b191",
  "user_nickname": "云***节",
  "user_avatar_url": "https://storage.360buyimg.com/i.imageUpload/..._sma.jpg",
  "user_avatar_local_path": null,
  "content": "京东物流超快，隔天就收到摩托罗拉g100。...",
  "after_content": "",
  "score": null,
  "comment_time": "10-05",
  "images_urls": [
    "https://img30.360buyimg.com/shaidan/s300x300_...jpg.dpg",
    "https://img30.360buyimg.com/shaidan/s300x300_...jpg.dpg"
  ],
  "images_local_paths": [],
  "source_mode": "browser",
  "crawled_at": "2026-11-02T04:31:03+00:00"
}
```

---

## 4. 两表关系与数据分布

```text
products.jsonl (61)                  comments.jsonl (600)
┌──────────────────┐   1      N    ┌───────────────────────┐
│ sku_id (唯一)     │──────────────▶│ sku_id (可重复)        │
│ title / price …  │               │ comment_id (唯一)      │
└──────────────────┘               └───────────────────────┘
```

- **关联键**：`sku_id`。
- **评论数分布**：每个商品 1–10 条。59 个 SKU 达到上限 10 条，1 个 SKU 为 9 条，1 个 SKU 为 1 条。上限由运行时 `--max-comments 10` 控制。
- **关键词分布**：商品来自 13 个搜索关键词——笔记本电脑 7、手机 6、运动鞋 5、生活用品 5、零食 5、男装 5、护肤品 5、洗发水 4、冰箱 4、纸尿裤 4、厨房电器 4、运动户外 4、家居用品 3。

---

## 5. 数据特征与使用注意事项

1. **JSONL 解析**：逐行 `json.loads` 即可，不要当作 JSON 数组读取。
2. **`price` 是字符串**：比较价格前先 `float()`，且注意个别值带小数（如 `"30.64"`）。
3. **`comment_time` 没有年份**：格式为 `MM-DD`，跨年数据需要结合 `crawled_at` 推断年份；另有 12 条为空，解析时要容错。
4. **`score` 当前恒为 `null`**：如需评分，需要另行采集或改用其他路径，不要依赖该字段。
5. **`after_content` 大多为空**：只有 8/600 条有追评内容。
6. **图片只有 URL，没有本地文件**：`images_local_paths` 恒为空、`user_avatar_local_path` 恒为 `null`。URL 是否需要登录态或 Referer，由下游自行处理。商品图看 `main_image_urls` / `detail_image_urls`，评论图看 `images_urls`。
7. **`user_nickname` 已脱敏**：形如 `云***节`，不能用作真实用户标识。
8. **`params` 键不固定**：跨类目字段差异很大，取值前先判断键是否存在。
9. **去重策略**：同一 SKU 在商品表只写一次；评论按 `comment_id` 去重，重跑不会产生重复行。
10. **编码与时间**：文件为 UTF-8；`crawled_at` 统一为带时区的 UTC ISO-8601 字符串。

---

## 6. 快速读取示例

**Python（pandas）**

```python
import pandas as pd

products = pd.read_json("data/products.jsonl", lines=True)
comments = pd.read_json("data/comments.jsonl", lines=True)

# 关联商品标题与价格
merged = comments.merge(
    products[["sku_id", "title", "price", "brand", "shop_name"]],
    on="sku_id",
    how="left",
)

# 每条评论的晒图数量
merged["image_count"] = merged["images_urls"].apply(len)
print(merged[["sku_id", "title", "image_count"]].head())
```

**命令行（jq）**

```bash
# 只看带 3 张以上晒图的评论
jq -c 'select((.images_urls | length) >= 3)' data/comments.jsonl

# 提取所有手机类商品的 SKU
jq -r 'select(.discovery_keyword == "手机") | .sku_id' data/products.jsonl
```

---

*字段定义参考：[jd_spider/models.py](jd_spider/models.py)；采集流程与运行参数参考：[README.md](软件测试/README.md)。*
