# 架构模式

架构模式决定了系统的整体结构和组织方式。本文深入讲解常见的架构模式及其适用场景。

## 目录
- [单体架构](#单体架构)
- [微服务架构](#微服务架构)
- [SOA 架构](#soa-架构)
- [Serverless 架构](#serverless-架构)
- [事件驱动架构](#事件驱动架构)
- [架构选型](#架构选型)
- [面试题](#常见面试题)

---

## 单体架构

### 什么是单体架构

```
单体架构（Monolithic Architecture）：
将所有功能模块打包在一个应用中部署运行

┌─────────────────────────────────────────┐
│              单体应用                     │
│  ┌─────────┬─────────┬─────────┐       │
│  │ 用户模块 │ 订单模块 │ 商品模块 │       │
│  └─────────┴─────────┴─────────┘       │
│  ┌─────────────────────────────┐       │
│  │        共享数据库            │       │
│  └─────────────────────────────┘       │
└─────────────────────────────────────────┘
```

### 单体架构特点

```typescript
// 典型的单体应用结构
src/
├── controllers/
│   ├── userController.ts
│   ├── orderController.ts
│   └── productController.ts
├── services/
│   ├── userService.ts
│   ├── orderService.ts
│   └── productService.ts
├── models/
│   ├── User.ts
│   ├── Order.ts
│   └── Product.ts
├── routes/
│   └── index.ts
├── middlewares/
├── utils/
├── config/
└── app.ts

// 所有模块在同一进程中运行
// 共享同一数据库
// 统一部署
```

### 优缺点

**优点**：
```typescript
const advantages = {
  开发简单: '单一代码库，IDE 友好',
  部署简单: '一个包部署',
  测试简单: '端到端测试方便',
  性能: '进程内调用，无网络开销',
  事务: 'ACID 事务简单',
  调试: '本地调试方便'
};
```

**缺点**：
```typescript
const disadvantages = {
  扩展性差: '只能整体扩展，无法按模块扩展',
  部署风险: '任何改动需要重新部署整个应用',
  技术栈受限: '必须使用统一技术栈',
  团队协作: '代码冲突多，难以并行开发',
  启动慢: '应用越大启动越慢',
  故障影响大: '一个模块故障可能影响整个应用'
};
```

### 模块化单体

```typescript
// 模块化单体：在单体内部实现模块隔离

// 项目结构
src/
├── modules/
│   ├── user/
│   │   ├── controllers/
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── dto/
│   │   └── index.ts        // 模块入口
│   ├── order/
│   │   ├── controllers/
│   │   ├── services/
│   │   └── index.ts
│   └── product/
│       └── index.ts
├── shared/                  // 共享代码
│   ├── database/
│   ├── utils/
│   └── middlewares/
└── app.ts

// 模块间通过明确的接口通信
// user/index.ts
export interface UserModule {
  getUserById(id: string): Promise<User>;
  createUser(dto: CreateUserDto): Promise<User>;
}

export class UserModuleImpl implements UserModule {
  constructor(
    private userService: UserService,
    private userRepository: UserRepository
  ) {}

  async getUserById(id: string): Promise<User> {
    return this.userService.findById(id);
  }
}

// 其他模块通过接口调用，而非直接依赖实现
// order/services/orderService.ts
class OrderService {
  constructor(private userModule: UserModule) {}

  async createOrder(userId: string, items: any[]) {
    const user = await this.userModule.getUserById(userId);
    // ...
  }
}
```

### 适用场景

```typescript
const suitableScenarios = [
  '初创公司/MVP 阶段',
  '小型团队（<10人）',
  '业务相对简单',
  '快速迭代需求',
  '有限的运维能力',
  '对响应时间要求高的实时系统'
];

const unsuitableScenarios = [
  '大型团队',
  '复杂业务领域',
  '需要独立扩展某些模块',
  '需要使用不同技术栈',
  '频繁部署需求'
];
```

---

## 微服务架构

### 什么是微服务

```
微服务架构（Microservices Architecture）：
将应用拆分为多个小型、独立部署的服务

┌─────────┐  ┌─────────┐  ┌─────────┐
│ 用户服务 │  │ 订单服务 │  │ 商品服务 │
│  :3001  │  │  :3002  │  │  :3003  │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ User DB │  │ Order DB│  │Product  │
│(Postgres)│  │ (MySQL) │  │DB(Mongo)│
└─────────┘  └─────────┘  └─────────┘
```

### 微服务特点

```typescript
// 1. 服务独立
// 每个服务有自己的代码库、数据库、部署流程

// user-service/
├── src/
├── package.json
├── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── prisma/
    └── schema.prisma  // 独立数据库

// 2. 单一职责
// 每个服务只负责一个业务领域

// 3. 独立部署
// 可以独立更新某个服务，不影响其他服务

// 4. 技术多样性
// 不同服务可以使用不同的技术栈
const services = {
  userService: { language: 'Node.js', db: 'PostgreSQL' },
  orderService: { language: 'Java', db: 'MySQL' },
  recommendService: { language: 'Python', db: 'Redis' },
  searchService: { language: 'Go', db: 'Elasticsearch' }
};

// 5. 分布式系统
// 服务通过网络通信
```

### 服务拆分原则

```typescript
// 1. 按业务领域拆分（DDD 限界上下文）
const domainBoundaries = {
  userContext: ['用户注册', '登录', '个人信息'],
  orderContext: ['下单', '订单状态', '订单历史'],
  paymentContext: ['支付', '退款', '账单'],
  inventoryContext: ['库存', '预留', '发货'],
  productContext: ['商品', '分类', '搜索']
};

// 2. 按变化频率拆分
// 变化频繁的模块独立出来

// 3. 按团队结构拆分（康威定律）
// 服务边界对应团队边界

// 4. 按扩展性需求拆分
// 高负载模块独立，便于独立扩展

// 拆分原则
const principles = {
  highCohesion: '高内聚 - 相关功能放在一起',
  looseCoupling: '低耦合 - 服务间依赖最小化',
  singleResponsibility: '单一职责 - 一个服务只做一件事',
  autonomy: '自治性 - 服务可以独立开发、部署、扩展',
  dataOwnership: '数据所有权 - 每个服务管理自己的数据'
};
```

### 服务通信

```typescript
// 1. 同步通信 - REST
import axios from 'axios';

class OrderService {
  async createOrder(userId: string, items: any[]) {
    // 调用用户服务
    const user = await axios.get(`http://user-service/users/${userId}`);
    
    // 调用库存服务
    await axios.post('http://inventory-service/reserve', { items });
    
    // 创建订单
    return this.orderRepository.create({ userId, items });
  }
}

// 2. 同步通信 - gRPC
// user.proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
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
import { UserServiceClient } from './generated/user_grpc_pb';
import { GetUserRequest } from './generated/user_pb';

const client = new UserServiceClient('user-service:50051', grpc.credentials.createInsecure());

const request = new GetUserRequest();
request.setId('123');

client.getUser(request, (err, response) => {
  console.log(response.getName());
});

// 3. 异步通信 - 消息队列
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const producer = kafka.producer();

// 发布事件
await producer.send({
  topic: 'order-created',
  messages: [
    { value: JSON.stringify({ orderId: '123', userId: '456', items: [] }) }
  ]
});

// 消费事件
const consumer = kafka.consumer({ groupId: 'inventory-service' });
await consumer.subscribe({ topic: 'order-created' });

await consumer.run({
  eachMessage: async ({ message }) => {
    const order = JSON.parse(message.value.toString());
    await reserveInventory(order.items);
  }
});
```

### 数据管理

```typescript
// 1. 数据库独立（Database per Service）
// 每个服务有自己的数据库，不共享

// 2. 分布式事务 - Saga 模式
class CreateOrderSaga {
  async execute(orderData: any) {
    try {
      // Step 1: 创建订单（Pending 状态）
      const order = await this.orderService.create(orderData);

      // Step 2: 扣减库存
      await this.inventoryService.reserve(order.items);

      // Step 3: 扣款
      await this.paymentService.charge(order.userId, order.total);

      // Step 4: 确认订单
      await this.orderService.confirm(order.id);

      return order;
    } catch (error) {
      // 补偿操作
      await this.compensate(error);
      throw error;
    }
  }

  private async compensate(error: any) {
    // 根据失败阶段执行补偿
    if (error.step === 'payment') {
      await this.inventoryService.release(error.orderId);
    }
    await this.orderService.cancel(error.orderId);
  }
}

// 3. 数据一致性 - 最终一致性
// 通过事件驱动实现数据同步
class UserService {
  async updateUser(id: string, data: any) {
    const user = await this.userRepository.update(id, data);
    
    // 发布事件，其他服务订阅更新本地数据
    await this.eventBus.publish('user-updated', { id, ...data });
    
    return user;
  }
}
```

### 服务发现

```typescript
// 1. 客户端发现（Client-side Discovery）
// 客户端查询注册中心，自己选择实例

// 2. 服务端发现（Server-side Discovery）
// 通过负载均衡器转发，客户端不感知

// Kubernetes 服务发现
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

// 其他服务通过 DNS 访问
// http://user-service/users/123

// Consul 服务注册
import Consul from 'consul';

const consul = new Consul();

// 注册服务
await consul.agent.service.register({
  name: 'user-service',
  id: 'user-service-1',
  address: '10.0.0.1',
  port: 3000,
  check: {
    http: 'http://10.0.0.1:3000/health',
    interval: '10s'
  }
});

// 发现服务
const services = await consul.catalog.service.nodes('user-service');
const instances = services.map(s => `${s.ServiceAddress}:${s.ServicePort}`);
```

---

## SOA 架构

### SOA vs 微服务

```
SOA（Service-Oriented Architecture）面向服务架构

┌─────────────────────────────────────────────┐
│               ESB（企业服务总线）              │
├─────────────────────────────────────────────┤
│    ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐      │
│    │服务A│  │服务B│  │服务C│  │服务D│      │
│    └─────┘  └─────┘  └─────┘  └─────┘      │
└─────────────────────────────────────────────┘

微服务：去中心化，服务直接通信
```

| 特性 | SOA | 微服务 |
|------|-----|--------|
| **粒度** | 粗粒度 | 细粒度 |
| **通信** | ESB 集中式 | 去中心化 |
| **数据** | 共享数据库 | 独立数据库 |
| **治理** | 集中治理 | 分布式治理 |
| **技术栈** | 统一 | 多样 |
| **部署** | 整体部署 | 独立部署 |

---

## Serverless 架构

### 什么是 Serverless

```
Serverless = FaaS + BaaS

FaaS（Function as a Service）：
- 按需执行函数
- 自动扩缩容
- 按调用次数计费

BaaS（Backend as a Service）：
- 托管数据库（DynamoDB、Firebase）
- 认证服务（Cognito、Auth0）
- 存储服务（S3）
- 消息服务（SNS、SQS）
```

### AWS Lambda 架构

```typescript
// Lambda 函数
import { APIGatewayProxyHandler, APIGatewayProxyResult } from 'aws-lambda';
import { DynamoDB } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocument } from '@aws-sdk/lib-dynamodb';

const ddb = DynamoDBDocument.from(new DynamoDB());

export const handler: APIGatewayProxyHandler = async (event): Promise<APIGatewayProxyResult> => {
  const { httpMethod, pathParameters, body } = event;

  try {
    if (httpMethod === 'GET' && pathParameters?.id) {
      const result = await ddb.get({
        TableName: process.env.TABLE_NAME!,
        Key: { id: pathParameters.id }
      });

      return {
        statusCode: 200,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(result.Item)
      };
    }

    if (httpMethod === 'POST') {
      const data = JSON.parse(body || '{}');
      
      await ddb.put({
        TableName: process.env.TABLE_NAME!,
        Item: { id: crypto.randomUUID(), ...data, createdAt: new Date().toISOString() }
      });

      return {
        statusCode: 201,
        body: JSON.stringify({ success: true })
      };
    }

    return { statusCode: 404, body: 'Not Found' };
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};
```

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: nodejs18.x
  environment:
    TABLE_NAME: ${self:service}-${self:provider.stage}

functions:
  api:
    handler: src/handler.handler
    events:
      - http:
          path: /users/{id}
          method: get
      - http:
          path: /users
          method: post

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
```

### Serverless 优缺点

**优点**：
```typescript
const advantages = {
  无服务器管理: '无需管理服务器',
  自动扩缩容: '按需扩展，从 0 到无限',
  按使用付费: '只为实际调用付费',
  高可用: '内置高可用',
  快速部署: '秒级部署',
  专注业务: '减少运维工作'
};
```

**缺点**：
```typescript
const disadvantages = {
  冷启动: '首次调用延迟',
  执行时间限制: 'Lambda 最多 15 分钟',
  状态管理: '无状态，需要外部存储',
  调试困难: '本地调试复杂',
  供应商锁定: '依赖特定云平台',
  复杂系统难以管理: '函数过多时管理困难'
};
```

### 最佳实践

```typescript
// 1. 减少冷启动
// - 使用 Provisioned Concurrency
// - 减少依赖包大小
// - 使用 Lambda Layers

// 2. 代码组织
// 按功能拆分 Lambda，避免单个 Lambda 过大

// 3. 错误处理
// - DLQ（死信队列）处理失败
// - 重试策略

// 4. 监控
// - CloudWatch Logs
// - X-Ray 追踪

// 5. 安全
// - 最小权限 IAM 角色
// - 环境变量加密
// - VPC 内执行
```

---

## 事件驱动架构

### 什么是事件驱动架构

```
事件驱动架构（Event-Driven Architecture, EDA）：
组件通过事件进行异步通信

┌─────────┐    事件     ┌─────────────┐    事件     ┌─────────┐
│ 生产者  │ ─────────→ │  事件总线   │ ─────────→ │ 消费者  │
└─────────┘            └─────────────┘            └─────────┘
                              │
                              ▼
                       ┌─────────────┐
                       │  事件存储   │
                       └─────────────┘
```

### 事件溯源（Event Sourcing）

```typescript
// 传统方式：存储当前状态
// User: { id: 1, name: 'John', email: 'john@example.com', balance: 100 }

// 事件溯源：存储所有事件
const events = [
  { type: 'UserCreated', data: { id: 1, name: 'John', email: 'john@example.com' }, timestamp: '...' },
  { type: 'BalanceAdded', data: { amount: 100 }, timestamp: '...' },
  { type: 'EmailChanged', data: { email: 'john.doe@example.com' }, timestamp: '...' },
  { type: 'BalanceDeducted', data: { amount: 50 }, timestamp: '...' }
];

// 通过重放事件得到当前状态
function replay(events: Event[]): User {
  return events.reduce((user, event) => {
    switch (event.type) {
      case 'UserCreated':
        return { ...event.data, balance: 0 };
      case 'BalanceAdded':
        return { ...user, balance: user.balance + event.data.amount };
      case 'BalanceDeducted':
        return { ...user, balance: user.balance - event.data.amount };
      case 'EmailChanged':
        return { ...user, email: event.data.email };
      default:
        return user;
    }
  }, {} as User);
}

// 事件存储实现
class EventStore {
  private events: Map<string, Event[]> = new Map();

  async append(aggregateId: string, event: Event): Promise<void> {
    const events = this.events.get(aggregateId) || [];
    events.push({
      ...event,
      id: crypto.randomUUID(),
      timestamp: new Date(),
      version: events.length + 1
    });
    this.events.set(aggregateId, events);
  }

  async getEvents(aggregateId: string): Promise<Event[]> {
    return this.events.get(aggregateId) || [];
  }

  async getEventsAfter(aggregateId: string, version: number): Promise<Event[]> {
    const events = this.events.get(aggregateId) || [];
    return events.filter(e => e.version > version);
  }
}
```

### CQRS

```typescript
// CQRS（Command Query Responsibility Segregation）
// 命令查询职责分离

// 写模型（Command）
class OrderCommandHandler {
  constructor(
    private eventStore: EventStore,
    private eventBus: EventBus
  ) {}

  async handle(command: CreateOrderCommand): Promise<void> {
    // 业务逻辑验证
    const order = Order.create(command);

    // 存储事件
    for (const event of order.getUncommittedEvents()) {
      await this.eventStore.append(order.id, event);
      await this.eventBus.publish(event);
    }
  }
}

// 读模型（Query）
class OrderQueryHandler {
  constructor(private orderReadRepository: OrderReadRepository) {}

  async handle(query: GetOrderQuery): Promise<OrderDTO> {
    // 直接从读模型查询
    return this.orderReadRepository.findById(query.orderId);
  }

  async handleList(query: GetOrdersQuery): Promise<OrderDTO[]> {
    return this.orderReadRepository.findByUserId(query.userId);
  }
}

// 读模型同步（事件处理器）
class OrderProjection {
  constructor(private orderReadRepository: OrderReadRepository) {}

  @EventHandler('OrderCreated')
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    await this.orderReadRepository.insert({
      id: event.orderId,
      userId: event.userId,
      status: 'created',
      items: event.items,
      total: event.total,
      createdAt: event.timestamp
    });
  }

  @EventHandler('OrderConfirmed')
  async onOrderConfirmed(event: OrderConfirmedEvent): Promise<void> {
    await this.orderReadRepository.updateStatus(event.orderId, 'confirmed');
  }
}
```

### Saga 模式

```typescript
// Saga：分布式事务的一种实现
// 通过一系列本地事务和补偿操作保证一致性

// 编排式 Saga（Orchestration）
class OrderSagaOrchestrator {
  async createOrder(orderData: any): Promise<void> {
    const sagaId = crypto.randomUUID();
    const saga: SagaState = {
      id: sagaId,
      currentStep: 0,
      data: orderData,
      status: 'running'
    };

    const steps: SagaStep[] = [
      {
        name: 'createOrder',
        execute: () => this.orderService.create(orderData),
        compensate: () => this.orderService.cancel(saga.data.orderId)
      },
      {
        name: 'reserveInventory',
        execute: () => this.inventoryService.reserve(orderData.items),
        compensate: () => this.inventoryService.release(orderData.items)
      },
      {
        name: 'processPayment',
        execute: () => this.paymentService.charge(orderData.userId, orderData.total),
        compensate: () => this.paymentService.refund(saga.data.paymentId)
      },
      {
        name: 'confirmOrder',
        execute: () => this.orderService.confirm(saga.data.orderId),
        compensate: () => {} // 最后一步无需补偿
      }
    ];

    try {
      for (const step of steps) {
        const result = await step.execute();
        saga.data = { ...saga.data, ...result };
        saga.currentStep++;
      }
      saga.status = 'completed';
    } catch (error) {
      // 反向补偿
      for (let i = saga.currentStep - 1; i >= 0; i--) {
        await steps[i].compensate();
      }
      saga.status = 'compensated';
      throw error;
    }
  }
}

// 协同式 Saga（Choreography）
// 每个服务发布事件，其他服务订阅并做出反应

// order-service
class OrderService {
  async createOrder(data: any): Promise<void> {
    const order = await this.orderRepository.create(data);
    await this.eventBus.publish('OrderCreated', { orderId: order.id, ...data });
  }

  @EventHandler('PaymentCompleted')
  async onPaymentCompleted(event: PaymentCompletedEvent): Promise<void> {
    await this.orderRepository.updateStatus(event.orderId, 'confirmed');
  }

  @EventHandler('PaymentFailed')
  async onPaymentFailed(event: PaymentFailedEvent): Promise<void> {
    await this.orderRepository.updateStatus(event.orderId, 'cancelled');
  }
}

// inventory-service
class InventoryService {
  @EventHandler('OrderCreated')
  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    try {
      await this.reserve(event.items);
      await this.eventBus.publish('InventoryReserved', { orderId: event.orderId });
    } catch (error) {
      await this.eventBus.publish('InventoryReservationFailed', { orderId: event.orderId });
    }
  }
}

// payment-service
class PaymentService {
  @EventHandler('InventoryReserved')
  async onInventoryReserved(event: InventoryReservedEvent): Promise<void> {
    try {
      await this.charge(event.userId, event.total);
      await this.eventBus.publish('PaymentCompleted', { orderId: event.orderId });
    } catch (error) {
      await this.eventBus.publish('PaymentFailed', { orderId: event.orderId });
    }
  }
}
```

---

## 架构选型

### 选型决策树

```
项目规模？
├── 小型/MVP → 单体架构
│   └── 团队 < 10 人
│   └── 业务简单
│   └── 快速迭代
│
├── 中型 → 模块化单体 或 微服务
│   └── 团队 10-50 人
│   └── 业务复杂度中等
│   └── 需要一定扩展性
│
└── 大型 → 微服务
    └── 团队 > 50 人
    └── 业务复杂
    └── 高扩展性需求
    └── 多团队并行开发
```

### 对比总结

| 架构 | 适用场景 | 复杂度 | 扩展性 | 运维成本 |
|------|---------|--------|--------|---------|
| **单体** | 小型项目、MVP | 低 | 低 | 低 |
| **模块化单体** | 中型项目 | 中 | 中 | 中 |
| **微服务** | 大型项目、复杂业务 | 高 | 高 | 高 |
| **Serverless** | 事件驱动、不规则流量 | 中 | 高 | 低 |
| **事件驱动** | 异步处理、解耦 | 高 | 高 | 中 |

---

## 常见面试题

### 1. 单体 vs 微服务，如何选择？

**考虑因素**：
- **团队规模**：< 10 人考虑单体
- **业务复杂度**：简单业务用单体
- **扩展需求**：需要独立扩展用微服务
- **部署频率**：频繁部署考虑微服务
- **运维能力**：运维能力弱用单体

### 2. 微服务的挑战？

- **分布式复杂性**：网络延迟、分区容错
- **数据一致性**：分布式事务困难
- **服务发现**：服务实例动态变化
- **运维复杂**：多服务部署、监控
- **调试困难**：分布式追踪
- **测试复杂**：集成测试困难

### 3. 什么是事件溯源？

**事件溯源**：
- 存储所有事件而非当前状态
- 通过重放事件得到当前状态
- 完整的审计日志
- 支持时间旅行查询

### 4. CQRS 的优缺点？

**优点**：
- 读写分离，独立优化
- 读模型可针对查询优化
- 支持多种读模型

**缺点**：
- 复杂度增加
- 数据最终一致性
- 同步延迟

### 5. Serverless 适合什么场景？

**适合**：
- 事件驱动处理
- 不规则流量
- 快速原型
- 定时任务
- API 后端

**不适合**：
- 长时间运行任务
- 有状态应用
- 实时系统（冷启动问题）

---

## 总结

### 架构演进路径

```
单体 → 模块化单体 → 微服务
         ↓
    事件驱动架构
         ↓
    CQRS + Event Sourcing
```

### 实践检查清单

- [ ] 理解各种架构模式
- [ ] 能根据场景选择架构
- [ ] 掌握微服务拆分原则
- [ ] 理解服务通信方式
- [ ] 理解事件驱动架构
- [ ] 了解 CQRS 和事件溯源
- [ ] 掌握 Saga 分布式事务

---

**下一篇**：[分层架构](./02-layered-architecture.md)

