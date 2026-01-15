# React 19 源码全景图：从宏观到微观

> 本文是 React 源码系列的总览篇，帮你建立完整的知识框架，后续文章将逐一深入。

## 一、React 是什么？

一句话：**React 是一个将状态映射为 UI 的函数**。

```
UI = f(state)
```

当状态变化时，React 会：
1. 计算新的 UI（Reconciler）
2. 调度更新任务（Scheduler）
3. 将变化应用到 DOM（Renderer）

## 二、三大核心模块

```
┌─────────────────────────────────────────────────────────┐
│                        你的代码                          │
│            <App /> → useState → setState                │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   Scheduler 调度器                       │
│                                                         │
│  • 优先级管理（用户交互 > 动画 > 数据请求）               │
│  • 时间切片（5ms 一片，避免卡顿）                        │
│  • 任务队列（最小堆实现）                                │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  Reconciler 协调器                       │
│                                                         │
│  • Fiber 架构（可中断的链表结构）                        │
│  • Diff 算法（最小化 DOM 操作）                          │
│  • Hooks 系统（状态和副作用管理）                        │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   Renderer 渲染器                        │
│                                                         │
│  • ReactDOM（Web）                                      │
│  • React Native（移动端）                               │
│  • React Three Fiber（3D）                              │
└─────────────────────────────────────────────────────────┘
```

## 三、核心概念速览

### 1. Fiber

Fiber 是 React 的核心数据结构，每个组件对应一个 Fiber 节点：

```javascript
FiberNode {
  // 类型信息
  tag,              // 组件类型（函数组件=0，类组件=1，DOM=5）
  type,             // 组件函数或 DOM 标签
  
  // 树结构
  return,           // 父节点
  child,            // 第一个子节点
  sibling,          // 兄弟节点
  
  // 状态
  memoizedState,    // Hooks 链表
  memoizedProps,    // 上次的 props
  
  // 副作用
  flags,            // 标记（插入、更新、删除）
  
  // 双缓冲
  alternate,        // 另一棵树的对应节点
}
```

### 2. Lane（优先级）

React 19 使用 31 位二进制数表示优先级：

```javascript
SyncLane           = 0b0000000000000000000000000000010  // 同步（最高）
InputContinuousLane = 0b0000000000000000000000000001000  // 连续输入
DefaultLane        = 0b0000000000000000000000000100000  // 默认
TransitionLane     = 0b0000000000000000000000010000000  // 过渡
IdleLane           = 0b0010000000000000000000000000000  // 空闲（最低）
```

### 3. 双缓冲

React 维护两棵 Fiber 树：

- **current**：当前屏幕显示的
- **workInProgress**：正在构建的

更新完成后一行代码切换：`root.current = workInProgress`

## 四、渲染流程

### 完整流程图

```
setState() 
    │
    ▼
scheduleUpdateOnFiber()     ← 标记更新
    │
    ▼
ensureRootIsScheduled()     ← 确保调度
    │
    ▼
scheduleCallback()          ← Scheduler 调度
    │
    ▼
performConcurrentWorkOnRoot() ← 开始渲染
    │
    ├─────────────────────────────────────┐
    │         Render 阶段（可中断）         │
    │                                     │
    │  workLoopConcurrent()               │
    │      │                              │
    │      ▼                              │
    │  performUnitOfWork() ←──┐           │
    │      │                  │           │
    │      ▼                  │           │
    │  beginWork()            │ 循环      │
    │      │                  │           │
    │      ▼                  │           │
    │  completeWork() ────────┘           │
    │                                     │
    └─────────────────────────────────────┘
    │
    ▼
    ├─────────────────────────────────────┐
    │        Commit 阶段（不可中断）        │
    │                                     │
    │  commitBeforeMutationEffects()      │
    │      │                              │
    │      ▼                              │
    │  commitMutationEffects()  ← DOM操作 │
    │      │                              │
    │      ▼                              │
    │  root.current = finishedWork        │
    │      │                              │
    │      ▼                              │
    │  commitLayoutEffects()              │
    │                                     │
    └─────────────────────────────────────┘
    │
    ▼
flushPassiveEffects()       ← useEffect（异步）
```

### Render 阶段

**beginWork**（向下递归）：
- 根据组件类型处理（函数组件、类组件、DOM 元素）
- 调用组件函数，执行 Hooks
- Diff 子节点，创建/复用 Fiber

**completeWork**（向上回溯）：
- 创建 DOM 节点
- 收集副作用标记
- 冒泡 subtreeFlags

### Commit 阶段

三个子阶段：

| 阶段 | 时机 | 主要工作 |
|-----|------|---------|
| Before Mutation | DOM 操作前 | getSnapshotBeforeUpdate |
| Mutation | DOM 操作 | 增删改 DOM |
| Layout | DOM 操作后 | useLayoutEffect、componentDidMount |

## 五、Hooks 原理

Hooks 以链表形式存储在 Fiber 的 `memoizedState` 上：

```
Fiber.memoizedState → useState → useEffect → useMemo → null
```

### useState 流程

```
Mount:  mountState() → 创建 Hook → 初始化状态 → 返回 [state, setState]
Update: updateState() → 获取 Hook → 处理更新队列 → 返回 [newState, setState]
```

### useEffect 流程

```
Mount:  mountEffect() → 创建 Effect → 标记 Passive
Commit: flushPassiveEffects() → 执行销毁函数 → 执行创建函数
```

## 六、Diff 算法

React Diff 的三个策略：

1. **同层比较**：不跨层级移动节点
2. **类型判断**：类型不同直接替换
3. **key 标识**：通过 key 识别节点

### 单节点 Diff

```
key 相同 && type 相同 → 复用
key 相同 && type 不同 → 删除重建
key 不同 → 删除，继续找
```

### 多节点 Diff（三轮遍历）

```
第一轮：从左到右，处理更新
第二轮：处理新增或删除
第三轮：处理移动（Map 查找）
```

## 七、Scheduler 调度

### 优先级

```javascript
ImmediatePriority   // -1ms，立即执行
UserBlockingPriority // 250ms，用户交互
NormalPriority      // 5000ms，普通更新
LowPriority         // 10000ms，低优先级
IdlePriority        // 永不过期，空闲执行
```

### 时间切片

```javascript
function workLoop() {
  while (task && !shouldYield()) {  // 5ms 检查一次
    task = performTask(task);
  }
  if (task) {
    scheduleCallback(task);  // 还有任务，继续调度
  }
}
```

## 八、源码目录

```
packages/
├── react/                    # React API
│   └── src/ReactHooks.js     # Hooks 入口
│
├── react-reconciler/         # 协调器（核心）
│   └── src/
│       ├── ReactFiber.js           # Fiber 定义
│       ├── ReactFiberWorkLoop.js   # 工作循环
│       ├── ReactFiberBeginWork.js  # 递阶段
│       ├── ReactFiberCompleteWork.js # 归阶段
│       ├── ReactFiberHooks.js      # Hooks 实现
│       ├── ReactFiberCommitWork.js # Commit
│       ├── ReactFiberLane.js       # 优先级
│       └── ReactChildFiber.js      # Diff 算法
│
├── react-dom/                # DOM 渲染器
│
└── scheduler/                # 调度器
    └── src/
        ├── Scheduler.js          # 调度逻辑
        └── SchedulerMinHeap.js   # 最小堆
```

## 九、系列文章导航

| 序号 | 主题 | 核心内容 |
|-----|------|---------|
| 00 | [调试环境搭建](https://juejin.cn/post/7588149079400710187) | 项目介绍、快速开始 |
| 01 | 架构总览（本文） | 三大模块、核心概念 |
| 02 | [useState 原理](https://juejin.cn/post/7588007285547302918) | Hook 链表、更新队列 |
| 03 | [useEffect 原理](https://juejin.cn/post/7588001624905973779) | Effect 链表、执行时机 |
| 04 | [Fiber 工作循环](https://juejin.cn/post/7588061231589523497) | beginWork、completeWork |
| 05 | [Diff 算法](https://juejin.cn/post/7588001624905990163) | 单节点、多节点 Diff |
| 06 | [Scheduler 调度器](https://juejin.cn/post/7588064253635477558) | 优先级、时间切片 |
| 07 | [Commit 阶段](https://juejin.cn/post/7588061231589556265) | 三个子阶段、DOM 操作 |

## 十、学习建议

1. **先跑起来**：clone react-debug，打断点调试
2. **从 useState 开始**：最简单也最核心
3. **画流程图**：边看边画，加深理解
4. **写测试组件**：验证你的理解

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[React 源码调试环境搭建](https://juejin.cn/post/7588149079400710187)
> 
> 下一篇：[useState 的实现原理](https://juejin.cn/post/7588007285547302918)
