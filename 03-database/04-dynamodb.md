# DynamoDB

Amazon DynamoDB 是一个完全托管的 NoSQL 数据库服务，提供快速且可预测的性能以及无缝的可扩展性。

## 核心概念

### 主键设计

DynamoDB 的性能高度依赖于主键设计。

```typescript
// 1. 简单主键（Partition Key）
{
  TableName: 'Users',
  KeySchema: [
    { AttributeName: 'userId', KeyType: 'HASH' }  // Partition Key
  ]
}

// 2. 复合主键（Partition Key + Sort Key）
{
  TableName: 'Orders',
  KeySchema: [
    { AttributeName: 'userId', KeyType: 'HASH' },    // Partition Key
    { AttributeName: 'orderId', KeyType: 'RANGE' }   // Sort Key
  ]
}

// 访问模式设计
// ✅ 好：通过主键访问
await dynamodb.getItem({
  TableName: 'Orders',
  Key: {
    userId: { S: 'user123' },
    orderId: { S: 'order456' }
  }
});

// ✅ 好：查询同一分区的多个项目
await dynamodb.query({
  TableName: 'Orders',
  KeyConditionExpression: 'userId = :userId AND orderId BETWEEN :start AND :end',
  ExpressionAttributeValues: {
    ':userId': { S: 'user123' },
    ':start': { S: 'order100' },
    ':end': { S: 'order200' }
  }
});

// ❌ 不好：扫描整个表
await dynamodb.scan({
  TableName: 'Orders',
  FilterExpression: 'status = :status',  // 性能差
  ExpressionAttributeValues: {
    ':status': { S: 'pending' }
  }
});
```

### 分区键选择原则

```typescript
// ❌ 不好：低基数的分区键
{
  partitionKey: 'status'  // 只有几个值，数据倾斜
}

// ✅ 好：高基数的分区键
{
  partitionKey: 'userId'  // 均匀分布
}

// ✅ 好：组合键避免热分区
{
  partitionKey: `${userId}#${date}`  // 更细粒度
}

// 实战：时序数据的分区键设计
// ❌ 不好：所有数据在一个分区
{
  partitionKey: 'sensor-1',
  sortKey: timestamp
}

// ✅ 好：按时间窗口分区
{
  partitionKey: 'sensor-1#2024-01',  // 月度分区
  sortKey: timestamp
}

// ✅ 更好：使用写分片
{
  partitionKey: `sensor-1#${timestamp % 10}`,  // 10 个分片
  sortKey: timestamp
}
```

## 二级索引（GSI & LSI）

### Global Secondary Index (GSI)

```typescript
// 创建 GSI
const params = {
  TableName: 'Users',
  AttributeDefinitions: [
    { AttributeName: 'userId', AttributeType: 'S' },
    { AttributeName: 'email', AttributeType: 'S' },
    { AttributeName: 'country', AttributeType: 'S' },
    { AttributeName: 'createdAt', AttributeType: 'N' }
  ],
  GlobalSecondaryIndexes: [
    {
      IndexName: 'EmailIndex',
      KeySchema: [
        { AttributeName: 'email', KeyType: 'HASH' }
      ],
      Projection: { ProjectionType: 'ALL' },
      ProvisionedThroughput: {
        ReadCapacityUnits: 5,
        WriteCapacityUnits: 5
      }
    },
    {
      IndexName: 'CountryCreatedIndex',
      KeySchema: [
        { AttributeName: 'country', KeyType: 'HASH' },
        { AttributeName: 'createdAt', KeyType: 'RANGE' }
      ],
      Projection: {
        ProjectionType: 'INCLUDE',
        NonKeyAttributes: ['username', 'email']
      }
    }
  ]
};

// 使用 GSI 查询
await dynamodb.query({
  TableName: 'Users',
  IndexName: 'EmailIndex',
  KeyConditionExpression: 'email = :email',
  ExpressionAttributeValues: {
    ':email': { S: 'user@example.com' }
  }
});

// 查询特定国家的最新用户
await dynamodb.query({
  TableName: 'Users',
  IndexName: 'CountryCreatedIndex',
  KeyConditionExpression: 'country = :country',
  ExpressionAttributeValues: {
    ':country': { S: 'USA' }
  },
  ScanIndexForward: false,  // 降序
  Limit: 10
});
```

### Local Secondary Index (LSI)

```typescript
// LSI 必须在创建表时定义，不能后续添加
const params = {
  TableName: 'Orders',
  KeySchema: [
    { AttributeName: 'userId', KeyType: 'HASH' },
    { AttributeName: 'orderId', KeyType: 'RANGE' }
  ],
  LocalSecondaryIndexes: [
    {
      IndexName: 'UserOrderDateIndex',
      KeySchema: [
        { AttributeName: 'userId', KeyType: 'HASH' },
        { AttributeName: 'orderDate', KeyType: 'RANGE' }  // 替代 Sort Key
      ],
      Projection: { ProjectionType: 'ALL' }
    }
  ]
};

// 使用 LSI 按日期查询用户的订单
await dynamodb.query({
  TableName: 'Orders',
  IndexName: 'UserOrderDateIndex',
  KeyConditionExpression: 'userId = :userId AND orderDate BETWEEN :start AND :end',
  ExpressionAttributeValues: {
    ':userId': { S: 'user123' },
    ':start': { S: '2024-01-01' },
    ':end': { S: '2024-12-31' }
  }
});
```

### GSI vs LSI 对比

| 特性 | GSI | LSI |
|------|-----|-----|
| 分区键 | 可以不同 | 必须相同 |
| 排序键 | 可以任意 | 必须不同 |
| 创建时机 | 任何时候 | 只能创建表时 |
| 数量限制 | 20 个 | 5 个 |
| 容量 | 独立的 | 共享表的 |
| 一致性 | 最终一致 | 强一致或最终一致 |

## 高级查询模式

### 1. 单表设计（Single Table Design）

```typescript
// 核心思想：一个表存储多个实体类型
// 使用 PK/SK 模式

// User 实体
{
  PK: 'USER#alice',
  SK: 'METADATA',
  entityType: 'User',
  username: 'alice',
  email: 'alice@example.com'
}

// User 的 Profile
{
  PK: 'USER#alice',
  SK: 'PROFILE',
  entityType: 'Profile',
  bio: 'Software Engineer',
  location: 'NYC'
}

// Post 实体
{
  PK: 'POST#123',
  SK: 'METADATA',
  entityType: 'Post',
  title: 'My Post',
  authorId: 'USER#alice'
}

// Post 的评论
{
  PK: 'POST#123',
  SK: 'COMMENT#001',
  entityType: 'Comment',
  userId: 'USER#bob',
  content: 'Great post!'
}

// User 发布的 Posts（GSI）
{
  GSI1PK: 'USER#alice',
  GSI1SK: 'POST#123',
  ...
}

// 查询用户的所有相关数据
await dynamodb.query({
  KeyConditionExpression: 'PK = :pk',
  ExpressionAttributeValues: {
    ':pk': { S: 'USER#alice' }
  }
});

// 查询用户发布的所有帖子（使用 GSI）
await dynamodb.query({
  IndexName: 'GSI1',
  KeyConditionExpression: 'GSI1PK = :pk AND begins_with(GSI1SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': { S: 'USER#alice' },
    ':sk': { S: 'POST#' }
  }
});
```

### 2. 分层数据建模

```typescript
// 组织架构：公司 -> 部门 -> 员工
// PK: 组织层级  SK: 实体标识

{
  PK: 'ORG#company1',
  SK: 'METADATA',
  name: 'Tech Corp'
}

{
  PK: 'ORG#company1',
  SK: 'DEPT#eng',
  name: 'Engineering'
}

{
  PK: 'ORG#company1',
  SK: 'DEPT#eng#EMP#alice',
  name: 'Alice',
  role: 'Engineer'
}

// 查询公司的所有部门
await dynamodb.query({
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': { S: 'ORG#company1' },
    ':sk': { S: 'DEPT#' }
  }
});

// 查询工程部的所有员工
await dynamodb.query({
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': { S: 'ORG#company1' },
    ':sk': { S: 'DEPT#eng#EMP#' }
  }
});
```

### 3. 邻接列表模式

```typescript
// 社交网络：关注关系
// PK: 用户ID  SK: 关注的用户ID

{
  PK: 'USER#alice',
  SK: 'USER#bob',
  relationshipType: 'FOLLOWS',
  createdAt: timestamp
}

{
  PK: 'USER#alice',
  SK: 'USER#charlie',
  relationshipType: 'FOLLOWS',
  createdAt: timestamp
}

// 查询 Alice 关注的所有人
await dynamodb.query({
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :sk)',
  ExpressionAttributeValues: {
    ':pk': { S: 'USER#alice' },
    ':sk': { S: 'USER#' }
  }
});

// 使用 GSI 查询谁关注了 Bob
// GSI: SK 作为分区键
await dynamodb.query({
  IndexName: 'InverseIndex',
  KeyConditionExpression: 'SK = :sk AND begins_with(PK, :pk)',
  ExpressionAttributeValues: {
    ':sk': { S: 'USER#bob' },
    ':pk': { S: 'USER#' }
  }
});
```

## 事务（Transactions）

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, TransactWriteCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

// 事务写入（最多 25 个操作）
const command = new TransactWriteCommand({
  TransactItems: [
    {
      // 扣款
      Update: {
        TableName: 'Accounts',
        Key: { userId: 'user123', accountId: 'acc1' },
        UpdateExpression: 'SET balance = balance - :amount',
        ConditionExpression: 'balance >= :amount',  // 条件检查
        ExpressionAttributeValues: {
          ':amount': 100
        }
      }
    },
    {
      // 加款
      Update: {
        TableName: 'Accounts',
        Key: { userId: 'user456', accountId: 'acc2' },
        UpdateExpression: 'SET balance = balance + :amount',
        ExpressionAttributeValues: {
          ':amount': 100
        }
      }
    },
    {
      // 记录交易
      Put: {
        TableName: 'Transactions',
        Item: {
          transactionId: 'txn789',
          from: 'user123',
          to: 'user456',
          amount: 100,
          timestamp: Date.now()
        }
      }
    }
  ]
});

try {
  await docClient.send(command);
  console.log('Transaction successful');
} catch (error) {
  if (error.name === 'TransactionCanceledException') {
    // 事务被取消（通常是条件检查失败）
    console.error('Transaction failed:', error.CancellationReasons);
  }
  throw error;
}

// 事务读取
import { TransactGetCommand } from '@aws-sdk/lib-dynamodb';

const readCommand = new TransactGetCommand({
  TransactItems: [
    {
      Get: {
        TableName: 'Users',
        Key: { userId: 'user123' }
      }
    },
    {
      Get: {
        TableName: 'Accounts',
        Key: { userId: 'user123', accountId: 'acc1' }
      }
    }
  ]
});

const response = await docClient.send(readCommand);
const [user, account] = response.Responses;
```

## 条件表达式和更新表达式

```typescript
// 条件写入（乐观锁）
await docClient.send(new UpdateCommand({
  TableName: 'Items',
  Key: { itemId: 'item123' },
  UpdateExpression: 'SET #stock = :newStock, #version = :newVersion',
  ConditionExpression: '#version = :currentVersion AND #stock >= :minStock',
  ExpressionAttributeNames: {
    '#stock': 'stock',
    '#version': 'version'
  },
  ExpressionAttributeValues: {
    ':newStock': 95,
    ':newVersion': 2,
    ':currentVersion': 1,
    ':minStock': 0
  }
}));

// 复杂的更新表达式
await docClient.send(new UpdateCommand({
  TableName: 'Users',
  Key: { userId: 'user123' },
  UpdateExpression: `
    SET #loginCount = #loginCount + :inc,
        #lastLogin = :now,
        #profile.#status = :status
    ADD #tags :newTags
    REMOVE #tempData
  `,
  ExpressionAttributeNames: {
    '#loginCount': 'loginCount',
    '#lastLogin': 'lastLogin',
    '#profile': 'profile',
    '#status': 'status',
    '#tags': 'tags',
    '#tempData': 'tempData'
  },
  ExpressionAttributeValues: {
    ':inc': 1,
    ':now': Date.now(),
    ':status': 'active',
    ':newTags': new Set(['premium'])
  },
  ReturnValues: 'ALL_NEW'
}));

// 原子计数器
await docClient.send(new UpdateCommand({
  TableName: 'PageViews',
  Key: { pageId: 'page123' },
  UpdateExpression: 'ADD viewCount :inc',
  ExpressionAttributeValues: {
    ':inc': 1
  }
}));

// 列表操作
await docClient.send(new UpdateCommand({
  TableName: 'Users',
  Key: { userId: 'user123' },
  UpdateExpression: 'SET #history = list_append(#history, :newItem)',
  ExpressionAttributeNames: {
    '#history': 'loginHistory'
  },
  ExpressionAttributeValues: {
    ':newItem': [{ timestamp: Date.now(), ip: '1.2.3.4' }]
  }
}));
```

## DynamoDB Streams

```typescript
// 启用 Stream
const params = {
  TableName: 'Orders',
  StreamSpecification: {
    StreamEnabled: true,
    StreamViewType: 'NEW_AND_OLD_IMAGES'  // 新旧值都包含
  }
};

// StreamViewType 选项：
// - KEYS_ONLY: 只有主键
// - NEW_IMAGE: 只有新值
// - OLD_IMAGE: 只有旧值
// - NEW_AND_OLD_IMAGES: 新旧值都有

// 使用 Lambda 处理 Stream 事件
export const handler = async (event) => {
  for (const record of event.Records) {
    console.log('Event:', record.eventName);  // INSERT, MODIFY, REMOVE
    
    if (record.eventName === 'INSERT') {
      const newImage = record.dynamodb.NewImage;
      // 新订单通知
      await notifyNewOrder(newImage);
    }
    
    if (record.eventName === 'MODIFY') {
      const oldImage = record.dynamodb.OldImage;
      const newImage = record.dynamodb.NewImage;
      
      // 状态变更
      if (oldImage.status.S !== newImage.status.S) {
        await handleStatusChange(oldImage, newImage);
      }
    }
  }
};

// 实战：缓存失效
export const handleCacheInvalidation = async (event) => {
  for (const record of event.Records) {
    if (record.eventName === 'MODIFY' || record.eventName === 'REMOVE') {
      const keys = record.dynamodb.Keys;
      const cacheKey = `user:${keys.userId.S}`;
      
      await redis.del(cacheKey);
    }
  }
};

// 实战：数据复制
export const replicateToElasticsearch = async (event) => {
  const operations = event.Records.map(record => {
    const { eventName, dynamodb } = record;
    const id = dynamodb.Keys.id.S;
    
    if (eventName === 'INSERT' || eventName === 'MODIFY') {
      return {
        index: {
          _index: 'products',
          _id: id,
          _source: unmarshall(dynamodb.NewImage)
        }
      };
    } else if (eventName === 'REMOVE') {
      return {
        delete: {
          _index: 'products',
          _id: id
        }
      };
    }
  });
  
  await elasticsearch.bulk({ body: operations });
};
```

## 性能优化

### 批量操作

```typescript
// BatchGet - 一次获取多个项目（最多 100 个）
import { BatchGetCommand } from '@aws-sdk/lib-dynamodb';

const command = new BatchGetCommand({
  RequestItems: {
    'Users': {
      Keys: [
        { userId: 'user1' },
        { userId: 'user2' },
        { userId: 'user3' }
      ]
    },
    'Orders': {
      Keys: [
        { userId: 'user1', orderId: 'order1' },
        { userId: 'user2', orderId: 'order2' }
      ]
    }
  }
});

const response = await docClient.send(command);
const users = response.Responses.Users;
const orders = response.Responses.Orders;

// BatchWrite - 批量写入/删除（最多 25 个）
import { BatchWriteCommand } from '@aws-sdk/lib-dynamodb';

const writeCommand = new BatchWriteCommand({
  RequestItems: {
    'Users': [
      {
        PutRequest: {
          Item: { userId: 'user1', name: 'Alice' }
        }
      },
      {
        DeleteRequest: {
          Key: { userId: 'user2' }
        }
      }
    ]
  }
});

await docClient.send(writeCommand);

// 处理未处理的项目
if (response.UnprocessedItems && Object.keys(response.UnprocessedItems).length > 0) {
  // 重试未处理的项目
  await docClient.send(new BatchWriteCommand({
    RequestItems: response.UnprocessedItems
  }));
}
```

### 分页

```typescript
// 使用 ExclusiveStartKey 分页
async function queryAllItems(userId: string) {
  const items = [];
  let lastEvaluatedKey = null;
  
  do {
    const params: any = {
      TableName: 'Orders',
      KeyConditionExpression: 'userId = :userId',
      ExpressionAttributeValues: {
        ':userId': userId
      },
      Limit: 100
    };
    
    if (lastEvaluatedKey) {
      params.ExclusiveStartKey = lastEvaluatedKey;
    }
    
    const response = await docClient.send(new QueryCommand(params));
    items.push(...response.Items);
    lastEvaluatedKey = response.LastEvaluatedKey;
    
  } while (lastEvaluatedKey);
  
  return items;
}

// 游标分页（API 返回）
async function paginatedQuery(userId: string, cursor?: string, limit = 20) {
  const params: any = {
    TableName: 'Orders',
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: {
      ':userId': userId
    },
    Limit: limit
  };
  
  if (cursor) {
    params.ExclusiveStartKey = JSON.parse(Buffer.from(cursor, 'base64').toString());
  }
  
  const response = await docClient.send(new QueryCommand(params));
  
  return {
    items: response.Items,
    nextCursor: response.LastEvaluatedKey 
      ? Buffer.from(JSON.stringify(response.LastEvaluatedKey)).toString('base64')
      : null
  };
}
```

### PartiQL（SQL-like 查询）

```typescript
import { ExecuteStatementCommand } from '@aws-sdk/client-dynamodb';

// 使用 PartiQL 查询
const command = new ExecuteStatementCommand({
  Statement: 'SELECT * FROM Users WHERE userId = ?',
  Parameters: [{ S: 'user123' }]
});

// 事务性 PartiQL
const transactionCommand = new ExecuteStatementCommand({
  Statement: `
    UPDATE Accounts
    SET balance = balance - ?
    WHERE userId = ? AND accountId = ?
  `,
  Parameters: [
    { N: '100' },
    { S: 'user123' },
    { S: 'acc1' }
  ]
});

// 批量 PartiQL
import { BatchExecuteStatementCommand } from '@aws-sdk/client-dynamodb';

const batchCommand = new BatchExecuteStatementCommand({
  Statements: [
    {
      Statement: 'INSERT INTO Users VALUE {\'userId\': ?, \'name\': ?}',
      Parameters: [{ S: 'user1' }, { S: 'Alice' }]
    },
    {
      Statement: 'INSERT INTO Users VALUE {\'userId\': ?, \'name\': ?}',
      Parameters: [{ S: 'user2' }, { S: 'Bob' }]
    }
  ]
});
```

## 容量模式

### 按需模式 vs 预置模式

```typescript
// 按需模式（On-Demand）
// - 按实际请求付费
// - 自动扩展
// - 适合不可预测的工作负载

const onDemandTable = {
  TableName: 'Users',
  BillingMode: 'PAY_PER_REQUEST',
  // ...
};

// 预置模式（Provisioned）
// - 预先设置容量
// - 更便宜（稳定负载）
// - 需要配置自动扩展

const provisionedTable = {
  TableName: 'Users',
  BillingMode: 'PROVISIONED',
  ProvisionedThroughput: {
    ReadCapacityUnits: 5,
    WriteCapacityUnits: 5
  },
  // ...
};

// 自动扩展配置
const autoScaling = {
  ServiceNamespace: 'dynamodb',
  ResourceId: 'table/Users',
  ScalableDimension: 'dynamodb:table:ReadCapacityUnits',
  MinCapacity: 5,
  MaxCapacity: 100,
  TargetTrackingScalingPolicyConfiguration: {
    TargetValue: 70.0,  // 目标利用率 70%
    PredefinedMetricSpecification: {
      PredefinedMetricType: 'DynamoDBReadCapacityUtilization'
    }
  }
};
```

## 常见面试题

### 1. DynamoDB 的主键设计原则？

<details>
<summary>点击查看答案</summary>

**核心原则**：

1. **高基数**：分区键应该有大量不同的值
2. **均匀分布**：避免热分区
3. **访问模式驱动**：根据查询需求设计

**常见模式**：

```typescript
// 模式 1：简单主键
{ userId: 'user123' }  // 适合简单查询

// 模式 2：复合主键
{ 
  userId: 'user123',     // Partition Key
  orderId: 'order456'    // Sort Key
}

// 模式 3：组合键避免热分区
{
  partitionKey: `${userId}#${date}`,  // 时间分片
  sortKey: timestamp
}

// 模式 4：写分片
{
  partitionKey: `item-${itemId % 10}`,  // 10 个分片
  sortKey: timestamp
}
```

**反模式**：
- 使用低基数的分区键（status, type）
- 使用单调递增的分区键（timestamp）
- 不考虑访问模式的设计
</details>

### 2. 何时使用 GSI vs LSI？

<details>
<summary>点击查看答案</summary>

**使用 GSI**：
- 需要不同的分区键
- 表创建后需要添加索引
- 需要更多灵活性
- 可以接受最终一致性

**使用 LSI**：
- 需要强一致性读取
- 分区键相同，只改变排序键
- 必须在创建表时定义
- 数量限制（5 个）

**示例**：

```typescript
// GSI: 按邮箱查询用户
{
  GSI_PK: 'email@example.com',  // 不同的分区键
  GSI_SK: 'USER#123'
}

// LSI: 按不同维度排序同一用户的订单
{
  PK: 'USER#123',
  SK: 'ORDER#456'
}

{
  PK: 'USER#123',
  LSI_SK: '2024-01-15'  // 按日期排序
}
```
</details>

### 3. 单表设计的优缺点？

<details>
<summary>点击查看答案</summary>

**优点**：
1. 减少连接操作
2. 原子性操作更容易
3. 成本更低（一个表的费用）
4. 更好的性能（减少网络调用）

**缺点**：
1. 设计复杂
2. 难以理解和维护
3. 需要预先了解所有访问模式
4. 备份和恢复更复杂

**何时使用**：
- 访问模式明确且稳定
- 需要高性能
- 团队有 DynamoDB 经验

**何时避免**：
- 访问模式频繁变化
- 团队不熟悉 NoSQL
- 需要临时查询
</details>

### 4. 如何处理热分区问题？

<details>
<summary>点击查看答案</summary>

**检测热分区**：
- 使用 CloudWatch 监控 `UserErrors`
- 分析 `ConsumedReadCapacityUnits` 和 `ConsumedWriteCapacityUnits`

**解决方案**：

**1. 写分片**
```typescript
// 添加随机后缀
const suffix = Math.floor(Math.random() * 10);
const partitionKey = `${baseKey}#${suffix}`;
```

**2. 数据分散**
```typescript
// 时间分片
const partitionKey = `${baseKey}#${YYYYMM}`;
```

**3. 缓存热点数据**
```typescript
// 使用 DAX 或 ElastiCache
const cachedData = await cache.get(key);
if (cachedData) return cachedData;
```

**4. 使用 GSI 分散读取**
```typescript
// 创建多个 GSI 分散流量
```
</details>

### 5. DynamoDB 事务的限制？

<details>
<summary>点击查看答案</summary>

**限制**：
1. 最多 25 个操作
2. 总大小不超过 4MB
3. 不能跨区域
4. 不能包含 Scan 操作
5. 成本是普通操作的 2 倍

**使用场景**：
- 需要原子性的多项操作
- 条件检查很重要
- 可以接受额外成本

**最佳实践**：
```typescript
// ✅ 好：使用事务保证一致性
await docClient.send(new TransactWriteCommand({
  TransactItems: [
    { Update: { /* 扣库存 */ } },
    { Put: { /* 创建订单 */ } }
  ]
}));

// ❌ 不好：不必要的事务
await docClient.send(new TransactWriteCommand({
  TransactItems: [
    { Put: { /* 单个操作 */ } }
  ]
}));
```
</details>

## 最佳实践

1. **访问模式优先**：先设计访问模式，再设计表结构
2. **避免 Scan**：尽量使用 Query
3. **使用批量操作**：BatchGet/BatchWrite 提高效率
4. **监控和告警**：CloudWatch 监控容量和错误
5. **使用 DAX**：缓存热点数据
6. **定期备份**：使用按需备份或时间点恢复
7. **成本优化**：选择合适的容量模式
8. **使用 PartiQL**：简化查询语法（适度使用）

