# NestJS 源码解析：架构总览与启动流程

> 深入 NestFactory.create()，揭秘 NestJS 应用的启动过程。

## 启动入口

一个 NestJS 应用的入口通常是这样的：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

简单两行代码，背后发生了什么？

## NestFactory.create 流程

```
NestFactory.create(AppModule)
        ↓
┌───────────────────────────────────────┐
│  1. 创建 ApplicationConfig            │
│  2. 创建 NestContainer (IoC 容器)     │
│  3. 创建 GraphInspector               │
└───────────────────────────────────────┘
        ↓
┌───────────────────────────────────────┐
│  4. initialize()                      │
│     ├─ DependenciesScanner.scan()     │
│     └─ InstanceLoader.createInstances │
└───────────────────────────────────────┘
        ↓
┌───────────────────────────────────────┐
│  5. 创建 NestApplication              │
│  6. 返回 Proxy 包装的实例             │
└───────────────────────────────────────┘
```

## 源码分析

### NestFactory

```typescript
// packages/core/nest-factory.ts
export class NestFactoryStatic {
  public async create<T extends INestApplication = INestApplication>(
    moduleCls: IEntryNestModule,
    serverOrOptions?: AbstractHttpAdapter | NestApplicationOptions,
    options?: NestApplicationOptions,
  ): Promise<T> {
    // 1. 解析参数
    const [httpServer, appOptions] = this.isHttpServer(serverOrOptions!)
      ? [serverOrOptions, options]
      : [this.createHttpAdapter(), serverOrOptions];

    // 2. 创建核心组件
    const applicationConfig = new ApplicationConfig();
    const container = new NestContainer(applicationConfig, appOptions);
    const graphInspector = this.createGraphInspector(appOptions!, container);

    // 3. 初始化（扫描 + 实例化）
    await this.initialize(
      moduleCls,
      container,
      graphInspector,
      applicationConfig,
      appOptions,
      httpServer,
    );

    // 4. 创建应用实例
    const instance = new NestApplication(
      container,
      httpServer,
      applicationConfig,
      graphInspector,
      appOptions,
    );

    // 5. 返回代理包装
    const target = this.createNestInstance(instance);
    return this.createAdapterProxy<T>(target, httpServer);
  }
}
```

### initialize 方法

```typescript
private async initialize(
  module: IEntryNestModule,
  container: NestContainer,
  graphInspector: GraphInspector,
  config: ApplicationConfig,
  options: NestApplicationContextOptions = {},
  httpServer: HttpServer = null,
) {
  // 创建核心组件
  const injector = new Injector();
  const instanceLoader = new InstanceLoader(container, injector, graphInspector);
  const metadataScanner = new MetadataScanner();
  const dependenciesScanner = new DependenciesScanner(
    container,
    metadataScanner,
    graphInspector,
    config,
  );

  // 设置 HTTP 适配器
  container.setHttpAdapter(httpServer);

  try {
    // 阶段一：扫描模块依赖
    await dependenciesScanner.scan(module, options);

    // 阶段二：实例化所有依赖
    await instanceLoader.createInstancesOfDependencies();

    // 注册内部模块
    dependenciesScanner.applyApplicationProviders();
  } catch (e) {
    this.handleInitializationError(e);
  }
}
```

## 核心组件

### 1. NestContainer

IoC 容器，管理所有模块和依赖：

```typescript
// packages/core/injector/container.ts
export class NestContainer {
  private readonly globalModules = new Set<Module>();
  private readonly modules = new ModulesContainer();
  private readonly dynamicModulesMetadata = new Map<string, Partial<DynamicModule>>();
  private readonly internalProvidersStorage = new InternalProvidersStorage();

  // 添加模块
  public async addModule(metatype: ModuleMetatype, scope: ModuleScope) {
    const { type, dynamicMetadata, token } = await this.moduleCompiler.compile(metatype);
    
    if (this.modules.has(token)) {
      return { moduleRef: this.modules.get(token), inserted: false };
    }

    const moduleRef = new Module(type, this);
    moduleRef.token = token;
    this.modules.set(token, moduleRef);

    // 处理动态模块
    await this.addDynamicMetadata(token, dynamicMetadata, [].concat(scope, type));

    // 处理全局模块
    if (this.isGlobalModule(type, dynamicMetadata)) {
      moduleRef.isGlobal = true;
      this.addGlobalModule(moduleRef);
    }

    return { moduleRef, inserted: true };
  }
}
```

### 2. DependenciesScanner

扫描模块依赖树：

```typescript
// packages/core/scanner.ts
export class DependenciesScanner {
  public async scan(module: ModuleDefinition, options?: { overrides?: ModuleOverride[] }) {
    // 注册核心模块
    await this.registerCoreModule(options?.overrides);

    // 递归扫描模块
    await this.scanForModules({
      moduleDefinition: module,
      overrides: options?.overrides,
    });

    // 扫描模块依赖
    await this.scanModulesForDependencies();

    // 计算模块距离（用于依赖解析顺序）
    this.calculateModulesDistance();
  }

  // 扫描单个模块
  public async scanForModules({
    moduleDefinition,
    scope = [],
    ctxRegistry = [],
    overrides = [],
    lazy,
  }: ModulesScanParameters) {
    // 添加模块到容器
    const { moduleRef, inserted } = (await this.container.addModule(
      moduleDefinition,
      scope,
    ))!;

    // 递归扫描 imports
    const modules = !inserted
      ? []
      : await this.scanModulesRecursively(
          moduleRef.imports,
          [].concat(scope, moduleDefinition),
          ctxRegistry,
          overrides,
        );

    return [moduleRef].concat(modules);
  }
}
```

### 3. InstanceLoader

实例化所有依赖：

```typescript
// packages/core/injector/instance-loader.ts
export class InstanceLoader {
  public async createInstancesOfDependencies(
    modules: Map<string, Module> = this.container.getModules(),
  ) {
    // 阶段一：创建原型
    this.createPrototypes(modules);

    // 阶段二：创建实例
    await this.createInstances(modules);

    // 记录依赖图
    this.graphInspector.inspectModules(modules);
  }

  private createPrototypes(modules: Map<string, Module>) {
    modules.forEach(moduleRef => {
      this.createPrototypesOfProviders(moduleRef);
      this.createPrototypesOfInjectables(moduleRef);
      this.createPrototypesOfControllers(moduleRef);
    });
  }

  private async createInstances(modules: Map<string, Module>) {
    await Promise.all(
      [...modules.values()].map(async moduleRef => {
        await this.createInstancesOfProviders(moduleRef);
        await this.createInstancesOfInjectables(moduleRef);
        await this.createInstancesOfControllers(moduleRef);

        this.logger.log(MODULE_INIT_MESSAGE`${moduleRef.name}`);
      }),
    );
  }
}
```

### 4. Injector

依赖注入核心：

```typescript
// packages/core/injector/injector.ts
export class Injector {
  public async loadInstance<T>(
    wrapper: InstanceWrapper<T>,
    collection: Map<InjectionToken, InstanceWrapper>,
    moduleRef: Module,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
  ) {
    // 获取依赖
    const dependencies = await this.resolveConstructorParams<T>(
      wrapper,
      moduleRef,
      contextId,
      inquirer,
    );

    // 创建实例
    const instance = await this.instantiateClass(
      dependencies,
      wrapper,
      wrapper.inject,
      contextId,
      inquirer,
    );

    // 注入属性依赖
    await this.loadPropertiesOnInstance(instance, wrapper, moduleRef, contextId);

    return instance;
  }

  // 解析构造函数参数
  public async resolveConstructorParams<T>(
    wrapper: InstanceWrapper<T>,
    moduleRef: Module,
    contextId: ContextId,
    inquirer?: InstanceWrapper,
  ): Promise<unknown[]> {
    const dependencies = wrapper.getCtorMetadata();

    return Promise.all(
      dependencies.map(async (dependency, index) => {
        const { wrapper: instanceWrapper } = await this.lookupComponent(
          dependency,
          moduleRef,
          contextId,
          wrapper,
          index,
        );
        return instanceWrapper.instance;
      }),
    );
  }
}
```

## 启动时序图

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ NestFactory │  │  Scanner    │  │InstanceLoader│  │  Injector   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │                │
       │ scan()         │                │                │
       │───────────────>│                │                │
       │                │ addModule()    │                │
       │                │───────────────>│                │
       │                │                │                │
       │                │ scanForDeps()  │                │
       │                │───────────────>│                │
       │                │                │                │
       │ createInstances│                │                │
       │───────────────────────────────>│                │
       │                │                │ loadInstance() │
       │                │                │───────────────>│
       │                │                │                │
       │                │                │<───────────────│
       │<───────────────────────────────│                │
       │                │                │                │
```

## 生命周期钩子

启动过程中会触发一系列生命周期钩子：

```typescript
// 1. 模块初始化
onModuleInit()

// 2. 应用启动
onApplicationBootstrap()

// 3. 应用关闭前
beforeApplicationShutdown(signal?: string)

// 4. 模块销毁
onModuleDestroy()

// 5. 应用关闭
onApplicationShutdown(signal?: string)
```

钩子执行顺序：

```typescript
// packages/core/hooks/on-module-init.hook.ts
export async function callModuleInitHook(module: Module): Promise<void> {
  const providers = module.getNonAliasProviders();
  const [_, moduleClassHost] = providers.shift()!;

  const instances = [
    ...module.controllers,
    ...providers,
    ...module.injectables,
    ...module.middlewares,
  ];

  // 先执行非瞬态实例
  const nonTransientInstances = getNonTransientInstances(instances);
  await Promise.all(callOperator(nonTransientInstances));

  // 再执行瞬态实例
  const transientInstances = getTransientInstances(instances);
  await Promise.all(callOperator(transientInstances));

  // 最后执行模块类本身
  if (moduleClassInstance && hasOnModuleInitHook(moduleClassInstance)) {
    await moduleClassInstance.onModuleInit();
  }
}
```

## ExceptionsZone 异常处理

源码中有一个重要的细节：整个初始化过程被 `ExceptionsZone` 包裹：

```typescript
// packages/core/nest-factory.ts
await ExceptionsZone.asyncRun(
  async () => {
    await dependenciesScanner.scan(module);
    await instanceLoader.createInstancesOfDependencies();
    dependenciesScanner.applyApplicationProviders();
  },
  teardown,
  this.autoFlushLogs,
);
```

`ExceptionsZone` 提供了统一的异常处理机制，确保启动过程中的错误能被正确捕获和处理。

## Proxy 代理包装

`NestFactory.create()` 返回的不是原始的 `NestApplication` 实例，而是一个 Proxy 包装：

```typescript
// packages/core/nest-factory.ts
private createAdapterProxy<T>(app: NestApplication, adapter: HttpServer): T {
  const proxy = new Proxy(app, {
    get: (receiver: Record<string, any>, prop: string) => {
      const mapToProxy = (result: unknown) => {
        return result instanceof Promise
          ? result.then(mapToProxy)
          : result instanceof NestApplication
            ? proxy
            : result;
      };

      // 如果属性不在 app 上但在 adapter 上，代理到 adapter
      if (!(prop in receiver) && prop in adapter) {
        return (...args: unknown[]) => {
          const result = this.createExceptionZone(adapter, prop)(...args);
          return mapToProxy(result);
        };
      }
      // ...
    },
  });
  return proxy as unknown as T;
}
```

这个 Proxy 实现了两个功能：
1. **方法代理**：将 HTTP 适配器的方法代理到应用实例
2. **链式调用**：返回 `NestApplication` 的方法会返回 proxy 本身，支持链式调用

## 总结

NestJS 启动流程的核心步骤：

1. **创建容器**：`NestContainer` 作为 IoC 容器
2. **扫描模块**：`DependenciesScanner` 递归扫描模块依赖树
3. **实例化**：`InstanceLoader` + `Injector` 创建所有实例
4. **生命周期**：触发 `onModuleInit`、`onApplicationBootstrap`
5. **异常处理**：`ExceptionsZone` 统一捕获启动异常
6. **返回应用**：`NestApplication` 实例，通过 Proxy 包装实现方法代理

下一篇我们将深入分析依赖注入系统的实现。

---

> 📦 源码位置：`packages/core/nest-factory.ts`
>
> 下一篇：NestJS 依赖注入系统
