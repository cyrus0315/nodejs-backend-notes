# ORM/ODM 对比与选型

深入对比主流 ORM/ODM 框架，帮助做出正确的技术选型。

## 总览对比

| 特性 | Prisma | TypeORM | Sequelize | Mongoose | Drizzle |
|------|--------|---------|-----------|----------|---------|
| 类型安全 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 学习曲线 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 社区 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 文档 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| 数据库支持 | 6+ | 10+ | 6+ | MongoDB | 5+ |
| 发布年份 | 2019 | 2016 | 2010 | 2010 | 2022 |

## 详细对比

### Prisma

**优势**：
- ✅ 自动生成类型定义，完美的 TypeScript 支持
- ✅ 声明式 Schema，简单直观
- ✅ Prisma Studio 可视化工具
- ✅ 性能优秀（Rust 查询引擎）
- ✅ 优秀的开发体验
- ✅ 自动迁移生成
- ✅ 强大的关系查询

**劣势**：
- ❌ 灵活性略低
- ❌ 复杂 SQL 需要原始查询
- ❌ 社区相对较新
- ❌ 不支持某些高级数据库特性
- ❌ Schema 与代码分离

**适用场景**：
- 新项目
- 追求类型安全
- 需要快速开发
- 团队不熟悉 SQL

**示例**：
```prisma
model User {
  id    String @id @default(uuid())
  email String @unique
  posts Post[]
}

model Post {
  id       String @id @default(uuid())
  title    String
  author   User   @relation(fields: [authorId], references: [id])
  authorId String
}
```

```typescript
const users = await prisma.user.findMany({
  include: {
    posts: true
  }
});
```

### TypeORM

**优势**：
- ✅ 成熟稳定，社区大
- ✅ 支持 Active Record 和 Data Mapper 模式
- ✅ 强大的 QueryBuilder
- ✅ 支持多种数据库
- ✅ 装饰器语法优雅
- ✅ 丰富的功能（监听器、订阅者）
- ✅ 原生 SQL 支持好

**劣势**：
- ❌ TypeScript 类型推断较弱
- ❌ 性能一般
- ❌ 复杂的关系配置
- ❌ 迁移可能不稳定

**适用场景**：
- 已有项目
- 需要复杂 SQL
- 多数据库支持
- 熟悉装饰器模式

**示例**：
```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column({ unique: true })
  email: string;
  
  @OneToMany(() => Post, post => post.author)
  posts: Post[];
}

// 使用
const users = await userRepository.find({
  relations: ['posts']
});
```

### Sequelize

**优势**：
- ✅ 最早的 Node.js ORM
- ✅ 社区成熟
- ✅ 文档完善
- ✅ 功能丰富
- ✅ 支持多种数据库
- ✅ 钩子系统强大

**劣势**：
- ❌ TypeScript 支持较弱
- ❌ 性能一般
- ❌ API 设计较老
- ❌ 类型定义复杂
- ❌ Promise 链可能冗长

**适用场景**：
- 遗留项目
- JavaScript 项目
- 熟悉 Sequelize API

**示例**：
```typescript
const User = sequelize.define('User', {
  email: {
    type: DataTypes.STRING,
    unique: true
  }
});

const users = await User.findAll({
  include: [{ model: Post }]
});
```

### Mongoose

**优势**：
- ✅ MongoDB 官方推荐
- ✅ 社区最大（MongoDB）
- ✅ Schema 验证强大
- ✅ 中间件系统完善
- ✅ 虚拟属性和实例方法
- ✅ 聚合管道支持好
- ✅ 灵活的文档模型

**劣势**：
- ❌ 只支持 MongoDB
- ❌ TypeScript 支持需要额外配置
- ❌ 性能依赖 MongoDB 驱动
- ❌ 关系查询相对复杂

**适用场景**：
- MongoDB 项目
- 文档型数据模型
- 需要灵活 Schema
- 实时数据更新

**示例**：
```typescript
const UserSchema = new Schema({
  email: { type: String, unique: true },
  posts: [{ type: ObjectId, ref: 'Post' }]
});

const users = await User.find()
  .populate('posts');
```

### Drizzle ORM

**优势**：
- ✅ 极致的类型安全
- ✅ 零依赖
- ✅ 轻量级（~7KB）
- ✅ 性能极高
- ✅ SQL-like API
- ✅ 无代码生成

**劣势**：
- ❌ 社区较小
- ❌ 文档不够完善
- ❌ 功能相对简单
- ❌ 缺少高级特性

**适用场景**：
- 追求极致性能
- 需要完全类型安全
- 小型到中型项目
- 熟悉 SQL

**示例**：
```typescript
import { pgTable, serial, text } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  email: text('email').notNull()
});

const result = await db.select().from(users);
```

## 性能对比

### 基准测试

```
测试：插入 10,000 条记录

Drizzle:    1,200ms  ⭐⭐⭐⭐⭐
Prisma:     1,500ms  ⭐⭐⭐⭐
TypeORM:    3,200ms  ⭐⭐⭐
Sequelize:  3,500ms  ⭐⭐⭐
Mongoose:   2,100ms  ⭐⭐⭐⭐ (MongoDB)
```

```
测试：查询 1,000 条记录（含关联）

Drizzle:     150ms  ⭐⭐⭐⭐⭐
Prisma:      200ms  ⭐⭐⭐⭐⭐
Mongoose:    250ms  ⭐⭐⭐⭐
TypeORM:     450ms  ⭐⭐⭐
Sequelize:   500ms  ⭐⭐⭐
```

### 内存占用

```
Drizzle:    ~15MB
Prisma:     ~35MB
Mongoose:   ~40MB
TypeORM:    ~50MB
Sequelize:  ~55MB
```

## TypeScript 支持对比

### Prisma
```typescript
// 自动生成类型
const user = await prisma.user.findUnique({
  where: { id: '123' },
  include: { posts: true }
});

// user 类型完全推断
// user.email: string
// user.posts: Post[]
```

### TypeORM
```typescript
// 需要手动定义类型
const user = await userRepository.findOne({
  where: { id: 123 },
  relations: ['posts']
});

// user 类型：User | undefined
// 需要类型断言或检查
```

### Sequelize
```typescript
// 类型支持较弱
const user = await User.findByPk(123, {
  include: [Post]
});

// user 类型复杂，需要额外配置
```

### Mongoose
```typescript
// 需要定义接口
interface IUser extends Document {
  email: string;
  posts: Types.ObjectId[];
}

const user = await User.findById(id)
  .populate('posts');

// user 类型：IUser | null
```

### Drizzle
```typescript
// 完全类型安全
const users = await db
  .select()
  .from(usersTable)
  .leftJoin(postsTable, eq(usersTable.id, postsTable.userId));

// 类型自动推断，完全类型安全
```

## 关系查询对比

### 一对多查询

**Prisma**：
```typescript
const users = await prisma.user.findMany({
  include: {
    posts: {
      where: { published: true }
    }
  }
});
```

**TypeORM**：
```typescript
const users = await userRepository.find({
  relations: ['posts'],
  where: {
    posts: { published: true }
  }
});

// 或使用 QueryBuilder
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post', 'post.published = :published', { published: true })
  .getMany();
```

**Mongoose**：
```typescript
const users = await User.find()
  .populate({
    path: 'posts',
    match: { published: true }
  });
```

**Drizzle**：
```typescript
const users = await db
  .select()
  .from(usersTable)
  .leftJoin(postsTable, eq(usersTable.id, postsTable.userId))
  .where(eq(postsTable.published, true));
```

## 迁移对比

### Prisma
```bash
# 创建迁移
npx prisma migrate dev --name init

# 应用迁移
npx prisma migrate deploy
```

优点：自动生成，声明式
缺点：较难自定义

### TypeORM
```bash
# 生成迁移
npx typeorm migration:generate -n Migration

# 运行迁移
npx typeorm migration:run
```

优点：灵活，支持自定义
缺点：生成的迁移可能不完美

### Sequelize
```bash
# 创建迁移
npx sequelize-cli migration:generate --name migration

# 运行迁移
npx sequelize-cli db:migrate
```

优点：CLI 工具完善
缺点：需要手动编写

### Mongoose
无内置迁移系统，需要使用第三方工具（如 migrate-mongo）

### Drizzle
```bash
# 生成迁移
npx drizzle-kit generate:pg

# 应用迁移
npx drizzle-kit push:pg
```

优点：类型安全的迁移
缺点：功能相对简单

## 选型建议

### 按项目类型

**新项目，追求类型安全和性能**：
1. Prisma（综合最佳）
2. Drizzle（追求极致性能）

**大型企业项目**：
1. TypeORM（功能丰富，成熟）
2. Prisma（类型安全）

**MongoDB 项目**：
1. Mongoose（唯一选择）

**遗留项目**：
1. 保持现有 ORM
2. 评估迁移成本

### 按团队经验

**熟悉 SQL**：
- TypeORM（QueryBuilder）
- Drizzle（SQL-like）

**不熟悉 SQL**：
- Prisma（声明式）
- Mongoose（文档模型）

**重视类型安全**：
1. Drizzle
2. Prisma
3. TypeORM

### 按性能要求

**高性能要求**：
1. Drizzle
2. Prisma
3. Mongoose（MongoDB）

**普通性能要求**：
- TypeORM
- Sequelize

### 按数据库类型

**PostgreSQL**：
1. Prisma
2. Drizzle
3. TypeORM

**MySQL**：
1. Prisma
2. TypeORM
3. Sequelize

**MongoDB**：
1. Mongoose（唯一选择）

**多数据库**：
1. TypeORM
2. Sequelize

## 常见问题

### 1. 为什么 Prisma 比 TypeORM 快？

<details>
<summary>点击查看答案</summary>

**主要原因**：

1. **查询引擎**：
   - Prisma：Rust 编写，性能极高
   - TypeORM：JavaScript/TypeScript

2. **查询优化**：
   - Prisma：自动优化查询
   - TypeORM：需要手动优化

3. **连接管理**：
   - Prisma：更高效的连接池
   - TypeORM：标准连接池

4. **批量操作**：
   - Prisma：自动批量处理
   - TypeORM：需要手动实现

5. **类型生成**：
   - Prisma：编译时生成，零运行时开销
   - TypeORM：装饰器有运行时开销

**基准测试**：
- 简单查询：Prisma 快 20-30%
- 关系查询：Prisma 快 30-50%
- 批量操作：Prisma 快 40-60%
</details>

### 2. N+1 查询问题如何解决？

<details>
<summary>点击查看答案</summary>

**Prisma**：
```typescript
// ❌ N+1
const users = await prisma.user.findMany();
for (const user of users) {
  user.posts = await prisma.post.findMany({
    where: { authorId: user.id }
  });
}

// ✅ include 自动处理
const users = await prisma.user.findMany({
  include: { posts: true }
});
```

**TypeORM**：
```typescript
// ❌ N+1
const users = await userRepository.find();
for (const user of users) {
  user.posts = await postRepository.find({
    where: { authorId: user.id }
  });
}

// ✅ relations
const users = await userRepository.find({
  relations: ['posts']
});

// ✅ QueryBuilder + JOIN
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .getMany();
```

**Mongoose**：
```typescript
// ❌ N+1
const users = await User.find();
for (const user of users) {
  user.posts = await Post.find({ author: user._id });
}

// ✅ populate
const users = await User.find().populate('posts');

// ✅ aggregation
const users = await User.aggregate([
  {
    $lookup: {
      from: 'posts',
      localField: '_id',
      foreignField: 'author',
      as: 'posts'
    }
  }
]);
```
</details>

### 3. 如何在 ORM 中使用原始 SQL？

<details>
<summary>点击查看答案</summary>

**Prisma**：
```typescript
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE email LIKE ${`%${domain}%`}
`;

await prisma.$executeRaw`
  UPDATE users SET updated_at = NOW()
`;
```

**TypeORM**：
```typescript
const users = await connection.query(
  'SELECT * FROM users WHERE email LIKE ?',
  [`%${domain}%`]
);

await connection.query(
  'UPDATE users SET updated_at = NOW()'
);
```

**Sequelize**：
```typescript
const [users, metadata] = await sequelize.query(
  'SELECT * FROM users WHERE email LIKE :domain',
  {
    replacements: { domain: `%${domain}%` },
    type: QueryTypes.SELECT
  }
);
```

**Mongoose**：
```typescript
// 使用 aggregation pipeline
const users = await User.aggregate([
  { $match: { email: { $regex: domain } } }
]);

// 或直接使用 MongoDB 驱动
const users = await User.collection.find({
  email: { $regex: domain }
}).toArray();
```
</details>

### 4. ORM 事务如何处理？

<details>
<summary>点击查看答案</summary>

**Prisma**：
```typescript
await prisma.$transaction(async (tx) => {
  const user = await tx.user.create({ data: {...} });
  await tx.post.create({ data: { authorId: user.id, ...} });
});
```

**TypeORM**：
```typescript
await connection.transaction(async manager => {
  const user = await manager.save(User, {...});
  await manager.save(Post, { author: user, ...});
});
```

**Sequelize**：
```typescript
const t = await sequelize.transaction();
try {
  const user = await User.create({...}, { transaction: t });
  await Post.create({ userId: user.id, ...}, { transaction: t });
  await t.commit();
} catch (error) {
  await t.rollback();
}
```

**Mongoose**：
```typescript
const session = await mongoose.startSession();
await session.withTransaction(async () => {
  const user = await User.create([{...}], { session });
  await Post.create([{ author: user[0]._id, ...}], { session });
});
```
</details>

## 总结

| 需求 | 推荐 ORM |
|------|----------|
| 类型安全 + 性能 | Prisma / Drizzle |
| 功能丰富 + 灵活 | TypeORM |
| MongoDB | Mongoose |
| 极致性能 | Drizzle |
| 学习成本低 | Prisma |
| 复杂 SQL | TypeORM |
| 遗留项目 | 保持现有 |

**2024 年推荐排序**：
1. **Prisma**：综合最佳，适合大多数场景
2. **Drizzle**：追求极致性能和类型安全
3. **TypeORM**：成熟稳定，功能丰富
4. **Mongoose**：MongoDB 唯一选择
5. **Sequelize**：遗留项目维护

