# Kafka

Apache Kafka 是一个分布式流处理平台，以高吞吐量、低延迟和强大的可扩展性著称。

## 核心架构

### 关键概念

```
Producer → Kafka Cluster → Consumer
              ↓
         Topic (Partition 0, 1, 2, ...)
              ↓
         Consumer Group
```

**Topic & Partition**：
- Topic：逻辑概念，消息的分类
- Partition：物理概念，Topic 的分片
- 每个 Partition 是一个有序的、不可变的消息序列

**Broker**：
- Kafka 服务器节点
- 存储 Partition 数据
- 处理读写请求

**Replication**：
- 每个 Partition 有多个副本
- Leader 处理读写
- Follower 同步数据
- ISR（In-Sync Replicas）：同步副本集

## Producer 高级特性

### 分区策略

```typescript
import { Kafka, Partitioners } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
});

const producer = kafka.producer({
  // 默认分区器：轮询
  createPartitioner: Partitioners.DefaultPartitioner
});

await producer.connect();

// 1. 指定分区
await producer.send({
  topic: 'my-topic',
  messages: [
    {
      partition: 0,
      key: 'user-123',
      value: JSON.stringify({ event: 'click' })
    }
  ]
});

// 2. 按 key 分区（相同 key 的消息总是发到同一分区）
await producer.send({
  topic: 'user-events',
  messages: [
    {
      key: 'user-123',  // 会被 hash 到固定分区
      value: JSON.stringify({ event: 'login' })
    }
  ]
});

// 3. 自定义分区器
const customPartitioner = () => {
  return ({ topic, partitionMetadata, message }) => {
    const numPartitions = partitionMetadata.length;
    
    // 根据消息内容分区
    const data = JSON.parse(message.value.toString());
    
    if (data.priority === 'high') {
      return 0;  // 高优先级消息发到分区 0
    }
    
    // 其他消息按 key hash
    const key = message.key?.toString() || '';
    return Math.abs(hashCode(key)) % numPartitions;
  };
};

const customProducer = kafka.producer({
  createPartitioner: customPartitioner
});
```

### 消息可靠性保证

```typescript
// ACK 策略
const producer = kafka.producer({
  // acks:
  // 0 - 不等待确认（最快，可能丢失）
  // 1 - 等待 Leader 确认（默认）
  // -1/all - 等待所有 ISR 确认（最安全）
  acks: -1,
  
  // 重试配置
  retry: {
    initialRetryTime: 100,
    retries: 8,
    multiplier: 2,
    maxRetryTime: 30000
  },
  
  // 压缩
  compression: 'gzip',  // gzip, snappy, lz4, zstd
  
  // 批处理
  batch: {
    size: 16384,        // 16KB
    maxBytes: 1048576   // 1MB
  },
  
  // 等待时间（积累消息后再发送）
  linger: 10  // 10ms
});

// 幂等性生产者（防止重复）
const idempotentProducer = kafka.producer({
  idempotent: true,  // 启用幂等性
  maxInFlightRequests: 5,
  acks: -1
});

// 事务性生产者
const transactionalProducer = kafka.producer({
  transactionalId: 'my-transactional-producer',
  maxInFlightRequests: 1,
  idempotent: true
});

await transactionalProducer.connect();

const transaction = await transactionalProducer.transaction();

try {
  await transaction.send({
    topic: 'orders',
    messages: [{ value: 'order-1' }]
  });
  
  await transaction.send({
    topic: 'payments',
    messages: [{ value: 'payment-1' }]
  });
  
  await transaction.commit();
} catch (error) {
  await transaction.abort();
  throw error;
}
```

### 批量发送优化

```typescript
class BatchProducer {
  private producer: Producer;
  private batch: Array<{ topic: string; messages: any[] }> = [];
  private batchSize = 100;
  private flushInterval = 1000;  // 1 秒
  private timer?: NodeJS.Timeout;
  
  constructor(kafka: Kafka) {
    this.producer = kafka.producer({
      batch: {
        size: 65536,  // 64KB
        maxBytes: 1048576  // 1MB
      }
    });
    
    this.startFlushTimer();
  }
  
  async send(topic: string, message: any) {
    this.batch.push({
      topic,
      messages: [{ value: JSON.stringify(message) }]
    });
    
    if (this.batch.length >= this.batchSize) {
      await this.flush();
    }
  }
  
  private async flush() {
    if (this.batch.length === 0) return;
    
    const messages = this.batch.splice(0, this.batch.length);
    
    // 按 topic 分组
    const grouped = messages.reduce((acc, item) => {
      if (!acc[item.topic]) {
        acc[item.topic] = [];
      }
      acc[item.topic].push(...item.messages);
      return acc;
    }, {} as Record<string, any[]>);
    
    // 批量发送
    await Promise.all(
      Object.entries(grouped).map(([topic, msgs]) =>
        this.producer.send({
          topic,
          messages: msgs
        })
      )
    );
  }
  
  private startFlushTimer() {
    this.timer = setInterval(() => {
      this.flush().catch(console.error);
    }, this.flushInterval);
  }
  
  async close() {
    if (this.timer) {
      clearInterval(this.timer);
    }
    await this.flush();
    await this.producer.disconnect();
  }
}
```

## Consumer 高级特性

### 消费者组和并行消费

```typescript
const consumer = kafka.consumer({
  groupId: 'my-consumer-group',
  
  // 分区分配策略
  // - RoundRobinAssignor: 轮询分配
  // - RangeAssignor: 范围分配（默认）
  // - StickyAssignor: 尽量保持原有分配
  sessionTimeout: 30000,
  heartbeatInterval: 3000,
  
  // 再平衡监听
  rebalanceTimeout: 60000,
  
  // 最大拉取字节数
  maxBytesPerPartition: 1048576,  // 1MB
  
  // 最小/最大拉取字节
  minBytes: 1,
  maxBytes: 10485760,  // 10MB
  
  // 最大等待时间
  maxWaitTimeInMs: 5000
});

await consumer.connect();

// 订阅 topic
await consumer.subscribe({
  topics: ['user-events', 'order-events'],
  fromBeginning: false
});

// 处理消息
await consumer.run({
  // 每个分区并行处理
  partitionsConsumedConcurrently: 3,
  
  // 自动提交偏移量
  autoCommit: true,
  autoCommitInterval: 5000,
  
  eachMessage: async ({ topic, partition, message }) => {
    console.log({
      topic,
      partition,
      offset: message.offset,
      key: message.key?.toString(),
      value: message.value?.toString(),
      headers: message.headers
    });
    
    // 处理消息
    await processMessage(message);
  }
});
```

### 偏移量管理

```typescript
// 手动提交偏移量
await consumer.run({
  autoCommit: false,  // 禁用自动提交
  
  eachMessage: async ({ topic, partition, message }) => {
    try {
      await processMessage(message);
      
      // 手动提交
      await consumer.commitOffsets([
        {
          topic,
          partition,
          offset: (parseInt(message.offset) + 1).toString()
        }
      ]);
    } catch (error) {
      console.error('Failed to process message:', error);
      // 不提交偏移量，下次会重新消费
    }
  }
});

// 批量提交
await consumer.run({
  autoCommit: false,
  
  eachBatch: async ({
    batch,
    resolveOffset,
    heartbeat,
    isRunning,
    isStale
  }) => {
    for (const message of batch.messages) {
      if (!isRunning() || isStale()) break;
      
      await processMessage(message);
      
      // 标记已处理
      resolveOffset(message.offset);
      
      // 发送心跳
      await heartbeat();
    }
    
    // 批量提交（自动）
  }
});

// 从指定偏移量开始消费
await consumer.seek({
  topic: 'my-topic',
  partition: 0,
  offset: '100'
});

// 从最早开始
await consumer.seek({
  topic: 'my-topic',
  partition: 0,
  offset: '-2'  // 特殊值：最早
});

// 从最新开始
await consumer.seek({
  topic: 'my-topic',
  partition: 0,
  offset: '-1'  // 特殊值：最新
});
```

### 消息处理模式

```typescript
// 1. At Most Once（最多一次，可能丢失）
await consumer.run({
  autoCommit: true,  // 先提交
  eachMessage: async ({ message }) => {
    // 后处理
    await processMessage(message);
  }
});

// 2. At Least Once（至少一次，可能重复）✅ 推荐
await consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message }) => {
    // 先处理
    await processMessage(message);
    
    // 后提交
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (parseInt(message.offset) + 1).toString()
    }]);
  }
});

// 3. Exactly Once（精确一次）
// 需要使用事务 + 幂等性处理
class ExactlyOnceProcessor {
  private processedOffsets = new Set<string>();
  
  async process(message: any) {
    const offsetKey = `${message.topic}-${message.partition}-${message.offset}`;
    
    // 检查是否已处理（幂等性）
    if (this.processedOffsets.has(offsetKey)) {
      return;
    }
    
    try {
      await this.processWithTransaction(message);
      this.processedOffsets.add(offsetKey);
    } catch (error) {
      console.error('Processing failed:', error);
      throw error;
    }
  }
  
  private async processWithTransaction(message: any) {
    const session = await db.startSession();
    
    try {
      await session.withTransaction(async () => {
        // 业务处理
        await db.collection('orders').insertOne(
          message.value,
          { session }
        );
        
        // 保存偏移量
        await db.collection('kafka_offsets').updateOne(
          {
            topic: message.topic,
            partition: message.partition
          },
          {
            $set: { offset: message.offset }
          },
          { upsert: true, session }
        );
      });
    } finally {
      await session.endSession();
    }
  }
}
```

### 错误处理和重试

```typescript
class RobustConsumer {
  private consumer: Consumer;
  private dlqProducer: Producer;
  
  async run() {
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        try {
          await this.processWithRetry(message, 3);
        } catch (error) {
          // 发送到死信队列
          await this.sendToDLQ(topic, message, error);
        }
      }
    });
  }
  
  private async processWithRetry(
    message: any,
    maxRetries: number
  ): Promise<void> {
    let lastError: Error | undefined;
    
    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        await this.processMessage(message);
        return;
      } catch (error) {
        lastError = error as Error;
        
        // 指数退避
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    throw lastError;
  }
  
  private async sendToDLQ(
    originalTopic: string,
    message: any,
    error: Error
  ) {
    await this.dlqProducer.send({
      topic: `${originalTopic}-dlq`,
      messages: [
        {
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            'original-topic': originalTopic,
            'error-message': error.message,
            'failed-at': new Date().toISOString()
          }
        }
      ]
    });
  }
  
  private async processMessage(message: any) {
    // 实际业务逻辑
  }
}
```

## 性能优化

### Producer 优化

```typescript
const optimizedProducer = kafka.producer({
  // 1. 启用压缩
  compression: 'snappy',  // 或 lz4, 速度快
  
  // 2. 批处理配置
  batch: {
    size: 65536,      // 64KB
    maxBytes: 1048576 // 1MB
  },
  linger: 10,         // 等待 10ms 积累消息
  
  // 3. 并发请求
  maxInFlightRequests: 5,
  
  // 4. 幂等性（防止重复）
  idempotent: true,
  
  // 5. 网络配置
  requestTimeout: 30000,
  socketTimeout: 60000
});

// 异步发送（不等待确认）
const sendAsync = async (messages: any[]) => {
  const promises = messages.map(msg =>
    producer.send({
      topic: 'my-topic',
      messages: [msg]
    }).catch(error => {
      console.error('Send failed:', error);
      return null;
    })
  );
  
  // 并行发送，不阻塞
  Promise.all(promises).catch(console.error);
};
```

### Consumer 优化

```typescript
const optimizedConsumer = kafka.consumer({
  groupId: 'optimized-group',
  
  // 1. 增加拉取大小
  maxBytes: 52428800,  // 50MB
  maxBytesPerPartition: 1048576,  // 1MB
  
  // 2. 减少等待时间
  maxWaitTimeInMs: 100,
  
  // 3. 会话超时
  sessionTimeout: 30000,
  heartbeatInterval: 3000
});

await optimizedConsumer.run({
  // 4. 并行处理分区
  partitionsConsumedConcurrently: 5,
  
  // 5. 批量处理
  eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
    // 并行处理消息
    await Promise.all(
      batch.messages.map(async (message) => {
        await processMessage(message);
        resolveOffset(message.offset);
      })
    );
    
    await heartbeat();
  }
});
```

### Topic 配置优化

```bash
# 创建 topic 时的重要配置
kafka-topics.sh --create \
  --topic my-topic \
  --partitions 10 \              # 分区数（影响并行度）
  --replication-factor 3 \       # 副本数（影响可靠性）
  --config retention.ms=604800000 \         # 7 天保留
  --config segment.bytes=1073741824 \       # 1GB 段大小
  --config compression.type=snappy \        # 压缩类型
  --config min.insync.replicas=2            # 最小同步副本数
```

```typescript
// 运行时配置
const admin = kafka.admin();

await admin.createTopics({
  topics: [
    {
      topic: 'high-throughput',
      numPartitions: 20,
      replicationFactor: 3,
      configEntries: [
        { name: 'compression.type', value: 'snappy' },
        { name: 'retention.ms', value: '604800000' },
        { name: 'segment.bytes', value: '1073741824' },
        { name: 'min.insync.replicas', value: '2' }
      ]
    }
  ]
});
```

## 实战场景

### 1. 事件溯源（Event Sourcing）

```typescript
interface Event {
  type: string;
  aggregateId: string;
  data: any;
  timestamp: number;
  version: number;
}

class EventStore {
  private producer: Producer;
  
  async appendEvent(event: Event) {
    await this.producer.send({
      topic: 'events',
      messages: [
        {
          // 使用 aggregateId 作为 key，确保顺序
          key: event.aggregateId,
          value: JSON.stringify(event),
          headers: {
            'event-type': event.type,
            'aggregate-id': event.aggregateId
          }
        }
      ]
    });
  }
  
  async replayEvents(
    aggregateId: string,
    handler: (event: Event) => void
  ) {
    const consumer = kafka.consumer({
      groupId: `replay-${Date.now()}`
    });
    
    await consumer.subscribe({
      topic: 'events',
      fromBeginning: true
    });
    
    await consumer.run({
      eachMessage: async ({ message }) => {
        const event: Event = JSON.parse(message.value!.toString());
        
        if (event.aggregateId === aggregateId) {
          handler(event);
        }
      }
    });
  }
}

// 使用
const eventStore = new EventStore(producer);

// 保存事件
await eventStore.appendEvent({
  type: 'OrderCreated',
  aggregateId: 'order-123',
  data: { items: [...], total: 100 },
  timestamp: Date.now(),
  version: 1
});

// 重放事件重建状态
let orderState = {};
await eventStore.replayEvents('order-123', (event) => {
  switch (event.type) {
    case 'OrderCreated':
      orderState = event.data;
      break;
    case 'OrderUpdated':
      orderState = { ...orderState, ...event.data };
      break;
  }
});
```

### 2. CQRS（命令查询职责分离）

```typescript
// 写入端：处理命令，产生事件
class CommandHandler {
  async handleCreateOrder(command: CreateOrderCommand) {
    // 验证命令
    this.validate(command);
    
    // 产生事件
    await producer.send({
      topic: 'order-events',
      messages: [
        {
          key: command.orderId,
          value: JSON.stringify({
            type: 'OrderCreated',
            orderId: command.orderId,
            items: command.items,
            total: command.total,
            timestamp: Date.now()
          })
        }
      ]
    });
  }
}

// 读取端：消费事件，更新查询模型
class QueryModelUpdater {
  async updateReadModel() {
    await consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value!.toString());
        
        switch (event.type) {
          case 'OrderCreated':
            await db.orders.insertOne({
              orderId: event.orderId,
              items: event.items,
              total: event.total,
              status: 'created',
              createdAt: event.timestamp
            });
            break;
            
          case 'OrderPaid':
            await db.orders.updateOne(
              { orderId: event.orderId },
              { $set: { status: 'paid', paidAt: event.timestamp } }
            );
            break;
        }
      }
    });
  }
}
```

### 3. Saga 模式（分布式事务）

```typescript
class OrderSaga {
  async execute(order: Order) {
    const saga = {
      orderId: order.id,
      steps: [
        { name: 'createOrder', status: 'pending' },
        { name: 'reserveInventory', status: 'pending' },
        { name: 'processPayment', status: 'pending' },
        { name: 'shipOrder', status: 'pending' }
      ]
    };
    
    try {
      // 1. 创建订单
      await this.sendCommand('orders', {
        type: 'CreateOrder',
        data: order
      });
      saga.steps[0].status = 'completed';
      
      // 2. 预留库存
      await this.sendCommand('inventory', {
        type: 'ReserveInventory',
        data: { orderId: order.id, items: order.items }
      });
      saga.steps[1].status = 'completed';
      
      // 3. 处理支付
      await this.sendCommand('payments', {
        type: 'ProcessPayment',
        data: { orderId: order.id, amount: order.total }
      });
      saga.steps[2].status = 'completed';
      
      // 4. 发货
      await this.sendCommand('shipping', {
        type: 'ShipOrder',
        data: { orderId: order.id }
      });
      saga.steps[3].status = 'completed';
      
    } catch (error) {
      // 补偿操作（回滚）
      await this.compensate(saga);
      throw error;
    }
  }
  
  private async compensate(saga: any) {
    // 反向执行补偿操作
    for (let i = saga.steps.length - 1; i >= 0; i--) {
      const step = saga.steps[i];
      
      if (step.status === 'completed') {
        switch (step.name) {
          case 'createOrder':
            await this.sendCommand('orders', {
              type: 'CancelOrder',
              data: { orderId: saga.orderId }
            });
            break;
            
          case 'reserveInventory':
            await this.sendCommand('inventory', {
              type: 'ReleaseInventory',
              data: { orderId: saga.orderId }
            });
            break;
            
          case 'processPayment':
            await this.sendCommand('payments', {
              type: 'RefundPayment',
              data: { orderId: saga.orderId }
            });
            break;
        }
      }
    }
  }
  
  private async sendCommand(topic: string, command: any) {
    await producer.send({
      topic,
      messages: [{ value: JSON.stringify(command) }]
    });
  }
}
```

## 监控和运维

### 消费者监控

```typescript
class ConsumerMonitor {
  private consumer: Consumer;
  private metrics = {
    messagesProcessed: 0,
    errors: 0,
    lag: new Map<string, number>()
  };
  
  async start() {
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const start = Date.now();
        
        try {
          await this.processMessage(message);
          this.metrics.messagesProcessed++;
        } catch (error) {
          this.metrics.errors++;
          throw error;
        } finally {
          const duration = Date.now() - start;
          
          // 记录处理时间
          console.log(`Processed in ${duration}ms`);
        }
      }
    });
    
    // 定期检查消费延迟
    setInterval(() => this.checkLag(), 60000);
  }
  
  private async checkLag() {
    const admin = kafka.admin();
    
    const offsets = await admin.fetchOffsets({
      groupId: 'my-consumer-group',
      topics: ['my-topic']
    });
    
    for (const offset of offsets) {
      for (const partition of offset.partitions) {
        const lag = partition.high - partition.offset;
        this.metrics.lag.set(
          `${offset.topic}-${partition.partition}`,
          parseInt(lag)
        );
        
        if (parseInt(lag) > 10000) {
          console.warn(`High lag detected: ${lag}`);
        }
      }
    }
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      lag: Object.fromEntries(this.metrics.lag)
    };
  }
}
```

## 常见面试题

### 1. Kafka 为什么这么快？

<details>
<summary>点击查看答案</summary>

**核心原因**：

1. **顺序写磁盘**：
   - 顺序写比随机写快 100 倍
   - 利用操作系统的页缓存

2. **零拷贝（Zero Copy）**：
   - 使用 sendfile 系统调用
   - 数据直接从磁盘到网卡，不经过应用层

3. **批量处理**：
   - 批量发送消息
   - 批量压缩
   - 批量写磁盘

4. **分区并行**：
   - 多分区支持并行读写
   - 负载均衡

5. **高效的协议**：
   - 二进制协议
   - 紧凑的消息格式

6. **页缓存**：
   - 依赖操作系统页缓存
   - 预读和写后缓存
</details>

### 2. 如何保证消息不丢失？

<details>
<summary>点击查看答案</summary>

**Producer 端**：
```typescript
const producer = kafka.producer({
  acks: -1,          // 等待所有 ISR 确认
  idempotent: true,  // 幂等性，防止重复
  retry: {
    retries: 3
  }
});
```

**Broker 端**：
```bash
# 配置
min.insync.replicas=2  # 最少 2 个副本同步
replication.factor=3    # 3 个副本
```

**Consumer 端**：
```typescript
// 先处理，后提交
await processMessage(message);
await consumer.commitOffsets([offset]);
```

**完整流程**：
1. Producer 设置 acks=-1
2. Broker 配置多副本和 min.insync.replicas
3. Consumer 手动提交偏移量
4. 业务逻辑做幂等性处理
</details>

### 3. 如何保证消息顺序？

<details>
<summary>点击查看答案</summary>

**单分区顺序**：
- Kafka 保证同一分区内的消息顺序
- 使用相同的 key 发送到同一分区

```typescript
// Producer：使用 key 保证顺序
await producer.send({
  topic: 'user-events',
  messages: [
    {
      key: 'user-123',  // 相同 key 发到同一分区
      value: JSON.stringify({ event: 'login' })
    }
  ]
});

// Consumer：单线程消费
await consumer.run({
  partitionsConsumedConcurrently: 1,  // 每次只处理一个分区
  eachMessage: async ({ message }) => {
    await processInOrder(message);
  }
});
```

**全局顺序**：
- 只使用 1 个分区（牺牲性能）
- 或使用外部协调（如数据库序号）

**部分顺序**：
- 按业务 key 分区
- 保证相同 key 的消息顺序
</details>

### 4. Kafka 的消费者再平衡（Rebalance）机制？

<details>
<summary>点击查看答案</summary>

**触发条件**：
1. 消费者加入或离开
2. 分区数量变化
3. 消费者心跳超时

**过程**：
1. 消费者停止消费
2. 重新分配分区
3. 消费者开始消费新分配的分区

**影响**：
- 短暂的消费暂停
- 可能的重复消费

**优化**：
```typescript
const consumer = kafka.consumer({
  groupId: 'my-group',
  
  // 增加会话超时
  sessionTimeout: 30000,
  
  // 减少心跳间隔
  heartbeatInterval: 3000,
  
  // 增加再平衡超时
  rebalanceTimeout: 60000
});

// 监听再平衡
consumer.on(consumer.events.GROUP_JOIN, ({ payload }) => {
  console.log('Rebalance: Joining group');
});

consumer.on(consumer.events.REBALANCING, ({ payload }) => {
  console.log('Rebalancing...');
});
```

**避免频繁再平衡**：
1. 调整心跳和会话超时参数
2. 增加处理速度
3. 监控消费延迟
</details>

### 5. Kafka vs RabbitMQ 的区别？

<details>
<summary>点击查看答案</summary>

| 特性 | Kafka | RabbitMQ |
|------|-------|----------|
| 架构 | 分布式日志 | 消息队列 |
| 吞吐量 | 非常高 | 中等 |
| 延迟 | 较低 | 非常低 |
| 消息顺序 | 分区内有序 | 队列有序 |
| 消息持久化 | 默认持久化 | 可选 |
| 消息保留 | 时间或大小 | 消费后删除 |
| 消息路由 | 简单（topic） | 复杂（exchange） |
| 消费模式 | Pull | Push |

**使用场景**：

**Kafka**：
- 大数据管道
- 事件溯源
- 日志聚合
- 流处理
- 需要重放消息

**RabbitMQ**：
- 任务队列
- RPC 调用
- 复杂路由
- 低延迟要求
- 传统消息队列场景
</details>

## 最佳实践

### 1. 分区数量选择

```typescript
// 考虑因素：
// 1. 目标吞吐量
// 2. 消费者数量
// 3. Broker 数量

// 计算公式：
// partitions = max(
//   targetThroughput / producerThroughput,
//   targetThroughput / consumerThroughput
// )

// 示例：
// 目标吞吐量：100MB/s
// Producer 吞吐量：10MB/s
// Consumer 吞吐量：5MB/s
// 分区数 = max(100/10, 100/5) = 20
```

### 2. 消息格式设计

```typescript
interface StandardMessage {
  // 消息ID（幂等性）
  messageId: string;
  
  // 时间戳
  timestamp: number;
  
  // 事件类型
  eventType: string;
  
  // 数据
  data: any;
  
  // 元数据
  metadata: {
    source: string;
    version: string;
    correlationId?: string;
  };
}
```

### 3. 错误处理策略

```typescript
// 1. 重试有限次数
// 2. 使用死信队列
// 3. 记录失败日志
// 4. 监控告警
```

### 4. 监控指标

- **Producer**: 发送速率、失败率、批次大小
- **Broker**: CPU、磁盘、网络、分区数
- **Consumer**: 消费延迟（lag）、处理速率、错误率

