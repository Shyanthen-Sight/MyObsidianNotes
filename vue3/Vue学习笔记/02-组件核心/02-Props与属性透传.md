# Props 与属性透传

> `props` 是父组件向子组件传递数据的唯一推荐通道；属性透传则负责把父组件写在子组件标签上的 `class`、`style`、`id` 等属性自动落到子组件根元素上。

## 一、核心概念

- **单向数据流**：父组件把数据传给子组件，子组件只读。子组件想修改时，应派发事件让父组件改，避免数据来源变得不可追踪。
- **运行时声明**：`defineProps({ title: String })` 会在开发环境做类型检查、必填校验和默认值处理。
- **类型声明**：`defineProps<{ title: string }>()` 配合 TypeScript 使用，编译时生成对应的运行时声明。
- **默认值**：对象或数组的默认值必须使用工厂函数，避免多个组件实例共享同一个引用。
- **属性透传（Fallthrough Attributes）**：父组件传给子组件的非 props 属性，会自动落到子组件的单根元素上。`class` 和 `style` 会与子组件自身的类名/样式合并。
- **`inheritAttrs: false`**：关闭自动透传，改由开发者通过 `$attrs` 决定属性落到哪个元素，多根组件必须这样做。

---

## 二、基础语法

```vue
<script setup>
// 运行时声明
const props = defineProps({
  title: {
    type: String,
    required: true
  },
  count: {
    type: Number,
    default: 0
  },
  tags: {
    type: Array,
    default: () => []
  },
  status: {
    type: String,
    default: 'draft',
    validator: (value) => ['draft', 'published', 'archived'].includes(value)
  }
})

// 只读访问
console.log(props.title)
</script>

<template>
  <h2>{{ title }}</h2>
</template>
```

TypeScript 写法：

```vue
<script setup lang="ts">
interface Props {
  title: string
  count?: number
  tags?: string[]
}

// withDefaults 为可选 props 提供默认值
const props = withDefaults(defineProps<Props>(), {
  count: 0,
  tags: () => []
})
</script>
```

属性透传：

```vue
<!-- 父组件 -->
<BaseInput
  id="email"
  class="form-input"
  placeholder="请输入邮箱"
  data-test="email"
/>

<!-- BaseInput.vue：单根元素自动继承上述属性 -->
<template>
  <input type="text" />
</template>
```

关闭自动透传并手动指定位置：

```vue
<script setup>
defineOptions({ inheritAttrs: false })
</script>

<template>
  <label class="field">
    <span>邮箱</span>
    <!-- $attrs 包含 id、placeholder、data-test、class 等 -->
    <input v-bind="$attrs" />
  </label>
</template>
```

---

## 三、完整示例

下面是一个商品卡片组件，展示 props 校验、默认值、只读数据、属性透传和选项式 API 对照：

```vue
<script setup>
import { computed } from 'vue'

const props = defineProps({
  product: {
    type: Object,
    required: true,
    validator: (value) => {
      return typeof value.name === 'string' && typeof value.price === 'number'
    }
  },
  currency: {
    type: String,
    default: 'CNY'
  },
  showStock: {
    type: Boolean,
    default: true
  },
  tags: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['select'])

const priceText = computed(() => {
  const symbols = { CNY: '¥', USD: '$', EUR: '€' }
  return `${symbols[props.currency] ?? ''}${props.product.price}`
})

const soldOut = computed(() => props.product.stock <= 0)

function selectProduct() {
  if (soldOut.value) return
  emit('select', props.product)
}
</script>

<template>
  <!-- 父组件传入的 class、data-* 会自动落到这个根元素 -->
  <article class="product-card" @click="selectProduct">
    <div class="cover">
      <img :src="product.cover" :alt="product.name" />
      <span v-if="soldOut" class="sold-out">售罄</span>
    </div>

    <div class="body">
      <h3>{{ product.name }}</h3>

      <p class="price">{{ priceText }}</p>

      <p v-if="showStock" class="stock">
        {{ soldOut ? '暂时无货' : `剩余 ${product.stock} 件` }}
      </p>

      <div v-if="tags.length > 0" class="tags">
        <span v-for="tag in tags" :key="tag">{{ tag }}</span>
      </div>
    </div>
  </article>
</template>

<style scoped>
.product-card {
  width: 260px;
  overflow: hidden;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #ffffff;
  cursor: pointer;
  transition: box-shadow 0.2s ease;
}

.product-card:hover {
  box-shadow: 0 12px 28px rgb(15 23 42 / 12%);
}

.cover {
  position: relative;
  height: 160px;
  background: #f3f4f6;
}

.cover img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.sold-out {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgb(17 24 39 / 55%);
  color: #ffffff;
  font-size: 20px;
  letter-spacing: 4px;
}

.body {
  padding: 14px;
}

h3 {
  margin: 0 0 8px;
  font-size: 16px;
}

.price {
  margin: 0 0 8px;
  color: #dc2626;
  font-size: 20px;
  font-weight: 700;
}

.stock {
  margin: 0 0 10px;
  color: #6b7280;
  font-size: 13px;
}

.tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.tags span {
  padding: 2px 8px;
  border-radius: 999px;
  background: #f3f4f6;
  color: #4b5563;
  font-size: 12px;
}
</style>
```

父组件使用：

```vue
<script setup>
import ProductCard from './components/ProductCard.vue'

function handleSelect(product) {
  console.log('选择商品', product.name)
}
</script>

<template>
  <ProductCard
    class="recommend-card"
    data-source="home"
    :product="{
      id: 1,
      name: '降噪耳机',
      price: 899,
      stock: 0,
      cover: '/images/headphone.jpg'
    }"
    :tags="['热销', '包邮']"
    @select="handleSelect"
  />
</template>
```

选项式 API 对照：

```vue
<script>
export default {
  props: {
    product: {
      type: Object,
      required: true
    },
    currency: {
      type: String,
      default: 'CNY'
    }
  },
  emits: ['select'],
  computed: {
    soldOut() {
      return this.product.stock <= 0
    }
  },
  methods: {
    selectProduct() {
      if (!this.soldOut) this.$emit('select', this.product)
    }
  }
}
</script>
```

---

## 四、常见坑 / 注意事项

1. **子组件不能直接修改 props**：`props.product.price = 100` 会触发警告，并且破坏单向数据流。应派发事件，由父组件修改源数据。
2. **对象/数组默认值必须用工厂函数**：`default: []` 会让所有组件实例共享同一个数组；正确写法是 `default: () => []`。
3. **Boolean 类型的特殊转换**：声明为 `Boolean` 的 prop，不传时值为 `false`，只写属性名时值为 `true`，容易与“未传”混淆。
4. **props 解构在旧版本会丢失响应性**：Vue 3.5 之前 `const { title } = defineProps()` 得到的是普通值；需要响应式时使用 `props.title` 或 `toRefs`。即使 3.5 支持响应式解构，把值传入组合式函数时仍建议传 getter。
5. **`class` 和 `style` 也会出现在 `$attrs` 中**：关闭自动继承后，别忘了用 `v-bind="$attrs"` 把它们绑定到实际元素，否则样式会丢。
6. **多根组件不会自动透传**：模板有多个根节点时，Vue 无法判断属性该给谁，会给出警告；需要手动绑定 `$attrs`。
7. **`inheritAttrs: false` 不会影响 `class` / `style` 的合并逻辑**：它只关闭自动绑定，`$attrs` 里仍然包含它们。
8. **prop 命名在模板中用 kebab-case**：JavaScript 中写 `userName`，模板中写 `:user-name`；HTML 属性不区分大小写，驼峰写法在 DOM 模板中可能失效。

---

## 五、官方文档

- Props：https://cn.vuejs.org/guide/components/props.html
- 属性透传：https://cn.vuejs.org/guide/components/attrs.html
- `defineProps`：https://cn.vuejs.org/api/sfc-script-setup.html#defineprops-defineemits
- `withDefaults`：https://cn.vuejs.org/api/sfc-script-setup.html#default-props-values-when-using-type-declaration

---

## 六、速查表

| 语法                   | 作用                 | 示例                                               |
| ---------------------- | -------------------- | -------------------------------------------------- |
| `defineProps({})`      | 运行时声明 props     | `defineProps({ title: String })`                   |
| `defineProps<Props>()` | TypeScript 类型声明  | `defineProps<{ title: string }>()`                 |
| `withDefaults`         | 为类型声明提供默认值 | `withDefaults(defineProps<Props>(), { count: 0 })` |
| `default: () => []`    | 对象/数组默认值      | `tags: { type: Array, default: () => [] }`         |
| `inheritAttrs: false`  | 关闭自动透传         | `defineOptions({ inheritAttrs: false })`           |
