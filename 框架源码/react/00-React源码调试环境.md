# React 源码调试环境搭建（v18 & v19）

> 工欲善其事，必先利其器。本文提供一个开箱即用的 React 源码调试项目，支持 v18 和 v19 两个版本。

## 项目介绍

[react-debug](https://github.com/220529/react-debug) 是一个专门用于调试 React 源码的项目，特点：

- ✅ **开箱即用**：clone 后直接 `pnpm dev` 即可调试
- ✅ **双版本支持**：v18.1.0 和 v19.0.0 两个分支
- ✅ **源码映射**：通过 webpack alias 直接引用源码
- ✅ **调试示例**：内置多个测试组件

## 快速开始

```bash
# 克隆项目
git clone https://github.com/220529/react-debug.git
cd react-debug

# 安装依赖
pnpm install

# 启动开发服务器
pnpm dev
```

### 切换版本

```bash
# 调试 React 19（推荐）
git checkout v19.0.0

# 调试 React 18
git checkout v18.1.0
```

## 项目结构

```
react-debug/
├── src/
│   ├── packages/           # React 源码（核心！）
│   │   ├── react/          # React API
│   │   ├── react-dom/      # DOM 渲染器
│   │   ├── react-reconciler/ # 协调器
│   │   ├── scheduler/      # 调度器
│   │   └── shared/         # 共享工具
│   │
│   ├── components/         # 调试用的测试组件
│   │   ├── counter/        # 计数器（useState）
│   │   ├── useEffect/      # Effect 测试
│   │   ├── setState/       # 批量更新测试
│   │   └── ...
│   │
│   ├── docs/               # 源码分析文档
│   └── App.js              # 入口组件
│
├── config/
│   ├── webpack.config.js   # webpack 配置（alias）
│   └── paths.js            # 路径配置
│
└── package.json
```

## 核心配置

项目通过 webpack alias 将 `react`、`react-dom` 等包指向本地源码：

```javascript
// config/webpack.config.js
alias: {
  "react": path.join(paths.reactSrc, "react"),
  "react-dom": path.join(paths.reactSrc, "react-dom"),
  "react-reconciler": path.join(paths.reactSrc, "react-reconciler"),
  "scheduler": path.join(paths.reactSrc, "scheduler"),
  "shared": path.join(paths.reactSrc, "shared"),
  // ...
}
```

这样 `import React from 'react'` 实际引用的是 `src/packages/react`。

## 调试技巧

### 1. 入口断点

在 Chrome DevTools 中打开 Sources 面板，找到这些文件打断点：

| 场景 | 文件 | 函数 |
|-----|------|------|
| 初始化 | `react-dom/src/client/ReactDOMRoot.js` | `createRoot` |
| 触发更新 | `react-reconciler/src/ReactFiberHooks.js` | `dispatchSetState` |
| 调度 | `react-reconciler/src/ReactFiberWorkLoop.js` | `scheduleUpdateOnFiber` |
| 渲染 | `react-reconciler/src/ReactFiberWorkLoop.js` | `performUnitOfWork` |
| 提交 | `react-reconciler/src/ReactFiberCommitWork.js` | `commitRoot` |

### 2. 使用 Counter 组件

项目内置了一个简单的 Counter 组件，非常适合调试：

```javascript
// src/components/counter/index.js
import { useState, useEffect } from "react";

const Counter = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    console.log("effect...start", count);
    return () => {
      console.log("effect...end", count);
    };
  }, [count]);

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
};
```

在 `useState(0)` 处打断点，点击按钮，就能完整追踪更新流程。

### 3. 查看 Fiber 树

在控制台输入：

```javascript
// 获取根 Fiber
const root = document.getElementById('root')._reactRootContainer._internalRoot;
console.log(root.current);  // 当前 Fiber 树
```

## 版本差异

| 特性 | v18.1.0 | v19.0.0 |
|-----|---------|---------|
| 并发模式 | ✅ | ✅ |
| useId | ✅ | ✅ |
| useTransition | ✅ | ✅ |
| useActionState | ❌ | ✅ |
| useOptimistic | ❌ | ✅ |
| use | ❌ | ✅ |
| Activity | ❌ | ✅ |

建议从 **v19.0.0** 开始学习，这是最新的稳定版本。

## v19.3.0 分支：为什么使用预构建产物？

项目还有一个 `v19.3.0` 分支，但它的调试方式与 v18/v19.0.0 不同。

### Flow Component Syntax

React 19.1+ 源码开始使用 Flow 的 **Component Syntax** 新特性：

```javascript
// Flow Component Syntax（React 源码中的写法）
component MyComponent(name: string, age: number) {
  return <div>{name}: {age}</div>;
}

// 等价于传统函数组件
function MyComponent({ name, age }: { name: string, age: number }) {
  return <div>{name}: {age}</div>;
}
```

这是 Flow 团队为 React 设计的语法糖，可以：
- 更简洁地声明组件 props 类型
- 自动推断 props 的只读性
- 更好的类型检查

### Babel 无法解析

问题在于，Babel 的 `@babel/plugin-transform-flow-strip-types` 插件无法识别这种新语法：

```
SyntaxError: Unexpected token, expected "("
  component MyComponent(name: string) {
            ^
```

Babel 只能剥离传统的 Flow 类型注解，不支持 `component` 关键字。

### v19.3.0 的解决方案

v19.3.0 分支使用 React 官方预构建的 development 版本：

```
src/packages/
├── react/cjs/
│   └── react.development.js      # 预构建产物
├── react-dom/cjs/
│   └── react-dom.development.js
└── scheduler/cjs/
    └── scheduler.development.js
```

webpack alias 直接指向这些构建产物：

```javascript
alias: {
  react: path.join(paths.reactSrc, "react"),
  "react-dom": path.join(paths.reactSrc, "react-dom"),
  scheduler: path.join(paths.reactSrc, "scheduler"),
}
```

### 调试体验对比

| 特性 | v19.0.0（源码） | v19.3.0（预构建） |
|-----|----------------|------------------|
| 代码可读性 | 原始源码，清晰 | 构建后代码，可读性降低 |
| 断点调试 | 精确到源码行 | 在构建产物中调试 |
| 修改源码 | ✅ 可修改测试 | ❌ 不可修改 |
| Flow 类型 | 运行时由 Babel 剥离 | 已在构建时剥离 |

**建议**：学习 React 源码推荐使用 `v19.0.0` 分支，v19.0.0 与 v19.3.0 的核心逻辑差异不大，但调试体验更好。

## 常见问题

### Q: 启动报错怎么办？

确保 Node 版本 >= 18，使用 pnpm 安装依赖：

```bash
node -v  # >= 18.19.1
pnpm -v  # >= 9.7.0
```

### Q: 如何添加新的测试组件？

1. 在 `src/components/` 下创建组件
2. 在 `src/App.js` 中引入并使用

### Q: 源码修改后不生效？

webpack 配置了热更新，修改源码后会自动刷新。如果不生效，尝试重启开发服务器。

## 下一步

环境准备好了，接下来我们将深入 React 源码：

1. [React 19 源码全景图](https://juejin.cn/post/7588007285547286534) - 宏观理解三大模块
2. [useState 原理](https://juejin.cn/post/7588007285547302918) - 状态管理的秘密
3. [useEffect 原理](https://juejin.cn/post/7588001624905973779) - 副作用的生命周期
4. [Fiber 工作循环](https://juejin.cn/post/7588061231589523497) - 可中断的渲染

---

> 📦 项目地址：[github.com/220529/react-debug](https://github.com/220529/react-debug)
> 
> 下一篇：[React 19 源码全景图：从宏观到微观](https://juejin.cn/post/7588007285547286534)
>
> 如果觉得有帮助，欢迎 Star ⭐
