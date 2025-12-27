# Egg.js 源码揭秘（七）：Context 扩展机制

> 本文深入 Extend 和 ContextLoader 源码，解析 Egg.js 如何扩展 Application、Context、Request、Response、Helper。

## 一、扩展机制概览

```
┌─────────────────────────────────────────────────────────────┐
│                      扩展类型                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  app/extend/application.ts  ──► Application.prototype       │
│  app/extend/context.ts      ──► Context.prototype           │
│  app/extend/request.ts      ──► Request.prototype           │
│  app/extend/response.ts     ──► Response.prototype          │
│  app/extend/helper.ts       ──► Helper.prototype            │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                      懒加载                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  app/service/               ──► ctx.service                 │
│  app/controller/            ──► app.controller              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 二、Extend 加载

### 2.1 加载入口

```typescript
// packages/egg/src/lib/loader/AppWorkerLoader.ts
async load(): Promise<void> {
  // 加载扩展（app > plugin > core）
  await this.loadApplicationExtend();
  await this.loadRequestExtend();
  await this.loadResponseExtend();
  await this.loadContextExtend();
  await this.loadHelperExtend();
  
  // 加载其他
  await this.loadService();
  await this.loadMiddleware();
  await this.loadController();
  await this.loadRouter();
}
```

### 2.2 loadExtend 实现

```typescript
// packages/core/src/loader/egg_loader.ts
async loadExtend(name: string, proto: object): Promise<void> {
  this.timing.start(`Load extend/${name}.js`);
  
  // 获取所有扩展文件路径
  const filepaths = this.getExtendFilePaths(name);
  
  // 支持环境特定扩展
  for (let i = 0; i < filepaths.length; i++) {
    const filepath = filepaths[i];
    filepaths.push(filepath + `.${this.serverEnv}`);
  }
  
  const mergeRecord = new Map();
  
  for (const rawFilepath of filepaths) {
    const filepath = this.resolveModule(rawFilepath);
    if (!filepath) continue;
    
    let ext = await this.requireFile(filepath);
    
    // 如果是 Class，使用其 prototype
    if (isClass(ext)) {
      ext = ext.prototype;
    }
    
    // 获取所有属性
    const properties = Object.getOwnPropertyNames(ext)
      .concat(Object.getOwnPropertySymbols(ext))
      .filter(name => name !== 'constructor');
    
    // 复制属性描述符到目标原型
    for (const property of properties) {
      let descriptor = Object.getOwnPropertyDescriptor(ext, property);
      Object.defineProperty(proto, property, descriptor);
      mergeRecord.set(property, filepath);
    }
  }
  
  this.timing.end(`Load extend/${name}.js`);
}
```

### 2.3 扩展文件路径

```typescript
protected getExtendFilePaths(name: string): string[] {
  return this.getLoadUnits().map(
    unit => path.join(unit.path, 'app/extend', name)
  );
}

// 加载顺序：plugin → framework → app
// 后加载的会覆盖先加载的
```

## 三、Application 扩展

### 3.1 扩展示例

```typescript
// app/extend/application.ts
import type { Application } from 'egg';

export default {
  // 属性
  get cache(): Map<string, any> {
    if (!this._cache) {
      this._cache = new Map();
    }
    return this._cache;
  },
  
  // 方法
  async getConfig(key: string): Promise<any> {
    const app = this as Application;
    return app.config[key];
  },
};
```

### 3.2 使用方式

```typescript
// 在任意位置使用
app.cache.set('key', 'value');
await app.getConfig('redis');
```

## 四、Context 扩展

### 4.1 扩展示例

```typescript
// app/extend/context.ts
import type { Context } from 'egg';

export default {
  // getter
  get isIOS(): boolean {
    const ctx = this as Context;
    return /iPhone|iPad|iPod/i.test(ctx.get('user-agent'));
  },
  
  // 方法
  success(data: any): void {
    const ctx = this as Context;
    ctx.body = { code: 0, data };
  },
  
  error(message: string, code = -1): void {
    const ctx = this as Context;
    ctx.body = { code, message };
  },
};
```

### 4.2 使用方式

```typescript
// 在 Controller 中使用
async index() {
  if (this.ctx.isIOS) {
    this.ctx.success({ platform: 'iOS' });
  } else {
    this.ctx.error('Not iOS');
  }
}
```

## 五、Request/Response 扩展

### 5.1 Request 扩展

```typescript
// app/extend/request.ts
export default {
  get token(): string | undefined {
    return this.get('Authorization')?.replace('Bearer ', '');
  },
  
  get clientIP(): string {
    return this.get('X-Real-IP') || this.ip;
  },
};
```

### 5.2 Response 扩展

```typescript
// app/extend/response.ts
export default {
  set token(value: string) {
    this.set('X-Token', value);
  },
  
  noCache(): void {
    this.set('Cache-Control', 'no-cache, no-store');
  },
};
```

## 六、Helper 扩展

### 6.1 扩展示例

```typescript
// app/extend/helper.ts
export default {
  formatDate(date: Date, format = 'YYYY-MM-DD'): string {
    // 格式化日期
    return date.toISOString().split('T')[0];
  },
  
  escape(html: string): string {
    return html
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;');
  },
};
```

### 6.2 使用方式

```typescript
// 在 Controller 中
const date = this.ctx.helper.formatDate(new Date());

// 在模板中
<%= helper.escape(content) %>
```

## 七、Service 懒加载

### 7.1 ContextLoader

```typescript
// packages/core/src/loader/context_loader.ts
export class ContextLoader extends FileLoader {
  constructor(options: ContextLoaderOptions) {
    const target = {};
    if (options.fieldClass) {
      options.inject[options.fieldClass] = target;
    }
    super({ ...options, target });
    
    const app = this.#inject;
    const property = options.property;
    
    // 定义 ctx.service
    Object.defineProperty(app.context, property, {
      get() {
        const ctx = this;
        
        // 每个 ctx 实例独立缓存
        if (!ctx[CLASS_LOADER]) {
          ctx[CLASS_LOADER] = new Map();
        }
        
        let instance = ctx[CLASS_LOADER].get(property);
        if (!instance) {
          instance = getInstance(target, ctx);
          ctx[CLASS_LOADER].set(property, instance);
        }
        return instance;
      },
    });
  }
}
```

### 7.2 ClassLoader

```typescript
export class ClassLoader {
  readonly _cache: Map<string, any> = new Map();
  _ctx: Context;

  constructor(options: ClassLoaderOptions) {
    const properties = options.properties;
    this._ctx = options.ctx;

    for (const property in properties) {
      this.#defineProperty(property, properties[property]);
    }
  }

  #defineProperty(property: string, values: any): void {
    Object.defineProperty(this, property, {
      get() {
        let instance = this._cache.get(property);
        if (!instance) {
          instance = getInstance(values, this._ctx);
          this._cache.set(property, instance);
        }
        return instance;
      },
    });
  }
}
```

### 7.3 实例化逻辑

```typescript
function getInstance(values: any, ctx: Context): any {
  const Class = values[EXPORTS] ? values : null;
  
  if (Class) {
    if (isClass(Class)) {
      // Service 类，传入 ctx
      return new Class(ctx);
    }
    // 普通对象
    return Class;
  }
  
  if (isPrimitive(values)) {
    return values;
  }
  
  // 目录，递归创建 ClassLoader
  return new ClassLoader({ ctx, properties: values });
}
```

## 八、Service 加载

```typescript
// packages/core/src/loader/egg_loader.ts
async loadService(options?: Partial<ContextLoaderOptions>): Promise<void> {
  this.timing.start('Load Service');
  
  const servicePaths = this.getLoadUnits().map(
    unit => path.join(unit.path, 'app/service')
  );
  
  await this.loadToContext(servicePaths, 'service', {
    call: true,
    caseStyle: CaseStyle.lower,
    fieldClass: 'serviceClasses',
    directory: servicePaths,
  });
  
  this.timing.end('Load Service');
}
```

### 8.1 Service 示例

```typescript
// app/service/user.ts
import { Service } from 'egg';

export default class UserService extends Service {
  async find(id: number) {
    return await this.ctx.model.User.findByPk(id);
  }
  
  async create(data: any) {
    return await this.ctx.model.User.create(data);
  }
}
```

### 8.2 使用方式

```typescript
// 在 Controller 中
async show() {
  const { ctx } = this;
  const user = await ctx.service.user.find(ctx.params.id);
  ctx.body = user;
}
```

## 九、扩展加载顺序

```
┌─────────────────────────────────────────────────────────────┐
│                    加载顺序（从低到高）                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 框架 egg/app/extend/                                    │
│        ↓                                                    │
│  2. 插件 plugin/app/extend/                                 │
│        ↓                                                    │
│  3. 应用 app/extend/                                        │
│        ↓                                                    │
│  4. 环境特定 app/extend/context.local.ts                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

后加载的会覆盖先加载的同名属性
```

## 十、调试技巧

### 10.1 关键断点

```typescript
// 扩展加载
packages/core/src/loader/egg_loader.ts → loadExtend

// Service 加载
packages/core/src/loader/egg_loader.ts → loadService

// 懒加载实例化
packages/core/src/loader/context_loader.ts → getInstance
```

### 10.2 查看扩展

```typescript
// 查看 Context 原型
console.log(Object.keys(app.context));

// 查看 Service 类
console.log(app.serviceClasses);

// 查看 Controller
console.log(app.controller);
```

## 十一、小结

Egg.js 扩展机制的核心：

1. **五种扩展**：Application、Context、Request、Response、Helper
2. **属性描述符**：通过 Object.defineProperty 复制到原型
3. **懒加载**：Service 通过 ContextLoader 实现按需实例化
4. **请求隔离**：每个 ctx 实例独立缓存 Service 实例
5. **加载顺序**：框架 → 插件 → 应用，后者覆盖前者

---

> 📦 源码地址：[github.com/eggjs/egg](https://github.com/eggjs/egg)
> 
> 本系列完结，感谢阅读！
> 
> 如果觉得有帮助，欢迎点赞收藏 👍
