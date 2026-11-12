# TypeScript 与组合式 API

> 在 Vue 3 中使用 TypeScript，可以在 props、emits、ref、注入和组件模板上获得类型检查与编辑器补全，减少运行时错误。

## 一、核心概念

- **`lang="ts"`**：在 `<script setup lang="ts">` 中启用 TypeScript，Vite 通过 esbuild 转译，类型检查由 `vue-tsc` 负责。
- **props 类型**：`defineProps<{ title: string }>()` 用接口声明 props，`withDefaults` 为可选属性提供默认值。
- **emits 类型**：`defineEmits<{ save: [payload: Order] }>()` 用元组声明事件参数，`emit('save', order)` 会得到参数校验。
- **ref 泛型**：`ref<Order | null>(null)` 明确初始值和可空性，避免 `ref(null)` 被推断为 `null`。
- **注入类型**：使用 `InjectionKey<T>` 定义 Symbol key，`provide` / `inject` 两端共享类型。
- **模板类型检查**：`vue-tsc --noEmit` 会检查模板中的属性、事件和插槽类型，应纳入 CI。

---

## 二、基础语法

```vue
<script setup lang="ts">
import { computed, ref } from 'vue'

interface Order {
  id: number
  no: string
  amount: number
  status: 'paid' | 'shipped' | 'done'
}

const props = withDefaults(
  defineProps<{
    orders: Order[]
    currency?: string
  }>(),
  {
    currency: 'CNY'
  }
)

const emit = defineEmits<{
  select: [order: Order]
  remove: [id: number]
}>()

const keyword = ref('')
const selected = ref<Order | null>(null)

const total = computed(() => {
  return props.orders.reduce((sum, order) => sum + order.amount, 0)
})

function selectOrder(order: Order) {
  selected.value = order
  emit('select', order)
}
</script>
```

注入类型：

```ts
// keys.ts
import type { InjectionKey, Ref } from 'vue'

export interface User {
  id: number
  name: string
}

export const userKey: InjectionKey<Ref<User | null>> = Symbol('user')
```

```ts
// 祖先
provide(userKey, ref<User | null>(null))

// 后代
const user = inject(userKey, ref(null))
```

`tsconfig` 中常见配置：

```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "preserve",
    "moduleResolution": "Bundler",
    "types": ["vite/client"],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.vue"]
}
```

---

## 三、完整示例

下面是一个类型安全的订单表格组件，包含泛型列定义、props/emits 类型、可空选中项和状态映射：

```vue
<script setup lang="ts">
import { computed, ref } from 'vue'

type OrderStatus = 'paid' | 'shipped' | 'done' | 'cancelled'

interface Order {
  id: number
  no: string
  customer: string
  amount: number
  status: OrderStatus
  createdAt: string
}

interface Props {
  orders: Order[]
  loading?: boolean
  pageSize?: number
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  pageSize: 5
})

const emit = defineEmits<{
  select: [order: Order]
  'update-status': [id: number, status: OrderStatus]
}>()

const keyword = ref('')
const status = ref<OrderStatus | 'all'>('all')
const page = ref(1)
const selectedId = ref<number | null>(null)

const statusText: Record<OrderStatus, string> = {
  paid: '已支付',
  shipped: '已发货',
  done: '已完成',
  cancelled: '已取消'
}

const statusOptions: Array<{ value: OrderStatus | 'all'; text: string }> = [
  { value: 'all', text: '全部状态' },
  { value: 'paid', text: statusText.paid },
  { value: 'shipped', text: statusText.shipped },
  { value: 'done', text: statusText.done },
  { value: 'cancelled', text: statusText.cancelled }
]

const filtered = computed(() => {
  const text = keyword.value.trim().toLowerCase()

  return props.orders.filter((order) => {
    const matchKeyword = !text
      || order.no.toLowerCase().includes(text)
      || order.customer.toLowerCase().includes(text)

    const matchStatus = status.value === 'all' || order.status === status.value

    return matchKeyword && matchStatus
  })
})

const totalPages = computed(() => {
  return Math.max(1, Math.ceil(filtered.value.length / props.pageSize))
})

const pageOrders = computed(() => {
  const start = (page.value - 1) * props.pageSize
  return filtered.value.slice(start, start + props.pageSize)
})

const selectedOrder = computed(() => {
  return props.orders.find((order) => order.id === selectedId.value) ?? null
})

function select(order: Order) {
  selectedId.value = order.id
  emit('select', order)
}

function markDone(order: Order) {
  emit('update-status', order.id, 'done')
}

function prevPage() {
  if (page.value > 1) page.value -= 1
}

function nextPage() {
  if (page.value < totalPages.value) page.value += 1
}
</script>

<template>
  <section class="order-table">
    <header>
      <div>
        <h1>订单列表</h1>
        <p v-if="selectedOrder">已选：{{ selectedOrder.no }}</p>
      </div>

      <input v-model.trim="keyword" type="search" placeholder="搜索订单号或客户" />
    </header>

    <div class="filters">
      <select v-model="status">
        <option
          v-for="option in statusOptions"
          :key="option.value"
          :value="option.value"
        >
          {{ option.text }}
        </option>
      </select>
    </div>

    <p v-if="loading" class="state">加载中...</p>

    <table v-else>
      <thead>
        <tr>
          <th>订单号</th>
          <th>客户</th>
          <th>金额</th>
          <th>状态</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="order in pageOrders"
          :key="order.id"
          :class="{ selected: order.id === selectedId }"
          @click="select(order)"
        >
          <td>{{ order.no }}</td>
          <td>{{ order.customer }}</td>
          <td>¥{{ order.amount }}</td>
          <td>{{ statusText[order.status] }}</td>
          <td>
            <button
              type="button"
              :disabled="order.status === 'done'"
              @click.stop="markDone(order)"
            >
              标记完成
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <footer v-if="!loading">
      <button type="button" :disabled="page <= 1" @click="prevPage">上一页</button>
      <span>{{ page }} / {{ totalPages }}</span>
      <button type="button" :disabled="page >= totalPages" @click="nextPage">
        下一页
      </button>
    </footer>
  </section>
</template>

<style scoped>
.order-table {
  max-width: 820px;
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
  margin: 0 0 4px;
  font-size: 22px;
}

header p {
  margin: 0;
  color: #6b7280;
  font-size: 13px;
}

header input {
  width: 240px;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.filters {
  margin: 14px 0;
}

.filters select {
  padding: 8px 10px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

th,
td {
  padding: 11px 10px;
  border-bottom: 1px solid #f1f5f9;
  text-align: left;
}

th {
  background: #f9fafb;
  color: #6b7280;
  font-size: 13px;
}

tbody tr {
  cursor: pointer;
}

tbody tr:hover,
tbody tr.selected {
  background: #f0fdf4;
}

td button {
  padding: 5px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
  font-size: 12px;
}

td button:disabled {
  cursor: not-allowed;
  opacity: 0.4;
}

footer {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-top: 18px;
}

footer button {
  padding: 7px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

.state {
  padding: 32px;
  color: #9ca3af;
  text-align: center;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **`defineProps` 的类型参数不能是运行时值**：类型声明只支持本地接口或类型别名，不能引用外部运行时变量。
2. **`withDefaults` 只用于类型声明**：运行时声明 `defineProps({ count: Number })` 直接写 `default` 即可。
3. **`ref(null)` 会被推断为 `null`**：需要赋其他类型时写 `ref<Order | null>(null)`。
4. **事件类型用元组**：`{ save: [order: Order] }` 表示一个参数；多个参数写成 `[id: number, status: string]`。
5. **`reactive` 的泛型不要过度嵌套**：`reactive<Order[]>([])` 等写法可能丢失深层类型推导，复杂结构优先用 `ref`。
6. **模板中的类型检查需要 `vue-tsc`**：单纯 `tsc` 不会检查 `.vue` 模板，CI 中应运行 `vue-tsc --noEmit`。
7. **不要用 `any` 绕过错误**：`any` 会传播到调用方，最终失去类型安全；用 `unknown` 加类型守卫更稳妥。
8. **`strictNullChecks` 下要处理可空**：DOM 引用、接口返回、`inject` 默认值都可能是 `null`/`undefined`，使用前判空。

---

## 五、官方文档

- TypeScript 概览：https://cn.vuejs.org/guide/typescript/overview.html
- 组合式 API 与 TypeScript：https://cn.vuejs.org/guide/typescript/composition-api.html
- `defineProps` 类型声明：https://cn.vuejs.org/api/sfc-script-setup.html#type-only-props-emit-declarations
- `InjectionKey`：https://cn.vuejs.org/api/composition-api-dependency-injection.html#injectionkey

---

## 六、速查表

| 语法                               | 作用              | 示例                                                 |
| ---------------------------------- | ----------------- | ---------------------------------------------------- |
| `lang="ts"`                        | 启用 TypeScript   | `<script setup lang="ts">`                           |
| `defineProps<Props>()`             | 类型声明 props    | `defineProps<{ title: string }>()`                   |
| `withDefaults`                     | 类型 props 默认值 | `withDefaults(defineProps<Props>(), { size: 'md' })` |
| `defineEmits<{ save: [Order] }>()` | 类型声明事件      | `emit('save', order)`                                |
| `ref<Order \| null>(null)`         | 明确可空类型      | `const current = ref<Order \| null>(null)`           |
