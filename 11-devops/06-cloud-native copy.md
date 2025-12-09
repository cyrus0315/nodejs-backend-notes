# 云原生架构

云原生（Cloud Native）是一种构建和运行应用程序的方法，充分利用云计算的优势。本文深入讲解 12-Factor App、Service Mesh、Serverless 等云原生核心概念。

## 目录
- [云原生概述](#云原生概述)
- [12-Factor App](#12-factor-app)
- [Service Mesh](#service-mesh)
- [Serverless](#serverless)
- [云原生设计模式](#云原生设计模式)
- [可观测性](#可观测性)
- [常见面试题](#常见面试题)

---

## 云原生概述

### 什么是云原生？

```
云原生 = 容器化 + 微服务 + DevOps + 持续交付

┌─────────────────────────────────────────────────────────────┐
│                      云原生技术栈                            │
├─────────────────────────────────────────────────────────────┤
│  应用层    │ 微服务 │ Serverless │ Service Mesh │ API Gateway│
├─────────────────────────────────────────────────────────────┤
│  编排层    │     Kubernetes │ Docker Swarm │ Nomad        │
├─────────────────────────────────────────────────────────────┤
│  运行时    │     Docker │ containerd │ CRI-O              │
├─────────────────────────────────────────────────────────────┤
│  基础设施  │     AWS │ GCP │ Azure │ 私有云               │
└─────────────────────────────────────────────────────────────┘
```

### CNCF 云原生定义

**云原生技术**有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。

**关键技术**：
- 容器
- 服务网格
- 微服务
- 不可变基础设施
- 声明式 API

### 云原生 vs 传统应用

| 方面 | 传统应用 | 云原生应用 |
|------|---------|-----------|
| 架构 | 单体 | 微服务 |
| 部署 | 物理机/VM | 容器 |
| 扩展 | 垂直扩展 | 水平扩展 |
| 交付 | 周期长 | 持续交付 |
| 故障处理 | 人工干预 | 自动恢复 |
| 配置 | 静态配置文件 | 动态配置中心 |

---

## 12-Factor App

12-Factor App 是构建 SaaS 应用的方法论，Node.js 应用应遵循这些原则。

### 1. Codebase（代码库）

**一份基准代码，多份部署**

```bash
# 正确：一个代码库，多环境部署
my-app/
├── src/
├── package.json
└── .env.example

# 部署
git push origin main      # -> production
git push origin develop   # -> staging
```

```typescript
// 环境区分通过环境变量
const config = {
  apiUrl: process.env.API_URL,
  logLevel: process.env.LOG_LEVEL || 'info',
};
```

### 2. Dependencies（依赖）

**显式声明依赖关系**

```json
// package.json - 锁定依赖版本
{
  "dependencies": {
    "express": "4.18.2",
    "prisma": "5.0.0"
  },
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  }
}
```

```dockerfile
# Dockerfile - 不依赖系统全局包
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
CMD ["node", "dist/main.js"]
```

### 3. Config（配置）

**在环境中存储配置**

```typescript
// config/configuration.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type EnvConfig = z.infer<typeof envSchema>;

export function validateEnv(): EnvConfig {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error('❌ Invalid environment variables:', result.error.flatten());
    process.exit(1);
  }
  
  return result.data;
}

// 使用
const config = validateEnv();
```

```yaml
# K8s ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  NODE_ENV: production
  LOG_LEVEL: info
  
---
# 敏感配置使用 Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
stringData:
  DATABASE_URL: postgresql://user:pass@db:5432/mydb
  JWT_SECRET: your-secret-key
```

### 4. Backing Services（后端服务）

**把后端服务当作附加资源**

```typescript
// 数据库、缓存、消息队列等都是可替换的资源

// 通过环境变量注入连接信息
const dbClient = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL,  // 可切换到任何 PostgreSQL
    },
  },
});

const redis = new Redis(process.env.REDIS_URL);  // 可切换到任何 Redis

const kafka = new Kafka({
  brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9092'],
});
```

```yaml
# 本地开发
DATABASE_URL=postgresql://localhost:5432/mydb
REDIS_URL=redis://localhost:6379

# 生产环境（AWS RDS + ElastiCache）
DATABASE_URL=postgresql://user:pass@mydb.xxx.rds.amazonaws.com:5432/mydb
REDIS_URL=redis://xxx.cache.amazonaws.com:6379
```

### 5. Build, Release, Run（构建、发布、运行）

**严格分离构建和运行**

```yaml
# CI/CD Pipeline
stages:
  - build      # 构建阶段
  - test       # 测试阶段
  - release    # 发布阶段（打包 + 版本）
  - deploy     # 部署阶段

build:
  stage: build
  script:
    - npm ci
    - npm run build
    - docker build -t myapp:$CI_COMMIT_SHA .

release:
  stage: release
  script:
    - docker tag myapp:$CI_COMMIT_SHA myregistry/myapp:v1.2.3
    - docker push myregistry/myapp:v1.2.3

deploy:
  stage: deploy
  script:
    - kubectl set image deployment/myapp app=myregistry/myapp:v1.2.3
```

### 6. Processes（进程）

**以一个或多个无状态进程运行应用**

```typescript
// ❌ 错误：在内存中存储状态
class BadSessionStore {
  private sessions: Map<string, any> = new Map();
  
  get(id: string) {
    return this.sessions.get(id);  // 其他实例无法访问
  }
}

// ✅ 正确：使用外部存储
import Redis from 'ioredis';

class SessionStore {
  constructor(private redis: Redis) {}
  
  async get(id: string) {
    const data = await this.redis.get(`session:${id}`);
    return data ? JSON.parse(data) : null;
  }
  
  async set(id: string, data: any, ttl: number = 3600) {
    await this.redis.setex(`session:${id}`, ttl, JSON.stringify(data));
  }
}

// ❌ 错误：本地文件存储
app.use(multer({ dest: '/tmp/uploads' }));  // 其他实例无法访问

// ✅ 正确：使用对象存储
app.use(multer({
  storage: multerS3({
    s3: s3Client,
    bucket: process.env.S3_BUCKET,
    key: (req, file, cb) => cb(null, `uploads/${Date.now()}-${file.originalname}`),
  }),
}));
```

### 7. Port Binding（端口绑定）

**通过端口绑定提供服务**

```typescript
// 应用自包含 HTTP 服务器
import express from 'express';

const app = express();
const port = process.env.PORT || 3000;

// 应用配置
app.use(express.json());
app.get('/health', (req, res) => res.json({ status: 'healthy' }));

// 直接监听端口，不依赖外部 Web 服务器
app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
```

### 8. Concurrency（并发）

**通过进程模型进行扩展**

```typescript
// 使用 PM2 进行进程管理
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'api',
    script: 'dist/main.js',
    instances: 'max',      // 根据 CPU 核心数
    exec_mode: 'cluster',  // 集群模式
    max_memory_restart: '500M',
    env: {
      NODE_ENV: 'production',
    },
  }],
};

// 或使用 Node.js cluster 模块
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  const numCPUs = os.cpus().length;
  
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker) => {
    console.log(`Worker ${worker.process.pid} died, forking a new one`);
    cluster.fork();
  });
} else {
  // 工作进程
  startServer();
}
```

```yaml
# K8s 水平扩展
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 9. Disposability（易处理）

**快速启动和优雅终止**

```typescript
// 快速启动
async function bootstrap() {
  const app = express();
  
  // 延迟初始化耗时资源
  app.get('/', (req, res) => res.send('OK'));
  
  // 先启动服务器
  const server = app.listen(3000, () => {
    console.log('Server started in', Date.now() - startTime, 'ms');
  });
  
  // 后台初始化数据库连接等
  await initializeDatabase();
  
  return server;
}

// 优雅关闭
let isShuttingDown = false;

async function gracefulShutdown(signal: string, server: Server) {
  if (isShuttingDown) return;
  isShuttingDown = true;
  
  console.log(`${signal} received, starting graceful shutdown`);
  
  // 1. 停止接收新请求
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // 2. 等待现有请求完成（最多 30 秒）
  await new Promise(resolve => setTimeout(resolve, 30000));
  
  // 3. 关闭数据库连接
  await prisma.$disconnect();
  await redis.quit();
  
  console.log('Graceful shutdown completed');
  process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM', server));
process.on('SIGINT', () => gracefulShutdown('SIGINT', server));
```

### 10. Dev/Prod Parity（开发环境与线上环境等价）

**保持开发、预发布、线上环境相似**

```yaml
# docker-compose.yml - 本地开发环境模拟生产
version: '3.8'

services:
  app:
    build: .
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
  
  db:
    image: postgres:15-alpine  # 与生产相同版本
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
  
  redis:
    image: redis:7-alpine  # 与生产相同版本
```

```typescript
// 使用相同的 ORM 和查询方式
// 开发和生产使用相同的 Prisma schema
```

### 11. Logs（日志）

**把日志当作事件流**

```typescript
import winston from 'winston';

// 输出到 stdout，由运行环境收集
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console({
      // 不写文件，输出到 stdout
    }),
  ],
});

// 结构化日志
logger.info('Request received', {
  method: req.method,
  path: req.path,
  userId: req.user?.id,
  traceId: req.headers['x-trace-id'],
});

// 由 K8s/Docker 收集日志
// kubectl logs myapp-xxx
// 或发送到 CloudWatch/ELK
```

### 12. Admin Processes（管理进程）

**后台管理任务当作一次性进程运行**

```typescript
// scripts/db-migrate.ts - 数据库迁移
async function migrate() {
  console.log('Running migrations...');
  await exec('npx prisma migrate deploy');
  console.log('Migrations completed');
  process.exit(0);
}

migrate().catch(err => {
  console.error('Migration failed:', err);
  process.exit(1);
});

// scripts/seed.ts - 数据填充
async function seed() {
  const prisma = new PrismaClient();
  await prisma.user.createMany({ data: seedData });
  await prisma.$disconnect();
}
```

```yaml
# K8s Job 运行管理任务
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:v1.0.0
          command: ["npm", "run", "db:migrate"]
          envFrom:
            - secretRef:
                name: app-secrets
      restartPolicy: Never
  backoffLimit: 3
```

---

## Service Mesh

Service Mesh 为微服务提供基础设施层的通信能力。

### Istio 架构

```
┌─────────────────────────────────────────────────────────────┐
│                       Control Plane                          │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                        Istiod                            ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────┐ ││
│  │  │ Pilot   │  │ Citadel │  │ Galley  │  │ Mixer       │ ││
│  │  │(配置)   │  │(安全)   │  │(验证)   │  │(遥测/策略)  │ ││
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────────┘ ││
│  └─────────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────────┤
│                       Data Plane                             │
│  ┌───────────────────┐    ┌───────────────────┐             │
│  │      Pod A        │    │      Pod B        │             │
│  │  ┌──────┐┌──────┐ │    │  ┌──────┐┌──────┐ │             │
│  │  │ App  ││Envoy │←┼────┼→│Envoy ││ App  │ │             │
│  │  │      ││Proxy │ │    │  │Proxy ││      │ │             │
│  │  └──────┘└──────┘ │    │  └──────┘└──────┘ │             │
│  └───────────────────┘    └───────────────────┘             │
└─────────────────────────────────────────────────────────────┘
```

### 安装 Istio

```bash
# 下载 Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*

# 安装
istioctl install --set profile=demo -y

# 启用 sidecar 自动注入
kubectl label namespace production istio-injection=enabled

# 验证
kubectl get pods -n istio-system
```

### 流量管理

```yaml
# VirtualService - 路由规则
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: nodejs-app
  namespace: production
spec:
  hosts:
    - nodejs-app
  http:
    # 金丝雀路由
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: nodejs-app
            subset: canary
    
    # A/B 测试
    - match:
        - headers:
            cookie:
              regex: ".*group=B.*"
      route:
        - destination:
            host: nodejs-app
            subset: v2
    
    # 默认路由（流量分配）
    - route:
        - destination:
            host: nodejs-app
            subset: stable
          weight: 90
        - destination:
            host: nodejs-app
            subset: canary
          weight: 10

---
# DestinationRule - 版本定义
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nodejs-app
  namespace: production
spec:
  host: nodejs-app
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

### 安全（mTLS）

```yaml
# PeerAuthentication - 启用 mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # 强制 mTLS

---
# AuthorizationPolicy - 访问控制
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: nodejs-app-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: nodejs-app
  action: ALLOW
  rules:
    # 允许来自 api-gateway 的请求
    - from:
        - source:
            principals:
              - cluster.local/ns/production/sa/api-gateway
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    
    # 允许来自监控系统的健康检查
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/health", "/metrics"]
```

### 可观测性

```yaml
# 启用追踪
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: default
  namespace: production
spec:
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 10.0  # 10% 采样

---
# 自定义指标
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            request_path:
              operation: UPSERT
              value: request.url_path
```

### 熔断器

```yaml
# DestinationRule 熔断配置
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: nodejs-app
  namespace: production
spec:
  host: nodejs-app
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
    outlierDetection:
      consecutive5xxErrors: 5       # 连续 5 个 5xx 错误
      interval: 30s                 # 检测间隔
      baseEjectionTime: 30s         # 基础驱逐时间
      maxEjectionPercent: 50        # 最多驱逐 50% 实例
      minHealthPercent: 30          # 最少保留 30% 健康实例
```

---

## Serverless

### AWS Lambda + Node.js

```typescript
// handler.ts
import { APIGatewayProxyHandler, APIGatewayProxyResult } from 'aws-lambda';
import { PrismaClient } from '@prisma/client';

// 在 Lambda 外部初始化（连接复用）
const prisma = new PrismaClient();

export const handler: APIGatewayProxyHandler = async (event) => {
  try {
    const { httpMethod, path, body, pathParameters } = event;
    
    if (httpMethod === 'GET' && path === '/users') {
      const users = await prisma.user.findMany({
        take: 10,
      });
      
      return {
        statusCode: 200,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*',
        },
        body: JSON.stringify(users),
      };
    }
    
    if (httpMethod === 'GET' && pathParameters?.id) {
      const user = await prisma.user.findUnique({
        where: { id: pathParameters.id },
      });
      
      if (!user) {
        return {
          statusCode: 404,
          body: JSON.stringify({ error: 'User not found' }),
        };
      }
      
      return {
        statusCode: 200,
        body: JSON.stringify(user),
      };
    }
    
    if (httpMethod === 'POST' && path === '/users') {
      const data = JSON.parse(body || '{}');
      const user = await prisma.user.create({ data });
      
      return {
        statusCode: 201,
        body: JSON.stringify(user),
      };
    }
    
    return {
      statusCode: 404,
      body: JSON.stringify({ error: 'Not found' }),
    };
  } catch (error) {
    console.error('Error:', error);
    
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' }),
    };
  }
};

// 冷启动优化
// 1. 使用 Provisioned Concurrency
// 2. 减小 bundle 大小
// 3. 延迟加载非关键依赖
```

### Serverless Framework

```yaml
# serverless.yml
service: nodejs-api

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  stage: ${opt:stage, 'dev'}
  memorySize: 256
  timeout: 30
  
  environment:
    DATABASE_URL: ${ssm:/myapp/${self:provider.stage}/database_url}
    NODE_ENV: ${self:provider.stage}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: !GetAtt UsersTable.Arn

functions:
  api:
    handler: dist/handler.handler
    events:
      - http:
          path: /{proxy+}
          method: any
          cors: true
    provisionedConcurrency: ${self:custom.provisionedConcurrency.${self:provider.stage}}

  # 异步处理
  processQueue:
    handler: dist/workers/queue.handler
    events:
      - sqs:
          arn: !GetAtt ProcessingQueue.Arn
          batchSize: 10
    reservedConcurrency: 5

  # 定时任务
  cleanup:
    handler: dist/workers/cleanup.handler
    events:
      - schedule: rate(1 day)

custom:
  provisionedConcurrency:
    dev: 0
    staging: 1
    production: 5

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-${self:provider.stage}-users
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
    
    ProcessingQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-${self:provider.stage}-processing
        VisibilityTimeout: 180

plugins:
  - serverless-esbuild
  - serverless-offline

package:
  individually: true
```

### 冷启动优化

```typescript
// 优化 1：延迟加载
let prisma: PrismaClient | null = null;

function getPrisma() {
  if (!prisma) {
    prisma = new PrismaClient();
  }
  return prisma;
}

// 优化 2：连接池配置
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=1`,  // Lambda 单连接
    },
  },
});

// 优化 3：使用 ESBuild 减小 bundle
// serverless.yml
// plugins:
//   - serverless-esbuild

// 优化 4：预热
// 使用 CloudWatch Events 定期调用 Lambda
```

---

## 云原生设计模式

### 1. Sidecar 模式

```yaml
# 使用 Sidecar 处理日志收集
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
    - name: app
      image: nodejs-app:v1
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    
    # Sidecar：日志收集
    - name: log-collector
      image: fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc
  
  volumes:
    - name: logs
      emptyDir: {}
    - name: fluent-bit-config
      configMap:
        name: fluent-bit-config
```

### 2. Ambassador 模式

```typescript
// Ambassador：处理外部通信
// 应用只关心业务逻辑，外部通信由 Ambassador 处理

// ambassador-service.ts
import { createProxyMiddleware } from 'http-proxy-middleware';

export const ambassadorProxy = createProxyMiddleware({
  target: process.env.EXTERNAL_API_URL,
  changeOrigin: true,
  pathRewrite: { '^/external': '' },
  onProxyReq: (proxyReq) => {
    // 添加认证头
    proxyReq.setHeader('Authorization', `Bearer ${process.env.API_KEY}`);
  },
  onProxyRes: (proxyRes, req, res) => {
    // 日志、监控
  },
});

app.use('/external', ambassadorProxy);
```

### 3. Circuit Breaker 模式

```typescript
import CircuitBreaker from 'opossum';

// 创建熔断器
const options = {
  timeout: 3000,          // 超时时间
  errorThresholdPercentage: 50,  // 错误率阈值
  resetTimeout: 30000,    // 重置时间
};

const breaker = new CircuitBreaker(callExternalService, options);

breaker.on('success', (result) => {
  console.log('Call succeeded');
});

breaker.on('timeout', () => {
  console.log('Call timed out');
});

breaker.on('reject', () => {
  console.log('Circuit breaker rejected call');
});

breaker.on('open', () => {
  console.log('Circuit breaker opened');
});

breaker.on('halfOpen', () => {
  console.log('Circuit breaker half-opened');
});

breaker.on('close', () => {
  console.log('Circuit breaker closed');
});

// 使用熔断器
async function getExternalData() {
  try {
    return await breaker.fire();
  } catch (error) {
    // 返回降级数据
    return { fallback: true, data: cachedData };
  }
}

breaker.fallback(() => ({ fallback: true, data: [] }));
```

### 4. Retry 模式

```typescript
import retry from 'async-retry';

async function fetchWithRetry(url: string) {
  return await retry(
    async (bail, attemptNumber) => {
      console.log(`Attempt ${attemptNumber}`);
      
      const response = await fetch(url);
      
      // 不重试的错误
      if (response.status === 400) {
        bail(new Error('Bad request - not retrying'));
        return;
      }
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      return response.json();
    },
    {
      retries: 3,
      factor: 2,           // 指数退避因子
      minTimeout: 1000,    // 最小等待时间
      maxTimeout: 10000,   // 最大等待时间
      randomize: true,     // 添加随机抖动
      onRetry: (error, attemptNumber) => {
        console.error(`Retry ${attemptNumber} due to ${error.message}`);
      },
    }
  );
}
```

### 5. Bulkhead 模式

```typescript
// 使用 p-limit 限制并发
import pLimit from 'p-limit';

// 创建不同的舱壁（资源池）
const dbPool = pLimit(10);      // 数据库操作限制 10 并发
const apiPool = pLimit(20);     // API 调用限制 20 并发
const criticalPool = pLimit(5); // 关键操作限制 5 并发

async function handleRequest(req: Request) {
  // 数据库操作使用 dbPool
  const userData = await dbPool(() => getUserFromDB(req.userId));
  
  // 外部 API 使用 apiPool
  const externalData = await apiPool(() => fetchExternalAPI(userData));
  
  return { userData, externalData };
}

// 批量操作时限制并发
async function processBatch(items: Item[]) {
  const results = await Promise.all(
    items.map(item => dbPool(() => processItem(item)))
  );
  return results;
}
```

---

## 可观测性

### 三大支柱

```
┌─────────────────────────────────────────────────────────────┐
│                     可观测性三大支柱                          │
├───────────────────┬───────────────────┬────────────────────┤
│       Logs        │      Metrics      │      Traces        │
│      (日志)       │      (指标)       │      (追踪)        │
├───────────────────┼───────────────────┼────────────────────┤
│ 离散事件          │ 聚合数据          │ 请求路径           │
│ 文本/JSON         │ 数值时间序列      │ Span 树            │
│ 高基数            │ 低基数            │ 中基数             │
│ Debug/问题排查    │ 监控/告警         │ 性能分析           │
├───────────────────┼───────────────────┼────────────────────┤
│ Winston/Pino      │ Prometheus        │ Jaeger/Zipkin      │
│ ELK/Loki          │ Grafana           │ OpenTelemetry      │
└───────────────────┴───────────────────┴────────────────────┘
```

### OpenTelemetry 集成

```typescript
// tracing.ts - OpenTelemetry 配置
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'nodejs-app',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/traces',
  }),
  
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT + '/v1/metrics',
    }),
    exportIntervalMillis: 60000,
  }),
  
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health', '/ready'],
      },
      '@opentelemetry/instrumentation-express': {},
      '@opentelemetry/instrumentation-pg': {},
      '@opentelemetry/instrumentation-redis': {},
    }),
  ],
});

sdk.start();

process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});

// 手动添加 span
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('nodejs-app');

async function processOrder(orderId: string) {
  const span = tracer.startSpan('processOrder', {
    attributes: {
      'order.id': orderId,
    },
  });
  
  try {
    await context.with(trace.setSpan(context.active(), span), async () => {
      await validateOrder(orderId);
      await chargePayment(orderId);
      await sendConfirmation(orderId);
    });
    
    span.setStatus({ code: SpanStatusCode.OK });
  } catch (error) {
    span.setStatus({
      code: SpanStatusCode.ERROR,
      message: error.message,
    });
    span.recordException(error);
    throw error;
  } finally {
    span.end();
  }
}
```

---

## 常见面试题

### 1. 什么是云原生？它的核心特征是什么？

**定义**：
云原生是一种充分利用云计算优势构建和运行应用的方法。

**核心特征**：
1. **容器化**：应用打包在容器中
2. **动态管理**：使用编排系统（K8s）
3. **微服务**：松耦合的服务架构
4. **声明式 API**：描述期望状态
5. **自动化**：CI/CD、自动扩缩容

### 2. 12-Factor App 中最重要的几条？

**关键原则**：
1. **配置（Config）**：环境变量存储配置
2. **无状态进程（Processes）**：应用无状态，状态存储在外部
3. **易处理（Disposability）**：快速启动、优雅关闭
4. **日志（Logs）**：日志作为事件流
5. **开发/生产等价（Dev/Prod Parity）**：环境一致性

### 3. Service Mesh 解决什么问题？

**核心能力**：
1. **流量管理**：负载均衡、金丝雀发布、流量镜像
2. **安全**：mTLS、访问控制
3. **可观测性**：分布式追踪、指标收集
4. **弹性**：熔断、重试、超时

**适用场景**：
- 微服务数量多
- 需要统一的流量控制
- 需要零信任安全模型
- 需要不侵入应用的可观测性

### 4. Serverless 的优缺点？

**优点**：
- 无需管理服务器
- 按使用付费
- 自动扩展
- 专注业务逻辑

**缺点**：
- 冷启动延迟
- 执行时间限制
- 调试困难
- 供应商锁定

**适用场景**：
- 事件驱动处理
- 不规则流量
- 快速原型开发

---

## 总结

### 云原生检查清单

- [ ] 遵循 12-Factor 原则
- [ ] 容器化部署
- [ ] 使用 K8s 编排
- [ ] 实现无状态设计
- [ ] 配置外部化
- [ ] 实现优雅关闭
- [ ] 结构化日志
- [ ] 完善可观测性（日志、指标、追踪）
- [ ] 实现弹性模式（熔断、重试）
- [ ] GitOps 部署

### 关键要点

1. **无状态是基础**
2. **配置与代码分离**
3. **可观测性优先**
4. **拥抱失败，设计弹性**
5. **自动化一切**

---

**上一篇**：[GitOps](./05-gitops.md)  
**返回**：[README](./README.md)

