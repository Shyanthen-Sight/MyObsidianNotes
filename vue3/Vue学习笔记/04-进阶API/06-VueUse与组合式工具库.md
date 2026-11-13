# VueUse 与组合式工具库

> VueUse 是一组基于组合式 API 的常用工具函数集合，把鼠标位置、本地存储、防抖、元素可见性等浏览器能力封装成可复用的 `useXxx`。

## 一、核心概念

- **组合式函数集合**：VueUse 提供数百个函数，覆盖浏览器 API、状态、动画、传感器、网络和组件。按需引入，未使用的函数会被 tree-shaking 移除。
- **返回 ref**：大多数函数返回 `ref` 或 `computed`，在模板中自动解包，在脚本中通过 `.value` 读写。
- **自动清理**：`useEventListener`、`useIntersectionObserver`、`useResizeObserver` 等会在组件卸载时自动移除监听，减少手写清理代码。
- **SSR 意识**：涉及 `window`、`document` 的函数在服务端渲染时需要判断环境，VueUse 多数函数已做兼容，但仍要避免在 `setup` 顶层直接访问浏览器对象。
- **与自写 composable 的关系**：简单逻辑可以自己写 `useXxx`；复杂浏览器能力优先使用 VueUse，避免重复处理兼容性和边界情况。
- **常用分类**：状态（`useLocalStorage`、`useToggle`）、浏览器（`useEventListener`、`useIntersectionObserver`）、工具（`useDebounceFn`、`useThrottleFn`）、传感器（`useMouse`、`useDeviceOrientation`）。

---

## 二、基础语法

安装：

```bash
npm install @vueuse/core
```

```js
import { ref } from 'vue'
import {
  useClipboard,
  useDebounceFn,
  useIntersectionObserver,
  useLocalStorage,
  useToggle
} from '@vueuse/core'

// 本地存储，自动序列化 JSON
const settings = useLocalStorage('settings', {
  theme: 'light',
  fontSize: 14
})

// 布尔开关
const [isOpen, toggleOpen] = useToggle(false)

// 防抖函数
const save = useDebounceFn(() => {
  console.log('保存', settings.value)
}, 500)

// 元素可见性
const target = ref(null)
const { stop } = useIntersectionObserver(target, ([entry]) => {
  if (entry.isIntersecting) {
    console.log('元素进入视口')
    stop()
  }
})

// 复制到剪贴板
const { copy, copied } = useClipboard()
await copy('https://example.com')
```

---

## 三、完整示例

下面是一个文章编辑页，使用 `useLocalStorage` 自动保存草稿、`useDebounceFn` 降低写入频率、`useClipboard` 复制分享链接、`useIntersectionObserver` 实现评论区懒加载：

```vue
<script setup>
import { computed, ref, watch } from 'vue'
import {
  useClipboard,
  useDebounceFn,
  useIntersectionObserver,
  useLocalStorage
} from '@vueuse/core'

const draft = useLocalStorage('article-draft', {
  title: '',
  content: ''
})

const savedAt = ref('')
const comments = ref([])
const loadingComments = ref(false)
const sentinel = ref(null)

const wordCount = computed(() => draft.value.content.trim().length)

const shareUrl = computed(() => {
  return `https://example.com/articles/draft?title=${encodeURIComponent(draft.value.title)}`
})

const { copy, copied } = useClipboard()

// 输入停止 600ms 后再记录保存时间，避免每次按键都写入
const markSaved = useDebounceFn(() => {
  savedAt.value = new Date().toLocaleTimeString('zh-CN')
}, 600)

watch(draft, markSaved, { deep: true })

async function loadComments() {
  if (loadingComments.value) return

  loadingComments.value = true
  try {
    await new Promise((resolve) => setTimeout(resolve, 500))
    comments.value = [
      { id: 1, author: '陈默', content: '示例很清晰，收藏了。' },
      { id: 2, author: '周宁', content: '请问支持 SSR 吗？' }
    ]
  } finally {
    loadingComments.value = false
  }
}

// 评论区占位元素进入视口时再请求，避免首屏加载无关数据
useIntersectionObserver(
  sentinel,
  ([entry]) => {
    if (entry.isIntersecting) loadComments()
  },
  { threshold: 0.2 }
)
</script>

<template>
  <section class="editor">
    <header>
      <input v-model="draft.title" type="text" placeholder="文章标题" />
      <div class="meta">
        <span>{{ wordCount }} 字</span>
        <span v-if="savedAt">已保存 {{ savedAt }}</span>
      </div>
    </header>

    <textarea
      v-model="draft.content"
      rows="10"
      placeholder="开始写作..."
    ></textarea>

    <div class="actions">
      <button type="button" :disabled="!draft.title" @click="copy(shareUrl)">
        {{ copied ? '已复制' : '复制分享链接' }}
      </button>
      <button type="button" @click="draft = { title: '', content: '' }">
        清空草稿
      </button>
    </div>

    <section class="comments">
      <h2>评论</h2>

      <p v-if="loadingComments" class="state">评论加载中...</p>

      <ul v-else-if="comments.length > 0">
        <li v-for="comment in comments" :key="comment.id">
          <strong>{{ comment.author }}</strong>
          <p>{{ comment.content }}</p>
        </li>
      </ul>

      <p v-else class="state">暂无评论</p>

      <div ref="sentinel" class="sentinel"></div>
    </section>
  </section>
</template>

<style scoped>
.editor {
  max-width: 680px;
  margin: 40px auto;
  padding: 24px;
  font-family: system-ui, sans-serif;
}

header input {
  width: 100%;
  box-sizing: border-box;
  padding: 10px 0;
  border: 0;
  border-bottom: 1px solid #e5e7eb;
  outline: none;
  font-size: 24px;
  font-weight: 700;
}

.meta {
  display: flex;
  gap: 14px;
  margin: 8px 0 16px;
  color: #9ca3af;
  font-size: 12px;
}

textarea {
  width: 100%;
  box-sizing: border-box;
  padding: 14px;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  outline: none;
  font: inherit;
  line-height: 1.7;
  resize: vertical;
}

.actions {
  display: flex;
  gap: 10px;
  margin: 14px 0 28px;
}

.actions button {
  padding: 8px 14px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  background: #ffffff;
  cursor: pointer;
}

.actions button:disabled {
  cursor: not-allowed;
  opacity: 0.4;
}

.comments h2 {
  font-size: 18px;
}

.comments ul {
  margin: 0;
  padding: 0;
  list-style: none;
}

.comments li {
  padding: 12px 0;
  border-bottom: 1px solid #f1f5f9;
}

.comments li p {
  margin: 6px 0 0;
  color: #4b5563;
  font-size: 14px;
}

.state {
  color: #9ca3af;
  font-size: 14px;
}

.sentinel {
  height: 1px;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **VueUse 函数返回 ref**：脚本中要写 `settings.value.theme`，模板中可以直接写 `settings.theme`。
2. **`useLocalStorage` 会尝试 JSON 序列化**：存储函数、DOM 节点、循环引用对象会失败；复杂数据应先转换为可序列化结构。
3. **`useIntersectionObserver` 的回调可能触发多次**：加载完成后应调用 `stop()` 或加 `loading` 守卫，避免重复请求。
4. **SSR 中不要直接使用浏览器对象**：`useMouse`、`useWindowSize` 等依赖 `window`，在服务端渲染时应放在客户端组件或 `onMounted` 后使用。
5. **VueUse 的 `useFetch` 与 Nuxt 的 `useFetch` 不是同一个函数**：前者是通用的响应式请求封装，后者是 Nuxt 内置的数据获取 API，导入路径和返回结构不同。
6. **按需导入，不要全量注册**：VueUse 函数很多，`import { useXxx } from '@vueuse/core'` 可以 tree-shaking，全量挂载会增加包体积。
7. **自动清理只对组合式函数生效**：在组件 `setup` 中调用 `useEventListener` 会自动清理；在普通函数或全局模块中调用则需要手动调用返回的 `stop`。
8. **不要用 VueUse 替代业务抽象**：它解决通用浏览器能力，业务规则仍应封装成自己的 composable 或 store。

---

## 五、官方文档

- VueUse 中文文档：https://vueuse.org/zh-CN/
- 按需安装与使用：https://vueuse.org/guide/
- `useLocalStorage`：https://vueuse.org/core/useLocalStorage/
- `useIntersectionObserver`：https://vueuse.org/core/useIntersectionObserver/
- `useDebounceFn`：https://vueuse.org/shared/useDebounceFn/
- Vue 组合式函数：https://cn.vuejs.org/guide/reusability/composables.html

---

## 六、速查表

| 语法                      | 作用           | 示例                                         |
| ------------------------- | -------------- | -------------------------------------------- |
| `useLocalStorage`         | 本地存储响应式 | `const draft = useLocalStorage('draft', '')` |
| `useToggle`               | 布尔切换       | `const [open, toggle] = useToggle(false)`    |
| `useDebounceFn`           | 防抖函数       | `const save = useDebounceFn(fn, 500)`        |
| `useThrottleFn`           | 节流函数       | `const onScroll = useThrottleFn(fn, 200)`    |
| `useIntersectionObserver` | 元素可见性     | `useIntersectionObserver(el, cb)`            |
