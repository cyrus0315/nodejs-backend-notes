# AWS Lambda 与 Serverless

AWS Lambda 是无服务器计算服务，让你无需管理服务器即可运行代码。

## Serverless 概念

### 什么是 Serverless

**定义**：
- 无需管理服务器基础设施
- 按需自动扩展
- 按实际使用付费
- 事件驱动

**优势**：
- ✅ 零服务器管理
- ✅ 自动扩展
- ✅ 成本效益（按执行时间付费）
- ✅ 高可用性（AWS 管理）
- ✅ 快速部署

**劣势**：
- ❌ 冷启动延迟
- ❌ 执行时间限制（15分钟）
- ❌ 供应商锁定
- ❌ 调试困难
- ❌ 有状态应用不适合

## Lambda 基础

### 创建 Lambda 函数

```typescript
// handler.ts
import { APIGatewayProxyEvent, APIGatewayProxyResult, Context } from 'aws-lambda';

export const handler = async (
  event: APIGatewayProxyEvent,
  context: Context
): Promise<APIGatewayProxyResult> => {
  console.log('Event:', JSON.stringify(event, null, 2));
  console.log('Context:', JSON.stringify(context, null, 2));
  
  return {
    statusCode: 200,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      message: 'Hello from Lambda!',
      requestId: context.requestId
    })
  };
};
```

### 使用 AWS SDK v3

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, GetCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event: any) => {
  const { action, data } = JSON.parse(event.body);
  
  if (action === 'create') {
    await docClient.send(new PutCommand({
      TableName: 'Users',
      Item: data
    }));
    
    return {
      statusCode: 201,
      body: JSON.stringify({ message: 'Created' })
    };
  }
  
  if (action === 'get') {
    const result = await docClient.send(new GetCommand({
      TableName: 'Users',
      Key: { id: data.id }
    }));
    
    return {
      statusCode: 200,
      body: JSON.stringify(result.Item)
    };
  }
};
```

## 触发器（Triggers）

### 1. API Gateway

```typescript
// API Gateway HTTP 事件
export const apiHandler = async (event: APIGatewayProxyEvent) => {
  const { httpMethod, path, queryStringParameters, body } = event;
  
  if (httpMethod === 'GET' && path === '/users') {
    // 获取用户列表
    const users = await getUsers();
    return {
      statusCode: 200,
      body: JSON.stringify(users)
    };
  }
  
  if (httpMethod === 'POST' && path === '/users') {
    const userData = JSON.parse(body || '{}');
    const user = await createUser(userData);
    return {
      statusCode: 201,
      body: JSON.stringify(user)
    };
  }
  
  return {
    statusCode: 404,
    body: JSON.stringify({ error: 'Not found' })
  };
};
```

### 2. SQS 队列

```typescript
import { SQSEvent, SQSRecord } from 'aws-lambda';

export const sqsHandler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    try {
      const message = JSON.parse(record.body);
      console.log('Processing message:', message);
      
      // 处理消息
      await processMessage(message);
      
      // 消息处理成功，自动从队列删除
    } catch (error) {
      console.error('Error processing message:', error);
      // 抛出错误，消息会重新进入队列
      throw error;
    }
  }
};

async function processMessage(message: any) {
  // 业务逻辑
}
```

### 3. S3 事件

```typescript
import { S3Event } from 'aws-lambda';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';

const s3Client = new S3Client({});

export const s3Handler = async (event: S3Event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    
    console.log(`File uploaded: s3://${bucket}/${key}`);
    
    // 获取文件内容
    const response = await s3Client.send(new GetObjectCommand({
      Bucket: bucket,
      Key: key
    }));
    
    const content = await response.Body?.transformToString();
    
    // 处理文件
    await processFile(content);
  }
};
```

### 4. DynamoDB Streams

```typescript
import { DynamoDBStreamEvent } from 'aws-lambda';
import { unmarshall } from '@aws-sdk/util-dynamodb';

export const dynamoStreamHandler = async (event: DynamoDBStreamEvent) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT') {
      const newItem = unmarshall(record.dynamodb!.NewImage as any);
      console.log('New item:', newItem);
      
      // 发送欢迎邮件
      await sendWelcomeEmail(newItem.email);
    }
    
    if (record.eventName === 'MODIFY') {
      const oldItem = unmarshall(record.dynamodb!.OldImage as any);
      const newItem = unmarshall(record.dynamodb!.NewImage as any);
      
      // 处理更新
      await handleUpdate(oldItem, newItem);
    }
    
    if (record.eventName === 'REMOVE') {
      const oldItem = unmarshall(record.dynamodb!.OldImage as any);
      console.log('Deleted item:', oldItem);
      
      // 清理相关资源
      await cleanup(oldItem.id);
    }
  }
};
```

### 5. EventBridge (CloudWatch Events)

```typescript
import { ScheduledEvent } from 'aws-lambda';

// 定时任务
export const scheduledHandler = async (event: ScheduledEvent) => {
  console.log('Scheduled event:', event);
  
  // 每天执行的任务
  await dailyCleanup();
  
  return {
    statusCode: 200,
    body: JSON.stringify({ message: 'Cleanup completed' })
  };
};

// 自定义事件
export const customEventHandler = async (event: any) => {
  const { detailType, detail } = event;
  
  if (detailType === 'user.registered') {
    await handleUserRegistered(detail);
  }
  
  if (detailType === 'order.placed') {
    await handleOrderPlaced(detail);
  }
};
```

## 环境变量和配置

```typescript
// 读取环境变量
const TABLE_NAME = process.env.TABLE_NAME;
const API_KEY = process.env.API_KEY;
const REGION = process.env.AWS_REGION;

export const handler = async (event: any) => {
  // 使用环境变量
  console.log(`Using table: ${TABLE_NAME}`);
  
  // 从 Secrets Manager 获取敏感信息
  const secret = await getSecret('my-api-key');
  
  return {
    statusCode: 200,
    body: JSON.stringify({ message: 'Success' })
  };
};

// Secrets Manager
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsClient = new SecretsManagerClient({});

async function getSecret(secretName: string): Promise<string> {
  const response = await secretsClient.send(
    new GetSecretValueCommand({ SecretId: secretName })
  );
  
  return response.SecretString!;
}
```

## Lambda Layers

```typescript
// layer/nodejs/utils.ts
export function formatResponse(data: any, statusCode = 200) {
  return {
    statusCode,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify(data)
  };
}

export function parseBody(body: string | null) {
  try {
    return JSON.parse(body || '{}');
  } catch {
    return {};
  }
}

// Lambda 函数中使用 Layer
import { formatResponse, parseBody } from '/opt/nodejs/utils';

export const handler = async (event: any) => {
  const data = parseBody(event.body);
  
  // 处理数据
  const result = await processData(data);
  
  // 使用 Layer 中的工具函数
  return formatResponse(result);
};
```

## 冷启动优化

### 1. 减少包大小

```typescript
// ❌ 导入整个 SDK
import AWS from 'aws-sdk';

// ✅ 只导入需要的客户端
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

// ❌ 导入大型库
import moment from 'moment';

// ✅ 使用轻量级替代
import dayjs from 'dayjs';
// 或使用原生 Date API
```

### 2. 预初始化连接

```typescript
// ❌ 每次请求都创建连接
export const handler = async (event: any) => {
  const client = new DynamoDBClient({});
  // ...
};

// ✅ 在 handler 外部初始化（复用）
const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

export const handler = async (event: any) => {
  // 直接使用已初始化的客户端
  await docClient.send(new PutCommand({...}));
};
```

### 3. 使用 Provisioned Concurrency

```typescript
// serverless.yml
functions:
  api:
    handler: handler.main
    provisionedConcurrency: 5  // 预留5个实例
```

### 4. 选择合适的运行时

- Node.js 18.x（ARM64）比 x86_64 启动快 20%
- 使用较新的 Node.js 版本

### 5. 优化依赖

```bash
# 使用 esbuild 打包
npm install --save-dev esbuild

# package.json
{
  "scripts": {
    "build": "esbuild handler.ts --bundle --minify --platform=node --target=node18 --outfile=dist/handler.js"
  }
}
```

## 并发控制

```typescript
// serverless.yml
functions:
  api:
    handler: handler.main
    reservedConcurrency: 100  // 最大并发数
    
  # 或设置预留并发
  critical:
    handler: handler.critical
    provisionedConcurrency: 10  // 预留实例
    reservedConcurrency: 50     // 最大并发
```

## 错误处理与重试

```typescript
export const handler = async (event: any) => {
  try {
    // 处理逻辑
    const result = await processEvent(event);
    return formatSuccess(result);
    
  } catch (error) {
    console.error('Error:', error);
    
    // 可重试的错误
    if (isRetryableError(error)) {
      // Lambda 会自动重试
      throw error;
    }
    
    // 不可重试的错误
    // 发送到 DLQ
    await sendToDLQ(event, error);
    
    return formatError(error);
  }
};

function isRetryableError(error: any): boolean {
  // 网络错误、超时等可重试
  return error.code === 'NetworkingError' ||
         error.code === 'TimeoutError';
}

// DLQ 处理器
export const dlqHandler = async (event: any) => {
  console.error('Processing DLQ message:', event);
  
  // 记录失败信息
  await logFailure(event);
  
  // 发送告警
  await sendAlert('Lambda execution failed', event);
};
```

## 监控和日志

```typescript
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cloudwatch = new CloudWatchClient({});

export const handler = async (event: any) => {
  const startTime = Date.now();
  
  try {
    // 处理逻辑
    const result = await processEvent(event);
    
    // 记录成功指标
    await recordMetric('ProcessingSuccess', 1);
    await recordMetric('ProcessingDuration', Date.now() - startTime);
    
    return formatSuccess(result);
    
  } catch (error) {
    // 记录失败指标
    await recordMetric('ProcessingFailure', 1);
    
    // 结构化日志
    console.error(JSON.stringify({
      level: 'ERROR',
      message: error.message,
      stack: error.stack,
      event,
      timestamp: new Date().toISOString()
    }));
    
    throw error;
  }
};

async function recordMetric(name: string, value: number) {
  await cloudwatch.send(new PutMetricDataCommand({
    Namespace: 'MyApp',
    MetricData: [{
      MetricName: name,
      Value: value,
      Unit: 'Count',
      Timestamp: new Date()
    }]
  }));
}
```

## 与其他 AWS 服务集成

### Lambda + SQS 批量处理

```typescript
import { SQSEvent, SQSBatchResponse } from 'aws-lambda';

export const handler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const batchItemFailures = [];
  
  for (const record of event.Records) {
    try {
      await processMessage(record);
    } catch (error) {
      console.error('Failed to process:', record.messageId);
      batchItemFailures.push({
        itemIdentifier: record.messageId
      });
    }
  }
  
  // 返回失败的消息，Lambda 会重试
  return { batchItemFailures };
};
```

### Lambda + Step Functions

```typescript
// Step Function 状态机调用的 Lambda
export const validateOrder = async (event: any) => {
  const { orderId, items } = event;
  
  // 验证订单
  const isValid = await checkInventory(items);
  
  return {
    orderId,
    isValid,
    items,
    timestamp: Date.now()
  };
};

export const processPayment = async (event: any) => {
  const { orderId, amount } = event;
  
  // 处理支付
  const paymentResult = await chargeCustomer(amount);
  
  return {
    ...event,
    paymentId: paymentResult.id,
    status: paymentResult.status
  };
};
```

## Serverless Framework 配置

```yaml
# serverless.yml
service: my-app

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  memorySize: 512
  timeout: 30
  environment:
    TABLE_NAME: ${self:custom.tableName}
    STAGE: ${self:provider.stage}
  
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:PutItem
            - dynamodb:GetItem
          Resource: 
            - arn:aws:dynamodb:${self:provider.region}:*:table/${self:custom.tableName}

functions:
  api:
    handler: src/handlers/api.handler
    events:
      - httpApi:
          path: /users
          method: get
      - httpApi:
          path: /users
          method: post
    layers:
      - arn:aws:lambda:us-east-1:123456789:layer:utils:1
  
  processQueue:
    handler: src/handlers/queue.handler
    events:
      - sqs:
          arn:
            Fn::GetAtt:
              - MyQueue
              - Arn
          batchSize: 10
          maximumBatchingWindowInSeconds: 5
  
  scheduled:
    handler: src/handlers/cron.handler
    events:
      - schedule: rate(1 hour)

resources:
  Resources:
    MyQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-queue
        VisibilityTimeout: 300

custom:
  tableName: ${self:service}-table-${self:provider.stage}

plugins:
  - serverless-esbuild
  - serverless-offline
```

## 最佳实践

1. **保持函数小而专注**：单一职责
2. **使用环境变量**：配置管理
3. **优化冷启动**：减小包大小，预初始化连接
4. **错误处理**：区分可重试和不可重试错误
5. **监控和告警**：CloudWatch Logs 和 Metrics
6. **安全**：最小权限原则，使用 Secrets Manager
7. **测试**：本地测试（serverless-offline）
8. **版本和别名**：安全部署
9. **成本优化**：选择合适的内存大小
10. **文档**：清晰的函数说明

## 常见面试题

### 1. Lambda 冷启动如何优化？

<details>
<summary>点击查看答案</summary>

**优化策略**：

1. **减小部署包大小**：
   - 使用 esbuild/webpack 打包
   - 只导入需要的依赖
   - 使用 tree-shaking

2. **选择合适的运行时**：
   - 使用 ARM64 架构（Graviton2）
   - 选择较新的 Node.js 版本

3. **使用 Provisioned Concurrency**：
   - 预留实例，消除冷启动
   - 成本较高，适用于关键路径

4. **优化代码**：
   - 在 handler 外部初始化连接
   - 延迟加载非必需依赖
   - 使用 Lambda Layers

5. **配置优化**：
   - 增加内存（CPU 也会增加）
   - 使用 VPC 时配置 NAT Gateway

**典型冷启动时间**：
- Node.js 18：~200-300ms
- Node.js 18 (ARM64)：~150-250ms
- 使用 Provisioned Concurrency：0ms
</details>

### 2. Lambda 的限制有哪些？

<details>
<summary>点击查看答案</summary>

**关键限制**：

1. **执行时间**：最长 15 分钟
2. **内存**：128MB - 10,240MB
3. **部署包大小**：
   - 直接上传：50MB（压缩）
   - 使用 S3：250MB（解压）
4. **环境变量**：4KB
5. **并发**：账户级别 1000（可申请提升）
6. **临时存储**（/tmp）：512MB - 10GB
7. **请求/响应大小**：6MB（同步）

**解决方案**：
- 长时间任务 → Step Functions
- 大文件处理 → S3 直接处理
- 高并发 → 申请提升限额或使用队列
</details>

### 3. Lambda 如何保证幂等性？

<details>
<summary>点击查看答案</summary>

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const handler = async (event: any) => {
  const requestId = event.requestContext?.requestId;
  
  // 1. 检查是否已处理
  const existing = await docClient.send(new GetCommand({
    TableName: 'ProcessedRequests',
    Key: { requestId }
  }));
  
  if (existing.Item) {
    console.log('Request already processed:', requestId);
    return existing.Item.result;
  }
  
  // 2. 处理请求
  const result = await processRequest(event);
  
  // 3. 记录结果
  await docClient.send(new PutCommand({
    TableName: 'ProcessedRequests',
    Item: {
      requestId,
      result,
      processedAt: Date.now(),
      ttl: Math.floor(Date.now() / 1000) + 86400  // 24小时后过期
    }
  }));
  
  return result;
};
```

**关键点**：
- 使用唯一 ID 去重
- DynamoDB 条件写入
- 设置 TTL 自动清理
</details>

## 总结

Lambda 是构建 Serverless 应用的核心，关键是：
1. 理解冷启动和优化策略
2. 合理设计函数粒度
3. 充分利用 AWS 服务集成
4. 做好监控和错误处理
5. 注意成本优化

