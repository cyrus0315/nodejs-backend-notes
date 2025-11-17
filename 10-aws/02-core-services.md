# AWS 核心服务

涵盖 Node.js 后端开发中最常用的 AWS 服务。

## IAM（身份与访问管理）

### 核心概念

```typescript
// IAM 由四个核心组件组成：
// 1. 用户（Users）：个人或应用
// 2. 组（Groups）：用户集合
// 3. 角色（Roles）：可被服务或用户临时担任
// 4. 策略（Policies）：权限定义（JSON）
```

### 策略示例

```json
// 允许访问特定 S3 bucket
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}

// Lambda 执行角色
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Users"
    }
  ]
}
```

### Node.js 中使用 IAM

```typescript
import { IAMClient, ListUsersCommand } from '@aws-sdk/client-iam';

const iamClient = new IAMClient({ region: 'us-east-1' });

// 列出用户
const users = await iamClient.send(new ListUsersCommand({}));
console.log(users.Users);

// 使用临时凭证（STS）
import { STSClient, AssumeRoleCommand } from '@aws-sdk/client-sts';

const stsClient = new STSClient({});

const assumeRole = await stsClient.send(new AssumeRoleCommand({
  RoleArn: 'arn:aws:iam::123456789:role/MyRole',
  RoleSessionName: 'session1',
  DurationSeconds: 3600
}));

const credentials = assumeRole.Credentials;
// 使用临时凭证创建其他服务客户端
```

## S3（Simple Storage Service）

### 基础操作

```typescript
import { 
  S3Client, 
  PutObjectCommand, 
  GetObjectCommand,
  DeleteObjectCommand,
  ListObjectsV2Command 
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({ region: 'us-east-1' });

// 上传文件
export async function uploadFile(bucket: string, key: string, body: Buffer) {
  await s3Client.send(new PutObjectCommand({
    Bucket: bucket,
    Key: key,
    Body: body,
    ContentType: 'image/jpeg',
    Metadata: {
      uploadedBy: 'user123',
      uploadDate: new Date().toISOString()
    }
  }));
}

// 下载文件
export async function downloadFile(bucket: string, key: string) {
  const response = await s3Client.send(new GetObjectCommand({
    Bucket: bucket,
    Key: key
  }));
  
  // 转换为 Buffer
  const chunks: Uint8Array[] = [];
  for await (const chunk of response.Body as any) {
    chunks.push(chunk);
  }
  
  return Buffer.concat(chunks);
}

// 删除文件
export async function deleteFile(bucket: string, key: string) {
  await s3Client.send(new DeleteObjectCommand({
    Bucket: bucket,
    Key: key
  }));
}

// 列出文件
export async function listFiles(bucket: string, prefix?: string) {
  const response = await s3Client.send(new ListObjectsV2Command({
    Bucket: bucket,
    Prefix: prefix,
    MaxKeys: 100
  }));
  
  return response.Contents || [];
}
```

### 预签名 URL

```typescript
// 生成上传 URL（允许直接从浏览器上传）
export async function generateUploadUrl(
  bucket: string,
  key: string,
  expiresIn: number = 3600
) {
  const command = new PutObjectCommand({
    Bucket: bucket,
    Key: key
  });
  
  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

// 生成下载 URL
export async function generateDownloadUrl(
  bucket: string,
  key: string,
  expiresIn: number = 3600
) {
  const command = new GetObjectCommand({
    Bucket: bucket,
    Key: key
  });
  
  const url = await getSignedUrl(s3Client, command, { expiresIn });
  return url;
}

// 使用示例
const uploadUrl = await generateUploadUrl('my-bucket', 'user-uploads/photo.jpg');

// 前端直接使用这个 URL 上传
await fetch(uploadUrl, {
  method: 'PUT',
  body: fileBlob,
  headers: {
    'Content-Type': 'image/jpeg'
  }
});
```

### S3 事件通知

```typescript
// Lambda 处理 S3 事件
import { S3Event } from 'aws-lambda';
import sharp from 'sharp';

export const handler = async (event: S3Event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    
    // 下载原始图片
    const original = await s3Client.send(new GetObjectCommand({
      Bucket: bucket,
      Key: key
    }));
    
    // 生成缩略图
    const thumbnail = await sharp(await original.Body!.transformToByteArray())
      .resize(200, 200)
      .toBuffer();
    
    // 上传缩略图
    const thumbnailKey = key.replace('originals/', 'thumbnails/');
    await s3Client.send(new PutObjectCommand({
      Bucket: bucket,
      Key: thumbnailKey,
      Body: thumbnail,
      ContentType: 'image/jpeg'
    }));
  }
};
```

## RDS（Relational Database Service）

### 连接 PostgreSQL

```typescript
import { Client } from 'pg';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

// 从 Secrets Manager 获取数据库凭证
async function getDbCredentials() {
  const secretsClient = new SecretsManagerClient({});
  
  const response = await secretsClient.send(
    new GetSecretValueCommand({ SecretId: 'db-credentials' })
  );
  
  return JSON.parse(response.SecretString!);
}

// 创建数据库连接
export async function createDbConnection() {
  const credentials = await getDbCredentials();
  
  const client = new Client({
    host: credentials.host,
    port: credentials.port,
    database: credentials.database,
    user: credentials.username,
    password: credentials.password,
    ssl: {
      rejectUnauthorized: true
    }
  });
  
  await client.connect();
  return client;
}

// 使用连接池
import { Pool } from 'pg';

let pool: Pool | null = null;

export function getDbPool() {
  if (!pool) {
    pool = new Pool({
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 20,
      idleTimeoutMillis: 30000,
      connectionTimeoutMillis: 2000,
      ssl: { rejectUnauthorized: true }
    });
  }
  
  return pool;
}

// Lambda 中使用（复用连接）
const pool = getDbPool();

export const handler = async (event: any) => {
  const client = await pool.connect();
  
  try {
    const result = await client.query('SELECT * FROM users WHERE id = $1', [event.userId]);
    return result.rows[0];
  } finally {
    client.release();
  }
};
```

## ElastiCache（Redis/Memcached）

### 使用 Redis

```typescript
import { createClient } from 'redis';

let redisClient: ReturnType<typeof createClient> | null = null;

export async function getRedisClient() {
  if (!redisClient) {
    redisClient = createClient({
      socket: {
        host: process.env.REDIS_HOST,
        port: parseInt(process.env.REDIS_PORT || '6379'),
        tls: true  // ElastiCache 加密传输
      }
    });
    
    await redisClient.connect();
    
    redisClient.on('error', (err) => {
      console.error('Redis error:', err);
    });
  }
  
  return redisClient;
}

// 缓存模式
export async function getCachedData(key: string, fetchFn: () => Promise<any>) {
  const redis = await getRedisClient();
  
  // 尝试从缓存获取
  const cached = await redis.get(key);
  if (cached) {
    return JSON.parse(cached);
  }
  
  // 缓存未命中，获取数据
  const data = await fetchFn();
  
  // 写入缓存（10分钟过期）
  await redis.setEx(key, 600, JSON.stringify(data));
  
  return data;
}

// 在 Lambda 中使用
export const handler = async (event: any) => {
  const userId = event.pathParameters.userId;
  
  const user = await getCachedData(
    `user:${userId}`,
    () => fetchUserFromDb(userId)
  );
  
  return {
    statusCode: 200,
    body: JSON.stringify(user)
  };
};
```

## CloudWatch

### 日志记录

```typescript
import { CloudWatchLogsClient, PutLogEventsCommand } from '@aws-sdk/client-cloudwatch-logs';

const logsClient = new CloudWatchLogsClient({});

// 结构化日志
export function logStructured(level: string, message: string, meta?: any) {
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    message,
    ...meta
  }));
}

// Lambda 自动发送到 CloudWatch Logs
export const handler = async (event: any) => {
  logStructured('INFO', 'Processing request', {
    requestId: event.requestContext.requestId,
    userId: event.userId
  });
  
  try {
    const result = await processRequest(event);
    
    logStructured('INFO', 'Request processed successfully', {
      requestId: event.requestContext.requestId,
      duration: Date.now()
    });
    
    return result;
  } catch (error) {
    logStructured('ERROR', 'Request failed', {
      error: error.message,
      stack: error.stack,
      event
    });
    
    throw error;
  }
};
```

### 自定义指标

```typescript
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cloudwatch = new CloudWatchClient({});

export async function recordMetric(
  metricName: string,
  value: number,
  unit: string = 'Count'
) {
  await cloudwatch.send(new PutMetricDataCommand({
    Namespace: 'MyApp',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
      Dimensions: [
        {
          Name: 'Environment',
          Value: process.env.STAGE || 'dev'
        }
      ]
    }]
  }));
}

// 使用
await recordMetric('UserRegistration', 1);
await recordMetric('APILatency', duration, 'Milliseconds');
```

## Secrets Manager

### 管理密钥

```typescript
import { 
  SecretsManagerClient, 
  GetSecretValueCommand,
  CreateSecretCommand,
  UpdateSecretCommand 
} from '@aws-sdk/client-secrets-manager';

const secretsClient = new SecretsManagerClient({});

// 获取密钥
export async function getSecret(secretName: string) {
  const response = await secretsClient.send(
    new GetSecretValueCommand({ SecretId: secretName })
  );
  
  return JSON.parse(response.SecretString!);
}

// 创建密钥
export async function createSecret(name: string, value: any) {
  await secretsClient.send(new CreateSecretCommand({
    Name: name,
    SecretString: JSON.stringify(value),
    Description: `Secret for ${name}`
  }));
}

// 更新密钥
export async function updateSecret(name: string, value: any) {
  await secretsClient.send(new UpdateSecretCommand({
    SecretId: name,
    SecretString: JSON.stringify(value)
  }));
}

// 缓存密钥（Lambda 复用）
let cachedSecrets: Map<string, { value: any; expiry: number }> = new Map();

export async function getCachedSecret(secretName: string, ttl: number = 300000) {
  const now = Date.now();
  const cached = cachedSecrets.get(secretName);
  
  if (cached && cached.expiry > now) {
    return cached.value;
  }
  
  const secret = await getSecret(secretName);
  cachedSecrets.set(secretName, {
    value: secret,
    expiry: now + ttl
  });
  
  return secret;
}
```

## EventBridge

### 发布和订阅事件

```typescript
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const eventBridge = new EventBridgeClient({});

// 发布事件
export async function publishEvent(eventType: string, detail: any) {
  await eventBridge.send(new PutEventsCommand({
    Entries: [{
      Source: 'my-app',
      DetailType: eventType,
      Detail: JSON.stringify(detail),
      EventBusName: 'default'
    }]
  }));
}

// 使用示例
await publishEvent('user.registered', {
  userId: '123',
  email: 'user@example.com',
  timestamp: Date.now()
});

await publishEvent('order.placed', {
  orderId: '456',
  userId: '123',
  total: 99.99,
  items: [...]
});

// 事件处理器（Lambda）
export const userRegisteredHandler = async (event: any) => {
  const { userId, email } = event.detail;
  
  // 发送欢迎邮件
  await sendWelcomeEmail(email);
  
  // 创建用户资源
  await createUserResources(userId);
};

export const orderPlacedHandler = async (event: any) => {
  const { orderId, userId } = event.detail;
  
  // 更新库存
  await updateInventory(orderId);
  
  // 发送确认邮件
  await sendOrderConfirmation(userId, orderId);
};
```

## Step Functions

### 状态机工作流

```typescript
// 订单处理工作流
{
  "Comment": "Order processing workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:validateOrder",
      "Next": "ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["ValidationError"],
        "Next": "OrderFailed"
      }]
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:processPayment",
      "Next": "UpdateInventory",
      "Retry": [{
        "ErrorEquals": ["PaymentError"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2
      }],
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "RefundPayment"
      }]
    },
    "UpdateInventory": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:updateInventory",
      "Next": "SendConfirmation"
    },
    "SendConfirmation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:sendConfirmation",
      "End": true
    },
    "RefundPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789:function:refundPayment",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Error": "OrderProcessingFailed",
      "Cause": "Order could not be processed"
    }
  }
}

// 启动状态机
import { SFNClient, StartExecutionCommand } from '@aws-sdk/client-sfn';

const sfnClient = new SFNClient({});

export async function startWorkflow(orderId: string, orderData: any) {
  const response = await sfnClient.send(new StartExecutionCommand({
    stateMachineArn: 'arn:aws:states:us-east-1:123456789:stateMachine:OrderProcessing',
    input: JSON.stringify({
      orderId,
      ...orderData
    }),
    name: `order-${orderId}-${Date.now()}`
  }));
  
  return response.executionArn;
}
```

## 最佳实践

1. **IAM**：最小权限原则
2. **S3**：使用生命周期策略管理成本
3. **RDS**：使用连接池，启用自动备份
4. **ElastiCache**：设置合适的 TTL
5. **CloudWatch**：结构化日志，设置告警
6. **Secrets**：定期轮换，使用 Secrets Manager
7. **EventBridge**：解耦服务，事件驱动架构
8. **Step Functions**：复杂工作流编排

## 常见面试题

### 1. IAM 角色 vs IAM 用户？

<details>
<summary>点击查看答案</summary>

**IAM 用户**：
- 长期凭证（Access Key/Secret）
- 用于人或应用
- 需要管理凭证

**IAM 角色**：
- 临时凭证（自动轮换）
- 可被服务或用户担任
- 更安全（无长期凭证）

**使用场景**：
- EC2/Lambda 访问其他服务 → 角色
- 跨账户访问 → 角色
- CI/CD 部署 → 角色（OIDC）
- 个人开发 → 用户
</details>

### 2. S3 预签名 URL 的原理？

<details>
<summary>点击查看答案</summary>

**原理**：
1. 使用 AWS 凭证对请求签名
2. 生成包含签名的临时 URL
3. URL 在过期时间内有效
4. 客户端可直接使用 URL 访问 S3

**优势**：
- 无需暴露 AWS 凭证
- 控制访问时间
- 减轻服务器负载

**安全注意**：
- 设置合理的过期时间
- 限制可访问的对象
- 使用 HTTPS
</details>

### 3. RDS vs DynamoDB 如何选择？

<details>
<summary>点击查看答案</summary>

**RDS（关系型）**：
- ✅ 复杂查询（JOIN）
- ✅ 事务支持
- ✅ 数据一致性
- ❌ 扩展性有限
- ❌ 需要管理实例

**DynamoDB（NoSQL）**：
- ✅ 水平扩展
- ✅ 高性能
- ✅ Serverless
- ❌ 查询灵活性低
- ❌ 学习曲线

**选择**：
- 复杂关系数据 → RDS
- 高并发KV存储 → DynamoDB
- 事务要求高 → RDS
- Serverless架构 → DynamoDB
</details>

## 总结

AWS 服务生态丰富，关键是：
1. 理解各服务的适用场景
2. 合理设计架构
3. 注意安全和成本
4. 充分利用托管服务
5. 做好监控和日志

