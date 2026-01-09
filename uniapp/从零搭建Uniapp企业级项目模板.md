# 从零搭建 Uniapp 企业级项目模板

> 一套开箱即用的 Uniapp 项目模板，包含请求封装、Token 管理、多环境切换、工程化配置等企业级功能。

## 前言

每次新建 Uniapp 项目，都要重复配置 ESLint、封装请求、处理 Token... 于是整理了一套模板，把常用功能都封装好，新项目直接用。

这篇文章分享模板的整体设计和搭建过程。

---

## 一、项目结构

```
uniapp-template/
├── src/
│   ├── api/                # 接口层
│   │   ├── core/           # 请求核心
│   │   │   ├── request.js  # 请求封装
│   │   │   └── interceptors.js
│   │   └── user.js         # 业务接口示例
│   ├── auth/               # 认证模块
│   │   └── tokenManager.js
│   ├── config/             # 配置
│   │   └── token.js
│   ├── utils/              # 工具函数
│   │   ├── env.js          # 环境管理
│   │   └── cache.js        # 页面缓存
│   ├── components/         # 通用组件
│   ├── pages/              # 页面
│   ├── store/              # 状态管理
│   ├── styles/             # 全局样式
│   │   └── variables.scss
│   ├── App.vue
│   ├── main.js
│   ├── pages.json
│   └── manifest.json
├── docs/                   # 项目文档
├── .husky/                 # Git Hooks
├── .env                    # 环境变量
├── .prettierrc
├── .cz-config.js           # 提交规范配置
├── commitlint.config.js
├── eslint.config.js
├── package.json
└── vite.config.js
```

---

## 二、工程化配置

### 2.1 ESLint + Prettier

统一代码风格，避免团队协作时的格式冲突。

```bash
pnpm add -D eslint eslint-plugin-vue eslint-config-prettier eslint-plugin-prettier prettier vue-eslint-parser
```

```javascript
// eslint.config.js
import vue from 'eslint-plugin-vue';
import prettierConfig from 'eslint-config-prettier';
import prettierPlugin from 'eslint-plugin-prettier';

export default [
  ...vue.configs['flat/strongly-recommended'],
  prettierConfig,
  {
    plugins: { prettier: prettierPlugin },
    rules: { 'prettier/prettier': 'error' },
  },
  {
    languageOptions: {
      globals: { uni: 'readonly', getCurrentPages: 'readonly' },
    },
    rules: {
      'vue/multi-word-component-names': 'off',
      'no-console': 'warn',
      'eqeqeq': ['error', 'always'],
    },
  },
];
```

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "trailingComma": "es5"
}
```

### 2.2 Husky + lint-staged

提交前自动检查代码，只检查暂存的文件。

```bash
pnpm add -D husky lint-staged
npx husky init
```

```json
// package.json
{
  "lint-staged": {
    "*.{js,vue}": ["eslint --fix", "prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
npx lint-staged
```

### 2.3 Commitizen + Commitlint

规范提交信息，支持中文提示。

```bash
pnpm add -D commitizen cz-customizable @commitlint/cli @commitlint/config-conventional
```

```javascript
// .cz-config.js
module.exports = {
  types: [
    { value: 'feat', name: '✨ feat:     新功能' },
    { value: 'fix', name: '🐛 fix:      修复bug' },
    { value: 'docs', name: '📝 docs:     文档变更' },
    { value: 'style', name: '💄 style:    代码格式' },
    { value: 'refactor', name: '♻️ refactor: 重构' },
    { value: 'perf', name: '⚡ perf:     性能优化' },
    { value: 'chore', name: '🔧 chore:    构建/工具变动' },
  ],
  messages: {
    type: '选择提交类型:',
    subject: '简短描述:',
    confirmCommit: '确认提交?',
  },
  skipQuestions: ['scope', 'body', 'breaking', 'footer'],
};
```

提交时使用 `npx cz` 或 `pnpm run commit`。

---

## 三、请求封装

### 3.1 核心功能

- 自动拼接 baseURL
- 自动注入 Token 和公共 Header
- Loading 管理（计数器，多请求不闪烁）
- 错误提示节流（相同错误 1.5s 内只提示一次）
- 请求取消（防竞态、页面卸载时取消）
- 防重复请求

### 3.2 基本用法

```javascript
import { http } from '@/api/core/request.js';

// GET
const res = await http.get('/api/user/info', { id: 123 });

// POST
const res = await http.post('/api/user/update', { name: '张三' });

// 配置选项
const res = await http.post('/api/data', data, {
  loading: false,         // 不显示 loading
  showError: false,       // 不显示错误提示
  cancelPrevious: true,   // 取消前一个相同请求（搜索场景）
});
```

### 3.3 请求取消

```javascript
import { onUnmounted } from 'vue';
import { http } from '@/api/core/request.js';

// 页面卸载时取消请求
onUnmounted(() => {
  http.cancelRequestsByUrl('/api/list');
});
```

### 3.4 业务接口组织

```javascript
// src/api/user.js
import { http } from './core/request.js';

export const userApi = {
  getInfo: (params) => http.get('/api/user/info', params),
  update: (data) => http.post('/api/user/update', data),
  search: (params) => http.get('/api/user/search', params, { cancelPrevious: true }),
};
```

---

## 四、Token 管理

### 4.1 功能特性

- 自动刷新（可配置开关）
- 并发控制（多请求同时触发刷新只执行一次）
- 请求队列（刷新期间的请求排队等待）
- 防重复跳转登录页

### 4.2 配置

```javascript
// src/config/token.js
export const AUTO_REFRESH_ENABLED = false;  // 是否启用自动刷新
export const BUFFER_TIME = 5 * 60 * 1000;   // 提前 5 分钟刷新
export const REDIRECT_DELAY = 3000;          // 防重复跳转间隔
```

### 4.3 使用

```javascript
import { tokenManager } from '@/auth/tokenManager.js';

// 保存 Token
tokenManager.saveToken({
  accessToken: 'xxx',
  refreshToken: 'yyy',
  expiresTime: Date.now() + 7200000,
});

// 检查登录状态
if (tokenManager.isLoggedIn()) {
  // 已登录
}

// 清除 Token
tokenManager.clearToken();
```

---

## 五、多环境切换

### 5.1 环境配置

```bash
# .env
VITE_LOCAL_API_BASE_URL=http://localhost:3000
VITE_DEV_API_BASE_URL=https://api-dev.example.com
VITE_PROD_API_BASE_URL=https://api.example.com
```

### 5.2 切换方式

```javascript
// 控制台执行（体验版/开发版可用）
switchEnv('dev')    // 测试环境
switchEnv('prod')   // 生产环境
switchEnv('local')  // 本地环境
```

### 5.3 安全机制

线上正式版自动锁定 prod 环境，无法切换：

```javascript
function isReleaseVersion() {
  const accountInfo = wx.getAccountInfoSync();
  return accountInfo.miniProgram?.envVersion === 'release';
}
```

---

## 六、页面缓存

解决小程序 URL 参数长度限制（约 2KB）。

```javascript
import { savePageCache, getPageCache, clearPageCache } from '@/utils/cache.js';

// 页面 A：保存数据后跳转
savePageCache('/pages/detail', { orderId: 123, items: [...] });
uni.navigateTo({ url: '/pages/detail' });

// 页面 B：获取数据
const data = getPageCache('/pages/detail');
clearPageCache('/pages/detail');  // 用完清理
```

---

## 七、样式管理

### 7.1 SCSS 变量

```scss
// src/styles/variables.scss
$primary-color: #007aff;
$text-color: #333;
$border-color: #eee;
$page-padding: 30rpx;
```

### 7.2 全局引入

```javascript
// vite.config.js
css: {
  preprocessorOptions: {
    scss: {
      additionalData: '@import "@/styles/variables.scss";',
    },
  },
}
```

---

## 八、使用模板

### 8.1 克隆项目

```bash
git clone https://github.com/220529/uniapp-template.git my-project
cd my-project
pnpm install
```

### 8.2 修改配置

1. `.env` - 修改 API 地址
2. `src/manifest.json` - 修改小程序 appid
3. `package.json` - 修改项目名称

### 8.3 启动开发

```bash
pnpm run dev:mp-weixin  # 微信小程序
pnpm run dev:h5         # H5
```

---

## 总结

这套模板包含了企业级项目常用的功能：

| 模块 | 功能 |
|------|------|
| 工程化 | ESLint + Prettier + Husky + Commitizen |
| 请求层 | 封装、拦截器、取消机制、防重复 |
| 认证 | Token 管理、自动刷新、并发控制 |
| 环境 | 多环境切换、线上锁定 |
| 工具 | 页面缓存、样式变量 |

开箱即用，新项目直接基于模板开发即可。

---

如果觉得有帮助，欢迎 Star ⭐

---

> 📦 项目地址：[github.com/220529/uniapp-template](https://github.com/220529/uniapp-template)
