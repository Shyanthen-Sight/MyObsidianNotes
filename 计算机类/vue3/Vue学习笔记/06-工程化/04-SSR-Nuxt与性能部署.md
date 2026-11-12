# SSR、Nuxt 与性能部署

> 服务端渲染（SSR）让首屏 HTML 在服务器生成，改善首屏速度和 SEO；Nuxt 是 Vue 的全栈框架，把路由、数据获取、SSR/SSG 和部署整合在一起。

## 一、核心概念

- **SSR 的价值**：服务器直接返回渲染好的 HTML，用户更快看到内容，搜索引擎也更容易抓取；代价是服务器压力、构建复杂度和水合（hydration）问题。
- **水合**：浏览器拿到 SSR HTML 后，Vue 在客户端复用这份 DOM 并绑定事件。服务端与客户端首次渲染结果必须一致，否则会出现 hydration mismatch。
- **Nuxt**：基于 Vue 的元框架，提供文件路由、自动导入、`useAsyncData` / `useFetch`、SSR/SSG/SPA 模式切换和 Nitro 部署能力。
- **SSG**：构建时预渲染静态 HTML，适合博客、文档、营销页；内容更新需要重新构建，动态内容可配合增量静态再生成。
- **VitePress**：基于 Vite 和 Vue 的静态站点生成器，用 Markdown 写文档，构建时输出预渲染 HTML，适合组件库文档、产品手册和博客。
- **跨端打包**：Capacitor 可把 Vue 应用包装成 iOS/Android 应用，Electron/Tauri 可用于桌面端；它们复用同一套前端代码，但需要处理原生权限和平台差异。
- **性能优化**：减少首屏 JS、延迟加载非关键组件、避免大对象深层响应式、使用 `v-memo`、虚拟列表和缓存。
- **生产部署**：静态站点部署到 CDN/Nginx；SSR 需要 Node 服务或边缘运行时；无论哪种方式都要配置正确的 `base`、缓存头和环境变量。

---

## 二、基础语法

最小 SSR 服务：

```js
import { createSSRApp } from 'vue'
import { renderToString } from 'vue/server-renderer'
import App from './App.vue'

export async function render(url) {
  const app = createSSRApp(App)
  const html = await renderToString(app)
  return html
}
```

客户端水合：

```js
import { createSSRApp } from 'vue'
import App from './App.vue'

// 复用服务端返回的 HTML，而不是重新创建 DOM
createSSRApp(App).mount('#app')
```

Nuxt 页面数据获取：

```vue
<script setup>
// useFetch 会在服务端和客户端正确处理请求
const { data: orders, pending, error, refresh } = await useFetch('/api/orders')
</script>

<template>
  <p v-if="pending">加载中...</p>
  <p v-else-if="error">加载失败</p>
  <ul v-else>
    <li v-for="order in orders" :key="order.id">{{ order.no }}</li>
  </ul>
</template>
```

---

## 三、完整示例

下面是一个性能友好的商品列表组件，使用 `shallowRef` 避免大对象深层代理、异步加载图表、`v-memo` 跳过无关行更新，并在挂载后记录性能标记：

```vue
<script setup>
import { computed, defineAsyncComponent, onMounted, shallowRef, ref } from 'vue'

// 图表库体积大，拆分为异步组件
const SalesChart = defineAsyncComponent(() => import('./SalesChart.vue'))

const products = shallowRef([
  { id: 1, name: '轻薄笔记本', price: 6299, sales: 3200, category: '电脑' },
  { id: 2, name: '电竞显示器', price: 1899, sales: 5100, category: '显示' },
  { id: 3, name: '无线鼠标', price: 199, sales: 12000, category: '配件' },
  { id: 4, name: '机械键盘', price: 499, sales: 8600, category: '配件' }
])

const category = ref('all')
const showChart = ref(false)

const categories = computed(() => {
  return ['all', ...new Set(products.value.map((item) => item.category))]
})

const visibleProducts = computed(() => {
  if (category.value === 'all') return products.value
  return products.value.filter((item) => item.category === category.value)
})

const totalSales = computed(() => {
  return visibleProducts.value.reduce((sum, item) => sum + item.sales, 0)
})

// 整体替换 shallowRef 才会触发更新
function updateSales(id, sales) {
  products.value = products.value.map((item) => {
    return item.id === id ? { ...item, sales } : item
  })
}

onMounted(() => {
  if ('performance' in window) {
    performance.mark('product-list-mounted')
  }
})
</script>

<template>
  <section class="products">
    <header>
      <h1>商品销售概览</h1>
      <select v-model="category">
        <option v-for="item in categories" :key="item" :value="item">
          {{ item === 'all' ? '全部分类' : item }}
        </option>
      </select>
    </header>

    <p class="total">总销量：{{ totalSales }}</p>

    <ul>
      <li
        v-for="product in visibleProducts"
        :key="product.id"
        v-memo="[product.sales]"
      >
        <div>
          <strong>{{ product.name }}</strong>
          <span>{{ product.category }} · ¥{{ product.price }}</span>
        </div>

        <div class="sales">
          <span>销量 {{ product.sales }}</span>
          <button type="button" @click="updateSales(product.id, product.sales + 1)">
            +1
          </button>
        </div>
      </li>
    </ul>

    <button type="button" class="chart-toggle" @click="showChart = !showChart">
      {{ showChart ? '收起图表' : '查看销售趋势' }}
    </button>

    <Suspense v-if="showChart">
      <template #default>
        <SalesChart :products="visibleProducts" />
      </template>
      <template #fallback>
        <div class="chart-fallback">图表加载中...</div>
      </template>
    </Suspense>
  </section>
</template>

<style scoped>
.products {
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

select {
  padding: 8px 10px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.total {
  margin: 14px 0;
  color: #6b7280;
  font-size: 14px;
}

ul {
  margin: 0;
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

li div:first-child {
  display: grid;
  gap: 4px;
}

li span {
  color: #6b7280;
  font-size: 13px;
}

.sales {
  display: flex;
  align-items: center;
  gap: 10px;
}

.sales button {
  width: 28px;
  height: 28px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
}

.chart-toggle {
  margin-top: 18px;
  padding: 9px 16px;
  border: 0;
  border-radius: 8px;
  background: #111827;
  color: #ffffff;
  cursor: pointer;
}

.chart-fallback {
  display: grid;
  height: 220px;
  margin-top: 16px;
  place-items: center;
  border-radius: 12px;
  background: #f9fafb;
  color: #9ca3af;
}
</style>
```

Nuxt 部署配置示例：

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true,
  nitro: {
    preset: 'node-server'
  },
  runtimeConfig: {
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE
    }
  }
})
```

---

## 四、常见坑 / 注意事项

1. **服务端不能访问 `window` / `document`**：在 `setup` 顶层直接使用会报错；应放到 `onMounted`、`import.meta.client` 或 `<ClientOnly>` 中。
2. **水合不匹配会触发警告并重新渲染**：服务端与客户端不能使用 `Math.random()`、`Date.now()` 等不确定值生成首屏内容。
3. **SSR 中每个请求要有独立状态**：不要把用户数据放在模块顶层或全局单例，Pinia 需要每请求创建实例并序列化状态。
4. **`shallowRef` 的嵌套修改不会更新**：必须整体替换 `.value`，或手动 `triggerRef`；这既是性能手段也是常见 bug 来源。
5. **`v-memo` 依赖数组漏字段会导致界面不更新**：所有影响该行渲染的值都必须写进数组。
6. **异步组件会增加请求瀑布**：首屏关键组件不要异步化，非关键图表、编辑器可以延迟加载。
7. **生产环境不要开启 source map 对外暴露**：需要排错时上传到监控平台，而不是直接部署到公开目录。
8. **部署子路径要同时配置 `base` 和路由**：Vite 的 `base`、Vue Router 的 `createWebHistory(base)`、Nginx 的 `try_files` 要一致。

---

## 五、官方文档

- 服务端渲染：https://cn.vuejs.org/guide/scaling-up/ssr.html
- Nuxt 中文文档：https://nuxt.com.cn/
- VitePress 中文文档：https://vitepress.dev/zh/
- Capacitor 文档：https://capacitorjs.com/docs
- 性能优化：https://cn.vuejs.org/guide/best-practices/performance.html
- 生产部署：https://cn.vuejs.org/guide/best-practices/production-deployment.html
- Vite 构建生产版本：https://cn.vitejs.dev/guide/build.html

---

## 六、速查表

| 语法             | 作用                  | 示例                                                        |
| ---------------- | --------------------- | ----------------------------------------------------------- |
| `createSSRApp`   | 创建可水合的 SSR 应用 | `createSSRApp(App).mount('#app')`                           |
| `renderToString` | 服务端渲染 HTML       | `await renderToString(app)`                                 |
| `useFetch`       | Nuxt 数据获取         | `const { data } = await useFetch('/api/list')`              |
| `useAsyncData`   | Nuxt 异步数据         | `await useAsyncData('orders', () => $fetch('/api/orders'))` |
| `shallowRef`     | 浅层响应式            | `const list = shallowRef(bigData)`                          |
