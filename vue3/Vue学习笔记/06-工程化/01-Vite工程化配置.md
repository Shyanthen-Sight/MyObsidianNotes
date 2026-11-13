# Vite 工程化配置

> Vite 是 Vue 官方推荐的构建工具，开发时基于原生 ESM 提供极速启动，生产构建使用 Rollup 打包，并通过插件系统支持 Vue、TypeScript、CSS 预处理器等能力。

## 一、核心概念

- **开发服务器**：Vite 不为整个项目打包，而是按需编译浏览器请求的模块，因此冷启动很快；热更新只更新受影响的模块。
- **插件**：`@vitejs/plugin-vue` 负责编译 `.vue` 文件；Tailwind、SVG、压缩等能力也通过插件接入。
- **路径别名**：`resolve.alias` 把 `@` 映射到 `src`，避免深层相对路径 `../../../`。
- **环境变量**：只有以 `VITE_` 开头的变量会暴露给客户端，通过 `import.meta.env.VITE_XXX` 访问；`.env.development`、`.env.production` 按模式加载。
- **开发代理**：`server.proxy` 把 `/api` 转发到后端，解决开发环境跨域；它只在开发服务器生效，生产环境需要网关或同源部署。
- **生产构建**：`build.outDir`、`build.sourcemap`、`build.rollupOptions` 控制输出目录、source map 和分包策略。

---

## 二、基础语法

```js
// vite.config.js
import { fileURLToPath, URL } from 'node:url'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true
      }
    }
  }
})
```

环境变量文件：

```bash
# .env.development
VITE_API_BASE_URL=/api

# .env.production
VITE_API_BASE_URL=https://api.example.com
```

```js
// src/api/http.js
const baseURL = import.meta.env.VITE_API_BASE_URL
console.log(import.meta.env.MODE) // development / production
```

---

## 三、完整示例

下面是一个部署信息面板，展示环境变量、静态资源 URL、动态导入和 `import.meta.glob` 的用法：

```vue
<script setup>
import { computed, onMounted, ref } from 'vue'

const env = import.meta.env

const buildInfo = computed(() => ({
  mode: env.MODE,
  baseUrl: env.VITE_API_BASE_URL,
  appTitle: env.VITE_APP_TITLE ?? '管理后台',
  production: env.PROD,
  dev: env.DEV
}))

const logoUrl = new URL('../assets/logo.svg', import.meta.url).href

const features = ref([])

// 动态导入：只有点击时才加载对应模块
async function loadReport() {
  const module = await import('../modules/report.js')
  features.value.push(module.default)
}

// 批量收集某个目录下的文档，按原始文本导入
const docs = import.meta.glob('../docs/*.md', {
  eager: true,
  query: '?raw',
  import: 'default'
})

onMounted(() => {
  console.log('已加载文档数量', Object.keys(docs).length)
})
</script>

<template>
  <section class="deploy-info">
    <header>
      <img :src="logoUrl" alt="Logo" class="logo" />
      <div>
        <h1>{{ buildInfo.appTitle }}</h1>
        <p>当前模式：{{ buildInfo.mode }}</p>
      </div>
    </header>

    <dl>
      <div>
        <dt>接口地址</dt>
        <dd>{{ buildInfo.baseUrl }}</dd>
      </div>
      <div>
        <dt>生产环境</dt>
        <dd>{{ buildInfo.production ? '是' : '否' }}</dd>
      </div>
      <div>
        <dt>开发环境</dt>
        <dd>{{ buildInfo.dev ? '是' : '否' }}</dd>
      </div>
    </dl>

    <button type="button" @click="loadReport">加载报表模块</button>

    <ul v-if="features.length > 0">
      <li v-for="feature in features" :key="feature">{{ feature }}</li>
    </ul>
  </section>
</template>

<style scoped>
.deploy-info {
  max-width: 560px;
  margin: 40px auto;
  padding: 24px;
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  background: #ffffff;
  font-family: system-ui, sans-serif;
}

header {
  display: flex;
  align-items: center;
  gap: 14px;
}

.logo {
  width: 44px;
  height: 44px;
}

h1 {
  margin: 0 0 4px;
  font-size: 20px;
}

header p {
  margin: 0;
  color: #6b7280;
  font-size: 13px;
}

dl {
  margin: 20px 0;
  padding: 14px;
  border-radius: 10px;
  background: #f9fafb;
}

dl div {
  display: flex;
  justify-content: space-between;
  padding: 5px 0;
  font-size: 14px;
}

dt {
  color: #6b7280;
}

dd {
  margin: 0;
  font-family: monospace;
}

button {
  padding: 9px 16px;
  border: 0;
  border-radius: 8px;
  background: #111827;
  color: #ffffff;
  cursor: pointer;
}

ul {
  margin: 14px 0 0;
  padding-left: 18px;
  color: #374151;
  font-size: 14px;
}
</style>
```

常用 `vite.config.js` 生产配置：

```js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  base: '/admin/',
  plugins: [vue()],
  build: {
    outDir: 'dist',
    sourcemap: false,
    chunkSizeWarningLimit: 800,
    rollupOptions: {
      output: {
        manualChunks: {
          vue: ['vue', 'vue-router', 'pinia'],
          charts: ['echarts']
        }
      }
    }
  }
})
```

---

## 四、常见坑 / 注意事项

1. **环境变量必须以 `VITE_` 开头**：`API_URL` 不会出现在客户端，`VITE_API_URL` 才会被注入。
2. **环境变量会被打包进客户端代码**：不要在里面放数据库密码、私钥等敏感信息。
3. **`server.proxy` 只在开发环境生效**：生产环境需要 Nginx、网关或后端 CORS，不能指望 Vite 代理。
4. **`base` 配错会导致资源 404**：部署到子路径时必须设置 `base`，并确保路由的 history 模式也配置了对应的 fallback。
5. **别名在 TypeScript 中也要配置**：`tsconfig.json` 的 `compilerOptions.paths` 要与 Vite 的 `resolve.alias` 保持一致，否则编辑器会报找不到模块。
6. **CommonJS 依赖可能需要预构建**：某些老包在 ESM 下行为异常，可通过 `optimizeDeps.include` 强制预构建。
7. **动态 `import()` 的路径要静态可分析**：变量拼接的路径无法被正确分包，使用 `import.meta.glob` 或完整静态路径。
8. **生产构建后要本地预览**：`npm run build && npm run preview` 能发现 base、路由 fallback、资源路径等问题。

---

## 五、官方文档

- Vue 工具链：https://cn.vuejs.org/guide/scaling-up/tooling.html
- Vite 中文文档：https://cn.vitejs.dev/
- Vite 配置参考：https://cn.vitejs.dev/config/
- 环境变量与模式：https://cn.vitejs.dev/guide/env-and-mode.html
- 构建生产版本：https://cn.vitejs.dev/guide/build.html

---

## 六、速查表

| 语法              | 作用           | 示例                                                    |
| ----------------- | -------------- | ------------------------------------------------------- |
| `defineConfig`    | 定义 Vite 配置 | `export default defineConfig({})`                       |
| `plugins`         | 注册插件       | `plugins: [vue()]`                                      |
| `resolve.alias`   | 路径别名       | `'@': fileURLToPath(new URL('./src', import.meta.url))` |
| `server.proxy`    | 开发代理       | `'/api': { target: 'http://localhost:8080' }`           |
| `import.meta.env` | 读取环境变量   | `import.meta.env.VITE_API_URL`                          |
