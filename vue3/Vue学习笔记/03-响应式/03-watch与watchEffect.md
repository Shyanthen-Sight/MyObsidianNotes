# watch 与 watchEffect

> `watch` 用于监听特定数据源并在变化时执行副作用；`watchEffect` 会自动收集回调中用到的响应式依赖，适合依赖关系不固定的场景。

## 一、核心概念

- **惰性 vs 立即执行**：`watch` 默认只在数据变化时执行，需要首次执行要加 `immediate: true`；`watchEffect` 会立即执行一次，并自动追踪依赖。
- **监听源**：`watch` 可以监听 ref、getter 函数、reactive 对象、ref 数组或 getter 数组。监听 reactive 对象时默认是深层监听。
- **新旧值**：回调参数是 `(newValue, oldValue)`。监听 reactive 对象时，新旧值是同一个代理引用，无法直接比较差异。
- **清理副作用**：回调第三个参数 `onCleanup` 用于取消上一次的请求、定时器或事件监听，避免竞态和内存泄漏。
- **执行时机**：`flush: 'pre'` 在 DOM 更新前执行（默认），`'post'` 在 DOM 更新后执行，`'sync'` 同步执行但性能较差。
- **停止监听**：`watch` 和 `watchEffect` 都返回停止函数，组件卸载时 Vue 会自动停止，但在组件外创建时需要手动调用。

---

## 二、基础语法

```vue
<script setup>
import { ref, watch, watchEffect } from 'vue'

const keyword = ref('')
const page = ref(1)

// 监听单个 ref
const stopKeyword = watch(keyword, (newValue, oldValue) => {
  console.log('关键词变化', oldValue, '->', newValue)
})

// 监听 getter，适合组合多个值
watch(
  () => `${keyword.value}-${page.value}`,
  (value) => {
    console.log('查询条件变化', value)
  }
)

// 监听多个源
watch([keyword, page], ([nextKeyword, nextPage], [prevKeyword, prevPage]) => {
  console.log('多源变化', { nextKeyword, nextPage, prevKeyword, prevPage })
})

// 立即执行 + 深度监听 + 一次性
watch(
  keyword,
  () => console.log('只触发一次'),
  { once: true }
)

// 自动追踪依赖，立即执行一次
const stopEffect = watchEffect(() => {
  console.log('当前关键词', keyword.value)
})

// 手动停止
// stopKeyword()
// stopEffect()
</script>
```

---

## 三、完整示例

下面是一个订单搜索组件，使用 `watch` 做防抖、取消上一次请求、处理页码重置，并用 `watchEffect` 同步浏览器标题：

```vue
<script setup>
import { onBeforeUnmount, ref, watch, watchEffect } from 'vue'

const keyword = ref('')
const status = ref('all')
const page = ref(1)
const pageSize = 10
const orders = ref([])
const loading = ref(false)
const errorMessage = ref('')

let debounceTimer = null
let controller = null

async function fetchOrders(signal) {
  loading.value = true
  errorMessage.value = ''

  try {
    // 模拟接口请求，真实项目替换为 fetch/axios
    await new Promise((resolve) => setTimeout(resolve, 500))

    if (signal.aborted) return

    const start = (page.value - 1) * pageSize
    orders.value = Array.from({ length: pageSize }, (_, index) => {
      const id = start + index + 1
      return {
        id,
        orderNo: `SO2026${String(id).padStart(4, '0')}`,
        customer: `客户 ${id}`,
        status: status.value === 'all' ? 'paid' : status.value
      }
    })
  } catch (error) {
    if (error.name !== 'AbortError') {
      errorMessage.value = '订单加载失败'
    }
  } finally {
    loading.value = false
  }
}

// 关键词变化时回到第一页
watch(keyword, () => {
  page.value = 1
})

// 查询条件变化：统一防抖 300ms，合并关键词、状态和页码的连续变化
watch([keyword, status, page], () => {
  if (debounceTimer) window.clearTimeout(debounceTimer)

  debounceTimer = window.setTimeout(() => {
    controller?.abort()
    controller = new AbortController()
    fetchOrders(controller.signal)
  }, 300)
}, { immediate: true })

// 自动同步页面标题，依赖变化时自动重跑
watchEffect(() => {
  document.title = keyword.value
    ? `${keyword.value} - 订单搜索`
    : '订单管理'
})

onBeforeUnmount(() => {
  if (debounceTimer) window.clearTimeout(debounceTimer)
  controller?.abort()
})
</script>

<template>
  <section class="orders">
    <header>
      <h1>订单管理</h1>
      <input v-model.trim="keyword" type="search" placeholder="搜索订单号或客户" />
    </header>

    <div class="filters">
      <select v-model="status">
        <option value="all">全部状态</option>
        <option value="paid">已支付</option>
        <option value="shipped">已发货</option>
        <option value="done">已完成</option>
      </select>

      <button type="button" :disabled="page <= 1" @click="page -= 1">
        上一页
      </button>
      <span>第 {{ page }} 页</span>
      <button type="button" @click="page += 1">下一页</button>
    </div>

    <p v-if="loading" class="state">加载中...</p>
    <p v-else-if="errorMessage" class="error">{{ errorMessage }}</p>

    <ul v-else>
      <li v-for="order in orders" :key="order.id">
        <span>{{ order.orderNo }}</span>
        <span>{{ order.customer }}</span>
        <span>{{ order.status }}</span>
      </li>
    </ul>
  </section>
</template>

<style scoped>
.orders {
  max-width: 720px;
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
  width: 260px;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.filters {
  display: flex;
  align-items: center;
  gap: 10px;
  margin: 18px 0;
}

.filters select {
  padding: 8px 10px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.filters button {
  padding: 8px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

.filters button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

ul {
  margin: 0;
  padding: 0;
  list-style: none;
}

li {
  display: grid;
  grid-template-columns: 1.2fr 1fr 0.6fr;
  padding: 12px;
  border-bottom: 1px solid #f1f5f9;
  font-size: 14px;
}

.state,
.error {
  padding: 32px;
  text-align: center;
}

.state {
  color: #9ca3af;
}

.error {
  color: #dc2626;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **监听 reactive 对象时新旧值相同**：`watch(form, (newValue, oldValue) => ...)` 中两者都是同一个代理，想比较差异要监听具体字段的 getter。
2. **直接监听解构后的普通值无效**：`const { keyword } = state` 得到的是字符串，`watch(keyword, ...)` 不会触发；应监听 `() => state.keyword`。
3. **`watchEffect` 的依赖是动态的**：回调中条件分支可能导致某些依赖未被收集，依赖关系复杂时优先使用 `watch` 显式声明。
4. **`watchEffect` 会立即执行**：不要在未初始化数据时访问 `undefined` 的属性，必要时先判空或用 `watch` 加 `immediate`。
5. **异步请求要处理竞态**：快速切换关键词时，旧请求可能后返回并覆盖新结果；使用 `onCleanup` + `AbortController` 取消旧请求。
6. **`deep: true` 有性能成本**：大对象深度监听会递归遍历所有属性，能监听具体字段就不要深度监听。
7. **`flush: 'post'` 才能拿到更新后的 DOM**：默认 `'pre'` 在 DOM 更新前执行，读取 `$el` 尺寸可能得到旧值。
8. **组件外创建的监听要手动停止**：在路由守卫、定时器或全局 store 中创建的 `watch` 不会自动随组件卸载，应保存停止函数并在适当时机调用。

---

## 五、官方文档

- 侦听器：https://cn.vuejs.org/guide/essentials/watchers.html
- `watch` API：https://cn.vuejs.org/api/reactivity-core.html#watch
- `watchEffect` API：https://cn.vuejs.org/api/reactivity-core.html#watcheffect
- 副作用清理：https://cn.vuejs.org/guide/essentials/watchers.html#side-effect-cleanup
- 回调触发时机：https://cn.vuejs.org/guide/essentials/watchers.html#callback-flush-timing

---

## 六、速查表

| 语法                     | 作用           | 示例                                     |
| ------------------------ | -------------- | ---------------------------------------- |
| `watch(source, cb)`      | 监听指定数据源 | `watch(keyword, handler)`                |
| `watch(() => obj.x, cb)` | 监听对象字段   | `watch(() => form.name, cb)`             |
| `watch([a, b], cb)`      | 监听多个源     | `watch([keyword, page], handler)`        |
| `{ immediate: true }`    | 立即执行一次   | `watch(source, cb, { immediate: true })` |
| `{ deep: true }`         | 深度监听       | `watch(obj, cb, { deep: true })`         |
