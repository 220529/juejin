# Qiankun 微前端源码解析：架构总览

> 本文深入分析 Qiankun 3.0 的核心架构，从 loadApp 入手，理解子应用加载的完整流程。

## 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         qiankun                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                        APIs                                │  │
│  │   registerMicroApps    loadMicroApp    start              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      loadApp                               │  │
│  │   创建沙箱 → 加载 Entry → 执行生命周期                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         ▼                    ▼                    ▼             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │  @qiankunjs │     │  @qiankunjs │     │  @qiankunjs │       │
│  │   /sandbox  │     │   /loader   │     │   /shared   │       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      single-spa                            │  │
│  │   registerApplication    mountRootParcel    start         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## loadApp 核心流程

`loadApp` 是 Qiankun 的核心函数，负责加载和初始化子应用：

```typescript
// packages/qiankun/src/core/loadApp.ts
export default async function loadApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration?: AppConfiguration,
  lifeCycles?: LifeCycles<T>,
): Promise<ParcelConfigObjectGetter> {
  const { name: appName, entry, container } = app;
  
  // 1. 创建沙箱
  if (sandbox) {
    const sandboxContainer = createSandboxContainer(appName, () => microAppDOMContainer, {
      globalContext,
      extraGlobals: {},
      fetch: enhancedFetch,
      nodeTransformer,
    });
    sandboxInstance = sandboxContainer.instance;
    global = sandboxInstance.globalThis;
    mountSandbox = (domContainer) => sandboxContainer.mount(domContainer);
    unmountSandbox = () => sandboxContainer.unmount();
  }

  // 2. 加载 HTML Entry
  const lifecyclesPromise = loadEntry<MicroAppLifeCycles>(entry, microAppDOMContainer, containerOpts);

  // 3. 执行 beforeLoad 钩子
  await execHooksChain(toArray(beforeLoad), app, global);

  // 4. 获取生命周期函数
  const lifecycles = await lifecyclesPromise;
  const { bootstrap, mount, unmount, update } = getLifecyclesFromExports(lifecycles, appName, global);

  // 5. 返回 ParcelConfig
  return (mountContainer) => ({
    name: appName,
    bootstrap,
    mount: [...bindMountHooks],
    unmount: [...bindUnmountHooks],
  });
}
```

## 流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        loadApp 流程                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     1. 初始化配置和容器         │
              │   - 解析 entry、container      │
              │   - 增强 fetch（缓存、重试）    │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     2. 创建沙箱容器            │
              │   - createSandboxContainer    │
              │   - 获取代理后的 global        │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     3. 加载 HTML Entry        │
              │   - loadEntry 流式加载        │
              │   - 解析并执行脚本            │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     4. 执行 beforeLoad        │
              │   - 执行用户钩子              │
              │   - 注入全局变量              │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     5. 获取生命周期函数        │
              │   - 从 exports 获取           │
              │   - 或从 window[appName]      │
              └───────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │     6. 返回 ParcelConfig      │
              │   - 包装 mount/unmount        │
              │   - 绑定沙箱生命周期          │
              └───────────────────────────────┘
```

## 两种加载模式

### 1. registerMicroApps（路由模式）

```typescript
// packages/qiankun/src/apis/registerMicroApps.ts
export function registerMicroApps<T extends ObjectType>(
  apps: Array<RegistrableApp<T>>, 
  lifeCycles?: LifeCycles<T>
) {
  // 过滤已注册的应用
  const unregisteredApps = apps.filter(
    (app) => !microApps.some((registeredApp) => registeredApp.name === app.name)
  );

  unregisteredApps.forEach((app) => {
    const { name, activeRule, loader = noop, props, entry, container } = app;

    // 调用 single-spa 的 registerApplication
    registerApplication({
      name,
      app: async () => {
        loader(true);
        await frameworkStartedDefer.promise;

        // 核心：调用 loadApp
        const { mount, ...otherMicroAppConfigs } = (
          await loadApp({ name, entry, container, props }, frameworkConfiguration, lifeCycles)
        )(container);

        return {
          mount: [async () => loader(true), ...toArray(mount), async () => loader(false)],
          ...otherMicroAppConfigs,
        };
      },
      activeWhen: activeRule,
      customProps: props,
    });
  });
}
```

### 2. loadMicroApp（手动模式）

```typescript
// packages/qiankun/src/apis/loadMicroApp.ts
export function loadMicroApp<T extends ObjectType>(
  app: LoadableApp<T>,
  configuration?: AppConfiguration,
  lifeCycles?: LifeCycles<T>,
): MicroApp {
  const { props, name, container } = app;
  const containerXPath = getContainerXPath(container);

  const memorizedLoadingFn = async (): Promise<ParcelConfigObject> => {
    // 缓存机制：相同容器复用配置
    if (containerXPath) {
      const parcelConfigGetterPromise = appConfigPromiseGetterMap.get(appContainerXPathKey);
      if (parcelConfigGetterPromise) {
        return wrapParcelConfigForRemount((await parcelConfigGetterPromise)(container));
      }
    }

    // 核心：调用 loadApp
    const parcelConfigObjectGetterPromise = loadApp(app, userConfiguration, lifeCycles);
    // ...
  };

  // 调用 single-spa 的 mountRootParcel
  microApp = mountRootParcel(memorizedLoadingFn, { domElement: document.createElement('div'), ...props });

  return microApp;
}
```

## 生命周期执行顺序

```typescript
// mount 阶段
mount: [
  // 1. 性能标记
  async () => performanceMark(markName),
  
  // 2. 重新挂载时重新加载 HTML
  async () => {
    if (mountTimes > 1) {
      initContainer(mountContainer, appName, opts);
      const htmlString = await getPureHTMLStringWithoutScripts(entry, enhancedFetch);
      await loadEntry({ url: entry, res: new Response(htmlString) }, mountContainer, containerOpts);
    }
  },
  
  // 3. 激活沙箱
  async () => mountSandbox(mountContainer),
  
  // 4. beforeMount 钩子
  async () => execHooksChain(toArray(beforeMount), app, global),
  
  // 5. 子应用 mount
  async (props) => mount({ ...props, container: mountContainer }),
  
  // 6. afterMount 钩子
  async () => execHooksChain(toArray(afterMount), app, global),
  
  // 7. 性能测量
  async () => performanceMeasure(measureName, markName),
  
  // 8. 挂载次数 +1
  async () => { mountTimes++; },
],

// unmount 阶段
unmount: [
  // 1. beforeUnmount 钩子
  async () => execHooksChain(toArray(beforeUnmount), app, global),
  
  // 2. 子应用 unmount
  async (props) => unmount({ ...props, container: mountContainer }),
  
  // 3. 卸载沙箱
  unmountSandbox,
  
  // 4. afterUnmount 钩子
  async () => execHooksChain(toArray(afterUnmount), app, global),
  
  // 5. 清理容器
  async () => clearContainer(mountContainer),
],
```

## 获取生命周期函数

Qiankun 支持多种方式获取子应用的生命周期：

```typescript
function getLifecyclesFromExports(
  scriptExports: MicroAppLifeCycles,
  appName: string,
  globalContext: WindowProxy,
  globalLatestSetProp?: PropertyKey,
): MicroAppLifeCycles {
  // 1. 优先从 exports 获取
  if (validateExportLifecycle(scriptExports)) {
    return scriptExports;
  }

  // 2. 从沙箱最后设置的属性获取
  if (globalLatestSetProp) {
    const lifecycles = globalContext[globalLatestSetProp];
    if (validateExportLifecycle(lifecycles)) {
      return lifecycles;
    }
  }

  // 3. 从 window[appName] 获取
  const globalVariableExports = globalContext[appName];
  if (validateExportLifecycle(globalVariableExports)) {
    return globalVariableExports;
  }

  throw new QiankunError(`lifecycle not found from ${appName}`);
}
```

## 与 single-spa 的关系

Qiankun 是 single-spa 的上层封装：

| 功能 | single-spa | qiankun |
|-----|-----------|---------|
| 应用注册 | `registerApplication` | `registerMicroApps` |
| 手动加载 | `mountRootParcel` | `loadMicroApp` |
| 启动 | `start` | `start` |
| 沙箱 | ❌ | ✅ Proxy 沙箱 |
| HTML Entry | ❌ | ✅ 自动解析 |
| 样式隔离 | ❌ | ✅ |
| 预加载 | ❌ | ✅ |

## 小结

Qiankun 3.0 的核心架构：

1. **loadApp** 是核心，负责创建沙箱、加载资源、管理生命周期
2. **两种模式**：路由模式（registerMicroApps）和手动模式（loadMicroApp）
3. **模块化设计**：sandbox、loader、shared 独立包
4. **基于 single-spa**：复用其路由和生命周期管理

---

> 下一篇：沙箱机制
