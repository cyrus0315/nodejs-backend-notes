# Node.js 框架

深入学习 Node.js 主流框架，重点是 NestJS 企业级应用开发。

## 📚 学习路径

### 第一阶段：NestJS 核心（必修）

- [ ] **基础架构**
  - [ ] 模块（Modules）系统
  - [ ] 依赖注入（DI）和 IoC 容器
  - [ ] 提供者（Providers）和作用域
  - [ ] 控制器（Controllers）和路由
  - [ ] 动态模块

- [ ] **请求生命周期组件**
  - [ ] 中间件（Middleware）
  - [ ] 守卫（Guards）- 认证授权
  - [ ] 拦截器（Interceptors）- AOP
  - [ ] 管道（Pipes）- 验证转换
  - [ ] 异常过滤器（Exception Filters）

- [ ] **装饰器系统**
  - [ ] 参数装饰器
  - [ ] 路由装饰器
  - [ ] 自定义装饰器
  - [ ] 装饰器组合

### 第二阶段：NestJS 高级特性（深入）

- [ ] **微服务架构**
  - [ ] TCP 微服务
  - [ ] Redis 传输层
  - [ ] RabbitMQ 集成
  - [ ] Kafka 集成
  - [ ] 混合应用（HTTP + 微服务）

- [ ] **实时通信**
  - [ ] WebSocket Gateway
  - [ ] Socket.io 集成
  - [ ] Redis 适配器（多实例）
  - [ ] 事件和房间管理

- [ ] **GraphQL**
  - [ ] Code First vs Schema First
  - [ ] Resolver 和 Field Resolver
  - [ ] DataLoader（N+1 问题）
  - [ ] 订阅（Subscriptions）
  - [ ] 复杂查询优化

- [ ] **任务和队列**
  - [ ] 定时任务（Cron、Interval、Timeout）
  - [ ] Bull 队列
  - [ ] 任务重试和优先级
  - [ ] 队列监控

- [ ] **其他特性**
  - [ ] 缓存管理
  - [ ] 配置管理（@nestjs/config）
  - [ ] 文件上传
  - [ ] 流式响应
  - [ ] Server-Sent Events（SSE）

### 第三阶段：NestJS 底层原理（深入）

- [ ] **装饰器原理**
  - [ ] 装饰器执行顺序
  - [ ] 元数据存储和读取
  - [ ] 自定义装饰器实现
  - [ ] Reflect Metadata API

- [ ] **依赖注入原理**
  - [ ] IoC 容器实现
  - [ ] 作用域管理
  - [ ] 循环依赖处理（forwardRef）
  - [ ] 提供者解析流程

- [ ] **模块系统原理**
  - [ ] 模块扫描和加载
  - [ ] 动态模块实现
  - [ ] 模块导入导出机制
  - [ ] 模块图构建

- [ ] **请求处理原理**
  - [ ] 完整的请求管道
  - [ ] 参数解析机制
  - [ ] 路由注册流程
  - [ ] 生命周期钩子

- [ ] **AOP 实现**
  - [ ] 拦截器链构建
  - [ ] RxJS 管道
  - [ ] 横切关注点分离

### 第四阶段：框架对比（必会）

- [ ] **Express**
  - [ ] 中间件机制
  - [ ] 路由系统
  - [ ] 优缺点和适用场景

- [ ] **Fastify**
  - [ ] 高性能原理
  - [ ] Schema 验证
  - [ ] 插件架构
  - [ ] 与 Express 对比

- [ ] **Koa**
  - [ ] 洋葱模型
  - [ ] Context 对象
  - [ ] 中间件设计
  - [ ] 与 Express 对比

- [ ] **框架选型**
  - [ ] 按项目规模选择
  - [ ] 性能对比
  - [ ] 生态系统对比
  - [ ] 迁移策略

### 第五阶段：Monorepo（企业必备）

- [ ] **Monorepo 概念**
  - [ ] Monorepo vs Multirepo
  - [ ] 适用场景
  - [ ] 优势与劣势

- [ ] **Nx**
  - [ ] 创建 Nx 工作区
  - [ ] 构建缓存机制
  - [ ] 智能任务调度
  - [ ] 依赖图可视化
  - [ ] 只构建受影响的项目

- [ ] **Turborepo**
  - [ ] 创建 Turborepo 项目
  - [ ] 极速构建
  - [ ] 远程缓存
  - [ ] Pipeline 配置

- [ ] **pnpm Workspaces**
  - [ ] Workspace 配置
  - [ ] 包之间依赖管理
  - [ ] 命令执行

- [ ] **实战应用**
  - [ ] NestJS Monorepo 架构
  - [ ] 共享库设计
  - [ ] TypeScript 配置
  - [ ] CI/CD 优化

## 📖 模块内容

### [01-nestjs-core.md](./01-nestjs-core.md)

NestJS 核心架构和基础概念。

**核心内容**：
- 模块系统和动态模块
- 依赖注入和作用域
- 控制器和路由
- 提供者（Providers）
- 中间件、守卫、拦截器、管道、过滤器
- 装饰器系统
- 应用启动配置

**面试重点**：
- NestJS 架构设计
- 依赖注入原理和作用域
- 请求生命周期
- 中间件 vs 守卫 vs 拦截器区别
- 动态模块实现

### [02-nestjs-advanced.md](./02-nestjs-advanced.md)

NestJS 高级特性和实战应用。

**核心内容**：
- 微服务架构（TCP、Redis、RabbitMQ、Kafka）
- WebSocket 和实时通信
- GraphQL 集成
- 任务调度
- Bull 队列
- 缓存管理
- 配置管理

**面试重点**：
- 微服务架构优缺点
- 分布式事务处理
- WebSocket vs HTTP
- GraphQL vs REST
- 队列应用场景
- 性能优化策略

### [03-framework-comparison.md](./03-framework-comparison.md)

Node.js 主流框架对比和选型建议。

**核心内容**：
- Express、Fastify、Koa、NestJS 对比
- 性能基准测试
- 代码风格对比
- 适用场景分析
- 框架迁移策略

**面试重点**：
- Fastify 为什么快？
- Koa 洋葱模型优势
- 何时选择 NestJS？
- 框架性能优化
- 技术选型依据

### [04-nestjs-internals.md](./04-nestjs-internals.md)

NestJS 底层实现原理和核心机制。

**核心内容**：
- 装饰器原理和元数据
- 依赖注入实现（IoC 容器）
- 模块系统加载流程
- 请求处理管道
- 生命周期钩子
- AOP 实现原理
- 循环依赖处理
- 性能优化原理

**面试重点**：
- 依赖注入如何实现？
- 装饰器执行顺序
- 模块加载流程
- 请求处理流程
- forwardRef 原理
- AOP 实现机制

### [05-monorepo.md](./05-monorepo.md) ⭐ 新增

Monorepo 单仓多包管理。

**核心内容**：
- Monorepo 概念和适用场景
- Nx 深度实战（构建缓存、任务调度、依赖图）
- Turborepo 实战（极速构建、远程缓存）
- pnpm Workspaces 使用
- NestJS Monorepo 完整示例
- 共享库设计和使用
- CI/CD 优化策略
- 最佳实践

**面试重点**：
- Monorepo vs Multirepo
- Nx vs Turborepo 选择
- 如何优化 Monorepo CI/CD
- TypeScript 配置共享
- 版本发布策略

## 🎯 面试高频问题

### 1. NestJS 的核心架构是什么？

**关键概念**：
- **模块化**：@Module 装饰器组织代码
- **依赖注入**：IoC 容器管理依赖
- **装饰器**：元编程，声明式配置
- **请求生命周期**：Middleware → Guards → Interceptors → Pipes → Controller → Service

**优势**：
- 清晰的架构约束
- 高可测试性
- 易于维护和扩展
- TypeScript 原生支持

### 2. 依赖注入的三种作用域？

| 作用域 | 生命周期 | 性能 | 使用场景 |
|--------|----------|------|----------|
| DEFAULT | 单例 | 最高 | 无状态服务 |
| REQUEST | 每个请求 | 较低 | 需要请求上下文 |
| TRANSIENT | 每次注入 | 最低 | 需要独立状态 |

**注意**：
- REQUEST 作用域会传播到整个依赖链
- 性能敏感场景优先 DEFAULT
- 多租户系统可能需要 REQUEST

### 3. 守卫（Guards）vs 拦截器（Interceptors）的区别？

**执行顺序**：
```
Middleware → Guards → Interceptors (before) 
  → Pipes → Handler → Interceptors (after)
```

**区别**：
- **Guards**：权限控制，返回 boolean
- **Interceptors**：AOP，可修改请求/响应

**使用场景**：
- Guards：认证、授权、限流
- Interceptors：日志、缓存、转换、超时

### 4. 如何实现 NestJS 微服务间通信？

**传输层选项**：
1. **TCP**：简单直接
2. **Redis**：发布/订阅
3. **RabbitMQ**：可靠消息队列
4. **Kafka**：高吞吐事件流

**通信模式**：
- **Request-Response**：`send()` - 等待响应
- **Event-Based**：`emit()` - 不等待响应

**示例**：
```typescript
// 发送消息并等待响应
const result = await client.send('pattern', data).toPromise();

// 发送事件，不等待响应
client.emit('event', data);
```

### 5. Express vs Fastify vs NestJS 如何选择？

| 场景 | 推荐 | 原因 |
|------|------|------|
| 快速原型 | Express | 简单、生态好 |
| 高性能API | Fastify | 性能2倍于Express |
| 企业应用 | NestJS | 完整架构、DI |
| 小型项目 | Express/Koa | 轻量、灵活 |
| 微服务 | NestJS | 多传输层支持 |
| 学习Node.js | Express | 最成熟 |

### 6. 如何优化 NestJS 性能？

**策略**：

1. **使用 Fastify 替代 Express**：
```typescript
const app = await NestFactory.create<NestFastifyApplication>(
  AppModule,
  new FastifyAdapter()
);
```

2. **合理使用作用域**：
- 优先使用 DEFAULT（单例）
- 避免不必要的 REQUEST 作用域

3. **启用缓存**：
```typescript
@UseInterceptors(CacheInterceptor)
@CacheTTL(300)
```

4. **数据库连接池**：
```typescript
{
  type: 'postgres',
  poolSize: 20
}
```

5. **批量操作**：
- 使用 DataLoader 解决 N+1 问题
- 批量数据库查询

6. **压缩响应**：
```typescript
app.use(compression());
```

### 7. NestJS 中如何实现全局异常处理？

```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    const status = exception instanceof HttpException
      ? exception.getStatus()
      : HttpStatus.INTERNAL_SERVER_ERROR;
    
    const message = exception instanceof HttpException
      ? exception.message
      : 'Internal server error';
    
    // 记录日志
    console.error(exception);
    
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message
    });
  }
}

// 应用
app.useGlobalFilters(new AllExceptionsFilter());
```

### 8. 如何在 NestJS 中实现认证和授权？

**认证（Authentication）**：
```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    
    try {
      const user = await this.verifyToken(token);
      request.user = user;
      return true;
    } catch {
      throw new UnauthorizedException();
    }
  }
}
```

**授权（Authorization）**：
```typescript
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>(
      'roles',
      context.getHandler()
    );
    
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// 使用
@Controller('admin')
@UseGuards(AuthGuard, RolesGuard)
export class AdminController {
  @Get()
  @Roles('admin')
  findAll() {
    return 'Admin data';
  }
}
```

### 9. 什么时候应该使用 Monorepo？

**适合 Monorepo 的场景**：

1. **多个相关项目**：
   - 前端 + 后端 + 移动端
   - 多个微服务
   - 共享大量代码

2. **需要原子性提交**：
   - 跨项目的功能变更
   - API 和客户端同步更新

3. **团队协作紧密**：
   - 统一代码规范
   - 统一工具链

4. **企业级应用**：
   - 需要代码复用
   - 需要统一版本管理

**不适合 Monorepo**：
- 完全独立的项目
- 不同技术栈
- 需要严格权限隔离

**工具选择**：
- 大型项目 → Nx（功能最强）
- 快速上手 → Turborepo（简单快速）
- 简单需求 → pnpm workspaces（原生支持）

## 📝 学习建议

### 基础阶段
1. 从 Express 开始，理解基本概念
2. 学习 NestJS 核心架构
3. 掌握依赖注入和模块系统
4. 理解请求生命周期

### 进阶阶段
1. 深入 NestJS 高级特性
2. 学习微服务和 WebSocket
3. 对比不同框架的优劣
4. 实践 GraphQL 和队列

### 实战阶段
1. 构建完整的 REST API
2. 实现认证和授权系统
3. 集成数据库和缓存
4. 性能优化和监控

### 面试准备
1. 熟记框架核心概念
2. 准备实际项目案例
3. 理解框架选型依据
4. 掌握性能优化技巧

## ✅ 检查清单

完成以下内容后，你应该能够：

### NestJS 核心
- [ ] 解释 NestJS 核心架构
- [ ] 实现依赖注入和模块化
- [ ] 使用守卫、拦截器、管道
- [ ] 理解装饰器和元数据原理
- [ ] 解释 IoC 容器实现机制
- [ ] 理解模块加载流程
- [ ] 掌握请求处理管道
- [ ] 理解 AOP 实现原理

### NestJS 高级
- [ ] 构建微服务架构
- [ ] 集成 WebSocket 和 GraphQL
- [ ] 处理认证和授权
- [ ] 实现任务调度和队列

### 框架对比
- [ ] 对比 Express、Fastify、Koa、NestJS
- [ ] 选择合适的框架
- [ ] 优化框架性能

### Monorepo
- [ ] 理解 Monorepo 概念和适用场景
- [ ] 使用 Nx 构建 Monorepo
- [ ] 使用 Turborepo 优化构建
- [ ] 设计共享库结构
- [ ] 配置 TypeScript paths
- [ ] 优化 Monorepo CI/CD
- [ ] 选择合适的 Monorepo 工具

## 🔗 相关资源

### NestJS
- [NestJS 官方文档](https://docs.nestjs.com/)
- [NestJS 源码](https://github.com/nestjs/nest)
- [NestJS 示例项目](https://github.com/nestjs/nest/tree/master/sample)

### Express
- [Express 官方文档](https://expressjs.com/)
- [Express 最佳实践](https://expressjs.com/en/advanced/best-practice-performance.html)

### Fastify
- [Fastify 官方文档](https://www.fastify.io/)
- [Fastify 性能优化](https://www.fastify.io/docs/latest/Guides/Getting-Started/)

### Koa
- [Koa 官方文档](https://koajs.com/)
- [Koa 中间件](https://github.com/koajs/koa/wiki)

### Monorepo
- [Nx 官方文档](https://nx.dev/)
- [Turborepo 官方文档](https://turbo.build/)
- [pnpm Workspaces](https://pnpm.io/workspaces)
- [Monorepo Tools](https://monorepo.tools/)

## 🎓 面试评分标准

### 初级（0-2 年）
- 会使用 Express 或 NestJS
- 理解基本的路由和中间件
- 能实现简单的 CRUD API
- 了解框架的基本概念

### 中级（2-4 年）
- 深入理解 NestJS 架构
- 熟练使用依赖注入
- 能处理认证授权
- 掌握性能优化技巧
- 了解多个框架的优劣

### 高级（4+ 年）
- 精通框架底层原理
  - 理解装饰器和元数据机制
  - 能解释依赖注入实现
  - 掌握模块加载流程
  - 理解请求处理管道
- 能设计微服务架构
- 实现过复杂的企业应用
- 能从业务角度选型
- 有框架迁移和优化经验
- 能解决框架层面的复杂问题
- **Monorepo 经验**：
  - 熟练使用 Nx 或 Turborepo
  - 设计过共享库架构
  - 优化过 Monorepo CI/CD
  - 有大型 Monorepo 项目经验

---

**下一模块**: [06-orm-odm（ORM/ODM）](../06-orm-odm/README.md)
