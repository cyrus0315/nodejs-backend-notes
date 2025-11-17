# Prisma

Prisma 是新一代的 TypeScript ORM，以类型安全、开发体验和性能著称。

## Prisma Schema

### 基础定义

```prisma
// schema.prisma
generator client {
  provider = "prisma-client-js"
  previewFeatures = ["fullTextSearch", "views"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  
  posts     Post[]
  profile   Profile?
  
  @@index([email])
  @@map("users")
}

model Profile {
  id     String @id @default(uuid())
  bio    String?
  avatar String?
  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("profiles")
}

model Post {
  id          String   @id @default(uuid())
  title       String
  content     String?
  published   Boolean  @default(false)
  authorId    String
  author      User     @relation(fields: [authorId], references: [id])
  categories  Category[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  @@index([authorId])
  @@index([published])
  @@map("posts")
}

model Category {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]
  
  @@map("categories")
}
```

### 字段类型和修饰符

```prisma
model Example {
  // 基本类型
  stringField    String
  intField       Int
  floatField     Float
  boolField      Boolean
  dateField      DateTime
  jsonField      Json
  bytesField     Bytes
  decimalField   Decimal
  bigIntField    BigInt
  
  // 可选字段
  optionalField  String?
  
  // 默认值
  defaultString  String   @default("default")
  defaultInt     Int      @default(0)
  defaultBool    Boolean  @default(false)
  defaultDate    DateTime @default(now())
  autoIncrement  Int      @default(autoincrement())
  uuid           String   @default(uuid())
  cuid           String   @default(cuid())
  
  // 唯一约束
  uniqueField    String   @unique
  
  // 数组
  tags           String[]
  
  // 枚举
  role           Role     @default(USER)
  
  // 自动更新
  updatedAt      DateTime @updatedAt
  
  // 自定义字段名
  emailAddress   String   @map("email_address")
  
  @@id([field1, field2])  // 复合主键
  @@unique([field1, field2])  // 复合唯一约束
  @@index([field1, field2])  // 复合索引
  @@map("table_name")  // 自定义表名
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### 关系定义

```prisma
// 一对一
model User {
  id      String   @id
  profile Profile?
}

model Profile {
  id     String @id
  userId String @unique
  user   User   @relation(fields: [userId], references: [id])
}

// 一对多
model User {
  id    String @id
  posts Post[]
}

model Post {
  id       String @id
  authorId String
  author   User   @relation(fields: [authorId], references: [id])
}

// 多对多（显式关系表）
model Post {
  id         String         @id
  categories PostCategory[]
}

model Category {
  id    String         @id
  posts PostCategory[]
}

model PostCategory {
  postId     String
  categoryId String
  assignedAt DateTime @default(now())
  
  post     Post     @relation(fields: [postId], references: [id])
  category Category @relation(fields: [categoryId], references: [id])
  
  @@id([postId, categoryId])
}

// 多对多（隐式）
model Post {
  id         String     @id
  categories Category[]
}

model Category {
  id    String @id
  posts Post[]
}

// 自引用关系
model User {
  id        String @id
  followers User[] @relation("UserFollows")
  following User[] @relation("UserFollows")
}
```

## Prisma Client 使用

### 基础 CRUD

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: ['query', 'info', 'warn', 'error'],
});

// Create
const user = await prisma.user.create({
  data: {
    email: 'user@example.com',
    name: 'John Doe',
    profile: {
      create: {
        bio: 'Software Developer'
      }
    }
  },
  include: {
    profile: true
  }
});

// Read - findUnique
const user = await prisma.user.findUnique({
  where: { email: 'user@example.com' }
});

// Read - findMany
const users = await prisma.user.findMany({
  where: {
    email: {
      contains: '@example.com'
    }
  },
  orderBy: {
    createdAt: 'desc'
  },
  take: 10,
  skip: 0
});

// Read - findFirst
const user = await prisma.user.findFirst({
  where: {
    email: {
      endsWith: '@example.com'
    }
  }
});

// Update
const updatedUser = await prisma.user.update({
  where: { id: '123' },
  data: {
    name: 'Jane Doe',
    profile: {
      update: {
        bio: 'Senior Developer'
      }
    }
  }
});

// Update many
const result = await prisma.user.updateMany({
  where: {
    email: {
      contains: '@old-domain.com'
    }
  },
  data: {
    updatedAt: new Date()
  }
});

// Upsert
const user = await prisma.user.upsert({
  where: { email: 'user@example.com' },
  update: {
    name: 'Updated Name'
  },
  create: {
    email: 'user@example.com',
    name: 'New User'
  }
});

// Delete
await prisma.user.delete({
  where: { id: '123' }
});

// Delete many
const result = await prisma.user.deleteMany({
  where: {
    createdAt: {
      lt: new Date('2020-01-01')
    }
  }
});
```

### 高级查询

```typescript
// 关系查询
const users = await prisma.user.findMany({
  include: {
    posts: true,
    profile: true
  }
});

// 嵌套查询
const users = await prisma.user.findMany({
  include: {
    posts: {
      where: {
        published: true
      },
      orderBy: {
        createdAt: 'desc'
      },
      take: 5,
      include: {
        categories: true
      }
    }
  }
});

// 选择字段（select）
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true,
    posts: {
      select: {
        title: true,
        published: true
      }
    }
  }
});

// 计数
const count = await prisma.user.count({
  where: {
    email: {
      endsWith: '@example.com'
    }
  }
});

// 聚合
const result = await prisma.post.aggregate({
  _count: {
    id: true
  },
  _avg: {
    viewCount: true
  },
  _sum: {
    viewCount: true
  },
  _max: {
    createdAt: true
  },
  _min: {
    createdAt: true
  }
});

// 分组
const result = await prisma.post.groupBy({
  by: ['authorId', 'published'],
  _count: {
    id: true
  },
  _sum: {
    viewCount: true
  },
  having: {
    viewCount: {
      _sum: {
        gt: 1000
      }
    }
  }
});

// 复杂过滤
const posts = await prisma.post.findMany({
  where: {
    AND: [
      { published: true },
      {
        OR: [
          { title: { contains: 'prisma' } },
          { content: { contains: 'prisma' } }
        ]
      }
    ],
    author: {
      email: {
        endsWith: '@example.com'
      }
    },
    categories: {
      some: {
        name: 'Technology'
      }
    }
  }
});

// 全文搜索（PostgreSQL）
const posts = await prisma.post.findMany({
  where: {
    title: {
      search: 'database | prisma'
    }
  }
});
```

### 原始查询

```typescript
// 原始 SQL 查询
const result = await prisma.$queryRaw`
  SELECT * FROM users 
  WHERE email LIKE ${`%${domain}%`}
  LIMIT 10
`;

// 类型安全的原始查询
import { Prisma } from '@prisma/client';

const users = await prisma.$queryRaw<User[]>(
  Prisma.sql`SELECT * FROM users WHERE email = ${email}`
);

// 执行 SQL（不返回结果）
await prisma.$executeRaw`
  UPDATE users 
  SET updated_at = NOW() 
  WHERE created_at < NOW() - INTERVAL '1 year'
`;

// 不安全的原始查询（避免使用）
const result = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE id = ${userId}`
);
```

## 事务

### 交互式事务

```typescript
// 自动提交/回滚
await prisma.$transaction(async (tx) => {
  // 创建用户
  const user = await tx.user.create({
    data: {
      email: 'user@example.com',
      name: 'John Doe'
    }
  });
  
  // 创建账户
  const account = await tx.account.create({
    data: {
      userId: user.id,
      balance: 1000
    }
  });
  
  // 如果抛出错误，整个事务回滚
  if (account.balance < 0) {
    throw new Error('Invalid balance');
  }
  
  return { user, account };
});

// 嵌套事务
await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({ data: {...} });
  
  // 嵌套操作
  await tx.post.createMany({
    data: [
      { authorId: user.id, title: 'Post 1' },
      { authorId: user.id, title: 'Post 2' }
    ]
  });
});

// 设置超时和隔离级别
await prisma.$transaction(
  async (tx) => {
    // 事务操作
  },
  {
    maxWait: 5000, // 最大等待时间
    timeout: 10000, // 事务超时时间
    isolationLevel: Prisma.TransactionIsolationLevel.Serializable
  }
);
```

### 顺序操作事务

```typescript
// 数组形式的事务（按顺序执行）
const [user, post, comment] = await prisma.$transaction([
  prisma.user.create({
    data: { email: 'user@example.com', name: 'John' }
  }),
  prisma.post.create({
    data: { title: 'My Post', authorId: '...' }
  }),
  prisma.comment.create({
    data: { content: 'Great!', postId: '...' }
  })
]);

// 如果任何一个操作失败，所有操作回滚
```

### 乐观锁

```typescript
// 使用版本号实现乐观锁
model Post {
  id      String @id
  title   String
  version Int    @default(0)
}

// 更新时检查版本号
async function updatePost(id: string, data: any, currentVersion: number) {
  try {
    const updated = await prisma.post.update({
      where: {
        id,
        version: currentVersion  // 确保版本匹配
      },
      data: {
        ...data,
        version: {
          increment: 1  // 增加版本号
        }
      }
    });
    return updated;
  } catch (error) {
    if (error.code === 'P2025') {
      throw new Error('Record has been modified by another user');
    }
    throw error;
  }
}
```

## 中间件

```typescript
// 日志中间件
prisma.$use(async (params, next) => {
  const before = Date.now();
  
  const result = await next(params);
  
  const after = Date.now();
  console.log(`Query ${params.model}.${params.action} took ${after - before}ms`);
  
  return result;
});

// 软删除中间件
prisma.$use(async (params, next) => {
  // 拦截 delete 操作
  if (params.action === 'delete') {
    params.action = 'update';
    params.args['data'] = { deletedAt: new Date() };
  }
  
  // 拦截 deleteMany 操作
  if (params.action === 'deleteMany') {
    params.action = 'updateMany';
    if (params.args.data !== undefined) {
      params.args.data['deletedAt'] = new Date();
    } else {
      params.args['data'] = { deletedAt: new Date() };
    }
  }
  
  return next(params);
});

// 查询时过滤已删除记录
prisma.$use(async (params, next) => {
  if (params.action === 'findUnique' || params.action === 'findFirst') {
    params.action = 'findFirst';
    params.args.where['deletedAt'] = null;
  }
  
  if (params.action === 'findMany') {
    if (params.args.where) {
      if (params.args.where.deletedAt === undefined) {
        params.args.where['deletedAt'] = null;
      }
    } else {
      params.args['where'] = { deletedAt: null };
    }
  }
  
  return next(params);
});

// 审计中间件
prisma.$use(async (params, next) => {
  const result = await next(params);
  
  // 记录所有修改操作
  if (['create', 'update', 'delete', 'updateMany', 'deleteMany'].includes(params.action)) {
    await prisma.auditLog.create({
      data: {
        model: params.model,
        action: params.action,
        data: JSON.stringify(params.args),
        timestamp: new Date()
      }
    });
  }
  
  return result;
});

// 加密中间件
import bcrypt from 'bcrypt';

prisma.$use(async (params, next) => {
  if (params.model === 'User' && params.action === 'create') {
    if (params.args.data.password) {
      params.args.data.password = await bcrypt.hash(
        params.args.data.password,
        10
      );
    }
  }
  
  return next(params);
});
```

## 迁移

```bash
# 创建迁移
npx prisma migrate dev --name init

# 创建迁移但不执行
npx prisma migrate dev --create-only

# 应用迁移到生产环境
npx prisma migrate deploy

# 重置数据库
npx prisma migrate reset

# 查看迁移状态
npx prisma migrate status

# 解决迁移冲突
npx prisma migrate resolve --applied 20230101000000_migration_name
npx prisma migrate resolve --rolled-back 20230101000000_migration_name
```

### 自定义迁移

```sql
-- migrations/20230101000000_custom/migration.sql

-- 创建全文搜索索引
CREATE INDEX posts_title_content_idx ON posts 
USING GIN (to_tsvector('english', title || ' ' || content));

-- 创建触发器
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_posts_updated_at 
BEFORE UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- 数据迁移
UPDATE users SET role = 'USER' WHERE role IS NULL;
```

## 数据填充（Seeding）

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  // 清空数据
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
  
  // 创建用户
  const user1 = await prisma.user.create({
    data: {
      email: 'alice@example.com',
      name: 'Alice',
      posts: {
        create: [
          {
            title: 'First Post',
            content: 'Hello World',
            published: true
          },
          {
            title: 'Second Post',
            content: 'Learning Prisma',
            published: false
          }
        ]
      }
    }
  });
  
  const user2 = await prisma.user.create({
    data: {
      email: 'bob@example.com',
      name: 'Bob',
      posts: {
        create: [
          {
            title: 'Bob\'s Post',
            content: 'My first post',
            published: true
          }
        ]
      }
    }
  });
  
  console.log({ user1, user2 });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

```json
// package.json
{
  "prisma": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

```bash
# 运行种子数据
npx prisma db seed
```

## 性能优化

### 1. 连接池配置

```typescript
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=10&pool_timeout=20`
    }
  }
});

// 或在 .env 中配置
// DATABASE_URL="postgresql://user:password@localhost:5432/db?connection_limit=10&pool_timeout=20"
```

### 2. 查询优化

```typescript
// ❌ N+1 问题
const users = await prisma.user.findMany();
for (const user of users) {
  const posts = await prisma.post.findMany({
    where: { authorId: user.id }
  });
}

// ✅ 使用 include 一次查询
const users = await prisma.user.findMany({
  include: {
    posts: true
  }
});

// ✅ 只查询需要的字段
const users = await prisma.user.findMany({
  select: {
    id: true,
    name: true,
    email: true
  }
});

// ✅ 分页
const posts = await prisma.post.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: {
    createdAt: 'desc'
  }
});

// ✅ 游标分页（更高效）
const posts = await prisma.post.findMany({
  take: 20,
  cursor: {
    id: lastPostId
  },
  skip: 1  // 跳过游标本身
});
```

### 3. 批量操作

```typescript
// ❌ 循环插入
for (const userData of users) {
  await prisma.user.create({ data: userData });
}

// ✅ 批量插入
await prisma.user.createMany({
  data: users,
  skipDuplicates: true
});

// ✅ 批量更新（使用事务）
await prisma.$transaction(
  users.map(user =>
    prisma.user.update({
      where: { id: user.id },
      data: user
    })
  )
);
```

### 4. 索引优化

```prisma
model Post {
  id        String   @id
  title     String
  authorId  String
  published Boolean
  
  // 单列索引
  @@index([authorId])
  @@index([published])
  
  // 复合索引（查询 authorId + published）
  @@index([authorId, published])
  
  // 唯一索引
  @@unique([authorId, title])
}
```

### 5. 数据加载器（DataLoader）

```typescript
import DataLoader from 'dataloader';

// 批量加载用户
const userLoader = new DataLoader(async (userIds: string[]) => {
  const users = await prisma.user.findMany({
    where: {
      id: {
        in: userIds
      }
    }
  });
  
  // 保证顺序一致
  const userMap = new Map(users.map(user => [user.id, user]));
  return userIds.map(id => userMap.get(id));
});

// 使用
const user = await userLoader.load(userId);
```

## 与 NestJS 集成

```typescript
// prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }
  
  async onModuleDestroy() {
    await this.$disconnect();
  }
  
  // 启用软删除
  constructor() {
    super();
    this.enableSoftDelete();
  }
  
  private enableSoftDelete() {
    this.$use(async (params, next) => {
      if (params.action === 'delete') {
        params.action = 'update';
        params.args['data'] = { deletedAt: new Date() };
      }
      return next(params);
    });
  }
}

// user.service.ts
@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}
  
  async findAll() {
    return this.prisma.user.findMany({
      include: {
        posts: true
      }
    });
  }
  
  async create(data: Prisma.UserCreateInput) {
    return this.prisma.user.create({ data });
  }
}
```

## 常见面试题

### 1. Prisma 相比传统 ORM 的优势？

<details>
<summary>点击查看答案</summary>

**优势**：

1. **类型安全**：
   - 自动生成的类型定义
   - 编译时检查
   - IDE 自动完成

2. **性能**：
   - 查询引擎用 Rust 编写
   - 优化的查询生成
   - 减少往返次数

3. **开发体验**：
   - 声明式 Schema
   - 自动迁移
   - Prisma Studio 可视化工具

4. **关系查询**：
   - 直观的嵌套查询
   - 自动处理 JOIN
   - 避免 N+1 问题

**劣势**：
- 灵活性略低
- 复杂查询需要原始 SQL
- 学习曲线（新概念）
</details>

### 2. 如何避免 N+1 查询问题？

<details>
<summary>点击查看答案</summary>

```typescript
// ❌ N+1 问题
const users = await prisma.user.findMany();
for (const user of users) {
  user.posts = await prisma.post.findMany({
    where: { authorId: user.id }
  });
}

// ✅ 解决方案 1：使用 include
const users = await prisma.user.findMany({
  include: {
    posts: true
  }
});

// ✅ 解决方案 2：使用 DataLoader
import DataLoader from 'dataloader';

const postLoader = new DataLoader(async (userIds: string[]) => {
  const posts = await prisma.post.findMany({
    where: {
      authorId: { in: userIds }
    }
  });
  
  // 按 authorId 分组
  const grouped = userIds.map(id =>
    posts.filter(post => post.authorId === id)
  );
  
  return grouped;
});

// 使用
const users = await prisma.user.findMany();
const usersWithPosts = await Promise.all(
  users.map(async user => ({
    ...user,
    posts: await postLoader.load(user.id)
  }))
);
```
</details>

### 3. Prisma 事务的隔离级别有哪些？

<details>
<summary>点击查看答案</summary>

```typescript
import { Prisma } from '@prisma/client';

await prisma.$transaction(
  async (tx) => {
    // 事务操作
  },
  {
    isolationLevel: Prisma.TransactionIsolationLevel.ReadUncommitted
    // ReadUncommitted
    // ReadCommitted（默认）
    // RepeatableRead
    // Serializable
  }
);
```

**隔离级别说明**：
- **Read Uncommitted**：最低隔离，可能脏读
- **Read Committed**：避免脏读，可能不可重复读
- **Repeatable Read**：避免不可重复读，可能幻读
- **Serializable**：最高隔离，避免所有问题，性能最差
</details>

### 4. 如何实现软删除？

<details>
<summary>点击查看答案</summary>

```prisma
model Post {
  id        String    @id
  title     String
  deletedAt DateTime?
  
  @@index([deletedAt])
}
```

```typescript
// 中间件实现
prisma.$use(async (params, next) => {
  // 拦截 delete
  if (params.action === 'delete') {
    params.action = 'update';
    params.args['data'] = { deletedAt: new Date() };
  }
  
  // 拦截 deleteMany
  if (params.action === 'deleteMany') {
    params.action = 'updateMany';
    params.args['data'] = { deletedAt: new Date() };
  }
  
  // 过滤查询
  if (params.action === 'findMany' || params.action === 'findFirst') {
    params.args.where = {
      ...params.args.where,
      deletedAt: null
    };
  }
  
  return next(params);
});

// 查询包括已删除
const allPosts = await prisma.post.findMany({
  where: {
    deletedAt: { not: null }  // 只查已删除
  }
});
```
</details>

### 5. Prisma 的连接池如何配置？

<details>
<summary>点击查看答案</summary>

```typescript
// 方式 1：在代码中配置
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: `${process.env.DATABASE_URL}?connection_limit=10&pool_timeout=20`
    }
  }
});

// 方式 2：在 .env 中配置
// DATABASE_URL="postgresql://user:password@localhost:5432/db?connection_limit=10&pool_timeout=20&connect_timeout=10"
```

**参数说明**：
- `connection_limit`：最大连接数（默认无限制）
- `pool_timeout`：获取连接超时时间（秒）
- `connect_timeout`：建立连接超时时间（秒）

**推荐配置**：
```
connection_limit = min(服务器最大连接数 / 实例数, 20)
pool_timeout = 20
```
</details>

## 最佳实践

1. **总是使用类型安全的查询**
2. **合理使用 include 和 select**
3. **为频繁查询的字段添加索引**
4. **使用事务保证数据一致性**
5. **批量操作使用 createMany/updateMany**
6. **定期运行 prisma format 格式化 schema**
7. **使用中间件实现横切关注点**
8. **生产环境使用 prisma migrate deploy**
9. **配置合适的连接池大小**
10. **监控查询性能（log: ['query']）**

