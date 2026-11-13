# Vue Router 路由

> Vue Router 是 Vue 官方路由库，负责把 URL 映射到组件，并在不刷新页面的情况下完成页面切换、参数传递和访问控制。

## 一、核心概念

- **路由模式**：`createWebHistory()` 使用 HTML5 History，URL 干净但需要服务器配置 fallback；`createWebHashHistory()` 使用 `#`，部署简单但 URL 不美观。
- **路由表**：每个路由包含 `path`、`name`、`component`，可以嵌套 `children`，通过 `redirect`、`alias` 处理跳转和别名。
- **`RouterView`**：渲染当前匹配的组件；嵌套路由中，子路由的 `RouterView` 写在父组件模板里。
- **`RouterLink`**：生成不会刷新页面的链接，自动处理激活样式；`to` 可以是字符串或包含 `name`、`params`、`query` 的对象。
- **导航守卫**：`beforeEach`、`beforeResolve`、`afterEach` 全局守卫，以及组件内的 `onBeforeRouteLeave`、`onBeforeRouteUpdate`，用于登录校验、权限控制、离开确认。
- **组合式 API**：`useRouter()` 获取路由器实例用于跳转，`useRoute()` 获取当前路由信息（只读响应式对象）。

---

## 二、基础语法

```js
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '../views/HomeView.vue'

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      name: 'home',
      component: HomeView
    },
    {
      path: '/orders/:id',
      name: 'order-detail',
      component: () => import('../views/OrderDetail.vue'),
      props: true
    },
    {
      path: '/admin',
      component: () => import('../views/AdminLayout.vue'),
      meta: { requiresAuth: true },
      children: [
        {
          path: 'users',
          name: 'admin-users',
          component: () => import('../views/AdminUsers.vue')
        }
      ]
    }
  ]
})

export default router
```

```js
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App).use(router).mount('#app')
```

```vue
<script setup>
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
  <nav>
    <RouterLink to="/">首页</RouterLink>
    <RouterLink :to="{ name: 'order-detail', params: { id: 1 } }">
      订单详情
    </RouterLink>
  </nav>

  <RouterView />
</template>
```

---

## 三、完整示例

下面是一个带登录校验和面包屑的订单管理页面，展示路由参数、查询参数、编程式导航和全局守卫：

```js
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

const routes = [
  {
    path: '/login',
    name: 'login',
    component: () => import('../views/LoginView.vue')
  },
  {
    path: '/orders',
    name: 'orders',
    component: () => import('../views/OrderListView.vue'),
    meta: { requiresAuth: true, title: '订单列表' }
  },
  {
    path: '/orders/:id',
    name: 'order-detail',
    component: () => import('../views/OrderDetailView.vue'),
    props: true,
    meta: { requiresAuth: true, title: '订单详情' }
  }
]

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) return savedPosition
    return { top: 0 }
  }
})

router.beforeEach((to) => {
  const token = localStorage.getItem('token')

  if (to.meta.requiresAuth && !token) {
    return {
      name: 'login',
      query: { redirect: to.fullPath }
    }
  }

  return true
})

router.afterEach((to) => {
  document.title = to.meta.title ? `${to.meta.title} - 管理后台` : '管理后台'
})

export default router
```

```vue
<!-- views/OrderListView.vue -->
<script setup>
import { computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const route = useRoute()
const router = useRouter()

const orders = ref([
  { id: 1, no: 'SO2026001', customer: '陈默', status: 'paid' },
  { id: 2, no: 'SO2026002', customer: '周宁', status: 'shipped' },
  { id: 3, no: 'SO2026003', customer: '苏晚', status: 'done' }
])

const activeStatus = computed(() => route.query.status ?? 'all')

const visibleOrders = computed(() => {
  if (activeStatus.value === 'all') return orders.value
  return orders.value.filter((order) => order.status === activeStatus.value)
})

function changeStatus(status) {
  // 修改 query 会触发路由更新，但不会刷新页面
  router.push({ query: { ...route.query, status } })
}

function openDetail(order) {
  router.push({
    name: 'order-detail',
    params: { id: order.id }
  })
}

// 监听路由参数变化，适合在详情页重新请求数据
watch(
  () => route.params.id,
  (id) => {
    if (id) console.log('加载订单', id)
  }
)
</script>

<template>
  <section class="orders">
    <header>
      <h1>订单列表</h1>
      <p>当前筛选：{{ activeStatus }}</p>
    </header>

    <div class="filters">
      <button
        v-for="item in [
          { value: 'all', label: '全部' },
          { value: 'paid', label: '已支付' },
          { value: 'shipped', label: '已发货' },
          { value: 'done', label: '已完成' }
        ]"
        :key="item.value"
        type="button"
        :class="{ active: activeStatus === item.value }"
        @click="changeStatus(item.value)"
      >
        {{ item.label }}
      </button>
    </div>

    <ul>
      <li v-for="order in visibleOrders" :key="order.id">
        <div>
          <strong>{{ order.no }}</strong>
          <span>{{ order.customer }}</span>
        </div>
        <button type="button" @click="openDetail(order)">查看详情</button>
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
  align-items: baseline;
  justify-content: space-between;
}

h1 {
  margin: 0;
  font-size: 22px;
}

header p {
  margin: 0;
  color: #6b7280;
  font-size: 13px;
}

.filters {
  display: flex;
  gap: 8px;
  margin: 18px 0;
}

.filters button {
  padding: 7px 14px;
  border: 1px solid #d1d5db;
  border-radius: 999px;
  background: #ffffff;
  cursor: pointer;
}

.filters button.active {
  border-color: #111827;
  background: #111827;
  color: #ffffff;
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

li div {
  display: flex;
  gap: 12px;
}

li strong {
  min-width: 110px;
}

li span {
  color: #6b7280;
}

li button {
  padding: 6px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **History 模式需要服务器 fallback**：直接访问 `/orders/1` 时，服务器要返回 `index.html`，否则 404；Nginx 可用 `try_files $uri $uri/ /index.html`。
2. **`route.params` 是字符串**：路由参数 `id` 拿到的是 `"1"`，与数字比较时要转换类型。
3. **守卫必须返回结果**：`beforeEach` 返回 `false` 取消导航，返回路由地址重定向，返回 `true` 或 `undefined` 放行；忘记返回可能导致导航挂起。
4. **避免无限重定向**：登录守卫重定向到 `/login` 时，要排除 `/login` 自身，否则会不断跳转。
5. **`useRoute` 与 `useRouter` 不要混用**：`route` 是当前路由信息（只读），`router` 是路由器实例（提供 `push`、`replace`、`back`）。
6. **组件复用不会重新创建**：`/orders/1` 跳到 `/orders/2` 时组件实例被复用，`onMounted` 不会再次执行，应监听 `route.params` 或使用 `onBeforeRouteUpdate`。
7. **`props: true` 才能把参数作为 props 传入**：否则子组件只能通过 `useRoute()` 读取参数，耦合度更高。
8. **路由懒加载的 import 路径要静态可分析**：`() => import('../views/' + name + '.vue')` 无法被 Vite 正确打包，应使用完整静态路径或 `import.meta.glob`。

---

## 五、官方文档

- Vue Router 中文文档：https://router.vuejs.org/zh/
- 路由起步：https://router.vuejs.org/zh/guide/
- 动态路由匹配：https://router.vuejs.org/zh/guide/essentials/dynamic-matching.html
- 导航守卫：https://router.vuejs.org/zh/guide/advanced/navigation-guards.html
- Vue 官方路由章节：https://cn.vuejs.org/guide/scaling-up/routing.html

---

## 六、速查表

| 语法               | 作用         | 示例                                         |
| ------------------ | ------------ | -------------------------------------------- |
| `createRouter`     | 创建路由器   | `createRouter({ history, routes })`          |
| `createWebHistory` | History 模式 | `createWebHistory()`                         |
| `<RouterView />`   | 渲染路由组件 | `<RouterView />`                             |
| `<RouterLink to>`  | 声明式导航   | `<RouterLink to="/orders">订单</RouterLink>` |
| `router.push`      | 编程式导航   | `router.push({ name: 'home' })`              |
