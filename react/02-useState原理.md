# React 19 源码揭秘（二）：useState 的实现原理

> 本文深入 React 源码，带你彻底搞懂 useState 从调用到更新的完整流程。

## 前言

`useState` 可能是你用得最多的 Hook，但你知道它背后是怎么工作的吗？

```javascript
const [count, setCount] = useState(0);
setCount(count + 1);  // 这行代码背后发生了什么？
```

本文将从源码角度，完整解析 useState 的实现原理。

## 一、Hook 的数据结构

首先，我们需要了解 Hook 在 React 内部是如何存储的。

### Hook 节点

每个 Hook 调用都会创建一个 Hook 对象：

```javascript
type Hook = {
  memoizedState: any,    // 存储的状态值
  baseState: any,        // 基础状态（用于更新计算）
  baseQueue: Update | null,  // 基础更新队列
  queue: UpdateQueue | null, // 更新队列
  next: Hook | null,     // 指向下一个 Hook
};
```

### Hook 链表

多个 Hook 以链表形式存储在 Fiber 节点的 `memoizedState` 上：

```
Fiber.memoizedState
        │
        ▼
    ┌───────┐     ┌───────┐     ┌───────┐
    │ Hook1 │ ──► │ Hook2 │ ──► │ Hook3 │ ──► null
    │useState│     │useEffect│    │useMemo│
    └───────┘     └───────┘     └───────┘
```

这就是为什么 **Hook 不能在条件语句中调用**——React 依赖调用顺序来匹配 Hook。

## 二、首次渲染：mountState

当组件首次渲染时，useState 会调用 `mountState`：

```javascript
// 源码位置：react-reconciler/src/ReactFiberHooks.js

function mountState(initialState) {
  // 1. 创建 Hook 节点，加入链表
  const hook = mountWorkInProgressHook();
  
  // 2. 处理初始值（支持函数式初始化）
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  
  // 3. 保存初始状态
  hook.memoizedState = hook.baseState = initialState;
  
  // 4. 创建更新队列
  const queue = {
    pending: null,           // 待处理的更新
    lanes: NoLanes,
    dispatch: null,          // setState 函数
    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  };
  hook.queue = queue;
  
  // 5. 绑定 dispatch 函数（就是 setState）
  const dispatch = dispatchSetState.bind(null, currentlyRenderingFiber, queue);
  queue.dispatch = dispatch;
  
  // 6. 返回 [state, setState]
  return [hook.memoizedState, dispatch];
}
```

### mountWorkInProgressHook

这个函数负责创建 Hook 节点并维护链表：

```javascript
function mountWorkInProgressHook() {
  const hook = {
    memoizedState: null,
    baseState: null,
    baseQueue: null,
    queue: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // 第一个 Hook，挂载到 Fiber
    currentlyRenderingFiber.memoizedState = workInProgressHook = hook;
  } else {
    // 追加到链表末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  
  return workInProgressHook;
}
```

## 三、触发更新：dispatchSetState

当你调用 `setCount(1)` 时，实际执行的是 `dispatchSetState`：

```javascript
function dispatchSetState(fiber, queue, action) {
  // 1. 获取更新优先级
  const lane = requestUpdateLane(fiber);
  
  // 2. 创建更新对象
  const update = {
    lane,
    action,              // 新值或更新函数
    hasEagerState: false,
    eagerState: null,
    next: null,
  };
  
  // 3. 性能优化：Eager State（提前计算）
  if (fiber.lanes === NoLanes) {
    const currentState = queue.lastRenderedState;
    const eagerState = basicStateReducer(currentState, action);
    update.hasEagerState = true;
    update.eagerState = eagerState;
    
    // 如果新旧状态相同，跳过更新！
    if (Object.is(eagerState, currentState)) {
      return;  // Bailout!
    }
  }
  
  // 4. 将更新加入队列
  enqueueConcurrentHookUpdate(fiber, queue, update, lane);
  
  // 5. 调度更新
  scheduleUpdateOnFiber(root, fiber, lane);
}
```

### Eager State 优化

这是一个重要的性能优化：

```javascript
const [count, setCount] = useState(0);

// 点击按钮
setCount(0);  // 状态没变，React 会跳过这次更新！
```

React 会在调度之前就计算新状态，如果和旧状态相同（通过 `Object.is` 比较），直接跳过整个更新流程。

## 四、更新渲染：updateState

当组件重新渲染时，useState 会调用 `updateState`：

```javascript
function updateState(initialState) {
  // useState 本质上是预设了 reducer 的 useReducer
  return updateReducer(basicStateReducer, initialState);
}

// 基础 reducer：支持值或函数
function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}
```

### updateReducer

这是处理更新的核心逻辑：

```javascript
function updateReducer(reducer, initialArg) {
  // 1. 获取当前 Hook
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;
  
  // 2. 获取待处理的更新
  const pending = queue.pending;
  
  // 3. 计算新状态
  let newState = hook.baseState;
  if (pending !== null) {
    let update = pending.first;
    do {
      const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== null);
  }
  
  // 4. 保存新状态
  hook.memoizedState = newState;
  
  // 5. 返回新状态和 dispatch
  return [hook.memoizedState, queue.dispatch];
}
```

### updateWorkInProgressHook

更新时，需要从 current 树复制 Hook：

```javascript
function updateWorkInProgressHook() {
  // 从 current Fiber 获取对应的 Hook
  let nextCurrentHook;
  if (currentHook === null) {
    const current = currentlyRenderingFiber.alternate;
    nextCurrentHook = current.memoizedState;
  } else {
    nextCurrentHook = currentHook.next;
  }
  
  currentHook = nextCurrentHook;
  
  // 复制 Hook 到 workInProgress
  const newHook = {
    memoizedState: currentHook.memoizedState,
    baseState: currentHook.baseState,
    baseQueue: currentHook.baseQueue,
    queue: currentHook.queue,
    next: null,
  };
  
  // 加入链表...
  return newHook;
}
```

## 五、完整流程图

```
┌─────────────────────────────────────────────────────────┐
│                    首次渲染 (Mount)                      │
├─────────────────────────────────────────────────────────┤
│  useState(0)                                            │
│      │                                                  │
│      ▼                                                  │
│  mountState(0)                                          │
│      │                                                  │
│      ├──► 创建 Hook 节点                                │
│      ├──► 初始化 memoizedState = 0                      │
│      ├──► 创建 UpdateQueue                              │
│      ├──► 绑定 dispatch = dispatchSetState              │
│      │                                                  │
│      ▼                                                  │
│  返回 [0, setCount]                                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    触发更新                              │
├─────────────────────────────────────────────────────────┤
│  setCount(1)                                            │
│      │                                                  │
│      ▼                                                  │
│  dispatchSetState(fiber, queue, 1)                      │
│      │                                                  │
│      ├──► 获取优先级 lane                               │
│      ├──► 创建 Update 对象                              │
│      ├──► Eager State: 计算新状态                       │
│      ├──► 比较新旧状态，相同则 Bailout                   │
│      ├──► 入队更新                                      │
│      │                                                  │
│      ▼                                                  │
│  scheduleUpdateOnFiber() ──► 调度重新渲染               │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    重新渲染 (Update)                     │
├─────────────────────────────────────────────────────────┤
│  useState(0)  // 初始值被忽略                           │
│      │                                                  │
│      ▼                                                  │
│  updateState(0)                                         │
│      │                                                  │
│      ▼                                                  │
│  updateReducer(basicStateReducer, 0)                    │
│      │                                                  │
│      ├──► 获取对应的 Hook                               │
│      ├──► 处理 UpdateQueue 中的更新                     │
│      ├──► 计算新状态 = 1                                │
│      │                                                  │
│      ▼                                                  │
│  返回 [1, setCount]                                     │
└─────────────────────────────────────────────────────────┘
```

## 六、Dispatcher 切换

React 如何区分 mount 和 update？答案是 **Dispatcher 切换**：

```javascript
// renderWithHooks 中
ReactSharedInternals.H = 
  current === null || current.memoizedState === null
    ? HooksDispatcherOnMount   // 首次渲染
    : HooksDispatcherOnUpdate; // 更新渲染

// 两个 Dispatcher 的 useState 指向不同函数
const HooksDispatcherOnMount = {
  useState: mountState,
  useEffect: mountEffect,
  // ...
};

const HooksDispatcherOnUpdate = {
  useState: updateState,
  useEffect: updateEffect,
  // ...
};
```

渲染完成后，切换到 ContextOnlyDispatcher，禁止在组件外调用 Hook：

```javascript
// 渲染完成后
ReactSharedInternals.H = ContextOnlyDispatcher;

const ContextOnlyDispatcher = {
  useState: throwInvalidHookError,  // 抛出错误
  // ...
};
```

## 七、为什么 Hook 不能条件调用？

现在你应该明白了：

```javascript
// ❌ 错误
if (condition) {
  const [a, setA] = useState(0);  // Hook 1
}
const [b, setB] = useState(0);    // Hook 2 或 Hook 1？

// ✅ 正确
const [a, setA] = useState(0);    // 始终是 Hook 1
const [b, setB] = useState(0);    // 始终是 Hook 2
```

React 通过遍历链表来匹配 Hook，如果顺序变了，状态就乱了。

## 八、调试技巧

想要亲自验证？在这些位置打断点：

```javascript
// 首次渲染
mountState          // react-reconciler/src/ReactFiberHooks.js

// 触发更新
dispatchSetState    // react-reconciler/src/ReactFiberHooks.js

// 重新渲染
updateReducer       // react-reconciler/src/ReactFiberHooks.js
```

用 Counter 组件测试：

```javascript
const Counter = () => {
  const [count, setCount] = useState(0);  // 断点这里
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
};
```

## 小结

本文深入分析了 useState 的实现原理：

1. **数据结构**：Hook 以链表形式存储在 Fiber.memoizedState
2. **首次渲染**：mountState 创建 Hook 和 UpdateQueue
3. **触发更新**：dispatchSetState 创建 Update，调度渲染
4. **Eager State**：提前计算，相同状态跳过更新
5. **重新渲染**：updateReducer 处理更新队列，计算新状态
6. **Dispatcher**：通过切换实现 mount/update 的区分

下一篇我们将分析 **useEffect 的实现原理**，看看副作用是如何被调度和执行的。

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[React 19 源码全景图](https://juejin.cn/post/7588007285547286534)
> 
> 下一篇：[useEffect 的实现原理](https://juejin.cn/post/7588001624905973779)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
