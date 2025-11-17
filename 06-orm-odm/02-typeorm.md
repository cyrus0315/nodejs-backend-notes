# TypeORM

TypeORM 是一个成熟的 TypeScript/JavaScript ORM，支持多种数据库，使用装饰器定义模型。

## Entity 定义

### 基础 Entity

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
  VersionColumn
} from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;
  
  @Column({ type: 'varchar', length: 255, unique: true })
  email: string;
  
  @Column({ nullable: true })
  name: string;
  
  @Column({ type: 'enum', enum: ['user', 'admin'], default: 'user' })
  role: string;
  
  @Column({ type: 'boolean', default: true })
  isActive: boolean;
  
  @Column({ type: 'json', nullable: true })
  metadata: any;
  
  @CreateDateColumn()
  createdAt: Date;
  
  @UpdateDateColumn()
  updatedAt: Date;
  
  @DeleteDateColumn()  // 软删除
  deletedAt: Date;
  
  @VersionColumn()  // 乐观锁
  version: number;
}
```

### 字段类型

```typescript
@Entity()
export class Example {
  // 主键
  @PrimaryGeneratedColumn()
  id: number;
  
  @PrimaryGeneratedColumn('uuid')
  uuid: string;
  
  @PrimaryColumn()
  customId: string;
  
  // 基础类型
  @Column('varchar', { length: 255 })
  stringField: string;
  
  @Column('int')
  intField: number;
  
  @Column('float')
  floatField: number;
  
  @Column('decimal', { precision: 10, scale: 2 })
  decimalField: number;
  
  @Column('boolean')
  boolField: boolean;
  
  @Column('date')
  dateField: Date;
  
  @Column('datetime')
  datetimeField: Date;
  
  @Column('timestamp')
  timestampField: Date;
  
  @Column('json')
  jsonField: any;
  
  @Column('text')
  textField: string;
  
  @Column('simple-array')
  arrayField: string[];
  
  @Column('simple-json')
  jsonObjectField: { key: string };
  
  // 默认值
  @Column({ default: 'default value' })
  defaultField: string;
  
  @Column({ default: () => 'CURRENT_TIMESTAMP' })
  defaultTimestamp: Date;
  
  // 可空
  @Column({ nullable: true })
  optionalField: string;
  
  // 唯一
  @Column({ unique: true })
  uniqueField: string;
  
  // 自定义列名
  @Column({ name: 'custom_column_name' })
  customName: string;
  
  // 选择/插入时忽略
  @Column({ select: false })
  password: string;
  
  @Column({ insert: false, update: false })
  computedField: string;
}
```

## 关系映射

### 一对一

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  name: string;
  
  @OneToOne(() => Profile, profile => profile.user, {
    cascade: true,
    onDelete: 'CASCADE'
  })
  @JoinColumn()  // 拥有外键的一方
  profile: Profile;
}

@Entity()
export class Profile {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  bio: string;
  
  @OneToOne(() => User, user => user.profile)
  user: User;
}

// 使用
const user = await userRepository.save({
  name: 'John',
  profile: {
    bio: 'Developer'
  }
});
```

### 一对多/多对一

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  name: string;
  
  @OneToMany(() => Post, post => post.author)
  posts: Post[];
}

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  title: string;
  
  @ManyToOne(() => User, user => user.posts, {
    onDelete: 'CASCADE',
    onUpdate: 'CASCADE'
  })
  @JoinColumn({ name: 'author_id' })
  author: User;
  
  @Column({ name: 'author_id' })
  authorId: number;
}

// 使用
const user = await userRepository.findOne({
  where: { id: 1 },
  relations: ['posts']
});
```

### 多对多

```typescript
// 方式 1：自动创建关系表
@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  title: string;
  
  @ManyToMany(() => Category, category => category.posts, {
    cascade: true
  })
  @JoinTable({
    name: 'post_categories',  // 关系表名
    joinColumn: { name: 'post_id', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'category_id', referencedColumnName: 'id' }
  })
  categories: Category[];
}

@Entity()
export class Category {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  name: string;
  
  @ManyToMany(() => Post, post => post.categories)
  posts: Post[];
}

// 使用
const post = await postRepository.save({
  title: 'My Post',
  categories: [
    { name: 'Tech' },
    { name: 'News' }
  ]
});

// 方式 2：显式关系表
@Entity()
export class PostCategory {
  @PrimaryColumn()
  postId: number;
  
  @PrimaryColumn()
  categoryId: number;
  
  @Column()
  assignedAt: Date;
  
  @ManyToOne(() => Post, post => post.postCategories, {
    onDelete: 'CASCADE'
  })
  @JoinColumn({ name: 'post_id' })
  post: Post;
  
  @ManyToOne(() => Category, category => category.postCategories, {
    onDelete: 'CASCADE'
  })
  @JoinColumn({ name: 'category_id' })
  category: Category;
}

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;
  
  @OneToMany(() => PostCategory, pc => pc.post)
  postCategories: PostCategory[];
}

@Entity()
export class Category {
  @PrimaryGeneratedColumn()
  id: number;
  
  @OneToMany(() => PostCategory, pc => pc.category)
  postCategories: PostCategory[];
}
```

### 自引用关系

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  name: string;
  
  // 关注者
  @ManyToMany(() => User, user => user.following)
  @JoinTable({
    name: 'user_follows',
    joinColumn: { name: 'follower_id' },
    inverseJoinColumn: { name: 'following_id' }
  })
  followers: User[];
  
  // 正在关注
  @ManyToMany(() => User, user => user.followers)
  following: User[];
  
  // 父子关系（树形结构）
  @ManyToOne(() => User, user => user.children)
  parent: User;
  
  @OneToMany(() => User, user => user.parent)
  children: User[];
}
```

## Repository 模式

### 基础操作

```typescript
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>
  ) {}
  
  // Create
  async create(data: Partial<User>): Promise<User> {
    const user = this.userRepository.create(data);
    return this.userRepository.save(user);
  }
  
  // Read
  async findAll(): Promise<User[]> {
    return this.userRepository.find();
  }
  
  async findOne(id: number): Promise<User> {
    return this.userRepository.findOne({
      where: { id },
      relations: ['posts', 'profile']
    });
  }
  
  // Update
  async update(id: number, data: Partial<User>): Promise<User> {
    await this.userRepository.update(id, data);
    return this.findOne(id);
  }
  
  // Delete
  async remove(id: number): Promise<void> {
    await this.userRepository.delete(id);
  }
  
  // 软删除
  async softRemove(id: number): Promise<void> {
    await this.userRepository.softDelete(id);
  }
  
  // 恢复
  async restore(id: number): Promise<void> {
    await this.userRepository.restore(id);
  }
}
```

### 高级查询

```typescript
// 条件查询
const users = await userRepository.find({
  where: {
    isActive: true,
    role: 'admin'
  },
  order: {
    createdAt: 'DESC'
  },
  take: 10,
  skip: 0
});

// 复杂查询
import { In, Like, MoreThan, LessThan, Between, IsNull, Not } from 'typeorm';

const users = await userRepository.find({
  where: {
    id: In([1, 2, 3]),
    name: Like('%John%'),
    age: MoreThan(18),
    createdAt: Between(startDate, endDate),
    deletedAt: IsNull()
  }
});

// OR 查询
const users = await userRepository.find({
  where: [
    { role: 'admin' },
    { role: 'moderator' }
  ]
});

// 关系查询
const users = await userRepository.find({
  relations: ['posts', 'profile', 'posts.categories'],
  where: {
    posts: {
      published: true
    }
  }
});

// 选择字段
const users = await userRepository.find({
  select: ['id', 'name', 'email']
});

// 计数
const count = await userRepository.count({
  where: { isActive: true }
});

// 分页
const [users, total] = await userRepository.findAndCount({
  take: 20,
  skip: (page - 1) * 20
});

// 原始查询
const users = await userRepository.query(
  'SELECT * FROM users WHERE email LIKE ?',
  ['%@example.com']
);
```

## Query Builder

### 基础使用

```typescript
// 简单查询
const users = await userRepository
  .createQueryBuilder('user')
  .where('user.isActive = :isActive', { isActive: true })
  .orderBy('user.createdAt', 'DESC')
  .take(10)
  .getMany();

// 复杂查询
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .leftJoinAndSelect('post.categories', 'category')
  .where('user.isActive = :isActive', { isActive: true })
  .andWhere('post.published = :published', { published: true })
  .orderBy('post.createdAt', 'DESC')
  .getMany();

// SELECT 特定字段
const users = await userRepository
  .createQueryBuilder('user')
  .select(['user.id', 'user.name', 'user.email'])
  .getMany();

// 聚合查询
const result = await userRepository
  .createQueryBuilder('user')
  .select('COUNT(user.id)', 'count')
  .addSelect('AVG(user.age)', 'averageAge')
  .getRawOne();

// 分组
const result = await postRepository
  .createQueryBuilder('post')
  .select('post.authorId')
  .addSelect('COUNT(post.id)', 'count')
  .groupBy('post.authorId')
  .having('count > :minCount', { minCount: 5 })
  .getRawMany();

// 子查询
const posts = await postRepository
  .createQueryBuilder('post')
  .where(qb => {
    const subQuery = qb
      .subQuery()
      .select('user.id')
      .from(User, 'user')
      .where('user.isActive = :isActive', { isActive: true })
      .getQuery();
    return 'post.authorId IN ' + subQuery;
  })
  .getMany();

// JOIN
const posts = await postRepository
  .createQueryBuilder('post')
  .innerJoin('post.author', 'author')
  .leftJoinAndSelect('post.categories', 'category')
  .where('author.isActive = :isActive', { isActive: true })
  .getMany();

// 分页
const [posts, total] = await postRepository
  .createQueryBuilder('post')
  .take(20)
  .skip((page - 1) * 20)
  .getManyAndCount();
```

### 更新和删除

```typescript
// 更新
await userRepository
  .createQueryBuilder()
  .update(User)
  .set({ isActive: false })
  .where('lastLoginAt < :date', { date: oneYearAgo })
  .execute();

// 删除
await userRepository
  .createQueryBuilder()
  .delete()
  .from(User)
  .where('id = :id', { id: 1 })
  .execute();

// 软删除
await userRepository
  .createQueryBuilder()
  .softDelete()
  .where('id = :id', { id: 1 })
  .execute();

// 插入
await userRepository
  .createQueryBuilder()
  .insert()
  .into(User)
  .values([
    { name: 'User 1', email: 'user1@example.com' },
    { name: 'User 2', email: 'user2@example.com' }
  ])
  .execute();
```

## 事务

### 方式 1：使用 Connection

```typescript
import { getConnection } from 'typeorm';

await getConnection().transaction(async transactionalEntityManager => {
  const user = await transactionalEntityManager.save(User, {
    name: 'John',
    email: 'john@example.com'
  });
  
  await transactionalEntityManager.save(Profile, {
    userId: user.id,
    bio: 'Developer'
  });
});
```

### 方式 2：使用 QueryRunner

```typescript
const connection = getConnection();
const queryRunner = connection.createQueryRunner();

await queryRunner.connect();
await queryRunner.startTransaction();

try {
  const user = await queryRunner.manager.save(User, {
    name: 'John',
    email: 'john@example.com'
  });
  
  await queryRunner.manager.save(Profile, {
    userId: user.id,
    bio: 'Developer'
  });
  
  await queryRunner.commitTransaction();
} catch (err) {
  await queryRunner.rollbackTransaction();
  throw err;
} finally {
  await queryRunner.release();
}
```

### 方式 3：装饰器（NestJS）

```typescript
import { Transaction, TransactionManager, EntityManager } from 'typeorm';

@Injectable()
export class UserService {
  @Transaction()
  async createUserWithProfile(
    data: any,
    @TransactionManager() manager: EntityManager
  ) {
    const user = await manager.save(User, data.user);
    const profile = await manager.save(Profile, {
      ...data.profile,
      userId: user.id
    });
    
    return { user, profile };
  }
}
```

### 隔离级别

```typescript
await connection.transaction('SERIALIZABLE', async manager => {
  // 事务操作
});

// 隔离级别：
// - READ UNCOMMITTED
// - READ COMMITTED
// - REPEATABLE READ
// - SERIALIZABLE
```

## 监听器和订阅者

### Entity 监听器

```typescript
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;
  
  @Column()
  email: string;
  
  @Column({ select: false })
  password: string;
  
  // 插入前
  @BeforeInsert()
  async hashPassword() {
    if (this.password) {
      this.password = await bcrypt.hash(this.password, 10);
    }
  }
  
  // 更新前
  @BeforeUpdate()
  async hashPasswordOnUpdate() {
    if (this.password) {
      this.password = await bcrypt.hash(this.password, 10);
    }
  }
  
  // 插入后
  @AfterInsert()
  logInsert() {
    console.log('Inserted user with id:', this.id);
  }
  
  // 加载后
  @AfterLoad()
  computeFullName() {
    this.fullName = `${this.firstName} ${this.lastName}`;
  }
}
```

### 全局订阅者

```typescript
import {
  EventSubscriber,
  EntitySubscriberInterface,
  InsertEvent,
  UpdateEvent,
  RemoveEvent
} from 'typeorm';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  // 指定监听的实体
  listenTo() {
    return User;
  }
  
  // 插入前
  beforeInsert(event: InsertEvent<User>) {
    console.log('Before user insert:', event.entity);
  }
  
  // 插入后
  afterInsert(event: InsertEvent<User>) {
    console.log('After user insert:', event.entity);
  }
  
  // 更新前
  beforeUpdate(event: UpdateEvent<User>) {
    console.log('Before user update:', event.entity);
  }
  
  // 更新后
  afterUpdate(event: UpdateEvent<User>) {
    console.log('After user update:', event.entity);
  }
  
  // 删除前
  beforeRemove(event: RemoveEvent<User>) {
    console.log('Before user remove:', event.entity);
  }
  
  // 删除后
  afterRemove(event: RemoveEvent<User>) {
    console.log('After user remove:', event.entity);
  }
}
```

## 迁移

```bash
# 创建迁移
npx typeorm migration:create -n CreateUserTable

# 生成迁移（根据 Entity 变化）
npx typeorm migration:generate -n UpdateUserTable

# 运行迁移
npx typeorm migration:run

# 回滚迁移
npx typeorm migration:revert

# 显示所有迁移
npx typeorm migration:show
```

### 迁移文件

```typescript
import { MigrationInterface, QueryRunner, Table, TableForeignKey } from 'typeorm';

export class CreateUserTable1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'users',
        columns: [
          {
            name: 'id',
            type: 'int',
            isPrimary: true,
            isGenerated: true,
            generationStrategy: 'increment'
          },
          {
            name: 'email',
            type: 'varchar',
            length: '255',
            isUnique: true
          },
          {
            name: 'name',
            type: 'varchar',
            length: '255',
            isNullable: true
          },
          {
            name: 'created_at',
            type: 'timestamp',
            default: 'now()'
          }
        ]
      }),
      true
    );
    
    // 添加索引
    await queryRunner.createIndex('users', {
      name: 'IDX_USERS_EMAIL',
      columnNames: ['email']
    });
    
    // 添加外键
    await queryRunner.createForeignKey('posts', new TableForeignKey({
      columnNames: ['author_id'],
      referencedTableName: 'users',
      referencedColumnNames: ['id'],
      onDelete: 'CASCADE'
    }));
  }
  
  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('users');
  }
}
```

## 与 NestJS 集成

```typescript
// app.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'postgres',
      password: 'password',
      database: 'mydb',
      entities: [User, Post, Profile],
      synchronize: false,  // 生产环境必须 false
      logging: true,
      // 连接池配置
      extra: {
        max: 10,
        min: 2,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 2000
      }
    }),
    TypeOrmModule.forFeature([User, Post])
  ]
})
export class AppModule {}

// user.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User, Profile])],
  providers: [UserService],
  controllers: [UserController]
})
export class UserModule {}

// user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
    
    @InjectRepository(Profile)
    private profileRepository: Repository<Profile>
  ) {}
  
  async findAll(): Promise<User[]> {
    return this.userRepository.find({
      relations: ['profile', 'posts']
    });
  }
  
  async create(data: Partial<User>): Promise<User> {
    const user = this.userRepository.create(data);
    return this.userRepository.save(user);
  }
}
```

## 性能优化

### 1. 查询优化

```typescript
// ❌ N+1 问题
const users = await userRepository.find();
for (const user of users) {
  user.posts = await postRepository.find({
    where: { authorId: user.id }
  });
}

// ✅ 使用 relations
const users = await userRepository.find({
  relations: ['posts']
});

// ✅ 使用 Query Builder + JOIN
const users = await userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .getMany();

// ✅ 只查询需要的字段
const users = await userRepository.find({
  select: ['id', 'name', 'email']
});
```

### 2. 批量操作

```typescript
// ❌ 循环插入
for (const userData of users) {
  await userRepository.save(userData);
}

// ✅ 批量插入
await userRepository
  .createQueryBuilder()
  .insert()
  .into(User)
  .values(users)
  .execute();

// ✅ 批量更新
await userRepository
  .createQueryBuilder()
  .update(User)
  .set({ isActive: false })
  .where('lastLoginAt < :date', { date: oneYearAgo })
  .execute();
```

### 3. 缓存

```typescript
// 查询缓存
const users = await userRepository.find({
  where: { isActive: true },
  cache: true  // 使用默认缓存
});

// 自定义缓存时间
const users = await userRepository.find({
  where: { isActive: true },
  cache: {
    id: 'users_active',
    milliseconds: 60000  // 1 分钟
  }
});

// Query Builder 缓存
const users = await userRepository
  .createQueryBuilder('user')
  .where('user.isActive = :isActive', { isActive: true })
  .cache('users_active', 60000)
  .getMany();

// 清除缓存
await connection.queryResultCache.clear();
await connection.queryResultCache.remove(['users_active']);
```

### 4. 索引

```typescript
@Entity()
@Index(['email'])  // 单列索引
@Index(['firstName', 'lastName'])  // 复合索引
@Index('IDX_USER_EMAIL_UNIQUE', ['email'], { unique: true })  // 唯一索引
export class User {
  @Column()
  @Index()  // 字段级索引
  email: string;
  
  @Column()
  @Index({ fulltext: true })  // 全文索引（MySQL）
  bio: string;
}
```

## 常见面试题

### 1. TypeORM 的生命周期钩子有哪些？

<details>
<summary>点击查看答案</summary>

**Entity 监听器**：
- `@BeforeInsert()`：插入前
- `@AfterInsert()`：插入后
- `@BeforeUpdate()`：更新前
- `@AfterUpdate()`：更新后
- `@BeforeRemove()`：删除前
- `@AfterRemove()`：删除后
- `@AfterLoad()`：从数据库加载后

**使用场景**：
- 密码加密（BeforeInsert/BeforeUpdate）
- 审计日志（After钩子）
- 计算字段（AfterLoad）
- 数据验证（Before钩子）
</details>

### 2. 如何优化 TypeORM 性能？

<details>
<summary>点击查看答案</summary>

1. **避免 N+1 查询**：
   - 使用 `relations` 或 `leftJoinAndSelect`
   - 预加载关联数据

2. **使用 QueryBuilder**：
   - 更精细的查询控制
   - 更好的性能

3. **批量操作**：
   - `createQueryBuilder().insert().values([...])`
   - 避免循环单条操作

4. **启用查询缓存**：
   - 配置 Redis 缓存
   - 为常用查询添加缓存

5. **添加索引**：
   - 为查询字段添加索引
   - 使用复合索引

6. **选择必要字段**：
   - 使用 `select` 只查询需要的字段
   - 减少数据传输

7. **连接池配置**：
   - 合理设置 `max`、`min` 连接数
   - 配置超时参数
</details>

### 3. TypeORM 的级联操作有哪些？

<details>
<summary>点击查看答案</summary>

```typescript
@Entity()
export class User {
  @OneToMany(() => Post, post => post.author, {
    cascade: true,  // 或指定具体操作
    // cascade: ['insert', 'update', 'remove']
  })
  posts: Post[];
}

// 使用
const user = await userRepository.save({
  name: 'John',
  posts: [
    { title: 'Post 1' },
    { title: 'Post 2' }
  ]
});
// posts 会自动保存

// 删除用户时
await userRepository.remove(user);
// posts 也会被删除（如果设置了 cascade: true）
```

**级联选项**：
- `cascade: true`：所有操作
- `cascade: ['insert']`：只级联插入
- `cascade: ['update']`：只级联更新
- `cascade: ['remove']`：只级联删除

**注意**：
- 谨慎使用 `cascade: true`
- 可能导致意外删除
- 建议明确指定级联操作
</details>

### 4. TypeORM vs Prisma 如何选择？

<details>
<summary>点击查看答案</summary>

**TypeORM**：
- ✅ 成熟稳定，社区大
- ✅ 支持多种数据库
- ✅ 功能丰富（QueryBuilder、Raw SQL）
- ✅ Active Record 和 Data Mapper 模式
- ❌ TypeScript 类型推断较弱
- ❌ 性能一般

**Prisma**：
- ✅ 类型安全，开发体验好
- ✅ 性能优秀
- ✅ 自动生成类型
- ✅ Prisma Studio 可视化工具
- ❌ 灵活性略低
- ❌ 社区相对较小

**选择建议**：
- 新项目 → Prisma（类型安全 + 性能）
- 已有项目 → TypeORM（成熟稳定）
- 需要复杂 SQL → TypeORM（QueryBuilder）
- 追求开发体验 → Prisma
</details>

## 最佳实践

1. **生产环境关闭 `synchronize`**
2. **使用迁移管理数据库结构**
3. **避免 N+1 查询，使用 eager loading**
4. **合理使用级联操作**
5. **为查询字段添加索引**
6. **使用事务保证数据一致性**
7. **配置合适的连接池大小**
8. **敏感字段使用 `select: false`**
9. **使用 QueryBuilder 处理复杂查询**
10. **监控慢查询，启用查询日志**

