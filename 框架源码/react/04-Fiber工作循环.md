# React 19 源码揭秘（四）：Fiber 工作循环

> 本文深入源码，带你理解 React 是如何遍历组件树、构建 Fiber 树的。

## 前言

当你调用 `setState` 后，React 是如何更新整个组件树的？答案就在 **Fiber 工作循环**。

本文将带你理解 React 渲染的核心流程：从触发更新到构建完整的 Fiber 树。

## 一、整体流程

```
setState
    │
    ▼
scheduleUpdateOnFiber ──► 标记更新
    │
    ▼
ensureRootIsScheduled ──► 调度任务
    │
    ▼
performConcurrentWorkOnRoot ──► 开始渲染
    │
    ▼
renderRootSync / renderRootConcurrent
    │
    ▼
workLoopSync / workLoopConcurrent ──► 工作循环
    │
    ▼
performUnitOfWork ──► 处理单个 Fiber
    │
    ├──► beginWork ──► 递（向下）
    │
    └──► completeWork ──► 归（向上）
```

## 二、工作循环核心

### 2.1 workLoopSync vs workLoopConcurrent

```javascript
// 同步模式：一口气干完
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// 并发模式：可中断
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

核心区别：
- `workLoopSync`：不检查时间，一次性完成所有工作
- `workLoopConcurrent`：每处理一个 Fiber 都检查 `shouldYield()`，超时则让出主线程

### 2.2 performUnitOfWork

这是工作循环的核心，处理单个 Fiber 节点：

```javascript
function performUnitOfWork(unitOfWork: Fiber): void {
  const current = unitOfWork.alternate;
  
  // 1. beginWork：处理当前节点，返回子节点
  let next = beginWork(current, unitOfWork, entangledRenderLanes);
  
  // 保存处理后的 props
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  
  if (next === null) {
    // 2. 没有子节点了，开始 completeWork
    completeUnitOfWork(unitOfWork);
  } else {
    // 3. 有子节点，继续向下
    workInProgress = next;
  }
}
```

## 三、beginWork 详解

`beginWork` 负责"递"阶段，根据 Fiber 类型执行不同的更新逻辑。

### 3.1 核心逻辑

```javascript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  
  // 1. 更新时的优化路径
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    
    if (oldProps !== newProps || hasLegacyContextChanged()) {
      didReceiveUpdate = true;
    } else {
      // 检查是否有待处理的更新
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes
      );
      if (!hasScheduledUpdateOrContext) {
        // 没有更新，走 bailout 优化
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes
        );
      }
    }
  }
  
  // 2. 根据组件类型分发处理
  switch (workInProgress.tag) {
    case FunctionComponent:
      return updateFunctionComponent(...);
    case ClassComponent:
      return updateClassComponent(...);
    case HostRoot:
      return updateHostRoot(...);
    case HostComponent:
      return updateHostComponent(...);
    case HostText:
      return updateHostText(...);
    // ... 更多类型
  }
}
```

### 3.2 不同组件类型的处理

| 组件类型 | tag | 处理函数 | 主要工作 |
|---------|-----|---------|---------|
| 函数组件 | FunctionComponent | updateFunctionComponent | 执行函数，处理 Hooks |
| 类组件 | ClassComponent | updateClassComponent | 调用生命周期，执行 render |
| 原生标签 | HostComponent | updateHostComponent | 处理 props，协调子节点 |
| 文本节点 | HostText | updateHostText | 处理文本内容 |
| 根节点 | HostRoot | updateHostRoot | 处理根节点更新 |

### 3.3 函数组件的处理

```javascript
function updateFunctionComponent(
  current,
  workInProgress,
  Component,
  nextProps,
  renderLanes
) {
  // 执行函数组件，内部会处理 Hooks
  let nextChildren = renderWithHooks(
    current,
    workInProgress,
    Component,
    nextProps,
    context,
    renderLanes
  );
  
  // 检查是否可以 bailout
  if (current !== null && !didReceiveUpdate) {
    bailoutHooks(current, workInProgress, renderLanes);
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  
  // 协调子节点（Diff 算法）
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  
  return workInProgress.child;
}
```

## 四、completeWork 详解

`completeWork` 负责"归"阶段，主要工作：
1. 创建/更新 DOM 节点
2. 收集 effect flags（副作用标记）
3. 属性冒泡（bubbleProperties）

### 4.1 completeUnitOfWork

```javascript
function completeUnitOfWork(unitOfWork: Fiber): void {
  let completedWork = unitOfWork;
  
  do {
    const current = completedWork.alternate;
    const returnFiber = completedWork.return;
    
    // 执行 completeWork
    let next = completeWork(current, completedWork, entangledRenderLanes);
    
    if (next !== null) {
      // completeWork 产生了新工作
      workInProgress = next;
      return;
    }
    
    // 处理兄弟节点
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      workInProgress = siblingFiber;
      return;
    }
    
    // 返回父节点
    completedWork = returnFiber;
    workInProgress = completedWork;
  } while (completedWork !== null);
  
  // 到达根节点，标记完成
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

### 4.2 HostComponent 的 completeWork

```javascript
case HostComponent: {
  popHostContext(workInProgress);
  const type = workInProgress.type;
  
  if (current !== null && workInProgress.stateNode != null) {
    // 更新：对比 props，标记更新
    updateHostComponent(current, workInProgress, type, newProps, renderLanes);
  } else {
    // 首次渲染：创建 DOM 节点
    const rootContainerInstance = getRootHostContainer();
    const instance = createInstance(
      type,
      newProps,
      rootContainerInstance,
      currentHostContext,
      workInProgress
    );
    
    // 将子 DOM 节点插入
    appendAllChildren(instance, workInProgress, false, false);
    
    // 保存 DOM 引用
    workInProgress.stateNode = instance;
    
    // 初始化 DOM 属性
    if (finalizeInitialChildren(instance, type, newProps, currentHostContext)) {
      markUpdate(workInProgress);
    }
  }
  
  // 属性冒泡
  bubbleProperties(workInProgress);
  return null;
}
```

### 4.3 bubbleProperties

属性冒泡是 React 的重要优化，将子树的 flags 和 lanes 收集到父节点：

```javascript
function bubbleProperties(completedWork: Fiber) {
  let newChildLanes = NoLanes;
  let subtreeFlags = NoFlags;
  
  let child = completedWork.child;
  while (child !== null) {
    // 合并子节点的 lanes
    newChildLanes = mergeLanes(
      newChildLanes,
      mergeLanes(child.lanes, child.childLanes)
    );
    
    // 合并子节点的 flags
    subtreeFlags |= child.subtreeFlags;
    subtreeFlags |= child.flags;
    
    child = child.sibling;
  }
  
  completedWork.subtreeFlags |= subtreeFlags;
  completedWork.childLanes = newChildLanes;
}
```

这样在 commit 阶段，可以通过 `subtreeFlags` 快速判断子树是否有副作用，避免无效遍历。

## 五、Fiber 树遍历顺序

React 采用深度优先遍历，遵循"递归"模式：

```
        App
       /   \
     Div   Span
     /
   Text

遍历顺序：
beginWork:    App → Div → Text
completeWork: Text → Div → Span → App
```

### 5.1 遍历示例

假设有如下组件结构：

```jsx
function App() {
  return (
    <div>
      <h1>Title</h1>
      <p>Content</p>
    </div>
  );
}
```

遍历过程：

```
1. beginWork(App)      → 返回 div
2. beginWork(div)      → 返回 h1
3. beginWork(h1)       → 返回 "Title"
4. beginWork("Title")  → 返回 null
5. completeWork("Title")
6. completeWork(h1)
7. beginWork(p)        → 返回 "Content"（h1 的 sibling）
8. beginWork("Content")→ 返回 null
9. completeWork("Content")
10. completeWork(p)
11. completeWork(div)
12. completeWork(App)
```

## 六、双缓冲机制

React 维护两棵 Fiber 树：
- `current`：当前屏幕上显示的
- `workInProgress`：正在构建的

```javascript
// 通过 alternate 互相引用
current.alternate = workInProgress;
workInProgress.alternate = current;

// commit 完成后交换
root.current = finishedWork;
```

这样可以：
1. 在内存中构建新树，不影响当前显示
2. 复用旧 Fiber 节点，减少内存分配
3. 支持并发渲染的中断和恢复

## 七、调试技巧

### 7.1 关键断点位置

```javascript
// 工作循环入口
ReactFiberWorkLoop.js → workLoopSync / workLoopConcurrent

// 单个 Fiber 处理
ReactFiberWorkLoop.js → performUnitOfWork

// 递阶段
ReactFiberBeginWork.js → beginWork

// 归阶段
ReactFiberCompleteWork.js → completeWork
```

### 7.2 观察 Fiber 结构

在断点处查看 `workInProgress` 对象：

```javascript
{
  tag: 0,              // FunctionComponent
  type: App,           // 组件函数
  stateNode: null,     // DOM 节点（HostComponent 才有）
  return: parentFiber, // 父节点
  child: childFiber,   // 第一个子节点
  sibling: null,       // 兄弟节点
  alternate: current,  // 对应的 current 节点
  flags: 0,            // 副作用标记
  lanes: 0,            // 优先级
}
```

## 八、总结

Fiber 工作循环的核心要点：

1. **两个循环**：`workLoopSync`（同步）和 `workLoopConcurrent`（可中断）
2. **两个阶段**：`beginWork`（递）和 `completeWork`（归）
3. **深度优先**：先处理子节点，再处理兄弟节点
4. **双缓冲**：current 和 workInProgress 两棵树交替
5. **属性冒泡**：子树的 flags 和 lanes 向上收集

理解了工作循环，你就掌握了 React 渲染的核心机制。

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[useEffect 的实现原理](https://juejin.cn/post/7588001624905973779)
> 
> 下一篇：[Diff 算法的实现](https://juejin.cn/post/7588001624905990163)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
