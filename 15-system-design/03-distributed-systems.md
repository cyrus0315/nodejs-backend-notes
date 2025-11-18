# 分布式系统设计

分布式系统是由多台计算机组成的系统，通过网络协作完成任务。本文深入讲解分布式系统的核心概念、理论基础和实战方案。

## 目录
- [分布式理论](#分布式理论)
- [分布式事务](#分布式事务)
- [分布式锁](#分布式锁)
- [分布式 ID](#分布式-id)
- [一致性算法](#一致性算法)
- [Node.js 实战](#nodejs-实战)
- [面试题](#常见面试题)

---

## 分布式理论

### CAP 理论

**CAP 定理**：分布式系统只能同时满足以下三项中的两项。

```
       C (一致性)
        / \
       /   \
      /     \
     /       \
    /         \
   /           \
  A ----------- P
(可用性)    (分区容错性)
```

#### C - Consistency（一致性）

所有节点在同一时刻看到的数据是一致的。

```typescript
// 强一致性示例
// 写入后立即读取，能读到最新值
await db.users.update({ where: { id: 1 }, data: { name: 'Alice' } });
const user = await db.users.findUnique({ where: { id: 1 } });
console.log(user.name); // 一定是 'Alice'
```

#### A - Availability（可用性）

系统始终能返回结果（成功或失败），不会一直等待。

```typescript
// 高可用示例
// 即使部分节点故障，系统仍然能响应
async function getUser(id: number) {
  try {
    return await primaryDB.findUser(id);
  } catch (error) {
    // 主库故障，降级到从库
    return await replicaDB.findUser(id);
  }
}
```

#### P - Partition Tolerance（分区容错性）

系统在网络分区（节点间通信故障）时仍能继续工作。

```typescript
// 网络分区场景
// 节点 A 和 B 无法通信，但系统仍能工作
```

### CAP 权衡

**重要**：网络分区是客观存在的，所以实际上是在 **CP** 和 **AP** 之间选择。

#### CP 系统（一致性 + 分区容错性）

牺牲可用性，保证一致性。

**特点**：
- 分区时，部分节点不可用
- 数据强一致
- 适合金融、账务系统

**代表**：
- HBase
- MongoDB（强一致性模式）
- ZooKeeper
- etcd

```typescript
// CP 示例：Zookeeper
import { createClient } from 'node-zookeeper-client';

const zk = createClient('localhost:2181');

// 分布式配置（强一致）
async function getConfig(key: string): Promise<string> {
  return new Promise((resolve, reject) => {
    // 如果与 leader 无法通信，读取会失败（牺牲可用性）
    zk.getData(`/config/${key}`, (error, data) => {
      if (error) {
        reject(error); // 不可用
      } else {
        resolve(data.toString());
      }
    });
  });
}
```

#### AP 系统（可用性 + 分区容错性）

牺牲一致性（最终一致），保证可用性。

**特点**：
- 分区时，所有节点仍可访问
- 数据可能短暂不一致
- 适合社交、内容系统

**代表**：
- Cassandra
- DynamoDB
- Eureka（服务注册中心）

```typescript
// AP 示例：Cassandra
import { Client } from 'cassandra-driver';

const client = new Client({
  contactPoints: ['node1', 'node2', 'node3'],
  localDataCenter: 'datacenter1',
  // ONE: 只要一个节点响应即可（高可用）
  consistency: 'ONE' 
});

// 即使部分节点故障，仍能读写（可能读到旧数据）
const result = await client.execute(
  'SELECT * FROM users WHERE id = ?',
  [userId]
);
```

### BASE 理论

BASE 是对 CAP 的延伸，AP 系统的实践。

#### Basically Available（基本可用）

系统允许部分功能降级。

```typescript
// 示例：推荐系统降级
async function getRecommendations(userId: number) {
  try {
    // 尝试个性化推荐（可能失败）
    return await mlService.getPersonalizedRecs(userId);
  } catch (error) {
    // 降级到热门推荐（基本可用）
    return await cacheService.getHotRecs();
  }
}
```

#### Soft State（软状态）

允许数据存在中间状态，不同节点数据可能不一致。

```typescript
// 示例：订单状态
enum OrderState {
  PENDING = 'pending',           // 待支付
  PROCESSING = 'processing',     // 支付中（中间状态）
  PAID = 'paid',                 // 已支付
  SHIPPING = 'shipping',         // 配送中（中间状态）
  COMPLETED = 'completed'        // 已完成
}
```

#### Eventually Consistent（最终一致性）

系统保证最终会达到一致状态。

```typescript
// 示例：用户积分
// 1. 用户下单，立即返回
await createOrder(order);

// 2. 异步更新积分（最终一致）
await messageQueue.publish('order.created', {
  userId: order.userId,
  points: order.amount * 0.01
});

// 积分服务消费消息
messageQueue.subscribe('order.created', async (event) => {
  await userService.addPoints(event.userId, event.points);
});
```

---

## 分布式事务

### 问题场景

```typescript
// 跨服务的事务场景：订单系统
async function createOrder(order: Order) {
  // 1. 订单服务：创建订单
  await orderService.create(order);
  
  // 2. 库存服务：扣减库存
  await inventoryService.deduct(order.productId, order.quantity);
  
  // 3. 支付服务：扣款
  await paymentService.charge(order.userId, order.amount);
  
  // 问题：任何一步失败，如何回滚前面的操作？
}
```

### 1. 两阶段提交（2PC）

**流程**：

```
阶段 1：准备阶段（Prepare）
  协调者 → 参与者：你能提交吗？
  参与者 → 协调者：可以 / 不可以

阶段 2：提交阶段（Commit）
  如果所有参与者都说"可以"：
    协调者 → 参与者：提交
  否则：
    协调者 → 参与者：回滚
```

**实现**：

```typescript
interface Participant {
  prepare(): Promise<boolean>;
  commit(): Promise<void>;
  rollback(): Promise<void>;
}

class TwoPhaseCommit {
  constructor(private participants: Participant[]) {}

  async execute(): Promise<boolean> {
    // 阶段 1：准备
    const prepared: boolean[] = [];
    
    try {
      for (const participant of this.participants) {
        const canCommit = await participant.prepare();
        prepared.push(canCommit);
        
        if (!canCommit) {
          // 有参与者无法提交，终止
          await this.rollbackAll();
          return false;
        }
      }
    } catch (error) {
      await this.rollbackAll();
      return false;
    }

    // 阶段 2：提交
    try {
      for (const participant of this.participants) {
        await participant.commit();
      }
      return true;
    } catch (error) {
      // 提交失败，尝试回滚（可能部分成功）
      await this.rollbackAll();
      return false;
    }
  }

  private async rollbackAll() {
    for (const participant of this.participants) {
      try {
        await participant.rollback();
      } catch (error) {
        console.error('Rollback failed:', error);
        // 记录到数据库，人工介入
      }
    }
  }
}

// 订单服务参与者
class OrderServiceParticipant implements Participant {
  private tempOrder: Order | null = null;

  async prepare(): Promise<boolean> {
    try {
      // 创建临时订单（不提交）
      this.tempOrder = await orderDB.createTemp(this.orderData);
      return true;
    } catch (error) {
      return false;
    }
  }

  async commit(): Promise<void> {
    // 提交订单
    await orderDB.commit(this.tempOrder.id);
  }

  async rollback(): Promise<void> {
    // 删除临时订单
    if (this.tempOrder) {
      await orderDB.delete(this.tempOrder.id);
    }
  }
}

// 使用
const transaction = new TwoPhaseCommit([
  new OrderServiceParticipant(orderData),
  new InventoryServiceParticipant(productId, quantity),
  new PaymentServiceParticipant(userId, amount)
]);

const success = await transaction.execute();
```

**优点**：
- ✅ 强一致性
- ✅ 理论完善

**缺点**：
- ❌ 同步阻塞（性能差）
- ❌ 单点故障（协调者）
- ❌ 数据不一致（阶段2失败）

### 2. TCC（Try-Confirm-Cancel）

**流程**：

```
Try：预留资源
Confirm：确认提交
Cancel：取消，释放资源
```

**实现**：

```typescript
interface TCCParticipant {
  try(): Promise<string>; // 返回事务 ID
  confirm(txId: string): Promise<void>;
  cancel(txId: string): Promise<void>;
}

class TCCTransaction {
  private txIds: Map<TCCParticipant, string> = new Map();

  constructor(private participants: TCCParticipant[]) {}

  async execute(): Promise<boolean> {
    // 阶段 1：Try
    try {
      for (const participant of this.participants) {
        const txId = await participant.try();
        this.txIds.set(participant, txId);
      }
    } catch (error) {
      // Try 失败，Cancel
      await this.cancelAll();
      return false;
    }

    // 阶段 2：Confirm
    try {
      for (const [participant, txId] of this.txIds.entries()) {
        await participant.confirm(txId);
      }
      return true;
    } catch (error) {
      // Confirm 失败，Cancel（可能部分成功，需要补偿）
      await this.cancelAll();
      return false;
    }
  }

  private async cancelAll() {
    for (const [participant, txId] of this.txIds.entries()) {
      try {
        await participant.cancel(txId);
      } catch (error) {
        console.error('Cancel failed:', error);
        // 记录到补偿表
      }
    }
  }
}

// 库存服务 TCC 实现
class InventoryTCCParticipant implements TCCParticipant {
  constructor(
    private productId: number,
    private quantity: number
  ) {}

  async try(): Promise<string> {
    const txId = generateId();
    
    // 冻结库存（不真正扣减）
    await db.inventory.update({
      where: { productId: this.productId },
      data: {
        available: { decrement: this.quantity },
        frozen: { increment: this.quantity }
      }
    });
    
    // 记录事务
    await db.inventoryTx.create({
      data: {
        id: txId,
        productId: this.productId,
        quantity: this.quantity,
        status: 'trying',
        createdAt: new Date()
      }
    });
    
    return txId;
  }

  async confirm(txId: string): Promise<void> {
    // 确认扣减（从冻结转为已用）
    await db.inventory.update({
      where: { productId: this.productId },
      data: {
        frozen: { decrement: this.quantity }
      }
    });
    
    await db.inventoryTx.update({
      where: { id: txId },
      data: { status: 'confirmed' }
    });
  }

  async cancel(txId: string): Promise<void> {
    // 取消：释放冻结库存
    await db.inventory.update({
      where: { productId: this.productId },
      data: {
        available: { increment: this.quantity },
        frozen: { decrement: this.quantity }
      }
    });
    
    await db.inventoryTx.update({
      where: { id: txId },
      data: { status: 'cancelled' }
    });
  }
}

// 支付服务 TCC 实现
class PaymentTCCParticipant implements TCCParticipant {
  constructor(
    private userId: number,
    private amount: number
  ) {}

  async try(): Promise<string> {
    const txId = generateId();
    
    // 冻结金额
    await db.account.update({
      where: { userId: this.userId },
      data: {
        balance: { decrement: this.amount },
        frozen: { increment: this.amount }
      }
    });
    
    return txId;
  }

  async confirm(txId: string): Promise<void> {
    // 确认扣款
    await db.account.update({
      where: { userId: this.userId },
      data: {
        frozen: { decrement: this.amount }
      }
    });
  }

  async cancel(txId: string): Promise<void> {
    // 取消：解冻金额
    await db.account.update({
      where: { userId: this.userId },
      data: {
        balance: { increment: this.amount },
        frozen: { decrement: this.amount }
      }
    });
  }
}

// 使用
const tcc = new TCCTransaction([
  new InventoryTCCParticipant(productId, quantity),
  new PaymentTCCParticipant(userId, amount)
]);

const success = await tcc.execute();
```

**优点**：
- ✅ 性能较好（无锁）
- ✅ 最终一致性

**缺点**：
- ❌ 实现复杂（需要三个接口）
- ❌ 业务侵入性强

### 3. Saga 模式

**思想**：长事务拆分成多个短事务，每个事务有对应的补偿操作。

**实现**：

```typescript
interface SagaStep {
  execute(): Promise<any>;
  compensate(): Promise<void>;
}

class SagaOrchestrator {
  private executedSteps: SagaStep[] = [];

  constructor(private steps: SagaStep[]) {}

  async execute(): Promise<boolean> {
    try {
      // 正向执行所有步骤
      for (const step of this.steps) {
        await step.execute();
        this.executedSteps.push(step);
      }
      return true;
    } catch (error) {
      console.error('Saga failed, compensating...', error);
      
      // 失败，反向补偿
      await this.compensate();
      return false;
    }
  }

  private async compensate() {
    // 反向执行补偿操作
    for (let i = this.executedSteps.length - 1; i >= 0; i--) {
      const step = this.executedSteps[i];
      try {
        await step.compensate();
      } catch (error) {
        console.error('Compensation failed:', error);
        // 记录到补偿日志，人工介入
      }
    }
  }
}

// 创建订单步骤
class CreateOrderStep implements SagaStep {
  private orderId?: number;

  async execute() {
    const order = await orderService.create(this.orderData);
    this.orderId = order.id;
    return order;
  }

  async compensate() {
    if (this.orderId) {
      await orderService.cancel(this.orderId);
    }
  }
}

// 扣减库存步骤
class DeductInventoryStep implements SagaStep {
  async execute() {
    await inventoryService.deduct(this.productId, this.quantity);
  }

  async compensate() {
    // 补偿：增加库存
    await inventoryService.add(this.productId, this.quantity);
  }
}

// 支付步骤
class PaymentStep implements SagaStep {
  private paymentId?: string;

  async execute() {
    const payment = await paymentService.charge(this.userId, this.amount);
    this.paymentId = payment.id;
    return payment;
  }

  async compensate() {
    if (this.paymentId) {
      // 补偿：退款
      await paymentService.refund(this.paymentId);
    }
  }
}

// 使用
async function createOrderWithSaga(orderData: any) {
  const saga = new SagaOrchestrator([
    new CreateOrderStep(orderData),
    new DeductInventoryStep(orderData.productId, orderData.quantity),
    new PaymentStep(orderData.userId, orderData.amount)
  ]);

  const success = await saga.execute();
  
  if (!success) {
    throw new Error('Order creation failed');
  }
}
```

**使用消息队列实现 Saga**：

```typescript
import { EventEmitter } from 'events';

class SagaEventManager extends EventEmitter {
  async startSaga(sagaId: string, data: any) {
    this.emit('saga.started', { sagaId, data });
  }

  async compensate(sagaId: string, fromStep: number) {
    this.emit('saga.compensate', { sagaId, fromStep });
  }
}

const sagaManager = new SagaEventManager();

// 订单服务监听
sagaManager.on('saga.started', async ({ sagaId, data }) => {
  try {
    const order = await createOrder(data);
    sagaManager.emit('order.created', { sagaId, orderId: order.id, data });
  } catch (error) {
    sagaManager.emit('saga.failed', { sagaId, step: 1, error });
  }
});

// 库存服务监听
sagaManager.on('order.created', async ({ sagaId, orderId, data }) => {
  try {
    await deductInventory(data.productId, data.quantity);
    sagaManager.emit('inventory.deducted', { sagaId, orderId, data });
  } catch (error) {
    // 失败，触发补偿
    sagaManager.emit('saga.compensate', { sagaId, fromStep: 1 });
  }
});

// 补偿监听
sagaManager.on('saga.compensate', async ({ sagaId, fromStep }) => {
  if (fromStep >= 1) {
    // 取消订单
    await cancelOrder(sagaId);
  }
});
```

**优点**：
- ✅ 适合长事务
- ✅ 无锁，性能好
- ✅ 灵活（可异步）

**缺点**：
- ❌ 需要设计补偿逻辑
- ❌ 可能出现不一致（补偿失败）

### 4. 本地消息表

**思想**：在本地数据库记录消息，定时发送到消息队列。

```typescript
// 本地事务 + 消息表
async function createOrderWithMessage(orderData: any) {
  await prisma.$transaction(async (tx) => {
    // 1. 创建订单
    const order = await tx.order.create({ data: orderData });
    
    // 2. 记录消息到本地表
    await tx.outboxMessage.create({
      data: {
        topic: 'order.created',
        payload: JSON.stringify({
          orderId: order.id,
          userId: order.userId,
          amount: order.amount
        }),
        status: 'pending',
        createdAt: new Date()
      }
    });
    
    // 本地事务保证一致性
  });
}

// 定时任务：发送消息
setInterval(async () => {
  const messages = await prisma.outboxMessage.findMany({
    where: { status: 'pending' },
    take: 100
  });

  for (const message of messages) {
    try {
      // 发送到消息队列
      await messageQueue.publish(message.topic, JSON.parse(message.payload));
      
      // 标记为已发送
      await prisma.outboxMessage.update({
        where: { id: message.id },
        data: { status: 'sent', sentAt: new Date() }
      });
    } catch (error) {
      console.error('Failed to send message:', error);
      // 重试次数 + 1
      await prisma.outboxMessage.update({
        where: { id: message.id },
        data: { retries: { increment: 1 } }
      });
    }
  }
}, 5000); // 每 5 秒
```

**优点**：
- ✅ 可靠性高
- ✅ 最终一致性

**缺点**：
- ❌ 延迟（定时任务）
- ❌ 需要清理已发送消息

---

## 分布式锁

### 为什么需要分布式锁？

```typescript
// 场景：秒杀
// 多个实例同时扣减库存，导致超卖
async function buyProduct(productId: number) {
  // ❌ 有问题的代码
  const product = await db.product.findUnique({ where: { id: productId } });
  
  if (product.stock > 0) {
    // 多个请求同时走到这里，都认为有库存
    await db.product.update({
      where: { id: productId },
      data: { stock: { decrement: 1 } }
    });
  }
}
```

### 1. Redis 分布式锁

**基本实现**：

```typescript
import Redis from 'ioredis';

class RedisLock {
  constructor(
    private redis: Redis,
    private key: string,
    private ttl = 10000 // 10 秒
  ) {}

  async acquire(): Promise<string | null> {
    const value = `${Date.now()}-${Math.random()}`;
    
    // SET key value NX PX ttl
    const result = await this.redis.set(
      this.key,
      value,
      'PX', this.ttl,
      'NX'
    );
    
    return result === 'OK' ? value : null;
  }

  async release(value: string): Promise<boolean> {
    // Lua 脚本：原子性检查并删除
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, this.key, value);
    return result === 1;
  }

  async extend(value: string, ttl: number): Promise<boolean> {
    // 延长锁的过期时间
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("pexpire", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, this.key, value, ttl);
    return result === 1;
  }
}

// 使用
async function buyProductWithLock(productId: number) {
  const lock = new RedisLock(redis, `lock:product:${productId}`, 5000);
  const lockValue = await lock.acquire();
  
  if (!lockValue) {
    throw new Error('Failed to acquire lock');
  }
  
  try {
    // 执行业务逻辑
    const product = await db.product.findUnique({ where: { id: productId } });
    
    if (product.stock <= 0) {
      throw new Error('Out of stock');
    }
    
    await db.product.update({
      where: { id: productId },
      data: { stock: { decrement: 1 } }
    });
    
    return { success: true };
  } finally {
    // 释放锁
    await lock.release(lockValue);
  }
}
```

**自动续期（Watchdog）**：

```typescript
class RedisLockWithWatchdog extends RedisLock {
  private watchdogTimer?: NodeJS.Timeout;
  private lockValue?: string;

  async acquire(): Promise<boolean> {
    this.lockValue = await super.acquire();
    
    if (this.lockValue) {
      this.startWatchdog();
      return true;
    }
    
    return false;
  }

  async release(): Promise<boolean> {
    this.stopWatchdog();
    
    if (this.lockValue) {
      return await super.release(this.lockValue);
    }
    
    return false;
  }

  private startWatchdog() {
    // 每 ttl/3 时间续期
    const renewInterval = this.ttl / 3;
    
    this.watchdogTimer = setInterval(async () => {
      if (this.lockValue) {
        const renewed = await this.extend(this.lockValue, this.ttl);
        if (!renewed) {
          console.error('Failed to renew lock');
          this.stopWatchdog();
        }
      }
    }, renewInterval);
  }

  private stopWatchdog() {
    if (this.watchdogTimer) {
      clearInterval(this.watchdogTimer);
      this.watchdogTimer = undefined;
    }
  }
}
```

### 2. Redlock 算法（多节点）

防止单点故障。

```typescript
class Redlock {
  constructor(
    private redisClients: Redis[],
    private ttl = 10000
  ) {}

  async acquire(resource: string): Promise<string | null> {
    const value = `${Date.now()}-${Math.random()}`;
    let successCount = 0;
    
    // 向所有节点获取锁
    const promises = this.redisClients.map(client =>
      client.set(resource, value, 'PX', this.ttl, 'NX')
    );
    
    const results = await Promise.allSettled(promises);
    
    for (const result of results) {
      if (result.status === 'fulfilled' && result.value === 'OK') {
        successCount++;
      }
    }
    
    // 超过半数成功，视为获取锁成功
    if (successCount >= Math.floor(this.redisClients.length / 2) + 1) {
      return value;
    }
    
    // 失败，释放所有锁
    await this.release(resource, value);
    return null;
  }

  async release(resource: string, value: string): Promise<void> {
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    
    await Promise.all(
      this.redisClients.map(client =>
        client.eval(script, 1, resource, value)
      )
    );
  }
}

// 使用
const redlock = new Redlock([
  new Redis({ host: 'redis1' }),
  new Redis({ host: 'redis2' }),
  new Redis({ host: 'redis3' })
]);

const lockValue = await redlock.acquire('lock:resource');
if (lockValue) {
  try {
    // 执行业务逻辑
  } finally {
    await redlock.release('lock:resource', lockValue);
  }
}
```

### 3. 数据库分布式锁

```typescript
// 使用数据库实现分布式锁
async function acquireDatabaseLock(
  resource: string,
  ttl: number
): Promise<boolean> {
  try {
    await prisma.distributedLock.create({
      data: {
        resource,
        owner: process.pid.toString(),
        expiresAt: new Date(Date.now() + ttl)
      }
    });
    return true;
  } catch (error) {
    // 唯一约束冲突，锁已被持有
    return false;
  }
}

async function releaseDatabaseLock(resource: string): Promise<void> {
  await prisma.distributedLock.deleteMany({
    where: {
      resource,
      owner: process.pid.toString()
    }
  });
}

// 清理过期锁
setInterval(async () => {
  await prisma.distributedLock.deleteMany({
    where: {
      expiresAt: {
        lt: new Date()
      }
    }
  });
}, 60000); // 每分钟
```

---

## 分布式 ID

### 1. UUID

```typescript
import { randomUUID } from 'crypto';

const id = randomUUID(); // e.g., '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d'
```

**优点**：
- ✅ 简单
- ✅ 无依赖
- ✅ 全局唯一

**缺点**：
- ❌ 无序（索引性能差）
- ❌ 太长（36 字符）
- ❌ 不可读

### 2. Snowflake 算法

**结构**（64 位）：

```
0 - 41位时间戳 - 10位机器ID - 12位序列号
|   (毫秒级)      (1024台)     (4096/ms)
```

**实现**：

```typescript
class SnowflakeIdGenerator {
  private sequence = 0;
  private lastTimestamp = -1;

  constructor(
    private workerId: number,        // 0-1023
    private epoch = 1640995200000    // 2022-01-01 00:00:00
  ) {
    if (workerId < 0 || workerId > 1023) {
      throw new Error('Worker ID must be between 0 and 1023');
    }
  }

  generate(): bigint {
    let timestamp = Date.now();

    // 时钟回拨
    if (timestamp < this.lastTimestamp) {
      throw new Error('Clock moved backwards');
    }

    if (timestamp === this.lastTimestamp) {
      // 同一毫秒内，序列号 + 1
      this.sequence = (this.sequence + 1) & 0xfff; // 12 位
      
      if (this.sequence === 0) {
        // 序列号溢出，等待下一毫秒
        timestamp = this.waitNextMillis(this.lastTimestamp);
      }
    } else {
      // 新的毫秒，序列号重置
      this.sequence = 0;
    }

    this.lastTimestamp = timestamp;

    // 组装 ID
    const timestampPart = BigInt(timestamp - this.epoch) << 22n;
    const workerIdPart = BigInt(this.workerId) << 12n;
    const sequencePart = BigInt(this.sequence);

    return timestampPart | workerIdPart | sequencePart;
  }

  private waitNextMillis(lastTimestamp: number): number {
    let timestamp = Date.now();
    while (timestamp <= lastTimestamp) {
      timestamp = Date.now();
    }
    return timestamp;
  }

  // 解析 ID
  parse(id: bigint): {
    timestamp: number;
    workerId: number;
    sequence: number;
  } {
    const sequence = Number(id & 0xfffn);
    const workerId = Number((id >> 12n) & 0x3ffn);
    const timestamp = Number(id >> 22n) + this.epoch;

    return { timestamp, workerId, sequence };
  }
}

// 使用
const idGen = new SnowflakeIdGenerator(1); // Worker ID = 1

const id = idGen.generate();
console.log(id); // 7234567890123456789n

const parsed = idGen.parse(id);
console.log(parsed);
// {
//   timestamp: 1704067200000,
//   workerId: 1,
//   sequence: 0
// }
```

**优点**：
- ✅ 高性能（本地生成）
- ✅ 趋势递增（索引友好）
- ✅ 包含时间信息

**缺点**：
- ❌ 依赖机器时钟
- ❌ 需要配置 Worker ID

### 3. MongoDB ObjectId

```typescript
import { ObjectId } from 'mongodb';

const id = new ObjectId();
console.log(id.toString()); // '507f1f77bcf86cd799439011'

// 结构：4字节时间戳 + 5字节随机值 + 3字节计数器
```

### 4. 数据库自增 ID

```typescript
// PostgreSQL
await prisma.$queryRaw`SELECT nextval('order_id_seq')`;

// MySQL
// AUTO_INCREMENT
```

**优点**：
- ✅ 简单
- ✅ 有序
- ✅ 数字较小

**缺点**：
- ❌ 依赖数据库
- ❌ 性能瓶颈（单点）
- ❌ 暴露业务量（可被推算）

### 对比

| 方案 | 性能 | 有序性 | 长度 | 依赖 | 推荐场景 |
|------|------|--------|------|------|---------|
| UUID | ⭐⭐⭐⭐ | ❌ | 36 字符 | 无 | 非主键、分布式文件 |
| Snowflake | ⭐⭐⭐⭐⭐ | ✅ | 19 位数字 | 时钟 | **分布式系统（推荐）** |
| ObjectId | ⭐⭐⭐⭐ | ✅ | 24 字符 | 无 | MongoDB |
| 数据库自增 | ⭐⭐ | ✅ | 较小 | 数据库 | 单机、非分布式 |

---

## 一致性算法

### Raft 算法（简化理解）

**角色**：
- Leader（领导者）：处理所有客户端请求
- Follower（跟随者）：被动接收 Leader 的日志
- Candidate（候选人）：选举时的临时角色

**选举过程**：

```typescript
enum NodeState {
  FOLLOWER = 'FOLLOWER',
  CANDIDATE = 'CANDIDATE',
  LEADER = 'LEADER'
}

class RaftNode {
  private state = NodeState.FOLLOWER;
  private currentTerm = 0;
  private votedFor: string | null = null;
  private votes = 0;
  private electionTimeout: NodeJS.Timeout;

  constructor(
    private id: string,
    private peers: string[]
  ) {
    this.resetElectionTimeout();
  }

  private resetElectionTimeout() {
    clearTimeout(this.electionTimeout);
    
    // 随机超时（150-300ms）
    const timeout = 150 + Math.random() * 150;
    
    this.electionTimeout = setTimeout(() => {
      this.startElection();
    }, timeout);
  }

  private startElection() {
    // 成为候选人
    this.state = NodeState.CANDIDATE;
    this.currentTerm++;
    this.votedFor = this.id;
    this.votes = 1; // 投票给自己

    console.log(`Node ${this.id} starting election for term ${this.currentTerm}`);

    // 向所有节点请求投票
    for (const peer of this.peers) {
      this.requestVote(peer);
    }
  }

  private async requestVote(peer: string) {
    // 发送投票请求
    const granted = await this.sendVoteRequest(peer, this.currentTerm);
    
    if (granted) {
      this.votes++;
      
      // 获得多数票
      const majority = Math.floor((this.peers.length + 1) / 2) + 1;
      if (this.votes >= majority && this.state === NodeState.CANDIDATE) {
        this.becomeLeader();
      }
    }
  }

  private becomeLeader() {
    this.state = NodeState.LEADER;
    console.log(`Node ${this.id} became leader for term ${this.currentTerm}`);
    
    // 定期发送心跳
    this.sendHeartbeats();
  }

  private sendHeartbeats() {
    if (this.state !== NodeState.LEADER) return;
    
    for (const peer of this.peers) {
      this.sendHeartbeat(peer);
    }
    
    // 每 50ms 发送一次心跳
    setTimeout(() => this.sendHeartbeats(), 50);
  }

  private receiveHeartbeat(term: number, leaderId: string) {
    if (term >= this.currentTerm) {
      this.currentTerm = term;
      this.state = NodeState.FOLLOWER;
      this.resetElectionTimeout(); // 重置选举超时
    }
  }
}
```

---

## Node.js 实战

### 完整的分布式任务调度器

```typescript
import Redis from 'ioredis';
import { PrismaClient } from '@prisma/client';

interface Task {
  id: string;
  type: string;
  payload: any;
  createdAt: Date;
  retries: number;
}

class DistributedTaskScheduler {
  private redis: Redis;
  private prisma: PrismaClient;
  private isRunning = false;
  private workerId: string;

  constructor() {
    this.redis = new Redis();
    this.prisma = new PrismaClient();
    this.workerId = `worker-${process.pid}`;
  }

  async start() {
    this.isRunning = true;
    console.log(`Task scheduler started: ${this.workerId}`);
    
    // 启动多个 worker
    for (let i = 0; i < 3; i++) {
      this.runWorker();
    }
  }

  async stop() {
    this.isRunning = false;
    console.log('Task scheduler stopping...');
  }

  private async runWorker() {
    while (this.isRunning) {
      try {
        // 尝试获取任务
        const task = await this.acquireTask();
        
        if (task) {
          await this.processTask(task);
        } else {
          // 没有任务，等待一会
          await new Promise(resolve => setTimeout(resolve, 1000));
        }
      } catch (error) {
        console.error('Worker error:', error);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }

  private async acquireTask(): Promise<Task | null> {
    // 使用 Lua 脚本原子性获取任务
    const script = `
      local taskId = redis.call('rpop', 'tasks:pending')
      if taskId then
        redis.call('hset', 'tasks:processing', taskId, ARGV[1])
        return taskId
      else
        return nil
      end
    `;

    const taskId = await this.redis.eval(
      script,
      0,
      this.workerId
    );

    if (!taskId) return null;

    // 从数据库加载任务详情
    const task = await this.prisma.task.findUnique({
      where: { id: taskId as string }
    });

    return task;
  }

  private async processTask(task: Task) {
    console.log(`Processing task ${task.id} by ${this.workerId}`);

    try {
      // 执行任务（根据类型）
      switch (task.type) {
        case 'send_email':
          await this.sendEmail(task.payload);
          break;
        case 'generate_report':
          await this.generateReport(task.payload);
          break;
        default:
          throw new Error(`Unknown task type: ${task.type}`);
      }

      // 标记为完成
      await this.completeTask(task.id);
    } catch (error) {
      console.error(`Task ${task.id} failed:`, error);
      
      // 重试
      await this.retryTask(task);
    }
  }

  private async completeTask(taskId: string) {
    await this.redis.hdel('tasks:processing', taskId);
    await this.prisma.task.update({
      where: { id: taskId },
      data: {
        status: 'completed',
        completedAt: new Date()
      }
    });
  }

  private async retryTask(task: Task) {
    const maxRetries = 3;

    if (task.retries < maxRetries) {
      // 重新放回队列
      await this.redis.lpush('tasks:pending', task.id);
      await this.prisma.task.update({
        where: { id: task.id },
        data: { retries: { increment: 1 } }
      });
    } else {
      // 超过最大重试次数，放入死信队列
      await this.redis.hdel('tasks:processing', task.id);
      await this.redis.lpush('tasks:dead_letter', task.id);
      await this.prisma.task.update({
        where: { id: task.id },
        data: { status: 'failed' }
      });
    }
  }

  private async sendEmail(payload: any) {
    // 发送邮件
    await new Promise(resolve => setTimeout(resolve, 1000));
  }

  private async generateReport(payload: any) {
    // 生成报表
    await new Promise(resolve => setTimeout(resolve, 2000));
  }

  // 添加任务
  async addTask(type: string, payload: any): Promise<string> {
    const task = await this.prisma.task.create({
      data: {
        type,
        payload,
        status: 'pending',
        retries: 0,
        createdAt: new Date()
      }
    });

    // 加入队列
    await this.redis.lpush('tasks:pending', task.id);

    return task.id;
  }
}

// 使用
const scheduler = new DistributedTaskScheduler();
scheduler.start();

// 添加任务
await scheduler.addTask('send_email', {
  to: 'user@example.com',
  subject: 'Hello',
  body: 'World'
});
```

---

## 常见面试题

### 1. CAP 定理的理解？

**回答要点**：

1. **定义**：
   - C（一致性）：所有节点数据一致
   - A（可用性）：系统总是能响应
   - P（分区容错性）：网络分区时仍能工作

2. **只能三选二**：
   - 网络分区是客观存在的，所以实际是 **CP vs AP**

3. **CP 系统**（牺牲可用性）：
   - 代表：ZooKeeper、HBase、MongoDB
   - 场景：金融、账务

4. **AP 系统**（牺牲一致性）：
   - 代表：Cassandra、DynamoDB、Eureka
   - 场景：社交、内容

5. **实际应用**：
   - 大部分系统不是纯 CP 或 AP
   - 通过配置调整（如 Quorum 机制）

### 2. 分布式事务的方案对比？

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|------|-------|------|--------|---------|
| **2PC** | 强 | 低 | 中 | 强一致性要求 |
| **TCC** | 最终 | 中 | 高 | 金融、交易 |
| **Saga** | 最终 | 高 | 中 | **微服务（推荐）** |
| **本地消息表** | 最终 | 高 | 低 | 异步场景 |
| **MQ 事务消息** | 最终 | 高 | 低 | 可靠消息 |

**选择建议**：
- 强一致性：2PC
- 微服务：Saga
- 异步场景：本地消息表或 MQ

### 3. 如何实现分布式锁？

**Redis 实现（推荐）**：

```typescript
// 核心要点：
1. SET key value NX PX timeout  // 原子性设置
2. Lua 脚本删除（检查 value）    // 防误删
3. 自动续期（Watchdog）         // 防过期
4. Redlock（多节点）            // 防单点故障
```

**注意事项**：
- 设置合理的过期时间
- 释放时检查 value（防止误删）
- 考虑时钟漂移
- 考虑网络延迟

### 4. Snowflake ID 的优缺点？

**优点**：
- ✅ 高性能（本地生成，无网络开销）
- ✅ 趋势递增（对数据库索引友好）
- ✅ 包含时间信息
- ✅ 可配置（Worker ID、数据中心 ID）

**缺点**：
- ❌ 依赖机器时钟（时钟回拨问题）
- ❌ 需要维护 Worker ID

**时钟回拨解决方案**：
1. 拒绝生成（抛异常）
2. 等待时钟追上
3. 使用备用位（扩展 Worker ID）

### 5. 如何保证分布式系统的数据一致性？

**强一致性**：
- 2PC/3PC
- Raft/Paxos
- 同步复制

**最终一致性**（推荐）：
- 异步复制
- 消息队列
- Saga 模式
- CQRS + Event Sourcing

**实践建议**：
1. 根据业务选择一致性级别
2. 核心业务：强一致性
3. 非核心业务：最终一致性
4. 使用补偿机制

---

## 总结

### 分布式系统设计要点

1. **理论基础**
   - CAP 定理：三选二
   - BASE 理论：最终一致性
   - 一致性算法：Raft、Paxos

2. **分布式事务**
   - 2PC：强一致，性能差
   - TCC：最终一致，复杂
   - Saga：推荐，适合微服务
   - 本地消息表：简单可靠

3. **分布式锁**
   - Redis：性能好，推荐
   - Redlock：防单点
   - 数据库：简单但性能差
   - ZooKeeper：强一致

4. **分布式 ID**
   - UUID：简单但无序
   - Snowflake：推荐
   - 数据库自增：单机
   - ObjectId：MongoDB

### 实践检查清单

- [ ] 是否理解 CAP 和 BASE？
- [ ] 是否选择了合适的一致性级别？
- [ ] 分布式事务方案是否合理？
- [ ] 是否正确实现了分布式锁？
- [ ] 分布式 ID 是否合适？
- [ ] 是否有超时和重试机制？
- [ ] 是否有幂等性设计？
- [ ] 是否有补偿机制？
- [ ] 是否有监控和告警？
- [ ] 是否考虑了时钟漂移？
- [ ] 是否考虑了网络分区？
- [ ] 是否有降级方案？

---

**下一篇**：[数据库设计](./04-database-design.md)

