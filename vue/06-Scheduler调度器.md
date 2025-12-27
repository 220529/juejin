# Vue.js 源码揭秘（六）：Scheduler 调度器

> 本文深入 scheduler 源码，解析 Vue3 的任务调度、批量更新、nextTick 实现。

## 一、调度器概览

```
┌─────────────────────────────────────────────────────────────┐
│                    调度器架构                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   响应式数据变化                                             │
│        │                                                    │
│        ▼                                                    │
│   effect.scheduler()  ──► queueJob(update)                  │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────┐               │
│   │              queue (任务队列)            │               │
│   │  [job1, job2, job3, ...]                │               │
│   │  按 id 排序，父组件先于子组件            │               │
│   └─────────────────────────────────────────┘               │
│        │                                                    │
│        ▼                                                    │
│   Promise.resolve().then(flushJobs)                         │
│        │                                                    │
│        ▼                                                    │
│   ┌─────────────────────────────────────────┐               │
│   │         pendingPostFlushCbs             │               │
│   │  [mounted, updated, ...]                │               │
│   │  DOM 更新后执行的回调                    │               │
│   └─────────────────────────────────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 二、核心数据结构

```typescript
// packages/runtime-core/src/scheduler.ts

// 任务队列
const queue: SchedulerJob[] = []
let flushIndex = -1

// 后置回调队列
const pendingPostFlushCbs: SchedulerJob[] = []
let activePostFlushCbs: SchedulerJob[] | null = null
let postFlushIndex = 0

// Promise
const resolvedPromise = Promise.resolve()
let currentFlushPromise: Promise<void> | null = null

// 递归限制
const RECURSION_LIMIT = 100
```

### 2.1 SchedulerJob

```typescript
export interface SchedulerJob extends Function {
  id?: number                    // 任务 ID（组件 uid）
  flags?: SchedulerJobFlags      // 标记
  i?: ComponentInternalInstance  // 组件实例
}

export enum SchedulerJobFlags {
  QUEUED = 1 << 0,        // 已入队
  PRE = 1 << 1,           // pre watcher
  ALLOW_RECURSE = 1 << 2, // 允许递归
  DISPOSED = 1 << 3       // 已销毁
}
```

## 三、queueJob

```typescript
export function queueJob(job: SchedulerJob): void {
  // 检查是否已入队
  if (!(job.flags! & SchedulerJobFlags.QUEUED)) {
    const jobId = getId(job)
    const lastJob = queue[queue.length - 1]
    
    if (
      !lastJob ||
      // 快速路径：id 大于队尾，直接 push
      (!(job.flags! & SchedulerJobFlags.PRE) && jobId >= getId(lastJob))
    ) {
      queue.push(job)
    } else {
      // 二分查找插入位置
      queue.splice(findInsertionIndex(jobId), 0, job)
    }
    
    // 标记已入队
    job.flags! |= SchedulerJobFlags.QUEUED
    
    // 触发刷新
    queueFlush()
  }
}

// 获取任务 ID
const getId = (job: SchedulerJob): number =>
  job.id == null 
    ? (job.flags! & SchedulerJobFlags.PRE ? -1 : Infinity) 
    : job.id
```

### 3.1 findInsertionIndex

```typescript
// 二分查找插入位置，保持队列有序
function findInsertionIndex(id: number) {
  let start = flushIndex + 1
  let end = queue.length

  while (start < end) {
    const middle = (start + end) >>> 1
    const middleJob = queue[middle]
    const middleJobId = getId(middleJob)
    
    if (
      middleJobId < id ||
      (middleJobId === id && middleJob.flags! & SchedulerJobFlags.PRE)
    ) {
      start = middle + 1
    } else {
      end = middle
    }
  }

  return start
}
```

## 四、queueFlush

```typescript
function queueFlush() {
  if (!currentFlushPromise) {
    currentFlushPromise = resolvedPromise.then(flushJobs)
  }
}
```

## 五、flushJobs

```typescript
function flushJobs(seen?: CountMap) {
  if (__DEV__) {
    seen = seen || new Map()
  }

  const check = __DEV__
    ? (job: SchedulerJob) => checkRecursiveUpdates(seen!, job)
    : NOOP

  try {
    // 遍历执行任务
    for (flushIndex = 0; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      
      if (job && !(job.flags! & SchedulerJobFlags.DISPOSED)) {
        // 开发环境检查递归
        if (__DEV__ && check(job)) {
          continue
        }
        
        // 允许递归的任务，先清除 QUEUED 标记
        if (job.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
        
        // 执行任务
        callWithErrorHandling(
          job,
          job.i,
          job.i ? ErrorCodes.COMPONENT_UPDATE : ErrorCodes.SCHEDULER
        )
        
        // 不允许递归的任务，执行后清除标记
        if (!(job.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
          job.flags! &= ~SchedulerJobFlags.QUEUED
        }
      }
    }
  } finally {
    // 清理剩余任务的 QUEUED 标记
    for (; flushIndex < queue.length; flushIndex++) {
      const job = queue[flushIndex]
      if (job) {
        job.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }

    flushIndex = -1
    queue.length = 0

    // 执行后置回调
    flushPostFlushCbs(seen)

    currentFlushPromise = null
    
    // 如果有新任务，继续刷新
    if (queue.length || pendingPostFlushCbs.length) {
      flushJobs(seen)
    }
  }
}
```

## 六、Pre Flush

```typescript
export function flushPreFlushCbs(
  instance?: ComponentInternalInstance,
  seen?: CountMap,
  i: number = flushIndex + 1
): void {
  if (__DEV__) {
    seen = seen || new Map()
  }
  
  for (; i < queue.length; i++) {
    const cb = queue[i]
    
    if (cb && cb.flags! & SchedulerJobFlags.PRE) {
      // 过滤特定组件
      if (instance && cb.id !== instance.uid) {
        continue
      }
      
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      
      // 从队列移除
      queue.splice(i, 1)
      i--
      
      // 清除标记并执行
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      cb()
      if (!(cb.flags! & SchedulerJobFlags.ALLOW_RECURSE)) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
    }
  }
}
```

## 七、Post Flush

### 7.1 queuePostFlushCb

```typescript
export function queuePostFlushCb(cb: SchedulerJobs): void {
  if (!isArray(cb)) {
    if (activePostFlushCbs && cb.id === -1) {
      // 插入到当前执行位置之后
      activePostFlushCbs.splice(postFlushIndex + 1, 0, cb)
    } else if (!(cb.flags! & SchedulerJobFlags.QUEUED)) {
      pendingPostFlushCbs.push(cb)
      cb.flags! |= SchedulerJobFlags.QUEUED
    }
  } else {
    // 数组直接 push（生命周期钩子）
    pendingPostFlushCbs.push(...cb)
  }
  
  queueFlush()
}
```

### 7.2 flushPostFlushCbs

```typescript
export function flushPostFlushCbs(seen?: CountMap): void {
  if (pendingPostFlushCbs.length) {
    // 去重并排序
    const deduped = [...new Set(pendingPostFlushCbs)].sort(
      (a, b) => getId(a) - getId(b)
    )
    pendingPostFlushCbs.length = 0

    // 嵌套调用处理
    if (activePostFlushCbs) {
      activePostFlushCbs.push(...deduped)
      return
    }

    activePostFlushCbs = deduped
    
    if (__DEV__) {
      seen = seen || new Map()
    }

    // 执行回调
    for (
      postFlushIndex = 0;
      postFlushIndex < activePostFlushCbs.length;
      postFlushIndex++
    ) {
      const cb = activePostFlushCbs[postFlushIndex]
      
      if (__DEV__ && checkRecursiveUpdates(seen!, cb)) {
        continue
      }
      
      if (cb.flags! & SchedulerJobFlags.ALLOW_RECURSE) {
        cb.flags! &= ~SchedulerJobFlags.QUEUED
      }
      
      if (!(cb.flags! & SchedulerJobFlags.DISPOSED)) {
        cb()
      }
      
      cb.flags! &= ~SchedulerJobFlags.QUEUED
    }
    
    activePostFlushCbs = null
    postFlushIndex = 0
  }
}
```

## 八、nextTick

```typescript
export function nextTick<T = void>(
  this: T,
  fn?: (this: T) => void
): Promise<void> {
  const p = currentFlushPromise || resolvedPromise
  return fn ? p.then(this ? fn.bind(this) : fn) : p
}
```

### 8.1 使用示例

```typescript
import { ref, nextTick } from 'vue'

const count = ref(0)

async function increment() {
  count.value++
  count.value++
  count.value++
  
  // DOM 还未更新
  console.log(document.getElementById('count').textContent) // 0
  
  await nextTick()
  
  // DOM 已更新
  console.log(document.getElementById('count').textContent) // 3
}
```

## 九、递归检测

```typescript
function checkRecursiveUpdates(seen: CountMap, fn: SchedulerJob) {
  const count = seen.get(fn) || 0
  
  if (count > RECURSION_LIMIT) {
    const instance = fn.i
    const componentName = instance && getComponentName(instance.type)
    
    handleError(
      `Maximum recursive updates exceeded${
        componentName ? ` in component <${componentName}>` : ``
      }. ` +
      `This means you have a reactive effect that is mutating its own ` +
      `dependencies and thus recursively triggering itself.`,
      null,
      ErrorCodes.APP_ERROR_HANDLER
    )
    return true
  }
  
  seen.set(fn, count + 1)
  return false
}
```

## 十、执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                    调度执行流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  同步代码执行                                                │
│     │                                                       │
│     ├── state.count++  ──► trigger ──► queueJob(update)    │
│     ├── state.count++  ──► trigger ──► 已入队，跳过         │
│     └── state.count++  ──► trigger ──► 已入队，跳过         │
│                                                             │
│  ─────────── 同步代码结束 ───────────                        │
│                                                             │
│  微任务执行                                                  │
│     │                                                       │
│     ▼                                                       │
│  flushJobs()                                                │
│     │                                                       │
│     ├── 1. 按 id 排序（父组件 → 子组件）                    │
│     │                                                       │
│     ├── 2. 执行 queue 中的任务                              │
│     │      └── componentUpdateFn()                          │
│     │           ├── beforeUpdate hooks                      │
│     │           ├── render() → patch()                      │
│     │           └── queuePostFlushCb(updated)               │
│     │                                                       │
│     ├── 3. 执行 flushPostFlushCbs()                         │
│     │      ├── mounted hooks                                │
│     │      └── updated hooks                                │
│     │                                                       │
│     └── 4. 检查是否有新任务，递归执行                        │
│                                                             │
│  ─────────── 微任务结束 ───────────                          │
│                                                             │
│  nextTick 回调执行                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 十一、与组件更新的关系

```typescript
// packages/runtime-core/src/renderer.ts
const setupRenderEffect = (instance, initialVNode, container, ...) => {
  const componentUpdateFn = () => {
    // 组件更新逻辑
  }
  
  // 创建响应式副作用
  const effect = (instance.effect = new ReactiveEffect(
    componentUpdateFn,
    NOOP,
    () => queueJob(update)  // scheduler：将更新任务入队
  ))
  
  const update = (instance.update = () => {
    if (effect.dirty) {
      effect.run()
    }
  })
  
  // 设置任务 ID 为组件 uid
  update.id = instance.uid
  
  // 允许递归（watch 回调可能触发自身）
  update.flags |= SchedulerJobFlags.ALLOW_RECURSE
  
  update()
}
```

## 十二、小结

Vue3 调度器的核心：

1. **批量更新**：多次状态变化只触发一次更新
2. **任务排序**：按组件 uid 排序，父组件先于子组件
3. **微任务**：使用 Promise.resolve() 延迟执行
4. **Pre/Post**：支持 DOM 更新前后的回调
5. **递归检测**：防止无限循环更新

---

> 📦 源码地址：[github.com/vuejs/core](https://github.com/vuejs/core)
> 
> 下一篇：Composition API 实现
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
