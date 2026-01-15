# 无界微前端源码解析

## 专栏介绍

深入剖析腾讯开源微前端框架 Wujie 源码，涵盖 iframe 沙箱、Shadow DOM 隔离、Proxy 代理、路由同步、通信机制和插件系统，带你理解 Web Components + iframe 双容器架构的设计精髓。

## 核心架构

```
┌─────────────────────────────────────────────────────────┐
│                      主应用                              │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              wujie-app (Web Component)          │   │
│  │  ┌───────────────────────────────────────────┐  │   │
│  │  │           Shadow DOM                      │  │   │
│  │  │           子应用 HTML/CSS 渲染             │  │   │
│  │  └───────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              隐藏 iframe                         │   │
│  │              子应用 JS 运行                      │   │
│  │              document 操作代理到 Shadow DOM      │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 文章目录

| 序号 | 标题 | 内容概要 | 链接 |
|:---:|------|---------|------|
| 01 | 项目介绍与环境搭建 | 了解无界的设计理念、项目结构和核心 API | [阅读](https://juejin.cn/post/7588704463621406739) |
| 02 | 架构总览 | 深入 startApp 启动流程，理解双容器架构 | [阅读](https://juejin.cn/post/7589126217234464774) |
| 03 | 沙箱机制 | 分析 iframe 沙箱的创建、patch 和 JS 执行隔离 | [阅读](https://juejin.cn/post/7588704463621455891) |
| 04 | CSS 隔离 | 深入 Shadow DOM 实现 CSS 隔离的原理 | [阅读](https://juejin.cn/post/7588570963295600683) |
| 05 | JS 隔离 | 分析 Proxy 代理 window/document/location | [阅读](https://juejin.cn/post/7588570963295633451) |
| 06 | 路由同步 | 理解主子应用路由同步机制和 sync 模式 | [阅读](https://juejin.cn/post/7588704463621505043) |
| 07 | 通信机制 | 分析 EventBus 事件总线和 props 数据传递 | [阅读](https://juejin.cn/post/7588827110506102827) |
| 08 | 插件系统 | 理解 loader 和 hook 机制，学习扩展能力 | [阅读](https://juejin.cn/post/7588704463621521427) |

## 适合人群

- 对微前端架构感兴趣的前端开发者
- 正在技术选型或使用无界的团队
- 想深入理解浏览器隔离机制的同学
- 希望学习优秀开源项目设计思想的开发者

## 前置知识

- JavaScript / TypeScript 基础
- 了解 Web Components 和 Shadow DOM
- 了解 Proxy 和 Object.defineProperty
- 熟悉 iframe 的基本使用

## 源码版本

本专栏基于 **wujie v1.0.22** 版本进行分析。

```bash
git clone https://github.com/Tencent/wujie.git
cd wujie
pnpm install
npm run start
```

## 相关资源

- 📦 [Wujie GitHub](https://github.com/Tencent/wujie)
- 📖 [Wujie 官方文档](https://wujie-micro.github.io/doc/)
- 🎯 [在线示例](https://wujie-micro.github.io/demo-main-vue/)
