# 插槽 Slots

> 插槽让父组件把模板片段“填”进子组件预留的位置，从而在不修改子组件源码的情况下定制结构和内容。

## 一、核心概念

- **默认插槽**：子组件写 `<slot />`，父组件写在子组件标签内部的内容会替换这个位置。
- **具名插槽**：子组件用 `<slot name="header" />` 预留多个位置，父组件用 `<template #header>` 填充。
- **作用域插槽**：子组件通过 `<slot :item="item" />` 把内部数据暴露给父组件，父组件用 `#default="{ item }"` 接收。它把“渲染什么”交给父组件，把“数据从哪来”留在子组件。
- **后备内容**：`<slot>默认文字</slot>` 在父组件没有提供内容时显示，适合做空状态或占位。
- **动态插槽名**：`<template #[slotName]>` 可以根据变量决定填充哪个具名插槽。
- **插槽是编译期结构**：父组件提供的内容在父组件作用域中编译，能访问父组件的数据；作用域插槽额外获得子组件暴露的数据。

---

## 二、基础语法

```vue
<!-- BaseCard.vue -->
<template>
  <div class="card">
    <header>
      <slot name="header">默认标题</slot>
    </header>

    <main>
      <slot />
    </main>

    <footer>
      <slot name="footer" :year="2026" />
    </footer>
  </div>
</template>
```

```vue
<!-- 父组件 -->
<template>
  <BaseCard>
    <template #header>
      <h2>季度报告</h2>
    </template>

    <p>这里是卡片主体内容。</p>

    <template #footer="{ year }">
      <small>© {{ year }} 示例科技</small>
    </template>
  </BaseCard>
</template>
```

只使用默认插槽时可以省略 `<template>`：

```vue
<BaseCard>
  <p>直接写在组件标签里的内容会进入默认插槽。</p>
</BaseCard>
```

---

## 三、完整示例

下面是一个数据表格组件，提供表头、单元格和空状态三个插槽，并用作用域插槽把行数据交给父组件渲染：

```vue
<script setup>
defineProps({
  rows: {
    type: Array,
    default: () => []
  },
  rowKey: {
    type: String,
    default: 'id'
  },
  loading: {
    type: Boolean,
    default: false
  }
})
</script>

<template>
  <div class="data-table">
    <div class="table-head">
      <slot name="head">
        <span>名称</span>
        <span>数量</span>
        <span>金额</span>
        <span>操作</span>
      </slot>
    </div>

    <div v-if="loading" class="state">加载中...</div>

    <div v-else-if="rows.length === 0" class="state">
      <slot name="empty">
        <p>暂无数据</p>
      </slot>
    </div>

    <template v-else>
      <div
        v-for="(row, index) in rows"
        :key="row[rowKey]"
        class="table-row"
      >
        <!-- 默认插槽：把行数据和索引暴露给父组件 -->
        <slot :row="row" :index="index">
          <span>{{ row.name }}</span>
          <span>{{ row.quantity }}</span>
          <span>¥{{ row.amount }}</span>
          <span>-</span>
        </slot>
      </div>
    </template>

    <footer v-if="$slots.footer" class="table-footer">
      <slot name="footer" :total="rows.length" />
    </footer>
  </div>
</template>

<style scoped>
.data-table {
  overflow: hidden;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #ffffff;
  font-family: system-ui, sans-serif;
}

.table-head,
.table-row {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr;
  gap: 12px;
  align-items: center;
  padding: 12px 16px;
}

.table-head {
  background: #f9fafb;
  color: #6b7280;
  font-size: 13px;
  font-weight: 600;
}

.table-row {
  border-top: 1px solid #f3f4f6;
  font-size: 14px;
}

.state {
  padding: 40px;
  color: #9ca3af;
  text-align: center;
}

.table-footer {
  padding: 12px 16px;
  border-top: 1px solid #f3f4f6;
  background: #fafafa;
  color: #6b7280;
  font-size: 13px;
  text-align: right;
}
</style>
```

父组件使用作用域插槽自定义每一行：

```vue
<script setup>
import { ref } from 'vue'
import DataTable from './components/DataTable.vue'

const loading = ref(false)
const rows = ref([
  { id: 1, name: '机械键盘', quantity: 2, amount: 998, status: 'paid' },
  { id: 2, name: '无线鼠标', quantity: 1, amount: 199, status: 'pending' }
])

const statusText = {
  paid: '已支付',
  pending: '待支付'
}

function removeRow(id) {
  rows.value = rows.value.filter((row) => row.id !== id)
}
</script>

<template>
  <DataTable :rows="rows" :loading="loading">
    <template #head>
      <span>商品</span>
      <span>数量</span>
      <span>金额</span>
      <span>状态</span>
    </template>

    <template #default="{ row, index }">
      <span>{{ index + 1 }}. {{ row.name }}</span>
      <span>{{ row.quantity }}</span>
      <span>¥{{ row.amount }}</span>
      <button type="button" @click="removeRow(row.id)">
        {{ statusText[row.status] }} · 删除
      </button>
    </template>

    <template #empty>
      <p>购物车里还没有商品</p>
      <button type="button">去逛逛</button>
    </template>

    <template #footer="{ total }">
      共 {{ total }} 种商品
    </template>
  </DataTable>
</template>
```

---

## 四、常见坑 / 注意事项

1. **插槽内容访问的是父组件作用域**：父组件写在 `<Child>` 里的 `{{ message }}` 读取的是父组件的 `message`，不是子组件的。
2. **作用域插槽的数据要通过 `slot` 的 props 显式暴露**：子组件不写 `:row="row"`，父组件就拿不到 `row`。
3. **`v-slot` 只能写在 `<template>` 或组件标签上**：`<div v-slot="...">` 是无效的，必须用 `<template #default="...">`。
4. **默认插槽的简写**：`<template #default="{ row }">` 可以简写为 `<template #default="{ row }">`；当只有一个默认插槽时，也可以直接在组件标签上写 `v-slot="{ row }"`。
5. **`$slots` 不是响应式的**：它会在父组件重新渲染时更新，但不要把它当普通响应式对象去 `watch`。
6. **具名插槽不要与默认插槽内容混在一起**：父组件里所有未包在 `<template #name>` 中的顶层节点都会进入默认插槽。
7. **动态插槽名要使用合法名称**：`#[slotName]` 中 `slotName` 求值结果不能包含空格或非法字符。
8. **插槽内容较多时注意性能**：插槽函数会在父组件更新时重新执行，复杂列表可配合 `v-memo` 或拆分组件。

---

## 五、官方文档

- 插槽：https://cn.vuejs.org/guide/components/slots.html
- 作用域插槽：https://cn.vuejs.org/guide/components/slots.html#scoped-slots
- `useSlots`：https://cn.vuejs.org/api/composition-api-setup.html#setup-context
- `$slots`：https://cn.vuejs.org/api/component-instance.html#slots

---

## 六、速查表

| 语法                     | 作用             | 示例                                |
| ------------------------ | ---------------- | ----------------------------------- |
| `<slot />`               | 默认插槽         | `<slot>默认内容</slot>`             |
| `<slot name="header" />` | 具名插槽         | `<template #header>标题</template>` |
| `<slot :item="item" />`  | 作用域插槽       | `<template #default="{ item }">`    |
| `#default`               | 默认插槽简写     | `<template #default="{ row }">`     |
| `$slots.footer`          | 判断插槽是否存在 | `<footer v-if="$slots.footer">`     |
