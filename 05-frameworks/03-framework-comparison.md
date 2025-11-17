# 框架对比：Express vs Fastify vs Koa vs NestJS

## 概览对比

| 特性 | Express | Fastify | Koa | NestJS |
|------|---------|---------|-----|--------|
| 发布年份 | 2010 | 2016 | 2013 | 2017 |
| 性能 | 中等 | 极高 | 高 | 中等 |
| 学习曲线 | 平缓 | 中等 | 中等 | 陡峭 |
| 架构 | 无约束 | 插件化 | 中间件 | 模块化/DI |
| TypeScript | 需配置 | 原生支持 | 需配置 | 原生支持 |
| 社区生态 | 最成熟 | 快速增长 | 中等 | 快速增长 |
| 适用场景 | 通用 | 高性能API | 中间件架构 | 企业级应用 |

## Express

### 特点

**优点**：
- 最成熟的 Node.js 框架
- 丰富的中间件生态
- 简单易学
- 文档完善
- 社区支持最好

**缺点**：
- 性能较低
- 缺乏内置功能
- 回调地狱（旧版本）
- TypeScript 支持需要额外配置
- 缺乏架构约束

### 代码示例

```typescript
import express, { Request, Response, NextFunction } from 'express';

const app = express();

// 中间件
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// 日志中间件
app.use((req: Request, res: Response, next: NextFunction) => {
  console.log(`${req.method} ${req.url}`);
  next();
});

// 路由
app.get('/users', async (req: Request, res: Response) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/users', async (req: Request, res: Response) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ error: 'Bad request' });
  }
});

// 错误处理
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 适用场景

- 小型到中型项目
- 快速原型开发
- 学习 Node.js
- 需要大量第三方中间件
- 对性能要求不高

## Fastify

### 特点

**优点**：
- 极高性能（接近原生 HTTP）
- JSON Schema 验证
- 原生 TypeScript 支持
- 插件架构
- 低开销
- 异步优先

**缺点**：
- 社区生态不如 Express
- 学习曲线较陡
- 插件质量参差不齐
- 某些 Express 中间件不兼容

### 代码示例

```typescript
import Fastify, { FastifyInstance, FastifyRequest, FastifyReply } from 'fastify';

const fastify: FastifyInstance = Fastify({
  logger: true
});

// 插件
fastify.register(require('@fastify/cors'));
fastify.register(require('@fastify/helmet'));

// Schema 验证
const userSchema = {
  type: 'object',
  required: ['email', 'name'],
  properties: {
    email: { type: 'string', format: 'email' },
    name: { type: 'string', minLength: 2 },
    age: { type: 'number', minimum: 0 }
  }
};

// 路由
fastify.get('/users', async (request: FastifyRequest, reply: FastifyReply) => {
  const users = await User.find();
  return users;  // 自动 JSON 化
});

fastify.post('/users', {
  schema: {
    body: userSchema,
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'string' },
          email: { type: 'string' },
          name: { type: 'string' }
        }
      }
    }
  }
}, async (request: FastifyRequest, reply: FastifyReply) => {
  const user = await User.create(request.body);
  reply.code(201).send(user);
});

// 钩子（类似中间件）
fastify.addHook('preHandler', async (request, reply) => {
  console.log(`${request.method} ${request.url}`);
});

// 装饰器（扩展功能）
fastify.decorate('utility', {
  getCurrentUser: async (token: string) => {
    // 获取当前用户
  }
});

// 插件
async function userPlugin(fastify: FastifyInstance, options: any) {
  fastify.get('/profile', async (request, reply) => {
    // 使用装饰器
    const user = await fastify.utility.getCurrentUser(request.headers.authorization);
    return user;
  });
}

fastify.register(userPlugin, { prefix: '/api' });

// 错误处理
fastify.setErrorHandler((error, request, reply) => {
  request.log.error(error);
  
  if (error.validation) {
    reply.status(400).send({ error: 'Validation failed' });
  } else {
    reply.status(500).send({ error: 'Internal server error' });
  }
});

fastify.listen({ port: 3000 }, (err, address) => {
  if (err) {
    fastify.log.error(err);
    process.exit(1);
  }
  console.log(`Server listening on ${address}`);
});
```

### 性能对比

```typescript
// 压测结果（请求/秒）
// Express: ~15,000 req/s
// Fastify: ~30,000 req/s
// 原生 HTTP: ~40,000 req/s
```

### 适用场景

- 高性能 API
- 微服务架构
- 需要 Schema 验证
- 追求极致性能
- 实时应用

## Koa

### 特点

**优点**：
- 轻量级
- 优雅的中间件机制（洋葱模型）
- 原生 async/await
- 更好的错误处理
- Context 对象

**缺点**：
- 中间件生态不如 Express
- 缺少内置功能
- 需要手动组装
- 学习曲线中等

### 代码示例

```typescript
import Koa from 'koa';
import Router from '@koa/router';
import bodyParser from 'koa-bodyparser';

const app = new Koa();
const router = new Router();

// 中间件（洋葱模型）
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();  // 执行后续中间件
  const ms = Date.now() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}ms`);
});

// 错误处理
app.use(async (ctx, next) => {
  try {
    await next();
  } catch (err) {
    ctx.status = err.status || 500;
    ctx.body = {
      error: err.message
    };
    ctx.app.emit('error', err, ctx);
  }
});

app.use(bodyParser());

// 路由
router.get('/users', async (ctx) => {
  const users = await User.find();
  ctx.body = users;
});

router.post('/users', async (ctx) => {
  const user = await User.create(ctx.request.body);
  ctx.status = 201;
  ctx.body = user;
});

router.get('/users/:id', async (ctx) => {
  const user = await User.findById(ctx.params.id);
  
  if (!user) {
    ctx.throw(404, 'User not found');
  }
  
  ctx.body = user;
});

app.use(router.routes());
app.use(router.allowedMethods());

// 错误监听
app.on('error', (err, ctx) => {
  console.error('Server error:', err);
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 洋葱模型

```typescript
app.use(async (ctx, next) => {
  console.log('1. 开始');
  await next();
  console.log('1. 结束');
});

app.use(async (ctx, next) => {
  console.log('2. 开始');
  await next();
  console.log('2. 结束');
});

app.use(async (ctx) => {
  console.log('3. 处理');
  ctx.body = 'Hello';
});

// 输出：
// 1. 开始
// 2. 开始
// 3. 处理
// 2. 结束
// 1. 结束
```

### 适用场景

- 喜欢洋葱模型
- 需要精细控制中间件
- 中小型项目
- 追求代码优雅
- 自定义框架封装

## NestJS

### 特点

**优点**：
- 完整的企业级架构
- 强大的依赖注入
- TypeScript 原生支持
- 模块化设计
- 内置功能丰富（微服务、GraphQL、WebSocket）
- 最佳实践内置
- 测试友好

**缺点**：
- 学习曲线陡峭
- 样板代码多
- 性能不如 Fastify
- 过度设计（小项目）
- 生态依赖 Express/Fastify

### 代码示例

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();

// user.module.ts
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService]
})
export class UserModule {}

// user.controller.ts
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}
  
  @Get()
  async findAll(): Promise<User[]> {
    return this.userService.findAll();
  }
  
  @Post()
  @UsePipes(ValidationPipe)
  async create(@Body() createUserDto: CreateUserDto): Promise<User> {
    return this.userService.create(createUserDto);
  }
  
  @Get(':id')
  async findOne(@Param('id', ParseIntPipe) id: number): Promise<User> {
    const user = await this.userService.findOne(id);
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return user;
  }
}

// user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>
  ) {}
  
  async findAll(): Promise<User[]> {
    return this.userRepository.find();
  }
  
  async create(dto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(dto);
    return this.userRepository.save(user);
  }
  
  async findOne(id: number): Promise<User> {
    return this.userRepository.findOne({ where: { id } });
  }
}

// create-user.dto.ts
export class CreateUserDto {
  @IsEmail()
  email: string;
  
  @IsString()
  @MinLength(2)
  name: string;
  
  @IsInt()
  @Min(18)
  age: number;
}
```

### 适用场景

- 大型企业应用
- 需要强架构约束
- 团队协作开发
- 微服务架构
- 需要多种传输层（HTTP、WebSocket、微服务）
- 长期维护的项目

## 性能对比

### 基准测试

```bash
# 工具：autocannon
# 测试：GET /hello，返回 JSON

Express:      15,000 req/s
Koa:          18,000 req/s  (+20%)
Fastify:      30,000 req/s  (+100%)
NestJS:       14,000 req/s  (-7%)
原生 HTTP:    40,000 req/s  (+166%)
```

### 内存占用

```
Express:   ~50MB
Koa:       ~40MB
Fastify:   ~35MB
NestJS:    ~60MB
```

## 选择建议

### 按项目规模

**小型项目**：
- Express：快速开发，生态丰富
- Koa：代码优雅，灵活

**中型项目**：
- Express：稳定可靠
- Fastify：高性能需求
- NestJS：需要架构约束

**大型项目**：
- NestJS：企业级特性
- Fastify + 自建架构：性能 + 灵活

### 按团队经验

**初学者**：
- Express：最简单

**有经验**：
- Koa：更现代
- Fastify：高性能

**企业团队**：
- NestJS：最佳实践内置

### 按性能要求

**高性能**：
1. Fastify
2. Koa
3. Express
4. NestJS

**功能丰富**：
1. NestJS
2. Express
3. Fastify
4. Koa

## 迁移建议

### Express → Fastify

```typescript
// Express
app.get('/users', (req, res) => {
  res.json(users);
});

// Fastify
fastify.get('/users', async (request, reply) => {
  return users;  // 自动 JSON 化
});
```

### Express → NestJS

```typescript
// Express
app.get('/users', async (req, res) => {
  const users = await userService.findAll();
  res.json(users);
});

// NestJS
@Controller('users')
export class UserController {
  constructor(private userService: UserService) {}
  
  @Get()
  async findAll() {
    return this.userService.findAll();
  }
}
```

### Koa → Fastify

```typescript
// Koa
router.get('/users', async (ctx) => {
  ctx.body = await User.find();
});

// Fastify
fastify.get('/users', async (request, reply) => {
  return await User.find();
});
```

## 混合使用

### NestJS + Fastify

```typescript
// 使用 Fastify 作为底层引擎
import { NestFactory } from '@nestjs/core';
import { FastifyAdapter, NestFastifyApplication } from '@nestjs/platform-fastify';

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter()
  );
  
  await app.listen(3000);
}
```

### Express + TypeScript + 模块化

```typescript
// 手动实现类似 NestJS 的结构

// user.controller.ts
export class UserController {
  constructor(private userService: UserService) {}
  
  async findAll(req: Request, res: Response) {
    const users = await this.userService.findAll();
    res.json(users);
  }
}

// user.service.ts
export class UserService {
  constructor(private userRepository: UserRepository) {}
  
  async findAll() {
    return this.userRepository.find();
  }
}

// app.ts
const userRepository = new UserRepository();
const userService = new UserService(userRepository);
const userController = new UserController(userService);

app.get('/users', (req, res) => userController.findAll(req, res));
```

## 常见面试题

### 1. 为什么 Fastify 比 Express 快？

<details>
<summary>点击查看答案</summary>

**核心原因**：

1. **JSON Schema 验证**：
   - 预编译 Schema，减少运行时开销
   - Express 每次都需要手动验证

2. **路由优化**：
   - 使用 find-my-way（基数树）
   - Express 使用正则匹配（慢）

3. **低开销抽象**：
   - 更少的中间件层
   - 异步优先设计

4. **请求/响应对象**：
   - 更轻量的对象
   - 减少不必要的属性

5. **序列化优化**：
   - 使用 fast-json-stringify
   - Express 使用 JSON.stringify

**性能差距**：
- Fastify: ~30,000 req/s
- Express: ~15,000 req/s
- 差距：2倍
</details>

### 2. Koa 的洋葱模型有什么优势？

<details>
<summary>点击查看答案</summary>

**优势**：

1. **精细控制**：
   - 可以在请求前后执行代码
   - 便于实现计时、日志等功能

2. **错误处理**：
   - 统一的错误处理机制
   - 更容易捕获和处理错误

3. **异步流程**：
   - 原生支持 async/await
   - 代码更清晰

**示例**：
```typescript
// 计时中间件
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();  // 等待后续处理完成
  const ms = Date.now() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});
```

**对比 Express**：
- Express：线性执行
- Koa：可以在请求处理前后执行代码
</details>

### 3. 什么时候应该选择 NestJS？

<details>
<summary>点击查看答案</summary>

**应该选择 NestJS**：

1. **大型项目**：
   - 需要清晰的架构
   - 多人协作开发

2. **企业应用**：
   - 需要依赖注入
   - 模块化管理

3. **多种传输层**：
   - HTTP + WebSocket + 微服务
   - GraphQL

4. **长期维护**：
   - 需要可维护性
   - 团队成员更替

**不应该选择 NestJS**：

1. **小型项目**：
   - 过度设计
   - 增加复杂度

2. **性能关键**：
   - Fastify 更快
   - 开销相对较大

3. **快速原型**：
   - 学习曲线陡
   - 开发速度慢

4. **团队无 OOP 经验**：
   - 学习成本高
</details>

### 4. 如何在 Express 中实现依赖注入？

<details>
<summary>点击查看答案</summary>

```typescript
// 简单的 DI 容器
class Container {
  private services = new Map<string, any>();
  
  register(name: string, service: any) {
    this.services.set(name, service);
  }
  
  resolve<T>(name: string): T {
    return this.services.get(name);
  }
}

const container = new Container();

// 注册服务
container.register('userRepository', new UserRepository());
container.register('userService', new UserService(
  container.resolve('userRepository')
));

// 在路由中使用
app.get('/users', async (req, res) => {
  const userService = container.resolve('userService');
  const users = await userService.findAll();
  res.json(users);
});

// 或使用第三方库
// npm i inversify reflect-metadata
```
</details>

### 5. 框架性能优化通用技巧？

<details>
<summary>点击查看答案</summary>

**通用优化**：

1. **使用压缩**：
```typescript
// Express/NestJS
app.use(compression());

// Fastify
fastify.register(require('@fastify/compress'));
```

2. **启用缓存**：
```typescript
// HTTP 缓存头
res.set('Cache-Control', 'public, max-age=300');

// 应用缓存
const cached = await cache.get(key);
if (cached) return cached;
```

3. **连接池**：
```typescript
// 数据库连接池
{
  max: 20,
  min: 5,
  idle: 10000
}
```

4. **限制请求大小**：
```typescript
app.use(express.json({ limit: '10mb' }));
```

5. **使用 CDN**：
- 静态资源放 CDN
- 减少服务器负载

6. **集群模式**：
```typescript
import cluster from 'cluster';
import os from 'os';

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) {
    cluster.fork();
  }
} else {
  app.listen(3000);
}
```
</details>

## 总结

| 场景 | 推荐框架 | 理由 |
|------|----------|------|
| 快速原型 | Express | 简单、生态好 |
| 高性能API | Fastify | 极致性能 |
| 优雅代码 | Koa | 洋葱模型 |
| 企业应用 | NestJS | 完整架构 |
| 微服务 | NestJS | 多传输层支持 |
| 学习Node.js | Express | 最成熟 |
| 实时应用 | Fastify/NestJS | WebSocket支持好 |

