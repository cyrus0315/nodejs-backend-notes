# 授权机制

授权（Authorization）是确定用户能做什么的过程。本文深入讲解 RBAC、ABAC、权限设计和实现。

## 目录
- [授权基础](#授权基础)
- [RBAC（基于角色）](#rbac)
- [ABAC（基于属性）](#abac)
- [权限设计模式](#权限设计模式)
- [数据级权限](#数据级权限)
- [NestJS 授权实现](#nestjs-授权实现)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## 授权基础

### 授权 vs 认证

```typescript
// 认证（Authentication）：谁在访问？
const user = await verifyToken(token); // 验证身份

// 授权（Authorization）：允许做什么？
if (user.role !== 'admin') {
  throw new Error('Forbidden'); // 检查权限
}
```

### 常见授权模型

| 模型 | 特点 | 适用场景 |
|------|------|---------|
| **ACL** | 访问控制列表 | 简单应用 |
| **RBAC** | 基于角色 | 大多数应用 ⭐⭐⭐⭐⭐ |
| **ABAC** | 基于属性 | 复杂权限 ⭐⭐⭐⭐ |
| **PBAC** | 基于策略 | 企业应用 |

---

## RBAC

### RBAC 核心概念

```
用户（User）→ 角色（Role）→ 权限（Permission）→ 资源（Resource）

示例：
User: John
  ↓
Role: Editor
  ↓
Permissions: [create_post, edit_post, delete_own_post]
  ↓
Resources: posts
```

### 数据库设计

```prisma
// schema.prisma

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String
  roles     UserRole[]
  createdAt DateTime @default(now())
}

model Role {
  id          Int          @id @default(autoincrement())
  name        String       @unique // admin, editor, viewer
  description String?
  permissions RolePermission[]
  users       UserRole[]
  createdAt   DateTime     @default(now())
}

model Permission {
  id          Int          @id @default(autoincrement())
  name        String       @unique // create_post, edit_post, delete_post
  resource    String       // post, user, comment
  action      String       // create, read, update, delete
  description String?
  roles       RolePermission[]
  createdAt   DateTime     @default(now())
}

// 用户-角色 多对多关系
model UserRole {
  userId    Int
  roleId    Int
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  role      Role     @relation(fields: [roleId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())

  @@id([userId, roleId])
}

// 角色-权限 多对多关系
model RolePermission {
  roleId       Int
  permissionId Int
  role         Role       @relation(fields: [roleId], references: [id], onDelete: Cascade)
  permission   Permission @relation(fields: [permissionId], references: [id], onDelete: Cascade)
  createdAt    DateTime   @default(now())

  @@id([roleId, permissionId])
}
```

### RBAC 实现

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

class RBACService {
  // 检查用户是否有某个权限
  async hasPermission(
    userId: number,
    resource: string,
    action: string
  ): Promise<boolean> {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: {
            role: {
              include: {
                permissions: {
                  include: {
                    permission: true
                  }
                }
              }
            }
          }
        }
      }
    });

    if (!user) return false;

    // 检查用户的所有角色的所有权限
    for (const userRole of user.roles) {
      const permissions = userRole.role.permissions;
      
      for (const rolePermission of permissions) {
        const permission = rolePermission.permission;
        
        if (
          permission.resource === resource &&
          permission.action === action
        ) {
          return true;
        }
      }
    }

    return false;
  }

  // 检查用户是否有某个角色
  async hasRole(userId: number, roleName: string): Promise<boolean> {
    const userRole = await prisma.userRole.findFirst({
      where: {
        userId,
        role: {
          name: roleName
        }
      }
    });

    return !!userRole;
  }

  // 获取用户的所有权限
  async getUserPermissions(userId: number): Promise<string[]> {
    const user = await prisma.user.findUnique({
      where: { id: userId },
      include: {
        roles: {
          include: {
            role: {
              include: {
                permissions: {
                  include: {
                    permission: true
                  }
                }
              }
            }
          }
        }
      }
    });

    if (!user) return [];

    const permissions = new Set<string>();

    for (const userRole of user.roles) {
      for (const rolePermission of userRole.role.permissions) {
        const p = rolePermission.permission;
        permissions.add(`${p.resource}:${p.action}`);
      }
    }

    return Array.from(permissions);
  }

  // 为用户分配角色
  async assignRole(userId: number, roleId: number): Promise<void> {
    await prisma.userRole.create({
      data: { userId, roleId }
    });
  }

  // 为角色添加权限
  async assignPermission(roleId: number, permissionId: number): Promise<void> {
    await prisma.rolePermission.create({
      data: { roleId, permissionId }
    });
  }
}

const rbacService = new RBACService();

// 授权中间件
function requirePermission(resource: string, action: string) {
  return async (req, res, next) => {
    const userId = req.user?.id;

    if (!userId) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    const hasPermission = await rbacService.hasPermission(
      userId,
      resource,
      action
    );

    if (!hasPermission) {
      return res.status(403).json({
        error: 'Forbidden',
        message: `Missing permission: ${resource}:${action}`
      });
    }

    next();
  };
}

// 角色中间件
function requireRole(roleName: string) {
  return async (req, res, next) => {
    const userId = req.user?.id;

    if (!userId) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    const hasRole = await rbacService.hasRole(userId, roleName);

    if (!hasRole) {
      return res.status(403).json({
        error: 'Forbidden',
        message: `Missing role: ${roleName}`
      });
    }

    next();
  };
}

// 使用示例
app.post('/api/posts',
  requireAuth,
  requirePermission('post', 'create'),
  createPost
);

app.delete('/api/posts/:id',
  requireAuth,
  requirePermission('post', 'delete'),
  deletePost
);

app.get('/api/admin/users',
  requireAuth,
  requireRole('admin'),
  getUsers
);
```

### 缓存优化

```typescript
import Redis from 'ioredis';

const redis = new Redis();

class CachedRBACService extends RBACService {
  // 缓存用户权限（5 分钟）
  async getUserPermissions(userId: number): Promise<string[]> {
    const cacheKey = `permissions:${userId}`;
    
    // 尝试从缓存获取
    const cached = await redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // 从数据库查询
    const permissions = await super.getUserPermissions(userId);

    // 写入缓存
    await redis.setex(cacheKey, 300, JSON.stringify(permissions));

    return permissions;
  }

  // 清除用户权限缓存
  async clearUserPermissionsCache(userId: number): Promise<void> {
    await redis.del(`permissions:${userId}`);
  }

  // 检查权限（带缓存）
  async hasPermission(
    userId: number,
    resource: string,
    action: string
  ): Promise<boolean> {
    const permissions = await this.getUserPermissions(userId);
    return permissions.includes(`${resource}:${action}`);
  }
}

const rbacService = new CachedRBACService();
```

---

## ABAC

### ABAC 核心概念

**ABAC（Attribute-Based Access Control）**基于属性做决策：

- **主体属性**：用户的角色、部门、级别
- **资源属性**：文档的状态、所有者、分类
- **环境属性**：时间、地点、IP

```typescript
// ABAC 示例
if (
  user.department === 'engineering' &&
  document.status === 'draft' &&
  currentTime.hour < 18
) {
  allow();
}
```

### ABAC 实现

```typescript
interface User {
  id: number;
  email: string;
  role: string;
  department: string;
  level: number;
}

interface Resource {
  id: number;
  type: string;
  ownerId: number;
  status: string;
  department?: string;
}

interface Context {
  time: Date;
  ip: string;
  userAgent: string;
}

interface Policy {
  id: string;
  name: string;
  effect: 'allow' | 'deny';
  conditions: Condition[];
}

interface Condition {
  attribute: string; // user.role, resource.status, context.time
  operator: '==' | '!=' | '>' | '<' | 'in' | 'contains';
  value: any;
}

class ABACService {
  private policies: Policy[] = [];

  // 添加策略
  addPolicy(policy: Policy): void {
    this.policies.push(policy);
  }

  // 评估策略
  async evaluate(
    user: User,
    resource: Resource,
    action: string,
    context: Context
  ): Promise<boolean> {
    for (const policy of this.policies) {
      // 检查所有条件是否满足
      const allConditionsMet = policy.conditions.every(condition =>
        this.evaluateCondition(condition, { user, resource, context })
      );

      if (allConditionsMet) {
        return policy.effect === 'allow';
      }
    }

    // 默认拒绝
    return false;
  }

  // 评估单个条件
  private evaluateCondition(
    condition: Condition,
    data: { user: User; resource: Resource; context: Context }
  ): boolean {
    const value = this.getAttribute(condition.attribute, data);

    switch (condition.operator) {
      case '==':
        return value == condition.value;
      case '!=':
        return value != condition.value;
      case '>':
        return value > condition.value;
      case '<':
        return value < condition.value;
      case 'in':
        return Array.isArray(condition.value) && condition.value.includes(value);
      case 'contains':
        return String(value).includes(condition.value);
      default:
        return false;
    }
  }

  // 获取属性值
  private getAttribute(
    attribute: string,
    data: { user: User; resource: Resource; context: Context }
  ): any {
    const parts = attribute.split('.');
    const [entity, field] = parts;

    switch (entity) {
      case 'user':
        return data.user[field];
      case 'resource':
        return data.resource[field];
      case 'context':
        return data.context[field];
      default:
        return undefined;
    }
  }
}

const abacService = new ABACService();

// 定义策略
abacService.addPolicy({
  id: 'policy1',
  name: '允许编辑自己部门的草稿文档',
  effect: 'allow',
  conditions: [
    { attribute: 'user.department', operator: '==', value: 'resource.department' },
    { attribute: 'resource.status', operator: '==', value: 'draft' }
  ]
});

abacService.addPolicy({
  id: 'policy2',
  name: '允许经理编辑任何草稿',
  effect: 'allow',
  conditions: [
    { attribute: 'user.role', operator: '==', value: 'manager' },
    { attribute: 'resource.status', operator: '==', value: 'draft' }
  ]
});

abacService.addPolicy({
  id: 'policy3',
  name: '工作时间才能删除',
  effect: 'allow',
  conditions: [
    { attribute: 'context.time.hour', operator: '>', value: 9 },
    { attribute: 'context.time.hour', operator: '<', value: 18 }
  ]
});

// ABAC 中间件
function requireABAC(action: string) {
  return async (req, res, next) => {
    const user = req.user;
    const resource = req.resource; // 需要提前加载资源
    const context = {
      time: new Date(),
      ip: req.ip,
      userAgent: req.headers['user-agent']
    };

    const allowed = await abacService.evaluate(user, resource, action, context);

    if (!allowed) {
      return res.status(403).json({ error: 'Forbidden by policy' });
    }

    next();
  };
}

// 使用示例
app.put('/api/documents/:id',
  requireAuth,
  loadResource('document'), // 加载资源
  requireABAC('update'),
  updateDocument
);
```

---

## 权限设计模式

### 1. 层次角色

```typescript
// 角色继承
const roleHierarchy = {
  admin: ['manager', 'editor', 'viewer'],
  manager: ['editor', 'viewer'],
  editor: ['viewer'],
  viewer: []
};

function hasRoleOrHigher(userRole: string, requiredRole: string): boolean {
  if (userRole === requiredRole) return true;

  const inheritedRoles = roleHierarchy[userRole] || [];
  return inheritedRoles.includes(requiredRole);
}

// 使用示例
if (hasRoleOrHigher(user.role, 'editor')) {
  // admin、manager、editor 都可以访问
}
```

### 2. 资源所有权

```typescript
// 检查资源所有权
function requireOwnership(resourceType: string) {
  return async (req, res, next) => {
    const resourceId = req.params.id;
    const userId = req.user.id;

    // 查询资源
    const resource = await prisma[resourceType].findUnique({
      where: { id: parseInt(resourceId) }
    });

    if (!resource) {
      return res.status(404).json({ error: 'Resource not found' });
    }

    // 检查所有权
    if (resource.userId !== userId) {
      // 检查是否是管理员
      const isAdmin = await rbacService.hasRole(userId, 'admin');
      if (!isAdmin) {
        return res.status(403).json({ error: 'Not the owner' });
      }
    }

    req.resource = resource;
    next();
  };
}

// 使用示例：只有作者或管理员可以删除文章
app.delete('/api/posts/:id',
  requireAuth,
  requireOwnership('post'),
  deletePost
);
```

### 3. 字段级权限

```typescript
// 根据用户角色过滤字段
function filterFields(data: any, userRole: string): any {
  const fieldPermissions = {
    admin: ['id', 'name', 'email', 'phone', 'salary', 'ssn'],
    manager: ['id', 'name', 'email', 'phone', 'salary'],
    user: ['id', 'name', 'email']
  };

  const allowedFields = fieldPermissions[userRole] || [];
  
  return Object.keys(data)
    .filter(key => allowedFields.includes(key))
    .reduce((obj, key) => {
      obj[key] = data[key];
      return obj;
    }, {});
}

// 使用示例
app.get('/api/users/:id', requireAuth, async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: parseInt(req.params.id) }
  });

  // 根据角色过滤字段
  const filteredUser = filterFields(user, req.user.role);

  res.json(filteredUser);
});
```

### 4. 动态权限

```typescript
// 基于条件的动态权限
async function canEditPost(userId: number, postId: number): Promise<boolean> {
  const post = await prisma.post.findUnique({
    where: { id: postId }
  });

  if (!post) return false;

  // 1. 作者可以编辑
  if (post.authorId === userId) return true;

  // 2. 管理员可以编辑
  if (await rbacService.hasRole(userId, 'admin')) return true;

  // 3. 协作者可以编辑
  const collaborator = await prisma.postCollaborator.findFirst({
    where: {
      postId,
      userId,
      permission: 'edit'
    }
  });
  if (collaborator) return true;

  // 4. 部门经理可以编辑本部门的文章
  const user = await prisma.user.findUnique({ where: { id: userId } });
  if (
    user?.role === 'manager' &&
    user.department === post.department
  ) {
    return true;
  }

  return false;
}

// 使用
app.put('/api/posts/:id', requireAuth, async (req, res) => {
  const postId = parseInt(req.params.id);
  const userId = req.user.id;

  if (!await canEditPost(userId, postId)) {
    return res.status(403).json({ error: 'Cannot edit this post' });
  }

  // 更新文章...
});
```

---

## 数据级权限

### 行级权限（Row-Level Security）

```typescript
// 自动过滤数据
class SecurePostService {
  async findAll(userId: number, userRole: string) {
    const where: any = {};

    // 普通用户只能看到自己的文章
    if (userRole === 'user') {
      where.authorId = userId;
    }

    // 编辑只能看到自己和草稿状态的文章
    if (userRole === 'editor') {
      where.OR = [
        { authorId: userId },
        { status: 'draft' }
      ];
    }

    // 管理员可以看到所有文章
    // where 保持为空

    return await prisma.post.findMany({ where });
  }

  async findOne(id: number, userId: number, userRole: string) {
    const post = await prisma.post.findUnique({ where: { id } });

    if (!post) return null;

    // 检查权限
    if (userRole === 'admin') return post;
    if (userRole === 'editor' && post.status === 'draft') return post;
    if (post.authorId === userId) return post;

    throw new Error('Forbidden');
  }
}

const securePostService = new SecurePostService();

// 使用
app.get('/api/posts', requireAuth, async (req, res) => {
  const posts = await securePostService.findAll(
    req.user.id,
    req.user.role
  );
  res.json(posts);
});
```

### Prisma 中间件实现 RLS

```typescript
// Prisma 中间件
prisma.$use(async (params, next) => {
  // 仅对 Post 模型应用 RLS
  if (params.model === 'Post') {
    const userId = getCurrentUserId(); // 从上下文获取当前用户
    const userRole = getCurrentUserRole();

    // 对查询操作添加过滤条件
    if (params.action === 'findMany' || params.action === 'findFirst') {
      if (userRole !== 'admin') {
        params.args.where = {
          ...params.args.where,
          OR: [
            { authorId: userId },
            { collaborators: { some: { userId } } }
          ]
        };
      }
    }

    // 对更新/删除操作检查权限
    if (params.action === 'update' || params.action === 'delete') {
      const post = await prisma.post.findUnique({
        where: params.args.where
      });

      if (post && post.authorId !== userId && userRole !== 'admin') {
        throw new Error('Forbidden');
      }
    }
  }

  return next(params);
});
```

---

## NestJS 授权实现

### Guards

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

// 自定义装饰器
export const Permissions = (...permissions: string[]) =>
  SetMetadata('permissions', permissions);

export const Roles = (...roles: string[]) =>
  SetMetadata('roles', roles);

// 权限 Guard
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private rbacService: RBACService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 获取装饰器定义的权限
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions',
      context.getHandler()
    );

    if (!requiredPermissions) {
      return true; // 没有权限要求
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      return false;
    }

    // 检查用户权限
    const userPermissions = await this.rbacService.getUserPermissions(user.id);

    return requiredPermissions.every(permission =>
      userPermissions.includes(permission)
    );
  }
}

// 角色 Guard
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private rbacService: RBACService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredRoles = this.reflector.get<string[]>(
      'roles',
      context.getHandler()
    );

    if (!requiredRoles) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    if (!user) {
      return false;
    }

    // 检查用户角色
    for (const role of requiredRoles) {
      if (await this.rbacService.hasRole(user.id, role)) {
        return true;
      }
    }

    return false;
  }
}

// 资源所有者 Guard
@Injectable()
export class OwnerGuard implements CanActivate {
  constructor(private prisma: PrismaService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resourceId = request.params.id;

    // 根据路径判断资源类型
    const resourceType = this.getResourceType(request.path);

    // 查询资源
    const resource = await this.prisma[resourceType].findUnique({
      where: { id: parseInt(resourceId) }
    });

    if (!resource) {
      return false;
    }

    // 检查所有权
    return resource.userId === user.id;
  }

  private getResourceType(path: string): string {
    // /api/posts/123 => post
    const match = path.match(/\/api\/(\w+)/);
    return match ? match[1].slice(0, -1) : '';
  }
}

// 使用示例
@Controller('posts')
@UseGuards(JwtAuthGuard)
export class PostsController {
  // 需要 create_post 权限
  @Post()
  @Permissions('post:create')
  @UseGuards(PermissionsGuard)
  async create(@Body() dto: CreatePostDto, @CurrentUser() user: User) {
    return this.postsService.create(dto, user.id);
  }

  // 需要 admin 或 editor 角色
  @Get('pending')
  @Roles('admin', 'editor')
  @UseGuards(RolesGuard)
  async getPending() {
    return this.postsService.findPending();
  }

  // 需要是资源所有者
  @Put(':id')
  @UseGuards(OwnerGuard)
  async update(@Param('id') id: string, @Body() dto: UpdatePostDto) {
    return this.postsService.update(+id, dto);
  }

  // 管理员或所有者可以删除
  @Delete(':id')
  @UseGuards(OwnerOrAdminGuard)
  async delete(@Param('id') id: string) {
    return this.postsService.delete(+id);
  }
}

// 组合 Guard
@Injectable()
export class OwnerOrAdminGuard implements CanActivate {
  constructor(
    private prisma: PrismaService,
    private rbacService: RBACService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    // 1. 检查是否是管理员
    if (await this.rbacService.hasRole(user.id, 'admin')) {
      return true;
    }

    // 2. 检查是否是所有者
    const resourceId = request.params.id;
    const resource = await this.prisma.post.findUnique({
      where: { id: parseInt(resourceId) }
    });

    return resource && resource.userId === user.id;
  }
}
```

### CASL（权限库）

```typescript
import { Ability, AbilityBuilder, AbilityClass } from '@casl/ability';

// 定义操作和主体
type Actions = 'create' | 'read' | 'update' | 'delete' | 'manage';
type Subjects = 'Post' | 'User' | 'Comment' | 'all';

export type AppAbility = Ability<[Actions, Subjects]>;

// 定义权限
export function defineAbilityFor(user: User): AppAbility {
  const { can, cannot, build } = new AbilityBuilder<AppAbility>(
    Ability as AbilityClass<AppAbility>
  );

  if (user.role === 'admin') {
    // 管理员可以做任何事
    can('manage', 'all');
  } else if (user.role === 'editor') {
    // 编辑可以管理文章
    can('create', 'Post');
    can('read', 'Post');
    can('update', 'Post', { authorId: user.id }); // 只能更新自己的
    can('delete', 'Post', { authorId: user.id });

    // 可以读取用户
    can('read', 'User');
  } else {
    // 普通用户
    can('read', 'Post', { published: true }); // 只能读已发布的
    can('create', 'Post');
    can('update', 'Post', { authorId: user.id });
    can('delete', 'Post', { authorId: user.id });
    can('read', 'User', { id: user.id }); // 只能读自己
  }

  return build();
}

// Guard
@Injectable()
export class CaslAbilityGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const resource = request.resource; // 需要提前加载

    // 获取装饰器定义的操作
    const reflector = new Reflector();
    const action = reflector.get<Actions>('action', context.getHandler());
    const subject = reflector.get<Subjects>('subject', context.getHandler());

    if (!action || !subject) {
      return true;
    }

    // 定义用户权限
    const ability = defineAbilityFor(user);

    // 检查权限
    return ability.can(action, subject, resource);
  }
}

// 装饰器
export const CheckAbility = (action: Actions, subject: Subjects) => {
  return applyDecorators(
    SetMetadata('action', action),
    SetMetadata('subject', subject),
    UseGuards(CaslAbilityGuard)
  );
};

// 使用
@Controller('posts')
export class PostsController {
  @Get(':id')
  @CheckAbility('read', 'Post')
  async findOne(@Param('id') id: string) {
    return this.postsService.findOne(+id);
  }

  @Put(':id')
  @CheckAbility('update', 'Post')
  async update(@Param('id') id: string, @Body() dto: UpdatePostDto) {
    return this.postsService.update(+id, dto);
  }
}
```

---

## 最佳实践

### 1. 最小权限原则

```typescript
// ✅ 好的做法：细粒度权限
const permissions = [
  'post:create',
  'post:read',
  'post:update',
  'post:delete'
];

// ❌ 不好的做法：粗粒度权限
const permissions = ['post:*']; // 太宽泛
```

### 2. 权限缓存

```typescript
// 缓存用户权限，避免每次查数据库
const getUserPermissions = async (userId: number): Promise<string[]> => {
  const cacheKey = `permissions:${userId}`;
  
  let permissions = await redis.get(cacheKey);
  if (permissions) {
    return JSON.parse(permissions);
  }

  permissions = await loadPermissionsFromDB(userId);
  await redis.setex(cacheKey, 300, JSON.stringify(permissions)); // 5 分钟

  return permissions;
};
```

### 3. 审计日志

```typescript
// 记录权限检查结果
async function logPermissionCheck(
  userId: number,
  resource: string,
  action: string,
  allowed: boolean
) {
  await prisma.auditLog.create({
    data: {
      userId,
      action: `${resource}:${action}`,
      result: allowed ? 'allowed' : 'denied',
      timestamp: new Date(),
      ip: req.ip,
      userAgent: req.headers['user-agent']
    }
  });
}
```

### 4. 权限测试

```typescript
import { Test } from '@nestjs/testing';

describe('RBAC Authorization', () => {
  let rbacService: RBACService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [RBACService, PrismaService]
    }).compile();

    rbacService = module.get(RBACService);
  });

  it('admin should have all permissions', async () => {
    const hasPermission = await rbacService.hasPermission(
      adminUser.id,
      'post',
      'delete'
    );

    expect(hasPermission).toBe(true);
  });

  it('user should not delete others posts', async () => {
    const hasPermission = await rbacService.hasPermission(
      regularUser.id,
      'post',
      'delete'
    );

    expect(hasPermission).toBe(false);
  });
});
```

---

## 常见面试题

### 1. RBAC vs ABAC，如何选择？

| 场景 | 推荐 | 理由 |
|------|------|------|
| 简单应用 | RBAC | 实现简单，易维护 |
| 复杂业务逻辑 | ABAC | 灵活，可表达复杂规则 |
| 组织结构明确 | RBAC | 映射组织架构 |
| 动态权限 | ABAC | 基于运行时属性 |
| 合规要求高 | ABAC | 细粒度控制 |

### 2. 如何设计可扩展的权限系统？

**要点**：

1. **分离关注点**：认证、授权、审计分离
2. **细粒度权限**：资源 + 操作
3. **角色继承**：支持角色层次
4. **策略引擎**：外部化权限规则
5. **缓存**：减少数据库查询
6. **审计**：记录所有权限检查

### 3. 如何实现动态权限？

**方案**：

1. **数据库驱动**：权限配置存数据库，运行时加载
2. **策略引擎**：使用规则引擎（如 ABAC）
3. **插件系统**：动态加载权限模块
4. **配置文件**：热加载配置

```typescript
// 示例：动态加载权限
const permissions = await loadPermissionsFromDB();
const policyEngine = new PolicyEngine(permissions);

app.use((req, res, next) => {
  const allowed = policyEngine.evaluate(req.user, req.resource, req.action);
  if (!allowed) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  next();
});
```

### 4. 如何处理数据级权限？

**方法**：

1. **查询过滤**：在 WHERE 子句中添加权限条件
2. **ORM 中间件**：自动注入过滤条件（如 Prisma middleware）
3. **视图**：数据库视图实现 RLS
4. **后处理**：查询后过滤（性能差，不推荐）

### 5. 权限系统的性能优化？

**优化点**：

1. **缓存**：Redis 缓存用户权限（5-10 分钟）
2. **批量查询**：DataLoader 批量加载权限
3. **索引**：权限表添加索引
4. **预计算**：提前计算并存储用户权限
5. **降级**：权限服务不可用时的降级策略

---

## ReBAC（基于关系的访问控制）

### ReBAC 简介

ReBAC（Relationship-Based Access Control）基于实体之间的关系来确定访问权限，特别适用于社交应用、协作工具等场景。

```
核心概念：
- 主体（Subject）：请求访问的用户
- 对象（Object）：被访问的资源
- 关系（Relation）：主体和对象之间的关系

示例：
- user:alice 是 doc:readme 的 owner
- user:bob 是 folder:engineering 的 viewer
- folder:engineering 包含 doc:readme
- 因此 user:bob 可以查看 doc:readme
```

### ReBAC 数据模型

```typescript
// Prisma Schema
model Entity {
  id        String   @id @default(uuid())
  namespace String   // user, doc, folder, org
  objectId  String   // 实际的 ID
  
  @@unique([namespace, objectId])
}

model Relation {
  id          String   @id @default(uuid())
  subject     Entity   @relation("Subject", fields: [subjectId], references: [id])
  subjectId   String
  relation    String   // owner, editor, viewer, member, parent
  object      Entity   @relation("Object", fields: [objectId], references: [id])
  objectId    String
  
  createdAt   DateTime @default(now())
  
  @@unique([subjectId, relation, objectId])
  @@index([objectId, relation])
  @@index([subjectId])
}
```

### ReBAC 实现

```typescript
interface TupleKey {
  namespace: string;
  objectId: string;
}

class ReBAC {
  private prisma: PrismaClient;
  
  // 关系继承规则
  private relationHierarchy: Record<string, string[]> = {
    owner: ['editor', 'viewer'],
    editor: ['viewer'],
    admin: ['member'],
    parent: [] // 用于层次结构
  };

  constructor(prisma: PrismaClient) {
    this.prisma = prisma;
  }

  // 添加关系
  async addRelation(
    subject: TupleKey,
    relation: string,
    object: TupleKey
  ): Promise<void> {
    const subjectEntity = await this.getOrCreateEntity(subject);
    const objectEntity = await this.getOrCreateEntity(object);

    await this.prisma.relation.upsert({
      where: {
        subjectId_relation_objectId: {
          subjectId: subjectEntity.id,
          relation,
          objectId: objectEntity.id
        }
      },
      update: {},
      create: {
        subjectId: subjectEntity.id,
        relation,
        objectId: objectEntity.id
      }
    });
  }

  // 删除关系
  async removeRelation(
    subject: TupleKey,
    relation: string,
    object: TupleKey
  ): Promise<void> {
    const subjectEntity = await this.findEntity(subject);
    const objectEntity = await this.findEntity(object);

    if (subjectEntity && objectEntity) {
      await this.prisma.relation.deleteMany({
        where: {
          subjectId: subjectEntity.id,
          relation,
          objectId: objectEntity.id
        }
      });
    }
  }

  // 检查权限
  async check(
    subject: TupleKey,
    relation: string,
    object: TupleKey
  ): Promise<boolean> {
    // 1. 直接关系检查
    const directCheck = await this.checkDirect(subject, relation, object);
    if (directCheck) return true;

    // 2. 关系继承检查
    const inheritedRelations = this.getInheritingRelations(relation);
    for (const inheritedRelation of inheritedRelations) {
      const inherited = await this.checkDirect(subject, inheritedRelation, object);
      if (inherited) return true;
    }

    // 3. 层次结构检查（如文件夹权限继承）
    const parentObjects = await this.getParentObjects(object);
    for (const parent of parentObjects) {
      const hasParentAccess = await this.check(subject, relation, parent);
      if (hasParentAccess) return true;
    }

    // 4. 组成员检查（用户组权限）
    const groups = await this.getGroups(subject);
    for (const group of groups) {
      const hasGroupAccess = await this.check(group, relation, object);
      if (hasGroupAccess) return true;
    }

    return false;
  }

  // 直接关系检查
  private async checkDirect(
    subject: TupleKey,
    relation: string,
    object: TupleKey
  ): Promise<boolean> {
    const subjectEntity = await this.findEntity(subject);
    const objectEntity = await this.findEntity(object);

    if (!subjectEntity || !objectEntity) return false;

    const exists = await this.prisma.relation.findFirst({
      where: {
        subjectId: subjectEntity.id,
        relation,
        objectId: objectEntity.id
      }
    });

    return !!exists;
  }

  // 获取继承的关系
  private getInheritingRelations(relation: string): string[] {
    const inheriting: string[] = [];
    for (const [higherRelation, lowerRelations] of Object.entries(this.relationHierarchy)) {
      if (lowerRelations.includes(relation)) {
        inheriting.push(higherRelation);
      }
    }
    return inheriting;
  }

  // 获取父对象
  private async getParentObjects(object: TupleKey): Promise<TupleKey[]> {
    const objectEntity = await this.findEntity(object);
    if (!objectEntity) return [];

    const parentRelations = await this.prisma.relation.findMany({
      where: {
        subjectId: objectEntity.id,
        relation: 'parent'
      },
      include: { object: true }
    });

    return parentRelations.map(r => ({
      namespace: r.object.namespace,
      objectId: r.object.objectId
    }));
  }

  // 获取用户所属的组
  private async getGroups(subject: TupleKey): Promise<TupleKey[]> {
    if (subject.namespace !== 'user') return [];

    const subjectEntity = await this.findEntity(subject);
    if (!subjectEntity) return [];

    const groupRelations = await this.prisma.relation.findMany({
      where: {
        subjectId: subjectEntity.id,
        relation: 'member'
      },
      include: { object: true }
    });

    return groupRelations
      .filter(r => r.object.namespace === 'group')
      .map(r => ({
        namespace: r.object.namespace,
        objectId: r.object.objectId
      }));
  }

  // 列出用户可访问的对象
  async listObjects(
    subject: TupleKey,
    relation: string,
    objectNamespace: string
  ): Promise<string[]> {
    // 实际实现需要优化，这里是简化版本
    const allObjects = await this.prisma.entity.findMany({
      where: { namespace: objectNamespace }
    });

    const accessibleObjects: string[] = [];
    for (const obj of allObjects) {
      const hasAccess = await this.check(
        subject,
        relation,
        { namespace: obj.namespace, objectId: obj.objectId }
      );
      if (hasAccess) {
        accessibleObjects.push(obj.objectId);
      }
    }

    return accessibleObjects;
  }

  private async getOrCreateEntity(key: TupleKey): Promise<Entity> {
    return this.prisma.entity.upsert({
      where: {
        namespace_objectId: {
          namespace: key.namespace,
          objectId: key.objectId
        }
      },
      update: {},
      create: {
        namespace: key.namespace,
        objectId: key.objectId
      }
    });
  }

  private async findEntity(key: TupleKey): Promise<Entity | null> {
    return this.prisma.entity.findUnique({
      where: {
        namespace_objectId: {
          namespace: key.namespace,
          objectId: key.objectId
        }
      }
    });
  }
}

// 使用示例
const rebac = new ReBAC(prisma);

// 设置关系
await rebac.addRelation(
  { namespace: 'user', objectId: 'alice' },
  'owner',
  { namespace: 'doc', objectId: 'readme' }
);

await rebac.addRelation(
  { namespace: 'doc', objectId: 'readme' },
  'parent',
  { namespace: 'folder', objectId: 'engineering' }
);

await rebac.addRelation(
  { namespace: 'user', objectId: 'bob' },
  'viewer',
  { namespace: 'folder', objectId: 'engineering' }
);

// 检查权限
const canAliceEdit = await rebac.check(
  { namespace: 'user', objectId: 'alice' },
  'editor', // owner 继承 editor
  { namespace: 'doc', objectId: 'readme' }
); // true

const canBobView = await rebac.check(
  { namespace: 'user', objectId: 'bob' },
  'viewer',
  { namespace: 'doc', objectId: 'readme' }
); // true (通过 folder 权限继承)

// Express 中间件
const requireReBAC = (relation: string, getObject: (req: Request) => TupleKey) => {
  return async (req, res, next) => {
    const subject = {
      namespace: 'user',
      objectId: req.user.id.toString()
    };
    const object = getObject(req);

    const hasAccess = await rebac.check(subject, relation, object);

    if (!hasAccess) {
      return res.status(403).json({ error: 'Forbidden' });
    }

    next();
  };
};

// 使用
app.get('/docs/:id',
  requireAuth,
  requireReBAC('viewer', (req) => ({
    namespace: 'doc',
    objectId: req.params.id
  })),
  getDocument
);
```

---

## Policy as Code（OPA/Rego）

### OPA 简介

Open Policy Agent (OPA) 是一个通用的策略引擎，使用 Rego 语言定义策略。

```
优势：
- 策略与代码分离
- 声明式策略语言
- 高性能（编译为 WASM）
- 丰富的生态系统
```

### Rego 策略示例

```rego
# policy.rego
package authz

import future.keywords.if
import future.keywords.in

# 默认拒绝
default allow := false

# 管理员可以做任何事
allow if {
    input.user.role == "admin"
}

# 用户可以读取自己的资源
allow if {
    input.action == "read"
    input.resource.owner == input.user.id
}

# 用户可以创建资源
allow if {
    input.action == "create"
    input.user.role in ["user", "editor", "admin"]
}

# 编辑可以更新草稿
allow if {
    input.action == "update"
    input.user.role == "editor"
    input.resource.status == "draft"
}

# 用户可以更新自己的资源
allow if {
    input.action == "update"
    input.resource.owner == input.user.id
}

# 只有管理员可以删除
allow if {
    input.action == "delete"
    input.user.role == "admin"
}

# 用户可以删除自己的草稿
allow if {
    input.action == "delete"
    input.resource.owner == input.user.id
    input.resource.status == "draft"
}

# 复杂规则：工作时间内才能发布
allow if {
    input.action == "publish"
    input.user.role in ["editor", "admin"]
    is_working_hours
}

is_working_hours if {
    now := time.now_ns()
    hour := time.clock([now, "Asia/Shanghai"])[0]
    hour >= 9
    hour < 18
    weekday := time.weekday(now)
    weekday != "Saturday"
    weekday != "Sunday"
}

# 基于属性的规则
allow if {
    input.action == "view_salary"
    input.user.department == "hr"
    input.user.level >= 3
}

# 数据过滤规则
filtered_data[field] = value if {
    input.action == "read"
    some field, value in input.resource.data
    can_see_field(input.user.role, field)
}

can_see_field(role, field) if {
    role == "admin"
}

can_see_field(role, field) if {
    role == "user"
    not field in ["salary", "ssn", "phone"]
}
```

### OPA 集成

```typescript
import { OpaClient } from '@open-policy-agent/opa-wasm';
import fs from 'fs';

class OPAService {
  private policy: OpaClient | null = null;

  async init() {
    // 加载编译后的 WASM
    const wasmBundle = fs.readFileSync('./policy.wasm');
    this.policy = await OpaClient.fromBundle(wasmBundle);
  }

  async evaluate(input: PolicyInput): Promise<PolicyResult> {
    if (!this.policy) {
      throw new Error('OPA not initialized');
    }

    const result = await this.policy.evaluate(input, 'authz/allow');
    return result;
  }

  async isAllowed(
    user: User,
    action: string,
    resource: Resource
  ): Promise<boolean> {
    const input = {
      user: {
        id: user.id,
        role: user.role,
        department: user.department,
        level: user.level
      },
      action,
      resource: {
        type: resource.type,
        id: resource.id,
        owner: resource.ownerId,
        status: resource.status,
        data: resource.data
      }
    };

    const result = await this.evaluate(input);
    return result === true;
  }

  async filterData(
    user: User,
    resource: Resource
  ): Promise<Record<string, any>> {
    const input = {
      user: {
        id: user.id,
        role: user.role
      },
      action: 'read',
      resource: {
        data: resource.data
      }
    };

    const result = await this.policy!.evaluate(input, 'authz/filtered_data');
    return result as Record<string, any>;
  }
}

const opaService = new OPAService();
await opaService.init();

// Express 中间件
const requireOPA = (action: string) => {
  return async (req, res, next) => {
    const resource = req.resource; // 需要先加载资源

    const allowed = await opaService.isAllowed(
      req.user,
      action,
      resource
    );

    if (!allowed) {
      return res.status(403).json({ error: 'Forbidden by policy' });
    }

    next();
  };
};

// 使用
app.put('/api/posts/:id',
  requireAuth,
  loadResource('post'),
  requireOPA('update'),
  updatePost
);

app.get('/api/users/:id',
  requireAuth,
  async (req, res) => {
    const user = await prisma.user.findUnique({
      where: { id: parseInt(req.params.id) }
    });

    // 过滤敏感字段
    const filteredData = await opaService.filterData(req.user, {
      type: 'user',
      id: user.id,
      ownerId: user.id,
      data: user
    });

    res.json(filteredData);
  }
);
```

### OPA REST API 集成

```typescript
import axios from 'axios';

class OPARestClient {
  private baseUrl: string;

  constructor(baseUrl: string = 'http://localhost:8181') {
    this.baseUrl = baseUrl;
  }

  async query(path: string, input: any): Promise<any> {
    const response = await axios.post(
      `${this.baseUrl}/v1/data/${path}`,
      { input },
      {
        headers: { 'Content-Type': 'application/json' }
      }
    );

    return response.data.result;
  }

  async isAllowed(input: any): Promise<boolean> {
    const result = await this.query('authz/allow', input);
    return result === true;
  }
}

// 使用 OPA 服务
const opaClient = new OPARestClient();

const allowed = await opaClient.isAllowed({
  user: { id: '123', role: 'editor' },
  action: 'update',
  resource: { type: 'post', status: 'draft' }
});
```

---

## API 网关授权

### API Gateway 授权模式

```
模式 1：网关集中授权
┌─────────┐     ┌──────────┐     ┌─────────┐
│ Client  │────▶│ Gateway  │────▶│ Service │
└─────────┘     │ (Auth)   │     └─────────┘
                └──────────┘

模式 2：网关 + 服务授权
┌─────────┐     ┌──────────┐     ┌─────────┐
│ Client  │────▶│ Gateway  │────▶│ Service │
└─────────┘     │ (AuthN)  │     │ (AuthZ) │
                └──────────┘     └─────────┘

模式 3：Sidecar 模式（Service Mesh）
┌─────────┐     ┌──────────────────────────┐
│ Client  │────▶│ ┌───────┐  ┌───────────┐ │
└─────────┘     │ │Envoy  │──│  Service  │ │
                │ │(Auth) │  └───────────┘ │
                │ └───────┘                │
                └──────────────────────────┘
```

### Kong 授权插件

```yaml
# kong.yml
_format_version: "2.1"

services:
  - name: api-service
    url: http://api:3000

routes:
  - name: api-route
    service: api-service
    paths:
      - /api

plugins:
  # JWT 认证
  - name: jwt
    route: api-route
    config:
      claims_to_verify:
        - exp
      header_names:
        - Authorization

  # ACL 授权
  - name: acl
    route: api-route
    config:
      allow:
        - admin
        - user

  # Rate Limiting
  - name: rate-limiting
    route: api-route
    config:
      minute: 100
      policy: local

  # OPA 授权
  - name: opa
    route: api-route
    config:
      opa_url: http://opa:8181/v1/data/authz/allow
```

### Express Gateway 授权

```javascript
// gateway.config.yml
http:
  port: 8080

apiEndpoints:
  api:
    host: '*'
    paths: '/api/*'

serviceEndpoints:
  backend:
    url: 'http://backend:3000'

policies:
  - jwt
  - rbac
  - proxy

pipelines:
  default:
    apiEndpoints:
      - api
    policies:
      # JWT 验证
      - jwt:
          action:
            secretOrPublicKey: ${JWT_SECRET}
            checkCredentialExistence: true

      # RBAC 检查
      - rbac:
          action:
            roles:
              admin:
                - '*'
              user:
                - 'GET /api/posts/*'
                - 'POST /api/posts'
              viewer:
                - 'GET /api/posts/*'

      # 代理到后端
      - proxy:
          action:
            serviceEndpoint: backend
```

### 自定义网关授权

```typescript
import express from 'express';
import httpProxy from 'http-proxy-middleware';

const app = express();

// 服务路由配置
const serviceRoutes = {
  '/api/users': 'http://user-service:3001',
  '/api/posts': 'http://post-service:3002',
  '/api/orders': 'http://order-service:3003'
};

// 路由权限配置
const routePermissions = {
  'GET /api/users': ['admin', 'user'],
  'POST /api/users': ['admin'],
  'DELETE /api/users/:id': ['admin'],
  'GET /api/posts': ['admin', 'user', 'viewer'],
  'POST /api/posts': ['admin', 'user'],
  'PUT /api/posts/:id': ['admin', 'user'],
  'DELETE /api/posts/:id': ['admin'],
  'GET /api/orders': ['admin', 'user'],
  'POST /api/orders': ['user']
};

// JWT 验证中间件
async function authenticateJWT(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// RBAC 中间件
function authorizeRBAC(req, res, next) {
  const method = req.method;
  const path = req.path;
  
  // 查找匹配的路由规则
  const matchedRoute = findMatchingRoute(method, path);
  
  if (!matchedRoute) {
    // 默认拒绝未定义的路由
    return res.status(403).json({ error: 'Route not allowed' });
  }

  const allowedRoles = routePermissions[matchedRoute];
  
  if (!allowedRoles.includes(req.user.role)) {
    return res.status(403).json({ error: 'Insufficient permissions' });
  }

  next();
}

function findMatchingRoute(method: string, path: string): string | null {
  const routeKey = `${method} ${path}`;
  
  for (const [pattern, roles] of Object.entries(routePermissions)) {
    const regex = patternToRegex(pattern);
    if (regex.test(routeKey)) {
      return pattern;
    }
  }
  
  return null;
}

function patternToRegex(pattern: string): RegExp {
  const [method, path] = pattern.split(' ');
  const regexPath = path
    .replace(/:\w+/g, '[^/]+')
    .replace(/\*/g, '.*');
  return new RegExp(`^${method} ${regexPath}$`);
}

// 速率限制
const rateLimiters = new Map<string, RateLimiter>();

function rateLimit(req, res, next) {
  const key = `${req.user?.id || req.ip}:${req.path}`;
  
  let limiter = rateLimiters.get(key);
  if (!limiter) {
    limiter = new RateLimiter(100, 60000); // 100 req/min
    rateLimiters.set(key, limiter);
  }

  if (!limiter.tryConsume()) {
    return res.status(429).json({ error: 'Too many requests' });
  }

  next();
}

// 审计日志
function auditLog(req, res, next) {
  const startTime = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    
    logger.info('API Request', {
      method: req.method,
      path: req.path,
      userId: req.user?.id,
      role: req.user?.role,
      statusCode: res.statusCode,
      duration,
      ip: req.ip,
      userAgent: req.headers['user-agent']
    });
  });

  next();
}

// 设置代理
for (const [path, target] of Object.entries(serviceRoutes)) {
  app.use(
    path,
    authenticateJWT,
    authorizeRBAC,
    rateLimit,
    auditLog,
    httpProxy.createProxyMiddleware({
      target,
      changeOrigin: true,
      pathRewrite: { [`^${path}`]: '' },
      onProxyReq: (proxyReq, req) => {
        // 将用户信息传递给后端服务
        if (req.user) {
          proxyReq.setHeader('X-User-Id', req.user.id);
          proxyReq.setHeader('X-User-Role', req.user.role);
          proxyReq.setHeader('X-User-Email', req.user.email);
        }
      }
    })
  );
}

app.listen(8080, () => {
  console.log('API Gateway running on port 8080');
});
```

### 服务间授权（mTLS + JWT）

```typescript
import https from 'https';
import fs from 'fs';

// 服务间通信的 JWT
class ServiceToken {
  private privateKey: string;
  private publicKey: string;
  private serviceName: string;

  constructor(serviceName: string) {
    this.serviceName = serviceName;
    this.privateKey = fs.readFileSync(`./certs/${serviceName}-private.pem`, 'utf8');
    this.publicKey = fs.readFileSync(`./certs/${serviceName}-public.pem`, 'utf8');
  }

  // 生成服务间调用 Token
  generateToken(targetService: string): string {
    return jwt.sign(
      {
        iss: this.serviceName,
        aud: targetService,
        iat: Math.floor(Date.now() / 1000),
        exp: Math.floor(Date.now() / 1000) + 300 // 5 分钟
      },
      this.privateKey,
      { algorithm: 'RS256' }
    );
  }

  // 验证服务间 Token
  verifyToken(token: string, expectedIssuer: string): any {
    const publicKey = fs.readFileSync(
      `./certs/${expectedIssuer}-public.pem`,
      'utf8'
    );
    
    return jwt.verify(token, publicKey, {
      algorithms: ['RS256'],
      audience: this.serviceName
    });
  }
}

// mTLS 配置
const tlsOptions = {
  key: fs.readFileSync('./certs/service-key.pem'),
  cert: fs.readFileSync('./certs/service-cert.pem'),
  ca: fs.readFileSync('./certs/ca-cert.pem'),
  requestCert: true,
  rejectUnauthorized: true
};

// 创建 HTTPS 服务
https.createServer(tlsOptions, app).listen(443);

// 验证客户端证书
app.use((req, res, next) => {
  const cert = req.socket.getPeerCertificate();
  
  if (!cert || !cert.subject) {
    return res.status(401).json({ error: 'Client certificate required' });
  }

  // 验证证书 CN
  const allowedServices = ['api-gateway', 'user-service', 'order-service'];
  const serviceName = cert.subject.CN;
  
  if (!allowedServices.includes(serviceName)) {
    return res.status(403).json({ error: 'Unknown service' });
  }

  req.clientService = serviceName;
  next();
});

// 验证服务间 JWT
app.use((req, res, next) => {
  const serviceToken = req.headers['x-service-token'] as string;
  
  if (!serviceToken) {
    return res.status(401).json({ error: 'Service token required' });
  }

  try {
    const payload = serviceTokenManager.verifyToken(
      serviceToken,
      req.clientService
    );
    req.callingService = payload.iss;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid service token' });
  }
});
```

---

## 总结

### 授权模型选择

| 模型 | 适用场景 | 复杂度 | 灵活性 |
|------|---------|--------|--------|
| **ACL** | 简单应用 | ⭐ | ⭐⭐ |
| **RBAC** | 大多数应用 | ⭐⭐ | ⭐⭐⭐ |
| **ABAC** | 复杂业务 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ReBAC** | 社交/协作应用 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Policy as Code** | 企业级应用 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

### 实践检查清单

- [ ] 是否使用 RBAC 或 ABAC？
- [ ] 权限粒度是否合适？
- [ ] 是否支持角色继承？
- [ ] 是否缓存权限？
- [ ] 是否有审计日志？
- [ ] 是否实现数据级权限？
- [ ] 是否有权限测试？
- [ ] 权限变更如何通知？
- [ ] 如何处理权限冲突？
- [ ] 是否有权限管理界面？
- [ ] 是否考虑 ReBAC 模型？
- [ ] 是否使用 Policy as Code？
- [ ] API 网关授权是否完善？
- [ ] 服务间通信是否安全？

### 高级授权面试题

#### 6. ReBAC vs RBAC vs ABAC？

| 特性 | RBAC | ABAC | ReBAC |
|------|------|------|-------|
| **核心** | 角色 | 属性 | 关系 |
| **灵活性** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **表达能力** | 有限 | 强大 | 自然 |
| **适用场景** | 组织权限 | 复杂规则 | 社交/协作 |

**ReBAC 优势**：
- 自然表达"所有者"、"成员"等关系
- 支持权限继承（文件夹→文件）
- 易于理解和维护

#### 7. 什么是 Policy as Code？

**概念**：将授权策略定义为代码，与应用代码分离管理。

**优势**：
- 📝 声明式定义，易于理解
- 🔄 策略热更新，无需重启服务
- 🧪 策略可测试、可版本控制
- 🔒 审计友好

**工具**：
- OPA（Open Policy Agent）
- Casbin
- AWS IAM Policy
- HashiCorp Sentinel

#### 8. 微服务间如何授权？

**方案**：

1. **JWT 传递用户身份**
```typescript
// 网关验证后，将用户信息放入 Header
proxyReq.setHeader('X-User-Id', user.id);
proxyReq.setHeader('X-User-Role', user.role);
```

2. **服务间 mTLS**
```typescript
// 使用客户端证书验证服务身份
const cert = req.socket.getPeerCertificate();
const serviceName = cert.subject.CN;
```

3. **服务间 JWT**
```typescript
// 调用其他服务时生成短期 Token
const serviceToken = generateServiceToken(targetService);
```

4. **Policy as Code（集中管理）**
```rego
# 服务 A 可以调用服务 B 的 API
allow if {
    input.caller == "service-a"
    input.callee == "service-b"
    input.action in ["read", "list"]
}
```

---

**上一篇**：[认证机制](./01-authentication.md)  
**下一篇**：[Web 安全](./03-web-security.md)

