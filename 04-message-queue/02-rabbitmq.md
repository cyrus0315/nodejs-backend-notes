# RabbitMQ

RabbitMQ 是一个成熟的消息代理（Message Broker），实现了 AMQP 协议，以灵活的路由和可靠的消息传递著称。

## 核心概念

### 架构组件

```
Producer → Exchange → Queue → Consumer
              ↓
           Binding
```

**Exchange（交换机）**：
- 接收消息并路由到队列
- 类型：Direct、Topic、Fanout、Headers

**Queue（队列）**：
- 存储消息
- 消费者从队列中取消息

**Binding（绑定）**：
- Exchange 和 Queue 之间的路由规则

**Routing Key**：
- 消息的路由标识

## Exchange 类型详解

### 1. Direct Exchange（直连）

精确匹配 routing key。

```typescript
import amqplib from 'amqplib';

async function setupDirectExchange() {
  const connection = await amqplib.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  // 创建 exchange
  await channel.assertExchange('logs_direct', 'direct', {
    durable: true
  });
  
  // 创建队列
  await channel.assertQueue('error_logs', { durable: true });
  await channel.assertQueue('info_logs', { durable: true });
  
  // 绑定
  await channel.bindQueue('error_logs', 'logs_direct', 'error');
  await channel.bindQueue('info_logs', 'logs_direct', 'info');
  
  // 发送消息
  channel.publish(
    'logs_direct',
    'error',  // routing key
    Buffer.from(JSON.stringify({ msg: 'Error occurred' }))
  );
  
  // error routing key → error_logs 队列
  // info routing key → info_logs 队列
}
```

### 2. Topic Exchange（主题）

模式匹配 routing key，支持通配符。

- `*`：匹配一个单词
- `#`：匹配零个或多个单词

```typescript
async function setupTopicExchange() {
  const channel = await connection.createChannel();
  
  await channel.assertExchange('logs_topic', 'topic', {
    durable: true
  });
  
  // 创建队列
  await channel.assertQueue('all_logs');
  await channel.assertQueue('critical_logs');
  await channel.assertQueue('system_logs');
  
  // 绑定模式
  await channel.bindQueue('all_logs', 'logs_topic', '#');
  await channel.bindQueue('critical_logs', 'logs_topic', '*.critical');
  await channel.bindQueue('system_logs', 'logs_topic', 'system.*');
  
  // 发送消息
  channel.publish(
    'logs_topic',
    'app.critical',  // 匹配 *.critical
    Buffer.from('Critical error')
  );
  
  channel.publish(
    'logs_topic',
    'system.info',  // 匹配 system.*
    Buffer.from('System info')
  );
  
  // app.critical → critical_logs, all_logs
  // system.info → system_logs, all_logs
  // db.warning.critical → critical_logs, all_logs
}
```

### 3. Fanout Exchange（扇出）

广播到所有绑定的队列，忽略 routing key。

```typescript
async function setupFanoutExchange() {
  const channel = await connection.createChannel();
  
  await channel.assertExchange('notifications', 'fanout', {
    durable: true
  });
  
  // 多个队列
  await channel.assertQueue('email_queue');
  await channel.assertQueue('sms_queue');
  await channel.assertQueue('push_queue');
  
  // 绑定（无需 routing key）
  await channel.bindQueue('email_queue', 'notifications', '');
  await channel.bindQueue('sms_queue', 'notifications', '');
  await channel.bindQueue('push_queue', 'notifications', '');
  
  // 发送消息
  channel.publish(
    'notifications',
    '',  // routing key 被忽略
    Buffer.from(JSON.stringify({
      user: 'user-123',
      message: 'New order'
    }))
  );
  
  // 消息会发送到所有三个队列
}
```

### 4. Headers Exchange（头部）

根据消息头部属性路由，不使用 routing key。

```typescript
async function setupHeadersExchange() {
  const channel = await connection.createChannel();
  
  await channel.assertExchange('tasks', 'headers', {
    durable: true
  });
  
  await channel.assertQueue('urgent_tasks');
  await channel.assertQueue('normal_tasks');
  
  // x-match: all（所有匹配）或 any（任意匹配）
  await channel.bindQueue('urgent_tasks', 'tasks', '', {
    'x-match': 'all',
    'priority': 'high',
    'type': 'urgent'
  });
  
  await channel.bindQueue('normal_tasks', 'tasks', '', {
    'x-match': 'any',
    'priority': 'low',
    'type': 'normal'
  });
  
  // 发送消息
  channel.publish(
    'tasks',
    '',
    Buffer.from('Urgent task'),
    {
      headers: {
        priority: 'high',
        type: 'urgent'
      }
    }
  );
}
```

## 消息可靠性

### 1. 生产者确认（Publisher Confirms）

```typescript
class ReliableProducer {
  private channel: Channel;
  
  async setup() {
    // 启用确认模式
    await this.channel.confirmSelect();
  }
  
  async sendMessage(
    exchange: string,
    routingKey: string,
    message: any
  ): Promise<void> {
    const content = Buffer.from(JSON.stringify(message));
    
    return new Promise((resolve, reject) => {
      this.channel.publish(
        exchange,
        routingKey,
        content,
        { persistent: true },  // 消息持久化
        (err) => {
          if (err) {
            reject(err);
          } else {
            resolve();
          }
        }
      );
    });
  }
  
  async sendBatch(messages: Array<{
    exchange: string;
    routingKey: string;
    content: any;
  }>) {
    // 批量发送
    messages.forEach(msg => {
      this.channel.publish(
        msg.exchange,
        msg.routingKey,
        Buffer.from(JSON.stringify(msg.content)),
        { persistent: true }
      );
    });
    
    // 等待所有确认
    await this.channel.waitForConfirms();
  }
}
```

### 2. 消息持久化

```typescript
async function setupDurableQueue() {
  const channel = await connection.createChannel();
  
  // 1. Exchange 持久化
  await channel.assertExchange('durable_exchange', 'direct', {
    durable: true  // 重启后保留
  });
  
  // 2. Queue 持久化
  await channel.assertQueue('durable_queue', {
    durable: true,
    arguments: {
      // 队列最大长度
      'x-max-length': 10000,
      // 消息 TTL（毫秒）
      'x-message-ttl': 86400000,  // 1 天
      // 队列 TTL（未使用时删除）
      'x-expires': 3600000  // 1 小时
    }
  });
  
  await channel.bindQueue('durable_queue', 'durable_exchange', 'key');
  
  // 3. 消息持久化
  channel.publish(
    'durable_exchange',
    'key',
    Buffer.from('Important message'),
    {
      persistent: true,  // 消息持久化
      
      // 消息优先级（0-255）
      priority: 10,
      
      // 消息过期时间
      expiration: '60000',  // 60 秒
      
      // 消息 ID
      messageId: 'msg-123',
      
      // 时间戳
      timestamp: Date.now(),
      
      // 内容类型
      contentType: 'application/json'
    }
  );
}
```

### 3. 消费者确认（ACK）

```typescript
class ReliableConsumer {
  private channel: Channel;
  
  async consume(queueName: string) {
    // 关闭自动确认
    await this.channel.consume(
      queueName,
      async (msg) => {
        if (!msg) return;
        
        try {
          // 处理消息
          const content = JSON.parse(msg.content.toString());
          await this.processMessage(content);
          
          // 手动确认
          this.channel.ack(msg);
          
        } catch (error) {
          console.error('Processing failed:', error);
          
          // 拒绝消息
          this.channel.nack(msg, false, true);
          // 参数：msg, allUpTo, requeue
          // allUpTo: false - 只拒绝当前消息
          // requeue: true - 重新入队
        }
      },
      {
        noAck: false,  // 手动确认
        prefetch: 10   // 预取数量
      }
    );
  }
  
  // 批量确认
  async batchConsume(queueName: string) {
    let messageCount = 0;
    let lastMessage: Message | null = null;
    
    await this.channel.consume(
      queueName,
      async (msg) => {
        if (!msg) return;
        
        await this.processMessage(msg);
        
        messageCount++;
        lastMessage = msg;
        
        // 每 10 条消息批量确认
        if (messageCount >= 10) {
          this.channel.ack(lastMessage, true);  // true: 确认所有
          messageCount = 0;
        }
      },
      { noAck: false }
    );
  }
  
  private async processMessage(msg: any) {
    // 业务逻辑
  }
}
```

### 4. 死信队列（DLQ）

```typescript
async function setupDLQ() {
  const channel = await connection.createChannel();
  
  // 1. 创建死信 exchange
  await channel.assertExchange('dlx', 'direct', {
    durable: true
  });
  
  // 2. 创建死信队列
  await channel.assertQueue('dead_letters', {
    durable: true
  });
  
  await channel.bindQueue('dead_letters', 'dlx', 'dead');
  
  // 3. 创建业务队列，配置 DLX
  await channel.assertQueue('tasks', {
    durable: true,
    arguments: {
      // 死信交换机
      'x-dead-letter-exchange': 'dlx',
      // 死信 routing key
      'x-dead-letter-routing-key': 'dead',
      
      // 消息 TTL（过期进入 DLQ）
      'x-message-ttl': 60000,
      
      // 最大重试次数
      'x-max-length': 1000
    }
  });
  
  // 消息处理失败时发送到 DLQ
  await channel.consume('tasks', async (msg) => {
    if (!msg) return;
    
    try {
      await processMessage(msg);
      channel.ack(msg);
    } catch (error) {
      // 不重新入队，让消息进入 DLQ
      channel.nack(msg, false, false);
    }
  }, { noAck: false });
  
  // 处理死信
  await channel.consume('dead_letters', async (msg) => {
    if (!msg) return;
    
    console.error('Dead letter:', {
      content: msg.content.toString(),
      reason: msg.properties.headers['x-death'],
      originalQueue: msg.properties.headers['x-death'][0].queue
    });
    
    channel.ack(msg);
  });
}
```

## 高级特性

### 1. 优先级队列

```typescript
async function setupPriorityQueue() {
  const channel = await connection.createChannel();
  
  // 创建优先级队列（0-255）
  await channel.assertQueue('priority_tasks', {
    durable: true,
    arguments: {
      'x-max-priority': 10  // 最大优先级
    }
  });
  
  // 发送不同优先级的消息
  for (let i = 0; i < 10; i++) {
    channel.sendToQueue(
      'priority_tasks',
      Buffer.from(`Task ${i}`),
      {
        priority: i,  // 优先级 0-9
        persistent: true
      }
    );
  }
  
  // 消费时，高优先级消息先被处理
  await channel.consume('priority_tasks', (msg) => {
    if (!msg) return;
    console.log('Processing:', msg.content.toString());
    channel.ack(msg);
  });
}
```

### 2. 延迟队列

```typescript
async function setupDelayedQueue() {
  const channel = await connection.createChannel();
  
  // 方法 1：使用 TTL + DLX
  await channel.assertExchange('delayed', 'direct', { durable: true });
  await channel.assertExchange('actual', 'direct', { durable: true });
  
  // 延迟队列（无消费者）
  await channel.assertQueue('delay_queue', {
    durable: true,
    arguments: {
      'x-message-ttl': 30000,  // 30 秒后过期
      'x-dead-letter-exchange': 'actual',  // 过期后发送到 actual
      'x-dead-letter-routing-key': 'process'
    }
  });
  
  // 实际处理队列
  await channel.assertQueue('process_queue', { durable: true });
  
  await channel.bindQueue('delay_queue', 'delayed', 'delay');
  await channel.bindQueue('process_queue', 'actual', 'process');
  
  // 发送延迟消息
  channel.publish(
    'delayed',
    'delay',
    Buffer.from('Delayed task'),
    { persistent: true }
  );
  
  // 30 秒后消息会出现在 process_queue
  await channel.consume('process_queue', (msg) => {
    if (!msg) return;
    console.log('Processing delayed:', msg.content.toString());
    channel.ack(msg);
  });
  
  // 方法 2：使用插件（rabbitmq-delayed-message-exchange）
  await channel.assertExchange('delayed_plugin', 'x-delayed-message', {
    durable: true,
    arguments: {
      'x-delayed-type': 'direct'
    }
  });
  
  channel.publish(
    'delayed_plugin',
    'routing_key',
    Buffer.from('Delayed message'),
    {
      headers: {
        'x-delay': 30000  // 30 秒延迟
      }
    }
  );
}
```

### 3. RPC 模式

```typescript
// RPC Server
class RPCServer {
  private channel: Channel;
  
  async start() {
    await this.channel.assertQueue('rpc_queue', {
      durable: false
    });
    
    // 设置 prefetch
    this.channel.prefetch(1);
    
    await this.channel.consume('rpc_queue', async (msg) => {
      if (!msg) return;
      
      const input = parseInt(msg.content.toString());
      const result = this.fibonacci(input);
      
      // 发送响应到 replyTo 队列
      this.channel.sendToQueue(
        msg.properties.replyTo,
        Buffer.from(result.toString()),
        {
          correlationId: msg.properties.correlationId
        }
      );
      
      this.channel.ack(msg);
    });
  }
  
  private fibonacci(n: number): number {
    if (n <= 1) return n;
    return this.fibonacci(n - 1) + this.fibonacci(n - 2);
  }
}

// RPC Client
class RPCClient {
  private channel: Channel;
  private replyQueue: string;
  private pendingRequests = new Map<string, (value: any) => void>();
  
  async setup() {
    // 创建回复队列
    const { queue } = await this.channel.assertQueue('', {
      exclusive: true
    });
    this.replyQueue = queue;
    
    // 监听回复
    await this.channel.consume(
      this.replyQueue,
      (msg) => {
        if (!msg) return;
        
        const correlationId = msg.properties.correlationId;
        const resolve = this.pendingRequests.get(correlationId);
        
        if (resolve) {
          const result = parseInt(msg.content.toString());
          resolve(result);
          this.pendingRequests.delete(correlationId);
        }
      },
      { noAck: true }
    );
  }
  
  async call(n: number): Promise<number> {
    const correlationId = this.generateUuid();
    
    return new Promise((resolve) => {
      this.pendingRequests.set(correlationId, resolve);
      
      this.channel.sendToQueue(
        'rpc_queue',
        Buffer.from(n.toString()),
        {
          correlationId,
          replyTo: this.replyQueue
        }
      );
      
      // 超时处理
      setTimeout(() => {
        if (this.pendingRequests.has(correlationId)) {
          this.pendingRequests.delete(correlationId);
          resolve(-1);  // 超时返回 -1
        }
      }, 5000);
    });
  }
  
  private generateUuid(): string {
    return Math.random().toString(36) + Date.now().toString(36);
  }
}

// 使用
const server = new RPCServer(channel);
await server.start();

const client = new RPCClient(channel);
await client.setup();
const result = await client.call(10);
console.log('Fibonacci(10) =', result);
```

### 4. 消息路由模式

```typescript
// 工作队列模式（Work Queue）
async function workQueue() {
  const channel = await connection.createChannel();
  
  await channel.assertQueue('tasks', { durable: true });
  
  // 公平分发
  channel.prefetch(1);
  
  // Worker 1
  channel.consume('tasks', async (msg) => {
    if (!msg) return;
    
    const task = msg.content.toString();
    await processTask(task);
    channel.ack(msg);
  });
  
  // Worker 2
  channel.consume('tasks', async (msg) => {
    if (!msg) return;
    
    const task = msg.content.toString();
    await processTask(task);
    channel.ack(msg);
  });
  
  // 发送任务
  for (let i = 0; i < 100; i++) {
    channel.sendToQueue(
      'tasks',
      Buffer.from(`Task ${i}`),
      { persistent: true }
    );
  }
}
```

## 性能优化

### 1. 连接池

```typescript
class RabbitMQPool {
  private connection: Connection | null = null;
  private channels: Channel[] = [];
  private maxChannels = 10;
  private availableChannels: Channel[] = [];
  
  async getConnection(): Promise<Connection> {
    if (!this.connection) {
      this.connection = await amqplib.connect({
        hostname: 'localhost',
        port: 5672,
        username: 'guest',
        password: 'guest',
        
        // 连接选项
        heartbeat: 60,
        frameMax: 0,
        
        // 连接池
        connectionTimeout: 10000
      });
      
      this.connection.on('error', (err) => {
        console.error('Connection error:', err);
        this.connection = null;
      });
    }
    
    return this.connection;
  }
  
  async getChannel(): Promise<Channel> {
    // 从池中获取
    if (this.availableChannels.length > 0) {
      return this.availableChannels.pop()!;
    }
    
    // 创建新 channel
    if (this.channels.length < this.maxChannels) {
      const connection = await this.getConnection();
      const channel = await connection.createChannel();
      
      this.channels.push(channel);
      return channel;
    }
    
    // 等待可用 channel
    return new Promise((resolve) => {
      const interval = setInterval(() => {
        if (this.availableChannels.length > 0) {
          clearInterval(interval);
          resolve(this.availableChannels.pop()!);
        }
      }, 100);
    });
  }
  
  releaseChannel(channel: Channel) {
    this.availableChannels.push(channel);
  }
  
  async close() {
    await Promise.all(
      this.channels.map(ch => ch.close())
    );
    
    if (this.connection) {
      await this.connection.close();
    }
  }
}

// 使用
const pool = new RabbitMQPool();

async function sendMessage(msg: any) {
  const channel = await pool.getChannel();
  
  try {
    channel.sendToQueue('my_queue', Buffer.from(JSON.stringify(msg)));
  } finally {
    pool.releaseChannel(channel);
  }
}
```

### 2. 批量操作

```typescript
class BatchPublisher {
  private channel: Channel;
  private batch: Array<{
    queue: string;
    content: Buffer;
    options?: any;
  }> = [];
  private batchSize = 100;
  private flushInterval = 1000;
  
  constructor(channel: Channel) {
    this.channel = channel;
    this.startFlushTimer();
  }
  
  async publish(
    queue: string,
    content: any,
    options?: any
  ) {
    this.batch.push({
      queue,
      content: Buffer.from(JSON.stringify(content)),
      options
    });
    
    if (this.batch.length >= this.batchSize) {
      await this.flush();
    }
  }
  
  private async flush() {
    if (this.batch.length === 0) return;
    
    const messages = this.batch.splice(0, this.batch.length);
    
    // 批量发送
    messages.forEach(msg => {
      this.channel.sendToQueue(
        msg.queue,
        msg.content,
        msg.options
      );
    });
  }
  
  private startFlushTimer() {
    setInterval(() => {
      this.flush().catch(console.error);
    }, this.flushInterval);
  }
}
```

### 3. Prefetch 优化

```typescript
async function optimizeConsumer() {
  const channel = await connection.createChannel();
  
  // 设置预取数量
  // 太小：吞吐量低
  // 太大：内存占用高，负载不均
  channel.prefetch(10);  // 建议 5-20
  
  await channel.consume('tasks', async (msg) => {
    if (!msg) return;
    
    try {
      await processMessage(msg);
      channel.ack(msg);
    } catch (error) {
      channel.nack(msg, false, true);
    }
  }, { noAck: false });
}
```

## 监控和管理

### 1. 管理 API

```typescript
import axios from 'axios';

class RabbitMQManager {
  private baseUrl = 'http://localhost:15672/api';
  private auth = {
    username: 'guest',
    password: 'guest'
  };
  
  // 获取队列信息
  async getQueue(queueName: string) {
    const response = await axios.get(
      `${this.baseUrl}/queues/%2f/${queueName}`,
      { auth: this.auth }
    );
    
    return {
      messages: response.data.messages,
      consumers: response.data.consumers,
      messageStats: response.data.message_stats
    };
  }
  
  // 获取所有队列
  async listQueues() {
    const response = await axios.get(
      `${this.baseUrl}/queues`,
      { auth: this.auth }
    );
    
    return response.data.map((q: any) => ({
      name: q.name,
      messages: q.messages,
      consumers: q.consumers
    }));
  }
  
  // 清空队列
  async purgeQueue(queueName: string) {
    await axios.delete(
      `${this.baseUrl}/queues/%2f/${queueName}/contents`,
      { auth: this.auth }
    );
  }
  
  // 获取连接信息
  async getConnections() {
    const response = await axios.get(
      `${this.baseUrl}/connections`,
      { auth: this.auth }
    );
    
    return response.data.map((c: any) => ({
      name: c.name,
      user: c.user,
      state: c.state,
      channels: c.channels
    }));
  }
}
```

### 2. 健康检查

```typescript
class HealthCheck {
  private connection: Connection;
  private channel: Channel;
  
  async check(): Promise<boolean> {
    try {
      // 检查连接
      if (!this.connection) {
        return false;
      }
      
      // 检查 channel
      const testQueue = `health_check_${Date.now()}`;
      await this.channel.assertQueue(testQueue, {
        autoDelete: true
      });
      
      // 发送测试消息
      this.channel.sendToQueue(
        testQueue,
        Buffer.from('ping')
      );
      
      // 接收测试消息
      const msg = await this.channel.get(testQueue, { noAck: true });
      
      // 删除测试队列
      await this.channel.deleteQueue(testQueue);
      
      return msg !== false;
    } catch (error) {
      console.error('Health check failed:', error);
      return false;
    }
  }
}
```

## 实战场景

### 1. 任务队列系统

```typescript
interface Task {
  id: string;
  type: string;
  data: any;
  priority: number;
  retries: number;
  maxRetries: number;
}

class TaskQueue {
  private channel: Channel;
  
  async setup() {
    // 创建任务队列
    await this.channel.assertQueue('tasks', {
      durable: true,
      arguments: {
        'x-max-priority': 10,
        'x-dead-letter-exchange': 'dlx',
        'x-dead-letter-routing-key': 'failed_tasks'
      }
    });
    
    // 死信队列
    await this.channel.assertExchange('dlx', 'direct', {
      durable: true
    });
    
    await this.channel.assertQueue('failed_tasks', {
      durable: true
    });
    
    await this.channel.bindQueue('failed_tasks', 'dlx', 'failed_tasks');
  }
  
  async enqueueTask(task: Task) {
    this.channel.sendToQueue(
      'tasks',
      Buffer.from(JSON.stringify(task)),
      {
        priority: task.priority,
        persistent: true,
        messageId: task.id
      }
    );
  }
  
  async processTask() {
    this.channel.prefetch(1);
    
    await this.channel.consume('tasks', async (msg) => {
      if (!msg) return;
      
      const task: Task = JSON.parse(msg.content.toString());
      
      try {
        await this.executeTask(task);
        this.channel.ack(msg);
      } catch (error) {
        task.retries++;
        
        if (task.retries < task.maxRetries) {
          // 重新入队
          await this.enqueueTask(task);
          this.channel.ack(msg);
        } else {
          // 发送到死信队列
          this.channel.nack(msg, false, false);
        }
      }
    });
  }
  
  private async executeTask(task: Task) {
    // 根据任务类型执行
    switch (task.type) {
      case 'email':
        await this.sendEmail(task.data);
        break;
      case 'export':
        await this.exportData(task.data);
        break;
      default:
        throw new Error(`Unknown task type: ${task.type}`);
    }
  }
  
  private async sendEmail(data: any) {
    // 发送邮件逻辑
  }
  
  private async exportData(data: any) {
    // 导出数据逻辑
  }
}
```

### 2. 事件驱动架构

```typescript
class EventBus {
  private channel: Channel;
  
  async setup() {
    // 事件交换机
    await this.channel.assertExchange('events', 'topic', {
      durable: true
    });
  }
  
  // 发布事件
  async publish(eventType: string, data: any) {
    const event = {
      type: eventType,
      data,
      timestamp: Date.now(),
      id: this.generateId()
    };
    
    this.channel.publish(
      'events',
      eventType,
      Buffer.from(JSON.stringify(event)),
      { persistent: true }
    );
  }
  
  // 订阅事件
  async subscribe(
    pattern: string,
    handler: (event: any) => Promise<void>
  ) {
    const { queue } = await this.channel.assertQueue('', {
      exclusive: true
    });
    
    await this.channel.bindQueue(queue, 'events', pattern);
    
    await this.channel.consume(queue, async (msg) => {
      if (!msg) return;
      
      const event = JSON.parse(msg.content.toString());
      
      try {
        await handler(event);
        this.channel.ack(msg);
      } catch (error) {
        console.error('Handler error:', error);
        this.channel.nack(msg, false, true);
      }
    });
  }
  
  private generateId(): string {
    return `${Date.now()}-${Math.random().toString(36)}`;
  }
}

// 使用
const eventBus = new EventBus(channel);
await eventBus.setup();

// 发布事件
await eventBus.publish('user.created', {
  userId: '123',
  email: 'user@example.com'
});

await eventBus.publish('order.placed', {
  orderId: '456',
  amount: 100
});

// 订阅事件
await eventBus.subscribe('user.*', async (event) => {
  console.log('User event:', event);
  // 处理用户相关事件
});

await eventBus.subscribe('order.placed', async (event) => {
  console.log('Order placed:', event);
  // 发送确认邮件
  // 更新库存
  // 通知物流
});
```

## 常见面试题

### 1. RabbitMQ vs Kafka 如何选择？

<details>
<summary>点击查看答案</summary>

**RabbitMQ 优势**：
- 低延迟（微秒级）
- 灵活的路由（Exchange 类型丰富）
- 消息确认机制完善
- 支持优先级队列
- 适合传统消息队列场景

**Kafka 优势**：
- 高吞吐量（百万级 TPS）
- 消息持久化和重放
- 分布式架构
- 流处理支持
- 适合大数据和事件流场景

**选择建议**：
- 低延迟、复杂路由 → RabbitMQ
- 高吞吐、消息重放 → Kafka
- RPC、任务队列 → RabbitMQ
- 日志收集、事件溯源 → Kafka
</details>

### 2. 如何保证消息不丢失？

<details>
<summary>点击查看答案</summary>

**三个层面保证**：

1. **Producer**：
```typescript
// 启用确认模式
await channel.confirmSelect();

channel.publish(
  exchange,
  routingKey,
  content,
  { persistent: true },
  (err) => {
    if (err) {
      // 重试或记录
    }
  }
);
```

2. **Broker**：
```typescript
// 持久化 exchange、queue、message
await channel.assertExchange(name, type, { durable: true });
await channel.assertQueue(name, { durable: true });
channel.publish(exchange, key, msg, { persistent: true });
```

3. **Consumer**：
```typescript
// 手动 ACK
await channel.consume(queue, async (msg) => {
  await processMessage(msg);
  channel.ack(msg);  // 处理成功后确认
}, { noAck: false });
```
</details>

### 3. RabbitMQ 的集群和高可用？

<details>
<summary>点击查看答案</summary>

**普通集群**：
- 元数据同步
- 消息不复制
- 节点故障消息丢失

**镜像队列**：
- 消息复制到多个节点
- 主节点故障自动切换
- 性能下降（同步复制）

```bash
# 设置镜像策略
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
```

**Quorum 队列**（推荐）：
```typescript
await channel.assertQueue('my_queue', {
  durable: true,
  arguments: {
    'x-queue-type': 'quorum'  // 使用 Raft 算法
  }
});
```

**特点**：
- 基于 Raft 共识算法
- 自动主从切换
- 更好的数据一致性
- 更高的可用性
</details>

### 4. 如何处理消息堆积？

<details>
<summary>点击查看答案</summary>

**原因分析**：
1. 消费速度慢
2. 消费者数量不足
3. 消息处理失败重试

**解决方案**：

1. **增加消费者**：
```typescript
// 启动多个消费者
for (let i = 0; i < 10; i++) {
  startConsumer();
}
```

2. **提高消费速度**：
```typescript
// 增加 prefetch
channel.prefetch(50);

// 并行处理
await channel.consume(queue, async (msg) => {
  // 不等待处理完成
  processAsync(msg).catch(console.error);
  channel.ack(msg);
});
```

3. **流量控制**：
```typescript
// 限制队列长度
await channel.assertQueue('limited', {
  arguments: {
    'x-max-length': 10000,
    'x-overflow': 'reject-publish'  // 拒绝新消息
  }
});
```

4. **优先级处理**：
```typescript
// 高优先级消息优先处理
await channel.assertQueue('priority', {
  arguments: {
    'x-max-priority': 10
  }
});
```
</details>

### 5. RabbitMQ 的事务 vs 确认模式？

<details>
<summary>点击查看答案</summary>

**事务模式**（不推荐）：
```typescript
channel.txSelect();
try {
  channel.publish(exchange, key, msg1);
  channel.publish(exchange, key, msg2);
  await channel.txCommit();
} catch (error) {
  await channel.txRollback();
}
```

**缺点**：
- 同步操作，性能差
- 吞吐量降低 250 倍

**确认模式**（推荐）：
```typescript
await channel.confirmSelect();

channel.publish(exchange, key, msg, {}, (err) => {
  if (err) {
    // 处理失败
  }
});
```

**优点**：
- 异步确认
- 高性能
- 批量确认

**对比**：
| 特性 | 事务 | 确认 |
|------|------|------|
| 性能 | 低 | 高 |
| 吞吐量 | 250 msg/s | 50,000+ msg/s |
| 可靠性 | 强 | 强 |
| 实现复杂度 | 简单 | 中等 |
</details>

## 最佳实践

### 1. 连接和 Channel 管理

```typescript
// ✅ 推荐：一个应用一个连接，多个 channel
const connection = await amqplib.connect(url);
const channel1 = await connection.createChannel();
const channel2 = await connection.createChannel();

// ❌ 避免：多个连接
```

### 2. 队列命名规范

```typescript
// 建议命名规范
const queueNames = {
  tasks: 'app.tasks',
  events: 'app.events.user',
  dlq: 'app.tasks.dlq'
};
```

### 3. 消息格式

```typescript
interface Message {
  id: string;
  type: string;
  timestamp: number;
  data: any;
  metadata: {
    source: string;
    version: string;
    correlationId?: string;
  };
}
```

### 4. 错误处理

```typescript
// 分类错误，区别处理
try {
  await processMessage(msg);
  channel.ack(msg);
} catch (error) {
  if (isRetryableError(error)) {
    // 可重试错误：重新入队
    channel.nack(msg, false, true);
  } else {
    // 不可重试错误：发送到 DLQ
    channel.nack(msg, false, false);
  }
}
```

