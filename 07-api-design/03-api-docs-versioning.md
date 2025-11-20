# API 文档与版本控制

完善的 API 文档和合理的版本控制是 API 设计的重要组成部分。本文讲解如何使用 Swagger/OpenAPI 生成文档，以及 API 版本控制的最佳实践。

## 目录
- [Swagger/OpenAPI](#swaggeropenapi)
- [自动生成文档](#自动生成文档)
- [API 版本控制](#api-版本控制)
- [API 弃用](#api-弃用)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## Swagger/OpenAPI

### OpenAPI 规范

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: User management API
  contact:
    name: API Support
    email: support@example.com
  license:
    name: MIT

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server

paths:
  /users:
    get:
      summary: Get all users
      tags:
        - Users
      parameters:
        - in: query
          name: page
          schema:
            type: integer
            default: 1
        - in: query
          name: pageSize
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  meta:
                    $ref: '#/components/schemas/PaginationMeta'
    
    post:
      summary: Create a new user
      tags:
        - Users
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserInput'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    $ref: '#/components/schemas/User'
        '400':
          description: Bad request
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '409':
          description: User already exists
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  
  /users/{id}:
    get:
      summary: Get user by ID
      tags:
        - Users
      parameters:
        - in: path
          name: id
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    $ref: '#/components/schemas/User'
        '404':
          description: User not found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
    
    patch:
      summary: Update user
      tags:
        - Users
      parameters:
        - in: path
          name: id
          required: true
          schema:
            type: integer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateUserInput'
      responses:
        '200':
          description: User updated
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    $ref: '#/components/schemas/User'
    
    delete:
      summary: Delete user
      tags:
        - Users
      parameters:
        - in: path
          name: id
          required: true
          schema:
            type: integer
      responses:
        '204':
          description: User deleted
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          example: 123
        name:
          type: string
          example: John Doe
        email:
          type: string
          format: email
          example: john@example.com
        role:
          type: string
          enum: [admin, user, guest]
          example: user
        createdAt:
          type: string
          format: date-time
          example: '2024-01-01T00:00:00Z'
    
    CreateUserInput:
      type: object
      required:
        - name
        - email
        - password
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 50
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
    
    UpdateUserInput:
      type: object
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 50
        email:
          type: string
          format: email
        bio:
          type: string
    
    PaginationMeta:
      type: object
      properties:
        page:
          type: integer
        pageSize:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer
    
    Error:
      type: object
      properties:
        error:
          type: string
        code:
          type: string
        details:
          type: object
  
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - BearerAuth: []
```

---

## 自动生成文档

### 方案 1：swagger-jsdoc + swagger-ui-express

```typescript
import express from 'express';
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';

const app = express();

// Swagger 配置
const swaggerOptions = {
  definition: {
    openapi: '3.0.0',
    info: {
      title: 'User API',
      version: '1.0.0',
      description: 'User management API',
      contact: {
        name: 'API Support',
        email: 'support@example.com'
      }
    },
    servers: [
      {
        url: 'http://localhost:3000/api/v1',
        description: 'Development server'
      }
    ],
    components: {
      securitySchemes: {
        bearerAuth: {
          type: 'http',
          scheme: 'bearer',
          bearerFormat: 'JWT'
        }
      }
    },
    security: [
      {
        bearerAuth: []
      }
    ]
  },
  apis: ['./src/routes/*.ts'] // 扫描路由文件
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);

// Swagger UI
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec, {
  customCss: '.swagger-ui .topbar { display: none }',
  customSiteTitle: 'User API Documentation'
}));

// 导出 JSON
app.get('/api-docs.json', (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.send(swaggerSpec);
});

// 路由文件中使用 JSDoc 注释
/**
 * @swagger
 * /users:
 *   get:
 *     summary: Get all users
 *     tags: [Users]
 *     parameters:
 *       - in: query
 *         name: page
 *         schema:
 *           type: integer
 *           default: 1
 *       - in: query
 *         name: pageSize
 *         schema:
 *           type: integer
 *           default: 20
 *     responses:
 *       200:
 *         description: Successful response
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 data:
 *                   type: array
 *                   items:
 *                     $ref: '#/components/schemas/User'
 *                 meta:
 *                   type: object
 *                   properties:
 *                     page:
 *                       type: integer
 *                     pageSize:
 *                       type: integer
 *                     total:
 *                       type: integer
 */
app.get('/api/v1/users', async (req, res) => {
  // 实现...
});

/**
 * @swagger
 * /users/{id}:
 *   get:
 *     summary: Get user by ID
 *     tags: [Users]
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema:
 *           type: integer
 *     responses:
 *       200:
 *         description: Successful response
 *       404:
 *         description: User not found
 */
app.get('/api/v1/users/:id', async (req, res) => {
  // 实现...
});

/**
 * @swagger
 * components:
 *   schemas:
 *     User:
 *       type: object
 *       properties:
 *         id:
 *           type: integer
 *         name:
 *           type: string
 *         email:
 *           type: string
 *           format: email
 *         createdAt:
 *           type: string
 *           format: date-time
 */
```

### 方案 2：NestJS + @nestjs/swagger

```typescript
import { NestFactory } from '@nestjs/core';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger 配置
  const config = new DocumentBuilder()
    .setTitle('User API')
    .setDescription('User management API')
    .setVersion('1.0')
    .addBearerAuth()
    .addTag('users')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api-docs', app, document);

  await app.listen(3000);
}
bootstrap();

// Controller 中使用装饰器
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  Patch,
  Delete,
  Query
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiParam,
  ApiQuery,
  ApiBearerAuth
} from '@nestjs/swagger';

@ApiTags('users')
@ApiBearerAuth()
@Controller('users')
export class UserController {
  @Get()
  @ApiOperation({ summary: 'Get all users' })
  @ApiQuery({ name: 'page', required: false, type: Number })
  @ApiQuery({ name: 'pageSize', required: false, type: Number })
  @ApiResponse({
    status: 200,
    description: 'Successful response',
    type: [UserDto]
  })
  async findAll(
    @Query('page') page: number = 1,
    @Query('pageSize') pageSize: number = 20
  ) {
    // 实现...
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: Number })
  @ApiResponse({ status: 200, description: 'User found', type: UserDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  async findOne(@Param('id') id: number) {
    // 实现...
  }

  @Post()
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({ status: 201, description: 'User created', type: UserDto })
  @ApiResponse({ status: 400, description: 'Bad request' })
  async create(@Body() createUserDto: CreateUserDto) {
    // 实现...
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Update user' })
  @ApiParam({ name: 'id', type: Number })
  @ApiResponse({ status: 200, description: 'User updated', type: UserDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  async update(
    @Param('id') id: number,
    @Body() updateUserDto: UpdateUserDto
  ) {
    // 实现...
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Delete user' })
  @ApiParam({ name: 'id', type: Number })
  @ApiResponse({ status: 204, description: 'User deleted' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async remove(@Param('id') id: number) {
    // 实现...
  }
}

// DTO 中使用装饰器
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsString, IsEmail, MinLength, MaxLength } from 'class-validator';

export class CreateUserDto {
  @ApiProperty({ example: 'John Doe', minLength: 2, maxLength: 50 })
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @ApiProperty({ example: 'john@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'password123', minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;
}

export class UpdateUserDto {
  @ApiPropertyOptional({ minLength: 2, maxLength: 50 })
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name?: string;

  @ApiPropertyOptional()
  @IsEmail()
  email?: string;

  @ApiPropertyOptional()
  @IsString()
  bio?: string;
}

export class UserDto {
  @ApiProperty({ example: 123 })
  id: number;

  @ApiProperty({ example: 'John Doe' })
  name: string;

  @ApiProperty({ example: 'john@example.com' })
  email: string;

  @ApiProperty({ example: 'user', enum: ['admin', 'user', 'guest'] })
  role: string;

  @ApiProperty({ example: '2024-01-01T00:00:00Z' })
  createdAt: Date;
}
```

### 方案 3：tsoa（TypeScript OpenAPI）

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Route,
  Tags,
  Body,
  Path,
  Query,
  Response,
  SuccessResponse
} from 'tsoa';

interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'guest';
  createdAt: Date;
}

interface CreateUserInput {
  name: string;
  email: string;
  password: string;
}

interface UpdateUserInput {
  name?: string;
  email?: string;
  bio?: string;
}

@Route('users')
@Tags('Users')
export class UserController extends Controller {
  /**
   * Get all users
   * @param page Page number
   * @param pageSize Page size
   */
  @Get()
  public async getUsers(
    @Query() page: number = 1,
    @Query() pageSize: number = 20
  ): Promise<{ data: User[]; meta: any }> {
    // 实现...
  }

  /**
   * Get user by ID
   * @param id User ID
   */
  @Get('{id}')
  @Response<void>(404, 'User not found')
  public async getUser(@Path() id: number): Promise<{ data: User }> {
    // 实现...
  }

  /**
   * Create a new user
   */
  @Post()
  @SuccessResponse('201', 'Created')
  @Response<void>(400, 'Bad request')
  @Response<void>(409, 'User already exists')
  public async createUser(
    @Body() body: CreateUserInput
  ): Promise<{ data: User }> {
    this.setStatus(201);
    // 实现...
  }

  /**
   * Update user
   */
  @Put('{id}')
  @Response<void>(404, 'User not found')
  public async updateUser(
    @Path() id: number,
    @Body() body: UpdateUserInput
  ): Promise<{ data: User }> {
    // 实现...
  }

  /**
   * Delete user
   */
  @Delete('{id}')
  @SuccessResponse('204', 'Deleted')
  @Response<void>(404, 'User not found')
  public async deleteUser(@Path() id: number): Promise<void> {
    this.setStatus(204);
    // 实现...
  }
}

// 生成 OpenAPI spec
// tsoa spec-and-routes
```

---

## API 版本控制

### 方案对比

| 方案 | 示例 | 优点 | 缺点 | 推荐度 |
|------|------|------|------|--------|
| **URL 版本** | `/api/v1/users` | 清晰、易缓存 | URL 爆炸 | ⭐⭐⭐⭐⭐ |
| **Header 版本** | `API-Version: 1` | URL 清爽 | 不直观、难缓存 | ⭐⭐⭐ |
| **查询参数** | `/api/users?version=1` | 灵活 | 不规范 | ⭐⭐ |
| **内容协商** | `Accept: app/vnd.api.v1+json` | RESTful | 复杂 | ⭐⭐ |

### 1. URL 版本控制（推荐）

```typescript
import express from 'express';

const app = express();

// V1 路由
const v1Router = express.Router();

v1Router.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    select: { id: true, name: true, email: true }
  });
  res.json({ data: users });
});

app.use('/api/v1', v1Router);

// V2 路由
const v2Router = express.Router();

v2Router.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    select: {
      id: true,
      fullName: true, // 字段名变更
      email: true,
      avatar: true    // 新字段
    }
  });
  res.json({ data: users }); // 响应格式保持一致
});

app.use('/api/v2', v2Router);

// 默认版本（最新版本）
app.use('/api', v2Router);
```

### 2. Header 版本控制

```typescript
// 中间件：解析版本
function parseApiVersion(req, res, next) {
  const version = req.headers['api-version'] || req.headers['x-api-version'] || '1';
  req.apiVersion = parseInt(version);
  next();
}

app.use(parseApiVersion);

// 路由
app.get('/api/users', async (req, res) => {
  if (req.apiVersion === 2) {
    return getUsersV2(req, res);
  }
  return getUsersV1(req, res);
});

async function getUsersV1(req, res) {
  const users = await prisma.user.findMany({
    select: { id: true, name: true, email: true }
  });
  res.setHeader('API-Version', '1');
  res.json({ data: users });
}

async function getUsersV2(req, res) {
  const users = await prisma.user.findMany({
    select: { id: true, fullName: true, email: true, avatar: true }
  });
  res.setHeader('API-Version', '2');
  res.json({ data: users });
}
```

### 3. 内容协商

```typescript
app.get('/api/users', async (req, res) => {
  const accept = req.headers.accept || '';

  // application/vnd.api.v2+json
  if (accept.includes('vnd.api.v2')) {
    return getUsersV2(req, res);
  }

  // application/vnd.api.v1+json 或默认
  return getUsersV1(req, res);
});
```

### 版本管理策略

```typescript
// 版本配置
const API_VERSIONS = {
  v1: {
    deprecated: true,
    sunsetDate: new Date('2025-12-31'),
    docs: '/api-docs/v1'
  },
  v2: {
    deprecated: false,
    current: true,
    docs: '/api-docs/v2'
  },
  v3: {
    deprecated: false,
    beta: true,
    docs: '/api-docs/v3'
  }
};

// 中间件：版本检查
function checkApiVersion(req, res, next) {
  const version = req.apiVersion || 'v1';
  const versionConfig = API_VERSIONS[version];

  if (!versionConfig) {
    return res.status(400).json({
      error: 'Invalid API version',
      supportedVersions: Object.keys(API_VERSIONS)
    });
  }

  // 弃用警告
  if (versionConfig.deprecated) {
    res.setHeader('Warning', `299 - "API version ${version} is deprecated"`);
    res.setHeader('Sunset', versionConfig.sunsetDate.toUTCString());
  }

  // Beta 警告
  if (versionConfig.beta) {
    res.setHeader('Warning', `299 - "API version ${version} is in beta"`);
  }

  next();
}

app.use(checkApiVersion);
```

---

## API 弃用

### 弃用流程

```
1. 公告（提前 6 个月）
   ↓
2. 添加弃用警告头
   ↓
3. 更新文档
   ↓
4. 设置日落时间（Sunset）
   ↓
5. 移除 API
```

### 实现

```typescript
// 弃用中间件
function deprecateEndpoint(
  message: string,
  sunsetDate: Date,
  alternatives?: string[]
) {
  return (req, res, next) => {
    // Deprecation 头
    res.setHeader(
      'Deprecation',
      sunsetDate.toUTCString()
    );

    // Sunset 头
    res.setHeader(
      'Sunset',
      sunsetDate.toUTCString()
    );

    // Warning 头
    let warning = `299 - "${message}"`;
    if (alternatives && alternatives.length > 0) {
      warning += `. Use ${alternatives.join(' or ')} instead.`;
    }
    res.setHeader('Warning', warning);

    // Link 头（指向文档）
    res.setHeader(
      'Link',
      '</api-docs/migration>; rel="deprecation"; type="text/html"'
    );

    // 记录使用情况
    logDeprecatedApiUsage(req.path, req.user?.id);

    next();
  };
}

// 使用
app.get(
  '/api/v1/users',
  deprecateEndpoint(
    'API v1 is deprecated',
    new Date('2025-12-31'),
    ['/api/v2/users']
  ),
  async (req, res) => {
    // 实现...
  }
);

// 响应头示例：
// Deprecation: Wed, 31 Dec 2025 23:59:59 GMT
// Sunset: Wed, 31 Dec 2025 23:59:59 GMT
// Warning: 299 - "API v1 is deprecated. Use /api/v2/users instead."
// Link: </api-docs/migration>; rel="deprecation"; type="text/html"
```

### 通知用户

```typescript
// 发送邮件通知
async function notifyDeprecationToUsers() {
  const users = await prisma.user.findMany({
    where: { isActive: true }
  });

  for (const user of users) {
    await sendEmail({
      to: user.email,
      subject: 'API Deprecation Notice',
      body: `
        Dear ${user.name},
        
        We are deprecating API v1 on December 31, 2025.
        Please migrate to API v2 before this date.
        
        Migration guide: https://docs.example.com/migration
        
        Thank you!
      `
    });
  }
}

// 定期任务
cron.schedule('0 0 1 * *', async () => {
  // 每月 1 号发送提醒
  await notifyDeprecationToUsers();
});
```

---

## 最佳实践

### 1. 语义化版本

```
Major.Minor.Patch

Major: 不兼容的 API 变更（v1 → v2）
Minor: 向后兼容的功能新增（v1.1 → v1.2）
Patch: 向后兼容的 Bug 修复（v1.1.0 → v1.1.1）
```

### 2. 变更日志

```markdown
# API Changelog

## v2.0.0 - 2024-01-01

### Breaking Changes
- ❌ Removed `/api/v1/users` endpoint
- 🔄 Changed field name from `name` to `fullName`
- 🔄 Changed response format

### New Features
- ✨ Added `/api/v2/users/:id/posts` endpoint
- ✨ Added `avatar` field to User

### Bug Fixes
- 🐛 Fixed pagination issue in `/api/v2/posts`

## v1.2.0 - 2023-12-01

### New Features
- ✨ Added search endpoint `/api/v1/search`

### Bug Fixes
- 🐛 Fixed email validation

## v1.1.0 - 2023-11-01

### New Features
- ✨ Added `bio` field to User
```

### 3. 迁移指南

```markdown
# Migration Guide: v1 → v2

## Overview
API v2 introduces several improvements and breaking changes.

## Breaking Changes

### 1. Field Name Changes

**Before (v1):**
```json
{
  "id": 123,
  "name": "John Doe"
}
```

**After (v2):**
```json
{
  "id": 123,
  "fullName": "John Doe"
}
```

### 2. Response Format

**Before (v1):**
```json
{
  "users": [...]
}
```

**After (v2):**
```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "total": 100
  }
}
```

## Step-by-Step Migration

1. Update base URL from `/api/v1` to `/api/v2`
2. Update field names in your code
3. Update response parsing logic
4. Test thoroughly
5. Deploy

## Support
Contact support@example.com for help.
```

### 4. 版本兼容性测试

```typescript
import { describe, it, expect } from '@jest/globals';
import request from 'supertest';
import app from '../app';

describe('API Version Compatibility', () => {
  describe('V1 API', () => {
    it('should return users in v1 format', async () => {
      const response = await request(app)
        .get('/api/v1/users')
        .expect(200);

      expect(response.body).toHaveProperty('users');
      expect(response.body.users[0]).toHaveProperty('name');
    });

    it('should include deprecation warning', async () => {
      const response = await request(app)
        .get('/api/v1/users');

      expect(response.headers).toHaveProperty('warning');
      expect(response.headers.warning).toContain('deprecated');
    });
  });

  describe('V2 API', () => {
    it('should return users in v2 format', async () => {
      const response = await request(app)
        .get('/api/v2/users')
        .expect(200);

      expect(response.body).toHaveProperty('data');
      expect(response.body).toHaveProperty('meta');
      expect(response.body.data[0]).toHaveProperty('fullName');
      expect(response.body.data[0]).toHaveProperty('avatar');
    });

    it('should not include deprecation warning', async () => {
      const response = await request(app)
        .get('/api/v2/users');

      expect(response.headers).not.toHaveProperty('deprecation');
    });
  });

  describe('Version Negotiation', () => {
    it('should use v1 with API-Version: 1 header', async () => {
      const response = await request(app)
        .get('/api/users')
        .set('API-Version', '1')
        .expect(200);

      expect(response.body).toHaveProperty('users');
    });

    it('should use v2 with API-Version: 2 header', async () => {
      const response = await request(app)
        .get('/api/users')
        .set('API-Version', '2')
        .expect(200);

      expect(response.body).toHaveProperty('data');
    });

    it('should default to latest version', async () => {
      const response = await request(app)
        .get('/api/users')
        .expect(200);

      expect(response.body).toHaveProperty('data'); // v2 格式
    });
  });
});
```

---

## 常见面试题

### 1. API 版本控制有哪些方案？

**回答**：

| 方案 | 适用场景 | 推荐度 |
|------|---------|--------|
| **URL 版本** | 大多数场景 | ⭐⭐⭐⭐⭐ |
| **Header 版本** | RESTful 纯粹主义 | ⭐⭐⭐ |
| **查询参数** | 简单场景 | ⭐⭐ |
| **内容协商** | 学术场景 | ⭐⭐ |

**推荐**：URL 版本控制，因为：
- 清晰直观
- 易于缓存
- 浏览器友好
- 文档清晰

### 2. 如何优雅地弃用 API？

**回答**：

1. **提前公告**（至少 6 个月）
2. **添加响应头**：
   - `Deprecation`: 弃用时间
   - `Sunset`: 移除时间
   - `Warning`: 警告信息
3. **更新文档**：标记为已弃用
4. **提供迁移指南**
5. **通知用户**：邮件、公告
6. **监控使用情况**
7. **逐步移除**

### 3. 如何生成 API 文档？

**回答**：

| 方案 | 特点 | 推荐场景 |
|------|------|---------|
| **swagger-jsdoc** | JSDoc 注释 | Express |
| **@nestjs/swagger** | 装饰器 | NestJS |
| **tsoa** | TypeScript 注解 | 类型安全 |
| **手写 OpenAPI** | 完全控制 | 复杂需求 |

**推荐**：
- NestJS → @nestjs/swagger
- Express → swagger-jsdoc
- TypeScript → tsoa

### 4. API 文档应该包含什么？

**回答**：

1. **概览**：API 介绍、基础 URL
2. **认证**：如何认证、获取 token
3. **端点**：所有 API 端点
4. **参数**：请求参数、路径参数、查询参数
5. **请求示例**：各种语言的示例代码
6. **响应示例**：成功和失败的响应
7. **错误码**：所有可能的错误码
8. **限流规则**：请求限制
9. **变更日志**：版本更新记录
10. **迁移指南**：版本升级指南

### 5. OpenAPI 和 Swagger 的关系？

**回答**：

- **Swagger**：最初的 API 规范和工具集
- **OpenAPI**：Swagger 规范捐献给 Linux 基金会后的新名称
- **OpenAPI 2.0**：也叫 Swagger 2.0
- **OpenAPI 3.0**：最新规范（2017 年发布）

**现在**：
- 规范叫 **OpenAPI Specification**
- 工具仍叫 **Swagger**（Swagger UI、Swagger Editor）

---

## 总结

### API 文档要点

1. **自动生成**：从代码生成，保持同步
2. **完整性**：包含所有端点、参数、响应
3. **示例**：提供清晰的请求和响应示例
4. **交互性**：Swagger UI、Postman Collection
5. **维护**：及时更新

### 版本控制要点

1. **方案选择**：推荐 URL 版本控制
2. **语义化**：遵循语义化版本规范
3. **向后兼容**：尽量保持向后兼容
4. **弃用流程**：提前公告、逐步移除
5. **文档化**：变更日志、迁移指南

### 实践检查清单

- [ ] 是否有完整的 API 文档？
- [ ] 文档是否自动生成？
- [ ] 是否有交互式文档（Swagger UI）？
- [ ] 是否有明确的版本控制策略？
- [ ] 是否有弃用流程？
- [ ] 是否有变更日志？
- [ ] 是否有迁移指南？
- [ ] 是否有版本兼容性测试？
- [ ] 是否有监控和告警？

---

**上一篇**：[GraphQL](./02-graphql.md)  
**返回目录**：[README](./README.md)

