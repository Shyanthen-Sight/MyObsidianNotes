# Teleport 与过渡动画

> `Teleport` 把组件的一部分模板传送到 DOM 的其他位置，`Transition` / `TransitionGroup` 则为元素的进入、离开和列表移动提供动画能力。

## 一、核心概念

- **Teleport 的作用**：弹窗、通知、下拉菜单经常需要挂到 `<body>` 下，避免被父元素的 `overflow: hidden`、`transform`、`z-index` 层级影响。
- **`to` 目标**：`<Teleport to="body">` 把内容传送到 `body`；也可以传 CSS 选择器或真实 DOM 元素。
- **`disabled`**：为 `true` 时内容留在原位置，适合在移动端或特定布局下关闭传送。
- **Transition 的 CSS 类**：进入分 `enter-from`、`enter-active`、`enter-to`，离开分 `leave-from`、`leave-active`、`leave-to`；加 `name="fade"` 后类名前缀变为 `fade-`。
- **过渡模式**：`mode="out-in"` 先执行离开动画再进入，`mode="in-out"` 相反；不加模式时两个元素可能同时存在。
- **TransitionGroup**：用于 `v-for` 列表，除了进入/离开，还支持 `move-class` 实现移动动画，要求每个子元素有稳定的 `key`。

---

## 二、基础语法

Teleport：

```vue
<template>
  <button @click="open = true">打开弹窗</button>

  <Teleport to="body">
    <div v-if="open" class="modal">
      弹窗内容
    </div>
  </Teleport>
</template>
```

Transition：

```vue
<template>
  <Transition name="fade">
    <p v-if="visible">这段内容会淡入淡出</p>
  </Transition>
</template>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.25s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

TransitionGroup：

```vue
<template>
  <TransitionGroup name="list" tag="ul">
    <li v-for="item in items" :key="item.id">{{ item.name }}</li>
  </TransitionGroup>
</template>

<style scoped>
.list-enter-active,
.list-leave-active {
  transition: all 0.3s ease;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(20px);
}

.list-move {
  transition: transform 0.3s ease;
}
</style>
```

---

## 三、完整示例

下面是一个商品编辑弹窗，使用 Teleport 挂到 `body`，使用 Transition 做遮罩和面板动画，并在打开时锁定页面滚动：

```vue
<script setup>
import { onBeforeUnmount, ref, watch } from 'vue'

const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false
  },
  product: {
    type: Object,
    default: () => ({ name: '', price: 0 })
  }
})

const emit = defineEmits(['update:modelValue', 'save'])

const form = ref({
  name: props.product.name,
  price: props.product.price
})

const visible = ref(props.modelValue)

watch(() => props.modelValue, (value) => {
  visible.value = value
  if (value) {
    form.value = {
      name: props.product.name,
      price: props.product.price
    }
  }
})

watch(visible, (value) => {
  emit('update:modelValue', value)
  document.body.style.overflow = value ? 'hidden' : ''
})

function close() {
  visible.value = false
}

function save() {
  if (!form.value.name.trim()) return
  emit('save', {
    ...props.product,
    name: form.value.name.trim(),
    price: Number(form.value.price)
  })
  close()
}

onBeforeUnmount(() => {
  document.body.style.overflow = ''
})
</script>

<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="visible" class="overlay" @click.self="close">
        <div class="dialog" role="dialog" aria-modal="true">
          <header>
            <h2>编辑商品</h2>
            <button type="button" aria-label="关闭" @click="close">×</button>
          </header>

          <label>
            商品名称
            <input v-model.trim="form.name" type="text" />
          </label>

          <label>
            商品价格
            <input v-model.number="form.price" type="number" min="0" />
          </label>

          <footer>
            <button type="button" class="cancel" @click="close">取消</button>
            <button type="button" class="confirm" @click="save">保存</button>
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.overlay {
  position: fixed;
  inset: 0;
  display: grid;
  place-items: center;
  padding: 20px;
  background: rgb(15 23 42 / 55%);
  z-index: 1000;
}

.dialog {
  width: 420px;
  max-width: 100%;
  padding: 22px;
  border-radius: 14px;
  background: #ffffff;
  box-shadow: 0 30px 70px rgb(15 23 42 / 30%);
  font-family: system-ui, sans-serif;
}

header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 18px;
}

h2 {
  margin: 0;
  font-size: 20px;
}

header button {
  border: 0;
  background: transparent;
  cursor: pointer;
  font-size: 24px;
  line-height: 1;
}

label {
  display: grid;
  gap: 8px;
  margin-bottom: 16px;
  color: #374151;
  font-size: 14px;
}

label input {
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 8px;
}

footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 22px;
}

footer button {
  padding: 9px 18px;
  border-radius: 8px;
  cursor: pointer;
}

.cancel {
  border: 1px solid #d1d5db;
  background: #ffffff;
}

.confirm {
  border: 0;
  background: #111827;
  color: #ffffff;
}

/* 遮罩淡入淡出 */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.25s ease;
}

.modal-enter-active .dialog,
.modal-leave-active .dialog {
  transition: transform 0.25s ease, opacity 0.25s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-from .dialog,
.modal-leave-to .dialog {
  opacity: 0;
  transform: translateY(16px) scale(0.98);
}
</style>
```

列表移动动画：

```vue
<TransitionGroup name="card" tag="div" class="grid">
  <ProductCard
    v-for="product in products"
    :key="product.id"
    :product="product"
  />
</TransitionGroup>
```

---

## 四、常见坑 / 注意事项

1. **Teleport 的目标必须已存在**：`to="#modal-root"` 时，目标节点要在 Teleport 渲染前就在 DOM 中，否则会报错或挂载失败。
2. **`scoped` 样式仍然作用于 Teleport 内容**：Vue 会给传送的元素保留作用域属性，但父组件的后代选择器可能因为 DOM 位置变化而失效。
3. **Transition 只能包裹一个根元素**：多个并列元素不会触发过渡，列表动画应使用 `TransitionGroup`。
4. **`v-if` 与 `v-show` 的过渡行为不同**：`v-if` 会触发真正的进入/离开；`v-show` 使用 `display` 切换，过渡类仍然可用，但元素始终在 DOM 中。
5. **`mode` 不加时可能重叠**：路由切换、弹窗切换时，两个元素可能同时存在，导致布局跳动；需要 `out-in` 或 `in-out`。
6. **TransitionGroup 的 `key` 必须稳定**：用索引做 key 会让移动动画错乱，应使用业务 id。
7. **`move-class` 要配合 `transition` 使用**：只写 `move-class` 没有过渡属性不会产生动画。
8. **动画要考虑无障碍与性能**：尊重 `prefers-reduced-motion`，优先使用 `transform` 和 `opacity`，避免动画期间触发布局重排。

---

## 五、官方文档

- Teleport：https://cn.vuejs.org/guide/built-ins/teleport.html
- Transition：https://cn.vuejs.org/guide/built-ins/transition.html
- TransitionGroup：https://cn.vuejs.org/guide/built-ins/transition-group.html
- 过渡的 CSS 类名：https://cn.vuejs.org/guide/built-ins/transition.html#transition-classes

---

## 六、速查表

| 语法                       | 作用           | 示例                                                   |
| -------------------------- | -------------- | ------------------------------------------------------ |
| `<Teleport to="body">`     | 传送到 body    | `<Teleport to="body"><Modal /></Teleport>`             |
| `to`                       | 指定目标选择器 | `<Teleport to="#modal-root">`                          |
| `:disabled`                | 关闭传送       | `<Teleport to="body" :disabled="isMobile">`            |
| `<Transition name="fade">` | 单元素过渡     | `<Transition name="fade"><p v-if="ok" /></Transition>` |
| `mode="out-in"`            | 先出后进       | `<Transition mode="out-in">`                           |
