# 依赖注入 provide / inject

> 当祖先组件需要向深层后代传递数据，而中间层并不关心这些数据时，`provide` / `inject` 可以避免逐层传递 props 的“prop drilling”。

## 一、核心概念

- **`provide`**：祖先组件调用 `provide(key, value)` 向其后代提供数据。`key` 可以是字符串或 Symbol，Symbol 能避免命名冲突。
- **`inject`**：后代组件调用 `inject(key, defaultValue)` 获取最近一层祖先提供的数据。
- **响应式传递**：如果 provide 的是 `ref` 或 `reactive` 对象，后代拿到的是同一个引用，修改会同步；如果 provide 的是普通值，后代不会自动更新。
- **只读保护**：把 `readonly(ref)` 或 `readonly(reactiveObj)` 提供给后代，可以防止后代直接修改状态。
- **默认值**：`inject(key, defaultValue)` 在找不到时返回默认值；默认值本身是函数时，需要第三个参数 `false` 才能把函数作为值而不是工厂函数。
- **应用级注入**：`app.provide(key, value)` 对整个应用生效，适合主题、配置、国际化等全局能力。

---

## 二、基础语法

```vue
<!-- 祖先组件 -->
<script setup>
import { provide, readonly, ref } from 'vue'

const theme = ref('light')
const toggleTheme = () => {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

// 提供响应式数据和方法
provide('theme', readonly(theme))
provide('toggleTheme', toggleTheme)
</script>
```

```vue
<!-- 后代组件 -->
<script setup>
import { inject } from 'vue'

const theme = inject('theme', 'light')
const toggleTheme = inject('toggleTheme', () => {})
</script>

<template>
  <button @click="toggleTheme">当前主题：{{ theme }}</button>
</template>
```

使用 Symbol 作为 key，避免字符串冲突：

```js
// keys.js
export const themeKey = Symbol('theme')
export const userKey = Symbol('user')
```

```js
import { provide, inject } from 'vue'
import { themeKey } from './keys'

provide(themeKey, 'dark')
const theme = inject(themeKey, 'light')
```

---

## 三、完整示例

下面是一个主题与权限上下文：根组件提供当前用户、主题和切换方法，深层菜单组件注入使用，避免中间层反复透传：

```vue
<!-- App.vue -->
<script setup>
import { computed, provide, readonly, ref } from 'vue'
import AppLayout from './components/AppLayout.vue'
import { themeKey, toggleThemeKey, userKey } from './keys'

const user = ref({
  id: 1,
  name: '林晓',
  role: 'admin',
  permissions: ['order:read', 'order:write', 'user:manage']
})

const theme = ref('light')

const isAdmin = computed(() => user.value.role === 'admin')

function toggleTheme() {
  theme.value = theme.value === 'light' ? 'dark' : 'light'
}

function hasPermission(code) {
  return user.value.permissions.includes(code)
}

provide(userKey, readonly(user))
provide(themeKey, readonly(theme))
provide(toggleThemeKey, toggleTheme)
provide('isAdmin', isAdmin)
provide('hasPermission', hasPermission)
</script>

<template>
  <div class="app" :class="theme">
    <AppLayout />
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: #f8fafc;
  color: #0f172a;
}

.app.dark {
  background: #0f172a;
  color: #e2e8f0;
}
</style>
```

```vue
<!-- components/AppLayout.vue -->
<script setup>
import SideMenu from './SideMenu.vue'
</script>

<template>
  <div class="layout">
    <SideMenu />
    <main>
      <slot />
    </main>
  </div>
</template>
```

```vue
<!-- components/SideMenu.vue -->
<script setup>
import { inject } from 'vue'
import { themeKey, toggleThemeKey, userKey } from '../keys'

// 中间层不需要声明任何 props
const user = inject(userKey)
const theme = inject(themeKey)
const toggleTheme = inject(toggleThemeKey)
const hasPermission = inject('hasPermission', () => false)
</script>

<template>
  <aside class="menu">
    <p class="user">{{ user.name }}</p>

    <nav>
      <a v-if="hasPermission('order:read')" href="/orders">订单管理</a>
      <a v-if="hasPermission('user:manage')" href="/users">用户管理</a>
      <a href="/settings">个人设置</a>
    </nav>

    <button type="button" @click="toggleTheme">
      切换到{{ theme === 'light' ? '深色' : '浅色' }}主题
    </button>
  </aside>
</template>

<style scoped>
.menu {
  width: 220px;
  padding: 20px;
  border-right: 1px solid #e2e8f0;
}

.user {
  margin: 0 0 16px;
  font-weight: 700;
}

nav {
  display: grid;
  gap: 10px;
}

a {
  color: inherit;
  text-decoration: none;
}

button {
  margin-top: 20px;
  padding: 8px 12px;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  background: transparent;
  color: inherit;
  cursor: pointer;
}
</style>
```

选项式 API 对照：

```vue
<script>
export default {
  data() {
    return {
      // data 返回的对象会被 Vue 变成响应式
      themeState: { value: 'light' }
    }
  },
  provide() {
    return {
      // 后代通过 theme.value 读取，修改 theme.value 会同步
      theme: this.themeState,
      toggleTheme: this.toggleTheme
    }
  },
  inject: {
    user: {
      from: 'user',
      default: null
    }
  },
  methods: {
    toggleTheme() {
      this.themeState.value = this.themeState.value === 'light' ? 'dark' : 'light'
    }
  }
}
</script>
```

---

## 四、常见坑 / 注意事项

1. **`inject` 必须在 `setup` 同步阶段调用**：在异步回调或 `await` 之后调用会找不到注入上下文，应提前保存到变量。
2. **provide 普通值不会自动更新**：`provide('theme', theme.value)` 提供的是字符串快照；应提供 `ref` 本身或 `readonly(ref)`。
3. **后代可以修改 provide 的响应式对象**：如果不想允许，使用 `readonly` 包装，或只提供修改方法，不提供原始 ref。
4. **字符串 key 可能冲突**：大型项目、第三方库都可能用同名 key，优先使用 Symbol 或统一的 `keys.js`。
5. **`inject` 找不到 key 时返回 `undefined`**：没有默认值时要判空，否则访问属性会报错。
6. **默认值是函数时要传第三个参数**：`inject(key, () => 'fallback', false)` 才会把函数本身作为默认值，否则 Vue 会把它当工厂函数调用。
7. **不要用 provide/inject 替代所有 props**：它降低了数据来源的可见性，只适合跨多层、上下文型的数据；父子直接通信仍应使用 props/emits。
8. **应用级 provide 要在 mount 前注册**：`app.provide` 之后再挂载，确保所有组件都能注入到。

---

## 五、官方文档

- 依赖注入：https://cn.vuejs.org/guide/components/provide-inject.html
- `provide` API：https://cn.vuejs.org/api/composition-api-dependency-injection.html#provide
- `inject` API：https://cn.vuejs.org/api/composition-api-dependency-injection.html#inject
- 应用级 provide：https://cn.vuejs.org/api/application.html#app-provide

---

## 六、速查表

| 语法                    | 作用         | 示例                                      |
| ----------------------- | ------------ | ----------------------------------------- |
| `provide(key, value)`   | 提供数据     | `provide('theme', theme)`                 |
| `inject(key)`           | 注入数据     | `const theme = inject('theme')`           |
| `inject(key, fallback)` | 提供默认值   | `inject('theme', 'light')`                |
| `readonly(ref)`         | 防止后代修改 | `provide('theme', readonly(theme))`       |
| `Symbol()`              | 唯一注入 key | `export const themeKey = Symbol('theme')` |
