# Vue.js 源码揭秘（三）：虚拟 DOM 与 Diff 算法

> 本文深入 runtime-core 源码，解析 VNode 结构和 patch 算法的实现。

## 一、VNode 结构

```typescript
// packages/runtime-core/src/vnode.ts
interface VNode {
  __v_isVNode: true
  type: VNodeTypes              // 节点类型
  props: VNodeProps | null      // 属性
  key: string | number | null   // diff key
  children: VNodeChildren       // 子节点
  
  // DOM 相关
  el: HostElement | null        // 真实 DOM
  anchor: HostNode | null       // Fragment 锚点
  
  // 优化标记
  shapeFlag: number             // 节点形状
  patchFlag: number             // patch 优化标记
  dynamicProps: string[] | null // 动态属性
  dynamicChildren: VNode[] | null // 动态子节点
  
  // 组件相关
  component: ComponentInternalInstance | null
  suspense: SuspenseBoundary | null
  
  // 其他
  ref: VNodeRef | null
  scopeId: string | null
}
```

## 二、创建 VNode

### 2.1 createVNode

```typescript
// packages/runtime-core/src/vnode.ts
export function createVNode(
  type: VNodeTypes,
  props?: VNodeProps | null,
  children?: unknown,
  patchFlag?: number,
  dynamicProps?: string[] | null
): VNode {
  // 标准化 class 和 style
  if (props) {
    props = guardReactiveProps(props)
    let { class: klass, style } = props
    if (klass && !isString(klass)) {
      props.class = normalizeClass(klass)
    }
    if (isObject(style)) {
      props.style = normalizeStyle(style)
    }
  }
  
  // 计算 shapeFlag
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isTeleport(type)
        ? ShapeFlags.TELEPORT
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0
  
  return createBaseVNode(
    type,
    props,
    children,
    patchFlag,
    dynamicProps,
    shapeFlag
  )
}

function createBaseVNode(
  type,
  props,
  children,
  patchFlag,
  dynamicProps,
  shapeFlag
): VNode {
  const vnode: VNode = {
    __v_isVNode: true,
    type,
    props,
    key: props?.key ?? null,
    children,
    el: null,
    anchor: null,
    shapeFlag,
    patchFlag,
    dynamicProps,
    dynamicChildren: null,
    component: null,
    suspense: null,
    ref: props?.ref ?? null,
    scopeId: currentScopeId
  }
  
  // 标准化子节点
  if (children) {
    vnode.shapeFlag |= isString(children)
      ? ShapeFlags.TEXT_CHILDREN
      : ShapeFlags.ARRAY_CHILDREN
  }
  
  // 收集动态子节点
  if (currentBlock && patchFlag > 0) {
    currentBlock.push(vnode)
  }
  
  return vnode
}
```

### 2.2 h 函数

```typescript
// packages/runtime-core/src/h.ts
export function h(type, propsOrChildren?, children?) {
  const l = arguments.length
  
  if (l === 2) {
    if (isObject(propsOrChildren) && !isArray(propsOrChildren)) {
      // h('div', { id: 'foo' })
      if (isVNode(propsOrChildren)) {
        return createVNode(type, null, [propsOrChildren])
      }
      return createVNode(type, propsOrChildren)
    } else {
      // h('div', [child1, child2])
      return createVNode(type, null, propsOrChildren)
    }
  } else {
    if (l > 3) {
      children = Array.from(arguments).slice(2)
    } else if (l === 3 && isVNode(children)) {
      children = [children]
    }
    return createVNode(type, propsOrChildren, children)
  }
}
```

## 三、patch 算法

### 3.1 patch 入口

```typescript
// packages/runtime-core/src/renderer.ts
const patch = (
  n1: VNode | null,  // 旧节点
  n2: VNode,         // 新节点
  container,
  anchor = null,
  parentComponent = null,
  parentSuspense = null,
  namespace,
  slotScopeIds,
  optimized
) => {
  // 相同节点，跳过
  if (n1 === n2) return
  
  // 类型不同，卸载旧节点
  if (n1 && !isSameVNodeType(n1, n2)) {
    anchor = getNextHostNode(n1)
    unmount(n1, parentComponent, parentSuspense, true)
    n1 = null
  }
  
  const { type, ref, shapeFlag } = n2
  
  switch (type) {
    case Text:
      processText(n1, n2, container, anchor)
      break
    case Comment:
      processCommentNode(n1, n2, container, anchor)
      break
    case Static:
      if (n1 == null) {
        mountStaticNode(n2, container, anchor, namespace)
      }
      break
    case Fragment:
      processFragment(n1, n2, container, anchor, ...)
      break
    default:
      if (shapeFlag & ShapeFlags.ELEMENT) {
        processElement(n1, n2, container, anchor, ...)
      } else if (shapeFlag & ShapeFlags.COMPONENT) {
        processComponent(n1, n2, container, anchor, ...)
      } else if (shapeFlag & ShapeFlags.TELEPORT) {
        type.process(n1, n2, container, anchor, ...)
      } else if (shapeFlag & ShapeFlags.SUSPENSE) {
        type.process(n1, n2, container, anchor, ...)
      }
  }
  
  // 设置 ref
  if (ref != null && parentComponent) {
    setRef(ref, n1?.ref, parentSuspense, n2 || n1, !n2)
  }
}
```

### 3.2 isSameVNodeType

```typescript
export function isSameVNodeType(n1: VNode, n2: VNode): boolean {
  return n1.type === n2.type && n1.key === n2.key
}
```

## 四、Element Diff

### 4.1 processElement

```typescript
const processElement = (n1, n2, container, anchor, ...) => {
  if (n1 == null) {
    // 挂载
    mountElement(n2, container, anchor, ...)
  } else {
    // 更新
    patchElement(n1, n2, parentComponent, ...)
  }
}
```

### 4.2 mountElement

```typescript
const mountElement = (vnode, container, anchor, ...) => {
  let el
  const { props, shapeFlag, transition, dirs } = vnode
  
  // 创建元素
  el = vnode.el = hostCreateElement(vnode.type, namespace, props?.is, props)
  
  // 处理子节点
  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    hostSetElementText(el, vnode.children)
  } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
    mountChildren(vnode.children, el, null, ...)
  }
  
  // 处理指令
  if (dirs) {
    invokeDirectiveHook(vnode, null, parentComponent, 'created')
  }
  
  // 处理 props
  if (props) {
    for (const key in props) {
      if (!isReservedProp(key)) {
        hostPatchProp(el, key, null, props[key], namespace, parentComponent)
      }
    }
  }
  
  // 插入 DOM
  hostInsert(el, container, anchor)
  
  // 触发钩子
  if (dirs) {
    invokeDirectiveHook(vnode, null, parentComponent, 'mounted')
  }
}
```

### 4.3 patchElement

```typescript
const patchElement = (n1, n2, parentComponent, ...) => {
  const el = (n2.el = n1.el)
  let { patchFlag, dynamicChildren, dirs } = n2
  const oldProps = n1.props || EMPTY_OBJ
  const newProps = n2.props || EMPTY_OBJ
  
  // 触发 beforeUpdate 钩子
  if (dirs) {
    invokeDirectiveHook(n2, n1, parentComponent, 'beforeUpdate')
  }
  
  // 优化路径：只更新动态子节点
  if (dynamicChildren) {
    patchBlockChildren(n1.dynamicChildren, dynamicChildren, el, ...)
  } else if (!optimized) {
    // 全量 diff
    patchChildren(n1, n2, el, null, ...)
  }
  
  // 更新 props
  if (patchFlag > 0) {
    if (patchFlag & PatchFlags.FULL_PROPS) {
      patchProps(el, oldProps, newProps, parentComponent, namespace)
    } else {
      if (patchFlag & PatchFlags.CLASS) {
        if (oldProps.class !== newProps.class) {
          hostPatchProp(el, 'class', null, newProps.class, namespace)
        }
      }
      if (patchFlag & PatchFlags.STYLE) {
        hostPatchProp(el, 'style', oldProps.style, newProps.style, namespace)
      }
      if (patchFlag & PatchFlags.PROPS) {
        for (const key of n2.dynamicProps) {
          const prev = oldProps[key]
          const next = newProps[key]
          if (next !== prev || key === 'value') {
            hostPatchProp(el, key, prev, next, namespace, parentComponent)
          }
        }
      }
      if (patchFlag & PatchFlags.TEXT) {
        if (n1.children !== n2.children) {
          hostSetElementText(el, n2.children)
        }
      }
    }
  }
}
```

## 五、Children Diff

### 5.1 patchChildren

```typescript
const patchChildren = (n1, n2, container, anchor, ...) => {
  const c1 = n1 && n1.children
  const c2 = n2.children
  const prevShapeFlag = n1 ? n1.shapeFlag : 0
  const { shapeFlag } = n2
  
  // 新子节点是文本
  if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      unmountChildren(c1, parentComponent, parentSuspense)
    }
    if (c2 !== c1) {
      hostSetElementText(container, c2)
    }
  } else {
    // 新子节点是数组或空
    if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        // 两个数组，核心 diff
        patchKeyedChildren(c1, c2, container, anchor, ...)
      } else {
        // 卸载旧子节点
        unmountChildren(c1, parentComponent, parentSuspense, true)
      }
    } else {
      if (prevShapeFlag & ShapeFlags.TEXT_CHILDREN) {
        hostSetElementText(container, '')
      }
      if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        mountChildren(c2, container, anchor, ...)
      }
    }
  }
}
```

### 5.2 patchKeyedChildren（核心 Diff）

```typescript
const patchKeyedChildren = (c1, c2, container, parentAnchor, ...) => {
  let i = 0
  const l2 = c2.length
  let e1 = c1.length - 1  // 旧子节点结束索引
  let e2 = l2 - 1         // 新子节点结束索引
  
  // 1. 从头部开始同步
  while (i <= e1 && i <= e2) {
    const n1 = c1[i]
    const n2 = c2[i]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container, null, ...)
    } else {
      break
    }
    i++
  }
  
  // 2. 从尾部开始同步
  while (i <= e1 && i <= e2) {
    const n1 = c1[e1]
    const n2 = c2[e2]
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container, null, ...)
    } else {
      break
    }
    e1--
    e2--
  }
  
  // 3. 旧节点遍历完，新节点有剩余 → 挂载
  if (i > e1) {
    if (i <= e2) {
      const nextPos = e2 + 1
      const anchor = nextPos < l2 ? c2[nextPos].el : parentAnchor
      while (i <= e2) {
        patch(null, c2[i], container, anchor, ...)
        i++
      }
    }
  }
  
  // 4. 新节点遍历完，旧节点有剩余 → 卸载
  else if (i > e2) {
    while (i <= e1) {
      unmount(c1[i], parentComponent, parentSuspense, true)
      i++
    }
  }
  
  // 5. 中间部分乱序 → 最长递增子序列
  else {
    const s1 = i  // 旧子节点开始索引
    const s2 = i  // 新子节点开始索引
    
    // 5.1 建立新子节点 key → index 映射
    const keyToNewIndexMap = new Map()
    for (i = s2; i <= e2; i++) {
      const nextChild = c2[i]
      if (nextChild.key != null) {
        keyToNewIndexMap.set(nextChild.key, i)
      }
    }
    
    // 5.2 遍历旧子节点，尝试 patch 或卸载
    let j
    let patched = 0
    const toBePatched = e2 - s2 + 1
    let moved = false
    let maxNewIndexSoFar = 0
    
    // newIndex → oldIndex 映射
    const newIndexToOldIndexMap = new Array(toBePatched).fill(0)
    
    for (i = s1; i <= e1; i++) {
      const prevChild = c1[i]
      
      if (patched >= toBePatched) {
        // 新节点已全部 patch，剩余旧节点卸载
        unmount(prevChild, parentComponent, parentSuspense, true)
        continue
      }
      
      let newIndex
      if (prevChild.key != null) {
        newIndex = keyToNewIndexMap.get(prevChild.key)
      } else {
        // 无 key，遍历查找
        for (j = s2; j <= e2; j++) {
          if (newIndexToOldIndexMap[j - s2] === 0 &&
              isSameVNodeType(prevChild, c2[j])) {
            newIndex = j
            break
          }
        }
      }
      
      if (newIndex === undefined) {
        // 旧节点在新列表中不存在，卸载
        unmount(prevChild, parentComponent, parentSuspense, true)
      } else {
        newIndexToOldIndexMap[newIndex - s2] = i + 1
        
        if (newIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = newIndex
        } else {
          moved = true
        }
        
        patch(prevChild, c2[newIndex], container, null, ...)
        patched++
      }
    }
    
    // 5.3 移动和挂载
    const increasingNewIndexSequence = moved
      ? getSequence(newIndexToOldIndexMap)
      : EMPTY_ARR
    
    j = increasingNewIndexSequence.length - 1
    
    for (i = toBePatched - 1; i >= 0; i--) {
      const nextIndex = s2 + i
      const nextChild = c2[nextIndex]
      const anchor = nextIndex + 1 < l2 ? c2[nextIndex + 1].el : parentAnchor
      
      if (newIndexToOldIndexMap[i] === 0) {
        // 新节点，挂载
        patch(null, nextChild, container, anchor, ...)
      } else if (moved) {
        if (j < 0 || i !== increasingNewIndexSequence[j]) {
          // 需要移动
          move(nextChild, container, anchor, MoveType.REORDER)
        } else {
          j--
        }
      }
    }
  }
}
```

## 六、最长递增子序列

```typescript
// 获取最长递增子序列的索引
function getSequence(arr: number[]): number[] {
  const p = arr.slice()
  const result = [0]
  let i, j, u, v, c
  const len = arr.length
  
  for (i = 0; i < len; i++) {
    const arrI = arr[i]
    if (arrI !== 0) {
      j = result[result.length - 1]
      if (arr[j] < arrI) {
        p[i] = j
        result.push(i)
        continue
      }
      
      // 二分查找
      u = 0
      v = result.length - 1
      while (u < v) {
        c = (u + v) >> 1
        if (arr[result[c]] < arrI) {
          u = c + 1
        } else {
          v = c
        }
      }
      
      if (arrI < arr[result[u]]) {
        if (u > 0) {
          p[i] = result[u - 1]
        }
        result[u] = i
      }
    }
  }
  
  // 回溯
  u = result.length
  v = result[u - 1]
  while (u-- > 0) {
    result[u] = v
    v = p[v]
  }
  
  return result
}
```

## 七、Diff 流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    patchKeyedChildren                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  旧: [a, b, c, d, e, f, g]                                  │
│  新: [a, b, e, c, d, h, f, g]                               │
│                                                             │
│  Step 1: 头部同步                                           │
│  ─────────────────                                          │
│  旧: [a, b, | c, d, e, f, g]                                │
│  新: [a, b, | e, c, d, h, f, g]                             │
│       ↑  ↑                                                  │
│       patch                                                 │
│                                                             │
│  Step 2: 尾部同步                                           │
│  ─────────────────                                          │
│  旧: [a, b, | c, d, e, | f, g]                              │
│  新: [a, b, | e, c, d, h, | f, g]                           │
│                           ↑  ↑                              │
│                           patch                             │
│                                                             │
│  Step 3: 中间乱序                                           │
│  ─────────────────                                          │
│  旧: [c, d, e]                                              │
│  新: [e, c, d, h]                                           │
│                                                             │
│  - 建立 keyToNewIndexMap: { e: 0, c: 1, d: 2, h: 3 }       │
│  - newIndexToOldIndexMap: [3, 1, 2, 0]                     │
│  - 最长递增子序列: [1, 2] → 索引 [1, 2]                     │
│  - c, d 不需要移动                                          │
│  - e 需要移动到最前面                                       │
│  - h 是新节点，需要挂载                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 八、小结

Vue3 Diff 算法的核心：

1. **双端对比**：先从头尾同步相同节点
2. **最长递增子序列**：最小化 DOM 移动操作
3. **key 的作用**：快速定位可复用节点
4. **PatchFlags**：编译时标记，跳过静态内容
5. **Block Tree**：只追踪动态节点

---

> 📦 源码地址：[github.com/vuejs/core](https://github.com/vuejs/core)
> 
> 下一篇：组件系统详解
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
