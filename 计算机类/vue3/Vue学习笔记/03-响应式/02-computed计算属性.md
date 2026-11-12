# computed 计算属性

> `computed` 用来声明“由其他响应式数据推导出来的值”，它会自动追踪依赖并缓存结果，只有依赖变化时才重新计算。

## 一、核心概念

- **派生状态**：总价、过滤后的列表、格式化后的日期都属于派生状态，不应该再单独用 `ref` 存一份，否则容易出现数据不同步。
- **缓存机制**：计算属性会缓存 getter 的返回值。模板多次读取同一个计算属性，只要依赖没变，getter 只执行一次。
- **依赖追踪**：getter 执行时读取了哪些响应式属性，Vue 就记录哪些依赖；依赖变化时标记为脏，下次读取时重新计算。
- **只读与可写**：默认是只读的；传入 `{ get, set }` 可以创建可写计算属性，常用于把“全选”“单价”这类派生值反向写回源数据。
- **返回值是 ref**：脚本中访问要加 `.value`，模板中自动解包。
- **与方法的区别**：方法每次调用都会执行；计算属性只在依赖变化时重新计算。模板中不要用方法代替计算属性做昂贵计算。

---

## 二、基础语法

```vue
<script setup>
import { computed, ref } from 'vue'

const price = ref(100)
const quantity = ref(2)
const taxRate = ref(0.06)

// 只读计算属性
const subtotal = computed(() => price.value * quantity.value)
const total = computed(() => subtotal.value * (1 + taxRate.value))

// 可写计算属性：读取时返回格式化值，写入时反推源数据
const priceWithTax = computed({
  get() {
    return Number((price.value * (1 + taxRate.value)).toFixed(2))
  },
  set(value) {
    price.value = Number((value / (1 + taxRate.value)).toFixed(2))
  }
})
</script>

<template>
  <p>小计：{{ subtotal }}</p>
  <p>含税总价：{{ total }}</p>
  <input v-model.number="priceWithTax" type="number" />
</template>
```

---

## 三、完整示例

下面是一个订单结算面板，包含商品数量、折扣、运费、税费和可编辑的收货备注，展示多个计算属性之间的链式依赖：

```vue
<script setup>
import { computed, reactive } from 'vue'

const order = reactive({
  items: [
    { id: 1, name: '机械键盘', price: 499, quantity: 1, stock: 20 },
    { id: 2, name: '人体工学椅', price: 1299, quantity: 1, stock: 5 },
    { id: 3, name: '桌面收纳盒', price: 89, quantity: 3, stock: 50 }
  ],
  discountRate: 0.9,
  shippingFee: 12,
  taxRate: 0.06,
  remark: ''
})

const itemCount = computed(() => {
  return order.items.reduce((sum, item) => sum + item.quantity, 0)
})

const subtotal = computed(() => {
  return order.items.reduce((sum, item) => {
    return sum + item.price * item.quantity
  }, 0)
})

const discountAmount = computed(() => {
  return subtotal.value * (1 - order.discountRate)
})

const taxableAmount = computed(() => {
  return subtotal.value - discountAmount.value + order.shippingFee
})

const tax = computed(() => {
  return taxableAmount.value * order.taxRate
})

const payable = computed(() => {
  return taxableAmount.value + tax.value
})

const hasOverStock = computed(() => {
  return order.items.some((item) => item.quantity > item.stock)
})

// 可写计算属性：在输入框里直接修改某件商品的数量
const firstItemQuantity = computed({
  get() {
    return order.items[0].quantity
  },
  set(value) {
    const next = Number(value)
    order.items[0].quantity = Number.isNaN(next)
      ? 1
      : Math.min(order.items[0].stock, Math.max(1, next))
  }
})

function changeQuantity(item, delta) {
  item.quantity = Math.min(item.stock, Math.max(1, item.quantity + delta))
}

function formatMoney(value) {
  return `¥${value.toFixed(2)}`
}
</script>

<template>
  <section class="checkout">
    <h1>订单结算</h1>

    <ul class="items">
      <li v-for="item in order.items" :key="item.id">
        <div>
          <strong>{{ item.name }}</strong>
          <span>{{ formatMoney(item.price) }} / 件</span>
        </div>

        <div class="quantity">
          <button type="button" @click="changeQuantity(item, -1)">-</button>
          <span>{{ item.quantity }}</span>
          <button type="button" @click="changeQuantity(item, 1)">+</button>
        </div>

        <span class="line-total">
          {{ formatMoney(item.price * item.quantity) }}
        </span>
      </li>
    </ul>

    <label class="quick-edit">
      快速修改第一件商品数量
      <input v-model.number="firstItemQuantity" type="number" min="1" />
    </label>

    <p v-if="hasOverStock" class="warning">部分商品数量超过库存，请调整</p>

    <dl class="summary">
      <div>
        <dt>商品件数</dt>
        <dd>{{ itemCount }} 件</dd>
      </div>
      <div>
        <dt>商品小计</dt>
        <dd>{{ formatMoney(subtotal) }}</dd>
      </div>
      <div>
        <dt>折扣优惠</dt>
        <dd class="discount">-{{ formatMoney(discountAmount) }}</dd>
      </div>
      <div>
        <dt>运费</dt>
        <dd>{{ formatMoney(order.shippingFee) }}</dd>
      </div>
      <div>
        <dt>税费</dt>
        <dd>{{ formatMoney(tax) }}</dd>
      </div>
      <div class="total">
        <dt>应付金额</dt>
        <dd>{{ formatMoney(payable) }}</dd>
      </div>
    </dl>

    <label class="remark">
      订单备注
      <textarea v-model="order.remark" rows="2" placeholder="选填"></textarea>
    </label>

    <button class="submit" type="button" :disabled="hasOverStock">
      提交订单
    </button>
  </section>
</template>

<style scoped>
.checkout {
  max-width: 620px;
  margin: 40px auto;
  padding: 24px;
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  background: #ffffff;
  font-family: system-ui, sans-serif;
}

h1 {
  margin: 0 0 20px;
  font-size: 22px;
}

.items {
  margin: 0;
  padding: 0;
  list-style: none;
}

.items li {
  display: grid;
  grid-template-columns: 1fr auto auto;
  gap: 16px;
  align-items: center;
  padding: 12px 0;
  border-bottom: 1px solid #f1f5f9;
}

.items strong {
  display: block;
  margin-bottom: 4px;
}

.items span {
  color: #6b7280;
  font-size: 13px;
}

.quantity {
  display: flex;
  align-items: center;
  gap: 8px;
}

.quantity button {
  width: 26px;
  height: 26px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
}

.line-total {
  min-width: 80px;
  color: #111827;
  font-weight: 600;
  text-align: right;
}

.quick-edit {
  display: grid;
  gap: 6px;
  margin: 16px 0;
  color: #4b5563;
  font-size: 13px;
}

.quick-edit input {
  width: 100px;
  padding: 7px 10px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
}

.warning {
  color: #dc2626;
  font-size: 13px;
}

.summary {
  margin: 18px 0;
  padding: 16px;
  border-radius: 10px;
  background: #f9fafb;
}

.summary div {
  display: flex;
  justify-content: space-between;
  padding: 5px 0;
  color: #4b5563;
  font-size: 14px;
}

.summary dt,
.summary dd {
  margin: 0;
}

.summary .discount {
  color: #059669;
}

.summary .total {
  margin-top: 8px;
  padding-top: 10px;
  border-top: 1px solid #e5e7eb;
  color: #111827;
  font-size: 16px;
  font-weight: 700;
}

.summary .total dd {
  color: #dc2626;
  font-size: 22px;
}

.remark {
  display: grid;
  gap: 6px;
  margin-bottom: 16px;
  color: #4b5563;
  font-size: 13px;
}

.remark textarea {
  padding: 10px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  resize: vertical;
}

.submit {
  width: 100%;
  padding: 12px;
  border: 0;
  border-radius: 8px;
  background: #111827;
  color: #ffffff;
  cursor: pointer;
}

.submit:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **getter 必须有返回值**：`computed(() => { total.value + 1 })` 没有 `return`，结果是 `undefined`。
2. **不要在 getter 里做副作用**：发请求、修改其他状态、操作 DOM 都不应该放在 computed 中，getter 应该是纯函数。
3. **computed 不能直接写 `async`**：异步逻辑应使用 `watch` 或 `watchEffect`，或用 `ref` 保存异步结果。
4. **脚本中访问要加 `.value`**：`total` 是 ref 对象，`total.value` 才是数值；模板中自动解包。
5. **依赖必须是响应式的**：如果 getter 读取的是普通变量或已解构的普通值，依赖不会被追踪，计算属性不会更新。
6. **可写计算属性的 `set` 要负责校验**：直接 `set(value) { price.value = value }` 可能写入非法值，应做范围限制和类型转换。
7. **不要用 computed 替代 watch**：computed 适合“算出一个值”，watch 适合“值变化后执行动作”，职责不同。
8. **过度拆分计算属性会增加阅读成本**：简单表达式直接写在模板即可；只有被多处使用或逻辑复杂时才提取。

---

## 五、官方文档

- 计算属性：https://cn.vuejs.org/guide/essentials/computed.html
- `computed` API：https://cn.vuejs.org/api/reactivity-core.html#computed
- 可写计算属性：https://cn.vuejs.org/guide/essentials/computed.html#writable-computed
- 计算属性缓存 vs 方法：https://cn.vuejs.org/guide/essentials/computed.html#computed-caching-vs-methods

---

## 六、速查表

| 语法                     | 作用               | 示例                                              |
| ------------------------ | ------------------ | ------------------------------------------------- |
| `computed(() => ...)`    | 只读计算属性       | `const total = computed(() => a.value + b.value)` |
| `computed({ get, set })` | 可写计算属性       | `const full = computed({ get, set })`             |
| `total.value`            | 脚本中读取         | `console.log(total.value)`                        |
| `{{ total }}`            | 模板中自动解包     | `<p>{{ total }}</p>`                              |
| `computed` 依赖          | 自动追踪响应式依赖 | getter 中读取 `price.value`                       |
