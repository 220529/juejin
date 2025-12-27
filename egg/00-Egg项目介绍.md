# Egg.js 源码揭秘（零）：项目介绍与快速开始

> 本系列将深入 Egg.js v4 源码，带你理解企业级 Node.js 框架的设计与实现。

## 一、Egg.js 是什么？

Egg.js 是阿里开源的企业级 Node.js 框架，基于 Koa 构建，核心特性：

- **内置多进程管理**：Master-Agent-Worker 架构，充分利用多核 CPU
- **插件机制**：高度可扩展，官方和社区提供丰富插件
- **约定优于配置**：统一的目录结构和开发规范
- **框架定制**：可基于 Egg 定制上层框架

## 二、项目结构

Egg.js v4 采用 pnpm monorepo 管理：

```
egg/
├── packages/
│   ├── egg/              # 主框架
│   ├── core/             # 核心模块（EggCore、Loader、Lifecycle）
│   ├── cluster/          # 多进程管理
│   ├── koa/              # Koa 封装
│   ├── router/           # 路由
│   ├── cookies/          # Cookie 处理
│   └── utils/            # 工具函数
│
├── plugins/              # 官方插件
│   ├── security/         # 安全插件
│   ├── session/          # Session
│   ├── schedule/         # 定时任务
│   ├── multipart/        # 文件上传
│   └── ...
│
├── tegg/                 # 依赖注入框架
│
├── examples/             # 示例项目
│   ├── helloworld-commonjs/
│   └── helloworld-typescript/
│
└── site/                 # 文档站点
```

## 三、核心模块

### 3.1 packages/egg

主框架入口，导出 Application、Agent、Controller、Service 等核心类：

```typescript
// packages/egg/src/index.ts
export { Application } from './lib/application.ts';
export { Agent } from './lib/agent.ts';
export { BaseContextClass as Controller } from './lib/core/base_context_class.ts';
export { BaseContextClass as Service } from './lib/core/base_context_class.ts';
export * from '@eggjs/cluster';
```

### 3.2 packages/core

核心抽象层，包含：

- **EggCore**：继承 Koa，提供基础能力
- **EggLoader**：加载器，负责加载配置、插件、中间件、Controller、Service
- **Lifecycle**：生命周期管理

### 3.3 packages/cluster

多进程管理：

- **Master**：主进程，负责 fork Agent 和 Worker
- **Agent**：代理进程，处理后台任务
- **Worker**：工作进程，处理 HTTP 请求

## 四、快速开始

### 4.1 创建项目

```bash
mkdir my-egg-app && cd my-egg-app
pnpm create egg@beta
pnpm install
pnpm run dev
```

### 4.2 目录结构

```
my-egg-app/
├── app/
│   ├── controller/       # 控制器
│   ├── service/          # 服务层
│   ├── middleware/       # 中间件
│   ├── router.ts         # 路由配置
│   └── extend/           # 扩展
│
├── config/
│   ├── config.default.ts # 默认配置
│   ├── config.local.ts   # 本地开发配置
│   ├── config.prod.ts    # 生产环境配置
│   └── plugin.ts         # 插件配置
│
├── test/                 # 测试
└── package.json
```

### 4.3 Hello World

```typescript
// app/controller/home.ts
import { HTTPController, HTTPMethod, HTTPMethodEnum } from 'egg';

@HTTPController()
export class HomeController {
  @HTTPMethod({
    path: '/',
    method: HTTPMethodEnum.GET,
  })
  async index() {
    return 'Hello Egg.js!';
  }
}
```

## 五、版本说明

| 版本 | Node.js | 特性 |
|-----|---------|------|
| v4.x | >= 20.19.0 | ESM、TypeScript、Tegg 依赖注入 |
| v3.x | >= 14.20.0 | CommonJS、TypeScript 支持 |
| v2.x | >= 8.0.0 | 稳定版本 |

本系列基于 **v4.x** 版本分析，这是最新的 ESM + TypeScript 版本。

## 六、系列文章

| 序号 | 主题 | 核心内容 |
|-----|------|---------|
| 00 | 项目介绍（本文） | 项目结构、快速开始 |
| 01 | 架构总览 | 三大进程、核心模块 |
| 02 | 多进程模型 | Master、Agent、Worker |
| 03 | Loader 机制 | 配置、插件、中间件加载 |
| 04 | 生命周期 | Boot、Lifecycle |
| 05 | 插件系统 | 插件加载、依赖管理 |
| 06 | 中间件 | Koa 中间件、洋葱模型 |
| 07 | Context 扩展 | Request、Response、Helper |

## 七、调试技巧

### 7.1 启动调试

```bash
# 开发模式
pnpm run dev

# 调试模式
node --inspect-brk node_modules/.bin/egg-bin dev
```

### 7.2 关键断点

| 场景 | 文件 | 函数 |
|-----|------|------|
| 启动入口 | `packages/cluster/src/master.ts` | `Master.#start` |
| 加载配置 | `packages/core/src/loader/egg_loader.ts` | `loadConfig` |
| 加载插件 | `packages/core/src/loader/egg_loader.ts` | `loadPlugin` |
| 生命周期 | `packages/core/src/lifecycle.ts` | `triggerDidLoad` |

---

> 📦 源码地址：[github.com/eggjs/egg](https://github.com/eggjs/egg)
> 
> 下一篇：Egg.js 架构总览
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
