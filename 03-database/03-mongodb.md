# MongoDB

MongoDB 是一个基于文档的 NoSQL 数据库，以灵活的 JSON-like 文档存储著称。

## 核心概念

### 文档模型 vs 关系模型

```javascript
// 关系型数据库（需要 JOIN）
// users 表
{ id: 1, name: "Alice", email: "alice@example.com" }

// posts 表
{ id: 1, user_id: 1, title: "Post 1" }
{ id: 2, user_id: 1, title: "Post 2" }

// MongoDB（嵌套文档）
{
  _id: ObjectId("..."),
  name: "Alice",
  email: "alice@example.com",
  posts: [
    { title: "Post 1", createdAt: ISODate("...") },
    { title: "Post 2", createdAt: ISODate("...") }
  ]
}

// 或使用引用（类似 JOIN）
{
  _id: ObjectId("..."),
  name: "Alice",
  postIds: [ObjectId("..."), ObjectId("...")]
}
```

**何时嵌套 vs 引用？**

嵌套：
- 数据总是一起访问
- 数据量较小
- 一对少量（1-100）

引用：
- 数据独立访问
- 数据量大
- 多对多关系

## 索引策略

### 复合索引和索引优化

```javascript
// 创建复合索引
db.users.createIndex({ city: 1, age: -1 });

// ESR 规则：Equality, Sort, Range
db.users.createIndex({ 
  status: 1,      // Equality（精确匹配）
  createdAt: -1,  // Sort（排序）
  age: 1          // Range（范围查询）
});

// 部分索引（减小索引大小）
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
);

// 稀疏索引
db.users.createIndex(
  { phone: 1 },
  { sparse: true }  // 只索引存在 phone 字段的文档
);

// 文本索引（全文搜索）
db.articles.createIndex({ 
  title: "text", 
  content: "text" 
});

// 搜索
db.articles.find({ 
  $text: { $search: "mongodb database" } 
});

// 地理空间索引
db.locations.createIndex({ location: "2dsphere" });

db.locations.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.9, 40.7] },
      $maxDistance: 5000  // 5km
    }
  }
});

// 通配符索引（灵活但谨慎使用）
db.products.createIndex({ "specs.$**": 1 });

// 可以查询任意嵌套字段
db.products.find({ "specs.color": "red" });
```

### 索引性能分析

```javascript
// 查看查询计划
db.users.find({ city: "NYC" }).explain("executionStats");

// 关注这些指标：
// - totalDocsExamined: 扫描的文档数
// - totalKeysExamined: 扫描的索引键数
// - executionTimeMillis: 执行时间
// - indexName: 使用的索引

// 理想情况：totalDocsExamined ≈ nReturned

// 查看索引使用情况
db.users.aggregate([
  { $indexStats: {} }
]);

// 删除未使用的索引
db.users.dropIndex("city_1_age_-1");
```

## 聚合框架（Aggregation Pipeline）

### 高级聚合操作

```javascript
// 复杂的数据分析管道
db.orders.aggregate([
  // 1. 匹配条件
  {
    $match: {
      status: "completed",
      createdAt: { $gte: new Date("2024-01-01") }
    }
  },
  
  // 2. 关联查询（$lookup = JOIN）
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  
  // 3. 解构数组
  { $unwind: "$user" },
  { $unwind: "$items" },
  
  // 4. 分组统计
  {
    $group: {
      _id: {
        userId: "$userId",
        month: { $month: "$createdAt" }
      },
      totalAmount: { $sum: "$items.price" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$items.price" }
    }
  },
  
  // 5. 排序
  { $sort: { totalAmount: -1 } },
  
  // 6. 限制结果
  { $limit: 10 },
  
  // 7. 投影（选择字段）
  {
    $project: {
      _id: 0,
      userId: "$_id.userId",
      month: "$_id.month",
      totalAmount: 1,
      orderCount: 1
    }
  }
]);

// 实战：计算用户留存率
db.events.aggregate([
  {
    $match: {
      eventType: "login",
      timestamp: { $gte: new Date("2024-01-01") }
    }
  },
  {
    $group: {
      _id: {
        userId: "$userId",
        date: { $dateToString: { format: "%Y-%m-%d", date: "$timestamp" } }
      }
    }
  },
  {
    $group: {
      _id: "$_id.userId",
      loginDays: { $addToSet: "$_id.date" }
    }
  },
  {
    $project: {
      userId: "$_id",
      dayCount: { $size: "$loginDays" }
    }
  }
]);

// 使用 $facet 进行多维度分析
db.products.aggregate([
  {
    $facet: {
      // 价格分布
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 200, 500],
            default: "500+",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      // 分类统计
      categories: [
        {
          $group: {
            _id: "$category",
            count: { $sum: 1 },
            avgPrice: { $avg: "$price" }
          }
        }
      ],
      // 总体统计
      overall: [
        {
          $group: {
            _id: null,
            total: { $sum: 1 },
            avgPrice: { $avg: "$price" },
            minPrice: { $min: "$price" },
            maxPrice: { $max: "$price" }
          }
        }
      ]
    }
  }
]);
```

## 事务（Transactions）

```javascript
// MongoDB 4.0+ 支持多文档 ACID 事务
const session = client.startSession();

try {
  await session.withTransaction(async () => {
    // 扣款
    await db.collection("accounts").updateOne(
      { _id: fromAccountId },
      { $inc: { balance: -amount } },
      { session }
    );
    
    // 加款
    await db.collection("accounts").updateOne(
      { _id: toAccountId },
      { $inc: { balance: amount } },
      { session }
    );
    
    // 记录交易
    await db.collection("transactions").insertOne(
      {
        from: fromAccountId,
        to: toAccountId,
        amount,
        timestamp: new Date()
      },
      { session }
    );
  });
} finally {
  await session.endSession();
}

// 注意：事务有性能开销，能用原子操作就用原子操作
// ✅ 好：使用原子操作
db.users.updateOne(
  { _id: userId },
  { 
    $inc: { loginCount: 1 },
    $set: { lastLogin: new Date() }
  }
);

// ❌ 不好：不必要的事务
```

## 数据建模模式

### 1. 嵌套文档模式

```javascript
// 适用于一对少量关系
{
  _id: ObjectId("..."),
  name: "John Doe",
  addresses: [
    {
      type: "home",
      street: "123 Main St",
      city: "NYC"
    },
    {
      type: "work",
      street: "456 Office Rd",
      city: "NYC"
    }
  ]
}

// 限制：
// - 单个文档最大 16MB
// - 数组不应该无限增长
```

### 2. 引用模式

```javascript
// 适用于多对多、大量数据
// Users
{ _id: ObjectId("user1"), name: "Alice" }

// Posts
{
  _id: ObjectId("post1"),
  userId: ObjectId("user1"),  // 引用
  title: "My Post"
}

// 查询时需要 $lookup
db.posts.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "author"
    }
  }
]);
```

### 3. 桶模式（Bucket Pattern）

```javascript
// 适用于时序数据、IoT 数据
// ❌ 不好：每条数据一个文档
{ sensorId: 1, timestamp: ISODate("..."), value: 25.3 }
{ sensorId: 1, timestamp: ISODate("..."), value: 25.4 }

// ✅ 好：按时间段分桶
{
  sensorId: 1,
  date: ISODate("2024-01-01"),
  measurements: [
    { time: "00:00", value: 25.3 },
    { time: "00:01", value: 25.4 },
    // ... 1440 条（一天的数据）
  ],
  avgValue: 25.5,
  maxValue: 30.0,
  minValue: 20.0
}

// 优势：
// - 减少文档数量
// - 提高查询效率
// - 预计算统计数据
```

### 4. 扩展引用模式

```javascript
// 在引用的同时存储常用字段
{
  _id: ObjectId("post1"),
  userId: ObjectId("user1"),
  // 存储常用的用户信息，避免每次 JOIN
  userInfo: {
    name: "Alice",
    avatar: "avatar.jpg"
  },
  title: "My Post"
}

// 适用于：
// - 引用的数据不常变化
// - 避免频繁 $lookup
// - 允许短期的数据不一致
```

### 5. 预聚合模式

```javascript
// 实时计算开销大
db.pageViews.find({ pageId: "page1" }).count();

// ✅ 预聚合存储统计数据
{
  _id: "page1",
  viewCount: 1000000,
  lastUpdated: ISODate("...")
}

// 增量更新
db.pages.updateOne(
  { _id: "page1" },
  { 
    $inc: { viewCount: 1 },
    $set: { lastUpdated: new Date() }
  }
);
```

## Node.js 集成（Mongoose）

### Schema 设计最佳实践

```typescript
import mongoose, { Schema, Document } from 'mongoose';

// 使用 TypeScript 类型
interface IUser extends Document {
  email: string;
  username: string;
  profile: {
    firstName: string;
    lastName: string;
    avatar?: string;
  };
  settings: {
    notifications: boolean;
    privacy: 'public' | 'private';
  };
  createdAt: Date;
  updatedAt: Date;
}

const UserSchema = new Schema<IUser>({
  email: {
    type: String,
    required: true,
    unique: true,
    lowercase: true,
    trim: true,
    index: true
  },
  username: {
    type: String,
    required: true,
    unique: true,
    minlength: 3,
    maxlength: 30
  },
  profile: {
    firstName: { type: String, required: true },
    lastName: { type: String, required: true },
    avatar: String
  },
  settings: {
    notifications: { type: Boolean, default: true },
    privacy: { 
      type: String, 
      enum: ['public', 'private'], 
      default: 'public' 
    }
  }
}, {
  timestamps: true,  // 自动添加 createdAt, updatedAt
  toJSON: { 
    virtuals: true,
    transform: (doc, ret) => {
      delete ret.__v;
      return ret;
    }
  }
});

// 虚拟字段
UserSchema.virtual('fullName').get(function() {
  return `${this.profile.firstName} ${this.profile.lastName}`;
});

// 中间件
UserSchema.pre('save', async function(next) {
  if (this.isModified('email')) {
    // 邮箱变更时的逻辑
  }
  next();
});

// 实例方法
UserSchema.methods.comparePassword = async function(password: string) {
  // 比较密码
  return bcrypt.compare(password, this.password);
};

// 静态方法
UserSchema.statics.findByEmail = function(email: string) {
  return this.findOne({ email: email.toLowerCase() });
};

const User = mongoose.model<IUser>('User', UserSchema);
```

### 高级查询技巧

```typescript
// 1. 投影（只查询需要的字段）
const users = await User.find()
  .select('username email profile.avatar')
  .lean();  // 返回普通对象，性能更好

// 2. 填充引用（Populate）
const posts = await Post.find()
  .populate('author', 'username avatar')
  .populate({
    path: 'comments',
    populate: {
      path: 'author',
      select: 'username'
    }
  });

// 3. 聚合查询
const stats = await User.aggregate([
  { $match: { status: 'active' } },
  {
    $group: {
      _id: '$country',
      count: { $sum: 1 },
      avgAge: { $avg: '$age' }
    }
  },
  { $sort: { count: -1 } }
]);

// 4. 批量操作
const bulk = User.collection.initializeUnorderedBulkOp();
bulk.find({ status: 'inactive' }).update({ $set: { status: 'archived' } });
bulk.find({ lastLogin: { $lt: oldDate } }).remove();
await bulk.execute();

// 5. 游标（处理大量数据）
const cursor = User.find({ status: 'active' }).cursor();

for (let user = await cursor.next(); user != null; user = await cursor.next()) {
  await processUser(user);
}

// 或使用 eachAsync
await User.find({ status: 'active' }).cursor().eachAsync(async (user) => {
  await processUser(user);
}, { parallel: 10 });  // 并行处理 10 个
```

## 性能优化

### 查询优化

```javascript
// ❌ 不好：查询整个文档
db.users.find({ city: "NYC" });

// ✅ 好：只投影需要的字段
db.users.find({ city: "NYC" }, { name: 1, email: 1 });

// ❌ 不好：多次查询
for (const userId of userIds) {
  await User.findById(userId);
}

// ✅ 好：批量查询
const users = await User.find({ _id: { $in: userIds } });

// ❌ 不好：skip 在大偏移量时很慢
db.posts.find().skip(10000).limit(10);

// ✅ 好：基于范围的分页
db.posts.find({ _id: { $gt: lastId } }).limit(10);

// 使用 hint 强制使用索引
db.users.find({ city: "NYC" }).hint({ city: 1, age: -1 });

// 使用 allowDiskUse 处理大型聚合
db.orders.aggregate(
  [...pipeline],
  { allowDiskUse: true }
);
```

### 连接池配置

```typescript
import mongoose from 'mongoose';

mongoose.connect(process.env.MONGODB_URI, {
  maxPoolSize: 50,        // 最大连接数
  minPoolSize: 10,        // 最小连接数
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  family: 4               // 使用 IPv4
});

// 监控连接池
mongoose.connection.on('connected', () => {
  console.log('MongoDB connected');
});

mongoose.connection.on('error', (err) => {
  console.error('MongoDB error:', err);
});

mongoose.connection.on('disconnected', () => {
  console.log('MongoDB disconnected');
});
```

### 写入优化

```javascript
// ❌ 不好：逐个插入
for (const doc of docs) {
  await collection.insertOne(doc);
}

// ✅ 好：批量插入
await collection.insertMany(docs, { ordered: false });

// 使用 bulkWrite
await collection.bulkWrite([
  {
    insertOne: { document: doc1 }
  },
  {
    updateOne: {
      filter: { _id: id },
      update: { $set: { status: 'active' } }
    }
  },
  {
    deleteOne: { filter: { _id: oldId } }
  }
], { ordered: false });
```

## Change Streams（变更流）

```typescript
// 监听集合变化
const changeStream = User.watch();

changeStream.on('change', (change) => {
  console.log('Change detected:', change);
  
  switch (change.operationType) {
    case 'insert':
      // 新用户注册
      break;
    case 'update':
      // 用户信息更新
      break;
    case 'delete':
      // 用户删除
      break;
  }
});

// 过滤特定变化
const pipeline = [
  {
    $match: {
      'fullDocument.status': 'active',
      operationType: { $in: ['insert', 'update'] }
    }
  }
];

const filteredStream = User.watch(pipeline);

// 实战：缓存失效
const userStream = User.watch([
  {
    $match: {
      operationType: { $in: ['update', 'delete'] }
    }
  }
]);

userStream.on('change', async (change) => {
  const userId = change.documentKey._id;
  await redis.del(`user:${userId}`);
});
```

## 分片（Sharding）

```javascript
// 分片键选择
// ✅ 好的分片键：
// - 高基数（cardinality）
// - 查询常用
// - 均匀分布

// 示例：按用户 ID 分片
sh.shardCollection("mydb.users", { userId: 1 });

// 范围分片 vs 哈希分片
sh.shardCollection("mydb.orders", { orderId: 1 });  // 范围
sh.shardCollection("mydb.logs", { _id: "hashed" }); // 哈希

// 复合分片键
sh.shardCollection("mydb.events", { 
  userId: 1, 
  timestamp: 1 
});

// 查询时尽量包含分片键
// ✅ 好：定向查询（只查一个分片）
db.users.find({ userId: "user123" });

// ❌ 不好：广播查询（查所有分片）
db.users.find({ email: "test@example.com" });
```

## 常见面试题

### 1. MongoDB 的优势和适用场景？

<details>
<summary>点击查看答案</summary>

**优势**：
1. 灵活的 Schema
2. 横向扩展能力强
3. 高性能读写
4. 丰富的查询语言
5. 原生支持 JSON
6. 强大的聚合框架

**适用场景**：
- 内容管理系统
- 实时分析
- 物联网数据
- 移动应用后端
- 产品目录
- 用户配置文件

**不适用场景**：
- 复杂事务（虽然 4.0+ 支持）
- 复杂 JOIN 操作
- 强一致性要求
</details>

### 2. MongoDB 的索引原理？

<details>
<summary>点击查看答案</summary>

MongoDB 使用 **B-tree 索引**：

**工作原理**：
1. 索引存储字段值和文档指针
2. 按顺序组织，支持范围查询
3. 索引可以覆盖查询（Covered Query）

**ESR 规则**（索引优化）：
1. **E**quality - 精确匹配字段放最前
2. **S**ort - 排序字段放中间
3. **R**ange - 范围查询字段放最后

```javascript
// ✅ 好的索引顺序
db.orders.createIndex({ 
  status: 1,      // Equality
  createdAt: -1,  // Sort
  amount: 1       // Range
});

// 查询会使用索引
db.orders.find({ 
  status: "completed",
  amount: { $gt: 100 }
}).sort({ createdAt: -1 });
```

**索引注意事项**：
- 每个索引占用空间
- 写入时需要更新索引
- 太多索引影响性能
- 定期分析索引使用情况
</details>

### 3. 如何避免 MongoDB 性能问题？

<details>
<summary>点击查看答案</summary>

**查询优化**：
1. 创建合适的索引
2. 使用投影（只查询需要的字段）
3. 使用 explain() 分析查询
4. 避免大偏移量的 skip

**数据建模**：
1. 合理使用嵌套 vs 引用
2. 避免无限增长的数组
3. 使用桶模式处理时序数据
4. 预聚合统计数据

**写入优化**：
1. 使用批量操作
2. 使用 ordered: false
3. 考虑写入关注级别

**监控**：
1. 慢查询日志
2. 连接池监控
3. 磁盘使用率
4. 复制延迟
</details>

### 4. MongoDB 的复制集（Replica Set）工作原理？

<details>
<summary>点击查看答案</summary>

**架构**：
- 1 个 Primary（主节点）
- N 个 Secondary（从节点）
- 可选的 Arbiter（仲裁者）

**工作流程**：
1. 写入到 Primary
2. 操作记录到 oplog
3. Secondary 复制 oplog
4. Primary 故障时自动选举

**读取策略**：
```typescript
// 默认：只从 Primary 读
await User.find().read('primary');

// 从 Secondary 读（可能有延迟）
await User.find().read('secondary');

// 读取偏好
await User.find().read('primaryPreferred');
```

**写入关注**：
```typescript
// 等待写入到多个节点
await User.create(doc, {
  writeConcern: { 
    w: 'majority',    // 多数节点确认
    wtimeout: 5000    // 超时时间
  }
});
```
</details>

### 5. 嵌套文档 vs 引用，如何选择？

<details>
<summary>点击查看答案</summary>

**使用嵌套文档**：
```javascript
// 适用场景：
// 1. 数据总是一起访问
// 2. 一对少量（1-100）
// 3. 数据不独立更新

{
  _id: ObjectId("..."),
  name: "John",
  addresses: [
    { street: "123 Main", city: "NYC" },
    { street: "456 Work", city: "NYC" }
  ]
}
```

**使用引用**：
```javascript
// 适用场景：
// 1. 数据可以独立访问
// 2. 多对多关系
// 3. 数据量大
// 4. 频繁更新

{
  _id: ObjectId("user1"),
  name: "John"
}

{
  _id: ObjectId("post1"),
  userId: ObjectId("user1"),
  title: "My Post"
}
```

**混合方案**：
```javascript
// 扩展引用：存储常用字段
{
  _id: ObjectId("post1"),
  userId: ObjectId("user1"),
  userInfo: {
    name: "John",
    avatar: "avatar.jpg"
  },
  title: "My Post"
}
```

**决策树**：
1. 数据总是一起访问？→ 嵌套
2. 数据独立更新频繁？→ 引用
3. 数组会无限增长？→ 引用
4. 需要在多处引用？→ 引用
5. 数据量小且稳定？→ 嵌套
</details>

## 最佳实践

### 1. 使用 lean() 提升性能

```typescript
// ✅ 返回普通对象，性能提升 5-10 倍
const users = await User.find().lean();

// ❌ 返回 Mongoose 文档（带方法和 getter）
const users = await User.find();
```

### 2. 限制返回字段

```typescript
// ✅ 只返回需要的字段
const users = await User.find()
  .select('username email')
  .lean();
```

### 3. 合理使用事务

```typescript
// ❌ 不需要事务的场景
await User.updateOne({ _id: id }, { $inc: { loginCount: 1 } });

// ✅ 需要事务的场景（跨文档原子性）
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  await User.updateOne({ _id: fromId }, { $inc: { balance: -100 } }, { session });
  await User.updateOne({ _id: toId }, { $inc: { balance: 100 } }, { session });
});
```

### 4. 监控慢查询

```javascript
// 启用慢查询日志
db.setProfilingLevel(1, { slowms: 100 });

// 查看慢查询
db.system.profile.find().sort({ ts: -1 }).limit(10);
```

### 5. 定期维护

```javascript
// 重建索引
db.users.reIndex();

// 查看集合统计
db.users.stats();

// 清理孤儿文档（分片环境）
db.runCommand({ cleanupOrphaned: "mydb.mycollection" });
```

