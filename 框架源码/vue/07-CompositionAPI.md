# Vue.js 源码揭秘（七）：Composition API 实现

> 本文深入源码，解析 watch、watchEffect、provide/inject 等 Composition API 的实现。

## 一、Composition API 概览

```
┌─────────────────────────────────────────────────────────────┐
│                  Composition API                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  响应式                                                      │
│  ├── ref()                                                  │
│  ├── reactive()                                             │
│  ├── computed()                                             │
│  └── readonly()                                             │
│                                                             │
│  副作用                                                      │
│  ├── watchEffect()                                          │
│  ├── watch()                                                │
│  └── effectScope()                                          │
│                                                             │
│  生命周期                                                    │
│  ├── onMounted()                                            │
│  ├── onUpdated()                                            │
│  └── onUnmounted()                                          │
│                                                             │
│  依赖注入                                                    │
│  ├── provide()                                              │
│  └── inject()                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 二、watch 实现

### 2.1 watch 函数

```typescript
// packages/runtime-core/src/apiWatch.ts
export function watch<T>(
  source: WatchSource<T> | WatchSource<T>[],
  cb: WatchCallback<T>,
  options?: WatchOptions
): WatchStopHandle {
  return doWatch(source, cb, options)
}

export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchStopHandle {
  return doWatch(effect, null, options)
}
```

### 2.2 doWatch 核心

```typescript
function doWatch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb: WatchCallback | null,
  options: WatchOptions = EMPTY_OBJ
): WatchStopHandle {
  const { immediate, deep, flush, once } = options
  
  // 当前组件实例
  const instance = currentInstance
  
  // 构建 getter
  let getter: () => any
  let forceTrigger = false
  let isMultiSource = false
  
  if (isRef(source)) {
    // ref
    getter = () => source.value
    forceTrigger = isShallow(source)
  } else if (isReactive(source)) {
    // reactive 对象
    getter = () => reactiveGetter(source)
    deep = true
  } else if (isArray(source)) {
    // 数组
    isMultiSource = true
    forceTrigger = source.some(s => isReactive(s) || isShallow(s))
    getter = () => source.map(s => {
      if (isRef(s)) return s.value
      if (isReactive(s)) return reactiveGetter(s)
      if (isFunction(s)) return callWithErrorHandling(s, instance, ErrorCodes.WATCH_GETTER)
    })
  } else if (isFunction(source)) {
    if (cb) {
      // watch(getter, cb)
      getter = () => callWithErrorHandling(source, instance, ErrorCodes.WATCH_GETTER)
    } else {
      // watchEffect
      getter = () => {
        if (cleanup) {
          cleanup()
        }
        return callWithAsyncErrorHandling(
          source,
          instance,
          ErrorCodes.WATCH_CALLBACK,
          [onCleanup]
        )
      }
    }
  }
  
  // 深度监听
  if (cb && deep) {
    const baseGetter = getter
    getter = () => traverse(baseGetter())
  }
  
  // cleanup 函数
  let cleanup: (() => void) | undefined
  let onCleanup: OnCleanup = (fn: () => void) => {
    cleanup = effect.onStop = () => {
      callWithErrorHandling(fn, instance, ErrorCodes.WATCH_CLEANUP)
      cleanup = effect.onStop = undefined
    }
  }
  
  // 旧值
  let oldValue: any = isMultiSource
    ? new Array(source.length).fill(INITIAL_WATCHER_VALUE)
    : INITIAL_WATCHER_VALUE
  
  // job 函数
  const job: SchedulerJob = () => {
    if (!effect.active || !effect.dirty) return
    
    if (cb) {
      // watch(source, cb)
      const newValue = effect.run()
      
      if (
        deep ||
        forceTrigger ||
        (isMultiSource
          ? newValue.some((v, i) => hasChanged(v, oldValue[i]))
          : hasChanged(newValue, oldValue))
      ) {
        // 执行 cleanup
        if (cleanup) {
          cleanup()
        }
        
        // 执行回调
        callWithAsyncErrorHandling(cb, instance, ErrorCodes.WATCH_CALLBACK, [
          newValue,
          oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
          onCleanup
        ])
        
        oldValue = newValue
      }
    } else {
      // watchEffect
      effect.run()
    }
  }
  
  // 设置 job 属性
  job.flags |= SchedulerJobFlags.ALLOW_RECURSE
  
  // scheduler
  let scheduler: EffectScheduler
  if (flush === 'sync') {
    // 同步执行
    scheduler = job
  } else if (flush === 'post') {
    // DOM 更新后执行
    scheduler = () => queuePostRenderEffect(job, instance?.suspense)
  } else {
    // 默认：pre，DOM 更新前执行
    job.flags |= SchedulerJobFlags.PRE
    if (instance) job.id = instance.uid
    scheduler = () => queueJob(job)
  }
  
  // 创建 effect
  const effect = new ReactiveEffect(getter, NOOP, scheduler)
  
  // 初始执行
  if (cb) {
    if (immediate) {
      job()
    } else {
      oldValue = effect.run()
    }
  } else if (flush === 'post') {
    queuePostRenderEffect(effect.run.bind(effect), instance?.suspense)
  } else {
    effect.run()
  }
  
  // 返回停止函数
  const unwatch = () => {
    effect.stop()
    if (instance?.scope) {
      remove(instance.scope.effects, effect)
    }
  }
  
  return unwatch
}
```

### 2.3 traverse 深度遍历

```typescript
function traverse(value: unknown, seen?: Set<unknown>): unknown {
  if (!isObject(value) || (value as any)[ReactiveFlags.SKIP]) {
    return value
  }
  
  seen = seen || new Set()
  if (seen.has(value)) {
    return value
  }
  seen.add(value)
  
  if (isRef(value)) {
    traverse(value.value, seen)
  } else if (isArray(value)) {
    for (let i = 0; i < value.length; i++) {
      traverse(value[i], seen)
    }
  } else if (isSet(value) || isMap(value)) {
    value.forEach((v: any) => {
      traverse(v, seen)
    })
  } else if (isPlainObject(value)) {
    for (const key in value) {
      traverse(value[key], seen)
    }
    for (const key of Object.getOwnPropertySymbols(value)) {
      traverse(value[key as any], seen)
    }
  }
  
  return value
}
```

## 三、effectScope

### 3.1 EffectScope 类

```typescript
// packages/reactivity/src/effectScope.ts
export class EffectScope {
  private _active = true
  effects: ReactiveEffect[] = []
  cleanups: (() => void)[] = []
  parent: EffectScope | undefined
  scopes: EffectScope[] | undefined
  
  constructor(public detached = false) {
    this.parent = activeEffectScope
    if (!detached && activeEffectScope) {
      (activeEffectScope.scopes || (activeEffectScope.scopes = [])).push(this)
    }
  }
  
  get active() {
    return this._active
  }
  
  run<T>(fn: () => T): T | undefined {
    if (this._active) {
      const currentEffectScope = activeEffectScope
      try {
        activeEffectScope = this
        return fn()
      } finally {
        activeEffectScope = currentEffectScope
      }
    }
  }
  
  stop(fromParent?: boolean) {
    if (this._active) {
      // 停止所有 effects
      for (const effect of this.effects) {
        effect.stop()
      }
      
      // 执行 cleanups
      for (const cleanup of this.cleanups) {
        cleanup()
      }
      
      // 停止子 scopes
      if (this.scopes) {
        for (const scope of this.scopes) {
          scope.stop(true)
        }
      }
      
      // 从父 scope 移除
      if (!this.detached && this.parent && !fromParent) {
        const last = this.parent.scopes!.pop()
        if (last && last !== this) {
          this.parent.scopes![this.parent.scopes!.indexOf(this)] = last
        }
      }
      
      this.parent = undefined
      this._active = false
    }
  }
}
```

### 3.2 使用示例

```typescript
import { effectScope, ref, watch } from 'vue'

const scope = effectScope()

scope.run(() => {
  const count = ref(0)
  
  // 这些 effect 都会被 scope 收集
  watch(count, () => {
    console.log('count changed')
  })
  
  watchEffect(() => {
    console.log('count:', count.value)
  })
})

// 停止所有 effects
scope.stop()
```

## 四、provide/inject

### 4.1 provide

```typescript
// packages/runtime-core/src/apiInject.ts
export function provide<T, K = InjectionKey<T> | string | number>(
  key: K,
  value: K extends InjectionKey<infer V> ? V : T
): void {
  if (!currentInstance) {
    if (__DEV__) {
      warn(`provide() can only be used inside setup().`)
    }
  } else {
    let provides = currentInstance.provides
    const parentProvides = currentInstance.parent?.provides
    
    // 首次 provide，创建新对象
    if (parentProvides === provides) {
      provides = currentInstance.provides = Object.create(parentProvides)
    }
    
    provides[key as string] = value
  }
}
```

### 4.2 inject

```typescript
export function inject<T>(
  key: InjectionKey<T> | string,
  defaultValue?: T,
  treatDefaultAsFactory = false
): T | undefined {
  const instance = currentInstance || currentRenderingInstance
  
  if (instance || currentApp) {
    const provides = currentApp
      ? currentApp._context.provides
      : instance
        ? instance.parent == null
          ? instance.vnode.appContext?.provides
          : instance.parent.provides
        : undefined
    
    if (provides && (key as string | symbol) in provides) {
      return provides[key as string]
    } else if (arguments.length > 1) {
      return treatDefaultAsFactory && isFunction(defaultValue)
        ? defaultValue.call(instance && instance.proxy)
        : defaultValue
    } else if (__DEV__) {
      warn(`injection "${String(key)}" not found.`)
    }
  }
}
```

### 4.3 使用示例

```typescript
// 父组件
import { provide, ref } from 'vue'

setup() {
  const theme = ref('dark')
  provide('theme', theme)
}

// 子组件
import { inject } from 'vue'

setup() {
  const theme = inject('theme', 'light')
  return { theme }
}
```

## 五、生命周期钩子

### 5.1 实现原理

```typescript
// packages/runtime-core/src/apiLifecycle.ts
export const onBeforeMount = createHook(LifecycleHooks.BEFORE_MOUNT)
export const onMounted = createHook(LifecycleHooks.MOUNTED)
export const onBeforeUpdate = createHook(LifecycleHooks.BEFORE_UPDATE)
export const onUpdated = createHook(LifecycleHooks.UPDATED)
export const onBeforeUnmount = createHook(LifecycleHooks.BEFORE_UNMOUNT)
export const onUnmounted = createHook(LifecycleHooks.UNMOUNTED)
export const onErrorCaptured = createHook(LifecycleHooks.ERROR_CAPTURED)
export const onRenderTracked = createHook(LifecycleHooks.RENDER_TRACKED)
export const onRenderTriggered = createHook(LifecycleHooks.RENDER_TRIGGERED)
export const onActivated = createHook(LifecycleHooks.ACTIVATED)
export const onDeactivated = createHook(LifecycleHooks.DEACTIVATED)
export const onServerPrefetch = createHook(LifecycleHooks.SERVER_PREFETCH)

function createHook<T extends Function = () => any>(
  lifecycle: LifecycleHooks
) {
  return (hook: T, target: ComponentInternalInstance | null = currentInstance) =>
    injectHook(lifecycle, (...args: unknown[]) => hook(...args), target)
}
```

### 5.2 injectHook

```typescript
export function injectHook(
  type: LifecycleHooks,
  hook: Function & { __weh?: Function },
  target: ComponentInternalInstance | null = currentInstance,
  prepend: boolean = false
): Function | undefined {
  if (target) {
    // 获取或创建钩子数组
    const hooks = target[type] || (target[type] = [])
    
    // 包装钩子函数
    const wrappedHook =
      hook.__weh ||
      (hook.__weh = (...args: unknown[]) => {
        if (target.isUnmounted) {
          return
        }
        
        // 暂停依赖收集
        pauseTracking()
        
        // 设置当前实例
        const reset = setCurrentInstance(target)
        
        // 执行钩子
        const res = callWithAsyncErrorHandling(hook, target, type, args)
        
        reset()
        resetTracking()
        return res
      })
    
    if (prepend) {
      hooks.unshift(wrappedHook)
    } else {
      hooks.push(wrappedHook)
    }
    
    return wrappedHook
  }
}
```

## 六、getCurrentInstance

```typescript
// packages/runtime-core/src/component.ts
export let currentInstance: ComponentInternalInstance | null = null

export const getCurrentInstance = (): ComponentInternalInstance | null =>
  currentInstance || currentRenderingInstance

export const setCurrentInstance = (instance: ComponentInternalInstance | null) => {
  const prev = currentInstance
  currentInstance = instance
  instance?.scope.on()
  return () => {
    instance?.scope.off()
    currentInstance = prev
  }
}

export const unsetCurrentInstance = () => {
  currentInstance?.scope.off()
  currentInstance = null
}
```

## 七、useSlots / useAttrs

```typescript
// packages/runtime-core/src/apiSetupHelpers.ts
export function useSlots(): SetupContext['slots'] {
  return getContext().slots
}

export function useAttrs(): SetupContext['attrs'] {
  return getContext().attrs
}

function getContext(): SetupContext {
  const i = getCurrentInstance()!
  return i.setupContext || (i.setupContext = createSetupContext(i))
}
```

## 八、defineProps / defineEmits

```typescript
// 编译时宏，运行时为空实现
export function defineProps<T>(): Readonly<T> {
  return null as any
}

export function defineEmits<T>(): T {
  return null as any
}

export function defineExpose<T>(exposed?: T): void {
  // 编译时处理
}

export function defineOptions<T>(options?: T): void {
  // 编译时处理
}

export function defineSlots<T>(): T {
  return null as any
}

export function defineModel<T>(): Ref<T> {
  return null as any
}
```

## 九、小结

Composition API 的核心：

1. **watch/watchEffect**：基于 ReactiveEffect，支持多种数据源
2. **effectScope**：批量管理 effects，统一停止
3. **provide/inject**：原型链实现的依赖注入
4. **生命周期钩子**：注入到组件实例的钩子数组
5. **编译时宏**：defineProps 等在编译时处理

---

> 📦 源码地址：[github.com/vuejs/core](https://github.com/vuejs/core)
> 
> 本系列完结，感谢阅读！
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
