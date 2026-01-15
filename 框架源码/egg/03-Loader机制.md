# Egg.js 源码揭秘（三）：Loader 机制

> 本文深入 EggLoader 源码，解析 Egg.js 的约定式加载机制。

## 一、什么是 Loader？

Loader 是 Egg.js 的核心机制，负责按照约定加载各类文件：

```
app/
├── controller/     ──► loadController()
├── service/        ──► loadService()
├── middleware/     ──► loadMiddleware()
├── extend/         ──► loadExtend()
├── router.ts       ──► loadRouter()
└── schedule/       ──► loadSchedule()

config/
├── config.*.ts     ──► loadConfig()
└── plugin.ts       ──► loadPlugin()
```

## 二、Loader 继承关系

```
┌─────────────────────────────────────────────────────────────┐
│                      EggLoader                              │
│                    (packages/core)                          │
│                                                             │
│  • loadPlugin()      # 加载插件                             │
│  • loadConfig()      # 加载配置                             │
│  • loadExtend()      # 加载扩展                             │
│  • loadCustomLoader()# 自定义加载                           │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ extends
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  EggApplicationLoader                       │
│                    (packages/egg)                           │
│                                                             │
│  • loadService()     # 加载 Service                         │
│  • loadMiddleware()  # 加载中间件                           │
│  • loadController()  # 加载 Controller                      │
│  • loadRouter()      # 加载路由                             │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           │                               │
           ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│   AppWorkerLoader   │         │  AgentWorkerLoader  │
│                     │         │                     │
│  Worker 进程使用     │         │  Agent 进程使用     │
└─────────────────────┘         └─────────────────────┘
```

## 三、加载顺序

```typescript
// packages/egg/src/lib/loader/EggApplicationLoader.ts
async load(): Promise<void> {
  // 1. 加载插件
  await this.loadPlugin();
  
  // 2. 加载配置
  await this.loadConfig();
  
  // 3. 加载扩展
  await this.loadApplicationExtend();
  await this.loadRequestExtend();
  await this.loadResponseExtend();
  await this.loadContextExtend();
  await this.loadHelperExtend();
  
  // 4. 自定义加载器
  await this.loadCustomLoader();
  
  // 5. 加载 Service
  await this.loadService();
  
  // 6. 加载中间件
  await this.loadMiddleware();
  
  // 7. 加载 Controller
  await this.loadController();
  
  // 8. 加载路由
  await this.loadRouter();
}
```

## 四、插件加载

### 4.1 插件配置

```typescript
// config/plugin.ts
export default {
  mysql: {
    enable: true,
    package: 'egg-mysql',
  },
  redis: {
    enable: true,
    path: path.join(__dirname, '../lib/plugin/redis'),
  },
};
```

### 4.2 加载流程

```typescript
async loadPlugin(): Promise<void> {
  // 1. 获取查找目录
  this.lookupDirs = this.getLookupDirs();
  
  // 2. 加载各层插件配置
  this.eggPlugins = await this.loadEggPlugins();      // 框架插件
  this.appPlugins = await this.loadAppPlugins();      // 应用插件
  this.customPlugins = this.loadCustomPlugins();      // 自定义插件
  
  // 3. 合并插件配置
  this.#extendPlugins(this.allPlugins, this.eggPlugins);
  this.#extendPlugins(this.allPlugins, this.appPlugins);
  this.#extendPlugins(this.allPlugins, this.customPlugins);
  
  // 4. 解析插件路径
  for (const name in this.allPlugins) {
    const plugin = this.allPlugins[name];
    plugin.path = this.getPluginPath(plugin);
    await this.#mergePluginConfig(plugin);
  }
  
  // 5. 排序插件（处理依赖）
  this.orderPlugins = this.getOrderPlugins(plugins, enabledPluginNames);
  
  // 6. 设置启用的插件
  this.plugins = enablePlugins;
}
```

### 4.3 插件依赖排序

```typescript
// 使用 sequencify 算法处理依赖
const result = sequencify(allPlugins, enabledPluginNames);

// 示例：
// pluginA 依赖 pluginB
// pluginB 依赖 pluginC
// 排序结果：[pluginC, pluginB, pluginA]
```

## 五、配置加载

### 5.1 配置文件

```
config/
├── config.default.ts    # 默认配置
├── config.local.ts      # 本地开发
├── config.unittest.ts   # 单元测试
├── config.prod.ts       # 生产环境
└── config.{scope}.ts    # 自定义 scope
```

### 5.2 加载顺序

```typescript
async loadConfig(): Promise<void> {
  const target: EggAppConfig = {
    middleware: [],
    coreMiddleware: [],
  };

  // 按顺序加载配置文件
  for (const filename of this.getTypeFiles('config')) {
    for (const unit of this.getLoadUnits()) {
      const config = await this.#loadConfig(unit.path, filename);
      extend(true, target, config);  // 深度合并
    }
  }

  this.config = target;
}

// 加载单元顺序：plugin → framework → app
getLoadUnits(): EggDirInfo[] {
  return [
    ...this.orderPlugins.map(p => ({ path: p.path, type: 'plugin' })),
    ...this.eggPaths.map(p => ({ path: p, type: 'framework' })),
    { path: this.options.baseDir, type: 'app' },
  ];
}
```

### 5.3 配置合并示例

```
plugin/mysql/config/config.default.ts
    │
    ▼
egg/config/config.default.ts
    │
    ▼
app/config/config.default.ts
    │
    ▼
plugin/mysql/config/config.local.ts
    │
    ▼
egg/config/config.local.ts
    │
    ▼
app/config/config.local.ts
    │
    ▼
最终配置
```

## 六、FileLoader

通用文件加载器，支持多种加载模式：

```typescript
// packages/core/src/loader/file_loader.ts
export class FileLoader {
  constructor(options: FileLoaderOptions) {
    this.options = {
      directory: options.directory,
      target: options.target,
      match: options.match || ['**/*.ts', '**/*.js'],
      ignore: options.ignore,
      caseStyle: options.caseStyle || 'camel',
      initializer: options.initializer,
    };
  }

  async load(): Promise<Record<string, any>> {
    const items = await this.parse();
    const target = this.options.target || {};
    
    for (const item of items) {
      // 设置到目标对象
      // app.controller.home.index
      item.properties.reduce((obj, property, index) => {
        if (index === item.properties.length - 1) {
          obj[property] = item.exports;
        } else {
          obj[property] = obj[property] || {};
        }
        return obj[property];
      }, target);
    }
    
    return target;
  }
}
```

### 6.1 CaseStyle

文件名转换规则：

```typescript
// caseStyle: 'camel' (默认)
'user_profile.ts' → 'userProfile'

// caseStyle: 'upper'
'user_profile.ts' → 'UserProfile'

// caseStyle: 'lower'
'UserProfile.ts' → 'userprofile'
```

## 七、ContextLoader

用于加载需要挂载到 Context 的模块（如 Service）：

```typescript
// packages/core/src/loader/context_loader.ts
export class ContextLoader extends FileLoader {
  constructor(options: ContextLoaderOptions) {
    super(options);
    this.property = options.property;  // 'service'
  }

  async load(): Promise<void> {
    const items = await this.parse();
    
    // 定义 getter，延迟实例化
    Object.defineProperty(this.app.context, this.property, {
      get() {
        if (!this[CLASSLOADER]) {
          this[CLASSLOADER] = new Map();
        }
        const classLoader = this[CLASSLOADER];
        
        // 返回代理对象
        return new Proxy({}, {
          get: (target, prop) => {
            if (!classLoader.has(prop)) {
              const Clazz = items.get(prop);
              classLoader.set(prop, new Clazz(this));
            }
            return classLoader.get(prop);
          }
        });
      }
    });
  }
}
```

## 八、Controller 加载

```typescript
// packages/egg/src/lib/loader/EggApplicationLoader.ts
async loadController(): Promise<void> {
  const opt = {
    directory: path.join(this.options.baseDir, 'app/controller'),
    target: this.app.controller,
    caseStyle: 'lower',
    initializer: (exports, options) => {
      // 支持类和函数两种写法
      if (is.class(exports)) {
        // class HomeController extends Controller {}
        exports.prototype.pathName = options.pathName;
        exports.prototype.fullPath = options.path;
        return exports;
      }
      // exports = { index: async (ctx) => {} }
      return wrapClass(exports);
    },
  };
  
  await new FileLoader(opt).load();
}
```

### 8.1 Controller 示例

```typescript
// app/controller/home.ts
import { Controller } from 'egg';

export default class HomeController extends Controller {
  async index() {
    const { ctx } = this;
    ctx.body = 'Hello Egg.js!';
  }
}

// 加载后：app.controller.home.index
```

## 九、Service 加载

```typescript
async loadService(): Promise<void> {
  const opt = {
    directory: path.join(this.options.baseDir, 'app/service'),
    property: 'service',
    caseStyle: 'lower',
  };
  
  await new ContextLoader(opt).load();
}
```

### 9.1 Service 示例

```typescript
// app/service/user.ts
import { Service } from 'egg';

export default class UserService extends Service {
  async find(id: number) {
    return await this.ctx.model.User.findById(id);
  }
}

// 使用：ctx.service.user.find(1)
```

## 十、中间件加载

```typescript
async loadMiddleware(): Promise<void> {
  // 1. 加载中间件文件
  const opt = {
    directory: path.join(this.options.baseDir, 'app/middleware'),
    target: this.app.middlewares,
    caseStyle: 'lower',
  };
  await new FileLoader(opt).load();

  // 2. 按配置顺序使用中间件
  const middlewareNames = [
    ...this.config.coreMiddleware,  // 核心中间件
    ...this.config.middleware,       // 应用中间件
  ];

  for (const name of middlewareNames) {
    const middleware = this.app.middlewares[name];
    const options = this.config[name] || {};
    
    // 调用中间件工厂函数
    const mw = middleware(options, this.app);
    this.app.use(mw);
  }
}
```

### 10.1 中间件示例

```typescript
// app/middleware/auth.ts
export default (options, app) => {
  return async function auth(ctx, next) {
    const token = ctx.get('Authorization');
    if (!token) {
      ctx.throw(401, 'Unauthorized');
    }
    await next();
  };
};

// config/config.default.ts
export default {
  middleware: ['auth'],
  auth: {
    ignore: ['/login'],
  },
};
```

## 十一、路由加载

```typescript
async loadRouter(): Promise<void> {
  const routerFile = path.join(this.options.baseDir, 'app/router');
  const router = await this.loadFile(routerFile);
  
  if (typeof router === 'function') {
    router(this.app);
  }
  
  // 使用路由中间件
  this.app.use(this.app.router.routes());
  this.app.use(this.app.router.allowedMethods());
}
```

### 11.1 路由示例

```typescript
// app/router.ts
import { Application } from 'egg';

export default (app: Application) => {
  const { router, controller } = app;
  
  router.get('/', controller.home.index);
  router.get('/user/:id', controller.user.show);
  router.post('/user', controller.user.create);
  
  // RESTful
  router.resources('posts', '/api/posts', controller.posts);
};
```

## 十二、自定义 Loader

```typescript
// config/config.default.ts
export default {
  customLoader: {
    // 加载 app/model 目录
    model: {
      directory: 'app/model',
      inject: 'app',
      caseStyle: 'upper',
    },
    // 加载 app/rpc 目录到 ctx
    rpc: {
      directory: 'app/rpc',
      inject: 'ctx',
      caseStyle: 'lower',
    },
  },
};
```

## 十三、调试技巧

### 13.1 关键断点

```typescript
// 插件加载
packages/core/src/loader/egg_loader.ts → loadPlugin

// 配置加载
packages/core/src/loader/egg_loader.ts → loadConfig

// Controller 加载
packages/egg/src/lib/loader/EggApplicationLoader.ts → loadController

// Service 加载
packages/egg/src/lib/loader/EggApplicationLoader.ts → loadService
```

### 13.2 查看加载结果

```typescript
// 查看加载的插件
console.log(app.plugins);

// 查看配置
console.log(app.config);

// 查看 Controller
console.log(app.controller);

// 查看中间件
console.log(app.middlewares);
```

## 十四、小结

Egg.js Loader 机制的核心：

1. **约定优于配置**：固定的目录结构和加载顺序
2. **分层加载**：plugin → framework → app
3. **配置合并**：深度合并，后面覆盖前面
4. **延迟实例化**：Service 等通过 getter 延迟创建
5. **可扩展**：支持自定义 Loader

---

> 📦 源码地址：[github.com/eggjs/egg](https://github.com/eggjs/egg)
> 
> 下一篇：生命周期详解
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
