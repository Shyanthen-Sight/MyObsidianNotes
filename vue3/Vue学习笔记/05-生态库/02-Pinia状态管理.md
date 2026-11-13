# Pinia 状态管理

> Pinia 是 Vue 官方推荐的状态管理库，用多个独立的 store 管理跨组件共享状态，支持组合式写法、TypeScript 推导和开发工具调试。

## 一、核心概念

- **Store 是独立模块**：每个业务域一个 store，例如 `useUserStore`、`useCartStore`。store 通过 `defineStore(id, setup)` 定义。
- **Setup Store**：用 `ref`、`computed`、函数定义 state、getters、actions，写法与组件内的组合式 API 一致。
- **Options Store**：用 `state`、`getters`、`actions` 选项定义，更接近 Vuex 的习惯，但类型推导稍弱。
- **`storeToRefs`**：直接解构 store 会失去响应性；需要解构 state/getters 时使用 `storeToRefs(store)`，actions 可以直接解构。
- **修改状态**：可以直接赋值 `store.profile = user`，也可以调用 action；批量修改可用 `store.$patch()`。
- **插件与订阅**：`pinia.use()` 注册插件，`store.$subscribe()` 监听状态变化，`store.$onAction()` 监听 action 调用，适合持久化、埋点、日志。

---

## 二、基础语法

```js
// stores/user.js
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', () => {
  const profile = ref(null)

  const isLoggedIn = computed(() => Boolean(profile.value))

  function login(user) {
    profile.value = user
  }

  function logout() {
    profile.value = null
  }

  return { profile, isLoggedIn, login, logout }
})
```

选项式写法的 store：

```js
export const useSettingsStore = defineStore('settings', {
  state: () => ({
    theme: 'light',
    sidebarOpen: true
  }),
  getters: {
    isDark: (state) => state.theme === 'dark'
  },
  actions: {
    toggleTheme() {
      this.theme = this.theme === 'light' ? 'dark' : 'light'
    }
  }
})
```

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

createApp(App).use(createPinia()).mount('#app')
```

```vue
<script setup>
import { storeToRefs } from 'pinia'
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()

// state / getters 解构要经过 storeToRefs
const { profile, isLoggedIn } = storeToRefs(userStore)

// action 可以直接解构
const { login, logout } = userStore
</script>

<template>
  <p v-if="isLoggedIn">欢迎，{{ profile.name }}</p>
  <p v-else>请先登录</p>
  <button @click="login({ id: 1, name: '林晓' })">登录</button>
  <button @click="logout">退出</button>
</template>
```

---

## 三、完整示例

下面是一个购物车 store 和对应的购物车页面，包含商品增删、数量修改、优惠券、持久化和异步结算：

```js
// stores/cart.js
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'

export const useCartStore = defineStore('cart', () => {
  const items = ref([])
  const couponCode = ref('')
  const discount = ref(0)
  const submitting = ref(false)

  const count = computed(() => {
    return items.value.reduce((sum, item) => sum + item.quantity, 0)
  })

  const subtotal = computed(() => {
    return items.value.reduce((sum, item) => {
      return sum + item.price * item.quantity
    }, 0)
  })

  const payable = computed(() => {
    return Math.max(0, subtotal.value - discount.value)
  })

  function addItem(product, quantity = 1) {
    const existing = items.value.find((item) => item.id === product.id)

    if (existing) {
      existing.quantity += quantity
      return
    }

    items.value.push({
      id: product.id,
      name: product.name,
      price: product.price,
      quantity
    })
  }

  function updateQuantity(id, quantity) {
    const item = items.value.find((row) => row.id === id)
    if (!item) return

    const next = Math.max(1, Number(quantity) || 1)
    item.quantity = next
  }

  function removeItem(id) {
    items.value = items.value.filter((item) => item.id !== id)
  }

  function applyCoupon(code) {
    couponCode.value = code.trim().toUpperCase()

    if (couponCode.value === 'VUE20') {
      discount.value = 20
      return true
    }

    discount.value = 0
    return false
  }

  async function checkout() {
    if (items.value.length === 0 || submitting.value) return null

    submitting.value = true
    try {
      // 模拟下单接口，真实项目替换为 fetch/axios
      await new Promise((resolve) => setTimeout(resolve, 800))

      const order = {
        id: `SO${Date.now()}`,
        total: payable.value,
        items: items.value.map((item) => ({ ...item }))
      }

      items.value = []
      discount.value = 0
      couponCode.value = ''

      return order
    } finally {
      submitting.value = false
    }
  }

  return {
    items,
    couponCode,
    discount,
    submitting,
    count,
    subtotal,
    payable,
    addItem,
    updateQuantity,
    removeItem,
    applyCoupon,
    checkout
  }
})
```

```vue
<!-- views/CartView.vue -->
<script setup>
import { ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useCartStore } from '@/stores/cart'

const cart = useCartStore()
const { items, count, subtotal, payable, submitting } = storeToRefs(cart)

const couponInput = ref('')
const message = ref('')
const lastOrderId = ref('')

function handleCoupon() {
  const ok = cart.applyCoupon(couponInput.value)
  message.value = ok ? '优惠券已生效' : '优惠券无效'
}

async function handleCheckout() {
  const order = await cart.checkout()
  if (order) {
    lastOrderId.value = order.id
    message.value = '下单成功'
  }
}
</script>

<template>
  <section class="cart-page">
    <header>
      <h1>购物车</h1>
      <span>共 {{ count }} 件</span>
    </header>

    <p v-if="items.length === 0" class="empty">购物车是空的</p>

    <ul v-else>
      <li v-for="item in items" :key="item.id">
        <div>
          <strong>{{ item.name }}</strong>
          <span>¥{{ item.price }}</span>
        </div>

        <input
          type="number"
          min="1"
          :value="item.quantity"
          @change="cart.updateQuantity(item.id, $event.target.value)"
        />

        <button type="button" @click="cart.removeItem(item.id)">删除</button>
      </li>
    </ul>

    <div class="coupon">
      <input v-model="couponInput" placeholder="输入优惠码 VUE20" />
      <button type="button" @click="handleCoupon">使用</button>
    </div>

    <p v-if="message" class="message">{{ message }}</p>

    <footer>
      <p>小计：¥{{ subtotal }}</p>
      <p class="total">应付：<strong>¥{{ payable }}</strong></p>
      <button
        type="button"
        :disabled="items.length === 0 || submitting"
        @click="handleCheckout"
      >
        {{ submitting ? '提交中...' : '去结算' }}
      </button>
    </footer>
  </section>
</template>

<style scoped>
.cart-page {
  max-width: 640px;
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

header span {
  color: #6b7280;
  font-size: 13px;
}

ul {
  margin: 18px 0;
  padding: 0;
  list-style: none;
}

li {
  display: grid;
  grid-template-columns: 1fr 80px auto;
  gap: 12px;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #f1f5f9;
}

li div {
  display: grid;
  gap: 4px;
}

li span {
  color: #6b7280;
  font-size: 13px;
}

li input {
  width: 64px;
  padding: 6px 8px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
}

li button {
  border: 0;
  background: transparent;
  color: #dc2626;
  cursor: pointer;
  font-size: 13px;
}

.empty {
  padding: 40px;
  color: #9ca3af;
  text-align: center;
}

.coupon {
  display: flex;
  gap: 8px;
  margin-top: 16px;
}

.coupon input {
  flex: 1;
  padding: 9px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

.coupon button {
  padding: 9px 16px;
  border: 0;
  border-radius: 8px;
  background: #111827;
  color: #ffffff;
  cursor: pointer;
}

.message {
  color: #059669;
  font-size: 13px;
}

footer {
  margin-top: 20px;
  padding-top: 16px;
  border-top: 1px solid #e5e7eb;
  text-align: right;
}

footer p {
  margin: 6px 0;
}

.total strong {
  color: #dc2626;
  font-size: 24px;
}

footer > button {
  margin-top: 12px;
  padding: 11px 28px;
  border: 0;
  border-radius: 8px;
  background: #111827;
  color: #ffffff;
  cursor: pointer;
}

footer > button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}
</style>
```

持久化插件示例：

```js
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const pinia = createPinia()

pinia.use(({ store }) => {
  const saved = localStorage.getItem(`pinia:${store.$id}`)
  if (saved) store.$patch(JSON.parse(saved))

  store.$subscribe((mutation, state) => {
    localStorage.setItem(`pinia:${store.$id}`, JSON.stringify(state))
  })
})

createApp(App).use(pinia).mount('#app')
```

---

## 四、常见坑 / 注意事项

1. **直接解构 store 会失去响应性**：`const { count } = cart` 得到普通值；应使用 `storeToRefs(cart)`，action 除外。
2. **`useStore()` 必须在 `app.use(createPinia())` 之后调用**：在模块顶层直接调用 store 可能拿不到 active pinia，应放在 `setup` 或函数内部。
3. **Options Store 的 `state` 必须是函数**：`state: () => ({ count: 0 })`，写成对象会导致多个 store 实例共享同一份状态。
4. **`$reset()` 只适用于 Options Store**：Setup Store 需要自己实现 reset 方法，逐个重置 ref。
5. **异步 action 不会自动处理并发**：快速点击可能发起多次请求，应在 action 内用 `submitting` 或请求取消机制控制。
6. **SSR 中每个请求要创建新的 Pinia**：不能复用服务端进程级 store，否则不同用户的状态会串；需要序列化 `pinia.state.value` 注入客户端。
7. **持久化不要保存敏感信息**：token、手机号等不要直接写入 `localStorage`，并注意存储容量和版本迁移。
8. **store 不是替代组件状态的万能方案**：只有跨组件、跨页面共享的状态才放进 store，局部 UI 状态留在组件内。

---

## 五、官方文档

- Pinia 中文文档：https://pinia.vuejs.org/zh/
- 开始使用：https://pinia.vuejs.org/zh/getting-started.html
- Setup Store：https://pinia.vuejs.org/zh/core-concepts/#setup-stores
- `storeToRefs`：https://pinia.vuejs.org/zh/api/modules/pinia.html#storetorefs
- 插件：https://pinia.vuejs.org/zh/core-concepts/plugins.html
- Vue 官方状态管理：https://cn.vuejs.org/guide/scaling-up/state-management.html

---

## 六、速查表

| 语法                     | 作用             | 示例                                  |
| ------------------------ | ---------------- | ------------------------------------- |
| `createPinia()`          | 创建 Pinia 实例  | `app.use(createPinia())`              |
| `defineStore(id, setup)` | 定义 Setup Store | `defineStore('cart', () => { ... })`  |
| `storeToRefs(store)`     | 解构保持响应性   | `const { count } = storeToRefs(cart)` |
| `store.$patch(obj)`      | 批量修改状态     | `store.$patch({ count: 1 })`          |
| `store.$subscribe`       | 订阅状态变化     | `store.$subscribe((m, s) => {})`      |
