# React 19 源码揭秘（七）：Commit 阶段与 DOM 操作

> 本文深入 Commit 阶段源码，看看 React 是如何将 Fiber 树的变化同步到 DOM 的。

## 前言

Render 阶段计算出了"要做什么"，Commit 阶段则是"真正去做"。

这个阶段会：
- 执行 DOM 操作（增删改）
- 调用生命周期方法
- 执行 useLayoutEffect 和 useEffect

## 一、Commit 阶段概览

Commit 阶段分为三个子阶段：

```
┌─────────────────────────────────────────────────────────┐
│                    Commit 阶段                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Before Mutation ──► Mutation ──► Layout               │
│        │                │            │                  │
│        ▼                ▼            ▼                  │
│  getSnapshot       DOM 操作     useLayoutEffect        │
│  BeforeUpdate                   componentDidMount      │
│                                                         │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
              ┌─────────────────────────┐
              │   Passive Effects       │
              │      (异步)             │
              │                         │
              │     useEffect           │
              └─────────────────────────┘
```

## 二、commitRoot 入口

```javascript
function commitRoot(root, recoverableErrors, transitions, ...) {
  const finishedWork = root.finishedWork;
  
  // 1. 调度 useEffect（异步）
  if ((finishedWork.subtreeFlags & PassiveMask) !== NoFlags) {
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      return null;
    });
  }
  
  // 2. 检查是否有副作用需要处理
  const subtreeHasEffects = (finishedWork.subtreeFlags & 
    (BeforeMutationMask | MutationMask | LayoutMask | PassiveMask)) !== NoFlags;
  
  if (subtreeHasEffects || rootHasEffect) {
    // 3. Before Mutation 阶段
    commitBeforeMutationEffects(root, finishedWork);
    
    // 4. Mutation 阶段
    commitMutationEffects(root, finishedWork, lanes);
    
    // 5. 切换 Fiber 树！
    root.current = finishedWork;
    
    // 6. Layout 阶段
    commitLayoutEffects(finishedWork, root, lanes);
  } else {
    root.current = finishedWork;
  }
  
  // 7. 确保后续更新被调度
  ensureRootIsScheduled(root);
}
```

## 三、Before Mutation 阶段

DOM 变更前，读取 DOM 状态。

```javascript
function commitBeforeMutationEffects(root, firstChild) {
  nextEffect = firstChild;
  commitBeforeMutationEffects_begin();
}

function commitBeforeMutationEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    
    // 处理 Snapshot 标记
    if ((fiber.flags & Snapshot) !== NoFlags) {
      switch (fiber.tag) {
        case ClassComponent: {
          const instance = fiber.stateNode;
          // 调用 getSnapshotBeforeUpdate
          const snapshot = instance.getSnapshotBeforeUpdate(
            fiber.elementType === fiber.type
              ? prevProps
              : resolveDefaultProps(fiber.type, prevProps),
            prevState,
          );
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
          break;
        }
        case HostRoot: {
          // 清空容器
          clearContainer(root.containerInfo);
          break;
        }
      }
    }
    
    nextEffect = fiber.return;
  }
}
```

### 主要工作

- 调用类组件的 `getSnapshotBeforeUpdate`
- 清空根容器（首次渲染时）

## 四、Mutation 阶段

执行 DOM 操作，这是最核心的阶段。

```javascript
function commitMutationEffects(root, finishedWork, lanes) {
  commitMutationEffectsOnFiber(finishedWork, root, lanes);
}

function commitMutationEffectsOnFiber(finishedWork, root, lanes) {
  const flags = finishedWork.flags;
  
  // 1. 处理 Ref 卸载
  if (flags & Ref) {
    const current = finishedWork.alternate;
    if (current !== null) {
      commitDetachRef(current);
    }
  }
  
  // 2. 根据 flags 执行不同操作
  const primaryFlags = flags & (Placement | Update | ChildDeletion);
  
  switch (primaryFlags) {
    case Placement: {
      // 插入 DOM
      commitPlacement(finishedWork);
      finishedWork.flags &= ~Placement;
      break;
    }
    case Update: {
      // 更新 DOM
      commitWork(finishedWork);
      break;
    }
    case Placement | Update: {
      // 先插入，再更新
      commitPlacement(finishedWork);
      finishedWork.flags &= ~Placement;
      commitWork(finishedWork);
      break;
    }
    case ChildDeletion: {
      // 删除子节点
      commitDeletions(finishedWork.deletions, finishedWork);
      break;
    }
  }
}
```

### commitPlacement（插入 DOM）

```javascript
function commitPlacement(finishedWork) {
  // 1. 找到最近的 Host 父节点
  const parentFiber = getHostParentFiber(finishedWork);
  const parentDOM = parentFiber.stateNode;
  
  // 2. 找到插入位置（兄弟节点）
  const before = getHostSibling(finishedWork);
  
  // 3. 插入 DOM
  if (before) {
    insertBefore(parentDOM, finishedWork.stateNode, before);
  } else {
    appendChild(parentDOM, finishedWork.stateNode);
  }
}
```

### commitWork（更新 DOM）

```javascript
function commitWork(finishedWork) {
  switch (finishedWork.tag) {
    case HostComponent: {
      const instance = finishedWork.stateNode;
      if (instance !== null) {
        const newProps = finishedWork.memoizedProps;
        const oldProps = finishedWork.alternate?.memoizedProps;
        const type = finishedWork.type;
        
        // 更新 DOM 属性
        commitUpdate(instance, type, oldProps, newProps, finishedWork);
      }
      break;
    }
    case HostText: {
      const textInstance = finishedWork.stateNode;
      const newText = finishedWork.memoizedProps;
      
      // 更新文本内容
      commitTextUpdate(textInstance, newText);
      break;
    }
    case FunctionComponent: {
      // 执行 useInsertionEffect 和 useLayoutEffect 的销毁函数
      commitHookEffectListUnmount(HookInsertion | HookHasEffect, finishedWork);
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      break;
    }
  }
}
```

### commitDeletions（删除）

```javascript
function commitDeletions(deletions, parentFiber) {
  for (let i = 0; i < deletions.length; i++) {
    const childToDelete = deletions[i];
    
    // 递归卸载
    commitUnmount(childToDelete);
    
    // 从 DOM 中移除
    removeChild(parentDOM, childToDelete.stateNode);
  }
}

function commitUnmount(fiber) {
  switch (fiber.tag) {
    case FunctionComponent: {
      // 执行 useEffect 和 useLayoutEffect 的销毁函数
      commitHookEffectListUnmount(HookPassive, fiber);
      commitHookEffectListUnmount(HookLayout, fiber);
      break;
    }
    case ClassComponent: {
      // 调用 componentWillUnmount
      const instance = fiber.stateNode;
      instance.componentWillUnmount();
      break;
    }
  }
  
  // 递归处理子节点
  let child = fiber.child;
  while (child !== null) {
    commitUnmount(child);
    child = child.sibling;
  }
}
```

## 五、切换 Fiber 树

在 Mutation 和 Layout 之间，有一行关键代码：

```javascript
root.current = finishedWork;
```

这行代码将 `workInProgress` 树变成 `current` 树，完成双缓冲切换。

### 为什么在这个时机？

- **Mutation 阶段**：需要访问旧的 DOM 状态（componentWillUnmount）
- **Layout 阶段**：需要访问新的 DOM 状态（componentDidMount）

所以切换发生在两者之间。

## 六、Layout 阶段

DOM 变更后，可以安全读取 DOM。

```javascript
function commitLayoutEffects(finishedWork, root, lanes) {
  commitLayoutEffectOnFiber(root, finishedWork.alternate, finishedWork, lanes);
}

function commitLayoutEffectOnFiber(root, current, finishedWork, lanes) {
  const flags = finishedWork.flags;
  
  switch (finishedWork.tag) {
    case FunctionComponent: {
      // 执行 useLayoutEffect 的创建函数
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      break;
    }
    case ClassComponent: {
      const instance = finishedWork.stateNode;
      if (current === null) {
        // 首次渲染
        instance.componentDidMount();
      } else {
        // 更新
        const prevProps = current.memoizedProps;
        const prevState = current.memoizedState;
        instance.componentDidUpdate(prevProps, prevState, instance.__reactInternalSnapshotBeforeUpdate);
      }
      
      // 处理 setState 回调
      commitUpdateQueue(finishedWork, finishedWork.updateQueue, instance);
      break;
    }
    case HostRoot: {
      // 处理 ReactDOM.render 回调
      commitUpdateQueue(finishedWork, finishedWork.updateQueue, null);
      break;
    }
  }
  
  // 绑定 Ref
  if (flags & Ref) {
    commitAttachRef(finishedWork);
  }
}
```

### commitAttachRef

```javascript
function commitAttachRef(finishedWork) {
  const ref = finishedWork.ref;
  if (ref !== null) {
    const instance = finishedWork.stateNode;
    
    if (typeof ref === 'function') {
      ref(instance);
    } else {
      ref.current = instance;
    }
  }
}
```

## 七、Passive Effects（useEffect）

useEffect 是异步执行的，在 Commit 阶段只是调度：

```javascript
// commitRoot 中
scheduleCallback(NormalSchedulerPriority, () => {
  flushPassiveEffects();
  return null;
});
```

### flushPassiveEffects

```javascript
function flushPassiveEffects() {
  if (rootWithPendingPassiveEffects !== null) {
    // 1. 执行所有销毁函数
    commitPassiveUnmountEffects(root.current);
    
    // 2. 执行所有创建函数
    commitPassiveMountEffects(root, root.current);
  }
}
```

### 执行顺序

```
组件树：
  App
   └── Parent
        └── Child

Mount 时：
1. Child useLayoutEffect 创建
2. Parent useLayoutEffect 创建
3. App useLayoutEffect 创建
4. ─── 浏览器绘制 ───
5. Child useEffect 创建
6. Parent useEffect 创建
7. App useEffect 创建
```

## 八、Flags 标记

```javascript
// ReactFiberFlags.js
export const NoFlags = 0b0000000000000000000000000000;
export const Placement = 0b0000000000000000000000000010;      // 插入
export const Update = 0b0000000000000000000000000100;         // 更新
export const ChildDeletion = 0b0000000000000000000000010000;  // 删除子节点
export const Snapshot = 0b0000000100000000000000000000;       // getSnapshotBeforeUpdate
export const Passive = 0b0000100000000000000000000000;        // useEffect
export const Ref = 0b0001000000000000000000000000;            // ref

// 阶段掩码
export const BeforeMutationMask = Snapshot;
export const MutationMask = Placement | Update | ChildDeletion | Ref;
export const LayoutMask = Update | Callback | Ref;
export const PassiveMask = Passive | ChildDeletion;
```

## 九、调试技巧

```javascript
// 在这些位置打断点：

// 入口
commitRoot                    // ReactFiberWorkLoop.js

// Before Mutation
commitBeforeMutationEffects   // ReactFiberCommitWork.js

// Mutation
commitMutationEffects         // ReactFiberCommitWork.js
commitPlacement               // DOM 插入
commitWork                    // DOM 更新
commitDeletions               // DOM 删除

// Layout
commitLayoutEffects           // ReactFiberCommitWork.js

// Passive
flushPassiveEffects           // ReactFiberWorkLoop.js
```

## 小结

Commit 阶段的核心流程：

1. **Before Mutation**：读取 DOM 状态，getSnapshotBeforeUpdate
2. **Mutation**：执行 DOM 操作（增删改）
3. **切换 Fiber 树**：root.current = finishedWork
4. **Layout**：useLayoutEffect、componentDidMount/Update、绑定 ref
5. **Passive**：异步执行 useEffect

关键点：
- Commit 阶段不可中断
- useLayoutEffect 同步执行，useEffect 异步执行
- Fiber 树切换发生在 Mutation 和 Layout 之间

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[Scheduler 时间切片的秘密](https://juejin.cn/post/7588064253635477558)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
