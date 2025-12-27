# NestJS 源码解析：AOP 实现

> 深入 Guard、Pipe、Interceptor、ExceptionFilter，揭秘 NestJS 的切面编程。

## AOP 概述

NestJS 通过四种增强器实现 AOP：

| 增强器 | 作用 | 执行时机 |
|-------|------|---------|
| Guard | 权限控制 | 路由处理前 |
| Interceptor | 切面逻辑 | 路由处理前后 |
| Pipe | 数据转换/验证 | 参数绑定时 |
| ExceptionFilter | 异常处理 | 异常抛出时 |

## Guard 守卫

### 定义守卫

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return this.validateRequest(request);
  }
}
```

### 使用守卫

```typescript
// 方法级别
@UseGuards(AuthGuard)
@Get()
findAll() {}

// 控制器级别
@UseGuards(AuthGuard)
@Controller('cats')
export class CatsController {}

// 全局级别
app.useGlobalGuards(new AuthGuard());
```

### 守卫执行

```typescript
// packages/core/guards/guards-consumer.ts
export class GuardsConsumer {
  public async tryActivate(
    guards: CanActivate[],
    args: unknown[],
    instance: Controller,
    callback: (...args: unknown[]) => unknown,
    type?: string,
  ): Promise<boolean> {
    if (!guards || isEmpty(guards)) {
      return true;
    }

    const context = this.createContext(args, instance, callback);
    context.setType(type);

    // 依次执行守卫
    for (const guard of guards) {
      const result = guard.canActivate(context);

      if (typeof result === 'boolean') {
        if (!result) return false;
        continue;
      }

      // 处理 Promise 或 Observable
      if (await this.pickResult(result)) {
        continue;
      }
      return false;
    }
    return true;
  }

  public async pickResult(
    result: boolean | Promise<boolean> | Observable<boolean>,
  ): Promise<boolean> {
    if (result instanceof Observable) {
      return lastValueFrom(result);
    }
    return result;
  }
}
```

### 守卫上下文创建

```typescript
// packages/core/guards/guards-context-creator.ts
export class GuardsContextCreator extends ContextCreator {
  public create(
    instance: Controller,
    callback: (...args: any[]) => any,
    moduleKey: string,
  ): CanActivate[] {
    // 获取方法级别守卫
    const methodGuards = this.getMetadata<CanActivate>(
      GUARDS_METADATA,
      callback,
    );

    // 获取类级别守卫
    const classGuards = this.getMetadata<CanActivate>(
      GUARDS_METADATA,
      instance.constructor,
    );

    // 获取全局守卫
    const globalGuards = this.getGlobalMetadata<CanActivate>();

    // 合并：全局 → 类 → 方法
    return [...globalGuards, ...classGuards, ...methodGuards];
  }
}
```

## Interceptor 拦截器

### 定义拦截器

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');
    const now = Date.now();

    return next.handle().pipe(
      tap(() => console.log(`After... ${Date.now() - now}ms`)),
    );
  }
}
```

### 拦截器执行

```typescript
// packages/core/interceptors/interceptors-consumer.ts
export class InterceptorsConsumer {
  public async intercept(
    interceptors: NestInterceptor[],
    args: unknown[],
    instance: Controller,
    callback: (...args: unknown[]) => unknown,
    next: () => Promise<unknown>,
    type?: string,
  ): Promise<unknown> {
    if (isEmpty(interceptors)) {
      return next();
    }

    const context = this.createContext(args, instance, callback);
    context.setType(type);

    // 构建拦截器链（洋葱模型）
    const nextFn = async (i = 0) => {
      if (i >= interceptors.length) {
        // 最内层：执行路由处理器
        return defer(AsyncResource.bind(() => this.transformDeferred(next)));
      }

      // 创建 CallHandler
      const handler: CallHandler = {
        handle: () =>
          defer(AsyncResource.bind(() => nextFn(i + 1))).pipe(mergeAll()),
      };

      // 执行拦截器
      return interceptors[i].intercept(context, handler);
    };

    return defer(() => nextFn()).pipe(mergeAll());
  }

  // 转换为 Observable
  public transformDeferred(next: () => Promise<any>): Observable<any> {
    return fromPromise(next()).pipe(
      switchMap(res => {
        const isDeferred = res instanceof Promise || res instanceof Observable;
        return isDeferred ? res : Promise.resolve(res);
      }),
    );
  }
}
```

### 拦截器应用场景

```typescript
// 响应转换
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map(data => ({ data, code: 0, message: 'success' })),
    );
  }
}

// 缓存
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of(cachedData);
    }
    return next.handle();
  }
}

// 超时
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(timeout(5000));
  }
}
```

## Pipe 管道

### 定义管道

```typescript
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    // 验证逻辑
    if (!this.isValid(value)) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

### 管道执行

```typescript
// packages/core/pipes/pipes-consumer.ts
export class PipesConsumer {
  public async apply(
    value: unknown,
    { metatype, type, data }: ArgumentMetadata,
    pipes: PipeTransform[],
  ) {
    const token = this.paramsTokenFactory.exchangeEnumForString(type);
    return this.applyPipes(value, { metatype, type: token, data }, pipes);
  }

  public async applyPipes(
    value: unknown,
    metadata: { metatype: any; type?: any; data?: any },
    transforms: PipeTransform[],
  ) {
    // 依次执行管道（管道链）
    return transforms.reduce(async (deferredValue, pipe) => {
      const val = await deferredValue;
      const result = pipe.transform(val, metadata);
      return result;
    }, Promise.resolve(value));
  }
}
```

### 内置管道

```typescript
// ValidationPipe - 验证
@Get(':id')
findOne(@Param('id', ValidationPipe) id: string) {}

// ParseIntPipe - 转换为整数
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {}

// ParseUUIDPipe - 验证 UUID
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string) {}

// DefaultValuePipe - 默认值
@Get()
findAll(@Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number) {}
```

## ExceptionFilter 异常过滤器

### 定义过滤器

```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      message: exception.message,
      timestamp: new Date().toISOString(),
    });
  }
}
```

### 异常处理

```typescript
// packages/core/exceptions/exceptions-handler.ts
export class ExceptionsHandler {
  private filters: ExceptionFilterMetadata[] = [];

  public handle(exception: Error | HttpException | any, host: ArgumentsHost): void {
    // 尝试使用自定义过滤器
    for (const { exceptionMetatypes, func } of this.filters) {
      if (this.isExceptionOfType(exception, exceptionMetatypes)) {
        func(exception, host);
        return;
      }
    }

    // 使用默认处理
    this.handleUnknownError(exception, host);
  }

  private isExceptionOfType(exception: any, exceptionMetatypes: Type<any>[]): boolean {
    return exceptionMetatypes.some(metatype => exception instanceof metatype);
  }

  private handleUnknownError(exception: any, host: ArgumentsHost): void {
    const response = host.switchToHttp().getResponse();

    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      response.status(status).json(exception.getResponse());
    } else {
      response.status(500).json({
        statusCode: 500,
        message: 'Internal server error',
      });
    }
  }
}
```

### @Catch 装饰器

```typescript
// packages/common/decorators/core/catch.decorator.ts
export function Catch(...exceptions: Type<any>[]): ClassDecorator {
  return (target: object) => {
    Reflect.defineMetadata(CATCH_WATERMARK, true, target);
    Reflect.defineMetadata(FILTER_CATCH_EXCEPTIONS, exceptions, target);
  };
}
```

## ExecutionContext

所有增强器共享的执行上下文：

```typescript
// packages/core/helpers/execution-context-host.ts
export class ExecutionContextHost implements ExecutionContext {
  private contextType = 'http';

  constructor(
    private readonly args: any[],
    private readonly constructorRef: Type<any> = null,
    private readonly handler: Function = null,
  ) {}

  getArgs<T extends Array<any> = any[]>(): T {
    return this.args as T;
  }

  getArgByIndex<T = any>(index: number): T {
    return this.args[index] as T;
  }

  switchToHttp(): HttpArgumentsHost {
    return {
      getRequest: <T = any>() => this.getArgByIndex<T>(0),
      getResponse: <T = any>() => this.getArgByIndex<T>(1),
      getNext: <T = any>() => this.getArgByIndex<T>(2),
    };
  }

  getClass<T = any>(): Type<T> {
    return this.constructorRef;
  }

  getHandler(): Function {
    return this.handler;
  }

  getType<T extends string = ContextType>(): T {
    return this.contextType as T;
  }
}
```

## 增强器绑定顺序

```typescript
// packages/core/router/router-execution-context.ts
public create(
  instance: Controller,
  callback: (...args: any[]) => unknown,
  methodName: string,
  moduleKey: string,
  requestMethod: RequestMethod,
) {
  // 1. 获取管道
  const pipes = this.pipesContextCreator.create(instance, callback, moduleKey);

  // 2. 获取守卫
  const guards = this.guardsContextCreator.create(instance, callback, moduleKey);

  // 3. 获取拦截器
  const interceptors = this.interceptorsContextCreator.create(
    instance,
    callback,
    moduleKey,
  );

  // 4. 获取异常过滤器
  const exceptionFilters = this.exceptionsFilterContextCreator.create(
    instance,
    callback,
    moduleKey,
  );

  // 5. 组装执行链
  return this.createHandlerProxy(
    instance,
    callback,
    methodName,
    paramsFactory,
    guards,
    interceptors,
    pipes,
    exceptionFilters,
  );
}
```

## 总结

NestJS AOP 实现的核心：

1. **Guard**：权限控制，返回 boolean 决定是否继续
2. **Interceptor**：洋葱模型，可在处理前后执行逻辑
3. **Pipe**：管道链，依次转换/验证参数
4. **ExceptionFilter**：异常捕获，统一错误处理
5. **ExecutionContext**：共享上下文，访问请求信息
6. **绑定顺序**：全局 → 控制器 → 方法

这套 AOP 机制让 NestJS 具备了强大的扩展能力。

---

> 📦 源码位置：`packages/core/guards/`、`packages/core/interceptors/`、`packages/core/pipes/`、`packages/core/exceptions/`
>
> 系列完结，感谢阅读！
