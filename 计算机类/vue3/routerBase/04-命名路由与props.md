# 第 4 关 · 命名路由与 props

> 一句话：给每条路由起一个 `name`，跳转时只报「名字 + 参数」；再用 `props: true` 把路由参数变成组件的普通 props——页面从此不需要知道「路由」这回事。

📁 源码：`C:\Users\Shyanthen\Desktop\VueRouter\04-named`
🔗 上一关：[[计算机类/vue3/routerBase/03-动态参数.md|第 3 关 · 动态参数]] ｜ 下一关：[[计算机类/vue3/routerBase/05-嵌套路由.md|第 5 关 · 嵌套路由]]

---

## 一、本关目标

本关**不新增页面**，只做两处解耦：

| 关注点 | 第 3 关 | 第 4 关 |
| :--- | :--- | :--- |
| 跳转写法 | `to="/users/1"` / `router.push('/users/' + id)` | `:to="{ name: 'user', params: { id } }"` |
| 详情页取参 | `useRoute().params.id` | `defineProps({ id })` |

好处是：路由路径从 `/users/:id` 改成 `/people/:id` 时，**页面里的链接一行都不用改**。

### 路由表

```js
routes: [
  { path: '/', name: 'home', component: Home },
  { path: '/about', name: 'about', component: About },
  { path: '/users', name: 'users', component: Users },
  {
    path: '/users/:id',
    name: 'user',            // ← 名字叫 user
    component: UserDetail,
    props: true,             // ← 把 params 变成组件的 props
  },
]
```

---

## 二、逐文件说明

### `index.html`

- **作用**：入口 HTML。
- **差异**：`<title>第 4 关 · 命名路由与 props</title>`；其余仍为 `#app` + `./src/main.js`。

### `package.json`

- **作用**：依赖与脚本。
- **差异**：`"name": "vue-router-step-4"`；依赖版本与前面关卡一致。

### `package-lock.json`

- **作用**：锁定实际安装版本（vue 3.5.44 / vue-router 5.4.0 / vite 8.3.5）。
- **说明**：生成文件，不手改。

### `vite.config.js`

- **作用**：Vue 插件 + 固定端口。
- **差异**：`server.port: 5174`。

### `src/main.js`

- **作用**：路由表在本关加入 `name` 与 `props`。
- **完整路由表**：

```js
const router = createRouter({
  history: createWebHistory(),
  routes: [
    // 给每条路由起个名字：以后跳转说名字就行，不用再拼路径
    { path: '/', name: 'home', component: Home },
    { path: '/about', name: 'about', component: About },
    { path: '/users', name: 'users', component: Users },

    {
      path: '/users/:id',
      name: 'user', // ← 名字叫 user
      component: UserDetail,
      // props: true 会把 params 直接变成组件的 props
      props: true,
    },
  ],
})

createApp(App).use(router).mount('#app')
```

- **说明**：
  - `name` 在应用内必须唯一；后续关卡会继续给子路由命名（`settings`、`settings-security`）。
  - `name` 与 `path` 是两条独立的寻址方式；用名字跳转时，参数以 `params` 传入。
  - `props: true` 只对**动态段参数**生效：路由参数会以同名 prop 传给组件。若需要把静态值或查询串也变成 props，可以用对象形式或函数形式（本关未涉及）。
  - 参数经过 props 传递后仍然是**字符串**；`UserDetail` 里的 `id` prop 类型声明为 `String`。

### `src/data/users.js`

- **作用**：列表页与详情页共享的假数据。
- **差异**：与第 3 关完全相同。

```js
export const users = [
  { id: 1, name: '张伟', role: '前端工程师' },
  { id: 2, name: '李娜', role: '产品经理' },
  { id: 3, name: '王强', role: '后端工程师' },
]
```

### `src/App.vue`

- **作用**：根组件；导航从「写路径」改为「写名字」。
- **关键模板**：

```vue
<!-- to 可以直接写名字，不用拼路径 -->
<nav class="nav">
  <router-link :to="{ name: 'home' }">首页</router-link>
  <router-link :to="{ name: 'about' }">关于</router-link>
  <router-link :to="{ name: 'users' }">用户</router-link>
</nav>
```

- **说明**：
  - `:to` 传对象时，`name` 是必填的寻址字段，`params` / `query` 可选。
  - 对象形式的 `to` 会让 `router-link` 调 `router.resolve` 生成 `href`，因此右键复制链接、在新标签页打开仍然正常。
  - 「本步新增」说明块给出验证实验：把路由表里的 `/users/:id` 改成 `/people/:id`，页面里的链接不用动。
  - 注意：**只改 path 不改 name** 时，链接仍能正确解析；如果 name 拼错，开发环境会在控制台给出警告，且链接解析失败。

### `src/pages/Home.vue`

- **作用**：对比上一关与本关的两种写法。
- **说明**：纯静态文案，列出两行对照：

```text
链接从 to="/users/1" 变成 :to="{ name: 'user', params: { id: 1 } }"
详情页从 useRoute().params.id 变成 defineProps({ id })
```

### `src/pages/About.vue`

- **作用**：对照组页面，说明路由表「只加不改」。
- **说明**：与第 3 关文案相同；`/about` 仍是一条普通命名路由。

### `src/pages/Users.vue`

- **作用**：列表页；链接改为对象形式。
- **脚本**：`import { users } from '../data/users'`。
- **关键模板**：

```vue
<router-link :to="{ name: 'user', params: { id: user.id } }">
  <span class="avatar">{{ user.name.slice(0, 1) }}</span>
  <span>{{ user.name }}</span>
  <span class="tag">{{ user.role }}</span>
</router-link>
```

- **说明**：
  - 与第 3 关唯一的区别就是 `:to` 的写法；`.tag` 也从「路径」改成了「职位」。
  - 底部 `<pre class="code">` 用注释对比两种写法：「路径变了就得改这里」vs「路径怎么变，这里都不用动」。
  - `params` 里的 `id` 仍是数字，路由解析时会转成字符串交给组件。

### `src/pages/UserDetail.vue`

- **作用**：详情页；不再依赖 `useRoute()`，只接收 `id` prop。
- **脚本全文**：

```js
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { users } from '../data/users'

// props: true 之后，路由参数会作为 prop 传进来，
// 组件不需要知道「路由」这回事，它只是一个接收 id 的普通组件。
const props = defineProps({
  id: { type: String, required: true },
})

const router = useRouter()

const index = computed(() => users.findIndex((user) => String(user.id) === props.id))
const user = computed(() => users[index.value])
const prev = computed(() => users[index.value - 1])
const next = computed(() => users[index.value + 1])

function goto(id) {
  // 跳转也用名字：路径改了这里不用改
  router.push({ name: 'user', params: { id } })
}
```

- **说明**：
  - `defineProps` 声明 `id: String`，与路由参数的实际类型一致；`required: true` 表示必须传入。
  - `props.id` 在模板里可直接写 `id`，也可以写 `props.id`；本文件两种都用了（脚本里 `props.id`，模板里展示 `props.id`）。注意 `<script setup>` 中 `defineProps` 的返回值赋给 `props` 后，模板仍可用 `props.id`。
  - `computed` 依赖 `props.id`；prop 变化时自动重算，因此切换用户时页面内容会更新（组件实例仍被复用）。
  - `goto` 用名字跳转，不再拼路径。
  - 仍然保留了「没有这个人」的 `v-else` 分支：路由匹配成功但数据不存在。
  - 模板里的代码块回显了本关的两处关键配置。
- **与第 3 关的差异**：
  - 删除 `useRoute` 与 `route.params.id`；
  - 新增 `defineProps`；
  - `goto` 改为对象形式；
  - 计数器演示已被删除（复用知识点在第 3 关讲过）。

### `src/style.css`

- **作用**：共用演示样式。
- **差异**：与其它关卡完全相同（267 行，MD5 一致）。
- **本关用到的部分**：`.nav`、`.list`、`.avatar`、`.tag`、`.btn`、`.code` 等。

### `.gitignore`（关卡内）

- **作用**：忽略 `node_modules`、`dist`、`.DS_Store`；与其它关卡一致。

---

## 三、本关要点

- **命名路由**：`name` 是稳定的逻辑标识，`path` 是可变的物理地址；用名字跳转能把「地址变更」的影响限制在路由表内。
- **`props: true`**：把 `params` 映射为组件 props，降低组件对路由的耦合，组件也更容易单测（直接传 prop 即可）。
- **参数类型**：props 与 `route.params` 一样是字符串；需要数字时自行转换，或使用 `props` 函数做转换。
- **`useRouter()` 仍需要**：组件不再读路由参数，但「跳转」仍要路由器实例；若用 `<router-link>`，连 `useRouter` 都可以省掉。
- **name 唯一性**：重复命名会导致解析到错误记录，开发时留意控制台警告。

---

## 四、下一关

[[计算机类/vue3/routerBase/05-嵌套路由.md|第 5 关]] 引入 `children`：`/settings` 作为外壳，内部再放一个 `<router-view />` 渲染「个人资料」与「安全」两个子页面。
