# 微服务深入

微服务架构的深入实践。本文讲解服务治理、通信模式、部署策略等高级主题。

## 目录
- [服务拆分](#服务拆分)
- [服务通信](#服务通信)
- [服务发现](#服务发现)
- [配置管理](#配置管理)
- [API 网关](#api-网关)
- [熔断与限流](#熔断与限流)
- [分布式事务](#分布式事务)
- [部署策略](#部署策略)
- [面试题](#常见面试题)

---

## 服务拆分

### 拆分原则

```typescript
// 1. 按业务领域拆分（DDD 限界上下文）
const businessDomains = {
  用户域: ['注册', '登录', '个人信息', '权限'],
  商品域: ['商品管理', '分类', '搜索', '推荐'],
  订单域: ['下单', '订单管理', '购物车'],
  支付域: ['支付', '退款', '账单'],
  物流域: ['发货', '配送', '追踪'],
  库存域: ['库存管理', '预留', '盘点']
};

// 2. 按团队拆分（康威定律）
// 服务边界 ≈ 团队边界
// 一个服务由一个团队负责

// 3. 按变化频率拆分
const changeFrequency = {
  频繁变化: ['营销服务', '推荐服务'],
  稳定: ['用户服务', '支付服务']
};

// 4. 按扩展性需求拆分
const scalability = {
  高负载: ['搜索服务', '商品详情服务'],
  一般: ['用户服务', '后台管理服务']
};
```

### 拆分粒度

```typescript
// ❌ 过细：纳米服务
// 每个函数一个服务，管理成本过高

// ❌ 过粗：分布式单体
// 服务太大，失去微服务优势

// ✅ 合适粒度
// 2 Pizza Team Rule：一个服务由 5-10 人团队维护
// 单一职责：一个服务只做一件事
// 独立部署：可以独立开发、测试、部署

// 拆分示例：电商系统
const services = {
  'user-service': {
    功能: ['用户注册', '登录', '个人信息'],
    数据库: 'PostgreSQL',
    团队规模: 5
  },
  'product-service': {
    功能: ['商品管理', '分类'],
    数据库: 'MongoDB',
    团队规模: 6
  },
  'search-service': {
    功能: ['商品搜索', '筛选'],
    数据库: 'Elasticsearch',
    团队规模: 4
  },
  'order-service': {
    功能: ['订单创建', '订单管理'],
    数据库: 'MySQL',
    团队规模: 7
  },
  'payment-service': {
    功能: ['支付', '退款'],
    数据库: 'PostgreSQL',
    团队规模: 5
  },
  'inventory-service': {
    功能: ['库存管理', '预留'],
    数据库: 'Redis + MySQL',
    团队规模: 4
  },
  'notification-service': {
    功能: ['邮件', '短信', '推送'],
    数据库: 'Redis',
    团队规模: 3
  }
};
```

---

## 服务通信

### 同步通信

```typescript
// 1. REST
import axios from 'axios';

class UserClient {
  private baseUrl = 'http://user-service';

  async getUser(id: string): Promise<User> {
    const response = await axios.get(`${this.baseUrl}/users/${id}`, {
      timeout: 5000,
      headers: {
        'X-Request-ID': context.requestId
      }
    });
    return response.data;
  }
}

// 2. gRPC
// proto/user.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
  string id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

// 客户端调用
import * as grpc from '@grpc/grpc-js';
import { UserServiceClient } from './generated/user_grpc_pb';

const client = new UserServiceClient(
  'user-service:50051',
  grpc.credentials.createInsecure()
);

const request = new GetUserRequest();
request.setId('123');

client.getUser(request, (err, response) => {
  if (err) {
    console.error('gRPC error:', err);
    return;
  }
  console.log('User:', response.toObject());
});

// 3. GraphQL Federation
// gateway
import { ApolloGateway } from '@apollo/gateway';
import { ApolloServer } from 'apollo-server';

const gateway = new ApolloGateway({
  serviceList: [
    { name: 'users', url: 'http://user-service:4001/graphql' },
    { name: 'products', url: 'http://product-service:4002/graphql' },
    { name: 'orders', url: 'http://order-service:4003/graphql' }
  ]
});

const server = new ApolloServer({ gateway, subscriptions: false });
```

### 异步通信

```typescript
// 1. 消息队列（Kafka）
import { Kafka, Producer, Consumer } from 'kafkajs';

// 生产者
const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();

async function publishOrderCreated(order: Order) {
  await producer.send({
    topic: 'order-events',
    messages: [{
      key: order.id,
      value: JSON.stringify({
        type: 'OrderCreated',
        data: order,
        timestamp: new Date().toISOString()
      }),
      headers: {
        'correlation-id': context.correlationId
      }
    }]
  });
}

// 消费者
const consumer = kafka.consumer({ groupId: 'inventory-service' });

await consumer.subscribe({ topic: 'order-events' });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const event = JSON.parse(message.value.toString());
    
    if (event.type === 'OrderCreated') {
      await reserveInventory(event.data.items);
    }
  }
});

// 2. 事件总线（NestJS）
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'ORDER_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            brokers: ['kafka:9092']
          },
          consumer: {
            groupId: 'order-consumer'
          }
        }
      }
    ])
  ]
})
export class AppModule {}

// 事件发布
@Injectable()
export class OrderService {
  constructor(
    @Inject('ORDER_SERVICE') private client: ClientKafka
  ) {}

  async createOrder(data: any) {
    const order = await this.orderRepository.create(data);
    
    // 发布事件
    this.client.emit('order_created', order);
    
    return order;
  }
}

// 事件处理
@Controller()
export class InventoryController {
  @EventPattern('order_created')
  async handleOrderCreated(@Payload() order: any) {
    await this.inventoryService.reserve(order.items);
  }
}
```

### 通信模式对比

| 模式 | 场景 | 优点 | 缺点 |
|------|------|------|------|
| **REST** | 同步调用 | 简单、通用 | 性能一般 |
| **gRPC** | 高性能同步 | 高效、类型安全 | 复杂度高 |
| **消息队列** | 异步处理 | 解耦、可靠 | 最终一致 |
| **GraphQL** | 聚合查询 | 灵活查询 | 复杂度高 |

---

## 服务发现

### 客户端发现

```typescript
// 客户端从注册中心获取服务实例，自己选择

// 1. 服务注册
import Consul from 'consul';

const consul = new Consul();

async function registerService() {
  await consul.agent.service.register({
    name: 'user-service',
    id: `user-service-${process.env.HOSTNAME}`,
    address: process.env.POD_IP,
    port: 3000,
    check: {
      http: `http://${process.env.POD_IP}:3000/health`,
      interval: '10s',
      timeout: '5s'
    },
    tags: ['v1', 'production']
  });
}

// 2. 服务发现
async function getServiceInstances(serviceName: string) {
  const services = await consul.catalog.service.nodes(serviceName);
  return services.map(s => ({
    address: s.ServiceAddress,
    port: s.ServicePort
  }));
}

// 3. 客户端负载均衡
class ServiceClient {
  private instances: ServiceInstance[] = [];
  private currentIndex = 0;

  async refreshInstances() {
    this.instances = await getServiceInstances('user-service');
  }

  getNextInstance(): ServiceInstance {
    // 轮询
    const instance = this.instances[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.instances.length;
    return instance;
  }

  async request(path: string) {
    const instance = this.getNextInstance();
    return axios.get(`http://${instance.address}:${instance.port}${path}`);
  }
}
```

### 服务端发现

```typescript
// 通过负载均衡器/服务网格转发，客户端不感知

// Kubernetes Service
// service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP

// 客户端通过 DNS 访问
// http://user-service/users/123

// Kubernetes 自动负载均衡到 Pod
```

### Kubernetes 服务发现

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:1.0.0
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

## 配置管理

### 集中配置

```typescript
// 1. 环境变量
// .env
DATABASE_URL=postgresql://localhost:5432/mydb
REDIS_URL=redis://localhost:6379
JWT_SECRET=xxx

// 2. 配置中心（Consul KV）
import Consul from 'consul';

const consul = new Consul();

async function getConfig(key: string) {
  const result = await consul.kv.get(key);
  return result ? JSON.parse(result.Value) : null;
}

// 配置结构
// config/user-service/database
// config/user-service/redis
// config/user-service/feature-flags

// 3. Kubernetes ConfigMap
// configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  DATABASE_PORT: "5432"
  LOG_LEVEL: "info"

// 4. Kubernetes Secret
// secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQ=  # base64
  JWT_SECRET: c2VjcmV0

// 在 Pod 中使用
// deployment.yaml
spec:
  containers:
    - name: user-service
      envFrom:
        - configMapRef:
            name: user-service-config
        - secretRef:
            name: user-service-secrets
```

### 动态配置

```typescript
// 配置热更新
import Consul from 'consul';

class ConfigWatcher {
  private config: any = {};
  private consul = new Consul();

  async watch(key: string) {
    const watcher = this.consul.watch({
      method: this.consul.kv.get,
      options: { key }
    });

    watcher.on('change', (data) => {
      if (data) {
        this.config[key] = JSON.parse(data.Value);
        console.log(`Config updated: ${key}`);
        this.emit('configChanged', { key, value: this.config[key] });
      }
    });

    watcher.on('error', (err) => {
      console.error('Watch error:', err);
    });
  }

  get(key: string) {
    return this.config[key];
  }
}

// 使用
const configWatcher = new ConfigWatcher();
await configWatcher.watch('config/feature-flags');

configWatcher.on('configChanged', ({ key, value }) => {
  // 更新应用配置
  app.set(key, value);
});
```

---

## API 网关

### 网关功能

```typescript
// API 网关职责
const gatewayFeatures = {
  路由: '将请求路由到对应的微服务',
  认证: '统一认证，验证 JWT/API Key',
  限流: '防止服务过载',
  熔断: '故障隔离',
  负载均衡: '分发请求',
  日志: '请求日志',
  监控: '指标收集',
  协议转换: 'REST ↔ gRPC',
  缓存: '响应缓存',
  请求转换: '添加/修改 Headers'
};
```

### Kong 配置

```yaml
# kong.yaml
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service:3000
    routes:
      - name: users-route
        paths:
          - /api/users
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          second: 100
          policy: local
      - name: jwt
        config:
          claims_to_verify:
            - exp

  - name: order-service
    url: http://order-service:3000
    routes:
      - name: orders-route
        paths:
          - /api/orders

# 全局插件
plugins:
  - name: cors
    config:
      origins:
        - "*"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
      headers:
        - Authorization
        - Content-Type

  - name: request-transformer
    config:
      add:
        headers:
          - X-Request-ID:$(uuid)
```

### 自定义网关

```typescript
// Express 实现简单网关
import express from 'express';
import { createProxyMiddleware } from 'http-proxy-middleware';
import rateLimit from 'express-rate-limit';
import jwt from 'jsonwebtoken';

const app = express();

// 限流
const limiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100
});
app.use(limiter);

// 认证中间件
const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    req.headers['X-User-ID'] = decoded.sub;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// 路由配置
const routes = {
  '/api/users': { target: 'http://user-service:3000', auth: true },
  '/api/orders': { target: 'http://order-service:3000', auth: true },
  '/api/products': { target: 'http://product-service:3000', auth: false },
  '/api/auth': { target: 'http://auth-service:3000', auth: false }
};

// 设置代理
for (const [path, config] of Object.entries(routes)) {
  const middlewares = [];
  
  if (config.auth) {
    middlewares.push(authMiddleware);
  }
  
  middlewares.push(createProxyMiddleware({
    target: config.target,
    changeOrigin: true,
    pathRewrite: { [`^${path}`]: '' },
    onProxyReq: (proxyReq, req) => {
      proxyReq.setHeader('X-Request-ID', req.id);
      proxyReq.setHeader('X-Forwarded-For', req.ip);
    }
  }));
  
  app.use(path, ...middlewares);
}

app.listen(8080);
```

---

## 熔断与限流

### 熔断器模式

```typescript
// Circuit Breaker 实现
enum CircuitState {
  CLOSED = 'CLOSED',       // 正常
  OPEN = 'OPEN',          // 熔断
  HALF_OPEN = 'HALF_OPEN' // 半开
}

interface CircuitBreakerOptions {
  failureThreshold: number;   // 失败阈值
  successThreshold: number;   // 恢复阈值
  timeout: number;            // 熔断时间
}

class CircuitBreaker {
  private state = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime: number | null = null;

  constructor(private options: CircuitBreakerOptions) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailureTime! > this.options.timeout) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
      } else {
        throw new CircuitOpenError('Circuit is open');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.options.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.failureCount = 0;
      }
    } else {
      this.failureCount = 0;
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();

    if (this.failureCount >= this.options.failureThreshold) {
      this.state = CircuitState.OPEN;
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}

// 使用
const userServiceBreaker = new CircuitBreaker({
  failureThreshold: 5,
  successThreshold: 3,
  timeout: 30000
});

async function getUser(id: string) {
  return userServiceBreaker.execute(async () => {
    const response = await axios.get(`http://user-service/users/${id}`);
    return response.data;
  });
}
```

### 限流实现

```typescript
// 1. 令牌桶算法
class TokenBucket {
  private tokens: number;
  private lastRefillTime: number;

  constructor(
    private capacity: number,      // 桶容量
    private refillRate: number     // 每秒补充的令牌数
  ) {
    this.tokens = capacity;
    this.lastRefillTime = Date.now();
  }

  tryAcquire(count: number = 1): boolean {
    this.refill();
    
    if (this.tokens >= count) {
      this.tokens -= count;
      return true;
    }
    
    return false;
  }

  private refill(): void {
    const now = Date.now();
    const elapsed = (now - this.lastRefillTime) / 1000;
    const tokensToAdd = elapsed * this.refillRate;
    
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd);
    this.lastRefillTime = now;
  }
}

// 2. 滑动窗口
class SlidingWindowRateLimiter {
  private requests: number[] = [];

  constructor(
    private windowSize: number,  // 窗口大小（毫秒）
    private maxRequests: number  // 最大请求数
  ) {}

  tryAcquire(): boolean {
    const now = Date.now();
    const windowStart = now - this.windowSize;

    // 清理过期请求
    this.requests = this.requests.filter(time => time > windowStart);

    if (this.requests.length < this.maxRequests) {
      this.requests.push(now);
      return true;
    }

    return false;
  }
}

// 3. 分布式限流（Redis）
import Redis from 'ioredis';

class RedisRateLimiter {
  constructor(private redis: Redis) {}

  async tryAcquire(key: string, limit: number, window: number): Promise<boolean> {
    const now = Date.now();
    const windowKey = `ratelimit:${key}:${Math.floor(now / (window * 1000))}`;

    const count = await this.redis.incr(windowKey);
    
    if (count === 1) {
      await this.redis.expire(windowKey, window);
    }

    return count <= limit;
  }
}

// Express 中间件
function rateLimitMiddleware(limiter: RedisRateLimiter) {
  return async (req, res, next) => {
    const key = req.ip;
    const allowed = await limiter.tryAcquire(key, 100, 60);
    
    if (!allowed) {
      return res.status(429).json({ error: 'Too many requests' });
    }
    
    next();
  };
}
```

---

## 分布式事务

### Saga 模式

```typescript
// 编排式 Saga
interface SagaStep {
  name: string;
  execute: () => Promise<any>;
  compensate: () => Promise<void>;
}

class SagaOrchestrator {
  private executedSteps: SagaStep[] = [];

  async execute(steps: SagaStep[]): Promise<void> {
    try {
      for (const step of steps) {
        console.log(`Executing step: ${step.name}`);
        await step.execute();
        this.executedSteps.push(step);
      }
    } catch (error) {
      console.log('Saga failed, compensating...');
      await this.compensate();
      throw error;
    }
  }

  private async compensate(): Promise<void> {
    // 反向补偿
    for (const step of this.executedSteps.reverse()) {
      try {
        console.log(`Compensating step: ${step.name}`);
        await step.compensate();
      } catch (error) {
        console.error(`Compensation failed for ${step.name}:`, error);
      }
    }
  }
}

// 创建订单 Saga
class CreateOrderSaga {
  constructor(
    private orderService: OrderService,
    private inventoryService: InventoryService,
    private paymentService: PaymentService
  ) {}

  async execute(orderData: any): Promise<string> {
    const saga = new SagaOrchestrator();
    let orderId: string;
    let reservationId: string;
    let paymentId: string;

    const steps: SagaStep[] = [
      {
        name: 'createOrder',
        execute: async () => {
          orderId = await this.orderService.create(orderData);
          return orderId;
        },
        compensate: async () => {
          await this.orderService.cancel(orderId);
        }
      },
      {
        name: 'reserveInventory',
        execute: async () => {
          reservationId = await this.inventoryService.reserve(
            orderId,
            orderData.items
          );
          return reservationId;
        },
        compensate: async () => {
          await this.inventoryService.release(reservationId);
        }
      },
      {
        name: 'processPayment',
        execute: async () => {
          paymentId = await this.paymentService.charge(
            orderData.userId,
            orderData.total
          );
          return paymentId;
        },
        compensate: async () => {
          await this.paymentService.refund(paymentId);
        }
      },
      {
        name: 'confirmOrder',
        execute: async () => {
          await this.orderService.confirm(orderId);
        },
        compensate: async () => {
          // 最后一步无需补偿
        }
      }
    ];

    await saga.execute(steps);
    return orderId;
  }
}
```

### TCC 模式

```typescript
// Try-Confirm-Cancel 模式

interface TccParticipant {
  try(): Promise<void>;
  confirm(): Promise<void>;
  cancel(): Promise<void>;
}

class TccCoordinator {
  async execute(participants: TccParticipant[]): Promise<void> {
    const tried: TccParticipant[] = [];

    try {
      // Try 阶段
      for (const participant of participants) {
        await participant.try();
        tried.push(participant);
      }

      // Confirm 阶段
      for (const participant of tried) {
        await participant.confirm();
      }
    } catch (error) {
      // Cancel 阶段
      for (const participant of tried) {
        try {
          await participant.cancel();
        } catch (cancelError) {
          console.error('Cancel failed:', cancelError);
        }
      }
      throw error;
    }
  }
}

// 库存 TCC 参与者
class InventoryTccParticipant implements TccParticipant {
  private reservationId?: string;

  constructor(
    private items: OrderItem[],
    private inventoryService: InventoryService
  ) {}

  async try(): Promise<void> {
    // 预留库存（冻结）
    this.reservationId = await this.inventoryService.reserve(this.items);
  }

  async confirm(): Promise<void> {
    // 确认扣减
    await this.inventoryService.confirmReservation(this.reservationId!);
  }

  async cancel(): Promise<void> {
    // 释放预留
    await this.inventoryService.cancelReservation(this.reservationId!);
  }
}
```

---

## 部署策略

### 蓝绿部署

```yaml
# 蓝环境
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service-blue
  labels:
    app: user-service
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
      version: blue
  template:
    metadata:
      labels:
        app: user-service
        version: blue
    spec:
      containers:
        - name: user-service
          image: user-service:v1.0.0

---
# 绿环境
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service-green
  labels:
    app: user-service
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
      version: green
  template:
    metadata:
      labels:
        app: user-service
        version: green
    spec:
      containers:
        - name: user-service
          image: user-service:v2.0.0

---
# Service 切换
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
    version: blue  # 切换到 green 即完成部署
  ports:
    - port: 80
      targetPort: 3000
```

### 金丝雀发布

```yaml
# Istio 金丝雀发布
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
    - user-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: user-service
            subset: canary
    - route:
        - destination:
            host: user-service
            subset: stable
          weight: 90
        - destination:
            host: user-service
            subset: canary
          weight: 10

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

---

## 常见面试题

### 1. 微服务如何拆分？

**原则**：
- 按业务领域（DDD 限界上下文）
- 按团队结构（康威定律）
- 单一职责
- 高内聚低耦合

### 2. 微服务之间如何通信？

| 方式 | 同步/异步 | 适用场景 |
|------|---------|---------|
| **REST** | 同步 | 简单调用 |
| **gRPC** | 同步 | 高性能调用 |
| **消息队列** | 异步 | 事件驱动、解耦 |
| **GraphQL** | 同步 | 聚合查询 |

### 3. 什么是熔断器？

**熔断器**：防止故障扩散

**三种状态**：
- **CLOSED**：正常，请求通过
- **OPEN**：熔断，请求直接失败
- **HALF_OPEN**：半开，允许部分请求测试

### 4. 如何处理分布式事务？

**方案**：
- **Saga**：通过补偿操作保证最终一致
- **TCC**：Try-Confirm-Cancel 两阶段
- **本地消息表**：保证消息可靠发送
- **事务消息**：消息队列支持的事务

### 5. 微服务的挑战？

- **分布式复杂性**
- **数据一致性**
- **服务发现**
- **配置管理**
- **监控和追踪**
- **测试复杂度**
- **运维成本**

---

## 总结

### 微服务关键技术

| 领域 | 技术选型 |
|------|---------|
| **服务通信** | REST, gRPC, Kafka |
| **服务发现** | Kubernetes, Consul |
| **API 网关** | Kong, AWS API Gateway |
| **熔断限流** | Hystrix, Resilience4j |
| **配置管理** | Consul KV, ConfigMap |
| **分布式事务** | Saga, TCC |
| **部署** | 蓝绿、金丝雀 |

### 实践检查清单

- [ ] 合理拆分服务
- [ ] 选择合适的通信方式
- [ ] 实现服务发现
- [ ] 配置 API 网关
- [ ] 实现熔断和限流
- [ ] 处理分布式事务
- [ ] 选择合适的部署策略

---

**上一篇**：[领域驱动设计](./03-ddd.md)  
**下一篇**：[设计原则](./05-design-principles.md)

