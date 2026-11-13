# 组件 v-model

> 组件上的 `v-model` 是“prop + 事件”的语法糖，让父子组件可以像表单控件一样双向绑定，同时保持数据修改发生在父组件。

## 一、核心概念

- **默认约定**：`v-model` 默认绑定 `modelValue` prop，并监听 `update:modelValue` 事件。父组件写 `v-model="value"`，等价于 `:model-value="value" @update:model-value="value = $event"`。
- **`defineModel`**：Vue 3.4 起提供的编译宏，声明一个可读写的 ref，内部自动处理 prop 和事件，是目前最简洁的写法。
- **手动实现**：不依赖 `defineModel` 时，用 `defineProps` 接收 `modelValue`，用 `defineEmits` 派发 `update:modelValue`。
- **命名 v-model**：`v-model:title="title"` 对应 `title` prop 和 `update:title` 事件，一个组件可以同时使用多个 v-model。
- **修饰符**：`v-model.trim`、`v-model.number` 或自定义修饰符，可以通过 `defineModel` 的 `set` 选项统一处理。
- **单向数据流不变**：`v-model` 只是写法上的双向，真正的赋值仍由父组件完成，子组件不直接修改 prop。

---

## 二、基础语法

使用 `defineModel`：

```vue
<script setup>
// 默认 modelValue
const model = defineModel({ type: String, default: '' })
</script>

<template>
  <input v-model="model" />
</template>
```

手动实现：

```vue
<script setup>
const props = defineProps({
  modelValue: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['update:modelValue'])

function updateValue(event) {
  emit('update:modelValue', event.target.value)
}
</script>

<template>
  <input :value="props.modelValue" @input="updateValue" />
</template>
```

多个 v-model：

```vue
<!-- 子组件 -->
<script setup>
const firstName = defineModel('firstName')
const lastName = defineModel('lastName')
</script>

<template>
  <input v-model="firstName" />
  <input v-model="lastName" />
</template>

<!-- 父组件 -->
<UserNameForm v-model:first-name="first" v-model:last-name="last" />
```

---

## 三、完整示例

下面是一个数量选择器组件，支持 `v-model`、最小/最大值、步进和自定义修饰符，适合购物车场景：

```vue
<script setup>
import { computed } from 'vue'

const props = defineProps({
  min: {
    type: Number,
    default: 1
  },
  max: {
    type: Number,
    default: 99
  },
  step: {
    type: Number,
    default: 1
  }
})

// 第二个返回值是 modifiers 对象，例如 v-model.integer
const [quantity, modifiers] = defineModel({
  type: Number,
  default: 1,
  set(value) {
    let next = Number(value)

    if (Number.isNaN(next)) next = props.min
    if (modifiers.integer) next = Math.trunc(next)

    return Math.min(props.max, Math.max(props.min, next))
  }
})

const canDecrease = computed(() => quantity.value > props.min)
const canIncrease = computed(() => quantity.value < props.max)

function decrease() {
  if (canDecrease.value) {
    quantity.value -= props.step
  }
}

function increase() {
  if (canIncrease.value) {
    quantity.value += props.step
  }
}

function handleInput(event) {
  quantity.value = event.target.value
}
</script>

<template>
  <div class="quantity">
    <button type="button" :disabled="!canDecrease" @click="decrease">-</button>

    <input
      :value="quantity"
      type="number"
      :min="min"
      :max="max"
      :step="step"
      @input="handleInput"
    />

    <button type="button" :disabled="!canIncrease" @click="increase">+</button>
  </div>
</template>

<style scoped>
.quantity {
  display: inline-flex;
  align-items: center;
  overflow: hidden;
  border: 1px solid #d1d5db;
  border-radius: 8px;
  font-family: system-ui, sans-serif;
}

.quantity button {
  width: 34px;
  height: 34px;
  border: 0;
  background: #f9fafb;
  cursor: pointer;
  font-size: 18px;
}

.quantity button:disabled {
  cursor: not-allowed;
  opacity: 0.35;
}

.quantity input {
  width: 52px;
  height: 34px;
  border: 0;
  border-right: 1px solid #e5e7eb;
  border-left: 1px solid #e5e7eb;
  outline: none;
  text-align: center;
}

.quantity input::-webkit-outer-spin-button,
.quantity input::-webkit-inner-spin-button {
  margin: 0;
  appearance: none;
}
</style>
```

父组件直接双向绑定：

```vue
<script setup>
import { ref } from 'vue'
import QuantityInput from './components/QuantityInput.vue'

const count = ref(2)
const price = 199

function addToCart() {
  console.log('加入购物车', { count: count.value, total: count.value * price })
}
</script>

<template>
  <div class="cart-line">
    <span>无线鼠标</span>
    <QuantityInput v-model.integer="count" :min="1" :max="10" />
    <strong>¥{{ count * price }}</strong>
    <button @click="addToCart">加入购物车</button>
  </div>
</template>
```

选项式 API 手动实现：

```vue
<script>
export default {
  props: {
    modelValue: {
      type: Number,
      default: 1
    }
  },
  emits: ['update:modelValue'],
  methods: {
    updateValue(value) {
      this.$emit('update:modelValue', value)
    }
  }
}
</script>

<template>
  <input
    type="number"
    :value="modelValue"
    @input="updateValue(Number($event.target.value))"
  />
</template>
```

---

## 四、常见坑 / 注意事项

1. **`defineModel` 需要 Vue 3.4 及以上**：旧版本项目只能使用 `modelValue` + `update:modelValue` 的手动写法。
2. **不要同时写 `props.modelValue` 和 `defineModel`**：两者会重复声明同一个 prop，导致编译错误或行为不一致。
3. **`defineModel` 返回的是 ref**：脚本中读写要加 `.value`，模板中自动解包。
4. **直接给 `model.value` 赋值会触发父组件更新**：这是预期行为，但不要在 `computed` 的 getter 里赋值，否则会违反纯函数原则。
5. **自定义修饰符要在 `set` 中处理**：`v-model.integer` 不会自动转整数，需要在 `set(value)` 里读取 `modifiers.integer` 并返回处理后的值。
6. **`v-model` 与 `:value` 不能混用在同一元素上**：同时写会导致重复绑定，Vue 会以后者覆盖前者，行为难以预测。
7. **多个 v-model 的参数名要使用 kebab-case**：`v-model:first-name` 对应 `defineModel('firstName')`，模板中不要写 `v-model:firstName`。
8. **表单原生 `v-model` 与组件 `v-model` 不要混淆**：原生控件由 Vue 内置处理，组件 `v-model` 依赖组件自身声明的 prop 和事件。

---

## 五、官方文档

- 组件 `v-model`：https://cn.vuejs.org/guide/components/v-model.html
- `defineModel`：https://cn.vuejs.org/api/sfc-script-setup.html#definemodel
- 表单输入绑定：https://cn.vuejs.org/guide/essentials/forms.html
- 自定义修饰符：https://cn.vuejs.org/guide/components/v-model.html#handling-v-model-modifiers

---

## 六、速查表

| 语法                   | 作用           | 示例                                 |
| ---------------------- | -------------- | ------------------------------------ |
| `v-model="value"`      | 默认双向绑定   | `<Child v-model="value" />`          |
| `defineModel()`        | 声明默认 model | `const model = defineModel()`        |
| `v-model:title`        | 命名双向绑定   | `<Child v-model:title="title" />`    |
| `defineModel('title')` | 声明命名 model | `const title = defineModel('title')` |
| `update:modelValue`    | 手动派发更新   | `emit('update:modelValue', value)`   |
