# AWS SQS & SNS

AWS 提供的托管消息服务，SQS（Simple Queue Service）是消息队列，SNS（Simple Notification Service）是发布/订阅服务。

## SQS（消息队列）

### 核心概念

**Queue Types**：
- **Standard Queue**：无限吞吐量，至少一次传递，消息可能乱序
- **FIFO Queue**：顺序保证，精确一次传递，吞吐量有限（3000 TPS）

**消息特性**：
- 消息大小：最大 256KB
- 消息保留：1 分钟 - 14 天（默认 4 天）
- 可见性超时：0 - 12 小时（默认 30 秒）
- 延迟队列：0 - 15 分钟

### Standard Queue vs FIFO Queue

```typescript
import { SQSClient, SendMessageCommand, ReceiveMessageCommand } from '@aws-sdk/client-sqs';

const client = new SQSClient({ region: 'us-east-1' });

// Standard Queue
const standardQueueUrl = 'https://sqs.us-east-1.amazonaws.com/123456789/my-queue';

// FIFO Queue（必须以 .fifo 结尾）
const fifoQueueUrl = 'https://sqs.us-east-1.amazonaws.com/123456789/my-queue.fifo';

// 发送到 Standard Queue
await client.send(new SendMessageCommand({
  QueueUrl: standardQueueUrl,
  MessageBody: JSON.stringify({
    orderId: '123',
    amount: 100
  }),
  
  // 可选：延迟发送（0-900 秒）
  DelaySeconds: 10,
  
  // 可选：消息属性
  MessageAttributes: {
    'Priority': {
      DataType: 'String',
      StringValue: 'High'
    },
    'Timestamp': {
      DataType: 'Number',
      StringValue: Date.now().toString()
    }
  }
}));

// 发送到 FIFO Queue
await client.send(new SendMessageCommand({
  QueueUrl: fifoQueueUrl,
  MessageBody: JSON.stringify({ orderId: '456' }),
  
  // 必须：消息组 ID（相同组内有序）
  MessageGroupId: 'order-group',
  
  // 可选：去重 ID（5 分钟内去重）
  MessageDeduplicationId: '456'
  // 或启用 ContentBasedDeduplication（基于消息内容 hash）
}));
```

### 发送消息

```typescript
class SQSProducer {
  private client: SQSClient;
  private queueUrl: string;
  
  constructor(queueUrl: string) {
    this.client = new SQSClient({ region: 'us-east-1' });
    this.queueUrl = queueUrl;
  }
  
  // 发送单条消息
  async sendMessage(message: any, options?: {
    delaySeconds?: number;
    messageGroupId?: string;
    deduplicationId?: string;
  }) {
    const command = new SendMessageCommand({
      QueueUrl: this.queueUrl,
      MessageBody: JSON.stringify(message),
      DelaySeconds: options?.delaySeconds,
      MessageGroupId: options?.messageGroupId,
      MessageDeduplicationId: options?.deduplicationId,
      MessageAttributes: {
        'Timestamp': {
          DataType: 'Number',
          StringValue: Date.now().toString()
        }
      }
    });
    
    const response = await this.client.send(command);
    return response.MessageId;
  }
  
  // 批量发送（最多 10 条）
  async sendBatch(messages: any[]) {
    const { SendMessageBatchCommand } = await import('@aws-sdk/client-sqs');
    
    // 分批处理
    const batches = this.chunk(messages, 10);
    const results = [];
    
    for (const batch of batches) {
      const command = new SendMessageBatchCommand({
        QueueUrl: this.queueUrl,
        Entries: batch.map((msg, index) => ({
          Id: index.toString(),
          MessageBody: JSON.stringify(msg),
          MessageAttributes: {
            'Timestamp': {
              DataType: 'Number',
              StringValue: Date.now().toString()
            }
          }
        }))
      });
      
      const response = await this.client.send(command);
      results.push(response);
    }
    
    return results;
  }
  
  private chunk<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}
```

### 接收消息

```typescript
class SQSConsumer {
  private client: SQSClient;
  private queueUrl: string;
  private isRunning = false;
  
  constructor(queueUrl: string) {
    this.client = new SQSClient({ region: 'us-east-1' });
    this.queueUrl = queueUrl;
  }
  
  // 长轮询接收消息
  async start(handler: (message: any) => Promise<void>) {
    this.isRunning = true;
    
    while (this.isRunning) {
      try {
        const command = new ReceiveMessageCommand({
          QueueUrl: this.queueUrl,
          
          // 最多接收消息数（1-10）
          MaxNumberOfMessages: 10,
          
          // 可见性超时（秒）
          VisibilityTimeout: 30,
          
          // 长轮询（0-20 秒）
          WaitTimeSeconds: 20,
          
          // 接收消息属性
          MessageAttributeNames: ['All'],
          
          // 接收系统属性
          AttributeNames: ['All']
        });
        
        const response = await this.client.send(command);
        
        if (response.Messages && response.Messages.length > 0) {
          // 并行处理消息
          await Promise.all(
            response.Messages.map(msg => this.processMessage(msg, handler))
          );
        }
      } catch (error) {
        console.error('Receive error:', error);
        await this.sleep(5000);
      }
    }
  }
  
  private async processMessage(
    message: any,
    handler: (msg: any) => Promise<void>
  ) {
    try {
      const body = JSON.parse(message.Body);
      await handler(body);
      
      // 处理成功，删除消息
      await this.deleteMessage(message.ReceiptHandle);
      
    } catch (error) {
      console.error('Processing error:', error);
      
      // 处理失败，修改可见性超时（延迟重试）
      await this.changeVisibility(message.ReceiptHandle, 60);
    }
  }
  
  // 删除消息
  private async deleteMessage(receiptHandle: string) {
    const { DeleteMessageCommand } = await import('@aws-sdk/client-sqs');
    
    await this.client.send(new DeleteMessageCommand({
      QueueUrl: this.queueUrl,
      ReceiptHandle: receiptHandle
    }));
  }
  
  // 修改可见性超时
  private async changeVisibility(
    receiptHandle: string,
    timeoutSeconds: number
  ) {
    const { ChangeMessageVisibilityCommand } = await import('@aws-sdk/client-sqs');
    
    await this.client.send(new ChangeMessageVisibilityCommand({
      QueueUrl: this.queueUrl,
      ReceiptHandle: receiptHandle,
      VisibilityTimeout: timeoutSeconds
    }));
  }
  
  stop() {
    this.isRunning = false;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// 使用
const consumer = new SQSConsumer(queueUrl);

await consumer.start(async (message) => {
  console.log('Processing:', message);
  
  // 业务逻辑
  await processOrder(message);
});

// 优雅停止
process.on('SIGTERM', () => {
  consumer.stop();
});
```

### 死信队列（DLQ）

```typescript
import { 
  CreateQueueCommand, 
  SetQueueAttributesCommand,
  GetQueueAttributesCommand 
} from '@aws-sdk/client-sqs';

async function setupDLQ() {
  const client = new SQSClient({ region: 'us-east-1' });
  
  // 1. 创建死信队列
  const dlqResponse = await client.send(new CreateQueueCommand({
    QueueName: 'my-queue-dlq.fifo',
    Attributes: {
      'FifoQueue': 'true',
      'ContentBasedDeduplication': 'true'
    }
  }));
  
  const dlqUrl = dlqResponse.QueueUrl!;
  
  // 获取 DLQ 的 ARN
  const dlqAttrs = await client.send(new GetQueueAttributesCommand({
    QueueUrl: dlqUrl,
    AttributeNames: ['QueueArn']
  }));
  
  const dlqArn = dlqAttrs.Attributes!['QueueArn'];
  
  // 2. 创建主队列，配置 DLQ
  const mainQueueResponse = await client.send(new CreateQueueCommand({
    QueueName: 'my-queue.fifo',
    Attributes: {
      'FifoQueue': 'true',
      'ContentBasedDeduplication': 'true',
      
      // 重驱动策略
      'RedrivePolicy': JSON.stringify({
        deadLetterTargetArn: dlqArn,
        maxReceiveCount: '3'  // 重试 3 次后进入 DLQ
      })
    }
  }));
  
  return {
    mainQueueUrl: mainQueueResponse.QueueUrl!,
    dlqUrl
  };
}

// 处理死信队列中的消息
async function processDLQ(dlqUrl: string) {
  const consumer = new SQSConsumer(dlqUrl);
  
  await consumer.start(async (message) => {
    console.error('Dead letter:', message);
    
    // 记录到日志/数据库
    await logFailedMessage(message);
    
    // 可能的操作：
    // 1. 人工处理
    // 2. 修复后重新发送到主队列
    // 3. 归档
  });
}
```

### 延迟队列和延迟消息

```typescript
// 方法 1：队列级别延迟
async function createDelayedQueue() {
  const client = new SQSClient({ region: 'us-east-1' });
  
  const response = await client.send(new CreateQueueCommand({
    QueueName: 'delayed-queue',
    Attributes: {
      // 所有消息延迟 60 秒
      'DelaySeconds': '60'
    }
  }));
  
  return response.QueueUrl!;
}

// 方法 2：消息级别延迟（更灵活）
async function sendDelayedMessage(queueUrl: string) {
  const client = new SQSClient({ region: 'us-east-1' });
  
  await client.send(new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: JSON.stringify({ task: 'delayed-task' }),
    
    // 此消息延迟 300 秒（5 分钟）
    DelaySeconds: 300
  }));
}

// 实现定时任务
class ScheduledTaskQueue {
  private queueUrl: string;
  private client: SQSClient;
  
  async scheduleTask(task: any, delaySeconds: number) {
    // 最大延迟 15 分钟
    if (delaySeconds > 900) {
      // 使用多级延迟
      return this.scheduleTaskMultiLevel(task, delaySeconds);
    }
    
    await this.client.send(new SendMessageCommand({
      QueueUrl: this.queueUrl,
      MessageBody: JSON.stringify(task),
      DelaySeconds: delaySeconds
    }));
  }
  
  // 处理超过 15 分钟的延迟
  private async scheduleTaskMultiLevel(task: any, delaySeconds: number) {
    const firstDelay = 900;  // 15 分钟
    const remainingDelay = delaySeconds - firstDelay;
    
    // 第一级延迟
    await this.client.send(new SendMessageCommand({
      QueueUrl: this.queueUrl,
      MessageBody: JSON.stringify({
        type: 'delayed',
        task,
        remainingDelay
      }),
      DelaySeconds: firstDelay
    }));
  }
}
```

## SNS（发布/订阅）

### 核心概念

**Topic（主题）**：
- Standard Topic：无序，至少一次传递
- FIFO Topic：有序，精确一次传递

**Subscription（订阅）**：
- HTTP/HTTPS
- Email/Email-JSON
- SQS
- Lambda
- SMS
- Mobile Push

### 发布消息

```typescript
import { SNSClient, PublishCommand, CreateTopicCommand } from '@aws-sdk/client-sns';

class SNSPublisher {
  private client: SNSClient;
  private topicArn: string;
  
  constructor(topicArn: string) {
    this.client = new SNSClient({ region: 'us-east-1' });
    this.topicArn = topicArn;
  }
  
  // 发布消息
  async publish(message: any, options?: {
    subject?: string;
    messageGroupId?: string;
    deduplicationId?: string;
  }) {
    const command = new PublishCommand({
      TopicArn: this.topicArn,
      Message: JSON.stringify(message),
      Subject: options?.subject,
      
      // FIFO Topic 必需
      MessageGroupId: options?.messageGroupId,
      MessageDeduplicationId: options?.deduplicationId,
      
      // 消息属性（用于过滤）
      MessageAttributes: {
        'event_type': {
          DataType: 'String',
          StringValue: message.type
        },
        'priority': {
          DataType: 'Number',
          StringValue: message.priority?.toString() || '0'
        }
      }
    });
    
    const response = await this.client.send(command);
    return response.MessageId;
  }
  
  // 批量发布
  async publishBatch(messages: any[]) {
    const { PublishBatchCommand } = await import('@aws-sdk/client-sns');
    
    const batches = this.chunk(messages, 10);
    const results = [];
    
    for (const batch of batches) {
      const command = new PublishBatchCommand({
        TopicArn: this.topicArn,
        PublishBatchRequestEntries: batch.map((msg, index) => ({
          Id: index.toString(),
          Message: JSON.stringify(msg),
          MessageAttributes: {
            'timestamp': {
              DataType: 'Number',
              StringValue: Date.now().toString()
            }
          }
        }))
      });
      
      const response = await this.client.send(command);
      results.push(response);
    }
    
    return results;
  }
  
  private chunk<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}
```

### 订阅管理

```typescript
import { SubscribeCommand, SetSubscriptionAttributesCommand } from '@aws-sdk/client-sns';

async function subscribeToTopic(topicArn: string, queueArn: string) {
  const client = new SNSClient({ region: 'us-east-1' });
  
  // SQS 订阅
  const subscribeResponse = await client.send(new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'sqs',
    Endpoint: queueArn,
    
    // 返回订阅确认消息
    ReturnSubscriptionArn: true
  }));
  
  const subscriptionArn = subscribeResponse.SubscriptionArn!;
  
  // 设置过滤策略
  await client.send(new SetSubscriptionAttributesCommand({
    SubscriptionArn: subscriptionArn,
    AttributeName: 'FilterPolicy',
    AttributeValue: JSON.stringify({
      // 只接收特定事件类型
      event_type: ['order.created', 'order.updated'],
      
      // 只接收高优先级
      priority: [{ numeric: ['>=', 5] }]
    })
  }));
  
  // 设置重试策略
  await client.send(new SetSubscriptionAttributesCommand({
    SubscriptionArn: subscriptionArn,
    AttributeName: 'RedrivePolicy',
    AttributeValue: JSON.stringify({
      deadLetterTargetArn: dlqArn
    })
  }));
  
  return subscriptionArn;
}

// Lambda 订阅
async function subscribeLambda(topicArn: string, lambdaArn: string) {
  const client = new SNSClient({ region: 'us-east-1' });
  
  await client.send(new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'lambda',
    Endpoint: lambdaArn
  }));
}
```

### 消息过滤

```typescript
// 订阅时设置过滤策略
const filterPolicy = {
  // 字符串匹配
  event_type: ['user.created', 'user.updated'],
  
  // 数字比较
  price: [{ numeric: ['>', 100] }],
  
  // 存在性检查
  vip: [{ exists: true }],
  
  // 前缀匹配
  store: [{ prefix: 'us-' }],
  
  // 范围匹配
  age: [{ numeric: ['>=', 18, '<=', 65] }]
};

await client.send(new SetSubscriptionAttributesCommand({
  SubscriptionArn: subscriptionArn,
  AttributeName: 'FilterPolicy',
  AttributeValue: JSON.stringify(filterPolicy)
}));

// 发布消息时设置属性
await client.send(new PublishCommand({
  TopicArn: topicArn,
  Message: JSON.stringify({ userId: '123' }),
  MessageAttributes: {
    'event_type': {
      DataType: 'String',
      StringValue: 'user.created'
    },
    'price': {
      DataType: 'Number',
      StringValue: '150'
    }
  }
}));
```

## SNS + SQS 架构模式

### 1. Fanout 模式

```typescript
// 一个消息发送到多个队列
async function setupFanout() {
  const snsClient = new SNSClient({ region: 'us-east-1' });
  const sqsClient = new SQSClient({ region: 'us-east-1' });
  
  // 创建 SNS Topic
  const topicResponse = await snsClient.send(new CreateTopicCommand({
    Name: 'order-events'
  }));
  const topicArn = topicResponse.TopicArn!;
  
  // 创建多个 SQS 队列
  const queues = ['email-queue', 'sms-queue', 'analytics-queue'];
  
  for (const queueName of queues) {
    const queueResponse = await sqsClient.send(new CreateQueueCommand({
      QueueName: queueName
    }));
    const queueUrl = queueResponse.QueueUrl!;
    
    // 获取队列 ARN
    const attrsResponse = await sqsClient.send(new GetQueueAttributesCommand({
      QueueUrl: queueUrl,
      AttributeNames: ['QueueArn']
    }));
    const queueArn = attrsResponse.Attributes!['QueueArn'];
    
    // 设置队列策略（允许 SNS 发送消息）
    await sqsClient.send(new SetQueueAttributesCommand({
      QueueUrl: queueUrl,
      Attributes: {
        'Policy': JSON.stringify({
          Version: '2012-10-17',
          Statement: [
            {
              Effect: 'Allow',
              Principal: { Service: 'sns.amazonaws.com' },
              Action: 'sqs:SendMessage',
              Resource: queueArn,
              Condition: {
                ArnEquals: {
                  'aws:SourceArn': topicArn
                }
              }
            }
          ]
        })
      }
    }));
    
    // 订阅 Topic
    await snsClient.send(new SubscribeCommand({
      TopicArn: topicArn,
      Protocol: 'sqs',
      Endpoint: queueArn
    }));
  }
  
  return topicArn;
}

// 发布消息
const publisher = new SNSPublisher(topicArn);
await publisher.publish({
  orderId: '123',
  amount: 100
});

// 消息会自动发送到所有订阅的队列
```

### 2. 消息过滤分发

```typescript
async function setupFilteredDistribution() {
  const topicArn = await createTopic('events');
  
  // 队列 1：只接收高优先级订单
  const highPriorityQueue = await createQueue('high-priority-orders');
  await subscribeWithFilter(topicArn, highPriorityQueue, {
    event_type: ['order.created'],
    priority: [{ numeric: ['>=', 8] }]
  });
  
  // 队列 2：只接收低优先级订单
  const lowPriorityQueue = await createQueue('low-priority-orders');
  await subscribeWithFilter(topicArn, lowPriorityQueue, {
    event_type: ['order.created'],
    priority: [{ numeric: ['<', 8] }]
  });
  
  // 队列 3：接收所有用户事件
  const userEventsQueue = await createQueue('user-events');
  await subscribeWithFilter(topicArn, userEventsQueue, {
    event_type: [{ prefix: 'user.' }]
  });
}

async function subscribeWithFilter(
  topicArn: string,
  queueArn: string,
  filter: any
) {
  const client = new SNSClient({ region: 'us-east-1' });
  
  const response = await client.send(new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: 'sqs',
    Endpoint: queueArn,
    ReturnSubscriptionArn: true
  }));
  
  await client.send(new SetSubscriptionAttributesCommand({
    SubscriptionArn: response.SubscriptionArn!,
    AttributeName: 'FilterPolicy',
    AttributeValue: JSON.stringify(filter)
  }));
}
```

## 性能优化

### 1. 批量操作

```typescript
class OptimizedSQS {
  private client: SQSClient;
  private queueUrl: string;
  private sendBuffer: any[] = [];
  private deleteBuffer: string[] = [];
  private flushInterval = 1000;
  
  constructor(queueUrl: string) {
    this.client = new SQSClient({ region: 'us-east-1' });
    this.queueUrl = queueUrl;
    this.startFlushTimer();
  }
  
  // 缓冲发送
  async send(message: any) {
    this.sendBuffer.push(message);
    
    if (this.sendBuffer.length >= 10) {
      await this.flushSend();
    }
  }
  
  // 缓冲删除
  async delete(receiptHandle: string) {
    this.deleteBuffer.push(receiptHandle);
    
    if (this.deleteBuffer.length >= 10) {
      await this.flushDelete();
    }
  }
  
  private async flushSend() {
    if (this.sendBuffer.length === 0) return;
    
    const { SendMessageBatchCommand } = await import('@aws-sdk/client-sqs');
    const messages = this.sendBuffer.splice(0, 10);
    
    await this.client.send(new SendMessageBatchCommand({
      QueueUrl: this.queueUrl,
      Entries: messages.map((msg, i) => ({
        Id: i.toString(),
        MessageBody: JSON.stringify(msg)
      }))
    }));
  }
  
  private async flushDelete() {
    if (this.deleteBuffer.length === 0) return;
    
    const { DeleteMessageBatchCommand } = await import('@aws-sdk/client-sqs');
    const handles = this.deleteBuffer.splice(0, 10);
    
    await this.client.send(new DeleteMessageBatchCommand({
      QueueUrl: this.queueUrl,
      Entries: handles.map((handle, i) => ({
        Id: i.toString(),
        ReceiptHandle: handle
      }))
    }));
  }
  
  private startFlushTimer() {
    setInterval(() => {
      this.flushSend().catch(console.error);
      this.flushDelete().catch(console.error);
    }, this.flushInterval);
  }
}
```

### 2. 长轮询 vs 短轮询

```typescript
// ✅ 推荐：长轮询（节省成本，降低延迟）
await client.send(new ReceiveMessageCommand({
  QueueUrl: queueUrl,
  WaitTimeSeconds: 20,  // 最大 20 秒
  MaxNumberOfMessages: 10
}));

// ❌ 避免：短轮询（频繁请求，成本高）
await client.send(new ReceiveMessageCommand({
  QueueUrl: queueUrl,
  WaitTimeSeconds: 0,  // 立即返回
  MaxNumberOfMessages: 10
}));

// 队列级别启用长轮询
await client.send(new SetQueueAttributesCommand({
  QueueUrl: queueUrl,
  Attributes: {
    'ReceiveMessageWaitTimeSeconds': '20'
  }
}));
```

## 监控和告警

```typescript
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

class SQSMonitor {
  private cloudwatch: CloudWatchClient;
  
  constructor() {
    this.cloudwatch = new CloudWatchClient({ region: 'us-east-1' });
  }
  
  async reportMetrics(queueUrl: string) {
    const sqsClient = new SQSClient({ region: 'us-east-1' });
    
    // 获取队列属性
    const response = await sqsClient.send(new GetQueueAttributesCommand({
      QueueUrl: queueUrl,
      AttributeNames: [
        'ApproximateNumberOfMessages',
        'ApproximateNumberOfMessagesNotVisible',
        'ApproximateNumberOfMessagesDelayed'
      ]
    }));
    
    const attrs = response.Attributes!;
    
    // 发送到 CloudWatch
    await this.cloudwatch.send(new PutMetricDataCommand({
      Namespace: 'CustomMetrics/SQS',
      MetricData: [
        {
          MetricName: 'QueueDepth',
          Value: parseInt(attrs['ApproximateNumberOfMessages']),
          Unit: 'Count',
          Timestamp: new Date()
        },
        {
          MetricName: 'MessagesInFlight',
          Value: parseInt(attrs['ApproximateNumberOfMessagesNotVisible']),
          Unit: 'Count',
          Timestamp: new Date()
        }
      ]
    }));
  }
}
```

## 常见面试题

### 1. SQS Standard vs FIFO 如何选择？

<details>
<summary>点击查看答案</summary>

**Standard Queue**：
- ✅ 无限吞吐量
- ✅ 至少一次传递
- ❌ 消息可能乱序
- ❌ 可能重复

**FIFO Queue**：
- ✅ 严格顺序
- ✅ 精确一次传递
- ❌ 吞吐量限制（3000 TPS）
- ❌ 仅某些区域可用

**选择建议**：
- 需要严格顺序 → FIFO
- 高吞吐量场景 → Standard
- 幂等性处理 → Standard 足够
- 金融交易、订单处理 → FIFO
</details>

### 2. 如何处理消息重复？

<details>
<summary>点击查看答案</summary>

**方法 1：使用 FIFO Queue + 去重 ID**
```typescript
await client.send(new SendMessageCommand({
  QueueUrl: fifoQueueUrl,
  MessageBody: JSON.stringify(message),
  MessageGroupId: 'group-1',
  MessageDeduplicationId: message.id  // 5 分钟内去重
}));
```

**方法 2：应用层幂等性处理**
```typescript
const processedIds = new Set<string>();

async function processMessage(message: any) {
  if (processedIds.has(message.id)) {
    return;  // 已处理，跳过
  }
  
  // 处理逻辑
  await handleMessage(message);
  
  processedIds.add(message.id);
}
```

**方法 3：数据库唯一约束**
```typescript
try {
  await db.collection('orders').insertOne({
    _id: message.orderId,  // 唯一 ID
    ...message
  });
} catch (error) {
  if (error.code === 11000) {
    // 重复消息，忽略
    return;
  }
  throw error;
}
```
</details>

### 3. 可见性超时如何设置？

<details>
<summary>点击查看答案</summary>

**原则**：
- 设置为消息处理时间的 6 倍
- 最小：0 秒
- 最大：12 小时
- 默认：30 秒

**示例**：
```typescript
// 处理时间约 10 秒，设置 60 秒超时
await client.send(new ReceiveMessageCommand({
  QueueUrl: queueUrl,
  VisibilityTimeout: 60
}));

// 动态延长超时
async function processLongTask(message: any, receiptHandle: string) {
  const interval = setInterval(async () => {
    // 每 30 秒延长一次
    await client.send(new ChangeMessageVisibilityCommand({
      QueueUrl: queueUrl,
      ReceiptHandle: receiptHandle,
      VisibilityTimeout: 60
    }));
  }, 30000);
  
  try {
    await handleTask(message);
  } finally {
    clearInterval(interval);
  }
}
```
</details>

### 4. SNS vs SQS 的使用场景？

<details>
<summary>点击查看答案</summary>

**SQS（队列）**：
- 异步任务处理
- 解耦服务
- 削峰填谷
- 一对一通信

**SNS（发布/订阅）**：
- 事件通知
- 一对多通信
- 实时推送
- 移动推送

**SNS + SQS**：
- Fanout 架构
- 消息过滤分发
- 解耦 + 持久化
- 灵活路由

**示例场景**：
- 订单创建 → SNS（通知多个服务）→ SQS（持久化队列）
- 邮件发送 → 直接 SQS
- 用户注册 → SNS（通知邮件/SMS/分析）
</details>

## 最佳实践

### 1. 错误处理

```typescript
class RobustConsumer {
  private maxRetries = 3;
  
  async processWithRetry(message: any, receiptHandle: string) {
    const retryCount = this.getRetryCount(message);
    
    try {
      await this.process(message);
      await this.deleteMessage(receiptHandle);
    } catch (error) {
      if (retryCount < this.maxRetries) {
        // 指数退避
        const delay = Math.pow(2, retryCount) * 30;
        await this.changeVisibility(receiptHandle, delay);
      } else {
        // 发送到 DLQ（自动）或记录
        await this.logFailure(message, error);
      }
    }
  }
  
  private getRetryCount(message: any): number {
    return parseInt(
      message.Attributes?.ApproximateReceiveCount || '0'
    );
  }
}
```

### 2. 成本优化

- 使用长轮询（减少空请求）
- 批量操作（减少 API 调用）
- 合理设置消息保留时间
- 使用 SNS 过滤（减少不必要的消息）

### 3. 安全

```typescript
// IAM 策略示例
const queuePolicy = {
  Version: '2012-10-17',
  Statement: [
    {
      Effect: 'Allow',
      Principal: {
        AWS: 'arn:aws:iam::123456789:role/MyRole'
      },
      Action: [
        'sqs:SendMessage',
        'sqs:ReceiveMessage',
        'sqs:DeleteMessage'
      ],
      Resource: queueArn
    }
  ]
};

// 使用 VPC 端点（节省流量成本）
```

