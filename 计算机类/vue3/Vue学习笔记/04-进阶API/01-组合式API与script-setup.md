# 组合式 API 与 script setup

> 组合式 API 用函数和组合式函数组织逻辑，`<script setup>` 是它在单文件组件中的语法糖，让同一业务关注点的代码集中在一起。

## 一、核心概念

- **`setup` 是入口**：组合式 API 的代码在 `setup` 中执行，此时组件实例尚未完全创建，不能访问 `this`，但可以使用响应式 API、生命周期钩子和依赖注入。
- **`<script setup>`**：编译时会被转换成 `setup()` 函数。顶层声明的变量、函数、导入的组件自动暴露给模板，不需要 `return`。
- **编译宏**：`defineProps`、`defineEmits`、`defineExpose`、`defineOptions`、`defineSlots`、`defineModel` 只在 `<script setup>` 中可用，不需要导入。
- **组合式函数（Composable）**：以 `use` 开头的函数，内部使用响应式 API 封装可复用逻辑，返回 ref、方法或状态。它是组合式 API 复用的主要方式。
- **逻辑聚合**：与选项式 API 按“data / methods / computed”分块不同，组合式 API 可以按“搜索 / 分页 / 权限”等业务关注点聚合代码。
- **与选项式共存**：一个组件可以同时有普通 `<script>` 和 `<script setup>`，普通 `<script>` 用于声明 `name`、`inheritAttrs` 等无法在语法糖中直接写的选项。

---

## 二、基础语法

```vue
<script setup>
import { computed, ref } from 'vue'

// 顶层绑定自动暴露给模板
const keyword = ref('')
const list = ref([1, 2, 3])

const filtered = computed(() => {
  return list.value.filter((item) => String(item).includes(keyword.value))
})

function reset() {
  keyword.value = ''
}

// 编译宏无需 import
defineOptions({ name: 'SearchPanel' })
</script>

<template>
  <input v-model="keyword" />
  <p v-for="item in filtered" :key="item">{{ item }}</p>
  <button @click="reset">重置</button>
</template>
```

普通 `setup()` 写法（了解即可）：

```vue
<script>
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function increment() {
      count.value++
    }

    // 必须显式返回，模板才能访问
    return { count, increment }
  }
}
</script>
```

一个组合式函数示例：

```js
// composables/useDisclosure.js
import { ref } from 'vue'

export function useDisclosure(initial = false) {
  const isOpen = ref(initial)

  function open() {
    isOpen.value = true
  }

  function close() {
    isOpen.value = false
  }

  function toggle() {
    isOpen.value = !isOpen.value
  }

  return { isOpen, open, close, toggle }
}
```

```vue
<script setup>
import { useDisclosure } from '@/composables/useDisclosure'

const { isOpen, open, close, toggle } = useDisclosure()
</script>

<template>
  <button @click="toggle">切换详情</button>
  <aside v-if="isOpen">
    <button @click="close">关闭</button>
  </aside>
</template>
```

---

## 三、完整示例

下面是一个带搜索、分页和加载状态的文章列表组件，把分页逻辑抽成 `usePagination` 组合式函数：

```vue
<script setup>
import { computed, onMounted, ref, watch } from 'vue'

// 组合式函数：封装分页状态和页码计算
function usePagination(total, pageSize = 5) {
  const page = ref(1)
  const totalPages = computed(() => Math.max(1, Math.ceil(total.value / pageSize)))

  const offset = computed(() => (page.value - 1) * pageSize)

  function next() {
    if (page.value < totalPages.value) page.value += 1
  }

  function prev() {
    if (page.value > 1) page.value -= 1
  }

  function reset() {
    page.value = 1
  }

  return { page, totalPages, offset, next, prev, reset }
}

const keyword = ref('')
const loading = ref(false)
const errorMessage = ref('')

const articles = ref([
  { id: 1, title: 'Vue 3 组合式 API 实践', author: '林晓', category: '前端' },
  { id: 2, title: 'Pinia 状态管理入门', author: '陈默', category: '前端' },
  { id: 3, title: '从零搭建 Vite 项目', author: '周宁', category: '工程化' },
  { id: 4, title: '理解浏览器渲染流程', author: '林晓', category: '浏览器' },
  { id: 5, title: 'TypeScript 类型体操', author: '苏晚', category: 'TypeScript' },
  { id: 6, title: '前端性能优化清单', author: '陈默', category: '性能' },
  { id: 7, title: 'Vue Router 权限控制', author: '周宁', category: '前端' }
])

const filteredArticles = computed(() => {
  const text = keyword.value.trim().toLowerCase()
  if (!text) return articles.value

  return articles.value.filter((article) => {
    return article.title.toLowerCase().includes(text)
      || article.author.toLowerCase().includes(text)
  })
})

const total = computed(() => filteredArticles.value.length)

const { page, totalPages, offset, next, prev, reset } = usePagination(total, 5)

const pageArticles = computed(() => {
  return filteredArticles.value.slice(offset.value, offset.value + 5)
})

// 搜索条件变化时回到第一页
watch(keyword, () => {
  reset()
})

async function refresh() {
  loading.value = true
  errorMessage.value = ''

  try {
    // 模拟接口刷新
    await new Promise((resolve) => setTimeout(resolve, 500))
  } catch (error) {
    errorMessage.value = '刷新失败，请重试'
  } finally {
    loading.value = false
  }
}

onMounted(refresh)
</script>

<template>
  <section class="articles">
    <header>
      <h1>技术文章</h1>
      <input v-model.trim="keyword" type="search" placeholder="搜索标题或作者" />
    </header>

    <p v-if="loading" class="state">加载中...</p>
    <p v-else-if="errorMessage" class="error">{{ errorMessage }}</p>

    <template v-else>
      <ul>
        <li v-for="article in pageArticles" :key="article.id">
          <div>
            <h2>{{ article.title }}</h2>
            <p>{{ article.author }} · {{ article.category }}</p>
          </div>
          <span>#{{ article.id }}</span>
        </li>
      </ul>

      <p v-if="pageArticles.length === 0" class="state">没有匹配的文章</p>

      <footer>
        <button type="button" :disabled="page <= 1" @click="prev">上一页</button>
        <span>{{ page }} / {{ totalPages }}</span>
        <button type="button" :disabled="page >= totalPages" @click="next">
          下一页
        </button>
        <button type="button" @click="refresh">刷新</button>
      </footer>
    </template>
  </section>
</template>

<style scoped>
.articles {
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
  width: 240px;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

ul {
  margin: 20px 0 0;
  padding: 0;
  list-style: none;
}

li {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 14px 0;
  border-bottom: 1px solid #f1f5f9;
}

li h2 {
  margin: 0 0 6px;
  font-size: 16px;
}

li p {
  margin: 0;
  color: #6b7280;
  font-size: 13px;
}

li span {
  color: #9ca3af;
  font-size: 13px;
}

footer {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
  margin-top: 20px;
}

footer button {
  padding: 7px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

footer button:disabled {
  cursor: not-allowed;
  opacity: 0.4;
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

1. **`<script setup>` 中不能使用 `this`**：组件实例尚未创建，所有状态通过 ref/reactive 访问。
2. **顶层 `await` 会让组件变成异步组件**：必须配合 `<Suspense>` 使用，否则组件不会渲染，且 `onMounted` 等钩子的注册时机要特别注意。
3. **`defineProps` 等宏不需要 import**：从 `vue` 导入 `defineProps` 会报错；它们由编译器处理。
4. **组合式函数必须在 `setup` 同步阶段调用**：在 `setTimeout`、事件回调或 `await` 之后调用，可能无法正确绑定生命周期钩子和注入上下文。
5. **组合式函数返回的 ref 解构后仍然是 ref**：`const { isOpen } = useDisclosure()` 中的 `isOpen` 仍是 ref，模板会自动解包；但如果组合式函数返回的是普通对象，解构就会失去响应性。
6. **普通 `<script>` 与 `<script setup>` 的变量不共享**：两者作用域独立，普通脚本中的变量不会自动暴露给模板。
7. **`defineOptions` 只能声明组件选项**：如 `name`、`inheritAttrs`，不要试图在里面写 `data` 或 `methods`。
8. **逻辑聚合不等于无限堆叠**：一个组件里组合式函数过多时，应继续拆分子组件或按业务模块拆分 composable。

---

## 五、官方文档

- 组合式 API FAQ：https://cn.vuejs.org/guide/extras/composition-api-faq.html
- `<script setup>`：https://cn.vuejs.org/api/sfc-script-setup.html
- 组合式函数：https://cn.vuejs.org/guide/reusability/composables.html
- Setup 上下文：https://cn.vuejs.org/api/composition-api-setup.html
- 选项式 API 与组合式 API 对比：https://cn.vuejs.org/guide/extras/composition-api-faq.html#trade-offs

---

## 六、速查表

| 语法             | 作用              | 示例                                           |
| ---------------- | ----------------- | ---------------------------------------------- |
| `<script setup>` | 组合式 API 语法糖 | 顶层变量自动暴露                               |
| `defineOptions`  | 声明组件选项      | `defineOptions({ name: 'Panel' })`             |
| `defineProps`    | 声明 props        | `const props = defineProps({ title: String })` |
| `defineEmits`    | 声明事件          | `const emit = defineEmits(['save'])`           |
| `defineExpose`   | 暴露给父组件      | `defineExpose({ validate })`                   |
