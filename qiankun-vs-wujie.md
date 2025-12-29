# Qiankun vs Wujie：微前端框架深度对比

> 基于 Qiankun 3.0 和 Wujie 1.0.22 源码深度分析，从架构设计、隔离方案、资源加载、通信机制、路由处理、预加载策略、插件系统等维度进行全面对比，助你做出最佳技术选型。

## 一、设计哲学

### 1.1 架构理念

| 维度 | Qiankun | Wujie |
|-----|---------|-------|
| 基础架构 | 基于 single-spa 封装 | 原创双容器架构 |
| 隔离思路 | Proxy 代理模拟隔离 | 浏览器原生隔离 |
| 核心技术 | Proxy + with + Membrane | iframe + Shadow DOM + Proxy |
| 设计目标 | 开箱即用、生态完善 | 极致隔离、低侵入 |
| 包结构 | monorepo（sandbox/loader/shared） | 单包 |

### 1.2 架构图对比

```
┌─────────────────────────────────────────────────────────────────┐
│                        Qiankun 架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    主应用 Window                         │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│   │  │  子应用 A    │  │  子应用 B    │  │  子应用 C    │      │   │
│   │  │             │  │             │  │             │      │   │
│   │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │      │   │
│   │  │ │ Proxy   │ │  │ │ Proxy   │ │  │ │ Proxy   │ │      │   │
│   │  │ │ Sandbox │ │  │ │ Sandbox │ │  │ │ Sandbox │ │      │   │
│   │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │      │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│                        single-spa                                │
│                  (路由劫持 + 生命周期管理)                         │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        Wujie 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  wujie-app (Web Component)                               │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │           Shadow DOM ← CSS 完全隔离                │  │   │
│   │  │           子应用 HTML/CSS 渲染                      │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              ↑ Proxy 代理                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  隐藏 iframe ← JS 原生隔离                               │   │
│   │  ┌───────────────────────────────────────────────────┐  │   │
│   │  │  子应用 JS 运行                                     │  │   │
│   │  │  独立 window / document / location                 │  │   │
│   │  │  DOM 操作通过 Proxy 代理到 Shadow DOM               │  │   │
│   │  └───────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## 二、JS 沙箱机制

### 2.1 Qiankun：Proxy + Membrane + Compartment

Qiankun 3.0 采用三层架构实现 JS 隔离：

```typescript
// 1. Membrane 层：Proxy 代理拦截属性访问
const handler: ProxyHandler<Window> = {
  get(target, p) {
    if (p in endowments) return endowments[p].value;
    if (modifications.has(p)) return modifications.get(p);
    return Reflect.get(target, p);
  },
  set(target, p, value) {
    if (locked) return false;  // 沙箱锁定时禁止修改
    modifications.set(p, value);
    return true;
  },
};

// 2. Compartment 层：with 语句绑定作用域
;(function(){
  with(this){
    const {console,document,...} = this;  // 优化常用变量
    // 子应用代码
  }
}).bind(proxyWindow)();

// 3. Patchers 层：副作用管理
patchWindowListener(sandbox);   // 事件监听
patchInterval(sandbox);         // 定时器
patchHistoryListener(sandbox);  // History 监听
```

**特点**：
- modifications Map 记录所有修改，支持快照恢复
- 锁定机制：inactive 时禁止修改
- 副作用补丁：手动管理事件、定时器等

### 2.2 Wujie：iframe 原生隔离 + Proxy 代理

Wujie 利用 iframe 的原生隔离能力：

```typescript
// 1. 创建同域 iframe（避免跨域）
const iframe = document.createElement("iframe");
iframe.src = mainHostPath;  // 同域
iframe.style.display = "none";

// 2. 阻止 iframe 加载主应用内容
iframeWindow.stop();

// 3. IIFE 包装脚本，替换全局变量
(function(window, self, global, location) {
  // 子应用代码
  window.foo = 'bar';  // 实际操作 proxy
}).bind(window.__WUJIE.proxy)(
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxy,
  window.__WUJIE.proxyLocation,
);

// 4. Proxy 代理 document 操作到 Shadow DOM
const proxyDocument = new Proxy({}, {
  get(_, propKey) {
    if (propKey === "getElementById") {
      return (id) => shadowRoot.querySelector(`[id="${id}"]`);
    }
    // ...
  },
});
```

**特点**：
- iframe 提供完整独立的 window 对象
- 原生 JS 隔离，无需模拟
- 事件、定时器自动隔离，无需手动清理

### 2.3 隔离能力对比

| 场景 | Qiankun | Wujie | 说明 |
|-----|---------|-------|------|
| 全局变量污染 | ✅ Proxy 拦截 | ✅ 原生隔离 | 都能处理 |
| 原型链污染 | ⚠️ 需额外处理 | ✅ 原生隔离 | Array.prototype.xxx |
| eval/new Function | ⚠️ 可能逃逸 | ✅ 原生隔离 | 动态代码执行 |
| 定时器清理 | 需 patch | 自动清理 | setInterval/setTimeout |
| 事件监听清理 | 需 patch | 自动清理 | addEventListener |
| localStorage | 共享 | 可配置隔离 | 通过插件实现 |
| Cookie | 共享 | 共享 | 同域限制 |

## 三、CSS 隔离机制

### 3.1 Qiankun：Scoped CSS / Shadow DOM

```typescript
// 方式一：Scoped CSS（默认）
// 原始
.container { color: red; }
// 转换后
div[data-qiankun="app-name"] .container { color: red; }

// 方式二：Shadow DOM（实验性）
start({ sandbox: { experimentalStyleIsolation: true } });
```

**问题**：
- Scoped CSS 可能有选择器权重问题
- Shadow DOM 下弹窗样式需要特殊处理

### 3.2 Wujie：Shadow DOM + CSS Loader

```typescript
// 1. 创建 Web Component
class WujieApp extends HTMLElement {
  connectedCallback() {
    this.attachShadow({ mode: "open" });
  }
}
customElements.define("wujie-app", WujieApp);

// 2. 渲染到 Shadow DOM
shadowRoot.innerHTML = template;

// 3. :root 转换为 :host
const cssSelectorMap = { ":root": ":host" };

// 4. @font-face 提取到外部（Shadow DOM 内字体不生效）
if (cssRule.type === CSSRule.FONT_FACE_RULE) {
  fontCssRules.push(cssRuleText);
}
shadowRoot.host.appendChild(fontStyleSheetElement);

// 5. CSS 相对路径处理
code.replace(/url\((['"]?)(.*)(\1)\)/g, (_, pre, url, post) => {
  return `url(${pre}${getAbsolutePath(url, baseUrl)}${post})`;
});
```

**特点**：
- 原生 Shadow DOM 隔离
- 自动处理 :root → :host
- @font-face 自动提取
- CSS 相对路径自动转换

### 3.3 CSS 隔离对比

| 特性 | Qiankun Scoped | Qiankun Shadow | Wujie Shadow |
|-----|---------------|----------------|--------------|
| 隔离程度 | 中 | 高 | 高 |
| 兼容性 | 好 | 一般 | 一般 |
| 弹窗样式 | ✅ 正常 | ❌ 需处理 | ❌ 需处理 |
| 全局样式泄漏 | 可能 | 不会 | 不会 |
| :root 支持 | ✅ | ❌ 需转换 | ✅ 自动转换 |
| @font-face | ✅ | ❌ 需处理 | ✅ 自动处理 |

## 四、资源加载机制

### 4.1 Qiankun：流式加载 + HTML Entry

```typescript
// 1. Fetch 增强
const enhancedFetch = makeFetchCacheable(
  makeFetchRetryable(
    makeFetchThrowable(fetch)
  )
);

// 2. 流式加载
res.body
  .pipeThrough(new TextDecoderStream())
  .pipeThrough(createTagTransformStream([
    { tag: '<head>', alt: '<qiankun-head>' },  // 避免污染主应用 head
  ]))
  .pipeTo(new WritableDOMStream(container, nodeTransformer));

// 3. 脚本沙箱包装
const wrappedCode = sandbox.makeEvaluateFactory(code, entry);

// 4. defer 脚本队列保证顺序执行
```

**特点**：
- 边下载边解析，首屏更快
- head 标签转换避免污染
- Fetch 缓存 + 重试机制
- defer 脚本顺序保证

### 4.2 Wujie：importHTML + 脚本分类

```typescript
// 1. 加载 HTML
const { template, getExternalScripts, getExternalStyleSheets } = 
  await importHTML({ url, html, opts });

// 2. 脚本分类执行
const syncScriptResultList = [];   // 同步脚本
const asyncScriptResultList = [];  // 异步脚本
const deferScriptResultList = [];  // defer 脚本

// 3. 构建执行队列
syncScriptResultList.concat(deferScriptResultList).forEach((script) => {
  this.execQueue.push(() => insertScriptToIframe(script, iframeWindow));
});

// 4. 异步脚本不入队列
asyncScriptResultList.forEach((script) => {
  script.contentPromise.then((content) => {
    insertScriptToIframe({ ...script, content }, iframeWindow);
  });
});

// 5. 触发生命周期事件
this.execQueue.push(() => eventTrigger(iframeWindow.document, "DOMContentLoaded"));
this.execQueue.push(() => eventTrigger(iframeWindow, "load"));
```

**特点**：
- 脚本分类处理（sync/async/defer）
- 执行队列保证顺序
- 正确触发 DOMContentLoaded/load 事件

## 五、通信机制

### 5.1 Qiankun：Props + 全局状态

```typescript
// 1. Props 传递
registerMicroApps([{
  name: 'app',
  props: { user, token, onLogout, utils: { request, storage } },
}]);

// 子应用接收
export async function mount(props) {
  const { user, token, onLogout, utils } = props;
}

// 2. 全局状态（2.x，3.0 已移除）
const actions = initGlobalState({ user: null });
actions.onGlobalStateChange((state, prev) => {});
actions.setGlobalState({ user: newUser });

// 3. 自定义事件
window.dispatchEvent(new CustomEvent('micro-app-event', { detail }));
```

### 5.2 Wujie：EventBus + Props

```typescript
// 1. 全局事件 Map（支持跨应用通信）
export const appEventObjMap = new Map<String, EventObj>();

// 2. EventBus 实现
class EventBus {
  $emit(event, ...args) {
    // 遍历所有应用的事件对象
    appEventObjMap.forEach((eventObj) => {
      if (eventObj[event]) {
        eventObj[event].forEach((fn) => fn(...args));
      }
    });
  }
}

// 3. 主应用使用
import { bus } from 'wujie';
bus.$emit('theme-change', { theme: 'dark' });
bus.$on('user-logout', () => router.push('/login'));

// 4. 子应用使用
window.$wujie.bus.$on('theme-change', ({ theme }) => {});
window.$wujie.bus.$emit('user-logout');

// 5. Props 传递
startApp({ props: { user, api: { getUserInfo, logout } } });
const { props } = window.$wujie;
```

### 5.3 通信对比

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| Props 传递 | ✅ | ✅ |
| 事件总线 | 需自建 | ✅ 内置 |
| 全局状态 | ✅ initGlobalState（2.x） | 需自建 |
| 跨应用通信 | 通过主应用中转 | ✅ 直接通信 |
| 嵌套应用通信 | 需处理 | ✅ 自动支持 |

## 六、路由处理

### 6.1 Qiankun：single-spa 路由劫持

```typescript
// 1. History API 劫持
window.history.pushState = function(...args) {
  const result = originalPushState.apply(this, args);
  reroute();  // 触发路由变化检查
  return result;
};

// 2. 监听事件
window.addEventListener('popstate', reroute);
window.addEventListener('hashchange', reroute);

// 3. activeRule 匹配
registerMicroApps([{
  activeRule: '/react',  // 字符串
  activeRule: ['/react', '/react-app'],  // 数组
  activeRule: (location) => location.pathname.startsWith('/react'),  // 函数
}]);

// 4. 子应用需要设置 basename
<BrowserRouter basename={window.__POWERED_BY_QIANKUN__ ? '/react' : '/'}>
```

### 6.2 Wujie：URL Query 同步

```typescript
// 1. 子应用路由存入主应用 URL
// 主应用: https://main.com/home?vue3=%2Fuser%2F123
// 子应用: /user/123

// 2. History 劫持同步
history.pushState = function(data, title, url) {
  rawHistoryPushState.call(history, data, title, mainUrl);
  syncUrlToWindow(iframeWindow);  // 同步到主应用 URL
};

// 3. 短路径优化
startApp({
  sync: true,
  prefix: { 'u': '/user' },  // /user/123 → {u}/123
});

// 4. 刷新恢复
const syncUrl = getSyncUrl(id, prefix);  // 从 URL 获取子应用路由
```

### 6.3 路由对比

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| 路由模式 | hash/history | hash/history |
| URL 同步 | 需配置 | ✅ sync 模式 |
| 路由隔离 | 共享 history | 独立 history |
| 刷新恢复 | 需处理 | ✅ 自动恢复 |
| 短路径优化 | ❌ | ✅ prefix 配置 |
| 子应用改造 | 需设置 basename | 无需改造 |

## 七、预加载策略

### 7.1 Qiankun：多策略预加载

```typescript
// 1. 配置方式
start({ prefetch: true });  // 首屏后预加载所有
start({ prefetch: 'all' });  // 立即预加载所有
start({ prefetch: ['react-app'] });  // 指定应用
start({
  prefetch: (apps) => ({
    criticalAppNames: ['react-app'],  // 关键应用立即加载
    minorAppNames: ['vue-app'],       // 次要应用空闲时加载
  }),
});

// 2. 空闲时加载
requestIdleCallback(async () => {
  const { getExternalScripts, getExternalStyleSheets } = await importEntry(entry);
  requestIdleCallback(getExternalStyleSheets);
  requestIdleCallback(getExternalScripts);
});

// 3. 基于网络状况
if (connection.effectiveType === 'slow-2g') {
  return { criticalAppNames: [], minorAppNames: [] };
}
```

### 7.2 Wujie：preloadApp

```typescript
// 1. 预加载
import { preloadApp } from 'wujie';
preloadApp({ name: 'vue3', url: 'http://localhost:7300/' });

// 2. 预执行（创建沙箱但不渲染）
preloadApp({ name: 'vue3', url: '...', exec: true });

// 3. 保活模式（状态保留）
startApp({ alive: true });  // 切换时不销毁，直接激活
```

### 7.3 预加载对比

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| 预加载资源 | ✅ | ✅ |
| 预执行 | ❌ | ✅ exec 模式 |
| 保活模式 | ❌ | ✅ alive 模式 |
| 空闲加载 | ✅ requestIdleCallback | ❌ |
| 自定义策略 | ✅ 函数配置 | ❌ |
| 网络感知 | 可实现 | ❌ |

## 八、插件/扩展系统

### 8.1 Qiankun：生命周期钩子

```typescript
registerMicroApps(apps, {
  beforeLoad: [(app) => console.log('beforeLoad', app.name)],
  beforeMount: [(app) => console.log('beforeMount', app.name)],
  afterMount: [(app) => console.log('afterMount', app.name)],
  beforeUnmount: [(app) => console.log('beforeUnmount', app.name)],
  afterUnmount: [(app) => console.log('afterUnmount', app.name)],
});
```

### 8.2 Wujie：完整插件系统

```typescript
startApp({
  plugins: [{
    // HTML 处理
    htmlLoader: (code) => code,
    
    // JS 处理
    jsExcludes: [/google-analytics/],  // 排除
    jsIgnores: [/ad\.js/],             // 忽略执行
    jsBeforeLoaders: [{ content: 'window.ENV = "prod"' }],
    jsLoader: (code, url) => code.replace(/api\.dev/g, 'api.prod'),
    jsAfterLoaders: [{ src: 'https://cdn.com/init.js' }],
    
    // CSS 处理
    cssExcludes: [],
    cssBeforeLoaders: [{ content: ':root { --theme: dark; }' }],
    cssLoader: (code, url) => code,
    cssAfterLoaders: [],
    
    // 事件钩子
    windowAddEventListenerHook: (win, type, handler) => {},
    documentAddEventListenerHook: (win, type, handler) => {},
    
    // DOM 钩子
    appendOrInsertElementHook: (element, iframeWindow) => {},
    patchElementHook: (element, iframeWindow) => {},
    
    // 属性覆盖
    windowPropertyOverride: (iframeWindow) => {
      iframeWindow.localStorage = customStorage;
    },
    documentPropertyOverride: (iframeWindow) => {},
  }],
});
```

### 8.3 扩展能力对比

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| 生命周期钩子 | ✅ | ✅ |
| JS Loader | ❌ | ✅ |
| CSS Loader | ❌ | ✅ |
| 资源排除/忽略 | ❌ | ✅ |
| 前置/后置脚本 | ❌ | ✅ |
| 事件拦截 | ❌ | ✅ |
| DOM 拦截 | ❌ | ✅ |
| 属性覆盖 | ❌ | ✅ |

## 九、生命周期

### 9.1 Qiankun 生命周期

```
beforeLoad → bootstrap → beforeMount → mount → afterMount
                              ↓
                    beforeUnmount → unmount → afterUnmount
```

子应用必须导出：
```typescript
export async function bootstrap() {}
export async function mount(props) {}
export async function unmount(props) {}
```

### 9.2 Wujie 生命周期

```
beforeLoad → active → start → beforeMount → mount → afterMount
                                    ↓
                          beforeUnmount → unmount → afterUnmount
```

子应用可选导出：
```typescript
window.__WUJIE_MOUNT = () => {};
window.__WUJIE_UNMOUNT = () => {};
```

### 9.3 生命周期对比

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| 必须导出 | ✅ 必须 | ❌ 可选 |
| 改造成本 | 中 | 低 |
| 独立运行 | 需判断 `__POWERED_BY_QIANKUN__` | 自动兼容 |
| 生命周期数量 | 5 个 | 6 个 |

## 十、性能对比

| 维度 | Qiankun | Wujie |
|-----|---------|-------|
| 首次加载 | 快 | 稍慢（创建 iframe） |
| 切换速度 | 快 | 快（保活模式更快） |
| 内存占用 | 低 | 中（iframe 开销） |
| 沙箱创建 | 快（Proxy） | 稍慢（iframe） |
| 样式隔离开销 | 低（Scoped）/ 中（Shadow） | 中（Shadow） |

## 十一、兼容性

| 特性 | Qiankun | Wujie |
|-----|---------|-------|
| IE 支持 | ❌ 需 Proxy polyfill | ❌ |
| 最低浏览器 | Chrome 49+ | Chrome 53+ |
| Web Components | 可选 | 必需（有降级） |
| Proxy | 必需 | 必需 |

## 十二、选型建议

```
┌─────────────────────────────────────────────────────────────────┐
│                        选型决策树                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   需要极致 JS 隔离？                                             │
│       ├── 是 ──────────────────────────────▶ Wujie              │
│       └── 否 ──▶ 继续判断                                        │
│                                                                  │
│   子应用改造成本敏感？                                            │
│       ├── 是 ──────────────────────────────▶ Wujie              │
│       └── 否 ──▶ 继续判断                                        │
│                                                                  │
│   需要保活模式（状态保留）？                                       │
│       ├── 是 ──────────────────────────────▶ Wujie              │
│       └── 否 ──▶ 继续判断                                        │
│                                                                  │
│   使用 umi 框架？                                                │
│       ├── 是 ──────────────────────────────▶ Qiankun            │
│       └── 否 ──▶ 继续判断                                        │
│                                                                  │
│   需要成熟生态和社区支持？                                        │
│       ├── 是 ──────────────────────────────▶ Qiankun            │
│       └── 否 ──▶ 继续判断                                        │
│                                                                  │
│   需要丰富的插件能力？                                            │
│       ├── 是 ──────────────────────────────▶ Wujie              │
│       └── 否 ──▶ 都可以                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 选择 Qiankun 的场景

- umi 项目，需要深度集成
- 团队熟悉 single-spa 生态
- 对隔离要求不极致，接受 Proxy 方案
- 需要成熟稳定的解决方案
- 子应用可以配合改造

### 选择 Wujie 的场景

- 需要极致的 JS 隔离能力
- 子应用改造成本要求低
- 需要保活模式（频繁切换场景）
- 需要丰富的插件扩展能力
- 需要路由同步到 URL

## 十三、总结

| 维度 | Qiankun | Wujie | 胜出 |
|-----|---------|-------|------|
| JS 隔离 | Proxy 模拟 | iframe 原生 | Wujie |
| CSS 隔离 | Scoped/Shadow | Shadow | 平手 |
| 改造成本 | 中 | 低 | Wujie |
| 生态成熟度 | 高 | 中 | Qiankun |
| 插件能力 | 弱 | 强 | Wujie |
| 预加载 | 丰富 | 基础 | Qiankun |
| 保活模式 | ❌ | ✅ | Wujie |
| 路由同步 | 需处理 | 内置 | Wujie |
| 内存占用 | 低 | 中 | Qiankun |
| 社区支持 | 强 | 中 | Qiankun |

**最终结论**：

- **Qiankun**：成熟稳定，生态完善，是基于 single-spa 的事实标准，适合对稳定性要求高、有 umi 技术栈的团队
- **Wujie**：隔离彻底，改造成本低，创新的双容器架构，适合对隔离要求高、需要保活模式、子应用改造受限的场景

两个框架各有千秋，选型时需要根据项目实际情况，权衡隔离能力、改造成本、团队熟悉度、生态支持等因素综合考虑。

---

> 📦 源码参考：
> - [Qiankun GitHub](https://github.com/umijs/qiankun) - 基于 v3.0 分析
> - [Wujie GitHub](https://github.com/Tencent/wujie) - 基于 v1.0.22 分析
>
> 📚 相关专栏：
> - [Qiankun 源码解析系列](./qiankun/)
> - [Wujie 源码解析系列](./wujie/)
