# NestJS 源码解析：项目介绍与环境搭建

> NestJS 是 Node.js 最流行的企业级框架，本系列将深入源码，揭秘其设计思想。

## NestJS 是什么

NestJS 是一个用于构建高效、可扩展的 Node.js 服务端应用的框架。它使用 TypeScript 构建，融合了 OOP、FP、FRP 的编程范式。

核心特点：
- **模块化架构**：借鉴 Angular 的模块系统
- **依赖注入**：内置 IoC 容器
- **装饰器驱动**：大量使用 TypeScript 装饰器
- **平台无关**：支持 Express、Fastify 等底层框架

## 源码仓库

```bash
git clone https://github.com/nestjs/nest.git
cd nest
npm install
npm run build
```

当前版本：v11.1.10

## 项目结构

```
nest/
├── packages/
│   ├── common/           # 公共模块（装饰器、接口、工具）
│   ├── core/             # 核心模块（DI、路由、生命周期）
│   ├── microservices/    # 微服务支持
│   ├── platform-express/ # Express 适配器
│   ├── platform-fastify/ # Fastify 适配器
│   ├── testing/          # 测试工具
│   └── websockets/       # WebSocket 支持
│
├── integration/          # 集成测试
├── sample/               # 示例项目
└── tools/                # 构建工具
```

## 核心包说明

### @nestjs/common

公共模块，包含：
- 装饰器：`@Module`、`@Controller`、`@Injectable`、`@Get`、`@Post` 等
- 接口：`NestModule`、`CanActivate`、`PipeTransform` 等
- 异常：`HttpException`、`BadRequestException` 等
- 工具函数

### @nestjs/core

核心运行时，包含：
- `NestFactory`：应用工厂
- `NestContainer`：IoC 容器
- `Injector`：依赖注入器
- `DependenciesScanner`：模块扫描器
- `RouterExplorer`：路由探索器
- 生命周期钩子

### @nestjs/platform-express

Express 平台适配器，将 NestJS 的抽象映射到 Express。

## 核心概念

### 1. 模块 (Module)

模块是组织代码的基本单元：

```typescript
@Module({
  imports: [DatabaseModule],
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

### 2. 控制器 (Controller)

处理 HTTP 请求：

```typescript
@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  findAll(): Cat[] {
    return this.catsService.findAll();
  }
}
```

### 3. 提供者 (Provider)

可注入的服务：

```typescript
@Injectable()
export class CatsService {
  private cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

### 4. 中间件 (Middleware)

请求处理管道：

```typescript
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

### 5. 守卫 (Guard)

权限控制：

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```

### 6. 管道 (Pipe)

数据转换和验证：

```typescript
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```

### 7. 拦截器 (Interceptor)

AOP 切面：

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    return next.handle().pipe(tap(() => console.log('After...')));
  }
}
```

## 请求处理流程

```
Request
   ↓
Middleware
   ↓
Guards
   ↓
Interceptors (before)
   ↓
Pipes
   ↓
Route Handler
   ↓
Interceptors (after)
   ↓
Exception Filters
   ↓
Response
```

## 系列文章

本系列将深入分析 NestJS 源码：

1. **项目介绍**（本文）- 了解项目结构和核心概念
2. **架构总览** - NestFactory 启动流程
3. **依赖注入** - IoC 容器和 Injector 实现
4. **模块系统** - Module 扫描和加载
5. **装饰器原理** - 元数据存储机制
6. **路由系统** - 请求分发和处理
7. **中间件机制** - 中间件注册和执行
8. **AOP 实现** - Guard、Pipe、Interceptor

## 调试技巧

### 1. 使用示例项目

```bash
cd sample/01-cats-app
npm install
npm run start:dev
```

### 2. 断点调试

在 VS Code 中配置 launch.json：

```json
{
  "type": "node",
  "request": "launch",
  "name": "Debug Nest",
  "runtimeArgs": ["-r", "ts-node/register"],
  "args": ["${workspaceFolder}/sample/01-cats-app/src/main.ts"]
}
```

### 3. 关键断点位置

| 场景 | 文件 | 函数 |
|-----|------|------|
| 应用创建 | `core/nest-factory.ts` | `create` |
| 模块扫描 | `core/scanner.ts` | `scan` |
| 依赖注入 | `core/injector/injector.ts` | `loadInstance` |
| 路由注册 | `core/router/router-explorer.ts` | `explore` |

## 下一步

环境准备好了，下一篇我们将分析 NestJS 的启动流程，看看 `NestFactory.create()` 背后发生了什么。

---

> 📦 源码地址：[github.com/nestjs/nest](https://github.com/nestjs/nest)
>
> 下一篇：NestJS 架构总览
