# 组件事件 Emits

> 子组件通过 `emits` 向父组件报告“发生了什么”，父组件用 `@事件名` 监听。它是 props 之外另一条核心通信通道。

## 一、核心概念

- **事件声明**：`defineEmits(['submit'])` 声明组件会派发的事件，Vue 会据此区分组件事件和原生事件。
- **派发事件**：`emit('submit', payload)`，第一个参数是事件名，后续参数会原样传给父组件的监听器。
- **事件命名**：推荐使用 kebab-case，如 `update-user`、`item-selected`。在模板中监听时写 `@update-user`，与 HTML 属性大小写不敏感的规则兼容。
- **事件校验**：对象形式 `defineEmits({ submit: (payload) => ... })` 可以在开发环境校验参数，返回 `false` 会给出警告。
- **与 props 的分工**：props 向下传数据，emits 向上报变化。子组件不应既接收 prop 又直接修改它。
- **原生事件透传**：未在 `emits` 中声明的事件监听器会作为原生监听器落到组件根元素上；声明后则视为组件自定义事件。

---

## 二、基础语法

```vue
<script setup>
// 数组形式：只声明事件名
const emit = defineEmits(['save', 'cancel'])

function handleSave() {
  emit('save', { id: 1, title: '季度总结' })
}

function handleCancel() {
  emit('cancel')
}
</script>

<template>
  <button @click="handleSave">保存</button>
  <button @click="handleCancel">取消</button>
</template>
```

带校验的对象形式：

```vue
<script setup>
const emit = defineEmits({
  'select-user': (user) => {
    if (!user || typeof user.id !== 'number') {
      console.warn('select-user 需要一个带 id 的用户对象')
      return false
    }
    return true
  }
})
</script>
```

TypeScript 元组形式：

```vue
<script setup lang="ts">
const emit = defineEmits<{
  save: [payload: { id: number; title: string }]
  cancel: []
}>()

emit('save', { id: 1, title: '季度总结' })
</script>
```

父组件监听：

```vue
<template>
  <ArticleEditor @save="handleSave" @cancel="handleCancel" />
</template>
```

---

## 三、完整示例

下面是一个搜索工具栏组件，包含关键词输入、排序切换、清空和搜索事件，展示事件参数、校验和父组件协作：

```vue
<script setup>
import { ref } from 'vue'

const props = defineProps({
  placeholder: {
    type: String,
    default: '搜索订单号、客户或商品'
  },
  loading: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits({
  search: (payload) => {
    return payload && typeof payload.keyword === 'string'
  },
  'sort-change': (sort) => {
    return ['latest', 'amount'].includes(sort)
  },
  reset: () => true
})

const keyword = ref('')
const sort = ref('latest')

function submitSearch() {
  emit('search', {
    keyword: keyword.value.trim(),
    sort: sort.value
  })
}

function changeSort(nextSort) {
  sort.value = nextSort
  emit('sort-change', nextSort)
  // 排序变化后立即按新条件搜索
  emit('search', {
    keyword: keyword.value.trim(),
    sort: nextSort
  })
}

function reset() {
  keyword.value = ''
  sort.value = 'latest'
  emit('reset')
  emit('search', { keyword: '', sort: 'latest' })
}
</script>

<template>
  <section class="search-bar">
    <div class="input-wrap">
      <input
        v-model="keyword"
        type="search"
        :placeholder="placeholder"
        :disabled="loading"
        @keyup.enter="submitSearch"
      />
      <button type="button" :disabled="loading" @click="submitSearch">
        {{ loading ? '搜索中' : '搜索' }}
      </button>
    </div>

    <div class="sort-group">
      <button
        type="button"
        :class="{ active: sort === 'latest' }"
        @click="changeSort('latest')"
      >
        最新
      </button>
      <button
        type="button"
        :class="{ active: sort === 'amount' }"
        @click="changeSort('amount')"
      >
        金额
      </button>
      <button type="button" class="reset" @click="reset">重置</button>
    </div>
  </section>
</template>

<style scoped>
.search-bar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  padding: 14px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #ffffff;
  font-family: system-ui, sans-serif;
}

.input-wrap {
  display: flex;
  flex: 1;
  min-width: 260px;
  gap: 8px;
}

.input-wrap input {
  flex: 1;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  outline: none;
}

.input-wrap input:focus {
  border-color: #111827;
}

.input-wrap button,
.sort-group button {
  padding: 8px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

.input-wrap button {
  border-color: #111827;
  background: #111827;
  color: #ffffff;
}

.input-wrap button:disabled,
.input-wrap input:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

.sort-group {
  display: flex;
  gap: 6px;
}

.sort-group button.active {
  border-color: #42b883;
  background: #ecfdf5;
  color: #047857;
}

.sort-group .reset {
  color: #dc2626;
}
</style>
```

父组件接收事件并请求数据：

```vue
<script setup>
import { ref } from 'vue'
import SearchBar from './components/SearchBar.vue'

const loading = ref(false)
const orders = ref([])

async function handleSearch({ keyword, sort }) {
  loading.value = true
  try {
    // 实际项目替换为接口请求
    const params = new URLSearchParams({ keyword, sort })
    const response = await fetch(`/api/orders?${params}`)
    orders.value = await response.json()
  } finally {
    loading.value = false
  }
}

function handleReset() {
  orders.value = []
}
</script>

<template>
  <SearchBar
    :loading="loading"
    @search="handleSearch"
    @reset="handleReset"
  />
</template>
```

选项式 API 对照：

```vue
<script>
export default {
  emits: {
    search(payload) {
      return Boolean(payload && typeof payload.keyword === 'string')
    }
  },
  methods: {
    submitSearch(keyword) {
      this.$emit('search', { keyword })
    }
  }
}
</script>
```

---

## 四、常见坑 / 注意事项

1. **事件名不要用驼峰监听**：子组件 `emit('updateUser')`，父组件写 `@update-user` 更稳妥；HTML 模板中 `@updateUser` 可能无法匹配。
2. **声明 `emits` 不是为了“允许”派发，而是为了类型和透传行为**：未声明的事件仍可派发，但会失去校验，并可能与原生事件混淆。
3. **不要用事件代替双向绑定**：需要双向绑定组件时，使用 `defineModel` 或 `update:modelValue` 约定，而不是随意起事件名。
4. **事件参数不要传整个响应式对象后让父组件随意修改**：这会让数据流变得隐蔽，最好传 id 或不可变快照，由父组件按 id 查找并修改。
5. **`emit` 是同步调用**：父组件的监听器会立即执行。如果父组件监听器是异步函数，子组件不会等待它完成，需要额外的完成事件或状态。
6. **原生事件与组件事件同名会冲突**：组件声明了 `click` 事件后，父组件的 `@click` 只会监听组件事件，不再自动落到根元素的原生 click 上。
7. **选项式 API 中 `emits` 要写在组件选项里**：`emits: ['save']`，否则 `$emit('save')` 仍能工作，但会被当作未声明事件，失去校验和文档价值。

---

## 五、官方文档

- 组件事件：https://cn.vuejs.org/guide/components/events.html
- 事件声明与校验：https://cn.vuejs.org/guide/components/events.html#declaring-emitted-events
- `defineEmits`：https://cn.vuejs.org/api/sfc-script-setup.html#defineprops-defineemits
- `v-model` 与组件事件：https://cn.vuejs.org/guide/components/v-model.html

---

## 六、速查表

| 语法                              | 作用        | 示例                                 |
| --------------------------------- | ----------- | ------------------------------------ |
| `defineEmits(['save'])`           | 声明事件    | `const emit = defineEmits(['save'])` |
| `emit('save', data)`              | 派发事件    | `emit('save', { id: 1 })`            |
| `@save="fn"`                      | 父组件监听  | `<Editor @save="handleSave" />`      |
| `defineEmits<{ save: [Data] }>()` | TS 事件类型 | `emit('save', data)`                 |
| `emits: ['save']`                 | 选项式声明  | `export default { emits: ['save'] }` |
