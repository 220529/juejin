# Node.js 生产部署全攻略：PM2、Docker、K8s 实战对比

> 从单机 PM2 到容器化 Docker，再到 K8s 集群编排，全面剖析 Node.js 生产部署方案的演进与选型。

## 一、部署方案演进

```
┌─────────────────────────────────────────────────────────────────┐
│                    Node.js 部署方案演进                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   2010s 早期：直接运行                                           │
│   node app.js                                                   │
│   😱 进程挂了就没了                                              │
│                                                                  │
│   2012+：进程守护                                                │
│   forever / PM2                                                 │
│   ✅ 崩溃自动重启、多进程                                        │
│                                                                  │
│   2015+：容器化                                                  │
│   Docker                                                        │
│   ✅ 环境一致、隔离、可移植                                      │
│                                                                  │
│   2018+：容器编排                                                │
│   Kubernetes                                                    │
│   ✅ 自动扩缩容、服务发现、滚动更新                              │
│                                                                  │
│   2020+：Serverless                                             │
│   AWS Lambda / Vercel / Cloudflare Workers                      │
│   ✅ 免运维、按需付费                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 二、PM2：经典的进程管理器

### 2.1 为什么需要 PM2？

```bash
# 直接运行的问题
node app.js
# 1. 终端关闭，进程就死了
# 2. 程序崩溃，没人重启
# 3. 只用了 1 个 CPU 核心
# 4. 日志散落各处

# PM2 解决所有问题
pm2 start app.js -i max
# ✅ 后台运行
# ✅ 崩溃自动重启
# ✅ 多进程利用多核
# ✅ 日志集中管理
```

### 2.2 核心命令

```bash
# 启动
pm2 start app.js                    # 单进程
pm2 start app.js -i max             # 集群模式（CPU 核心数）
pm2 start app.js -i 4               # 指定 4 个进程
pm2 start app.js --name myapp       # 指定名称
pm2 start app.js --watch            # 文件变化自动重启

# 管理
pm2 list                            # 查看所有进程
pm2 show myapp                      # 查看详情
pm2 stop myapp                      # 停止
pm2 restart myapp                   # 重启
pm2 reload myapp                    # 零停机重启 🔥
pm2 delete myapp                    # 删除

# 日志
pm2 logs                            # 查看所有日志
pm2 logs myapp                      # 查看指定应用
pm2 logs --lines 100                # 最近 100 行
pm2 flush                           # 清空日志

# 监控
pm2 monit                           # 实时监控面板
pm2 plus                            # Web 监控（付费）

# 开机自启
pm2 startup                         # 生成启动脚本
pm2 save                            # 保存当前进程列表
```

### 2.3 配置文件

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: './dist/main.js',
    instances: 'max',           // 集群模式
    exec_mode: 'cluster',
    
    // 环境变量
    env: {
      NODE_ENV: 'development',
      PORT: 3000
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 8080
    },
    
    // 日志
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: './logs/error.log',
    out_file: './logs/out.log',
    merge_logs: true,
    
    // 重启策略
    max_memory_restart: '1G',   // 内存超限重启
    restart_delay: 3000,        // 重启延迟
    max_restarts: 10,           // 最大重启次数
    
    // 监听
    watch: false,
    ignore_watch: ['node_modules', 'logs'],
  }]
};
```

```bash
# 使用配置文件启动
pm2 start ecosystem.config.js
pm2 start ecosystem.config.js --env production
```

### 2.4 集群模式原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    PM2 集群模式                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      ┌─────────────┐                            │
│                      │   Master    │                            │
│                      │   (PM2)     │                            │
│                      └──────┬──────┘                            │
│                             │                                    │
│              ┌──────────────┼──────────────┐                    │
│              │              │              │                    │
│              ▼              ▼              ▼                    │
│        ┌──────────┐  ┌──────────┐  ┌──────────┐                │
│        │ Worker 1 │  │ Worker 2 │  │ Worker 3 │                │
│        │ (Node.js)│  │ (Node.js)│  │ (Node.js)│                │
│        └──────────┘  └──────────┘  └──────────┘                │
│                                                                  │
│   • Master 负责 fork/管理 Worker                                │
│   • 请求通过 Master 分发到 Worker（轮询）                       │
│   • Worker 崩溃，Master 自动重启                                │
│   • reload 时逐个重启，保证服务不中断                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 零停机部署

```bash
# 传统重启：有短暂中断
pm2 restart myapp

# 零停机重启：逐个重启 Worker
pm2 reload myapp

# 原理：
# 1. 启动新 Worker
# 2. 新 Worker 就绪后，停止旧 Worker
# 3. 重复直到所有 Worker 更新完成
# 4. 全程有 Worker 在服务，不中断
```

## 三、Docker：容器化部署

### 3.1 为什么用 Docker？

```
┌─────────────────────────────────────────────────────────────────┐
│                    PM2 vs Docker                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PM2 的局限：                                                   │
│   • 环境依赖：Node 版本、系统库需要手动管理                     │
│   • 隔离性差：多应用共享系统资源                                │
│   • 可移植性：换台服务器要重新配置                              │
│                                                                  │
│   Docker 的优势：                                                │
│   • 环境一致："在我机器上能跑" 问题消失                        │
│   • 完全隔离：每个容器独立的文件系统、网络                      │
│   • 可移植：镜像打包一次，到处运行                              │
│   • 版本控制：镜像可以版本化、回滚                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Dockerfile 最佳实践

```dockerfile
# ==================== 多阶段构建 ====================

# 阶段 1：构建
FROM node:20-alpine AS builder

WORKDIR /app

# 先复制依赖文件（利用缓存）
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

# 再复制源码并构建
COPY . .
RUN pnpm build

# 阶段 2：生产镜像
FROM node:20-alpine AS runner

WORKDIR /app

# 安全：使用非 root 用户
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodejs

# 只复制必要文件
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER nodejs

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "dist/main.js"]
```

### 3.3 镜像优化

```dockerfile
# ❌ 不好的做法：镜像 1.2GB
FROM node:20
COPY . .
RUN npm install
CMD ["node", "app.js"]

# ✅ 优化后：镜像 150MB
FROM node:20-alpine AS builder
# ... 构建阶段

FROM node:20-alpine
# ... 只复制必要文件
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    镜像优化技巧                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. 使用 Alpine 基础镜像                                       │
│      node:20 (1GB) → node:20-alpine (150MB)                     │
│                                                                  │
│   2. 多阶段构建                                                  │
│      构建阶段的 devDependencies 不进入最终镜像                  │
│                                                                  │
│   3. 利用缓存层                                                  │
│      先 COPY package.json → RUN install → 再 COPY 源码          │
│      源码变了不用重新 install                                   │
│                                                                  │
│   4. 使用 .dockerignore                                         │
│      排除 node_modules、.git、logs 等                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://db:5432/myapp
    depends_on:
      - db
      - redis
    restart: unless-stopped
    deploy:
      replicas: 3              # 运行 3 个实例
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: myapp
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

volumes:
  postgres_data:
```

```bash
# 启动
docker-compose up -d

# 扩容
docker-compose up -d --scale app=5

# 查看日志
docker-compose logs -f app

# 停止
docker-compose down
```

### 3.5 Docker 中还需要 PM2 吗？

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker 中的进程管理                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   方案 1：直接 node（推荐）                                     │
│   CMD ["node", "dist/main.js"]                                  │
│   • 容器 = 进程，崩溃由 Docker 重启                             │
│   • 多实例由 Docker/K8s 管理                                    │
│   • 简单、符合容器理念                                          │
│                                                                  │
│   方案 2：PM2 in Docker                                         │
│   CMD ["pm2-runtime", "ecosystem.config.js"]                    │
│   • 容器内多进程                                                │
│   • 适合单机多核、不用 K8s 的场景                               │
│                                                                  │
│   建议：                                                         │
│   • 用 K8s → 不需要 PM2，K8s 管理多实例                        │
│   • 单机 Docker → 可以用 PM2 利用多核                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 四、Kubernetes：容器编排

### 4.1 为什么需要 K8s？

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Compose vs Kubernetes                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Docker Compose 的局限：                                        │
│   • 单机部署，无法跨多台服务器                                  │
│   • 手动扩缩容                                                   │
│   • 无自动故障转移                                               │
│   • 无服务发现和负载均衡                                        │
│                                                                  │
│   Kubernetes 的能力：                                            │
│   • 多节点集群，自动调度                                        │
│   • 自动扩缩容（HPA）                                           │
│   • 自动故障恢复                                                 │
│   • 内置服务发现、负载均衡                                      │
│   • 滚动更新、回滚                                               │
│   • 配置管理、密钥管理                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│            

## 四、Kubernetes：容器编排

### 4.1 为什么需要 K8s？

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Compose vs Kubernetes                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Docker Compose 的局限：                                        │
│   • 单机部署，无法跨多台服务器                                  │
│   • 手动扩缩容                                                   │
│   • 无自动故障转移                                               │
│   • 无服务发现和负载均衡                                        │
│                                                                  │
│   Kubernetes 的能力：                                            │
│   • 多节点集群，自动调度                                        │
│   • 自动扩缩容（HPA）                                           │
│   • 自动故障恢复                                                 │
│   • 内置服务发现、负载均衡                                      │
│   • 滚动更新、回滚                                               │
│   • 配置管理、密钥管理                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                    K8s 核心概念                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Pod          最小部署单元，包含一个或多个容器                 │
│   Deployment   管理 Pod 的副本数、更新策略                      │
│   Service      为 Pod 提供稳定的网络访问入口                    │
│   Ingress      HTTP 路由，域名 → Service                        │
│   ConfigMap    配置文件                                          │
│   Secret       敏感信息（密码、密钥）                           │
│   HPA          自动水平扩缩容                                    │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                      Ingress                             │   │
│   │                    (域名路由)                            │   │
│   └─────────────────────┬───────────────────────────────────┘   │
│                         │                                        │
│   ┌─────────────────────▼───────────────────────────────────┐   │
│   │                      Service                             │   │
│   │                   (负载均衡)                             │   │
│   └─────────────────────┬───────────────────────────────────┘   │
│                         │                                        │
│         ┌───────────────┼───────────────┐                       │
│         ▼               ▼               ▼                       │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│   │   Pod    │   │   Pod    │   │   Pod    │                   │
│   │ (容器)   │   │ (容器)   │   │ (容器)   │                   │
│   └──────────┘   └──────────┘   └──────────┘                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 部署配置

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
        ports:
        - containerPort: 3000
        
        # 资源限制
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # 健康检查
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        
        # 环境变量
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
```

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 3000
  type: ClusterIP
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### 4.4 自动扩缩容（HPA）

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

```bash
# 更新镜像
kubectl set image deployment/myapp myapp=myapp:2.0.0

# 查看更新状态
kubectl rollout status deployment/myapp

# 回滚
kubectl rollout undo deployment/myapp
```

## 五、Serverless：免运维部署

### 5.1 主流平台

| 平台 | 特点 | 适用场景 |
|------|------|----------|
| AWS Lambda | 最成熟、生态全 | 企业级 |
| Vercel | 前端友好、部署极简 | Next.js/前端 |
| Cloudflare Workers | 边缘计算、延迟低 | API/边缘逻辑 |
| 阿里云函数计算 | 国内首选 | 国内业务 |

### 5.2 Vercel 示例

```javascript
// api/hello.js
export default function handler(req, res) {
  res.status(200).json({ message: 'Hello!' });
}
```

### 5.3 Serverless 的取舍

```
┌─────────────────────────────────────────────────────────────────┐
│                    Serverless 优缺点                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ✅ 优点：免运维、按需付费、自动扩容                           │
│                                                                  │
│   ❌ 缺点：冷启动延迟、执行时限、无法长连接、厂商锁定           │
│                                                                  │
│   适用：API、定时任务、低流量应用、前端 SSR                     │
│   不适用：WebSocket、高频调用、复杂有状态应用                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 六、方案对比与选型

### 6.1 全面对比

| 维度 | PM2 | Docker | K8s | Serverless |
|------|-----|--------|-----|------------|
| 学习成本 | ⭐低 | ⭐⭐中 | ⭐⭐⭐高 | ⭐⭐中 |
| 运维成本 | ⭐⭐中 | ⭐⭐中 | ⭐⭐⭐高 | ⭐低 |
| 扩展性 | 单机 | 单机 | 集群 | 自动 |
| 环境一致性 | ❌ | ✅ | ✅ | ✅ |
| 适合规模 | 小型 | 中型 | 大型 | 不定 |

### 6.2 选型建议

```
┌─────────────────────────────────────────────────────────────────┐
│                    场景推荐                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   🚀 个人项目/MVP     → Vercel / Railway                        │
│   🏢 初创/小团队      → Docker + Compose + PM2                  │
│   🏭 中型公司         → 托管 K8s（EKS/GKE/ACK）                 │
│   🌐 大型企业         → 自建 K8s + DevOps                       │
│                                                                  │
│   演进路径：PM2 → Docker → K8s                                  │
│   随业务增长逐步升级，不必一步到位                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 七、CI/CD 实战

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build & Test
      run: |
        pnpm install
        pnpm build
        pnpm test
    
    - name: Build & Push Image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker push registry.example.com/myapp:${{ github.sha }}
    
    - name: Deploy to K8s
      run: kubectl set image deployment/myapp myapp=registry.example.com/myapp:${{ github.sha }}
```

## 八、总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    部署方案总结                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PM2        入门首选，单机多核，零停机部署                     │
│   Docker     环境一致，镜像可移植，中小规模首选                 │
│   K8s        大规模集群，自动扩缩容，企业级标配                 │
│   Serverless 免运维，按量付费，适合 API 和 SSR                  │
│                                                                  │
│   核心原则：简单优先、环境一致、自动化、可观测                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

> 部署不是一次性的事，而是持续优化的过程。从 PM2 起步，随业务发展逐步容器化、集群化，找到最适合当前阶段的方案才是最优解。
