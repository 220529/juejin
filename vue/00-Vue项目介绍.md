# Vue.js 源码揭秘（零）：项目介绍与环境搭建

> 本系列基于 Vue 3.5 源码，带你深入理解 Vue 的核心实现。

## 一、项目结构

Vue.js 采用 pnpm monorepo 架构：

```
vue-core/
├── packages/
│   ├── compiler-core/     # 编译器核心（平台无关）
│   ├── compiler-dom/      # DOM 编译器
│   ├── compiler-sfc/      # 单文件组件编译器
│   ├── compiler-ssr/      # SSR 编译器
│   ├── reactivity/        # 响应式系统
│   ├── runtime-core/      # 运行时核心（平台无关）
│   ├── runtime-dom/       # DOM 运行时
│   ├── server-renderer/   # 服务端渲染
│   ├── shared/            # 共享工具函数
│   └── vue/               # 完整构建入口
├── scripts/               # 构建脚本
├── rollup.config.js       # Rollup 配置
└── vitest.config.ts       # 测试配置
```

## 二、核心模块

### 2.1 响应式系统（reactivity）

```
reactivity/src/
├── reactive.ts      # reactive() 实现
├── ref.ts           # ref() 实现
├── computed.ts      # computed() 实现
├── effect.ts        # 副作用系统
├── dep.ts           # 依赖收集
├── baseHandlers.ts  # Proxy handlers
└── watch.ts         # watch API
```

### 2.2 运行时核心（runtime-core）

```
runtime-core/src/
├── renderer.ts      # 渲染器
├── vnode.ts         # 虚拟 DOM
├── component.ts     # 组件实例
├── scheduler.ts     # 调度器
├── apiLifecycle.ts  # 生命周期 API
├── apiWatch.ts      # watch/watchEffect
└── h.ts             # h() 函数
```

### 2.3 编译器（compiler-core）

```
compiler-core/src/
├── parse.ts         # 模板解析
├── transform.ts     # AST 转换
├── codegen.ts       # 代码生成
├── ast.ts           # AST 节点定义
└── tokenizer.ts     # 词法分析
```

## 三、模块依赖关系

```
┌─────────────────────────────────────────────────────────────┐
│                         vue                                 │
│  (完整构建，包含编译器 + 运行时)                              │
└─────────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌─────────────────────┐          ┌─────────────────────┐
│   compiler-dom      │          │    runtime-dom      │
│   (DOM 编译器)       │          │    (DOM 运行时)      │
└─────────────────────┘          └─────────────────────┘
          │                                 │
          ▼                                 ▼
┌─────────────────────┐          ┌─────────────────────┐
│   compiler-core     │          │    runtime-core     │
│   (编译器核心)       │          │    (运行时核心)      │
└─────────────────────┘          └─────────────────────┘
                                            │
                                            ▼
                                 ┌─────────────────────┐
                                 │     reactivity      │
                                 │     (响应式系统)     │
                                 └─────────────────────┘
                                            │
                                            ▼
                                 ┌─────────────────────┐
                                 │       shared        │
                                 │     (工具函数)       │
                                 └─────────────────────┘
```

## 四、环境搭建

### 4.1 克隆仓库

```bash
git clone https://github.com/vuejs/core.git vue-core
cd vue-core
```

### 4.2 安装依赖

```bash
# 需要 Node.js >= 18.12.0
pnpm install
```

### 4.3 构建项目

```bash
# 构建所有包
pnpm build

# 开发模式（监听变化）
pnpm dev
```

### 4.4 运行测试

```bash
# 运行所有测试
pnpm test

# 运行单元测试
pnpm test-unit

# 带覆盖率
pnpm test-coverage
```

## 五、调试配置

### 5.1 VSCode launch.json

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Vue Test",
      "program": "${workspaceFolder}/node_modules/vitest/vitest.mjs",
      "args": ["run", "--reporter=verbose", "${relativeFile}"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    }
  ]
}
```

### 5.2 关键断点位置

```typescript
// 响应式
packages/reactivity/src/reactive.ts → createReactiveObject
packages/reactivity/src/effect.ts → ReactiveEffect.run

// 渲染器
packages/runtime-core/src/renderer.ts → patch
packages/runtime-core/src/renderer.ts → mountComponent

// 调度器
packages/runtime-core/src/scheduler.ts → queueJob
packages/runtime-core/src/scheduler.ts → flushJobs
```

## 六、核心概念预览

### 6.1 响应式原理

```typescript
// reactive() 核心实现
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      track(target, key)  // 依赖收集
      return Reflect.get(target, key, receiver)
    },
    set(target, key, value, receiver) {
      const result = Reflect.set(target, key, value, receiver)
      trigger(target, key)  // 触发更新
      return result
    }
  })
}
```

### 6.2 渲染流程

```
Template ──► Compiler ──► Render Function ──► VNode ──► DOM
                              │
                              ▼
                         patch(diff)
```

### 6.3 组件更新

```
State Change ──► trigger() ──► effect.run() ──► render() ──► patch()
```

## 七、阅读建议

推荐阅读顺序：

1. **shared** - 工具函数，快速了解
2. **reactivity** - 响应式系统，Vue 的核心
3. **runtime-core** - 运行时，理解渲染机制
4. **compiler-core** - 编译器，了解模板编译

## 八、小结

Vue 3 的架构特点：

1. **Monorepo**：模块化设计，按需引入
2. **TypeScript**：完整类型支持
3. **Proxy**：基于 Proxy 的响应式系统
4. **Tree-shaking**：支持摇树优化
5. **Composition API**：更灵活的组合式 API

---

> 📦 源码地址：[github.com/vuejs/core](https://github.com/vuejs/core)
> 
> 下一篇：Vue3 架构总览
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
