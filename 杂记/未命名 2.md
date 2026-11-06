# comments.json 字段分析

**规模**：600 条评论，覆盖 61 个 SKU，每 SKU 1–10 条；13 个字段在每条记录中均存在（结构稳定）。

## 字段清单

| 字段 | 类型 | 实际取值情况 |
|---|---|---|
| `sku_id` | string | 全部有值，关联 `products.json` |
| `comment_id` | string | 全部有值，格式 `auto_<hex>`，唯一 |
| `user_nickname` | string | 全部有值，已脱敏（如 `云***节`） |
| `user_avatar_url` | string | 全部有值 |
| `user_avatar_local_path` | null | **600 条全为 null**，未落地 |
| `content` | string | 全部非空 |
| `after_content` | string | 仅 8 条有追加内容，其余为 `""` |
| `score` | null | **600 条全为 null**，未采到评分 |
| `comment_time` | string | 格式不统一，见下 |
| `images_urls` | string[] | 全部非空，每条 2–10 张 |
| `images_local_paths` | array | **600 条全为空数组** |
| `source_mode` | string | 全部为 `browser`，无区分度 |
| `crawled_at` | string | ISO 8601，184 个不同值（按批次） |

## 数据质量问题

1. **`comment_time` 格式混杂**：558 条为 `MM-DD`（无年份），30 条为完整 `YYYY-MM-DD`（2024-10 至 2025-12），12 条为空。无法直接排序或算时长。
2. **三个字段实际无信息量**：`score` 全 null、`images_local_paths` 全空、`user_avatar_local_path` 全 null —— 抓取流程中未实现评分与本地化。
3. `source_mode` 恒为 `browser`，保留但无分析价值。
4. `crawled_at` 是批次时间而非评论时间，不能替代 `comment_time`。

## 私有属性小结

除 `sku_id`、`crawled_at` 外，其余 11 个字段均为本文件独有：用户维度（昵称、头像）、内容维度（正文、追评、评分、时间）、媒体维度（图片 URL/本地路径）以及采集元数据 `source_mode`。
