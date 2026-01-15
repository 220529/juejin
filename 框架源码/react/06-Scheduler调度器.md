# React 19 源码揭秘（六）：Scheduler 时间切片的秘密

> 本文深入 Scheduler 源码，揭秘 React 如何实现可中断渲染和优先级调度。

## 前言

你有没有想过：

- React 是怎么做到"不卡顿"的？
- 用户输入为什么能打断正在进行的渲染？
- 时间切片到底是什么？

答案就在 Scheduler 调度器中。

## 一、为什么需要 Scheduler？

### 问题：长任务阻塞

```javascript
// 假设有 10000 个节点需要渲染
function render() {
  for (let i = 0; i < 10000; i++) {
    processNode(i);  // 每个节点处理 1ms
  }
}
// 总共需要 10 秒，期间页面完全卡死！
```

### 解决：时间切片

```javascript
function render() {
  let i = 0;
  function work() {
    while (i < 10000 && !shouldYield()) {  // 每 5ms 检查一次
      processNode(i++);
    }
    if (i < 10000) {
      scheduleCallback(work);  // 还有任务，继续调度
    }
  }
  scheduleCallback(work);
}
// 每 5ms 让出主线程，用户可以正常交互
```

## 二、核心概念

### 优先级

```javascript
// SchedulerPriorities.js
export const NoPriority = 0;
export const ImmediatePriority = 1;    // 立即执行
export const UserBlockingPriority = 2; // 用户交互（250ms）
export const NormalPriority = 3;       // 普通更新（5000ms）
export const LowPriority = 4;          // 低优先级（10000ms）
export const IdlePriority = 5;         // 空闲执行（永不过期）
```

### 超时时间

不同优先级有不同的超时时间：

```javascript
var timeout;
switch (priorityLevel) {
  case ImmediatePriority:
    timeout = -1;           // 立即过期
    break;
  case UserBlockingPriority:
    timeout = 250;          // 250ms
    break;
  case NormalPriority:
    timeout = 5000;         // 5 秒
    break;
  case LowPriority:
    timeout = 10000;        // 10 秒
    break;
  case IdlePriority:
    timeout = 1073741823;   // 约 12 天，几乎永不过期
    break;
}
```

### Task 结构

```javascript
type Task = {
  id: number,              // 任务 ID
  callback: Function,      // 任务回调
  priorityLevel: number,   // 优先级
  startTime: number,       // 开始时间
  expirationTime: number,  // 过期时间 = startTime + timeout
  sortIndex: number,       // 排序索引
};
```

## 三、两个队列

Scheduler 维护两个队列：

```javascript
var taskQueue = [];   // 就绪队列（按 expirationTime 排序）
var timerQueue = [];  // 延迟队列（按 startTime 排序）
```

```
┌─────────────────────────────────────────────────────────┐
│                      timerQueue                         │
│                     （延迟任务）                          │
│                                                         │
│  startTime > currentTime 的任务在这里等待               │
└─────────────────────────────────────────────────────────┘
                           │
                           │ 到时间了
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      taskQueue                          │
│                     （就绪任务）                          │
│                                                         │
│  按 expirationTime 排序，最小堆实现                      │
└─────────────────────────────────────────────────────────┘
                           │
                           │ 取出执行
                           ▼
                      执行任务回调
```

## 四、最小堆

任务队列使用最小堆实现，保证 O(1) 获取最高优先级任务：

```javascript
// SchedulerMinHeap.js
export function push(heap, node) {
  const index = heap.length;
  heap.push(node);
  siftUp(heap, node, index);  // 上浮
}

export function peek(heap) {
  return heap.length === 0 ? null : heap[0];  // O(1)
}

export function pop(heap) {
  const first = heap[0];
  const last = heap.pop();
  if (last !== first) {
    heap[0] = last;
    siftDown(heap, last, 0);  // 下沉
  }
  return first;
}

// 比较函数：先比 sortIndex，再比 id
function compare(a, b) {
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

## 五、调度流程

### scheduleCallback

```javascript
function unstable_scheduleCallback(priorityLevel, callback, options) {
  var currentTime = getCurrentTime();
  var startTime = currentTime;
  
  // 计算过期时间
  var timeout = getTimeoutByPriority(priorityLevel);
  var expirationTime = startTime + timeout;
  
  // 创建任务
  var newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1,
  };
  
  if (startTime > currentTime) {
    // 延迟任务，加入 timerQueue
    newTask.sortIndex = startTime;
    push(timerQueue, newTask);
    
    // 如果是最早的延迟任务，设置定时器
    if (peek(taskQueue) === null && newTask === peek(timerQueue)) {
      requestHostTimeout(handleTimeout, startTime - currentTime);
    }
  } else {
    // 就绪任务，加入 taskQueue
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    
    // 请求调度
    requestHostCallback();
  }
  
  return newTask;
}
```

### workLoop

```javascript
function workLoop(initialTime) {
  let currentTime = initialTime;
  
  // 检查延迟任务是否到期
  advanceTimers(currentTime);
  
  // 获取最高优先级任务
  currentTask = peek(taskQueue);
  
  while (currentTask !== null) {
    // 任务未过期 && 时间片用完 → 中断
    if (currentTask.expirationTime > currentTime && shouldYieldToHost()) {
      break;
    }
    
    const callback = currentTask.callback;
    if (typeof callback === 'function') {
      currentTask.callback = null;
      
      // 执行任务
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      const continuationCallback = callback(didUserCallbackTimeout);
      
      if (typeof continuationCallback === 'function') {
        // 任务未完成，保留继续执行
        currentTask.callback = continuationCallback;
        return true;
      } else {
        // 任务完成，移除
        if (currentTask === peek(taskQueue)) {
          pop(taskQueue);
        }
      }
    } else {
      pop(taskQueue);
    }
    
    currentTask = peek(taskQueue);
  }
  
  return currentTask !== null;
}
```

### shouldYieldToHost

```javascript
let frameInterval = 5;  // 5ms 一个时间片
let startTime = -1;

function shouldYieldToHost() {
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    return false;  // 时间片未用完，继续
  }
  return true;  // 让出主线程
}
```

## 六、调度触发：MessageChannel

为什么用 MessageChannel 而不是 setTimeout？

```javascript
// setTimeout 有最小 4ms 延迟
setTimeout(callback, 0);  // 实际 >= 4ms

// MessageChannel 没有这个限制
const channel = new MessageChannel();
channel.port1.onmessage = callback;
channel.port2.postMessage(null);  // 立即触发
```

### 实现

```javascript
const channel = new MessageChannel();
const port = channel.port2;
channel.port1.onmessage = performWorkUntilDeadline;

function schedulePerformWorkUntilDeadline() {
  port.postMessage(null);
}

function performWorkUntilDeadline() {
  if (isMessageLoopRunning) {
    const currentTime = getCurrentTime();
    startTime = currentTime;
    
    let hasMoreWork = true;
    try {
      hasMoreWork = flushWork(currentTime);
    } finally {
      if (hasMoreWork) {
        // 还有任务，继续调度
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
      }
    }
  }
}
```

## 七、与 React 的关系

```
React 更新
    │
    ▼
scheduleUpdateOnFiber()
    │
    ▼
ensureRootIsScheduled()
    │
    ▼
scheduleSyncCallback() 或 scheduleCallback()
    │
    ▼
Scheduler.scheduleCallback(priority, performConcurrentWorkOnRoot)
    │
    ▼
workLoop() → performConcurrentWorkOnRoot()
    │
    ▼
workLoopConcurrent() ← 这里会检查 shouldYield()
```

### workLoopConcurrent

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

当 `shouldYield()` 返回 true 时，React 会暂停渲染，让出主线程。

## 八、优先级抢占

高优先级任务可以打断低优先级任务：

```javascript
// 正在进行低优先级渲染
scheduleCallback(NormalPriority, renderLowPriority);

// 用户点击，触发高优先级更新
scheduleCallback(UserBlockingPriority, renderHighPriority);

// workLoop 中：
// 1. 高优先级任务先过期
// 2. 取出高优先级任务执行
// 3. 低优先级任务被"饿死"后也会执行
```

## 九、调试技巧

```javascript
// 在这些位置打断点：

// 调度任务
unstable_scheduleCallback  // Scheduler.js

// 执行任务
workLoop                   // Scheduler.js

// 时间切片检查
shouldYieldToHost          // Scheduler.js

// 调度触发
performWorkUntilDeadline   // Scheduler.js
```

### 观察时间切片

```javascript
// 在 workLoopConcurrent 中加日志
function workLoopConcurrent() {
  let count = 0;
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
    count++;
  }
  console.log(`处理了 ${count} 个节点后让出`);
}
```

## 小结

Scheduler 的核心机制：

1. **优先级**：5 个级别，不同超时时间
2. **两个队列**：taskQueue（就绪）+ timerQueue（延迟）
3. **最小堆**：O(1) 获取最高优先级任务
4. **时间切片**：5ms 一片，shouldYield 检查
5. **MessageChannel**：避免 setTimeout 的 4ms 延迟
6. **可中断**：高优先级可以打断低优先级

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[Diff 算法的实现](https://juejin.cn/post/7588001624905990163)
> 
> 下一篇：[Commit 阶段与 DOM 操作](https://juejin.cn/post/7588061231589556265)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
