# 无界微前端源码解析：架构总览

> 深入 startApp 启动流程，理解无界如何将 iframe 和 Shadow DOM 结合实现微前端。

## 核心架构

无界的核心思想是将 JS 和 CSS 隔离分离：

```
┌─────────────────────────────────────────────────────────┐
│                      主应用                              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              wujie-app (Web Component)          │   │
│  │  ┌───────────────────────────────────────────┐  │   │
│  │  │           Shadow DOM                      │  │   │
│  │  │                                           │  │   │
│  │  │   子应用 HTML/CSS 渲染                     │  │   │
│  │  │   样式完全隔离                             │  │   │
│  │  │                                           │  │   │
│  │  └───────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              隐藏 iframe                         │   │
│  │                                                 │   │
│  │   子应用 JS 运行                                │   │
│  │   独立 window/document/location                 │   │
│  │   document 操作代理到 Shadow DOM                │   │
│  │                                                 │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## startApp 流程

```typescript
// packages/wujie-core/src/index.ts
export async function startApp(startOptions: startOptions): Promise<Function | void> {
  const sandbox = getWujieById(startOptions.name);
  const cacheOptions = getOptionsById(startOptions.name);
  const options = mergeOptions(startOptions, cacheOptions);
  
  // 1. 已存在沙箱 - 快速激活
  if (sandbox) {
    sandbox.plugins = getPlugins(plugins);
    sandbox.lifecycles = lifecycles;
    
    if (alive) {
      // 保活模式：直接激活
      await sandbox.active({ url, sync, prefix, el, props, alive, fetch, replace });
      if (!sandbox.execFlag) {
        // 预加载但未执行
        const { getExternalScripts } = await importHTML({ url, html, opts });
        await sandbox.start(getExternalScripts);
      }
      return () => sandbox.destroy();
    } else if (isFunction(iframeWindow.__WUJIE_MOUNT)) {
      // 有 mount 函数：重新挂载
      await sandbox.unmount();
      await sandbox.active({ url, sync, prefix, el, props, alive, fetch, replace });
      sandbox.rebuildStyleSheets();
      iframeWindow.__WUJIE_MOUNT();
      return () => sandbox.destroy();
    } else {
      // 无 mount 函数：销毁重建
      await sandbox.destroy();
    }
  }

  // 2. 创建新沙箱
  addLoading(el, loading);
  const newSandbox = new WuJie({ name, url, attrs, degradeAttrs, fiber, degrade, plugins, lifecycles });
  
  // 3. 加载资源
  newSandbox.lifecycles?.beforeLoad?.(newSandbox.iframe.contentWindow);
  const { template, getExternalScripts, getExternalStyleSheets } = await importHTML({ url, html, opts });

  // 4. 处理 CSS
  const processedHtml = await processCssLoader(newSandbox, template, getExternalStyleSheets);
  
  // 5. 激活沙箱
  await newSandbox.active({ url, sync, prefix, template: processedHtml, el, props, alive, fetch, replace });
  
  // 6. 执行 JS
  await newSandbox.start(getExternalScripts);
  
  return () => newSandbox.destroy();
}
```

## WuJie 沙箱类

```typescript
// packages/wujie-core/src/sandbox.ts
export default class Wujie {
  public id: string;              // 唯一标识
  public url: string;             // 子应用地址
  public alive: boolean;          // 保活模式
  public proxy: WindowProxy;      // window 代理
  public proxyDocument: Object;   // document 代理
  public proxyLocation: Object;   // location 代理
  public iframe: HTMLIFrameElement;  // JS 沙箱
  public shadowRoot: ShadowRoot;     // CSS 沙箱
  public bus: EventBus;           // 事件总线
  
  constructor(options) {
    const { name, url, attrs, fiber, degradeAttrs, degrade, lifecycles, plugins } = options;
    
    // 1. 初始化基础属性
    this.id = name;
    this.fiber = fiber;
    this.degrade = degrade || !wujieSupport;
    this.bus = new EventBus(this.id);
    
    // 2. 解析 URL
    const { urlElement, appHostPath, appRoutePath } = appRouteParse(url);
    const { mainHostPath } = this.inject;
    
    // 3. 创建 iframe 沙箱
    this.iframe = iframeGenerator(this, attrs, mainHostPath, appHostPath, appRoutePath);
    
    // 4. 创建代理
    if (this.degrade) {
      // 降级模式：简单代理
      const { proxyDocument, proxyLocation } = localGenerator(this.iframe, urlElement, mainHostPath, appHostPath);
      this.proxyDocument = proxyDocument;
      this.proxyLocation = proxyLocation;
    } else {
      // 正常模式：Proxy 代理
      const { proxyWindow, proxyDocument, proxyLocation } = proxyGenerator(
        this.iframe, urlElement, mainHostPath, appHostPath
      );
      this.proxy = proxyWindow;
      this.proxyDocument = proxyDocument;
      this.proxyLocation = proxyLocation;
    }
    
    // 5. 注册沙箱
    addSandboxCacheWithWujie(this.id, this);
  }
}
```

## 激活流程 (active)

```typescript
// packages/wujie-core/src/sandbox.ts
public async active(options): Promise<void> {
  const { sync, url, el, template, props, alive, prefix, fetch, replace } = options;
  
  // 1. 更新配置
  this.url = url;
  this.sync = sync;
  this.alive = alive;
  this.activeFlag = true;
  
  // 2. 等待 iframe 初始化
  await this.iframeReady;
  
  // 3. 处理自定义 fetch
  const iframeWindow = this.iframe.contentWindow;
  if (fetch) {
    iframeWindow.fetch = fetch;
    this.fetch = fetch;
  }
  
  // 4. 路由同步
  if (this.execFlag && this.alive) {
    syncUrlToWindow(iframeWindow);  // 保活模式：子 -> 主
  } else {
    syncUrlToIframe(iframeWindow);  // 先同步到 iframe
    syncUrlToWindow(iframeWindow);  // 再同步到主应用
  }
  
  // 5. 降级处理
  if (this.degrade) {
    const { iframe, container } = initRenderIframeAndContainer(this.id, el, this.degradeAttrs);
    this.el = container;
    await renderTemplateToIframe(iframe.contentDocument, this.iframe.contentWindow, this.template);
    this.document = iframe.contentDocument;
    return;
  }
  
  // 6. 正常模式：创建 Web Component
  if (this.shadowRoot) {
    this.el = renderElementToContainer(this.shadowRoot.host, el);
    if (this.alive) return;
  } else {
    this.el = renderElementToContainer(createWujieWebComponent(this.id), el);
  }
  
  // 7. 渲染到 Shadow DOM
  await renderTemplateToShadowRoot(this.shadowRoot, iframeWindow, this.template);
  this.patchCssRules();
  this.provide.shadowRoot = this.shadowRoot;
}
```

## 启动流程 (start)

```typescript
// packages/wujie-core/src/sandbox.ts
public async start(getExternalScripts): Promise<void> {
  this.execFlag = true;
  const scriptResultList = await getExternalScripts();
  if (!this.iframe) return;
  
  const iframeWindow = this.iframe.contentWindow;
  iframeWindow.__POWERED_BY_WUJIE__ = true;
  
  // 1. 分类脚本
  const syncScriptResultList = [];   // 同步脚本
  const asyncScriptResultList = [];  // 异步脚本
  const deferScriptResultList = [];  // defer 脚本
  
  scriptResultList.forEach((scriptResult) => {
    if (scriptResult.defer) deferScriptResultList.push(scriptResult);
    else if (scriptResult.async) asyncScriptResultList.push(scriptResult);
    else syncScriptResultList.push(scriptResult);
  });
  
  // 2. 构建执行队列
  // 前置脚本
  beforeScriptResultList.forEach((script) => {
    this.execQueue.push(() => insertScriptToIframe(script, iframeWindow));
  });
  
  // 同步 + defer 脚本
  syncScriptResultList.concat(deferScriptResultList).forEach((scriptResult) => {
    this.execQueue.push(() =>
      scriptResult.contentPromise.then((content) =>
        insertScriptToIframe({ ...scriptResult, content }, iframeWindow)
      )
    );
  });
  
  // 异步脚本（不入队列）
  asyncScriptResultList.forEach((scriptResult) => {
    scriptResult.contentPromise.then((content) => {
      insertScriptToIframe({ ...scriptResult, content }, iframeWindow);
    });
  });
  
  // 3. mount 调用
  this.execQueue.push(() => this.mount());
  
  // 4. 触发 DOMContentLoaded
  this.execQueue.push(() => {
    eventTrigger(iframeWindow.document, "DOMContentLoaded");
    eventTrigger(iframeWindow, "DOMContentLoaded");
    this.execQueue.shift()?.();
  });
  
  // 5. 触发 load
  this.execQueue.push(() => {
    eventTrigger(iframeWindow.document, "readystatechange");
    eventTrigger(iframeWindow, "load");
    this.execQueue.shift()?.();
  });
  
  // 6. 开始执行
  this.execQueue.shift()();
}
```

## 生命周期

```
startApp 调用
    │
    ▼
┌─────────────────┐
│   beforeLoad    │  ← 加载资源前
└────────┬────────┘
         │
    importHTML (加载 HTML/CSS/JS)
         │
         ▼
┌─────────────────┐
│     active      │  ← 激活沙箱，渲染 DOM
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│     start       │  ← 执行 JS
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   beforeMount   │  ← mount 前
└────────┬────────┘
         │
    __WUJIE_MOUNT()
         │
         ▼
┌─────────────────┐
│   afterMount    │  ← mount 后
└────────┬────────┘
         │
    (应用运行中...)
         │
         ▼
┌─────────────────┐
│  beforeUnmount  │  ← unmount 前
└────────┬────────┘
         │
    __WUJIE_UNMOUNT()
         │
         ▼
┌─────────────────┐
│  afterUnmount   │  ← unmount 后
└─────────────────┘
```

## 保活模式 vs 重建模式

| 特性 | 保活模式 (alive: true) | 重建模式 (alive: false) |
|-----|----------------------|------------------------|
| 状态保留 | ✅ 保留 | ❌ 重置 |
| DOM 保留 | ✅ 保留 | ❌ 重建 |
| 切换速度 | 快 | 慢 |
| 内存占用 | 高 | 低 |
| 适用场景 | 频繁切换 | 偶尔访问 |

## 降级模式

当浏览器不支持 Web Components 时，无界会降级到纯 iframe 方案：

```typescript
// 判断是否支持
export const wujieSupport = window.Proxy && window.CustomElementRegistry;

// 降级处理
if (this.degrade) {
  // 使用 iframe 渲染 DOM（而非 Shadow DOM）
  const { iframe, container } = initRenderIframeAndContainer(this.id, el, this.degradeAttrs);
  await renderTemplateToIframe(iframe.contentDocument, this.iframe.contentWindow, this.template);
  this.document = iframe.contentDocument;
}
```

## 小结

无界的架构设计：

1. **双容器分离**：iframe 运行 JS，Shadow DOM 渲染 CSS
2. **代理桥接**：通过 Proxy 将 iframe 中的 DOM 操作代理到 Shadow DOM
3. **执行队列**：保证脚本按序执行，正确触发生命周期事件
4. **优雅降级**：不支持 Web Components 时回退到纯 iframe

下一篇我们将深入分析 iframe 沙箱的创建和 patch 机制。

---

> 📦 源码版本：wujie v1.0.22
>
> 上一篇：项目介绍
>
> 下一篇：沙箱机制
