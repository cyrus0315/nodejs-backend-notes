# GraphQL API 设计

GraphQL 是一种用于 API 的查询语言和运行时。它提供了一种更高效、强大和灵活的替代 REST 的方式。

## 目录
- [GraphQL 基础](#graphql-基础)
- [Schema 定义](#schema-定义)
- [Resolver 实现](#resolver-实现)
- [查询与变更](#查询与变更)
- [N+1 问题](#n1-问题)
- [认证与授权](#认证与授权)
- [错误处理](#错误处理)
- [性能优化](#性能优化)
- [GraphQL vs REST](#graphql-vs-rest)
- [面试题](#常见面试题)

---

## GraphQL 基础

### 核心概念

```graphql
# Schema：定义数据结构和操作
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

# Query：读取操作
type Query {
  user(id: ID!): User
  users: [User!]!
  post(id: ID!): Post
}

# Mutation：写入操作
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

# Subscription：实时订阅
type Subscription {
  userCreated: User!
  postCreated: Post!
}
```

### 查询示例

```graphql
# 查询用户及其文章
query {
  user(id: "123") {
    id
    name
    email
    posts {
      id
      title
    }
  }
}

# 响应
{
  "data": {
    "user": {
      "id": "123",
      "name": "John Doe",
      "email": "john@example.com",
      "posts": [
        {
          "id": "1",
          "title": "My First Post"
        },
        {
          "id": "2",
          "title": "GraphQL Tutorial"
        }
      ]
    }
  }
}
```

### GraphQL 优势

- ✅ **按需查询**：客户端精确指定需要的字段
- ✅ **单次请求**：一次查询获取所有相关数据
- ✅ **强类型**：Schema 定义完整的类型系统
- ✅ **API 演化**：添加字段不影响现有查询
- ✅ **实时数据**：内置 Subscription 支持

---

## Schema 定义

### 基本类型

```graphql
# 标量类型
scalar Date
scalar JSON

type User {
  # ID 类型
  id: ID!
  
  # 基本类型
  name: String!
  email: String!
  age: Int
  isActive: Boolean!
  
  # 自定义标量
  createdAt: Date!
  metadata: JSON
  
  # 枚举
  role: UserRole!
  
  # 对象类型
  profile: Profile
  
  # 列表
  posts: [Post!]!
  tags: [String!]
}

# 枚举类型
enum UserRole {
  ADMIN
  USER
  GUEST
}

# 接口
interface Node {
  id: ID!
}

# 实现接口
type User implements Node {
  id: ID!
  name: String!
}

# 联合类型
union SearchResult = User | Post | Comment

# 输入类型
input CreateUserInput {
  name: String!
  email: String!
  password: String!
}

input UpdateUserInput {
  name: String
  email: String
  bio: String
}
```

### TypeScript 实现

```typescript
import { GraphQLScalarType, Kind } from 'graphql';

// 自定义标量类型：Date
const DateScalar = new GraphQLScalarType({
  name: 'Date',
  description: 'Date custom scalar type',
  
  // 序列化（发送给客户端）
  serialize(value: Date) {
    return value.toISOString();
  },
  
  // 解析值（来自客户端）
  parseValue(value: string) {
    return new Date(value);
  },
  
  // 解析字面量
  parseLiteral(ast) {
    if (ast.kind === Kind.STRING) {
      return new Date(ast.value);
    }
    return null;
  }
});

// 使用 type-graphql
import { ObjectType, Field, ID, Int, registerEnumType } from 'type-graphql';

enum UserRole {
  ADMIN = 'ADMIN',
  USER = 'USER',
  GUEST = 'GUEST'
}

registerEnumType(UserRole, {
  name: 'UserRole',
  description: 'User role'
});

@ObjectType()
class User {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;

  @Field()
  email: string;

  @Field(() => Int, { nullable: true })
  age?: number;

  @Field()
  isActive: boolean;

  @Field(() => UserRole)
  role: UserRole;

  @Field(() => Date)
  createdAt: Date;

  @Field(() => [Post])
  posts: Post[];
}

@ObjectType()
class Post {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;

  @Field()
  content: string;

  @Field(() => User)
  author: User;

  @Field(() => Date)
  createdAt: Date;
}
```

---

## Resolver 实现

### Query Resolver

```typescript
import { Query, Resolver, Arg, ID, Ctx } from 'type-graphql';
import { PrismaClient } from '@prisma/client';

interface Context {
  prisma: PrismaClient;
  user?: User;
}

@Resolver()
class UserResolver {
  @Query(() => User, { nullable: true })
  async user(
    @Arg('id', () => ID) id: string,
    @Ctx() ctx: Context
  ): Promise<User | null> {
    return await ctx.prisma.user.findUnique({
      where: { id }
    });
  }

  @Query(() => [User])
  async users(
    @Ctx() ctx: Context
  ): Promise<User[]> {
    return await ctx.prisma.user.findMany();
  }

  @Query(() => [User])
  async searchUsers(
    @Arg('query') query: string,
    @Ctx() ctx: Context
  ): Promise<User[]> {
    return await ctx.prisma.user.findMany({
      where: {
        OR: [
          { name: { contains: query, mode: 'insensitive' } },
          { email: { contains: query, mode: 'insensitive' } }
        ]
      }
    });
  }
}
```

### Mutation Resolver

```typescript
import { Mutation, Resolver, Arg, Ctx, InputType, Field } from 'type-graphql';
import * as bcrypt from 'bcryptjs';

@InputType()
class CreateUserInput {
  @Field()
  name: string;

  @Field()
  email: string;

  @Field()
  password: string;
}

@InputType()
class UpdateUserInput {
  @Field({ nullable: true })
  name?: string;

  @Field({ nullable: true })
  email?: string;

  @Field({ nullable: true })
  bio?: string;
}

@Resolver()
class UserMutationResolver {
  @Mutation(() => User)
  async createUser(
    @Arg('input') input: CreateUserInput,
    @Ctx() ctx: Context
  ): Promise<User> {
    // 验证邮箱唯一性
    const existing = await ctx.prisma.user.findUnique({
      where: { email: input.email }
    });

    if (existing) {
      throw new Error('Email already exists');
    }

    // 哈希密码
    const hashedPassword = await bcrypt.hash(input.password, 10);

    // 创建用户
    return await ctx.prisma.user.create({
      data: {
        name: input.name,
        email: input.email,
        password: hashedPassword
      }
    });
  }

  @Mutation(() => User)
  async updateUser(
    @Arg('id', () => ID) id: string,
    @Arg('input') input: UpdateUserInput,
    @Ctx() ctx: Context
  ): Promise<User> {
    return await ctx.prisma.user.update({
      where: { id },
      data: input
    });
  }

  @Mutation(() => Boolean)
  async deleteUser(
    @Arg('id', () => ID) id: string,
    @Ctx() ctx: Context
  ): Promise<boolean> {
    await ctx.prisma.user.delete({
      where: { id }
    });
    return true;
  }
}
```

### Field Resolver

```typescript
import { Resolver, FieldResolver, Root, Ctx } from 'type-graphql';

@Resolver(() => User)
class UserFieldResolver {
  // 解析 posts 字段
  @FieldResolver(() => [Post])
  async posts(
    @Root() user: User,
    @Ctx() ctx: Context
  ): Promise<Post[]> {
    return await ctx.prisma.post.findMany({
      where: { authorId: user.id }
    });
  }

  // 计算字段
  @FieldResolver(() => Int)
  async postCount(
    @Root() user: User,
    @Ctx() ctx: Context
  ): Promise<number> {
    return await ctx.prisma.post.count({
      where: { authorId: user.id }
    });
  }

  // 复杂计算
  @FieldResolver(() => String)
  fullName(@Root() user: User): string {
    return `${user.firstName} ${user.lastName}`;
  }
}
```

---

## 查询与变更

### 复杂查询

```graphql
# 带参数的查询
query GetUser($id: ID!) {
  user(id: $id) {
    id
    name
    email
    posts(first: 10, orderBy: CREATED_AT_DESC) {
      edges {
        node {
          id
          title
          createdAt
        }
        cursor
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
  }
}

# 查询片段
fragment UserInfo on User {
  id
  name
  email
  createdAt
}

query {
  user(id: "123") {
    ...UserInfo
    posts {
      id
      title
    }
  }
}

# 别名
query {
  admin: user(id: "1") {
    id
    name
  }
  regularUser: user(id: "2") {
    id
    name
  }
}

# 指令
query GetUser($includeEmail: Boolean!) {
  user(id: "123") {
    id
    name
    email @include(if: $includeEmail)
  }
}
```

### 实现分页

```typescript
import { ObjectType, Field, Int } from 'type-graphql';

// Cursor 分页（推荐）
@ObjectType()
class PageInfo {
  @Field()
  hasNextPage: boolean;

  @Field()
  hasPreviousPage: boolean;

  @Field({ nullable: true })
  startCursor?: string;

  @Field({ nullable: true })
  endCursor?: string;
}

@ObjectType()
class PostEdge {
  @Field()
  cursor: string;

  @Field(() => Post)
  node: Post;
}

@ObjectType()
class PostConnection {
  @Field(() => [PostEdge])
  edges: PostEdge[];

  @Field(() => PageInfo)
  pageInfo: PageInfo;

  @Field(() => Int)
  totalCount: number;
}

@Resolver()
class PostResolver {
  @Query(() => PostConnection)
  async posts(
    @Arg('first', () => Int, { nullable: true }) first: number = 20,
    @Arg('after', { nullable: true }) after?: string,
    @Ctx() ctx: Context
  ): Promise<PostConnection> {
    const limit = Math.min(first, 100); // 最多 100 条

    const posts = await ctx.prisma.post.findMany({
      take: limit + 1,
      ...(after && {
        cursor: { id: after },
        skip: 1
      }),
      orderBy: { createdAt: 'desc' }
    });

    const hasNextPage = posts.length > limit;
    const nodes = hasNextPage ? posts.slice(0, -1) : posts;

    const edges = nodes.map(node => ({
      cursor: node.id,
      node
    }));

    const totalCount = await ctx.prisma.post.count();

    return {
      edges,
      pageInfo: {
        hasNextPage,
        hasPreviousPage: !!after,
        startCursor: edges[0]?.cursor,
        endCursor: edges[edges.length - 1]?.cursor
      },
      totalCount
    };
  }
}
```

### Subscription

```typescript
import { Subscription, Root, Resolver, PubSub, PubSubEngine } from 'type-graphql';

@Resolver()
class UserSubscriptionResolver {
  @Subscription(() => User, {
    topics: 'USER_CREATED'
  })
  userCreated(@Root() user: User): User {
    return user;
  }

  @Subscription(() => Post, {
    topics: 'POST_CREATED',
    filter: ({ payload, args }) => {
      // 过滤：只订阅特定用户的文章
      return payload.authorId === args.userId;
    }
  })
  postCreated(
    @Arg('userId', () => ID) userId: string,
    @Root() post: Post
  ): Post {
    return post;
  }
}

// 触发 Subscription
@Resolver()
class UserMutationResolver {
  @Mutation(() => User)
  async createUser(
    @Arg('input') input: CreateUserInput,
    @Ctx() ctx: Context,
    @PubSub() pubSub: PubSubEngine
  ): Promise<User> {
    const user = await ctx.prisma.user.create({
      data: input
    });

    // 发布事件
    await pubSub.publish('USER_CREATED', user);

    return user;
  }
}
```

---

## N+1 问题

### 问题示例

```graphql
query {
  users {
    id
    name
    posts {  # N+1 问题！
      id
      title
    }
  }
}
```

```typescript
// ❌ 有 N+1 问题
@Resolver(() => User)
class UserResolver {
  @FieldResolver(() => [Post])
  async posts(@Root() user: User, @Ctx() ctx: Context): Promise<Post[]> {
    // 每个用户都会执行一次查询
    return await ctx.prisma.post.findMany({
      where: { authorId: user.id }
    });
  }
}
// 如果有 100 个用户，会执行 1 + 100 = 101 次查询
```

### 解决方案：DataLoader

```typescript
import DataLoader from 'dataloader';

// 创建 DataLoader
function createPostsByUserIdLoader(prisma: PrismaClient) {
  return new DataLoader<string, Post[]>(async (userIds) => {
    // 批量查询所有用户的文章
    const posts = await prisma.post.findMany({
      where: {
        authorId: { in: userIds as string[] }
      }
    });

    // 按 userId 分组
    const postsByUserId = new Map<string, Post[]>();
    for (const post of posts) {
      const userPosts = postsByUserId.get(post.authorId) || [];
      userPosts.push(post);
      postsByUserId.set(post.authorId, userPosts);
    }

    // 返回结果，顺序必须与 userIds 一致
    return userIds.map(userId => postsByUserId.get(userId) || []);
  });
}

// Context 中添加 DataLoader
interface Context {
  prisma: PrismaClient;
  loaders: {
    postsByUserId: DataLoader<string, Post[]>;
    userById: DataLoader<string, User | null>;
  };
}

function createContext(): Context {
  const prisma = new PrismaClient();
  
  return {
    prisma,
    loaders: {
      postsByUserId: createPostsByUserIdLoader(prisma),
      userById: createUserByIdLoader(prisma)
    }
  };
}

// ✅ 使用 DataLoader
@Resolver(() => User)
class UserResolver {
  @FieldResolver(() => [Post])
  async posts(@Root() user: User, @Ctx() ctx: Context): Promise<Post[]> {
    // 使用 DataLoader，自动批量查询
    return await ctx.loaders.postsByUserId.load(user.id);
  }
}

// 100 个用户只执行 1 + 1 = 2 次查询
```

### DataLoader 完整示例

```typescript
import DataLoader from 'dataloader';
import { PrismaClient } from '@prisma/client';

class DataLoaders {
  userById: DataLoader<string, User | null>;
  postsByUserId: DataLoader<string, Post[]>;
  commentsByPostId: DataLoader<string, Comment[]>;

  constructor(private prisma: PrismaClient) {
    this.userById = this.createUserByIdLoader();
    this.postsByUserId = this.createPostsByUserIdLoader();
    this.commentsByPostId = this.createCommentsByPostIdLoader();
  }

  private createUserByIdLoader() {
    return new DataLoader<string, User | null>(async (ids) => {
      const users = await this.prisma.user.findMany({
        where: { id: { in: ids as string[] } }
      });

      const userMap = new Map(users.map(u => [u.id, u]));
      return ids.map(id => userMap.get(id) || null);
    });
  }

  private createPostsByUserIdLoader() {
    return new DataLoader<string, Post[]>(async (userIds) => {
      const posts = await this.prisma.post.findMany({
        where: { authorId: { in: userIds as string[] } }
      });

      const postsByUserId = new Map<string, Post[]>();
      for (const post of posts) {
        const userPosts = postsByUserId.get(post.authorId) || [];
        userPosts.push(post);
        postsByUserId.set(post.authorId, userPosts);
      }

      return userIds.map(userId => postsByUserId.get(userId) || []);
    });
  }

  private createCommentsByPostIdLoader() {
    return new DataLoader<string, Comment[]>(async (postIds) => {
      const comments = await this.prisma.comment.findMany({
        where: { postId: { in: postIds as string[] } }
      });

      const commentsByPostId = new Map<string, Comment[]>();
      for (const comment of comments) {
        const postComments = commentsByPostId.get(comment.postId) || [];
        postComments.push(comment);
        commentsByPostId.set(comment.postId, postComments);
      }

      return postIds.map(postId => commentsByPostId.get(postId) || []);
    });
  }
}

// 每个请求创建新的 DataLoader 实例
function createContext(): Context {
  const prisma = new PrismaClient();
  const loaders = new DataLoaders(prisma);

  return { prisma, loaders };
}
```

---

## 认证与授权

### JWT 认证

```typescript
import jwt from 'jsonwebtoken';
import { AuthChecker } from 'type-graphql';

// 认证检查器
const authChecker: AuthChecker<Context> = ({ context }, roles) => {
  if (!context.user) {
    return false; // 未认证
  }

  if (roles.length === 0) {
    return true; // 只需要认证
  }

  // 检查角色
  return roles.includes(context.user.role);
};

// 从 token 解析用户
async function getUserFromToken(token: string): Promise<User | null> {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;
    return await prisma.user.findUnique({
      where: { id: decoded.userId }
    });
  } catch {
    return null;
  }
}

// 创建 Context
async function createContext({ req }): Promise<Context> {
  const token = req.headers.authorization?.replace('Bearer ', '');
  const user = token ? await getUserFromToken(token) : null;

  return {
    prisma: new PrismaClient(),
    user,
    loaders: new DataLoaders(prisma)
  };
}
```

### 使用装饰器保护

```typescript
import { Resolver, Query, Mutation, Authorized, Ctx } from 'type-graphql';

@Resolver()
class UserResolver {
  // 需要认证
  @Authorized()
  @Query(() => User)
  async me(@Ctx() ctx: Context): Promise<User> {
    return ctx.user!;
  }

  // 需要特定角色
  @Authorized(['ADMIN'])
  @Query(() => [User])
  async allUsers(@Ctx() ctx: Context): Promise<User[]> {
    return await ctx.prisma.user.findMany();
  }

  // 需要 ADMIN 或 MODERATOR 角色
  @Authorized(['ADMIN', 'MODERATOR'])
  @Mutation(() => Boolean)
  async deleteUser(
    @Arg('id', () => ID) id: string,
    @Ctx() ctx: Context
  ): Promise<boolean> {
    await ctx.prisma.user.delete({ where: { id } });
    return true;
  }
}
```

### 字段级授权

```typescript
@Resolver(() => User)
class UserResolver {
  @FieldResolver(() => String, { nullable: true })
  email(@Root() user: User, @Ctx() ctx: Context): string | null {
    // 只有自己或管理员可以看到邮箱
    if (ctx.user?.id === user.id || ctx.user?.role === 'ADMIN') {
      return user.email;
    }
    return null;
  }

  @FieldResolver(() => String, { nullable: true })
  phone(@Root() user: User, @Ctx() ctx: Context): string | null {
    // 只有自己可以看到电话
    if (ctx.user?.id === user.id) {
      return user.phone;
    }
    return null;
  }
}
```

---

## 错误处理

### 自定义错误

```typescript
import { GraphQLError } from 'graphql';

class AuthenticationError extends GraphQLError {
  constructor(message = 'Not authenticated') {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED'
      }
    });
  }
}

class ForbiddenError extends GraphQLError {
  constructor(message = 'Forbidden') {
    super(message, {
      extensions: {
        code: 'FORBIDDEN'
      }
    });
  }
}

class ValidationError extends GraphQLError {
  constructor(message: string, fields?: Record<string, string>) {
    super(message, {
      extensions: {
        code: 'VALIDATION_ERROR',
        fields
      }
    });
  }
}

// 使用
@Mutation(() => User)
async createUser(
  @Arg('input') input: CreateUserInput,
  @Ctx() ctx: Context
): Promise<User> {
  const existing = await ctx.prisma.user.findUnique({
    where: { email: input.email }
  });

  if (existing) {
    throw new ValidationError('Validation failed', {
      email: 'Email already exists'
    });
  }

  return await ctx.prisma.user.create({ data: input });
}
```

### 错误响应格式

```json
{
  "errors": [
    {
      "message": "Validation failed",
      "extensions": {
        "code": "VALIDATION_ERROR",
        "fields": {
          "email": "Email already exists"
        }
      },
      "path": ["createUser"],
      "locations": [{ "line": 2, "column": 3 }]
    }
  ],
  "data": null
}
```

### 全局错误处理

```typescript
import { ApolloServer } from '@apollo/server';
import { formatError as defaultFormatError } from 'graphql';

const server = new ApolloServer({
  schema,
  formatError: (formattedError, error) => {
    // 记录错误
    console.error('GraphQL Error:', {
      message: formattedError.message,
      path: formattedError.path,
      extensions: formattedError.extensions
    });

    // 生产环境隐藏内部错误
    if (process.env.NODE_ENV === 'production') {
      if (formattedError.extensions?.code === 'INTERNAL_SERVER_ERROR') {
        return {
          message: 'Internal server error',
          extensions: {
            code: 'INTERNAL_SERVER_ERROR'
          }
        };
      }
    }

    return formattedError;
  }
});
```

---

## 性能优化

### 1. 查询复杂度限制

```typescript
import { createComplexityLimitRule } from 'graphql-validation-complexity';

const server = new ApolloServer({
  schema,
  validationRules: [
    createComplexityLimitRule(1000, {
      onCost: (cost) => {
        console.log('Query cost:', cost);
      }
    })
  ]
});
```

### 2. 查询深度限制

```typescript
import depthLimit from 'graphql-depth-limit';

const server = new ApolloServer({
  schema,
  validationRules: [depthLimit(5)] // 最多 5 层嵌套
});
```

### 3. 缓存

```typescript
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';

const redis = new Redis();

// 缓存装饰器
function Cached(ttl: number = 60) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: any[]) {
      const cacheKey = `${propertyKey}:${JSON.stringify(args)}`;

      // 查缓存
      const cached = await redis.get(cacheKey);
      if (cached) {
        return JSON.parse(cached);
      }

      // 执行方法
      const result = await originalMethod.apply(this, args);

      // 写缓存
      await redis.setex(cacheKey, ttl, JSON.stringify(result));

      return result;
    };

    return descriptor;
  };
}

// 使用
@Resolver()
class UserResolver {
  @Cached(300) // 缓存 5 分钟
  @Query(() => [User])
  async users(@Ctx() ctx: Context): Promise<User[]> {
    return await ctx.prisma.user.findMany();
  }
}
```

### 4. Persisted Queries

```typescript
// 客户端发送查询哈希而不是完整查询
import { ApolloServer } from '@apollo/server';
import { ApolloServerPluginLandingPageDisabled } from '@apollo/server/plugin/disabled';

const server = new ApolloServer({
  schema,
  persistedQueries: {
    cache: new Map() // 使用 Redis 更好
  }
});

// 客户端配置
import { createPersistedQueryLink } from '@apollo/client/link/persisted-queries';
import { sha256 } from 'crypto-hash';

const link = createPersistedQueryLink({ sha256 }).concat(httpLink);
```

---

## GraphQL vs REST

### 对比

| 特性 | GraphQL | REST |
|------|---------|------|
| **数据获取** | 精确获取需要的数据 | 可能过度获取或获取不足 |
| **请求次数** | 单次请求获取所有数据 | 多次请求 |
| **API 演化** | 添加字段不影响现有查询 | 需要版本控制 |
| **学习曲线** | 较陡 | 较平缓 |
| **缓存** | 需要额外配置 | HTTP 缓存原生支持 |
| **错误处理** | 200 + errors 字段 | HTTP 状态码 |
| **文件上传** | 需要额外处理 | 原生支持 |
| **实时数据** | 内置 Subscription | 需要 WebSocket |

### 何时使用 GraphQL

✅ **适合**：
- 移动应用（减少请求次数）
- 复杂的数据关系
- 多个客户端（Web、移动、IoT）
- 频繁变化的需求
- 需要精确控制数据

❌ **不适合**：
- 简单的 CRUD
- 文件上传为主
- 需要 HTTP 缓存
- 团队缺乏 GraphQL 经验

---

## 常见面试题

### 1. GraphQL 的优缺点？

**优点**：
- ✅ 避免过度获取/获取不足
- ✅ 单次请求获取所有数据
- ✅ 强类型系统
- ✅ API 演化无需版本控制
- ✅ 实时数据支持（Subscription）

**缺点**：
- ❌ 学习曲线较陡
- ❌ 缓存复杂
- ❌ 文件上传麻烦
- ❌ 查询复杂度难以控制
- ❌ 错误处理不如 REST 直观

### 2. 如何解决 N+1 问题？

**回答**：使用 DataLoader

```typescript
// DataLoader 会自动：
// 1. 收集同一事件循环中的所有请求
// 2. 批量查询
// 3. 缓存结果

const loader = new DataLoader(async (ids) => {
  const users = await prisma.user.findMany({
    where: { id: { in: ids } }
  });
  return ids.map(id => users.find(u => u.id === id));
});
```

### 3. GraphQL 如何做认证授权？

**回答**：

1. **认证**：在 Context 中解析 token
```typescript
async function createContext({ req }) {
  const token = req.headers.authorization;
  const user = await verifyToken(token);
  return { user };
}
```

2. **授权**：使用 @Authorized 装饰器
```typescript
@Authorized(['ADMIN'])
@Query(() => [User])
async allUsers() { }
```

3. **字段级授权**：在 Field Resolver 中检查
```typescript
@FieldResolver(() => String)
email(@Root() user, @Ctx() ctx) {
  if (ctx.user.id !== user.id) return null;
  return user.email;
}
```

### 4. GraphQL 如何处理分页？

**回答**：推荐使用 Relay 风格的 Cursor 分页

```graphql
type Query {
  posts(first: Int, after: String): PostConnection
}

type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}
```

### 5. GraphQL vs REST，如何选择？

| 场景 | 推荐 |
|------|------|
| 复杂数据关系 | GraphQL |
| 移动应用 | GraphQL |
| 简单 CRUD | REST |
| 文件上传 | REST |
| 需要 HTTP 缓存 | REST |
| 微服务 | REST |
| 公共 API | REST |
| 内部 API | GraphQL |

---

## 总结

### GraphQL 核心要点

1. **Schema First**：先定义 Schema
2. **强类型**：完整的类型系统
3. **Resolver**：每个字段都有 Resolver
4. **DataLoader**：解决 N+1 问题
5. **认证授权**：在 Context 和 Resolver 中实现
6. **错误处理**：使用自定义错误类
7. **性能优化**：查询复杂度限制、缓存

### 实践检查清单

- [ ] Schema 定义是否清晰？
- [ ] 是否使用 DataLoader 解决 N+1？
- [ ] 是否有认证授权？
- [ ] 是否有错误处理？
- [ ] 是否限制了查询复杂度？
- [ ] 是否实现了分页？
- [ ] 是否有性能监控？
- [ ] 是否有完善的文档？

---

**上一篇**：[RESTful API](./01-restful-api.md)  
**下一篇**：[API 文档与版本控制](./03-api-docs-versioning.md)

