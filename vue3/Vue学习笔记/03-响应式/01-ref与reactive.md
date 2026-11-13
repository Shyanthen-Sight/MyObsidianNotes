# ref 与 reactive

> `ref` 和 `reactive` 是 Vue 3 创建响应式数据的两个核心 API：前者可以包装任意值，后者把对象变成深层响应式代理。

## 一、核心概念

- **`ref`**：`ref(initialValue)` 返回一个带 `.value` 属性的响应式引用。它既能包装数字、字符串、布尔值，也能包装对象、数组。
- **自动解包**：在模板中，顶层 `ref` 会自动解包，写 `{{ count }}` 而不是 `{{ count.value }}`；在 JavaScript 中必须写 `.value`。
- **`reactive`**：`reactive(obj)` 返回对象的 Proxy，读写属性都会被追踪。它只能用于对象类型（对象、数组、Map、Set），不能包装基本类型。
- **深层响应式**：`reactive` 默认递归代理嵌套对象；`ref` 包装对象时，内部也会用 `reactive` 处理。
- **替换与解构**：直接重新赋值 `reactive` 变量会丢失响应性；解构 `reactive` 对象也会丢失响应性，需要 `toRefs`。
- **选择建议**：官方推荐优先使用 `ref`，因为它在传递、替换和类型推导上更一致；需要把一组状态作为整体操作时再用 `reactive`。

---

## 二、基础语法

```vue
<script setup>
import { reactive, ref, toRefs } from 'vue'

// 基本类型用 ref
const count = ref(0)
count.value += 1

// 对象用 reactive
const form = reactive({
  name: '',
  age: 18
})
form.name = '林晓'

// 解构 reactive 会失去响应性，用 toRefs 保留
const { name, age } = toRefs(form)
name.value = '陈默'

// 对象也可以放进 ref
const user = ref({ name: '林晓' })
user.value.name = '陈默'
</script>

<template>
  <p>{{ count }}</p>
  <p>{{ form.name }} - {{ age }}</p>
</template>
```

`ref` 在数组和集合中的行为：

```js
import { reactive, ref } from 'vue'

const list = ref([1, 2, 3])
list.value.push(4)

const state = reactive({
  tags: new Set(['vue']),
  scores: new Map()
})

state.tags.add('pinia')
state.scores.set('math', 95)
```

---

## 三、完整示例

下面是一个购物车组件，使用 `ref` 管理可替换的列表、用 `reactive` 管理表单、用 `computed` 计算总价，并演示 `toRefs` 与 `shallowRef`：

```vue
<script setup>
import { computed, reactive, ref, shallowRef, toRefs } from 'vue'

const cartItems = ref([
  { id: 1, name: '机械键盘', price: 499, quantity: 1, selected: true },
  { id: 2, name: '无线鼠标', price: 199, quantity: 2, selected: true },
  { id: 3, name: '显示器支架', price: 299, quantity: 1, selected: false }
])

const coupon = reactive({
  code: '',
  discount: 0,
  message: ''
})

// 大对象不需要深层响应式时用 shallowRef，避免递归代理开销
const logistics = shallowRef({
  company: '顺丰',
  trackingNo: 'SF123456789',
  history: []
})

const selectedItems = computed(() => {
  return cartItems.value.filter((item) => item.selected)
})

const total = computed(() => {
  return selectedItems.value.reduce((sum, item) => {
    return sum + item.price * item.quantity
  }, 0)
})

const payable = computed(() => {
  return Math.max(0, total.value - coupon.discount)
})

// 解构 coupon 时用 toRefs，模板中仍保持响应性
const { code, message } = toRefs(coupon)

function applyCoupon() {
  if (code.value.trim().toUpperCase() === 'VUE20') {
    coupon.discount = 20
    coupon.message = '优惠券已生效'
  } else {
    coupon.discount = 0
    coupon.message = '优惠券无效'
  }
}

function updateQuantity(id, delta) {
  const item = cartItems.value.find((row) => row.id === id)
  if (!item) return

  item.quantity = Math.max(1, item.quantity + delta)
}

function removeItem(id) {
  // 重新赋值 ref.value 是响应式的
  cartItems.value = cartItems.value.filter((item) => item.id !== id)
}
</script>

<template>
  <section class="cart">
    <h1>购物车</h1>

    <ul>
      <li v-for="item in cartItems" :key="item.id">
        <label>
          <input v-model="item.selected" type="checkbox" />
          <span>{{ item.name }}</span>
        </label>

        <span>¥{{ item.price }}</span>

        <div class="stepper">
          <button type="button" @click="updateQuantity(item.id, -1)">-</button>
          <span>{{ item.quantity }}</span>
          <button type="button" @click="updateQuantity(item.id, 1)">+</button>
        </div>

        <button type="button" class="remove" @click="removeItem(item.id)">
          删除
        </button>
      </li>
    </ul>

    <div class="coupon">
      <input v-model="code" placeholder="输入优惠码 VUE20" />
      <button type="button" @click="applyCoupon">使用</button>
    </div>
    <p v-if="message" class="message">{{ message }}</p>

    <footer>
      <p>已选 {{ selectedItems.length }} 件，合计 ¥{{ total }}</p>
      <p class="payable">应付：<strong>¥{{ payable }}</strong></p>
    </footer>

    <p class="logistics">
      物流：{{ logistics.company }} · {{ logistics.trackingNo }}
    </p>
  </section>
</template>

<style scoped>
.cart {
  max-width: 640px;
  margin: 40px auto;
  padding: 24px;
  border-radius: 14px;
  background: #ffffff;
  box-shadow: 0 12px 30px rgb(15 23 42 / 8%);
  font-family: system-ui, sans-serif;
}

h1 {
  margin: 0 0 18px;
  font-size: 22px;
}

ul {
  margin: 0;
  padding: 0;
  list-style: none;
}

li {
  display: grid;
  grid-template-columns: 1.6fr 0.8fr 1fr auto;
  gap: 12px;
  align-items: center;
  padding: 14px 0;
  border-bottom: 1px solid #f1f5f9;
}

label {
  display: flex;
  align-items: center;
  gap: 8px;
}

.stepper {
  display: flex;
  align-items: center;
  gap: 8px;
}

.stepper button {
  width: 26px;
  height: 26px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
}

.remove {
  border: 0;
  background: transparent;
  color: #dc2626;
  cursor: pointer;
  font-size: 13px;
}

.coupon {
  display: flex;
  gap: 8px;
  margin-top: 18px;
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
  margin: 8px 0 0;
  color: #059669;
  font-size: 13px;
}

footer {
  margin-top: 18px;
  padding-top: 16px;
  border-top: 1px solid #e5e7eb;
  text-align: right;
}

footer p {
  margin: 6px 0;
}

.payable strong {
  color: #dc2626;
  font-size: 24px;
}

.logistics {
  margin: 18px 0 0;
  color: #6b7280;
  font-size: 13px;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **脚本中漏写 `.value`**：`count++` 不会修改 ref 的值，正确写法是 `count.value++`；模板中则不能写 `.value`。
2. **不要解构 `reactive` 对象**：`const { name } = form` 得到的是普通字符串，后续 `form.name` 变化不会同步；需要解构时用 `toRefs(form)`。
3. **不要整体替换 `reactive` 变量**：`form = reactive({ ... })` 会让变量指向新代理，但模板仍引用旧代理；应逐属性修改，或改用 `ref`。
4. **`reactive` 不能包装基本类型**：`reactive(0)` 会返回 `0` 本身，不会响应；基本类型用 `ref`。
5. **`ref` 放在 `reactive` 对象里会自动解包**：`state.count` 直接就是数值，不需要 `.value`；但从 `reactive` 中解构出来又会失去解包行为。
6. **深层响应式有性能成本**：大列表、复杂图表数据如果只整体替换，不需要深层追踪，可用 `shallowRef` 或 `shallowReactive` 降低开销。
7. **`shallowRef` 的嵌套属性变化不会触发更新**：必须整体替换 `.value`，或手动调用 `triggerRef`。
8. **DOM 更新是异步的**：修改数据后立刻读取 DOM 可能还是旧值，需要 `await nextTick()`。

---

## 五、官方文档

- 响应式基础：https://cn.vuejs.org/guide/essentials/reactivity-fundamentals.html
- `ref` API：https://cn.vuejs.org/api/reactivity-core.html#ref
- `reactive` API：https://cn.vuejs.org/api/reactivity-core.html#reactive
- `toRefs`：https://cn.vuejs.org/api/reactivity-utilities.html#torefs
- 响应式深入：https://cn.vuejs.org/guide/extras/reactivity-in-depth.html

---

## 六、速查表

| 语法                | 作用                 | 示例                                    |
| ------------------- | -------------------- | --------------------------------------- |
| `ref(value)`        | 创建响应式引用       | `const count = ref(0)`                  |
| `count.value`       | 脚本中读写 ref       | `count.value++`                         |
| `reactive(obj)`     | 创建对象代理         | `const form = reactive({ name: '' })`   |
| `toRefs(obj)`       | 解构并保留响应性     | `const { name } = toRefs(form)`         |
| `shallowRef(value)` | 只追踪 `.value` 替换 | `const data = shallowRef({ list: [] })` |
