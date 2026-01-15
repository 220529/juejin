# 环境变量管理优化方案

## 一、当前问题

### 现状：

```bash
# 当前只有一个 .env 文件
.env  # 包含所有环境的配置

# 内容：
VITE_LOCAL_API_BASE_URL=http://localhost:36003
VITE_DEV_API_BASE_URL=https://zs-test.ttjjerp.com
VITE_PROD_API_BASE_URL=https://zs.ttjjerp.com
VITE_STAGING_API_BASE_URL=https://zs-staging.ttjjerp.com
```

### 问题：

1. ❌ **配置混乱**：所有环境的配置混在一起
2. ❌ **容易出错**：开发时可能误用生产环境配置
3. ❌ **不够灵活**：无法针对不同环境做差异化配置
4. ❌ **安全隐患**：敏感配置（如密钥）可能被提交到代码库

---

## 二、改进方案

### 方案：多文件分离 + 本地覆盖

```bash
项目根目录/
├── .env                    # 通用配置（所有环境共享）
├── .env.development        # 开发环境专用
├── .env.production         # 生产环境专用
├── .env.local              # 本地覆盖（不提交到 Git）
└── .gitignore              # 忽略 .env.local
```

---

## 三、具体实施

### 1. 创建环境配置文件

#### `.env` - 通用配置

```bash
# ============================================
# 通用配置（所有环境共享）
# ============================================

# 应用信息
VITE_APP_NAME=CPM微信小程序
VITE_APP_VERSION=1.0.4

# 租户信息
VITE_TENANT_ID=1
VITE_LOGIN_USER_TYPE=3

# 第三方服务
VITE_TENCENT_MAP_KEY=your_tencent_map_key_here

# 功能开关
VITE_ENABLE_DEBUG=false
VITE_ENABLE_MOCK=false
```

---

#### `.env.development` - 开发环境

```bash
# ============================================
# 开发环境配置
# ============================================

# 环境标识
NODE_ENV=development
VITE_MODE=development

# API 地址（开发环境默认使用测试服务器）
VITE_DEFAULT_ENV=dev
VITE_API_BASE_URL=https://zs-test.ttjjerp.com

# 功能开关
VITE_ENABLE_DEBUG=true
VITE_ENABLE_MOCK=false
VITE_ENABLE_VCONSOLE=true

# 日志级别
VITE_LOG_LEVEL=debug

# 其他开发配置
VITE_REQUEST_TIMEOUT=30000
VITE_ENABLE_SOURCE_MAP=true
```

---

#### `.env.production` - 生产环境

```bash
# ============================================
# 生产环境配置
# ============================================

# 环境标识
NODE_ENV=production
VITE_MODE=production

# API 地址（生产环境）
VITE_DEFAULT_ENV=prod
VITE_API_BASE_URL=https://zs.ttjjerp.com

# 功能开关（生产环境关闭调试）
VITE_ENABLE_DEBUG=false
VITE_ENABLE_MOCK=false
VITE_ENABLE_VCONSOLE=false

# 日志级别
VITE_LOG_LEVEL=error

# 其他生产配置
VITE_REQUEST_TIMEOUT=10000
VITE_ENABLE_SOURCE_MAP=false
```

---

#### `.env.local` - 本地覆盖（不提交）

```bash
# ============================================
# 本地开发配置（覆盖其他配置）
# 此文件不会被提交到 Git
# ============================================

# 本地 API 地址
VITE_DEFAULT_ENV=local
VITE_API_BASE_URL=http://localhost:36003

# 本地调试
VITE_ENABLE_DEBUG=true
VITE_ENABLE_MOCK=true

# 个人配置
VITE_DEV_NAME=张三
VITE_DEV_EMAIL=zhangsan@example.com
```

---

### 2. 更新 `.gitignore`

```bash
# .gitignore

# 本地环境配置（不提交）
.env.local
.env.*.local

# 敏感信息
.env.secret
```

---

### 3. 创建环境配置模板

#### `.env.local.example` - 本地配置模板

```bash
# ============================================
# 本地开发配置模板
# 复制此文件为 .env.local 并修改配置
# ============================================

# 本地 API 地址（根据你的后端服务修改）
VITE_DEFAULT_ENV=local
VITE_API_BASE_URL=http://localhost:36003

# 调试开关
VITE_ENABLE_DEBUG=true
VITE_ENABLE_MOCK=false

# 个人信息（可选）
VITE_DEV_NAME=你的名字
VITE_DEV_EMAIL=your.email@example.com
```

---

### 4. 更新 `package.json` 脚本

```json
{
  "scripts": {
    // ============ 开发环境 ============
    "dev:wx": "uni -p mp-weixin --mode development",
    "dev:h5": "uni --mode development",
    
    // ============ 生产环境 ============
    "build:wx": "uni build -p mp-weixin --mode production",
    "build:h5": "uni build --mode production",
    
    // ============ 本地联调 ============
    "dev:local": "uni -p mp-weixin --mode development",
    // 注意：本地联调通过 .env.local 文件配置，不需要单独命令
    
    // ============ 预发环境（可选）============
    "build:staging": "uni build -p mp-weixin --mode staging",
    
    // ============ 其他 ============
    "lint": "eslint src --fix",
    "format": "prettier --write \"src/**/*.{js,vue,css,scss,json,md}\"",
    "commit": "git-cz"
  }
}
```

---

### 5. 创建预发环境配置（可选）

#### `.env.staging` - 预发环境

```bash
# ============================================
# 预发环境配置
# ============================================

NODE_ENV=production
VITE_MODE=staging

VITE_DEFAULT_ENV=staging
VITE_API_BASE_URL=https://zs-staging.ttjjerp.com

VITE_ENABLE_DEBUG=true
VITE_ENABLE_VCONSOLE=true
VITE_LOG_LEVEL=info
```

---

### 6. 更新 `vite.config.js`

```javascript
import { defineConfig, loadEnv } from 'vite'
import uni from '@dcloudio/vite-plugin-uni'
import AutoImport from 'unplugin-auto-import/vite'
import VueSetupExtend from 'vite-plugin-vue-setup-extend'
import path from 'path'

export default defineConfig(({ mode }) => {
  const root = process.cwd()
  
  // ✅ 加载环境变量（会自动合并 .env、.env.[mode]、.env.local）
  const env = loadEnv(mode, root)
  
  console.log('==========================================')
  console.log(`🚀 构建模式: ${mode}`)
  console.log(`🌍 默认环境: ${env.VITE_DEFAULT_ENV}`)
  console.log(`📡 API地址: ${env.VITE_API_BASE_URL}`)
  console.log(`🐛 调试模式: ${env.VITE_ENABLE_DEBUG}`)
  console.log('==========================================')
  
  return {
    plugins: [
      uni(),
      AutoImport({
        imports: ['vue', 'uni-app'],
      }),
      VueSetupExtend(),
    ],

    resolve: {
      alias: {
        '@': path.resolve(__dirname, 'src'),
        'dayjs/esm/index': 'dayjs',
      },
    },

    optimizeDeps: {
      include: ['dayjs'],
    },

    css: {
      preprocessorOptions: {
        scss: {
          additionalData: `@import "@/styles/variables.scss";`,
        },
      },
    },

    build: {
      minify: 'esbuild',
      esbuild: {
        // ✅ 生产环境移除 console 和 debugger
        drop: mode === 'production' ? ['console', 'debugger'] : [],
      },
      // ✅ 根据环境决定是否生成 sourcemap
      sourcemap: env.VITE_ENABLE_SOURCE_MAP === 'true',
    },
    
    // ✅ 定义全局常量（可在代码中直接使用）
    define: {
      __APP_VERSION__: JSON.stringify(env.VITE_APP_VERSION),
      __BUILD_TIME__: JSON.stringify(new Date().toISOString()),
    },
  }
})
```

---

### 7. 更新 `src/utils/env.js`

```javascript
// src/utils/env.js

/**
 * 获取环境变量（支持多文件配置）
 */
export function getEnvConfig() {
  return {
    // 应用信息
    APP_NAME: import.meta.env.VITE_APP_NAME,
    APP_VERSION: import.meta.env.VITE_APP_VERSION,
    
    // API 配置
    API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
    DEFAULT_ENV: import.meta.env.VITE_DEFAULT_ENV,
    
    // 租户信息
    TENANT_ID: import.meta.env.VITE_TENANT_ID,
    LOGIN_USER_TYPE: import.meta.env.VITE_LOGIN_USER_TYPE,
    
    // 第三方服务
    TENCENT_MAP_KEY: import.meta.env.VITE_TENCENT_MAP_KEY,
    
    // 功能开关
    ENABLE_DEBUG: import.meta.env.VITE_ENABLE_DEBUG === 'true',
    ENABLE_MOCK: import.meta.env.VITE_ENABLE_MOCK === 'true',
    ENABLE_VCONSOLE: import.meta.env.VITE_ENABLE_VCONSOLE === 'true',
    
    // 其他配置
    LOG_LEVEL: import.meta.env.VITE_LOG_LEVEL || 'info',
    REQUEST_TIMEOUT: Number(import.meta.env.VITE_REQUEST_TIMEOUT) || 10000,
    
    // 构建信息
    MODE: import.meta.env.MODE,
    DEV: import.meta.env.DEV,
    PROD: import.meta.env.PROD,
  }
}

// 导出常用配置
export const ENV_CONFIG = getEnvConfig()

// 便捷方法
export const isDev = import.meta.env.DEV
export const isProd = import.meta.env.PROD
export const isDebug = import.meta.env.VITE_ENABLE_DEBUG === 'true'

// 打印环境信息
console.log('==========================================')
console.log('📦 环境配置信息')
console.log('==========================================')
console.log('应用名称:', ENV_CONFIG.APP_NAME)
console.log('应用版本:', ENV_CONFIG.APP_VERSION)
console.log('构建模式:', ENV_CONFIG.MODE)
console.log('默认环境:', ENV_CONFIG.DEFAULT_ENV)
console.log('API地址:', ENV_CONFIG.API_BASE_URL)
console.log('调试模式:', ENV_CONFIG.ENABLE_DEBUG)
console.log('==========================================')
```

---

## 四、使用示例

### 1. 开发环境（使用测试服务器）

```bash
# 使用 .env + .env.development
pnpm run dev:wx

# 自动加载：
# - .env（通用配置）
# - .env.development（开发配置）
# - .env.local（如果存在，覆盖上面的配置）
```

---

### 2. 本地联调（使用本地服务器）

```bash
# 1. 复制模板
cp .env.local.example .env.local

# 2. 修改 .env.local
VITE_API_BASE_URL=http://localhost:36003

# 3. 启动开发
pnpm run dev:wx

# 自动加载：
# - .env（通用配置）
# - .env.development（开发配置）
# - .env.local（本地配置，优先级最高）✅
```

---

### 3. 生产构建

```bash
# 使用 .env + .env.production
pnpm run build:wx

# 自动加载：
# - .env（通用配置）
# - .env.production（生产配置）
```

---

### 4. 在代码中使用

```javascript
// 方式 1：直接使用
const apiUrl = import.meta.env.VITE_API_BASE_URL
const isDebug = import.meta.env.VITE_ENABLE_DEBUG === 'true'

// 方式 2：使用封装的配置
import { ENV_CONFIG, isDev, isDebug } from '@/utils/env.js'

console.log('API地址:', ENV_CONFIG.API_BASE_URL)

if (isDebug) {
  console.log('调试模式已开启')
}

// 方式 3：条件编译（Vite 会在构建时移除未使用的代码）
if (import.meta.env.DEV) {
  // 只在开发环境执行
  console.log('开发环境')
}

if (import.meta.env.PROD) {
  // 只在生产环境执行
  console.log('生产环境')
}
```

---

## 五、环境变量优先级

```
.env.local（最高优先级，不提交）
    ↓
.env.[mode]（如 .env.development）
    ↓
.env（最低优先级，通用配置）
```

**示例：**

```bash
# .env
VITE_API_BASE_URL=https://default.com

# .env.development
VITE_API_BASE_URL=https://dev.com

# .env.local
VITE_API_BASE_URL=http://localhost:3000

# 最终使用：http://localhost:3000 ✅
```

---

## 六、团队协作指南

### 新成员加入项目：

```bash
# 1. 克隆项目
git clone xxx

# 2. 安装依赖
pnpm install

# 3. 复制本地配置模板
cp .env.local.example .env.local

# 4. 修改 .env.local（根据个人环境）
# 编辑 VITE_API_BASE_URL 等配置

# 5. 启动开发
pnpm run dev:wx
```

---

### 添加新的环境变量：

```bash
# 1. 在 .env 中添加通用配置
VITE_NEW_CONFIG=default_value

# 2. 在 .env.development 中添加开发配置（如果需要）
VITE_NEW_CONFIG=dev_value

# 3. 在 .env.production 中添加生产配置（如果需要）
VITE_NEW_CONFIG=prod_value

# 4. 更新 .env.local.example 模板
VITE_NEW_CONFIG=local_value

# 5. 在代码中使用
const newConfig = import.meta.env.VITE_NEW_CONFIG
```

---

## 七、安全最佳实践

### 1. 敏感信息管理

```bash
# .env.secret（不提交到 Git）
VITE_SECRET_KEY=your_secret_key
VITE_API_TOKEN=your_api_token
VITE_ENCRYPTION_KEY=your_encryption_key

# .gitignore
.env.secret
.env.*.local
```

---

### 2. 环境变量命名规范

```bash
# ✅ 好的命名
VITE_API_BASE_URL          # 清晰明确
VITE_ENABLE_DEBUG          # 布尔值用 ENABLE_
VITE_MAX_UPLOAD_SIZE       # 数值用 MAX_/MIN_

# ❌ 不好的命名
VITE_URL                   # 太模糊
VITE_DEBUG                 # 不清楚是什么类型
VITE_SIZE                  # 不知道是什么的大小
```

---

### 3. 类型转换

```javascript
// src/utils/env.js

/**
 * 获取布尔类型环境变量
 */
export function getEnvBoolean(key, defaultValue = false) {
  const value = import.meta.env[key]
  if (value === undefined) return defaultValue
  return value === 'true' || value === '1'
}

/**
 * 获取数字类型环境变量
 */
export function getEnvNumber(key, defaultValue = 0) {
  const value = import.meta.env[key]
  if (value === undefined) return defaultValue
  const num = Number(value)
  return isNaN(num) ? defaultValue : num
}

// 使用
const isDebug = getEnvBoolean('VITE_ENABLE_DEBUG')
const timeout = getEnvNumber('VITE_REQUEST_TIMEOUT', 10000)
```

---

## 八、优势总结

### ✅ 改进后的优势：

1. **配置清晰**：不同环境的配置分离，一目了然
2. **安全性高**：敏感配置不提交到代码库
3. **灵活性强**：每个环境可以有不同的配置
4. **团队友好**：新成员通过模板快速配置
5. **易于维护**：修改配置不影响其他环境

### 📊 对比：

| 维度 | 改进前 | 改进后 |
|------|--------|--------|
| 配置文件 | 1 个 | 4+ 个（分离） |
| 环境切换 | 手动修改代码 | 自动加载 |
| 本地配置 | 提交到 Git | 不提交（.env.local） |
| 团队协作 | 容易冲突 | 各自配置 |
| 安全性 | 低 | 高 |
| 可维护性 | 低 | 高 |

---

## 九、迁移步骤

### 从当前配置迁移到新配置：

```bash
# 1. 备份当前 .env
cp .env .env.backup

# 2. 创建新的配置文件
touch .env.development
touch .env.production
touch .env.local.example

# 3. 分离配置
# - 通用配置 → .env
# - 开发配置 → .env.development
# - 生产配置 → .env.production

# 4. 更新 .gitignore
echo ".env.local" >> .gitignore
echo ".env.*.local" >> .gitignore

# 5. 测试
pnpm run dev:wx
pnpm run build:wx

# 6. 提交代码
git add .env .env.development .env.production .env.local.example .gitignore
git commit -m "refactor: 优化环境变量管理"
```

---

## 十、常见问题

### Q1: 为什么 .env.local 不生效？

**A:** 确保文件名正确，且 Vite 会自动加载。重启开发服务器。

---

### Q2: 如何在不同环境使用不同的 API 地址？

**A:** 
```bash
# .env.development
VITE_API_BASE_URL=https://dev-api.com

# .env.production
VITE_API_BASE_URL=https://api.com

# .env.local（本地联调）
VITE_API_BASE_URL=http://localhost:3000
```

---

### Q3: 如何添加敏感配置？

**A:**
```bash
# 1. 创建 .env.secret（不提交）
VITE_SECRET_KEY=xxx

# 2. 添加到 .gitignore
echo ".env.secret" >> .gitignore

# 3. 在代码中使用
const secretKey = import.meta.env.VITE_SECRET_KEY
```

---

**改进完成！** 🎉
