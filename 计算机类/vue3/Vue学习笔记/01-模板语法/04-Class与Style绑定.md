# Class 与 Style 绑定

> 在模板中根据状态动态切换样式，是构建交互界面的高频需求。Vue 对 `class` 和 `style` 做了专门增强，支持对象和数组语法。

## 一、核心概念

- **Class 对象语法**：键是类名，值是布尔表达式。值为真时添加该类，适合表示“选中”“禁用”“激活”等状态。
- **Class 数组语法**：数组元素可以是字符串、对象或三元表达式，适合组合多个来源的类名。
- **静态与动态共存**：`:class` 不会覆盖静态 `class`，Vue 会把两者合并。
- **Style 对象语法**：键使用 camelCase 或 kebab-case 字符串，值可以是静态值或响应式表达式。
- **Style 数组语法**：多个样式对象会按顺序合并，后面的覆盖前面的同名属性。
- **CSS 变量**：通过 `:style="{ '--theme-color': color }"` 把动态值传给 CSS，比拼接字符串更安全。

---

## 二、基础语法

```vue
<script setup>
import { computed, ref } from 'vue'

const isActive = ref(true)
const hasError = ref(false)
const size = ref('large')
const themeColor = ref('#42b883')

// 用 computed 把复杂判断从模板中抽离
const buttonClass = computed(() => ({
  'is-active': isActive.value,
  'is-error': hasError.value,
  [`size-${size.value}`]: true
}))

const buttonStyle = computed(() => ({
  backgroundColor: themeColor.value,
  borderRadius: '10px'
}))
</script>

<template>
  <!-- 对象语法 -->
  <button :class="{ 'is-active': isActive, 'is-error': hasError }">保存</button>

  <!-- 数组语法，可混合字符串与对象 -->
  <button :class="['btn', size, { 'is-active': isActive }]">提交</button>

  <!-- 计算属性，模板更简洁 -->
  <button :class="buttonClass">确认</button>

  <!-- 内联样式对象 -->
  <div :style="{ color: themeColor, fontSize: '18px' }">主题预览</div>

  <!-- 样式数组，后写的覆盖先写的 -->
  <div :style="[buttonStyle, { borderRadius: '4px' }]">合并样式</div>

  <!-- 动态 CSS 变量 -->
  <div class="theme-box" :style="{ '--theme-color': themeColor }">
    动态主题
  </div>
</template>

<style scoped>
.theme-box {
  border-left: 4px solid var(--theme-color);
}
</style>
```

---

## 三、完整示例

下面是一个带状态的告警通知组件，根据级别切换颜色、图标位置和背景，展示 Class 与 Style 的常见组合：

```vue
<script setup>
import { computed } from 'vue'

const props = defineProps({
  level: {
    type: String,
    default: 'info',
    validator: (value) => ['info', 'success', 'warning', 'error'].includes(value)
  },
  title: {
    type: String,
    required: true
  },
  message: {
    type: String,
    default: ''
  },
  closable: {
    type: Boolean,
    default: true
  },
  accentColor: {
    type: String,
    default: ''
  }
})

const emit = defineEmits(['close'])

const levelText = {
  info: '提示',
  success: '成功',
  warning: '警告',
  error: '错误'
}

// 根据级别生成 BEM 风格的类名
const levelClass = computed(() => `notice-${props.level}`)

// 允许外部覆盖左边框颜色
const accentStyle = computed(() => {
  return props.accentColor ? { borderLeftColor: props.accentColor } : {}
})
</script>

<template>
  <div
    class="notice"
    :class="[levelClass, { 'is-closable': closable }]"
    :style="accentStyle"
    role="alert"
  >
    <div class="notice-icon">{{ levelText[level].slice(0, 1) }}</div>

    <div class="notice-body">
      <h3>{{ title }}</h3>
      <p v-if="message">{{ message }}</p>
      <slot />
    </div>

    <button
      v-if="closable"
      type="button"
      class="notice-close"
      aria-label="关闭"
      @click="emit('close')"
    >
      ×
    </button>
  </div>
</template>

<style scoped>
.notice {
  display: flex;
  gap: 12px;
  padding: 16px;
  border: 1px solid transparent;
  border-left-width: 4px;
  border-radius: 10px;
  font-family: system-ui, sans-serif;
}

.notice-body {
  flex: 1;
}

.notice-body h3 {
  margin: 0 0 4px;
  font-size: 15px;
}

.notice-body p {
  margin: 0;
  color: inherit;
  font-size: 13px;
  line-height: 1.6;
  opacity: 0.8;
}

.notice-icon {
  display: grid;
  width: 28px;
  height: 28px;
  flex-shrink: 0;
  place-items: center;
  border-radius: 50%;
  background: currentColor;
  color: #ffffff;
  font-size: 13px;
}

.notice-close {
  align-self: flex-start;
  border: 0;
  background: transparent;
  color: inherit;
  cursor: pointer;
  font-size: 20px;
  line-height: 1;
  opacity: 0.5;
}

.notice-close:hover {
  opacity: 1;
}

.notice-info {
  border-color: #bae6fd;
  border-left-color: #0ea5e9;
  background: #f0f9ff;
  color: #075985;
}

.notice-success {
  border-color: #bbf7d0;
  border-left-color: #22c55e;
  background: #f0fdf4;
  color: #166534;
}

.notice-warning {
  border-color: #fde68a;
  border-left-color: #f59e0b;
  background: #fffbeb;
  color: #92400e;
}

.notice-error {
  border-color: #fecaca;
  border-left-color: #ef4444;
  background: #fef2f2;
  color: #991b1b;
}
</style>
```

---

## 四、常见坑 / 注意事项

1. **`:class` 的对象键要用引号包裹带连字符的类名**：`{ 'is-active': ok }` 正确，`{ is-active: ok }` 会被解析成减法表达式。
2. **不要直接拼接 `class` 字符串**：`class="btn {{ active ? 'on' : '' }}"` 是无效的，应使用 `:class="['btn', { on: active }]"`。
3. **样式对象的键推荐 camelCase**：`fontSize` 比 `'font-size'` 更符合 JavaScript 习惯；使用 kebab-case 时必须加引号。
4. **多个样式对象的合并是浅合并**：嵌套对象、CSS 变量不会递归合并，复杂主题建议使用 CSS 类或 CSS 变量。
5. **`scoped` 样式下动态类仍然有效**：Vue 会给元素加作用域属性，类名本身不变；但父组件想覆盖子组件内部类名时需要 `:deep()`。
6. **`!important` 会破坏动态样式**：如果 CSS 中写了 `color: red !important`，`:style="{ color: 'blue' }"` 不会生效。
7. **类名不要以数字开头**：`size-2` 可以，`2xl` 作为类名时需要转义，在模板中容易出错，建议加前缀。

---

## 五、官方文档

- Class 与 Style 绑定：https://cn.vuejs.org/guide/essentials/class-and-style.html
- 组件样式特性：https://cn.vuejs.org/guide/scaling-up/sfc.html#scoped-css
- `:deep()` 与 `:slotted()`：https://cn.vuejs.org/api/sfc-css-features.html

---

## 六、速查表

| 语法                        | 作用             | 示例                                    |
| --------------------------- | ---------------- | --------------------------------------- |
| `:class="{ active: ok }"`   | 条件添加类       | `<div :class="{ active: isActive }">`   |
| `:class="['a', { b: ok }]"` | 组合类名         | `<div :class="['card', { selected }]">` |
| `:style="{ color }"`        | 绑定内联样式     | `<p :style="{ color: themeColor }">`    |
| `:style="[a, b]"`           | 合并多个样式对象 | `<p :style="[base, override]">`         |
| `--var`                     | 动态 CSS 变量    | `:style="{ '--main': color }"`          |
