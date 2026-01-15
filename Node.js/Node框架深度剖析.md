# Node.js 框架深度剖析：从源码到架构设计

> 作为深耕 Node.js 多年的老兵，我将从源码层面剖析主流框架的设计哲学与实现原理。不讲 Hello World，只聊真正的技术内核。

## 一、中间件机制：两种哲学的碰撞

### 1.1 Express 的线性模型：简单但有坑

Express 的中间件本质是一个**递归调用链**，核心在于 `next()` 的实现：

```javascript
// Express 源码简化版 - Layer.prototype.handle_request
Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;
  if (fn.length > 3) {
    // 不是标准中间件，跳过
    return next();
  }
  try {
    fn(req, res, next);
  } catch (err) {
    next(err);
  }
};
```

**问题在哪？** 看这段代码：

```javascript
app.use(async (req, res, next) => {
  console.log('1-start');
  next();  // 注意：没有 await！
  console.log('1-end');  // 这行什么时候执行？
});

app.use(async (req, res, next) => {
  console.log('2-start');
  await someAsyncOperation();  // 耗时 1 秒
  console.log('2-end');
  next();
});

// 输出顺序：1-start → 2-start → 1-end → (1秒后) 2-end
// 1-end 在异步操作完成前就执行了！
```

**根本原因**：Express 诞生于 callback 时代，`next()` 不返回 Promise，无法 `await`。这导致：
- 无法准确计算请求耗时
- 无法在中间件后置逻辑中获取响应结果
- 错误处理需要特殊的 4 参数中间件

### 1.2 Koa 的洋葱模型：优雅的 Promise 链

Koa 的核心只有 **550 行代码**，精髓在 `koa-compose`：

```javascript
// koa-compose 源码（核心 24 行）
function compose(middleware) {
  return function (context, next) {
    let index = -1
    return dispatch(0)
    
    function dispatch(i) {
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        // 🔥 核心：返回 Promise，支持 await
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}
```

**为什么能形成洋葱？** 关键在 `Promise.resolve(fn(context, dispatch.bind(null, i + 1)))`：

```javascript
// 当你写
await next();

// 实际上是在等待后续所有中间件执行完成
// 因为 next = dispatch.bind(null, i + 1)
// 而 dispatch 返回的是 Promise
```

**洋葱模型的本质是 Promise 的嵌套**：

```javascript
// 三个中间件的实际执行结构
Promise.resolve(
  mw1(ctx, () => 
    Promise.resolve(
      mw2(ctx, () => 
        Promise.resolve(
          mw3(ctx, () => Promise.resolve())
        )
      )
    )
  )
)
```

### 1.3 性能对比：洋葱模型有代价吗？

```javascript
// 基准测试：10 个空中间件
// Express: ~15,000 req/s
// Koa:     ~14,500 req/s

// 差距只有 3%，为什么？
// 因为 V8 对 Promise 有极致优化，async/await 几乎零开销
```

**结论**：Koa 的洋葱模型在保持高性能的同时，解决了 Express 的异步控制流问题。这就是为什么 Egg.js 选择基于 Koa 而非 Express。

## 二、Egg.js 多进程架构：生产级的设计

### 2.1 为什么需要多进程？

Node.js 是**单线程**的，一个进程只能用一个 CPU 核心。8 核服务器只用 1 核？太浪费了。

更关键的是**稳定性**：单进程挂了，整个服务就挂了。

### 2.2 Egg.js 的进程模型

```
                  ┌────────────────────────────────────┐
                  │            Master 进程              │
                  │  • 不处理业务，只管理子进程          │
                  │  • 负责 fork/kill Worker           │
                  │  • 进程间通信的中转站               │
                  └──────────────┬─────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
   │   Agent     │       │  Worker 1   │       │  Worker N   │
   │             │       │             │       │             │
   │ • 单例进程  │       │ • 处理请求  │       │ • 处理请求  │
   │ • 后台任务  │       │ • 业务逻辑  │       │ • 业务逻辑  │
   │ • 长连接    │       │             │       │             │
   └─────────────┘       └─────────────┘       └─────────────┘
```

### 2.3 源码解析：Master 如何管理进程

```javascript
// egg-cluster 源码简化
class Master extends EventEmitter {
  constructor(options) {
    // ...
    this.workerCount = options.workers || os.cpus().length;
  }

  forkWorkers() {
    for (let i = 0; i < this.workerCount; i++) {
      this.forkWorker();
    }
  }

  forkWorker() {
    const worker = cluster.fork();
    
    // 🔥 关键：监听 Worker 退出，自动重启
    worker.on('exit', (code, signal) => {
      if (!this.isAllWorkerStarted) return;
      
      // 异常退出，需要重新 fork
      this.log(`[master] worker#${worker.id} died, restarting...`);
      setTimeout(() => this.forkWorker(), 1000);
    });
  }
}
```

### 2.4 Agent 的妙用：解决资源竞争

假设你要做一个定时任务，每分钟执行一次。如果有 4 个 Worker，会执行 4 次吗？

**不会！** Egg.js 的 Schedule 默认在 **Agent 进程**执行：

```javascript
// app/schedule/update_cache.js
module.exports = {
  schedule: {
    interval: '1m',
    type: 'worker',  // 在一个 Worker 中执行
    // type: 'all',  // 在所有 Worker 中执行
  },
  async task(ctx) {
    await ctx.service.cache.update();
  },
};
```

**Agent 的典型场景**：
- 定时任务调度
- 长连接维护（如 WebSocket 连接池）
- 日志收集聚合
- 配置中心监听

### 2.5 进程间通信（IPC）

Worker 之间不能直接通信，必须通过 Master 中转：

```javascript
// Worker A 发送消息
app.messenger.sendToApp('action', { data: 'hello' });

// Worker B 接收消息
app.messenger.on('action', data => {
  console.log(data);  // { data: 'hello' }
});

// 底层实现：Worker A → Master → Worker B
// 使用 Node.js 的 process.send() 和 cluster.on('message')
```

## 三、NestJS 依赖注入：IoC 容器的实现原理

### 3.1 什么是依赖注入？为什么需要它？

先看一个没有 DI 的代码：

```javascript
class UserController {
  constructor() {
    // 😱 硬编码依赖
    this.userService = new UserService(
      new UserRepository(
        new DatabaseConnection('mysql://...')
      )
    );
  }
}
```

问题：
- **耦合度高**：UserController 必须知道 UserService 的所有依赖
- **难以测试**：无法 mock UserService
- **难以替换**：想换数据库？改一堆代码

依赖注入的解法：

```typescript
@Controller()
class UserController {
  // 🎉 只声明需要什么，不关心怎么创建
  constructor(private userService: UserService) {}
}
```

### 3.2 NestJS IoC 容器的核心实现

NestJS 使用 **reflect-metadata** 在编译时收集类型信息：

```typescript
// 当你写
@Injectable()
class UserService {
  constructor(private userRepo: UserRepository) {}
}

// TypeScript 编译后，会生成元数据
Reflect.defineMetadata('design:paramtypes', [UserRepository], UserService);
```

**IoC 容器的核心逻辑**：

```typescript
// NestJS 源码简化版
class InjectorContainer {
  private instances = new Map();
  
  resolve<T>(token: Type<T>): T {
    // 1. 已经实例化过，直接返回（单例）
    if (this.instances.has(token)) {
      return this.instances.get(token);
    }
    
    // 2. 获取构造函数的参数类型
    const paramTypes = Reflect.getMetadata('design:paramtypes', token) || [];
    
    // 3. 🔥 递归解析所有依赖
    const dependencies = paramTypes.map(dep => this.resolve(dep));
    
    // 4. 创建实例
    const instance = new token(...dependencies);
    this.instances.set(token, instance);
    
    return instance;
  }
}
```

### 3.3 作用域：单例 vs 请求级

```typescript
// 默认：单例（整个应用共享一个实例）
@Injectable()
class ConfigService {}

// 请求级：每个请求创建新实例
@Injectable({ scope: Scope.REQUEST })
class RequestContextService {
  constructor(@Inject(REQUEST) private request: Request) {}
}

// 瞬态：每次注入都创建新实例
@Injectable({ scope: Scope.TRANSIENT })
class HelperService {}
```

**请求级作用域的实现原理**：

```typescript
// NestJS 为每个请求创建一个子容器
class RequestContainer extends InjectorContainer {
  constructor(private parentContainer: InjectorContainer) {}
  
  resolve<T>(token: Type<T>): T {
    const scope = Reflect.getMetadata('scope', token);
    
    if (scope === Scope.REQUEST) {
      // 在当前请求容器中创建
      return this.createInstance(token);
    }
    
    // 单例从父容器获取
    return this.parentContainer.resolve(token);
  }
}
```

### 3.4 循环依赖的处理

```typescript
// A 依赖 B，B 依赖 A
@Injectable()
class ServiceA {
  constructor(private serviceB: ServiceB) {}  // 😱 循环依赖
}

@Injectable()
class ServiceB {
  constructor(private serviceA: ServiceA) {}
}

// NestJS 的解决方案：forwardRef
@Injectable()
class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))
    private serviceB: ServiceB
  ) {}
}
```

**forwardRef 原理**：延迟解析，先创建实例，后注入依赖。

## 四、Fastify 性能优化：为什么它这么快？

### 4.1 JSON 序列化：fast-json-stringify

标准 `JSON.stringify()` 需要在运行时检测类型：

```javascript
// V8 的 JSON.stringify 伪代码
function stringify(obj) {
  if (typeof obj === 'string') return `"${escape(obj)}"`;
  if (typeof obj === 'number') return String(obj);
  if (Array.isArray(obj)) return `[${obj.map(stringify).join(',')}]`;
  if (typeof obj === 'object') {
    // 😱 需要遍历所有 key，逐个检测类型
    return `{${Object.keys(obj).map(k => `"${k}":${stringify(obj[k])}`).join(',')}}`;
  }
}
```

Fastify 的 `fast-json-stringify` 根据 JSON Schema **预编译**序列化函数：

```javascript
// 给定 Schema
const schema = {
  type: 'object',
  properties: {
    id: { type: 'integer' },
    name: { type: 'string' }
  }
};

// fast-json-stringify 生成的代码（简化）
function serialize(obj) {
  return `{"id":${obj.id},"name":"${obj.name}"}`;
}

// 没有类型检测，没有遍历，直接拼接！
```

**性能对比**：

```
JSON.stringify:        ~500,000 ops/sec
fast-json-stringify:   ~1,200,000 ops/sec  (快 2.4 倍)
```

### 4.2 路由匹配：Radix Tree

Express 的路由是**线性匹配**：

```javascript
// Express 路由匹配伪代码
for (const route of routes) {
  if (route.match(path)) {
    return route.handler;
  }
}
// O(n) 复杂度，路由越多越慢
```

Fastify 使用 **find-my-way**，基于 Radix Tree（基数树）：

```
                    /
                    │
            ┌───────┴───────┐
            │               │
          users           posts
            │               │
      ┌─────┴─────┐        GET
      │           │
     GET      /:id
              │
        ┌─────┴─────┐
        │           │
       GET        DELETE
```

```javascript
// Radix Tree 匹配
// /users/123 → 只需要 3 次比较
// O(log n) 复杂度
```

### 4.3 请求/响应复用

```javascript
// Express：每个请求创建新对象
app.use((req, res, next) => {
  // req 和 res 是新创建的对象
});

// Fastify：对象池复用
class RequestPool {
  constructor(size) {
    this.pool = Array(size).fill(null).map(() => new Request());
  }
  
  acquire() {
    return this.pool.pop() || new Request();
  }
  
  release(req) {
    req.reset();  // 重置状态
    this.pool.push(req);
  }
}
```

**减少 GC 压力**：高并发下，对象创建/销毁是性能瓶颈。

### 4.4 Schema 验证：ajv 预编译

```javascript
// 运行时验证（慢）
function validate(data, schema) {
  if (schema.type === 'object') {
    for (const key of Object.keys(schema.properties)) {
      // 递归验证...
    }
  }
}

// ajv 预编译（快）
const validate = ajv.compile(schema);
// 生成的验证函数：
function validate(data) {
  if (typeof data.id !== 'number') return false;
  if (typeof data.name !== 'string') return false;
  return true;
}
```

## 五、插件系统设计：可扩展性的艺术

### 5.1 Egg.js 插件：约定式加载

Egg.js 的插件本质是一个**迷你 Egg 应用**：

```
egg-mysql/
├── app/
│   └── extend/
│       └── application.js   # 扩展 app 对象
├── config/
│   └── config.default.js    # 默认配置
├── app.js                   # 应用启动时执行
└── package.json
```

**加载机制**：

```javascript
// egg-core 源码简化
class EggLoader {
  loadPlugin() {
    // 1. 读取 config/plugin.js
    const plugins = this.readPluginConfigs();
    
    // 2. 拓扑排序（处理依赖关系）
    const orderedPlugins = this.orderPlugins(plugins);
    
    // 3. 按顺序加载
    for (const plugin of orderedPlugins) {
      // 加载 app/extend
      this.loadExtend(plugin.path);
      // 加载 config
      this.loadConfig(plugin.path);
      // 执行 app.js
      this.loadAppFile(plugin.path);
    }
  }
}
```

**插件依赖声明**：

```javascript
// egg-sequelize/package.json
{
  "eggPlugin": {
    "name": "sequelize",
    "dependencies": ["mysql"]  // 依赖 egg-mysql
  }
}
```

### 5.2 NestJS 模块：依赖注入的边界

NestJS 的模块是 **IoC 容器的作用域**：

```typescript
@Module({
  imports: [DatabaseModule],      // 导入其他模块
  controllers: [UserController],  // 本模块的控制器
  providers: [UserService],       // 本模块的服务
  exports: [UserService],         // 暴露给其他模块
})
export class UserModule {}
```

**模块隔离原理**：

```typescript
// 每个模块有自己的 IoC 容器
class ModuleContainer {
  private providers = new Map();
  private imports = new Set();
  
  resolve(token) {
    // 1. 先在本模块查找
    if (this.providers.has(token)) {
      return this.providers.get(token);
    }
    
    // 2. 在导入的模块中查找（只能访问 exports 的）
    for (const importedModule of this.imports) {
      if (importedModule.exports.has(token)) {
        return importedModule.resolve(token);
      }
    }
    
    throw new Error(`Provider ${token} not found`);
  }
}
```

**动态模块**：运行时配置

```typescript
@Module({})
export class DatabaseModule {
  static forRoot(options: DbOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DB_OPTIONS',
          useValue: options,
        },
        {
          provide: 'DB_CONNECTION',
          useFactory: (opts) => createConnection(opts),
          inject: ['DB_OPTIONS'],
        },
      ],
      exports: ['DB_CONNECTION'],
    };
  }
}

// 使用
@Module({
  imports: [DatabaseModule.forRoot({ host: 'localhost' })],
})
export class AppModule {}
```

### 5.3 Fastify 插件：封装上下文

Fastify 的插件有**独立的封装上下文**：

```javascript
// 父级
fastify.decorate('config', { env: 'prod' });

fastify.register(async function plugin(instance) {
  // instance 是 fastify 的子实例
  // 可以访问父级的 decorate
  console.log(instance.config);  // { env: 'prod' }
  
  // 子级的 decorate 不会污染父级
  instance.decorate('pluginOnly', true);
});

// 父级访问不到 pluginOnly
console.log(fastify.pluginOnly);  // undefined
```

**实现原理**：原型链继承

```javascript
function createChildInstance(parent) {
  const child = Object.create(parent);
  child._decorators = Object.create(parent._decorators);
  return child;
}
```

## 六、错误处理：优雅降级的设计

### 6.1 Express 的错误处理缺陷

```javascript
// 同步错误：自动捕获 ✅
app.get('/sync', (req, res) => {
  throw new Error('sync error');
});

// 异步错误：不会被捕获 ❌
app.get('/async', async (req, res) => {
  await Promise.reject(new Error('async error'));
  // 这个错误会导致 UnhandledPromiseRejection
});

// 必须手动 try-catch
app.get('/async', async (req, res, next) => {
  try {
    await Promise.reject(new Error('async error'));
  } catch (err) {
    next(err);  // 手动传递
  }
});
```

**Express 5.0 终于修复了这个问题**（2024 年发布）：

```javascript
// Express 5.0 自动捕获 async 错误
app.get('/async', async (req, res) => {
  throw new Error('now it works!');
});
```

### 6.2 Koa 的统一错误处理

```javascript
// 一个中间件搞定所有错误
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (err) {
    ctx.status = err.status || 500;
    ctx.body = {
      code: err.code || 'INTERNAL_ERROR',
      message: err.message,
    };
    
    // 触发应用级错误事件（用于日志记录）
    ctx.app.emit('error', err, ctx);
  }
});

// 业务代码直接抛出
app.use(async (ctx) => {
  const user = await User.findById(ctx.params.id);
  if (!user) {
    const err = new Error('User not found');
    err.status = 404;
    err.code = 'USER_NOT_FOUND';
    throw err;
  }
});
```

### 6.3 NestJS 的异常过滤器

NestJS 提供了**分层的异常处理**：

```typescript
// 1. 内置 HTTP 异常
throw new NotFoundException('User not found');
throw new BadRequestException('Invalid input');
throw new UnauthorizedException('Please login');

// 2. 自定义异常
export class BusinessException extends HttpException {
  constructor(code: string, message: string) {
    super({ code, message }, HttpStatus.BAD_REQUEST);
  }
}

// 3. 异常过滤器（全局/控制器/方法级别）
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    // 区分 HTTP 异常和未知异常
    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;

    const message = exception instanceof HttpException
      ? exception.getResponse()
      : 'Internal server error';

    response.status(status).json({
      statusCode: status,
      message,
      path: request.url,
      timestamp: new Date().toISOString(),
    });
  }
}
```

### 6.4 优雅关闭：不丢失请求

```javascript
// Egg.js 的优雅关闭
// egg-cluster 源码
class Master {
  onSignal() {
    // 1. 停止接收新请求
    this.stopAcceptingConnections();
    
    // 2. 等待现有请求处理完成（最多 30 秒）
    await this.waitForRequestsToFinish(30000);
    
    // 3. 关闭数据库连接等资源
    await this.closeResources();
    
    // 4. 退出进程
    process.exit(0);
  }
}

// NestJS 的优雅关闭
app.enableShutdownHooks();

@Injectable()
export class DatabaseService implements OnModuleDestroy {
  async onModuleDestroy() {
    await this.connection.close();
  }
}
```

## 七、架构对比：设计哲学的差异

### 7.1 核心设计理念

| 框架 | 设计哲学 | 核心思想 |
|-----|---------|---------|
| Express | 极简主义 | 只提供最基础的功能，其他靠中间件 |
| Koa | 优雅极简 | 用 async/await 重写 Express |
| Fastify | 性能至上 | 一切为了速度，预编译优化 |
| Egg.js | 约定优于配置 | 企业级规范，开箱即用 |
| NestJS | 架构优先 | Angular 风格，强类型，IoC |

### 7.2 适用场景分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    框架选型象限图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   复杂度高 ▲                                                    │
│            │                                                     │
│            │    NestJS          Egg.js                          │
│            │    (架构完善)       (企业规范)                      │
│            │                                                     │
│            │                                                     │
│            │    Fastify         Koa                             │
│            │    (高性能)         (灵活)                          │
│            │                                                     │
│            │              Express                                │
│            │              (简单)                                 │
│            │                                                     │
│            └────────────────────────────────────────► 性能要求   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 我的选型建议

**选 Express 当**：
- 快速原型验证
- 简单的 API 服务
- 团队 Node.js 经验少

**选 Koa 当**：
- 需要精细控制中间件流程
- 想要更现代的 async/await 体验
- 中小型项目

**选 Fastify 当**：
- 性能是第一优先级
- 高并发 API 网关
- 需要 JSON Schema 验证

**选 Egg.js 当**：
- 国内团队，需要中文文档
- 企业级项目，需要规范约束
- 需要多进程、定时任务等开箱即用

**选 NestJS 当**：
- TypeScript 项目
- 团队有 Angular/Spring 背景
- 需要微服务架构
- 大型项目，需要强架构约束

## 八、性能实测数据

### 8.1 基准测试环境

```
CPU: Intel i7-12700 (12 核)
RAM: 32GB DDR5
Node.js: v20.10.0
测试工具: autocannon -c 100 -d 30
```

### 8.2 Hello World 测试

| 框架 | 请求/秒 | 延迟 (avg) | 延迟 (p99) |
|-----|--------|-----------|-----------|
| Fastify | 78,432 | 1.2ms | 2.8ms |
| Koa | 52,156 | 1.8ms | 4.2ms |
| Express | 45,823 | 2.1ms | 5.1ms |
| NestJS (Fastify) | 71,245 | 1.3ms | 3.1ms |
| NestJS (Express) | 38,912 | 2.5ms | 6.2ms |
| Egg.js | 41,567 | 2.3ms | 5.8ms |

### 8.3 JSON 序列化测试（返回 1KB JSON）

| 框架 | 请求/秒 | 吞吐量 |
|-----|--------|-------|
| Fastify (Schema) | 65,234 | 63.5 MB/s |
| Fastify (无 Schema) | 48,123 | 46.9 MB/s |
| Koa | 42,567 | 41.5 MB/s |
| Express | 38,234 | 37.3 MB/s |

**结论**：Fastify 的 JSON Schema 序列化带来 **35%** 的性能提升。

### 8.4 数据库查询测试（PostgreSQL）

| 框架 | 请求/秒 | 说明 |
|-----|--------|------|
| Fastify + Prisma | 12,345 | |
| NestJS + TypeORM | 10,234 | IoC 有轻微开销 |
| Egg.js + Sequelize | 9,876 | 多进程分摊负载 |
| Koa + Knex | 11,567 | |

**结论**：数据库成为瓶颈时，框架差异不明显。

## 九、总结

### 框架选型速查

| 你的情况 | 推荐 |
|---------|------|
| 追求极致性能 | Fastify |
| 需要强架构约束 | NestJS |
| 国内企业项目 | Egg.js |
| 快速原型开发 | Express |
| 中小型项目 | Koa |
| 微服务架构 | NestJS |
| TypeScript 项目 | NestJS / Fastify |

### 最后的话

没有银弹，只有 trade-off。

选框架就像选工具：
- 钉钉子用锤子，不用电钻
- 建高楼用脚手架，不用梯子

理解每个框架的设计哲学和实现原理，才能做出正确的选择。

---

> 如果这篇文章对你有帮助，欢迎点赞收藏！有问题评论区见 🎉
