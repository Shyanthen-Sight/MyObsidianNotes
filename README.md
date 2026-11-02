# JD 商品与评论采集器

一个可运行的 Python 3.11+ 项目，用于采集指定京东 SKU 的商品信息、评论、追评和图片，并输出为 JSONL。

项目提供两种模式：

- **模式 B（默认，推荐）**：DrissionPage 驱动真实 Chrome/Edge 浏览器，使用持久化 profile 保存登录态，被动监听页面自然产生的评论接口响应，必要时用 DOM 兜底。**不逆向签名、不伪造请求、不绕过验证码或风控。**
- **模式 A（官方 API 适配器骨架）**：用于你已获得合法京东开放平台/联盟应用凭证的场景。缺少凭证时给出明确提示并退出；签名函数是显式占位，必须按获批文档实现。

> **合规提示**
>
> 请仅在获得授权、且符合京东服务条款、当地法律和个人信息保护要求的前提下使用本工具。
> 本项目只采集你有权访问的数据，不包含绕过登录、验证码、风控、加密或签名算法的逻辑，也不应被用于高频、批量或商业转售抓取。
> 评论中的昵称、头像等属于个人信息，请控制采集范围、保存期限和使用目的。

---

## 1. 功能概览

### 商品信息

- SKU ID
- 商品标题
- 价格（页面能解析到时）
- 品牌
- 店铺名称
- 商品参数
- 主图 URL 列表
- 商品 URL
- 采集时间

### 评论信息

- SKU ID
- 评论 ID
- 用户昵称
- 用户头像 URL 与本地路径
- 评论内容
- 追评内容
- 评分
- 评论时间
- 评论图片 URL 列表与本地路径
- 来源模式（`browser` / `api`）
- 采集时间

### 其他能力

- `products.jsonl`、`comments.jsonl`、`image_mapping.jsonl` 增量写入。
- 按 `comment_id` 去重，重复运行时不会追加同一条评论。
- 图片和头像下载到本地，已有文件自动跳过。
- 图片下载携带 `Referer`，带重试、超时、图片类型校验和随机限速。
- 默认请求间隔 2~5 秒，可通过配置调整。
- 控制台 + 文件双日志。
- 单个评论/单张图片失败不会中断整个任务。
- 支持多个 SKU，单个 SKU 失败不影响其他 SKU（登录/风控类错误会停止批次，等待人工处理）。

---

## 2. 环境要求

- Python **3.11+**
- Windows 10/11、macOS 或 Linux
- 本机已安装 **Google Chrome** 或 **Microsoft Edge**
- 能正常访问京东商品页
- 首次登录需要人工扫码，因此浏览器不能使用无头模式

依赖：

```text
DrissionPage==4.1.1.4
requests>=2.31,<3.0
PyYAML>=6.0,<7.0
python-dotenv>=1.0,<2.0
```

---

## 3. 项目结构

```text
JDscript/
├─ jd_spider/
│  ├─ __init__.py          # 版本信息
│  ├─ __main__.py          # python -m jd_spider 入口
│  ├─ cli.py               # 参数解析、模式分发、退出码
│  ├─ config.py            # 默认值 / YAML / 环境变量 / CLI 覆盖
│  ├─ models.py            # Product、Comment、ImageMapping
│  ├─ storage.py           # JSONL 增量写入、去重、商品 UPSERT
│  ├─ downloader.py        # 图片/头像下载、Referer、重试、校验
│  ├─ api_client.py        # 模式 A：官方 API 适配器骨架
│  ├─ browser_crawler.py   # 模式 B：DrissionPage 浏览器采集
│  ├─ parsers.py           # 网络 JSON / DOM / 商品信息自适应解析
│  ├─ pipeline.py          # 评论 -> 图片 -> 持久化流水线
│  ├─ exceptions.py        # 自定义异常
│  └─ utils.py             # 日志、限速、文件名、URL 等工具
├─ tests/
│  ├─ test_parsers.py
│  ├─ test_storage.py
│  └─ test_downloader.py
├─ requirements.txt
├─ .env.example
├─ config.example.yaml
├─ .gitignore
└─ README.md
```

运行时还会生成：

```text
data/
├─ products.jsonl
├─ comments.jsonl
├─ image_mapping.jsonl
├─ images/
│  ├─ comments/<sku_id>/
│  └─ avatars/<sku_id>/
└─ logs/
   └─ jd_spider.log
.jd_profile/               # Chrome 持久化登录 profile（不要提交）
```

---

## 4. 安装

### Windows PowerShell

```powershell
cd C:\Users\Shyanthen\Desktop\JDscript

py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
pip install -r requirements.txt
```

如果 PowerShell 阻止激活脚本，可在当前窗口临时放开：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

### macOS / Linux

```bash
cd JDscript
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

验证安装：

```bash
python -m jd_spider --version
python -m jd_spider --help
```

---

## 5. 首次配置与扫码登录

### 5.1 创建本地配置

```powershell
Copy-Item .env.example .env
Copy-Item config.example.yaml config.yaml
```

`.env` 和 `config.yaml` 已被 `.gitignore` 忽略，不要提交真实凭证或登录态。

首次使用至少确认：

```dotenv
MODE=browser
OUTPUT_DIR=./data
HEADLESS=false
USER_DATA_PATH=./.jd_profile
MIN_DELAY=2.0
MAX_DELAY=5.0
```

如果 DrissionPage 找不到浏览器，在 `.env` 中填写 Chrome/Edge 路径，例如：

```dotenv
BROWSER_PATH=C:\Program Files\Google\Chrome\Application\chrome.exe
```

或者通过命令行：

```powershell
python -m jd_spider --login-only --no-headless `
  --browser-path "C:\Program Files\Google\Chrome\Application\chrome.exe"
```

### 5.2 扫码登录

```powershell
python -m jd_spider --login-only --no-headless
```

程序会打开京东登录页。请在弹出的真实浏览器中完成扫码/账号登录。程序检测到京东登录 Cookie（`pt_key` / `thor`）后结束登录流程。

登录态保存在 `USER_DATA_PATH` 指向的 Chrome profile 中，后续运行通常不需要重复扫码。

> 注意：
>
> - 首次登录**不能**使用 `--headless`。
> - 不要把 `.jd_profile` 分享给他人；其中可能包含登录凭据。
> - 如果登录态过期，重新执行上面的 `--login-only` 即可。
> - 程序不会注入账号密码、不会破解验证码，也不会尝试绕过安全验证。

### 5.3 直接运行（可选检查登录）

```powershell
python -m jd_spider --sku 100012043978 --out ./data --mode browser --max-pages 50
```

如果评论公开可见，通常不登录也能采集一部分。需要登录态时：

```powershell
python -m jd_spider --sku 100012043978 --out ./data --mode browser `
  --max-pages 50 --require-login
```

---

## 6. 运行示例

### 单个 SKU

```powershell
python -m jd_spider --sku 100012043978 --out ./data --mode browser --max-pages 50
```

### 多个 SKU

```powershell
python -m jd_spider --sku 100012043978 --sku 100026667910 --out ./data --max-pages 100
```

也可以逗号分隔：

```powershell
python -m jd_spider --sku "100012043978,100026667910" --out ./data
```

### 指定配置文件

```powershell
python -m jd_spider --config .\config.yaml --env-file .\.env `
  --sku 100012043978 --max-scrolls 200
```

### 无头运行

仅在登录态已存在且页面不触发额外验证时使用：

```powershell
python -m jd_spider --sku 100012043978 --mode browser --headless
```

遇到登录失效或安全验证时，程序会记录提示并停止/等待，不会自动绕过。此时改用非无头模式重新登录。

### 官方 API 模式

```powershell
python -m jd_spider --sku 100012043978 --mode api --out ./data
```

缺少配置时会直接退出，示例输出：

```text
模式 A 所需的官方 API 配置不完整，缺少：
  - JD_APP_KEY
  - JD_APP_SECRET
  - JD_ACCESS_TOKEN
  - JD_API_COMMENT_PATH
  - JD_API_METHOD
...
模式 B 不依赖这些凭证，可直接使用 --mode browser。
```

模式 A 的详细限制见第 11 节。

---

## 7. 配置说明

配置优先级：

```text
内置默认值 < config.yaml < 环境变量/.env < 命令行参数
```

CLI 未指定的参数不会覆盖已有 YAML/环境变量。

### 7.1 主要环境变量

| 变量 | 默认值 | 说明 |
|---|---|---|
| `MODE` | `browser` | `browser` 或 `api` |
| `OUTPUT_DIR` | `./data` | 输出目录 |
| `MAX_PAGES` | `50` | 最大评论页/监听数据批数 |
| `MIN_DELAY` | `2.0` | 主动请求最小间隔（秒） |
| `MAX_DELAY` | `5.0` | 主动请求最大间隔（秒） |
| `JD_COOKIE` | 空 | 可选，HTTP 图片下载时使用的 Cookie；浏览器模式的登录态由 profile 保存 |
| `HEADLESS` | `false` | 是否无头；首次扫码必须为 `false` |
| `USER_DATA_PATH` | `./.jd_profile` | 持久化 Chrome profile |
| `BROWSER_PATH` | 空 | Chrome/Edge 可执行文件路径 |
| `BROWSER_PORT` | `9222` | Chromium 调试端口 |
| `REQUIRE_LOGIN` | `false` | 运行前是否强制检查登录 |
| `LOGIN_TIMEOUT` | `300` | 等待人工登录秒数 |
| `VERIFICATION_TIMEOUT` | `180` | 遇到安全验证时等待人工处理秒数 |
| `MAX_SCROLLS` | `120` | 评论区最大滚动轮次 |
| `DOM_FALLBACK` | `true` | 网络未捕获评论时是否使用 DOM 兜底 |
| `NETWORK_WAIT` | `2.5` | 每次滚动后等待网络响应秒数 |
| `SCROLL_PAUSE` | `1.2` | 滚动间隔秒数 |
| `IMAGE_MIN_DELAY` / `IMAGE_MAX_DELAY` | 空 | 图片请求间隔；留空沿用 `MIN_DELAY`/`MAX_DELAY` |
| `COMMENT_URL_PATTERNS` | 见下 | JSON 数组形式的评论接口 URL 正则 |
| `LOG_LEVEL` | `INFO` | 日志级别 |
| `LOG_FILE` | 空 | 空则写入 `<输出目录>/logs/jd_spider.log` |

默认评论接口监听正则：

```text
(?i)comment
(?i)club\.jd\.com
(?i)discuss
(?i)evaluation
```

### 7.2 官方 API 环境变量

| 变量 | 说明 |
|---|---|
| `JD_APP_KEY` | 合法应用的 AppKey |
| `JD_APP_SECRET` | 合法应用的 AppSecret |
| `JD_ACCESS_TOKEN` | OAuth access_token |
| `JD_API_BASE_URL` | 网关地址，示例值仅作占位 |
| `JD_API_COMMENT_PATH` | 评论接口路径，必须按获批文档填写 |
| `JD_API_METHOD` | 接口 method，必须按获批文档填写 |
| `JD_API_VERSION` | 接口版本 |
| `JD_API_REQUEST_METHOD` | `GET` 或 `POST` |
| `JD_API_PAGE_SIZE` | 每页数量 |
| `JD_API_SKU_PARAM` | SKU 参数名（占位，需核对） |
| `JD_API_PAGE_PARAM` | 页码参数名（占位，需核对） |
| `JD_API_PAGE_SIZE_PARAM` | 每页数量参数名（占位，需核对） |
| `JD_API_TOKEN_PARAM` | token 参数名（占位，需核对） |
| `JD_API_SIGN_PARAM` | 签名参数名（占位，需核对） |

### 7.3 配置字段

完整示例见 `config.example.yaml`。常用字段：

```yaml
mode: browser
output_dir: ./data
max_pages: 50
min_delay: 2.0
max_delay: 5.0

browser:
  headless: false
  user_data_path: ./.jd_profile
  port: 9222
  require_login: false
  max_scrolls: 120
  max_empty_rounds: 3
  dom_fallback: true
  comment_url_patterns:
    - "(?i)comment"
    - "(?i)club\\.jd\\.com"
    - "(?i)discuss"
    - "(?i)evaluation"
```

---

## 8. 输出格式

所有 JSONL 均为 UTF-8，每行一个 JSON 对象。图片本地路径是**相对于输出目录**的 POSIX 路径，例如：

```text
images/comments/100012043978/123456789_001.jpg
```

可以直接用：

```python
from pathlib import Path
absolute = Path("./data") / record["images_local_paths"][0]
```

### 8.1 `products.jsonl`

同一 SKU 重复运行时会更新为最新一条记录，不会无限追加。

```json
{
  "sku_id": "100012043978",
  "title": "示例商品标题",
  "price": "199.00",
  "brand": "示例品牌",
  "shop_name": "示例店铺",
  "params": {
    "商品编号": "100012043978",
    "型号": "示例型号"
  },
  "main_image_urls": [
    "https://img30.360buyimg.com/..."
  ],
  "product_url": "https://item.jd.com/100012043978.html",
  "crawled_at": "2026-11-02T08:00:00+00:00"
}
```

### 8.2 `comments.jsonl`

```json
{
  "sku_id": "100012043978",
  "comment_id": "123456789",
  "user_nickname": "匿名用户",
  "user_avatar_url": "https://img30.360buyimg.com/...",
  "user_avatar_local_path": "images/avatars/100012043978/avatar_123456789.jpg",
  "content": "评论正文",
  "after_content": "追评正文",
  "score": 5,
  "comment_time": "2026-10-01 12:00:00",
  "images_urls": [
    "https://img30.360buyimg.com/..."
  ],
  "images_local_paths": [
    "images/comments/100012043978/123456789_001.jpg"
  ],
  "source_mode": "browser",
  "crawled_at": "2026-11-02T08:00:00+00:00"
}
```

说明：

- `images_urls` 是接口中识别到的全部评论图片 URL。
- `images_local_paths` 只包含**成功下载**的图片，顺序与成功下载的图片一致。
- 头像下载失败时，`user_avatar_url` 仍保留，`user_avatar_local_path` 为 `null`。
- 字段名是项目统一模型，不声称等同于京东接口的原始字段名。

### 8.3 `image_mapping.jsonl`

这是图片与原始 URL 的权威映射，建议以它为准。

```json
{
  "sku_id": "100012043978",
  "comment_id": "123456789",
  "url": "https://img30.360buyimg.com/...",
  "local_path": "images/comments/100012043978/123456789_001.jpg",
  "image_type": "comment",
  "index": 1,
  "crawled_at": "2026-11-02T08:00:00+00:00"
}
```

`image_type` 为 `comment` 或 `avatar`。

### 8.4 图片命名

- 评论图：`{安全化 comment_id}_{序号:03d}.{扩展名}`
- 头像：`avatar_{安全化 comment_id}.{扩展名}`

`comment_id` 中不适合作为文件名的字符会被替换，并在需要时附加短哈希，保证稳定且避免路径穿越。

---

## 9. 模式 B 的工作方式

1. 启动真实 Chromium 浏览器，加载持久化 profile。
2. 打开 `https://item.jd.com/{sku_id}.html`。
3. 解析商品信息并写入 `products.jsonl`。
4. 启动网络监听，只观察匹配 `comment_url_patterns` 的 XHR/Fetch 响应。程序不修改请求、不注入签名、不重放受保护接口。
5. 尝试点击“商品评价 / 查看全部评论 / 更多回复 / 加载更多 / 下一页”等入口。
6. 滚动页面和评论区，等待自然加载；把捕获到的 JSON/JSONP 响应交给自适应解析器。
7. 对评论去重、下载头像和评论图、增量写入 JSONL。
8. 达到最大页数、连续多轮无新数据或达到最大滚动次数时停止。
9. 如果网络监听一条评论都没解析到，且 `DOM_FALLBACK=true`，才从页面 DOM 兜底提取，避免与网络结果重复。

### 为什么优先监听网络响应

页面渲染后的 DOM 经常缺少评论 ID、完整图片列表或追评结构。监听浏览器自己发起的正常响应，可以拿到更完整的结构，同时不需要伪造请求。

### 关于“晒单”入口

默认**不会**点击“晒单”筛选。原因是它通常会把评论列表过滤成“仅带图评论”，导致纯文字评论丢失。评论图片会从网络响应中提取，不需要依赖“晒单”筛选。

如果实际页面只有点击“晒单”后才加载图片列表，可在 `config.yaml` 的 `browser.expand_texts` 中自行加入“晒单”，并接受纯文字评论可能减少的后果。这是需要根据页面实际行为调整的 TODO。

### 关于随机限速

- 程序主动发起的 HTTP 请求（模式 A、图片下载）默认每次等待 2~5 秒随机时长。
- 图片下载失败重试时也会限速并指数退避。
- 浏览器滚动、点击使用独立的 `scroll_pause` / `network_wait`。
- 不支持也不建议把间隔调到 0 进行高频抓取。

### 登录、验证码与风控

- 只检测登录 Cookie 是否存在，不注入账号密码。
- 遇到登录页会提示人工登录；`--require-login` 会强制等待。
- 遇到安全验证/滑块/验证码页面时，程序只等待人工处理，不尝试自动绕过。
- 若长时间未处理，会记录错误并停止相关任务。

---

## 10. 字段映射与 TODO

京东页面和接口会变化。本项目**不声称**内置字段名就是当前线上字段，采用自适应别名和递归查找，并在无法识别时保留空值。请根据你自己的合法抓包结果调整。

需要重点核对的映射位置：

| 内容 | 文件 | 说明 |
|---|---|---|
| 评论 ID | `jd_spider/parsers.py` → `COMMENT_ID_ALIASES` | `commentId`、`id`、`guid` 等 |
| 昵称 | `NICKNAME_ALIASES` | 包含 `userNickName`、`user_nickname` 等 |
| 头像 | `AVATAR_ALIASES` | 包含 `userImage`、`user_avatar_url` 等 |
| 正文 | `CONTENT_ALIASES` | 包含 `content`、`commentContent` 等 |
| 追评 | `AFTER_CONTAINER_ALIASES` / `AFTER_TEXT_ALIASES` | 兼容嵌套对象和纯文本 |
| 评分 | `SCORE_ALIASES` | 支持 `5`、`"5星"` 等 |
| 时间 | `TIME_ALIASES` | 原样保留接口时间字符串 |
| 图片容器 | `IMAGE_CONTAINER_ALIASES` / `_IMAGE_URL_KEYS` | 图片可能在多层对象中 |
| DOM 评论选择器 | `_DOM_COMMENTS_JS` | 页面结构变化时修改选择器 |
| 商品信息 | `PRODUCT_INFO_JS` | 标题、价格、品牌、店铺、参数、主图 |
| 商品 HTML 兜底 | `_fallback_product_from_html` | 仅能拿到标题和 og:image 等 |
| 评论接口 URL | `browser.comment_url_patterns` | 新接口 URL 不匹配时加入正则 |
| 展开按钮文本 | `browser.expand_texts` | 页面按钮文案变化时修改 |
| API 参数名 | `config.yaml` → `api.*_param` | 占位值，必须按官方文档核对 |
| API 签名 | `api_client.py` → `_sign()` | 显式占位，必须自行实现 |
| API 响应字段 | `parsers.py` 别名表 | 按真实响应补充 |

### 已知限制

- 价格可能来自异步接口，首次解析可能为空，程序会尝试二次解析；仍可能拿不到活动价。
- 品牌、店铺、参数在不同页面模板下可能缺失。
- DOM 兜底拿不到完整评论 ID 时会生成稳定的 `auto_...` ID；它适合本工具去重，但不等同于京东评论 ID。
- 程序只处理同一标签页内的加载。若页面把评论区强制打开到新标签页，当前实现可能无法监听新标签页；可在同一标签页手动打开评论区域，或按页面行为扩展 `browser_crawler.py`。
- 评论数量、可见性、是否需要登录会随商品和账号状态变化。
- 模式 A 只提供适配器骨架，不保证任何具体开放平台接口可用。

---

## 11. 模式 A：官方 API 适配器

模式 A 面向**已获得合法官方 API 凭证**的使用者。

### 11.1 特点

- 从 `.env` / 环境变量读取 AppKey、AppSecret、access_token。
- 支持自定义网关地址、评论路径、method、分页参数名。
- 复用统一的评论解析、图片下载、去重和 JSONL 存储流水线。
- 缺少必要配置时在发请求前退出，退出码为 `2`。
- 签名函数 `_sign()` 是占位实现，不会猜测或逆向京东签名（包括 h5st）。

### 11.2 使用步骤

1. 按获批的官方文档确认接口地址、method、参数名和返回结构。
2. 在 `.env` 填写凭证和 `JD_API_*` 配置。
3. 在 `config.yaml` 中核对 `api.*` 参数名。
4. 实现签名：
   - 向 `JdOpenPlatformCommentClient` 传入 `signer(params, app_secret) -> str`，或
   - 覆写 `JdOpenPlatformCommentClient._sign()`。
5. 根据真实响应补充 `parsers.py` 的字段别名。
6. 运行：

```powershell
python -m jd_spider --sku 100012043978 --mode api --out ./data
```

### 11.3 明确限制

- **模式 A 不采集商品基础信息。** 官方评论 API 通常只返回评论；`fetch_product()` 默认返回 `None`。需要商品标题、价格、主图等，请使用模式 B。
- 项目不提供可直接用于生产的京东签名实现。
- `base_url`、`comment_path`、`method` 和参数名都是可替换的占位/配置项，不能视为当前京东接口的保证。
- 未配置签名时，程序会抛出明确的 `ApiNotImplementedError`，退出码为 `3`，不会发送伪造签名的请求。

---

## 12. 断点续跑、去重与失败处理

### 去重

- 启动时读取已有 `comments.jsonl`，建立 `comment_id` 集合。
- 同一次运行内也有会话级去重集合。
- 已存在的评论不会重复追加。
- 图片映射以 `(local_path, image_type)` 去重。

### 续跑

- 直接使用相同 `--out` 再次运行即可。
- 已存在的有效图片文件会跳过，不重复下载。
- `.part` 临时文件不会被当成已完成图片。
- 评论写入发生在图片处理之后；如果中途中断，重新运行会补齐缺失图片并写入评论。

### 单点失败

- 单条评论解析或图片下载失败只记录日志并计数，不终止整个 SKU。
- 单个 SKU 异常不影响后续 SKU。
- 登录失效或明确触发安全验证时，程序会停止当前批次，避免继续请求。
- 不支持多个进程同时写同一个输出目录。

### 写入安全

- JSONL 追加后执行 flush + `fsync`。
- 商品记录使用临时文件 + `os.replace` 原子更新。
- 图片先写 `.part`，校验后原子重命名。

---

## 13. 日志与退出码

默认日志：

```text
<输出目录>/logs/jd_spider.log
```

同时输出到控制台。日志不会打印 Cookie、AppSecret 或 access_token；URL 日志会去掉查询串，避免泄露签名参数。

命令行覆盖：

```powershell
python -m jd_spider --sku 100012043978 --log-level DEBUG --log-file .\data\logs\run.log
```

退出码：

| 退出码 | 含义 |
|---|---|
| `0` | 成功 |
| `1` | 部分 SKU 失败或一般运行错误 |
| `2` | 配置/参数错误，或模式 A 缺少凭证 |
| `3` | 模式 A 未实现签名/API 响应错误 |
| `4` | 浏览器不可用、需要登录、访问被阻断 |
| `130` | 用户中断（Ctrl+C），已完成数据保留 |

---

## 14. 测试

项目包含解析器、存储和图片下载的单元测试。下载测试使用本地 HTTP 服务，不会访问京东。

```powershell
python -m compileall -q jd_spider tests
python -m unittest discover -s tests -v
```

当前测试覆盖：

- JSON / JSONP 解码
- 评论、追评、图片、头像解析
- DOM 行解析
- 商品 HTML 兜底
- 评论去重、图片映射去重、商品 UPSERT
- 下载 Referer、图片校验、重复下载跳过

未包含需要真实登录京东的端到端自动化测试，因为登录和安全验证必须由真人完成。

---

## 15. 常见问题（FAQ）

### Q1：图片下载 403 或下载到 HTML

可按顺序检查：

1. 图片是否需要登录态。浏览器模式可先扫码登录；HTTP 下载可设置 `JD_COOKIE`。
2. 图片 URL 是否需要 `Referer`。下载器默认尝试 `https://item.jd.com/{sku}.html`、`https://club.jd.com/`、`https://www.jd.com/`。
3. 是否请求过快。保持默认 2~5 秒限速，或调大 `MIN_DELAY` / `MAX_DELAY`。
4. 图片是否已失效或需要特定 CDN 参数。查看日志中的 URL（查询串会被隐藏）和 HTTP 状态。
5. 程序会拒绝 HTML 响应和非图片内容，不会把错误页保存成 `.jpg`。

### Q2：登录失效了怎么办

重新运行：

```powershell
python -m jd_spider --login-only --no-headless
```

完成扫码后，再执行采集。一般不需要删除 `.jd_profile`；只有 profile 损坏或需要彻底清理登录态时才考虑备份后重建。

### Q3：评论区一直加载不出来

1. 确认使用非无头模式，先观察页面是否正常显示评论。
2. 手动滚动到评论区域，看是否会自然发起请求。
3. 把日志级别调到 `DEBUG`，检查是否捕获到评论接口。
4. 检查 `browser.comment_url_patterns` 是否覆盖当前接口 URL。
5. 尝试调大 `NETWORK_WAIT`、`SCROLL_PAUSE`、`MAX_SCROLLS`。
6. 若网络监听完全无结果，保留 `DOM_FALLBACK=true`，让程序尝试 DOM 兜底。DOM 结果可能缺少评论 ID 或完整图片。
7. 页面按钮文案变化时，调整 `browser.expand_texts`。

### Q4：出现验证码、滑块或“安全验证”

程序不会自动绕过。请在浏览器中手动完成验证。若需要更长时间，调大：

```dotenv
VERIFICATION_TIMEOUT=300
```

无头模式无法人工处理验证，请改用非无头模式。

### Q5：浏览器启动失败或端口被占用

- 默认调试端口是 `9222`。关闭占用该端口的 Chrome，或修改 `BROWSER_PORT` / `--browser-port`。
- 如果 DrissionPage 找不到浏览器，设置 `BROWSER_PATH`。
- 持久化 profile 的目录必须可写。
- 不要同时启动两个使用相同 `USER_DATA_PATH` 和端口的程序。
- 本项目特意使用固定端口 + 持久化 profile；不要改成 `auto_port()`，否则 DrissionPage 可能在断开时清理 profile 目录。

### Q6：为什么有评论但没有商品价格/品牌/店铺

商品信息来自页面渲染结果，部分字段通过异步接口加载。程序会二次尝试，但仍可能缺失。可在页面加载完成后再运行，或调整 `PRODUCT_INFO_JS` 以匹配当前页面结构。

### Q7：为什么模式 A 没有商品信息

模式 A 是评论 API 适配器，默认不提供商品信息。商品采集请用模式 B。

### Q8：为什么评论图片比页面看到的少

- 图片可能位于异步懒加载接口中，需要继续滚动。
- 部分图片可能需要登录或特定 Referer。
- 下载失败不会写入 `images_local_paths`，但原始 URL 仍保留在 `images_urls`。
- “晒单”筛选默认不点击，避免丢失纯文字评论；图片依赖接口字段提取。

### Q9：评论 ID 以 `auto_` 开头

说明原始数据没有可识别的评论 ID，程序用昵称、时间、内容、首图等生成稳定 ID。它可用于本工具去重，但不应当作京东官方评论 ID。

### Q10：能否多进程/多账号并发

不支持多进程写同一输出目录。请控制并发和频率，并遵守平台规则。本项目没有内置账号池、代理池或风控规避功能。

---

## 16. 最小验收流程

1. 安装依赖并准备 Chrome/Edge。
2. 复制 `.env.example` → `.env`，保持 `HEADLESS=false`。
3. 执行 `python -m jd_spider --login-only --no-headless` 并扫码。
4. 执行：

   ```powershell
   python -m jd_spider --sku 100012043978 --out ./data --mode browser --max-pages 50
   ```

5. 检查：
   - `data/products.jsonl` 有该 SKU；
   - `data/comments.jsonl` 有评论且 `comment_id` 不重复；
   - `data/images/comments/100012043978/` 和 `data/images/avatars/100012043978/` 中图片可正常打开；
   - `data/image_mapping.jsonl` 能找到 URL 与本地路径的对应关系。
6. 再次执行同一命令，确认评论不重复、已下载图片被跳过。

---

## 17. 免责声明

本项目用于授权范围内的数据采集与技术研究。使用者应自行确保：

- 已获得必要的授权；
- 遵守京东服务条款、robots 规则和当地法律；
- 遵守个人信息保护要求，不采集、传播或滥用他人隐私数据；
- 不将本项目用于绕过安全机制、攻击服务、批量账号操作或其他不当用途。

作者不对使用者的采集行为及后果承担责任。
