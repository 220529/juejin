# React 19 源码揭秘（三）：useEffect 的实现原理

> 本文深入源码，带你理解 useEffect 从创建到执行的完整生命周期。

## 前言

useEffect 是 React 中最常用也最容易用错的 Hook。你是否遇到过这些问题：

- 为什么 useEffect 里拿到的是旧值？
- useEffect 和 useLayoutEffect 有什么区别？
- 为什么依赖数组写错会导致死循环？

本文将从源码角度，彻底解答这些问题。

## 一、Effect 的数据结构

### Effect 对象

每个 useEffect 调用都会创建一个 Effect 对象：

```javascript
type Effect = {
  tag: HookFlags,           // 标记：Passive、Layout、Insertion
  create: () => (() => void) | void,  // 创建函数（你传入的函数）
  inst: EffectInstance,     // 实例，存储 destroy 函数
  deps: Array<mixed> | null, // 依赖数组
  next: Effect,             // 下一个 Effect（环形链表）
};

type EffectInstance = {
  destroy: void | (() => void),  // 清理函数
};
```

### Effect 链表

Effect 存储在 Fiber 的 `updateQueue` 中，形成环形链表：

```
Fiber.updateQueue.lastEffect
              │
              ▼
        ┌──────────┐
        │ Effect 3 │◄─────┐
        └────┬─────┘      │
             │            │
             ▼            │
        ┌──────────┐      │
        │ Effect 1 │      │
        └────┬─────┘      │
             │            │
             ▼            │
        ┌──────────┐      │
        │ Effect 2 │──────┘
        └──────────┘
```

## 二、Effect 的标记（Flags）

React 用位运算标记不同类型的 Effect：

```javascript
// ReactHookEffectTags.js
export const NoFlags = 0b0000;
export const HasEffect = 0b0001;    // 需要执行
export const Insertion = 0b0010;    // useInsertionEffect
export const Layout = 0b0100;       // useLayoutEffect
export const Passive = 0b1000;      // useEffect
```

三种 Effect 的执行时机：

| Hook | 标记 | 执行时机 |
|------|------|---------|
| useInsertionEffect | Insertion | DOM 变更前（给 CSS-in-JS 用）|
| useLayoutEffect | Layout | DOM 变更后，浏览器绘制前 |
| useEffect | Passive | 浏览器绘制后（异步）|

## 三、首次渲染：mountEffect

```javascript
function mountEffect(create, deps) {
  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,  // Fiber flags
    HookPassive,                          // Hook flags
    create,
    deps,
  );
}

function mountEffectImpl(fiberFlags, hookFlags, create, deps) {
  // 1. 创建 Hook 节点
  const hook = mountWorkInProgressHook();
  
  // 2. 保存依赖
  const nextDeps = deps === undefined ? null : deps;
  
  // 3. 标记 Fiber 有副作用
  currentlyRenderingFiber.flags |= fiberFlags;
  
  // 4. 创建 Effect 并加入链表
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // 首次渲染一定要执行
    create,
    createEffectInstance(),
    nextDeps,
  );
}
```

### pushEffect

将 Effect 加入环形链表：

```javascript
function pushEffect(tag, create, inst, deps) {
  const effect = {
    tag,
    create,
    inst,
    deps,
    next: null,
  };
  
  let componentUpdateQueue = currentlyRenderingFiber.updateQueue;
  if (componentUpdateQueue === null) {
    // 创建更新队列
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber.updateQueue = componentUpdateQueue;
    componentUpdateQueue.lastEffect = effect.next = effect;  // 环形
  } else {
    // 加入环形链表
    const lastEffect = componentUpdateQueue.lastEffect;
    const firstEffect = lastEffect.next;
    lastEffect.next = effect;
    effect.next = firstEffect;
    componentUpdateQueue.lastEffect = effect;
  }
  
  return effect;
}
```

## 四、更新渲染：updateEffect

```javascript
function updateEffect(create, deps) {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(fiberFlags, hookFlags, create, deps) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevEffect = hook.memoizedState;
  const inst = prevEffect.inst;

  if (nextDeps !== null) {
    const prevDeps = prevEffect.deps;
    // 比较依赖是否变化
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      // 依赖没变，不需要执行
      hook.memoizedState = pushEffect(hookFlags, create, inst, nextDeps);
      return;
    }
  }

  // 依赖变了，标记需要执行
  currentlyRenderingFiber.flags |= fiberFlags;
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,  // 加上 HasEffect 标记
    create,
    inst,
    nextDeps,
  );
}
```

### areHookInputsEqual

依赖比较使用 `Object.is`：

```javascript
function areHookInputsEqual(nextDeps, prevDeps) {
  if (prevDeps === null) {
    return false;
  }
  
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (Object.is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

## 五、Commit 阶段：调度执行

Effect 的执行发生在 Commit 阶段，但 useEffect 是**异步调度**的：

```javascript
// commitRootImpl 中
if ((finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags) {
  
  // 调度 useEffect（异步）
  scheduleCallback(NormalSchedulerPriority, () => {
    flushPassiveEffects();
    return null;
  });
}
```

### flushPassiveEffects

这是 useEffect 真正执行的地方：

```javascript
function flushPassiveEffects() {
  if (rootWithPendingPassiveEffects !== null) {
    const root = rootWithPendingPassiveEffects;
    
    // 1. 先执行所有的销毁函数（destroy）
    commitPassiveUnmountEffects(root.current);
    
    // 2. 再执行所有的创建函数（create）
    commitPassiveMountEffects(root, root.current);
  }
}
```

### 执行销毁函数

```javascript
function commitPassiveUnmountEffects(finishedWork) {
  // 遍历 Fiber 树
  commitPassiveUnmountOnFiber(finishedWork);
}

function commitHookPassiveUnmountEffects(finishedWork, hookFlags) {
  const updateQueue = finishedWork.updateQueue;
  const lastEffect = updateQueue.lastEffect;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        // 执行销毁函数
        const inst = effect.inst;
        const destroy = inst.destroy;
        if (destroy !== undefined) {
          inst.destroy = undefined;
          destroy();  // 执行你 return 的清理函数
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

### 执行创建函数

```javascript
function commitHookPassiveMountEffects(finishedWork, hookFlags) {
  const updateQueue = finishedWork.updateQueue;
  const lastEffect = updateQueue.lastEffect;
  
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & hookFlags) === hookFlags) {
        // 执行创建函数
        const create = effect.create;
        const inst = effect.inst;
        const destroy = create();  // 执行你传入的函数
        inst.destroy = destroy;    // 保存返回的清理函数
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

## 六、useEffect vs useLayoutEffect

两者的区别在于执行时机：

```
┌─────────────────────────────────────────────────────────┐
│                    Commit 阶段                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Before Mutation ──► Mutation ──► Layout ──► 绘制      │
│                          │           │         │        │
│                          │           │         │        │
│                          ▼           ▼         ▼        │
│                       DOM 操作   useLayout   useEffect  │
│                                   Effect     (异步)     │
│                                   (同步)                │
└─────────────────────────────────────────────────────────┘
```

### useLayoutEffect

```javascript
function mountLayoutEffect(create, deps) {
  return mountEffectImpl(
    UpdateEffect | LayoutStaticEffect,
    HookLayout,  // Layout 标记
    create,
    deps,
  );
}
```

useLayoutEffect 在 `commitLayoutEffects` 中**同步执行**：

```javascript
function commitLayoutEffectOnFiber(finishedWork) {
  switch (finishedWork.tag) {
    case FunctionComponent: {
      // 同步执行 useLayoutEffect
      commitHookLayoutEffects(finishedWork, HookLayout | HookHasEffect);
      break;
    }
  }
}
```

### 什么时候用 useLayoutEffect？

当你需要在浏览器绘制前同步读取/修改 DOM 时：

```javascript
// ❌ 可能闪烁
useEffect(() => {
  element.style.transform = `translateX(${x}px)`;
}, [x]);

// ✅ 不会闪烁
useLayoutEffect(() => {
  element.style.transform = `translateX(${x}px)`;
}, [x]);
```

## 七、执行顺序

组件树的 Effect 执行顺序：

```
组件树：
  App
   └── Parent
        └── Child

Mount 时的执行顺序：
1. Child useLayoutEffect 创建
2. Parent useLayoutEffect 创建
3. App useLayoutEffect 创建
4. ─── 浏览器绘制 ───
5. Child useEffect 创建
6. Parent useEffect 创建
7. App useEffect 创建

Unmount 时的执行顺序：
1. App useLayoutEffect 销毁
2. Parent useLayoutEffect 销毁
3. Child useLayoutEffect 销毁
4. ─── 浏览器绘制 ───
5. App useEffect 销毁
6. Parent useEffect 销毁
7. Child useEffect 销毁
```

## 八、常见问题解析

### 1. 为什么 useEffect 里拿到的是旧值？

```javascript
const [count, setCount] = useState(0);

useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);  // 永远是 0！
  }, 1000);
  return () => clearInterval(timer);
}, []);  // 空依赖，只在 mount 时创建
```

Effect 创建时捕获了当时的 `count` 值（闭包）。解决方案：

```javascript
// 方案1：添加依赖
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);
  }, 1000);
  return () => clearInterval(timer);
}, [count]);  // count 变化时重新创建

// 方案2：使用 ref
const countRef = useRef(count);
countRef.current = count;

useEffect(() => {
  const timer = setInterval(() => {
    console.log(countRef.current);  // 始终是最新值
  }, 1000);
  return () => clearInterval(timer);
}, []);
```

### 2. 为什么会死循环？

```javascript
const [data, setData] = useState([]);

useEffect(() => {
  fetch('/api').then(res => setData(res));
}, [data]);  // data 变化 → 重新 fetch → data 变化 → ...
```

每次 `setData` 都会创建新数组，触发 Effect 重新执行。解决方案：

```javascript
useEffect(() => {
  fetch('/api').then(res => setData(res));
}, []);  // 只在 mount 时执行
```

### 3. 清理函数什么时候执行？

```javascript
useEffect(() => {
  console.log('create', count);
  return () => {
    console.log('destroy', count);  // 这里的 count 是上一次的值
  };
}, [count]);

// count: 0 → 1
// 输出：
// destroy 0  （先执行上一次的清理）
// create 1   （再执行这一次的创建）
```

## 九、调试技巧

```javascript
// 在这些位置打断点：

// 创建 Effect
mountEffectImpl     // react-reconciler/src/ReactFiberHooks.js
updateEffectImpl    // react-reconciler/src/ReactFiberHooks.js

// 执行 Effect
flushPassiveEffects // react-reconciler/src/ReactFiberWorkLoop.js
commitHookPassiveMountEffects  // 创建函数执行
commitHookPassiveUnmountEffects // 销毁函数执行
```

## 小结

本文深入分析了 useEffect 的实现原理：

1. **数据结构**：Effect 以环形链表存储在 Fiber.updateQueue
2. **标记系统**：通过 flags 区分是否需要执行
3. **依赖比较**：使用 Object.is 逐个比较
4. **异步执行**：通过 Scheduler 调度，在浏览器绘制后执行
5. **执行顺序**：先销毁后创建，子组件先于父组件

下一篇我们将分析 **Fiber 工作循环**，看看 React 是如何遍历和处理组件树的。

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[useState 的实现原理](https://juejin.cn/post/7588007285547302918)
> 
> 下一篇：[Fiber 工作循环](https://juejin.cn/post/7588061231589523497)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
