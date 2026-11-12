# 数据请求与 API 调用

> 前端应用的数据大多来自后端接口。这一节整理 `fetch`、Axios、Vue Query 三种常见方案，以及加载、错误、取消和竞态处理。

## 一、核心概念

- **`fetch`**：浏览器原生 API，返回 Promise，需要手动处理 HTTP 状态码、JSON 解析和超时。适合轻量请求或不想引入依赖的项目。
- **Axios**：社区常用的 HTTP 客户端，提供拦截器、自动 JSON 转换、超时、取消等能力，适合需要统一认证、错误处理的中大型项目。
- **TanStack Query（Vue Query）**：专注服务端状态管理，提供缓存、去重、重试、失效、分页和乐观更新，减少手写 loading/error 逻辑。
- **GraphQL 客户端**：后端使用 GraphQL 时，可用 Apollo Client 或 `@vue/apollo-composable`，通过 `useQuery` / `useMutation` 获取数据；REST 项目不需要为了“新”而引入 GraphQL。
- **取消请求**：`fetch` 使用 `AbortController`，Axios 使用 `signal` 或 `CancelToken`。组件卸载或条件变化时应取消未完成请求。
- **竞态问题**：后发请求可能先返回，覆盖先发请求的结果。解决方式是取消旧请求，或给每次请求打上序号只接受最新结果。
- **环境变量**：接口地址通过 `import.meta.env.VITE_API_BASE_URL` 注入，不要在前端代码里硬编码生产域名或密钥。

---

## 二、基础语法

`fetch` 封装：

```js
const controller = new AbortController()

const response = await fetch('/api/orders', {
  signal: controller.signal,
  headers: {
    'Content-Type': 'application/json'
  }
})

if (!response.ok) {
  throw new Error(`请求失败：${response.status}`)
}

const data = await response.json()
```

Axios 实例：

```js
// api/http.js
import axios from 'axios'

export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000
})

http.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

http.interceptors.response.use(
  (response) => response.data,
  (error) => {
    if (error.response?.status === 401) {
      // 跳转登录
    }
    return Promise.reject(error)
  }
)
```

Vue Query：

```js
import { useQuery } from '@tanstack/vue-query'
import { computed, ref } from 'vue'

const keyword = ref('')

const { data, isPending, isError, error, refetch } = useQuery({
  queryKey: computed(() => ['orders', keyword.value]),
  queryFn: async () => {
    const response = await fetch(`/api/orders?keyword=${keyword.value}`)
    if (!response.ok) throw new Error('加载失败')
    return response.json()
  },
  staleTime: 30_000
})
```

---

## 三、完整示例

下面是一个客户列表页面，使用 `fetch` + `AbortController` 实现搜索防抖、取消旧请求、加载态和错误重试：

```vue
<script setup>
import { onBeforeUnmount, ref, watch } from 'vue'

const keyword = ref('')
const customers = ref([])
const loading = ref(false)
const errorMessage = ref('')
const page = ref(1)
const hasMore = ref(true)

let controller = null
let debounceTimer = null

async function loadCustomers({ append = false } = {}) {
  controller?.abort()
  controller = new AbortController()

  loading.value = true
  errorMessage.value = ''

  try {
    const params = new URLSearchParams({
      keyword: keyword.value.trim(),
      page: String(page.value),
      pageSize: '10'
    })

    const response = await fetch(`/api/customers?${params}`, {
      signal: controller.signal,
      headers: { Accept: 'application/json' }
    })

    if (!response.ok) {
      throw new Error(`接口返回 ${response.status}`)
    }

    const data = await response.json()
    const list = Array.isArray(data.list) ? data.list : []

    customers.value = append ? [...customers.value, ...list] : list
    hasMore.value = list.length === 10
  } catch (error) {
    if (error.name === 'AbortError') return
    errorMessage.value = error.message || '加载失败，请稍后重试'
  } finally {
    loading.value = false
  }
}

watch(keyword, () => {
  page.value = 1
  hasMore.value = true

  if (debounceTimer) window.clearTimeout(debounceTimer)

  debounceTimer = window.setTimeout(() => {
    loadCustomers()
  }, 300)
})

function loadMore() {
  page.value += 1
  loadCustomers({ append: true })
}

function retry() {
  loadCustomers()
}

loadCustomers()

onBeforeUnmount(() => {
  controller?.abort()
  if (debounceTimer) window.clearTimeout(debounceTimer)
})
</script>

<template>
  <section class="customers">
    <header>
      <h1>客户列表</h1>
      <input v-model.trim="keyword" type="search" placeholder="搜索姓名或公司" />
    </header>

    <p v-if="loading" class="state">加载中...</p>

    <div v-else-if="errorMessage" class="state error">
      <p>{{ errorMessage }}</p>
      <button type="button" @click="retry">重试</button>
    </div>

    <ul v-else>
      <li v-for="customer in customers" :key="customer.id">
        <div class="avatar">{{ customer.name.slice(0, 1) }}</div>
        <div class="info">
          <strong>{{ customer.name }}</strong>
          <span>{{ customer.company }}</span>
        </div>
        <span class="level">{{ customer.level }}</span>
      </li>
    </ul>

    <p v-if="!loading && !errorMessage && customers.length === 0" class="state">
      没有匹配的客户
    </p>

    <button
      v-if="hasMore && !loading && !errorMessage"
      type="button"
      class="more"
      @click="loadMore"
    >
      加载更多
    </button>
  </section>
</template>

<style scoped>
.customers {
  max-width: 680px;
  margin: 40px auto;
  padding: 24px;
  font-family: system-ui, sans-serif;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
}

h1 {
  margin: 0;
  font-size: 22px;
}

header input {
  width: 240px;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

ul {
  margin: 18px 0 0;
  padding: 0;
  list-style: none;
}

li {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 0;
  border-bottom: 1px solid #f1f5f9;
}

.avatar {
  display: grid;
  width: 40px;
  height: 40px;
  place-items: center;
  border-radius: 50%;
  background: #ecfdf5;
  color: #047857;
  font-size: 15px;
}

.info {
  display: grid;
  flex: 1;
  gap: 4px;
}

.info span {
  color: #6b7280;
  font-size: 13px;
}

.level {
  padding: 2px 8px;
  border-radius: 999px;
  background: #f3f4f6;
  color: #4b5563;
  font-size: 12px;
}

.state {
  padding: 32px;
  color: #9ca3af;
  text-align: center;
}

.error {
  color: #dc2626;
}

.error button,
.more {
  padding: 8px 16px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

.more {
  display: block;
  margin: 18px auto 0;
}
</style>
```

Axios 版本的核心请求函数：

```js
import { http } from '@/api/http'

export async function fetchCustomers({ keyword, page, pageSize = 10 }, signal) {
  return http.get('/customers', {
    params: { keyword, page, pageSize },
    signal
  })
}
```

---

## 四、常见坑 / 注意事项

1. **`fetch` 不会因为 4xx/5xx 而 reject**：只有网络错误才 reject，必须手动检查 `response.ok`。
2. **组件卸载后仍 setState 会报警告或浪费内存**：在 `onBeforeUnmount` 中取消请求，或使用 `AbortController` 中断。
3. **快速搜索会产生竞态**：即使取消旧请求，也可能有请求已经返回；用请求序号或 Vue Query 的 `queryKey` 机制保证只采用最新结果。
4. **不要把 token 写在前端源码里**：环境变量会被打包进客户端代码，敏感密钥必须放在后端或服务端代理。
5. **Axios 拦截器要处理非 2xx 和业务错误码**：HTTP 200 但 `code !== 0` 的情况很常见，统一在响应拦截器里抛出业务错误。
6. **`useQuery` 的 `queryKey` 要包含所有影响结果的参数**：漏掉关键词或分页会导致缓存命中错误数据。
7. **请求失败要提供重试入口**：只显示“加载失败”会让用户无路可走；重试按钮或自动重试策略要按业务选择。
8. **跨域问题不是前端能单独解决的**：开发环境用 Vite 代理，生产环境需要后端配置 CORS 或同源部署。

---

## 五、官方文档

- Vue 官方工具链与数据请求：https://cn.vuejs.org/guide/scaling-up/tooling.html
- Fetch API：https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API
- AbortController：https://developer.mozilla.org/zh-CN/docs/Web/API/AbortController
- Axios 文档：https://axios-http.com/zh/docs/intro
- TanStack Query Vue：https://tanstack.com/query/latest/docs/framework/vue/overview
- Vue Apollo：https://apollo.vuejs.org/

---

## 六、速查表

| 语法              | 作用           | 示例                                        |
| ----------------- | -------------- | ------------------------------------------- |
| `fetch(url)`      | 发起请求       | `await fetch('/api/list')`                  |
| `response.ok`     | 判断 HTTP 状态 | `if (!response.ok) throw ...`               |
| `response.json()` | 解析 JSON      | `const data = await response.json()`        |
| `AbortController` | 取消请求       | `fetch(url, { signal: controller.signal })` |
| `axios.create`    | 创建实例       | `axios.create({ baseURL, timeout })`        |
