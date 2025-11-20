# RESTful API 设计

REST（Representational State Transfer）是目前最流行的 API 设计风格。本文深入讲解 RESTful API 的设计原则、最佳实践和 Node.js 实现。

## 目录
- [REST 基础](#rest-基础)
- [HTTP 方法](#http-方法)
- [状态码](#状态码)
- [URL 设计](#url-设计)
- [请求与响应](#请求与响应)
- [分页排序过滤](#分页排序过滤)
- [错误处理](#错误处理)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## REST 基础

### REST 六大约束

1. **客户端-服务器分离**：前后端独立开发和部署
2. **无状态**：每个请求包含所有必要信息
3. **可缓存**：响应可被缓存
4. **统一接口**：使用标准 HTTP 方法
5. **分层系统**：客户端不知道是否直接连接到服务器
6. **按需代码**（可选）：服务器可以返回可执行代码

### 资源和表现

```typescript
// 资源（Resource）：名词，用 URL 表示
GET /users          // 用户集合
GET /users/123      // 单个用户
GET /users/123/posts // 用户的文章

// 表现（Representation）：资源的表现形式
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```

---

## HTTP 方法

### 标准方法

| 方法 | 含义 | 幂等性 | 安全性 | 使用场景 |
|------|------|--------|--------|---------|
| **GET** | 获取资源 | ✅ | ✅ | 查询数据 |
| **POST** | 创建资源 | ❌ | ❌ | 创建数据 |
| **PUT** | 替换资源 | ✅ | ❌ | 完整更新 |
| **PATCH** | 部分更新 | ❌ | ❌ | 部分更新 |
| **DELETE** | 删除资源 | ✅ | ❌ | 删除数据 |
| **HEAD** | 获取元信息 | ✅ | ✅ | 检查资源 |
| **OPTIONS** | 获取支持的方法 | ✅ | ✅ | CORS 预检 |

### 详细说明

#### GET - 获取资源

```typescript
import express from 'express';
import { PrismaClient } from '@prisma/client';

const app = express();
const prisma = new PrismaClient();

// 获取用户列表
app.get('/api/users', async (req, res) => {
  const users = await prisma.user.findMany({
    select: {
      id: true,
      name: true,
      email: true,
      // 不返回密码等敏感信息
    }
  });

  res.json({
    data: users,
    total: users.length
  });
});

// 获取单个用户
app.get('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  const user = await prisma.user.findUnique({
    where: { id: parseInt(id) },
    include: {
      posts: true,  // 包含关联数据
      profile: true
    }
  });

  if (!user) {
    return res.status(404).json({
      error: 'User not found'
    });
  }

  res.json({ data: user });
});
```

#### POST - 创建资源

```typescript
import { z } from 'zod';

// 请求验证 Schema
const createUserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  password: z.string().min(8)
});

app.post('/api/users', async (req, res) => {
  try {
    // 1. 验证请求数据
    const data = createUserSchema.parse(req.body);

    // 2. 检查邮箱是否已存在
    const existing = await prisma.user.findUnique({
      where: { email: data.email }
    });

    if (existing) {
      return res.status(409).json({
        error: 'Email already exists'
      });
    }

    // 3. 创建用户
    const user = await prisma.user.create({
      data: {
        name: data.name,
        email: data.email,
        password: await hashPassword(data.password)
      },
      select: {
        id: true,
        name: true,
        email: true,
        createdAt: true
      }
    });

    // 4. 返回 201 Created
    res.status(201)
      .location(`/api/users/${user.id}`)  // Location 头
      .json({ data: user });

  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      });
    }
    throw error;
  }
});
```

#### PUT - 完整替换

```typescript
const updateUserSchema = z.object({
  name: z.string().min(2).max(50),
  email: z.string().email(),
  bio: z.string().optional(),
  avatar: z.string().url().optional()
});

// PUT 要求提供完整的资源表示
app.put('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  try {
    const data = updateUserSchema.parse(req.body);

    const user = await prisma.user.update({
      where: { id: parseInt(id) },
      data: {
        name: data.name,
        email: data.email,
        bio: data.bio,
        avatar: data.avatar,
        updatedAt: new Date()
      },
      select: {
        id: true,
        name: true,
        email: true,
        bio: true,
        avatar: true,
        updatedAt: true
      }
    });

    res.json({ data: user });

  } catch (error) {
    if (error.code === 'P2025') {
      return res.status(404).json({
        error: 'User not found'
      });
    }
    throw error;
  }
});
```

#### PATCH - 部分更新

```typescript
const patchUserSchema = z.object({
  name: z.string().min(2).max(50).optional(),
  email: z.string().email().optional(),
  bio: z.string().optional(),
  avatar: z.string().url().optional()
}).refine(data => Object.keys(data).length > 0, {
  message: 'At least one field must be provided'
});

// PATCH 只更新提供的字段
app.patch('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  try {
    const data = patchUserSchema.parse(req.body);

    const user = await prisma.user.update({
      where: { id: parseInt(id) },
      data: {
        ...data,
        updatedAt: new Date()
      },
      select: {
        id: true,
        name: true,
        email: true,
        bio: true,
        avatar: true,
        updatedAt: true
      }
    });

    res.json({ data: user });

  } catch (error) {
    if (error.code === 'P2025') {
      return res.status(404).json({
        error: 'User not found'
      });
    }
    throw error;
  }
});
```

#### DELETE - 删除资源

```typescript
// 硬删除
app.delete('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  try {
    await prisma.user.delete({
      where: { id: parseInt(id) }
    });

    // 204 No Content（无响应体）
    res.status(204).send();

  } catch (error) {
    if (error.code === 'P2025') {
      return res.status(404).json({
        error: 'User not found'
      });
    }
    throw error;
  }
});

// 软删除（推荐）
app.delete('/api/users/:id', async (req, res) => {
  const { id } = req.params;

  try {
    await prisma.user.update({
      where: { id: parseInt(id) },
      data: {
        deletedAt: new Date(),
        isActive: false
      }
    });

    res.status(204).send();

  } catch (error) {
    if (error.code === 'P2025') {
      return res.status(404).json({
        error: 'User not found'
      });
    }
    throw error;
  }
});
```

---

## 状态码

### 状态码分类

#### 2xx 成功

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| **200 OK** | 请求成功 | GET、PUT、PATCH 成功 |
| **201 Created** | 资源已创建 | POST 成功 |
| **202 Accepted** | 请求已接受 | 异步处理 |
| **204 No Content** | 无内容 | DELETE 成功 |

#### 4xx 客户端错误

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| **400 Bad Request** | 请求无效 | 参数错误、验证失败 |
| **401 Unauthorized** | 未认证 | 缺少或无效的认证 |
| **403 Forbidden** | 无权限 | 认证成功但无权限 |
| **404 Not Found** | 资源不存在 | 资源未找到 |
| **405 Method Not Allowed** | 方法不允许 | 使用了不支持的 HTTP 方法 |
| **409 Conflict** | 冲突 | 资源已存在 |
| **422 Unprocessable Entity** | 无法处理 | 语义错误 |
| **429 Too Many Requests** | 请求过多 | 触发限流 |

#### 5xx 服务器错误

| 状态码 | 含义 | 使用场景 |
|--------|------|---------|
| **500 Internal Server Error** | 服务器错误 | 未捕获的异常 |
| **502 Bad Gateway** | 网关错误 | 上游服务错误 |
| **503 Service Unavailable** | 服务不可用 | 维护中、过载 |
| **504 Gateway Timeout** | 网关超时 | 上游服务超时 |

### 状态码使用示例

```typescript
// 统一的错误处理
class ApiError extends Error {
  constructor(
    public statusCode: number,
    message: string,
    public details?: any
  ) {
    super(message);
  }
}

// 各种错误情况
app.get('/api/users/:id', async (req, res, next) => {
  try {
    const { id } = req.params;

    // 验证 ID 格式
    if (!/^\d+$/.test(id)) {
      throw new ApiError(400, 'Invalid user ID format');
    }

    const user = await prisma.user.findUnique({
      where: { id: parseInt(id) }
    });

    // 资源不存在
    if (!user) {
      throw new ApiError(404, 'User not found');
    }

    // 检查权限
    if (user.id !== req.user.id && !req.user.isAdmin) {
      throw new ApiError(403, 'Access denied');
    }

    res.json({ data: user });

  } catch (error) {
    next(error);
  }
});

// 全局错误处理
app.use((err: Error, req, res, next) => {
  if (err instanceof ApiError) {
    return res.status(err.statusCode).json({
      error: err.message,
      details: err.details
    });
  }

  // 未知错误
  console.error('Unhandled error:', err);
  res.status(500).json({
    error: 'Internal server error'
  });
});
```

---

## URL 设计

### 命名规范

#### ✅ 好的做法

```typescript
// 1. 使用名词，不要使用动词
GET    /api/users           // ✅
GET    /api/getUsers        // ❌

// 2. 使用复数形式
GET    /api/users           // ✅
GET    /api/user            // ❌

// 3. 使用小写字母和连字符
GET    /api/blog-posts      // ✅
GET    /api/blogPosts       // ❌
GET    /api/BlogPosts       // ❌

// 4. 资源嵌套（不超过 3 层）
GET    /api/users/123/posts           // ✅
GET    /api/users/123/posts/456       // ✅
GET    /api/users/123/posts/456/comments/789  // ❌ 太深

// 5. 使用查询参数进行过滤
GET    /api/users?status=active       // ✅
GET    /api/active-users              // ❌
```

### RESTful URL 设计

```typescript
// 用户资源
GET    /api/users              // 获取用户列表
GET    /api/users/:id          // 获取单个用户
POST   /api/users              // 创建用户
PUT    /api/users/:id          // 完整更新用户
PATCH  /api/users/:id          // 部分更新用户
DELETE /api/users/:id          // 删除用户

// 文章资源
GET    /api/posts              // 获取文章列表
GET    /api/posts/:id          // 获取单篇文章
POST   /api/posts              // 创建文章
PUT    /api/posts/:id          // 更新文章
DELETE /api/posts/:id          // 删除文章

// 用户的文章（子资源）
GET    /api/users/:userId/posts         // 获取某用户的文章
POST   /api/users/:userId/posts         // 为某用户创建文章

// 文章的评论（子资源）
GET    /api/posts/:postId/comments      // 获取文章评论
POST   /api/posts/:postId/comments      // 添加评论
DELETE /api/posts/:postId/comments/:id  // 删除评论
```

### 特殊操作

```typescript
// 对于不符合 CRUD 的操作，使用动词作为子资源
POST   /api/users/:id/activate         // 激活用户
POST   /api/users/:id/deactivate       // 停用用户
POST   /api/posts/:id/publish          // 发布文章
POST   /api/orders/:id/cancel          // 取消订单
POST   /api/password/reset             // 重置密码

// 批量操作
POST   /api/users/batch                // 批量创建
DELETE /api/users/batch                // 批量删除
PATCH  /api/users/batch                // 批量更新

// 搜索
GET    /api/search/users?q=john        // 搜索用户
GET    /api/search/posts?q=nodejs      // 搜索文章
```

---

## 请求与响应

### 请求格式

```typescript
// Content-Type: application/json
POST /api/users
{
  "name": "John Doe",
  "email": "john@example.com",
  "age": 30
}

// Content-Type: multipart/form-data（文件上传）
POST /api/users/avatar
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="photo.jpg"
Content-Type: image/jpeg

[binary data]
------WebKitFormBoundary--
```

### 响应格式

#### 统一响应结构

```typescript
interface ApiResponse<T> {
  data?: T;
  error?: string;
  message?: string;
  meta?: {
    page?: number;
    pageSize?: number;
    total?: number;
    [key: string]: any;
  };
}

// 成功响应
{
  "data": {
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "message": "User created successfully"
}

// 列表响应
{
  "data": [
    { "id": 1, "name": "User 1" },
    { "id": 2, "name": "User 2" }
  ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 100,
    "totalPages": 5
  }
}

// 错误响应
{
  "error": "Validation failed",
  "details": [
    {
      "field": "email",
      "message": "Invalid email format"
    },
    {
      "field": "password",
      "message": "Password must be at least 8 characters"
    }
  ]
}
```

#### 响应中间件

```typescript
// 统一响应格式中间件
function responseFormatter(req: Request, res: Response, next: NextFunction) {
  // 成功响应
  res.success = (data: any, message?: string) => {
    res.json({
      data,
      message,
      timestamp: new Date().toISOString()
    });
  };

  // 分页响应
  res.paginated = (data: any[], meta: any) => {
    res.json({
      data,
      meta: {
        ...meta,
        timestamp: new Date().toISOString()
      }
    });
  };

  // 错误响应
  res.error = (statusCode: number, message: string, details?: any) => {
    res.status(statusCode).json({
      error: message,
      details,
      timestamp: new Date().toISOString()
    });
  };

  next();
}

app.use(responseFormatter);

// 使用
app.get('/api/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.success(users);
});
```

---

## 分页排序过滤

### 分页

```typescript
interface PaginationParams {
  page?: number;      // 页码（从 1 开始）
  pageSize?: number;  // 每页数量
  limit?: number;     // 别名
  offset?: number;    // 偏移量
}

// Offset 分页（简单但性能差）
app.get('/api/posts', async (req, res) => {
  const page = parseInt(req.query.page as string) || 1;
  const pageSize = parseInt(req.query.pageSize as string) || 20;
  const offset = (page - 1) * pageSize;

  const [posts, total] = await Promise.all([
    prisma.post.findMany({
      skip: offset,
      take: pageSize,
      orderBy: { createdAt: 'desc' }
    }),
    prisma.post.count()
  ]);

  res.paginated(posts, {
    page,
    pageSize,
    total,
    totalPages: Math.ceil(total / pageSize)
  });
});

// Cursor 分页（推荐，性能好）
app.get('/api/posts', async (req, res) => {
  const cursor = req.query.cursor as string;
  const limit = parseInt(req.query.limit as string) || 20;

  const posts = await prisma.post.findMany({
    take: limit + 1,  // 多取一个，判断是否有下一页
    ...(cursor && {
      cursor: { id: parseInt(cursor) },
      skip: 1  // 跳过 cursor 本身
    }),
    orderBy: { id: 'desc' }
  });

  const hasMore = posts.length > limit;
  const data = hasMore ? posts.slice(0, -1) : posts;
  const nextCursor = hasMore ? data[data.length - 1].id : null;

  res.json({
    data,
    meta: {
      nextCursor,
      hasMore
    }
  });
});
```

### 排序

```typescript
interface SortParams {
  sortBy?: string;    // 排序字段
  order?: 'asc' | 'desc';  // 排序方向
}

app.get('/api/users', async (req, res) => {
  const sortBy = req.query.sortBy as string || 'createdAt';
  const order = req.query.order as 'asc' | 'desc' || 'desc';

  // 白名单验证
  const allowedSortFields = ['id', 'name', 'email', 'createdAt'];
  if (!allowedSortFields.includes(sortBy)) {
    return res.error(400, `Invalid sort field: ${sortBy}`);
  }

  const users = await prisma.user.findMany({
    orderBy: { [sortBy]: order }
  });

  res.success(users);
});

// 多字段排序
// GET /api/users?sort=name:asc,createdAt:desc
app.get('/api/users', async (req, res) => {
  const sortParam = req.query.sort as string;
  
  const orderBy = sortParam
    ? sortParam.split(',').map(field => {
        const [key, order] = field.split(':');
        return { [key]: order || 'asc' };
      })
    : [{ createdAt: 'desc' }];

  const users = await prisma.user.findMany({ orderBy });
  res.success(users);
});
```

### 过滤

```typescript
// 简单过滤
// GET /api/users?status=active&role=admin
app.get('/api/users', async (req, res) => {
  const { status, role } = req.query;

  const where: any = {};
  if (status) where.status = status;
  if (role) where.role = role;

  const users = await prisma.user.findMany({ where });
  res.success(users);
});

// 复杂过滤（支持操作符）
// GET /api/users?age[gte]=18&age[lte]=60
app.get('/api/users', async (req, res) => {
  const filters = parseFilters(req.query);
  
  const users = await prisma.user.findMany({
    where: filters
  });
  
  res.success(users);
});

function parseFilters(query: any) {
  const where: any = {};

  for (const [key, value] of Object.entries(query)) {
    if (typeof value === 'object') {
      // 操作符：gte, lte, gt, lt, contains, startsWith
      where[key] = value;
    } else {
      where[key] = value;
    }
  }

  return where;
}

// 搜索
// GET /api/users?q=john
app.get('/api/users', async (req, res) => {
  const query = req.query.q as string;

  if (!query) {
    return res.error(400, 'Query parameter required');
  }

  const users = await prisma.user.findMany({
    where: {
      OR: [
        { name: { contains: query, mode: 'insensitive' } },
        { email: { contains: query, mode: 'insensitive' } }
      ]
    }
  });

  res.success(users);
});
```

### 字段选择

```typescript
// 指定返回字段
// GET /api/users?fields=id,name,email
app.get('/api/users', async (req, res) => {
  const fields = req.query.fields as string;
  
  const select = fields
    ? fields.split(',').reduce((acc, field) => {
        acc[field.trim()] = true;
        return acc;
      }, {} as any)
    : undefined;

  const users = await prisma.user.findMany({ select });
  res.success(users);
});

// 排除字段
// GET /api/users?exclude=password,refreshToken
app.get('/api/users', async (req, res) => {
  const exclude = req.query.exclude as string;
  
  const users = await prisma.user.findMany();
  
  if (exclude) {
    const excludeFields = exclude.split(',');
    const filtered = users.map(user => {
      const copy = { ...user };
      excludeFields.forEach(field => delete copy[field]);
      return copy;
    });
    return res.success(filtered);
  }

  res.success(users);
});
```

---

## 错误处理

### 错误响应格式

```typescript
interface ErrorResponse {
  error: string;          // 错误信息
  code?: string;          // 错误码
  details?: any;          // 详细信息
  timestamp: string;      // 时间戳
  path?: string;          // 请求路径
  requestId?: string;     // 请求 ID
}

// 示例
{
  "error": "Validation failed",
  "code": "VALIDATION_ERROR",
  "details": [
    {
      "field": "email",
      "message": "Invalid email format",
      "value": "invalid-email"
    }
  ],
  "timestamp": "2024-01-01T00:00:00.000Z",
  "path": "/api/users",
  "requestId": "abc123"
}
```

### 错误分类

```typescript
// 自定义错误类
class AppError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super(400, 'VALIDATION_ERROR', message, details);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, 'NOT_FOUND', `${resource} not found`);
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, 'UNAUTHORIZED', message);
  }
}

class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, 'FORBIDDEN', message);
  }
}

class ConflictError extends AppError {
  constructor(message: string) {
    super(409, 'CONFLICT', message);
  }
}

// 使用
app.post('/api/users', async (req, res, next) => {
  try {
    const { email } = req.body;

    const existing = await prisma.user.findUnique({
      where: { email }
    });

    if (existing) {
      throw new ConflictError('Email already exists');
    }

    // 创建用户...

  } catch (error) {
    next(error);
  }
});
```

### 全局错误处理

```typescript
// 错误处理中间件
function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  // 请求 ID（用于追踪）
  const requestId = req.headers['x-request-id'] as string || generateId();

  // 记录错误
  console.error({
    requestId,
    error: err.message,
    stack: err.stack,
    path: req.path,
    method: req.method
  });

  // AppError
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.message,
      code: err.code,
      details: err.details,
      timestamp: new Date().toISOString(),
      path: req.path,
      requestId
    });
  }

  // Prisma 错误
  if (err.name === 'PrismaClientKnownRequestError') {
    const prismaError = err as any;
    
    if (prismaError.code === 'P2002') {
      // 唯一约束冲突
      return res.status(409).json({
        error: 'Resource already exists',
        code: 'CONFLICT',
        details: prismaError.meta,
        timestamp: new Date().toISOString(),
        requestId
      });
    }

    if (prismaError.code === 'P2025') {
      // 记录未找到
      return res.status(404).json({
        error: 'Resource not found',
        code: 'NOT_FOUND',
        timestamp: new Date().toISOString(),
        requestId
      });
    }
  }

  // Zod 验证错误
  if (err instanceof z.ZodError) {
    return res.status(400).json({
      error: 'Validation failed',
      code: 'VALIDATION_ERROR',
      details: err.errors.map(e => ({
        field: e.path.join('.'),
        message: e.message,
        code: e.code
      })),
      timestamp: new Date().toISOString(),
      requestId
    });
  }

  // 未知错误
  res.status(500).json({
    error: 'Internal server error',
    code: 'INTERNAL_ERROR',
    timestamp: new Date().toISOString(),
    requestId
  });
}

app.use(errorHandler);
```

---

## 最佳实践

### 1. 版本控制

```typescript
// URL 版本控制（推荐）
app.get('/api/v1/users', getUsersV1);
app.get('/api/v2/users', getUsersV2);

// Header 版本控制
app.get('/api/users', (req, res) => {
  const version = req.headers['api-version'] || '1';
  
  if (version === '2') {
    return getUsersV2(req, res);
  }
  
  return getUsersV1(req, res);
});
```

### 2. HATEOAS（超媒体）

```typescript
// 在响应中包含相关链接
app.get('/api/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: parseInt(req.params.id) }
  });

  res.json({
    data: user,
    links: {
      self: `/api/users/${user.id}`,
      posts: `/api/users/${user.id}/posts`,
      followers: `/api/users/${user.id}/followers`,
      following: `/api/users/${user.id}/following`
    }
  });
});
```

### 3. 幂等性

```typescript
// POST 使用幂等性键
app.post('/api/orders', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'] as string;

  if (!idempotencyKey) {
    return res.error(400, 'Idempotency-Key header required');
  }

  // 检查是否已处理
  const existing = await redis.get(`idempotency:${idempotencyKey}`);
  if (existing) {
    return res.json(JSON.parse(existing));
  }

  // 创建订单
  const order = await prisma.order.create({
    data: req.body
  });

  // 缓存结果（24 小时）
  await redis.setex(
    `idempotency:${idempotencyKey}`,
    86400,
    JSON.stringify({ data: order })
  );

  res.status(201).json({ data: order });
});
```

### 4. 限流

```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 最多 100 个请求
  message: 'Too many requests',
  standardHeaders: true, // RateLimit-* headers
  legacyHeaders: false
});

app.use('/api', limiter);
```

### 5. 缓存

```typescript
// ETag 缓存
app.get('/api/users/:id', async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: parseInt(req.params.id) }
  });

  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }

  // 计算 ETag
  const etag = generateETag(user);
  res.setHeader('ETag', etag);

  // 检查客户端 ETag
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).send(); // Not Modified
  }

  res.json({ data: user });
});

// Cache-Control
app.get('/api/public/posts', async (req, res) => {
  const posts = await prisma.post.findMany({
    where: { isPublic: true }
  });

  // 缓存 5 分钟
  res.setHeader('Cache-Control', 'public, max-age=300');
  res.json({ data: posts });
});
```

### 6. 压缩

```typescript
import compression from 'compression';

app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  level: 6  // 压缩级别 0-9
}));
```

### 7. CORS

```typescript
import cors from 'cors';

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count', 'X-Page'],
  credentials: true,
  maxAge: 86400  // 24 小时
}));
```

---

## 常见面试题

### 1. PUT 和 PATCH 的区别？

**回答**：

- **PUT**：完整替换资源
  - 必须提供完整的资源表示
  - 幂等性：多次请求结果相同
  - 缺少字段会被清空

- **PATCH**：部分更新资源
  - 只提供需要更新的字段
  - 不是严格幂等（取决于实现）
  - 其他字段保持不变

```typescript
// PUT - 必须提供所有字段
PUT /api/users/123
{
  "name": "John",
  "email": "john@example.com",
  "bio": "Developer"  // 必须提供
}

// PATCH - 只提供要更新的字段
PATCH /api/users/123
{
  "name": "John"  // 只更新 name
}
```

### 2. 如何设计幂等性接口？

**回答**：

1. **GET、PUT、DELETE 天然幂等**
2. **POST 使用幂等性键**：
   ```typescript
   Idempotency-Key: abc123
   ```
3. **使用唯一约束**：
   ```typescript
   const order = await prisma.order.upsert({
     where: { orderId: data.orderId },
     update: {},
     create: data
   });
   ```

### 3. RESTful API 的优缺点？

**优点**：
- ✅ 简单易懂，学习成本低
- ✅ 无状态，易于缓存和扩展
- ✅ 使用标准 HTTP，工具支持好
- ✅ 前后端分离

**缺点**：
- ❌ 过度获取/获取不足（Over/Under Fetching）
- ❌ 多次请求（N+1 问题）
- ❌ URL 爆炸（复杂资源关系）
- ❌ 版本管理困难

### 4. 如何设计 API 版本控制？

**方案对比**：

| 方案 | 示例 | 优点 | 缺点 |
|------|------|------|------|
| **URL 版本** | `/api/v1/users` | 清晰、易缓存 | URL 爆炸 |
| **Header 版本** | `API-Version: 1` | URL 清爽 | 不直观 |
| **内容协商** | `Accept: application/vnd.api.v1+json` | RESTful | 复杂 |

**推荐**：URL 版本控制（最常用）

### 5. API 分页的最佳实践？

**回答**：

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **Offset 分页** | 小数据量 | 简单、可跳页 | 性能差、数据变化 |
| **Cursor 分页** | 大数据量、实时数据 | 性能好、一致性 | 不能跳页 |

**推荐**：Cursor 分页

---

## 总结

### RESTful API 设计要点

1. **资源导向**：URL 表示资源，用名词不用动词
2. **HTTP 方法**：合理使用 GET、POST、PUT、PATCH、DELETE
3. **状态码**：正确使用 2xx、4xx、5xx
4. **统一格式**：请求和响应格式一致
5. **错误处理**：清晰的错误信息和错误码
6. **幂等性**：PUT、DELETE 幂等，POST 使用幂等性键
7. **缓存**：使用 ETag、Cache-Control
8. **安全性**：HTTPS、认证、授权、限流

### 实践检查清单

- [ ] URL 是否符合 RESTful 规范？
- [ ] HTTP 方法使用是否正确？
- [ ] 状态码是否合理？
- [ ] 响应格式是否统一？
- [ ] 是否有完善的错误处理？
- [ ] 是否支持分页、排序、过滤？
- [ ] 是否考虑了幂等性？
- [ ] 是否有 API 版本控制？
- [ ] 是否有限流保护？
- [ ] 是否有完善的文档？

---

**下一篇**：[GraphQL](./02-graphql.md)


