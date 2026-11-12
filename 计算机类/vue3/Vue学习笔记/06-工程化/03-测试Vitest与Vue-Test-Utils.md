# 测试 Vitest 与 Vue Test Utils

> 单元测试验证组件在给定输入下的渲染和交互，端到端测试验证真实浏览器中的完整流程。Vue 官方推荐 Vitest + Vue Test Utils 做单元测试，Playwright/Cypress 做 E2E。

## 一、核心概念

- **Vitest**：基于 Vite 的测试运行器，复用 Vite 的配置、别名和转换能力，启动快，API 与 Jest 类似。
- **Vue Test Utils**：Vue 官方组件测试库，提供 `mount`、`shallowMount`、`wrapper.find`、`trigger`、`emitted` 等工具。
- **`mount` 与 `shallowMount`**：`mount` 渲染完整组件树；`shallowMount` 把子组件替换为 stub，适合只测当前组件。
- **测试环境**：组件测试需要 DOM，Vitest 配置 `environment: 'jsdom'` 或 `happy-dom`。
- **断言交互**：先通过 `find` 定位元素，再 `trigger` 触发事件，最后断言 DOM 文本、类名或 `emitted` 事件。
- **异步更新**：修改响应式数据后要 `await nextTick()`，异步请求要用 `flushPromises()` 等待微任务完成。

---

## 二、基础语法

```js
// vite.config.js 或 vitest.config.js
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    globals: true
  }
})
```

```js
// components/__tests__/StatusBadge.spec.js
import { mount } from '@vue/test-utils'
import { h } from 'vue'
import { describe, expect, it } from 'vitest'

const StatusBadge = {
  props: {
    status: {
      type: String,
      required: true
    }
  },
  setup(props) {
    return () => h(
      'span',
      { 'data-test': 'badge' },
      props.status === 'paid' ? '已支付' : '待支付'
    )
  }
}

describe('StatusBadge', () => {
  it('根据状态显示对应文案', () => {
    const wrapper = mount(StatusBadge, {
      props: { status: 'paid' }
    })

    expect(wrapper.get('[data-test="badge"]').text()).toBe('已支付')
  })
})
```

---

## 三、完整示例

下面是一个用户卡片组件及其测试，覆盖 props 渲染、事件派发、插槽和异步刷新：

```vue
<!-- components/UserCard.vue -->
<script setup>
import { computed, ref } from 'vue'

const props = defineProps({
  user: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(['follow', 'refresh'])

const loading = ref(false)
const followed = ref(Boolean(props.user.followed))

const initial = computed(() => props.user.name.slice(0, 1))

async function toggleFollow() {
  followed.value = !followed.value
  emit('follow', { id: props.user.id, followed: followed.value })
}

async function refresh() {
  loading.value = true
  await new Promise((resolve) => setTimeout(resolve, 50))
  loading.value = false
  emit('refresh', props.user.id)
}
</script>

<template>
  <article class="user-card" data-test="user-card">
    <div class="avatar" data-test="avatar">{{ initial }}</div>

    <div class="info">
      <h3 data-test="name">{{ user.name }}</h3>
      <p data-test="email">{{ user.email }}</p>
      <slot name="badge">
        <span class="badge">{{ user.role }}</span>
      </slot>
    </div>

    <button
      type="button"
      data-test="follow"
      :disabled="loading"
      @click="toggleFollow"
    >
      {{ followed ? '已关注' : '关注' }}
    </button>

    <button type="button" data-test="refresh" @click="refresh">
      {{ loading ? '刷新中' : '刷新' }}
    </button>
  </article>
</template>

<style scoped>
.user-card {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px;
  border: 1px solid #e5e7eb;
  border-radius: 10px;
  background: #ffffff;
  font-family: system-ui, sans-serif;
}

.avatar {
  display: grid;
  width: 42px;
  height: 42px;
  place-items: center;
  border-radius: 50%;
  background: #42b883;
  color: #ffffff;
}

.info {
  flex: 1;
}

.info h3 {
  margin: 0 0 4px;
  font-size: 15px;
}

.info p {
  margin: 0 0 4px;
  color: #6b7280;
  font-size: 12px;
}

.badge {
  padding: 2px 7px;
  border-radius: 999px;
  background: #f3f4f6;
  color: #4b5563;
  font-size: 11px;
}

button {
  padding: 6px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
  font-size: 12px;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}
</style>
```

```js
// components/__tests__/UserCard.spec.js
import { flushPromises, mount } from '@vue/test-utils'
import { describe, expect, it } from 'vitest'
import UserCard from '../UserCard.vue'

const user = {
  id: 1,
  name: '林晓',
  email: 'lin@example.com',
  role: 'admin',
  followed: false
}

describe('UserCard', () => {
  it('渲染用户信息', () => {
    const wrapper = mount(UserCard, {
      props: { user }
    })

    expect(wrapper.get('[data-test="name"]').text()).toBe('林晓')
    expect(wrapper.get('[data-test="email"]').text()).toBe('lin@example.com')
    expect(wrapper.get('[data-test="avatar"]').text()).toBe('林')
  })

  it('点击关注后派发 follow 事件并切换文案', async () => {
    const wrapper = mount(UserCard, {
      props: { user }
    })

    await wrapper.get('[data-test="follow"]').trigger('click')

    expect(wrapper.get('[data-test="follow"]').text()).toBe('已关注')
    expect(wrapper.emitted('follow')).toEqual([
      [{ id: 1, followed: true }]
    ])
  })

  it('允许通过插槽覆盖角色标签', () => {
    const wrapper = mount(UserCard, {
      props: { user },
      slots: {
        badge: '<span data-test="custom-badge">VIP</span>'
      }
    })

    expect(wrapper.get('[data-test="custom-badge"]').text()).toBe('VIP')
  })

  it('刷新期间禁用按钮，完成后派发 refresh', async () => {
    const wrapper = mount(UserCard, {
      props: { user }
    })

    const button = wrapper.get('[data-test="refresh"]')
    await button.trigger('click')

    expect(button.text()).toBe('刷新中')
    expect(button.attributes('disabled')).toBeDefined()

    await flushPromises()

    expect(wrapper.emitted('refresh')).toEqual([[1]])
  })

  it('刷新完成后恢复按钮状态', async () => {
    const wrapper = mount(UserCard, {
      props: { user }
    })

    await wrapper.get('[data-test="refresh"]').trigger('click')
    await flushPromises()

    const button = wrapper.get('[data-test="refresh"]')
    expect(button.text()).toBe('刷新')
    expect(button.attributes('disabled')).toBeUndefined()
  })
})
```

E2E 示例（Playwright）：

```js
// e2e/orders.spec.js
import { expect, test } from '@playwright/test'

test('用户可以搜索订单并打开详情', async ({ page }) => {
  await page.goto('/orders')
  await page.getByPlaceholder('搜索订单').fill('SO2026001')
  await expect(page.getByText('SO2026001')).toBeVisible()

  await page.getByRole('button', { name: '查看详情' }).first().click()
  await expect(page).toHaveURL(/\/orders\/1/)
})
```

---

## 四、常见坑 / 注意事项

1. **忘记 `await trigger()`**：Vue 的 DOM 更新是异步的，触发事件后不等待，断言可能读到旧 DOM。
2. **异步请求要用 `flushPromises`**：`setTimeout`、Promise 链完成后，测试才能看到最终状态；假定时器要配合 `vi.useFakeTimers()`。
3. **`shallowMount` 不会渲染子组件内部**：如果断言依赖子组件内容，应使用 `mount` 或为子组件提供 stub。
4. **全局插件要显式注册**：组件使用 `RouterLink`、Pinia store 时，测试中要传入 `global.plugins`，否则会报注入缺失。
5. **不要测试实现细节**：断言内部变量名、私有方法会让重构频繁失败；应断言用户可见的文本、角色和事件。
6. **`data-test` 比 class 稳定**：样式类名经常变化，测试选择器应使用专门的 `data-test` 属性。
7. **E2E 测试要独立准备数据**：依赖上一个用例留下的数据会导致顺序耦合，应在 `beforeEach` 中重置数据库或使用测试账号。
8. **测试环境与生产环境不同**：jsdom 不实现布局、滚动、Canvas 等能力，涉及这些的组件应放到浏览器测试或 E2E 中。

---

## 五、官方文档

- Vue 测试：https://cn.vuejs.org/guide/scaling-up/testing.html
- Vue Test Utils：https://test-utils.vuejs.org/zh/
- Vitest：https://cn.vitest.dev/
- Playwright：https://playwright.dev/docs/intro
- Cypress：https://docs.cypress.io/

---

## 六、速查表

| 语法                 | 作用         | 示例                                |
| -------------------- | ------------ | ----------------------------------- |
| `mount(Comp)`        | 渲染完整组件 | `const wrapper = mount(UserCard)`   |
| `shallowMount(Comp)` | 浅渲染       | 子组件替换为 stub                   |
| `wrapper.get(sel)`   | 查找单个元素 | `wrapper.get('[data-test="name"]')` |
| `trigger('click')`   | 触发事件     | `await button.trigger('click')`     |
| `emitted('follow')`  | 获取派发事件 | `wrapper.emitted('follow')`         |
