# React 19 源码揭秘（五）：Diff 算法的实现

> 本文深入源码，带你理解 React 如何高效地计算 UI 变化。

## 前言

Diff 算法是 React 性能优化的核心。当状态变化时，React 不会重建整个 DOM，而是通过 Diff 找出最小变更。

本文将从源码角度，解析 React 的 Diff 策略。

## 一、Diff 的三个假设

React 的 Diff 基于三个假设来优化性能：

1. **不同类型的元素产生不同的树**
2. **通过 key 标识哪些子元素是稳定的**
3. **只对同级元素进行 Diff**

这些假设让 Diff 的时间复杂度从 O(n³) 降到 O(n)。

## 二、Diff 入口

Diff 发生在 `reconcileChildFibers`：

```javascript
// ReactChildFiber.js
function reconcileChildFibers(returnFiber, currentFirstChild, newChild, lanes) {
  // 处理对象类型
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        // 单节点 Diff
        return reconcileSingleElement(returnFiber, currentFirstChild, newChild, lanes);
    }
    
    if (isArray(newChild)) {
      // 多节点 Diff
      return reconcileChildrenArray(returnFiber, currentFirstChild, newChild, lanes);
    }
  }
  
  // 文本节点
  if (typeof newChild === 'string' || typeof newChild === 'number') {
    return reconcileSingleTextNode(...);
  }
  
  // 删除所有旧节点
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

## 三、单节点 Diff

当新节点是单个元素时：

```javascript
function reconcileSingleElement(returnFiber, currentFirstChild, element, lanes) {
  const key = element.key;
  let child = currentFirstChild;
  
  // 遍历旧的子节点链表
  while (child !== null) {
    if (child.key === key) {
      // key 相同
      if (child.elementType === element.type) {
        // type 也相同，可以复用！
        deleteRemainingChildren(returnFiber, child.sibling);  // 删除其他兄弟
        const existing = useFiber(child, element.props);       // 复用 Fiber
        existing.return = returnFiber;
        return existing;
      }
      // key 相同但 type 不同，删除所有旧节点
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // key 不同，删除当前节点，继续找
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }
  
  // 没有可复用的，创建新 Fiber
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.return = returnFiber;
  return created;
}
```

### 单节点 Diff 流程图

```
旧: A → B → C
新: B

步骤：
1. A.key !== B.key → 删除 A
2. B.key === B.key && B.type === B.type → 复用 B，删除 C

结果：复用 B，删除 A 和 C
```

### 关键点

- **先比 key，再比 type**
- key 相同 + type 相同 = 复用
- key 相同 + type 不同 = 删除所有，重建
- key 不同 = 删除当前，继续找

## 四、多节点 Diff

当新节点是数组时，分三轮遍历：

```javascript
function reconcileChildrenArray(returnFiber, currentFirstChild, newChildren, lanes) {
  let oldFiber = currentFirstChild;
  let newIdx = 0;
  let lastPlacedIndex = 0;
  
  // ========== 第一轮：处理更新 ==========
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    const newFiber = updateSlot(returnFiber, oldFiber, newChildren[newIdx], lanes);
    
    if (newFiber === null) {
      // key 不同，跳出第一轮
      break;
    }
    
    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    oldFiber = oldFiber.sibling;
  }
  
  // ========== 第二轮：处理新增/删除 ==========
  if (newIdx === newChildren.length) {
    // 新节点遍历完，删除剩余旧节点
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }
  
  if (oldFiber === null) {
    // 旧节点遍历完，新增剩余新节点
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    }
    return resultingFirstChild;
  }
  
  // ========== 第三轮：处理移动 ==========
  // 将剩余旧节点放入 Map
  const existingChildren = mapRemainingChildren(oldFiber);
  
  for (; newIdx < newChildren.length; newIdx++) {
    // 从 Map 中查找可复用的节点
    const newFiber = updateFromMap(existingChildren, returnFiber, newIdx, newChildren[newIdx], lanes);
    
    if (newFiber !== null) {
      existingChildren.delete(newFiber.key ?? newIdx);
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
    }
  }
  
  // 删除 Map 中剩余的旧节点
  existingChildren.forEach(child => deleteChild(returnFiber, child));
  
  return resultingFirstChild;
}
```

### 第一轮：处理更新

从左到右遍历，比较 key：

```
旧: A → B → C → D
新: A → B → E → D

第一轮：
- A.key === A.key → 更新 A
- B.key === B.key → 更新 B
- C.key !== E.key → 跳出

结果：更新了 A、B
```

### 第二轮：处理新增/删除

```
情况1：新节点遍历完
旧: A → B → C
新: A → B
→ 删除 C

情况2：旧节点遍历完
旧: A → B
新: A → B → C
→ 新增 C
```

### 第三轮：处理移动

将剩余旧节点放入 Map，通过 key 查找复用：

```
旧: A → B → C → D
新: A → C → D → B

第一轮后：更新了 A
第三轮：
- Map: {B, C, D}
- 找 C → 复用
- 找 D → 复用
- 找 B → 复用，需要移动
```

## 五、移动判断：placeChild

```javascript
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;
  
  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // 旧位置在参照物左边，需要移动
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      // 不需要移动，更新参照物
      return oldIndex;
    }
  } else {
    // 新增节点
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

### 移动判断示例

```
旧: A(0) → B(1) → C(2) → D(3)
新: A → C → D → B

遍历新节点：
1. A: oldIndex=0, lastPlacedIndex=0 → 不移动, lastPlacedIndex=0
2. C: oldIndex=2, lastPlacedIndex=0 → 2>0, 不移动, lastPlacedIndex=2
3. D: oldIndex=3, lastPlacedIndex=2 → 3>2, 不移动, lastPlacedIndex=3
4. B: oldIndex=1, lastPlacedIndex=3 → 1<3, 需要移动！

结果：只移动 B
```

### 为什么这样判断？

`lastPlacedIndex` 是"参照物"，表示已处理节点中最右边的旧位置。

如果当前节点的旧位置在参照物左边，说明它需要移动到右边。

## 六、key 的重要性

### 没有 key 的问题

```jsx
// ❌ 没有 key
{items.map(item => <Item data={item} />)}

// React 使用 index 作为隐式 key
// 当列表顺序变化时，可能导致：
// 1. 状态错乱
// 2. 不必要的 DOM 操作
```

### 正确使用 key

```jsx
// ✅ 使用稳定的 id
{items.map(item => <Item key={item.id} data={item} />)}

// ❌ 不要用 index（除非列表不会变化）
{items.map((item, index) => <Item key={index} data={item} />)}
```

### key 的作用

```
旧: [A, B, C]  key: [1, 2, 3]
新: [C, A, B]  key: [3, 1, 2]

有 key：React 知道只是顺序变了，复用所有节点
无 key：React 认为每个位置都变了，更新所有节点
```

## 七、Diff 优化建议

1. **保持组件类型稳定**
   ```jsx
   // ❌ 条件渲染不同组件
   {isAdmin ? <AdminPanel /> : <UserPanel />}
   
   // ✅ 使用同一组件，通过 props 区分
   <Panel isAdmin={isAdmin} />
   ```

2. **使用稳定的 key**
   ```jsx
   // ❌ 随机 key
   <Item key={Math.random()} />
   
   // ✅ 稳定 key
   <Item key={item.id} />
   ```

3. **避免深层嵌套**
   ```jsx
   // ❌ 深层嵌套
   <A><B><C><D><E>...</E></D></C></B></A>
   
   // ✅ 扁平结构
   <A /><B /><C /><D /><E />
   ```

## 八、调试技巧

```javascript
// 在这些位置打断点：

// 单节点 Diff
reconcileSingleElement  // ReactChildFiber.js

// 多节点 Diff
reconcileChildrenArray  // ReactChildFiber.js

// 移动判断
placeChild              // ReactChildFiber.js

// 删除标记
deleteChild             // ReactChildFiber.js
```

## 小结

React Diff 算法的核心：

1. **三个假设**：同层比较、类型判断、key 标识
2. **单节点**：先比 key，再比 type
3. **多节点**：三轮遍历（更新 → 增删 → 移动）
4. **移动判断**：通过 lastPlacedIndex 参照物
5. **key 的作用**：帮助 React 识别节点身份

---

> 📦 配套源码：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 上一篇：[Fiber 工作循环](https://juejin.cn/post/7588061231589523497)
> 
> 下一篇：[Scheduler 时间切片的秘密](https://juejin.cn/post/7588064253635477558)
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
