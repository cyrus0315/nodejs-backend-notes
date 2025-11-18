# 数据库设计与优化

数据库是系统的核心，良好的数据库设计能够显著提升系统性能和可维护性。本文深入讲解数据库设计原则、优化技巧和实战方案。

## 目录
- [数据库设计原则](#数据库设计原则)
- [索引设计](#索引设计)
- [查询优化](#查询优化)
- [分库分表](#分库分表)
- [读写分离](#读写分离)
- [数据一致性](#数据一致性)
- [Node.js 实战](#nodejs-实战)
- [面试题](#常见面试题)

---

## 数据库设计原则

### 三范式（3NF）

#### 第一范式（1NF）：原子性

每个字段不可再分。

```sql
-- ❌ 违反 1NF
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  phones VARCHAR(255)  -- '1234567890,9876543210'
);

-- ✅ 符合 1NF
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100)
);

CREATE TABLE user_phones (
  id INT PRIMARY KEY,
  user_id INT,
  phone VARCHAR(20),
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

#### 第二范式（2NF）：完全依赖

非主键字段完全依赖于主键。

```sql
-- ❌ 违反 2NF（order_status 只依赖 order_id）
CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  quantity INT,
  order_status VARCHAR(20),  -- 部分依赖
  PRIMARY KEY (order_id, product_id)
);

-- ✅ 符合 2NF
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  order_status VARCHAR(20)
);

CREATE TABLE order_items (
  order_id INT,
  product_id INT,
  quantity INT,
  PRIMARY KEY (order_id, product_id),
  FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

#### 第三范式（3NF）：消除传递依赖

非主键字段不依赖于其他非主键字段。

```sql
-- ❌ 违反 3NF（department_name 依赖于 department_id）
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  department_id INT,
  department_name VARCHAR(100)  -- 传递依赖
);

-- ✅ 符合 3NF
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  department_id INT,
  FOREIGN KEY (department_id) REFERENCES departments(id)
);

CREATE TABLE departments (
  id INT PRIMARY KEY,
  name VARCHAR(100)
);
```

### 反范式化

为性能牺牲规范化。

```typescript
// 场景：商品表 + 分类表
// 方案 1：规范化（符合 3NF）
interface Product {
  id: number;
  name: string;
  categoryId: number;  // 外键
}

interface Category {
  id: number;
  name: string;
}

// 查询需要 JOIN
const products = await prisma.product.findMany({
  include: { category: true }  // JOIN
});

// 方案 2：反范式化（性能优化）
interface Product {
  id: number;
  name: string;
  categoryId: number;
  categoryName: string;  // 冗余字段
}

// 查询无需 JOIN（性能提升）
const products = await prisma.product.findMany();

// 缺点：更新分类名称时，需要更新所有商品
await prisma.$transaction([
  prisma.category.update({
    where: { id: categoryId },
    data: { name: newName }
  }),
  prisma.product.updateMany({
    where: { categoryId },
    data: { categoryName: newName }
  })
]);
```

**何时反范式化？**
- ✅ 查询频繁，更新少
- ✅ 关联查询性能差
- ✅ 数据量大
- ❌ 数据一致性要求高

---

## 索引设计

### 索引原理

```typescript
// 无索引：全表扫描（O(n)）
SELECT * FROM users WHERE email = 'john@example.com';
// 扫描 100 万行

// 有索引：B+树查找（O(log n)）
CREATE INDEX idx_users_email ON users(email);
// 只需查找 log2(1000000) ≈ 20 次
```

### B+树索引

```
        [P]
       /   \
     [F]   [T]
    / |    | \
  [A][C]  [P][T]
  
叶子节点：存储数据或指针
非叶子节点：存储索引
叶子节点之间：链表（范围查询）
```

### 索引类型

#### 1. 单列索引

```sql
CREATE INDEX idx_users_email ON users(email);

-- 使用场景
SELECT * FROM users WHERE email = 'john@example.com';
```

```typescript
// Prisma
model User {
  id    Int    @id
  email String @unique  // 自动创建唯一索引
  
  @@index([email])  // 普通索引
}
```

#### 2. 复合索引

```sql
CREATE INDEX idx_users_lastname_firstname 
ON users(last_name, first_name);

-- ✅ 会使用索引
SELECT * FROM users 
WHERE last_name = 'Smith';

SELECT * FROM users 
WHERE last_name = 'Smith' AND first_name = 'John';

-- ❌ 不会使用索引（违反最左前缀原则）
SELECT * FROM users 
WHERE first_name = 'John';
```

**最左前缀原则**：复合索引 (A, B, C) 可以当作 (A)、(A,B)、(A,B,C) 使用。

#### 3. 唯一索引

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- 保证唯一性
INSERT INTO users (email) VALUES ('john@example.com');
INSERT INTO users (email) VALUES ('john@example.com'); -- 错误
```

#### 4. 全文索引

```sql
-- PostgreSQL
CREATE INDEX idx_posts_content_fulltext 
ON posts USING gin(to_tsvector('english', content));

-- 全文搜索
SELECT * FROM posts 
WHERE to_tsvector('english', content) @@ to_tsquery('node & javascript');
```

```typescript
// Prisma (PostgreSQL)
model Post {
  id      Int    @id
  content String
  
  @@index([content(ops: raw("gin_trgm_ops"))], type: Gin)
}

// 使用
const posts = await prisma.$queryRaw`
  SELECT * FROM posts
  WHERE content @@ plainto_tsquery('english', ${query})
`;
```

### 索引优化

#### 1. 选择性（Selectivity）

**定义**：不同值的数量 / 总行数

```typescript
// 高选择性（好）
// email: 100 万行，100 万个不同值，选择性 = 1.0
CREATE INDEX idx_users_email ON users(email);

// 低选择性（差）
// gender: 100 万行，只有 2 个不同值，选择性 = 0.000002
CREATE INDEX idx_users_gender ON users(gender); -- ❌ 不建议
```

**建议**：选择性 > 0.1 时建索引。

#### 2. 索引覆盖

```sql
-- 索引：(email, name, age)
SELECT name, age FROM users WHERE email = 'john@example.com';
-- ✅ 覆盖索引：只查索引，不回表

SELECT * FROM users WHERE email = 'john@example.com';
-- ❌ 非覆盖索引：查索引 + 回表查数据
```

```typescript
// Prisma
model User {
  id    Int    @id
  email String
  name  String
  age   Int
  
  // 覆盖索引
  @@index([email, name, age])
}
```

#### 3. 索引下推（Index Condition Pushdown）

```sql
-- 索引：(age, city)
SELECT * FROM users 
WHERE age > 25 AND city = 'New York';

-- 传统方式：
-- 1. 使用索引查找 age > 25
-- 2. 回表获取数据
-- 3. 过滤 city = 'New York'

-- 索引下推（MySQL 5.6+）：
-- 1. 使用索引查找 age > 25
-- 2. 在索引中过滤 city = 'New York'
-- 3. 回表获取数据
-- 减少回表次数
```

### 索引失效场景

```sql
-- 1. ❌ 使用函数
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
-- ✅ 改为
SELECT * FROM users WHERE email = 'john@example.com';

-- 2. ❌ 隐式类型转换
SELECT * FROM users WHERE phone = 1234567890;  -- phone 是 VARCHAR
-- ✅ 改为
SELECT * FROM users WHERE phone = '1234567890';

-- 3. ❌ OR 连接不同字段
SELECT * FROM users WHERE email = 'john@example.com' OR phone = '1234567890';
-- ✅ 改为 UNION
SELECT * FROM users WHERE email = 'john@example.com'
UNION
SELECT * FROM users WHERE phone = '1234567890';

-- 4. ❌ LIKE 以通配符开头
SELECT * FROM users WHERE email LIKE '%@example.com';
-- ✅ 改为（如果可能）
SELECT * FROM users WHERE email LIKE 'john@%';

-- 5. ❌ NOT、!=、<>
SELECT * FROM users WHERE status != 'active';
-- ✅ 改为
SELECT * FROM users WHERE status IN ('pending', 'suspended', 'deleted');
```

---

## 查询优化

### EXPLAIN 分析

```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';
```

**重要字段**：
- `type`：访问类型（system > const > eq_ref > ref > range > index > ALL）
- `key`：使用的索引
- `rows`：扫描的行数
- `Extra`：额外信息

```typescript
// Node.js 查询分析
async function analyzeQuery(query: string) {
  const result = await prisma.$queryRaw`EXPLAIN ${query}`;
  console.table(result);
}

await analyzeQuery('SELECT * FROM users WHERE email = "john@example.com"');
```

### 查询优化技巧

#### 1. SELECT 只查需要的字段

```typescript
// ❌ 查询所有字段
const users = await prisma.user.findMany();

// ✅ 只查需要的字段
const users = await prisma.user.findMany({
  select: { id: true, name: true, email: true }
});
```

#### 2. 避免 N+1 查询

```typescript
// ❌ N+1 问题
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id }
  });
  user.posts = posts;
}
// 1 + N 次查询

// ✅ 使用 JOIN
const users = await prisma.user.findMany({
  include: { posts: true }
});
// 1 次查询（或 2 次，取决于 ORM 实现）

// ✅ 使用 DataLoader
import DataLoader from 'dataloader';

const postLoader = new DataLoader(async (userIds: number[]) => {
  const posts = await prisma.post.findMany({
    where: { authorId: { in: userIds } }
  });
  
  // 按 userId 分组
  const postsByUserId = new Map<number, Post[]>();
  for (const post of posts) {
    const userPosts = postsByUserId.get(post.authorId) || [];
    userPosts.push(post);
    postsByUserId.set(post.authorId, userPosts);
  }
  
  return userIds.map(id => postsByUserId.get(id) || []);
});

// 使用
const users = await prisma.user.findMany();
await Promise.all(
  users.map(async (user) => {
    user.posts = await postLoader.load(user.id);
  })
);
```

#### 3. 分页优化

```typescript
// ❌ OFFSET 分页（大偏移量性能差）
const users = await prisma.user.findMany({
  skip: 100000,  // 仍需扫描前 100000 行
  take: 20
});

// ✅ 游标分页（推荐）
const users = await prisma.user.findMany({
  take: 20,
  cursor: { id: lastUserId },  // 从上次结束的地方继续
  skip: 1,  // 跳过游标本身
  orderBy: { id: 'asc' }
});
```

#### 4. 批量操作

```typescript
// ❌ 循环插入
for (const user of users) {
  await prisma.user.create({ data: user });
}
// N 次数据库往返

// ✅ 批量插入
await prisma.user.createMany({
  data: users
});
// 1 次数据库往返
```

#### 5. 使用事务

```typescript
// ✅ 事务（保证一致性）
await prisma.$transaction(async (tx) => {
  const order = await tx.order.create({ data: orderData });
  
  await tx.inventory.update({
    where: { productId: orderData.productId },
    data: { stock: { decrement: orderData.quantity } }
  });
  
  await tx.payment.create({ data: paymentData });
});
```

---

## 分库分表

### 垂直拆分

**按业务模块拆分**。

```typescript
// 拆分前：单库
const prisma = new PrismaClient();

// 拆分后：多库
const userDB = new PrismaClient({
  datasources: { db: { url: 'postgresql://localhost/user_db' } }
});

const orderDB = new PrismaClient({
  datasources: { db: { url: 'postgresql://localhost/order_db' } }
});

const productDB = new PrismaClient({
  datasources: { db: { url: 'postgresql://localhost/product_db' } }
});

// 使用
const user = await userDB.user.findUnique({ where: { id: userId } });
const orders = await orderDB.order.findMany({ where: { userId } });
```

**优点**：
- ✅ 解耦业务
- ✅ 提高并发
- ✅ 故障隔离

**缺点**：
- ❌ 跨库查询困难
- ❌ 跨库事务复杂

### 水平拆分

**按数据量拆分**。

#### 分表策略

**1. 范围分表**

```typescript
// 按用户 ID 范围分表
function getTableName(userId: number): string {
  if (userId < 1000000) return 'users_0';
  if (userId < 2000000) return 'users_1';
  if (userId < 3000000) return 'users_2';
  return 'users_3';
}

async function getUser(userId: number) {
  const tableName = getTableName(userId);
  return await prisma.$queryRaw`
    SELECT * FROM ${tableName} WHERE id = ${userId}
  `;
}
```

**2. 哈希分表**

```typescript
// 按用户 ID 哈希分表
function getTableName(userId: number): string {
  const tableCount = 4;
  const tableIndex = userId % tableCount;
  return `users_${tableIndex}`;
}

async function getUser(userId: number) {
  const tableName = getTableName(userId);
  return await prisma.$queryRaw`
    SELECT * FROM ${tableName} WHERE id = ${userId}
  `;
}
```

**3. 时间分表**

```typescript
// 按时间分表（日志表）
function getTableName(date: Date): string {
  const month = date.toISOString().slice(0, 7).replace('-', '');
  return `logs_${month}`;  // logs_202401, logs_202402, ...
}

async function getLogs(startDate: Date, endDate: Date) {
  // 需要查询多个表
  const tables = getTablesBetween(startDate, endDate);
  
  const results = await Promise.all(
    tables.map(table => 
      prisma.$queryRaw`SELECT * FROM ${table} WHERE date >= ${startDate} AND date <= ${endDate}`
    )
  );
  
  return results.flat();
}
```

#### Sharding 实现

```typescript
class ShardingManager {
  private shards: PrismaClient[] = [];

  constructor(shardCount: number) {
    for (let i = 0; i < shardCount; i++) {
      this.shards.push(new PrismaClient({
        datasources: {
          db: { url: `postgresql://localhost/shard_${i}` }
        }
      }));
    }
  }

  getShard(key: number): PrismaClient {
    const index = key % this.shards.length;
    return this.shards[index];
  }

  async get(userId: number) {
    const shard = this.getShard(userId);
    return await shard.user.findUnique({ where: { id: userId } });
  }

  async create(userData: any) {
    const userId = await this.generateId();
    const shard = this.getShard(userId);
    return await shard.user.create({
      data: { ...userData, id: userId }
    });
  }

  async query(condition: any): Promise<any[]> {
    // 查询所有分片
    const results = await Promise.all(
      this.shards.map(shard => 
        shard.user.findMany({ where: condition })
      )
    );
    return results.flat();
  }

  private async generateId(): Promise<number> {
    // 使用 Snowflake 等分布式 ID 生成器
    return snowflake.generate();
  }
}

// 使用
const sharding = new ShardingManager(4);

// 单条查询（路由到特定分片）
const user = await sharding.get(1234567);

// 全局查询（需要查询所有分片）
const activeUsers = await sharding.query({ status: 'active' });
```

### 分库分表的问题

**1. 跨分片查询**

```typescript
// 问题：JOIN 跨分片
SELECT u.*, o.*
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.city = 'New York';

// 解决：应用层聚合
async function getUsersWithOrders(city: string) {
  // 1. 查询所有分片的用户
  const users = await Promise.all(
    shards.map(shard => 
      shard.user.findMany({ where: { city } })
    )
  ).then(results => results.flat());

  // 2. 查询订单
  const userIds = users.map(u => u.id);
  const orders = await Promise.all(
    shards.map(shard => 
      shard.order.findMany({ where: { userId: { in: userIds } } })
    )
  ).then(results => results.flat());

  // 3. 应用层聚合
  const ordersByUserId = new Map<number, any[]>();
  for (const order of orders) {
    const userOrders = ordersByUserId.get(order.userId) || [];
    userOrders.push(order);
    ordersByUserId.set(order.userId, userOrders);
  }

  return users.map(user => ({
    ...user,
    orders: ordersByUserId.get(user.id) || []
  }));
}
```

**2. 分布式事务**

```typescript
// 使用 Saga 模式
async function createOrderAcrossShards(orderData: any) {
  const userShard = sharding.getShard(orderData.userId);
  const productShard = sharding.getShard(orderData.productId);

  // Saga 步骤
  try {
    // 步骤 1：扣减库存
    await productShard.product.update({
      where: { id: orderData.productId },
      data: { stock: { decrement: orderData.quantity } }
    });

    // 步骤 2：创建订单
    const order = await userShard.order.create({
      data: orderData
    });

    return order;
  } catch (error) {
    // 补偿：恢复库存
    await productShard.product.update({
      where: { id: orderData.productId },
      data: { stock: { increment: orderData.quantity } }
    });
    throw error;
  }
}
```

**3. 全局唯一 ID**

```typescript
// 不能使用数据库自增 ID
// 使用 Snowflake
const idGenerator = new SnowflakeIdGenerator(workerId);

async function createUser(userData: any) {
  const userId = idGenerator.generate();
  const shard = sharding.getShard(userId);
  
  return await shard.user.create({
    data: { ...userData, id: userId }
  });
}
```

---

## 读写分离

### 架构

```
               ┌────────┐
               │ 应用   │
               └───┬────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
        ▼          ▼          ▼
     [写]       [读]       [读]
    ┌────┐     ┌────┐     ┌────┐
    │主库│────>│从库│────>│从库│
    └────┘     └────┘     └────┘
      (异步复制)
```

### 实现

```typescript
import { PrismaClient } from '@prisma/client';

class DatabaseManager {
  private primary: PrismaClient;  // 主库
  private replicas: PrismaClient[];  // 从库
  private replicaIndex = 0;

  constructor() {
    this.primary = new PrismaClient({
      datasources: {
        db: { url: process.env.PRIMARY_DB_URL }
      }
    });

    this.replicas = [
      new PrismaClient({
        datasources: { db: { url: process.env.REPLICA_1_URL } }
      }),
      new PrismaClient({
        datasources: { db: { url: process.env.REPLICA_2_URL } }
      }),
      new PrismaClient({
        datasources: { db: { url: process.env.REPLICA_3_URL } }
      })
    ];
  }

  // 轮询选择从库
  private getReadConnection(): PrismaClient {
    const replica = this.replicas[this.replicaIndex];
    this.replicaIndex = (this.replicaIndex + 1) % this.replicas.length;
    return replica;
  }

  // 写操作：主库
  async create(data: any) {
    return await this.primary.user.create({ data });
  }

  async update(id: number, data: any) {
    return await this.primary.user.update({ where: { id }, data });
  }

  async delete(id: number) {
    return await this.primary.user.delete({ where: { id } });
  }

  // 读操作：从库
  async findOne(id: number) {
    const replica = this.getReadConnection();
    return await replica.user.findUnique({ where: { id } });
  }

  async findMany(query: any) {
    const replica = this.getReadConnection();
    return await replica.user.findMany(query);
  }

  // 强制读主库（需要最新数据）
  async findOneFromPrimary(id: number) {
    return await this.primary.user.findUnique({ where: { id } });
  }
}

const db = new DatabaseManager();

// 使用
app.post('/users', async (req, res) => {
  const user = await db.create(req.body);  // 写主库
  res.json(user);
});

app.get('/users/:id', async (req, res) => {
  const user = await db.findOne(Number(req.params.id));  // 读从库
  res.json(user);
});

app.put('/users/:id', async (req, res) => {
  const user = await db.update(Number(req.params.id), req.body);  // 写主库
  
  // 立即读取：读主库（避免主从延迟）
  const updated = await db.findOneFromPrimary(user.id);
  res.json(updated);
});
```

---

## 数据一致性

### 主从复制延迟

**问题**：写入主库后，从库可能需要几百毫秒同步。

**解决方案**：

#### 1. 写后读主库

```typescript
class SmartDatabaseManager extends DatabaseManager {
  private recentWrites = new Map<string, number>();

  async create(data: any) {
    const user = await this.primary.user.create({ data });
    
    // 记录最近写入
    this.recentWrites.set(`user:${user.id}`, Date.now());
    
    return user;
  }

  async findOne(id: number) {
    const writeTime = this.recentWrites.get(`user:${id}`);
    
    // 5 秒内有写入，读主库
    if (writeTime && Date.now() - writeTime < 5000) {
      return await this.primary.user.findUnique({ where: { id } });
    }
    
    // 否则读从库
    return await super.findOne(id);
  }
}
```

#### 2. 会话粘性

```typescript
// 同一用户的请求，在一段时间内都读主库
app.use((req, res, next) => {
  const sessionId = req.headers['session-id'] as string;
  const lastWrite = cache.get(`session:${sessionId}:write`);
  
  if (lastWrite && Date.now() - lastWrite < 5000) {
    req.forceReadPrimary = true;
  }
  
  next();
});

app.get('/users/:id', async (req, res) => {
  const db = req.forceReadPrimary ? primary : replica;
  const user = await db.user.findUnique({ where: { id: Number(req.params.id) } });
  res.json(user);
});
```

---

## Node.js 实战

### 完整的数据库管理系统

```typescript
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';

interface DatabaseConfig {
  primary: string;
  replicas: string[];
  redis: {
    host: string;
    port: number;
  };
}

class EnterpriseDatabase {
  private primary: PrismaClient;
  private replicas: PrismaClient[];
  private redis: Redis;
  private replicaIndex = 0;

  constructor(config: DatabaseConfig) {
    this.primary = new PrismaClient({
      datasources: { db: { url: config.primary } },
      log: ['query', 'error', 'warn']
    });

    this.replicas = config.replicas.map(url => 
      new PrismaClient({
        datasources: { db: { url } }
      })
    );

    this.redis = new Redis(config.redis);
  }

  // ========== 读操作（带缓存） ==========
  async findUser(id: number): Promise<User | null> {
    // L1: 缓存
    const cached = await this.redis.get(`user:${id}`);
    if (cached) {
      return JSON.parse(cached);
    }

    // L2: 从库
    const replica = this.getReadConnection();
    const user = await replica.user.findUnique({ where: { id } });

    if (user) {
      // 写入缓存
      await this.redis.setex(`user:${id}`, 3600, JSON.stringify(user));
    }

    return user;
  }

  // ========== 写操作 ==========
  async createUser(data: any): Promise<User> {
    const user = await this.primary.user.create({ data });
    
    // 写入缓存
    await this.redis.setex(`user:${user.id}`, 3600, JSON.stringify(user));
    
    return user;
  }

  async updateUser(id: number, data: any): Promise<User> {
    const user = await this.primary.user.update({ where: { id }, data });
    
    // 删除缓存（让下次读取时重新加载）
    await this.redis.del(`user:${id}`);
    
    return user;
  }

  async deleteUser(id: number): Promise<void> {
    await this.primary.user.delete({ where: { id } });
    await this.redis.del(`user:${id}`);
  }

  // ========== 事务 ==========
  async transfer(fromId: number, toId: number, amount: number): Promise<void> {
    await this.primary.$transaction(async (tx) => {
      // 扣款
      await tx.account.update({
        where: { userId: fromId },
        data: { balance: { decrement: amount } }
      });

      // 加款
      await tx.account.update({
        where: { userId: toId },
        data: { balance: { increment: amount } }
      });

      // 记录流水
      await tx.transaction.create({
        data: {
          fromUserId: fromId,
          toUserId: toId,
          amount,
          type: 'transfer',
          createdAt: new Date()
        }
      });
    });

    // 清理缓存
    await Promise.all([
      this.redis.del(`account:${fromId}`),
      this.redis.del(`account:${toId}`)
    ]);
  }

  // ========== 批量操作 ==========
  async bulkCreate(users: any[]): Promise<number> {
    const result = await this.primary.user.createMany({
      data: users,
      skipDuplicates: true
    });
    
    return result.count;
  }

  // ========== 工具方法 ==========
  private getReadConnection(): PrismaClient {
    const replica = this.replicas[this.replicaIndex];
    this.replicaIndex = (this.replicaIndex + 1) % this.replicas.length;
    return replica;
  }

  async healthCheck(): Promise<{
    primary: boolean;
    replicas: boolean[];
    redis: boolean;
  }> {
    const checks = await Promise.allSettled([
      this.primary.$queryRaw`SELECT 1`,
      ...this.replicas.map(r => r.$queryRaw`SELECT 1`),
      this.redis.ping()
    ]);

    return {
      primary: checks[0].status === 'fulfilled',
      replicas: checks.slice(1, -1).map(c => c.status === 'fulfilled'),
      redis: checks[checks.length - 1].status === 'fulfilled'
    };
  }

  async disconnect(): Promise<void> {
    await this.primary.$disconnect();
    await Promise.all(this.replicas.map(r => r.$disconnect()));
    await this.redis.quit();
  }
}

// 使用
const db = new EnterpriseDatabase({
  primary: 'postgresql://localhost/primary',
  replicas: [
    'postgresql://localhost/replica1',
    'postgresql://localhost/replica2'
  ],
  redis: {
    host: 'localhost',
    port: 6379
  }
});

// 读取
const user = await db.findUser(123);

// 写入
await db.updateUser(123, { name: 'John' });

// 事务
await db.transfer(1, 2, 100);

// 健康检查
const health = await db.healthCheck();
console.log(health);
```

---

## 常见面试题

### 1. 如何设计高性能的数据库表？

**回答要点**：

1. **遵循范式**（适度）
   - 3NF：消除冗余
   - 反范式：性能优化

2. **合理建索引**
   - 高选择性字段
   - WHERE、ORDER BY、JOIN 字段
   - 避免过多索引（影响写入）

3. **选择合适的数据类型**
   - 能用 INT 不用 VARCHAR
   - 能用 TIMESTAMP 不用 VARCHAR
   - 定长优于变长

4. **合理使用分区/分表**
   - 单表 < 1000 万行
   - 历史数据归档

5. **考虑查询模式**
   - 读多写少：冗余、缓存
   - 写多读少：异步、队列

### 2. 索引的优缺点？

**优点**：
- ✅ 加快查询速度（O(log n)）
- ✅ 唯一性约束
- ✅ 减少排序成本

**缺点**：
- ❌ 占用存储空间
- ❌ 降低写入性能（需要维护索引）
- ❌ 索引选择错误可能更慢

**何时不建索引**：
- 小表（< 1000 行）
- 频繁更新的字段
- 低选择性字段（如性别）
- 表经常大批量插入

### 3. 如何优化慢查询？

**步骤**：

1. **EXPLAIN 分析**
```sql
EXPLAIN SELECT * FROM users WHERE email = 'john@example.com';
```

2. **检查索引**
   - type = ALL（全表扫描）→ 建索引
   - key = NULL（未使用索引）→ 检查索引失效原因

3. **优化查询**
   - 只查需要的字段（避免 SELECT *）
   - 避免子查询（改为 JOIN）
   - 使用 LIMIT
   - 避免函数、类型转换

4. **考虑缓存**
   - Redis 缓存热点数据
   - 结果缓存

5. **数据库优化**
   - 读写分离
   - 分库分表
   - 升级硬件

### 4. 分库分表的方案？

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **垂直拆分** | 业务模块清晰 | 业务解耦 | 跨库查询 |
| **水平拆分（哈希）** | 数据量大 | 分布均匀 | 扩容困难 |
| **水平拆分（范围）** | 有时间维度 | 扩容容易 | 热点问题 |
| **一致性哈希** | 需要动态扩容 | 扩容影响小 | 实现复杂 |

**选择建议**：
- 按业务模块：垂直拆分
- 按数据量：水平拆分（哈希）
- 历史数据归档：范围拆分

### 5. 如何保证主从一致性？

**问题**：主从复制延迟（通常 < 100ms）

**解决方案**：

1. **强制读主库**
   - 写入后立即读取
   - 需要最新数据的场景

2. **延迟读取**
   - 写入后等待 100ms
   - 再从从库读取

3. **会话粘性**
   - 同一用户在一段时间内读主库

4. **版本号/时间戳**
   - 客户端记录版本号
   - 读取时检查版本

5. **同步复制**（牺牲性能）
   - 主库等待从库确认
   - 适合金融等场景

---

## 总结

### 数据库设计要点

1. **设计原则**
   - 遵循范式（适度）
   - 合理反范式化
   - 考虑查询模式

2. **索引设计**
   - 高选择性字段
   - 复合索引（最左前缀）
   - 覆盖索引
   - 避免过多索引

3. **查询优化**
   - EXPLAIN 分析
   - 避免 N+1
   - 合理使用缓存
   - 批量操作

4. **扩展方案**
   - 垂直拆分（业务）
   - 水平拆分（数据量）
   - 读写分离（性能）
   - 分布式 ID

5. **一致性保证**
   - 事务（ACID）
   - 主从延迟处理
   - 分布式事务（Saga）

### 实践检查清单

- [ ] 表设计是否合理？
- [ ] 索引是否合理？
- [ ] 是否有慢查询？
- [ ] 是否有 N+1 问题？
- [ ] 是否使用了缓存？
- [ ] 是否需要分库分表？
- [ ] 是否需要读写分离？
- [ ] 事务隔离级别是否合理？
- [ ] 是否考虑了主从延迟？
- [ ] 是否有监控和告警？
- [ ] 备份策略是否完善？
- [ ] 是否有容灾方案？

---

**下一篇**：[实战案例](./05-real-world-cases.md)

