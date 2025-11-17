# NestJS 高级特性

## 微服务（Microservices）

NestJS 提供了多种传输层支持，用于构建微服务架构。

### TCP 微服务

```typescript
// main.ts（微服务端）
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: 'localhost',
        port: 3001
      }
    }
  );
  
  await app.listen();
}

// 消息处理器
@Controller()
export class MathController {
  @MessagePattern({ cmd: 'sum' })
  sum(data: number[]): number {
    return data.reduce((a, b) => a + b, 0);
  }
  
  @MessagePattern({ cmd: 'multiply' })
  multiply(data: number[]): number {
    return data.reduce((a, b) => a * b, 1);
  }
  
  @EventPattern('user_created')
  async handleUserCreated(data: Record<string, unknown>) {
    console.log('User created:', data);
    // 处理事件（不需要响应）
  }
}

// main.ts（API 网关）
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

// API 网关
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.TCP,
        options: {
          host: 'localhost',
          port: 3001
        }
      }
    ])
  ],
  controllers: [AppController]
})
export class AppModule {}

@Controller()
export class AppController {
  constructor(
    @Inject('MATH_SERVICE') private client: ClientProxy
  ) {}
  
  async onModuleInit() {
    await this.client.connect();
  }
  
  @Get('sum')
  async sum(@Query('numbers') numbers: string) {
    const data = numbers.split(',').map(Number);
    return this.client.send({ cmd: 'sum' }, data);
  }
  
  @Post('user')
  async createUser(@Body() user: any) {
    // 发送事件（不等待响应）
    this.client.emit('user_created', user);
    return { success: true };
  }
}
```

### Redis 微服务

```typescript
// 微服务端
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.REDIS,
    options: {
      host: 'localhost',
      port: 6379,
      retryAttempts: 5,
      retryDelay: 3000
    }
  }
);

// 客户端
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'REDIS_SERVICE',
        transport: Transport.REDIS,
        options: {
          host: 'localhost',
          port: 6379
        }
      }
    ])
  ]
})
export class AppModule {}
```

### RabbitMQ 微服务

```typescript
// 微服务端
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.RMQ,
    options: {
      urls: ['amqp://localhost:5672'],
      queue: 'orders_queue',
      queueOptions: {
        durable: true
      },
      // 预取数量
      prefetchCount: 10
    }
  }
);

// 处理器
@Controller()
export class OrderController {
  @MessagePattern('create_order')
  async createOrder(data: CreateOrderDto) {
    // 处理订单
    return { orderId: '123', status: 'created' };
  }
  
  @EventPattern('order_placed')
  async handleOrderPlaced(data: any) {
    // 异步处理订单事件
  }
}

// 客户端
@Injectable()
export class OrderService {
  constructor(
    @Inject('ORDER_SERVICE') private client: ClientProxy
  ) {}
  
  async createOrder(orderData: any) {
    // 同步调用，等待响应
    return this.client
      .send('create_order', orderData)
      .pipe(timeout(5000))
      .toPromise();
  }
  
  async notifyOrderPlaced(orderData: any) {
    // 异步调用，不等待响应
    this.client.emit('order_placed', orderData);
  }
}
```

### Kafka 微服务

```typescript
// 微服务端
const app = await NestFactory.createMicroservice<MicroserviceOptions>(
  AppModule,
  {
    transport: Transport.KAFKA,
    options: {
      client: {
        clientId: 'order-service',
        brokers: ['localhost:9092']
      },
      consumer: {
        groupId: 'order-consumer',
        allowAutoTopicCreation: true
      }
    }
  }
);

// 处理器
@Controller()
export class OrderController {
  @MessagePattern('order.created')
  async handleOrderCreated(
    @Payload() data: any,
    @Ctx() context: KafkaContext
  ) {
    const originalMessage = context.getMessage();
    const partition = context.getPartition();
    const topic = context.getTopic();
    
    console.log(`Received from topic ${topic}, partition ${partition}`);
    
    // 处理消息
    return { success: true };
  }
  
  @EventPattern('user.registered')
  async handleUserRegistered(@Payload() data: any) {
    // 异步处理
  }
}

// 客户端
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'KAFKA_SERVICE',
        transport: Transport.KAFKA,
        options: {
          client: {
            clientId: 'api-gateway',
            brokers: ['localhost:9092']
          },
          producer: {
            allowAutoTopicCreation: true
          }
        }
      }
    ])
  ]
})
export class AppModule {}

@Injectable()
export class EventService {
  constructor(
    @Inject('KAFKA_SERVICE') private client: ClientProxy
  ) {}
  
  async publishEvent(topic: string, data: any) {
    this.client.emit(topic, {
      key: data.id,
      value: data,
      headers: {
        'correlation-id': uuidv4()
      }
    });
  }
}
```

### 混合应用（HTTP + 微服务）

```typescript
async function bootstrap() {
  // 创建 HTTP 应用
  const app = await NestFactory.create(AppModule);
  
  // 连接微服务
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.TCP,
    options: { port: 3001 }
  });
  
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.REDIS,
    options: {
      host: 'localhost',
      port: 6379
    }
  });
  
  // 启动所有服务
  await app.startAllMicroservices();
  await app.listen(3000);
}

// 控制器同时支持 HTTP 和微服务
@Controller('users')
export class UserController {
  // HTTP 端点
  @Get()
  async findAll() {
    return this.userService.findAll();
  }
  
  // 微服务消息处理
  @MessagePattern({ cmd: 'get_user' })
  async getUser(id: number) {
    return this.userService.findOne(id);
  }
}
```

## WebSocket

### 网关（Gateway）

```typescript
@WebSocketGateway(3001, {
  cors: {
    origin: '*'
  },
  namespace: '/chat'
})
export class ChatGateway implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;
  
  private logger = new Logger('ChatGateway');
  
  afterInit(server: Server) {
    this.logger.log('WebSocket Gateway initialized');
  }
  
  handleConnection(client: Socket, ...args: any[]) {
    this.logger.log(`Client connected: ${client.id}`);
  }
  
  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
  }
  
  @SubscribeMessage('message')
  handleMessage(
    @MessageBody() data: string,
    @ConnectedSocket() client: Socket
  ): WsResponse<string> {
    return {
      event: 'message',
      data: `Echo: ${data}`
    };
  }
  
  @SubscribeMessage('join_room')
  handleJoinRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket
  ) {
    client.join(room);
    this.server.to(room).emit('user_joined', {
      userId: client.id,
      room
    });
  }
  
  @SubscribeMessage('leave_room')
  handleLeaveRoom(
    @MessageBody() room: string,
    @ConnectedSocket() client: Socket
  ) {
    client.leave(room);
    this.server.to(room).emit('user_left', {
      userId: client.id,
      room
    });
  }
  
  // 广播给所有客户端
  broadcastMessage(event: string, data: any) {
    this.server.emit(event, data);
  }
  
  // 发送给特定房间
  sendToRoom(room: string, event: string, data: any) {
    this.server.to(room).emit(event, data);
  }
  
  // 发送给特定客户端
  sendToClient(clientId: string, event: string, data: any) {
    this.server.to(clientId).emit(event, data);
  }
}

// 使用守卫
@WebSocketGateway()
@UseGuards(WsAuthGuard)
export class SecureChatGateway {
  @SubscribeMessage('private_message')
  handlePrivateMessage(
    @MessageBody() data: any,
    @ConnectedSocket() client: Socket
  ) {
    // client.data.user 由守卫注入
    const user = client.data.user;
    return { sender: user.id, message: data };
  }
}

// WebSocket 守卫
@Injectable()
export class WsAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}
  
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const client = context.switchToWs().getClient<Socket>();
    const token = client.handshake.auth.token;
    
    try {
      const payload = await this.jwtService.verifyAsync(token);
      client.data.user = payload;
      return true;
    } catch {
      return false;
    }
  }
}
```

### 适配器

```typescript
// Redis 适配器（多实例支持）
import { IoAdapter } from '@nestjs/platform-socket.io';
import { ServerOptions } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;
  
  async connectToRedis(): Promise<void> {
    const pubClient = createClient({ url: 'redis://localhost:6379' });
    const subClient = pubClient.duplicate();
    
    await Promise.all([pubClient.connect(), subClient.connect()]);
    
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }
  
  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}

// 使用
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  const redisIoAdapter = new RedisIoAdapter(app);
  await redisIoAdapter.connectToRedis();
  
  app.useWebSocketAdapter(redisIoAdapter);
  
  await app.listen(3000);
}
```

## GraphQL

### Code First

```typescript
// 安装依赖
// npm i @nestjs/graphql @nestjs/apollo @apollo/server graphql

// app.module.ts
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.gql',
      sortSchema: true,
      playground: true,
      context: ({ req }) => ({ req })
    })
  ]
})
export class AppModule {}

// 实体
@ObjectType()
export class User {
  @Field(() => ID)
  id: string;
  
  @Field()
  email: string;
  
  @Field()
  name: string;
  
  @Field(() => [Post], { nullable: 'itemsAndList' })
  posts?: Post[];
}

@ObjectType()
export class Post {
  @Field(() => ID)
  id: string;
  
  @Field()
  title: string;
  
  @Field()
  content: string;
  
  @Field(() => User)
  author: User;
}

// Input 类型
@InputType()
export class CreateUserInput {
  @Field()
  @IsEmail()
  email: string;
  
  @Field()
  @MinLength(2)
  name: string;
  
  @Field()
  @MinLength(6)
  password: string;
}

// Resolver
@Resolver(() => User)
export class UserResolver {
  constructor(
    private userService: UserService,
    private postService: PostService
  ) {}
  
  @Query(() => [User])
  async users(): Promise<User[]> {
    return this.userService.findAll();
  }
  
  @Query(() => User, { nullable: true })
  async user(@Args('id', { type: () => ID }) id: string): Promise<User> {
    return this.userService.findOne(id);
  }
  
  @Mutation(() => User)
  async createUser(
    @Args('input') input: CreateUserInput
  ): Promise<User> {
    return this.userService.create(input);
  }
  
  @Mutation(() => Boolean)
  async deleteUser(
    @Args('id', { type: () => ID }) id: string
  ): Promise<boolean> {
    await this.userService.remove(id);
    return true;
  }
  
  // Field Resolver（解析关联字段）
  @ResolveField(() => [Post])
  async posts(@Parent() user: User): Promise<Post[]> {
    return this.postService.findByUserId(user.id);
  }
}

// 分页
@ObjectType()
export class PaginatedPosts {
  @Field(() => [Post])
  items: Post[];
  
  @Field()
  total: number;
  
  @Field()
  hasMore: boolean;
}

@Resolver(() => Post)
export class PostResolver {
  @Query(() => PaginatedPosts)
  async posts(
    @Args('page', { type: () => Int, defaultValue: 1 }) page: number,
    @Args('limit', { type: () => Int, defaultValue: 10 }) limit: number
  ): Promise<PaginatedPosts> {
    const [items, total] = await this.postService.findAndCount({
      skip: (page - 1) * limit,
      take: limit
    });
    
    return {
      items,
      total,
      hasMore: page * limit < total
    };
  }
}
```

### DataLoader（解决 N+1 问题）

```typescript
@Injectable()
export class UserLoader {
  constructor(private userService: UserService) {}
  
  createLoader(): DataLoader<string, User> {
    return new DataLoader<string, User>(async (ids: string[]) => {
      const users = await this.userService.findByIds(ids);
      
      // 保证返回顺序与 ids 一致
      const userMap = new Map(users.map(user => [user.id, user]));
      return ids.map(id => userMap.get(id));
    });
  }
}

// 提供 DataLoader
@Module({
  providers: [
    UserLoader,
    {
      provide: 'USER_LOADER',
      useFactory: (userLoader: UserLoader) => userLoader.createLoader(),
      inject: [UserLoader],
      scope: Scope.REQUEST
    }
  ]
})
export class UserModule {}

// 使用 DataLoader
@Resolver(() => Post)
export class PostResolver {
  constructor(
    @Inject('USER_LOADER') private userLoader: DataLoader<string, User>
  ) {}
  
  @ResolveField(() => User)
  async author(@Parent() post: Post): Promise<User> {
    return this.userLoader.load(post.authorId);
  }
}
```

### 订阅（Subscriptions）

```typescript
@Resolver()
export class PostResolver {
  constructor(private pubSub: PubSub) {}
  
  @Mutation(() => Post)
  async createPost(@Args('input') input: CreatePostInput): Promise<Post> {
    const post = await this.postService.create(input);
    
    // 发布事件
    await this.pubSub.publish('postCreated', { postCreated: post });
    
    return post;
  }
  
  @Subscription(() => Post, {
    filter: (payload, variables) => {
      // 过滤订阅
      return payload.postCreated.authorId === variables.authorId;
    }
  })
  postCreated(@Args('authorId') authorId: string) {
    return this.pubSub.asyncIterator('postCreated');
  }
}

// 配置
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: true,
      subscriptions: {
        'graphql-ws': true,
        'subscriptions-transport-ws': true
      }
    })
  ],
  providers: [
    {
      provide: PubSub,
      useValue: new PubSub()
    }
  ]
})
export class AppModule {}
```

## 任务调度（Task Scheduling）

```typescript
// npm i @nestjs/schedule

@Module({
  imports: [ScheduleModule.forRoot()]
})
export class AppModule {}

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);
  
  // Cron 任务
  @Cron('0 0 * * *')  // 每天午夜执行
  handleCron() {
    this.logger.debug('Running daily cleanup');
    // 清理任务
  }
  
  @Cron('*/5 * * * *')  // 每 5 分钟
  handleEveryFiveMinutes() {
    this.logger.debug('Running every 5 minutes');
  }
  
  @Cron(CronExpression.EVERY_30_SECONDS)
  handleEvery30Seconds() {
    this.logger.debug('Running every 30 seconds');
  }
  
  // 间隔任务
  @Interval(10000)  // 每 10 秒
  handleInterval() {
    this.logger.debug('Running every 10 seconds');
  }
  
  // 超时任务（只执行一次）
  @Timeout(5000)  // 5 秒后执行
  handleTimeout() {
    this.logger.debug('Running after 5 seconds');
  }
  
  // 动态调度
  constructor(private schedulerRegistry: SchedulerRegistry) {}
  
  addCronJob(name: string, cronTime: string) {
    const job = new CronJob(cronTime, () => {
      this.logger.warn(`Running job ${name}`);
    });
    
    this.schedulerRegistry.addCronJob(name, job);
    job.start();
    
    this.logger.warn(`Job ${name} added`);
  }
  
  deleteCronJob(name: string) {
    this.schedulerRegistry.deleteCronJob(name);
    this.logger.warn(`Job ${name} deleted`);
  }
  
  addInterval(name: string, milliseconds: number) {
    const callback = () => {
      this.logger.warn(`Interval ${name} executing`);
    };
    
    const interval = setInterval(callback, milliseconds);
    this.schedulerRegistry.addInterval(name, interval);
  }
  
  deleteInterval(name: string) {
    this.schedulerRegistry.deleteInterval(name);
  }
}
```

## 队列（Bull）

```typescript
// npm i @nestjs/bull bull

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: 'localhost',
        port: 6379
      }
    }),
    BullModule.registerQueue({
      name: 'email',
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 1000
        }
      }
    })
  ]
})
export class AppModule {}

// Producer
@Injectable()
export class EmailService {
  constructor(
    @InjectQueue('email') private emailQueue: Queue
  ) {}
  
  async sendWelcomeEmail(user: User) {
    await this.emailQueue.add('welcome', {
      userId: user.id,
      email: user.email
    }, {
      priority: 1,
      delay: 5000  // 延迟 5 秒
    });
  }
  
  async sendBulkEmails(users: User[]) {
    const jobs = users.map(user => ({
      name: 'bulk',
      data: { userId: user.id, email: user.email }
    }));
    
    await this.emailQueue.addBulk(jobs);
  }
}

// Consumer
@Processor('email')
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);
  
  @Process('welcome')
  async handleWelcome(job: Job<{ userId: string; email: string }>) {
    this.logger.log(`Sending welcome email to ${job.data.email}`);
    
    // 更新进度
    await job.progress(50);
    
    // 发送邮件
    await this.sendEmail(job.data);
    
    await job.progress(100);
    
    return { sent: true };
  }
  
  @Process({ name: 'bulk', concurrency: 5 })
  async handleBulk(job: Job) {
    // 批量处理，最多 5 个并发
    await this.sendEmail(job.data);
  }
  
  @OnQueueActive()
  onActive(job: Job) {
    this.logger.log(`Processing job ${job.id} of type ${job.name}`);
  }
  
  @OnQueueCompleted()
  onCompleted(job: Job, result: any) {
    this.logger.log(`Job ${job.id} completed with result:`, result);
  }
  
  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed with error:`, error);
  }
  
  private async sendEmail(data: any) {
    // 实际发送邮件逻辑
  }
}

// 监控队列
@Controller('queues')
export class QueueController {
  constructor(
    @InjectQueue('email') private emailQueue: Queue
  ) {}
  
  @Get('stats')
  async getStats() {
    const jobCounts = await this.emailQueue.getJobCounts();
    const completed = await this.emailQueue.getCompletedCount();
    const failed = await this.emailQueue.getFailedCount();
    
    return {
      waiting: jobCounts.waiting,
      active: jobCounts.active,
      completed,
      failed
    };
  }
  
  @Post('pause')
  async pause() {
    await this.emailQueue.pause();
    return { paused: true };
  }
  
  @Post('resume')
  async resume() {
    await this.emailQueue.resume();
    return { resumed: true };
  }
  
  @Delete('clean')
  async clean() {
    await this.emailQueue.clean(0, 'completed');
    await this.emailQueue.clean(0, 'failed');
    return { cleaned: true };
  }
}
```

## 缓存

```typescript
// npm i cache-manager

@Module({
  imports: [
    CacheModule.register({
      ttl: 60,  // 秒
      max: 100  // 最大项数
    })
  ]
})
export class AppModule {}

// 使用缓存
@Injectable()
export class UserService {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache
  ) {}
  
  async findAll(): Promise<User[]> {
    const cacheKey = 'users:all';
    
    // 尝试从缓存获取
    const cached = await this.cacheManager.get<User[]>(cacheKey);
    if (cached) {
      return cached;
    }
    
    // 从数据库查询
    const users = await this.userRepository.find();
    
    // 写入缓存
    await this.cacheManager.set(cacheKey, users, { ttl: 300 });
    
    return users;
  }
  
  async invalidateCache(key: string) {
    await this.cacheManager.del(key);
  }
  
  async clearCache() {
    await this.cacheManager.reset();
  }
}

// 自动缓存拦截器
@Controller('users')
@UseInterceptors(CacheInterceptor)
export class UserController {
  @Get()
  @CacheKey('users:all')
  @CacheTTL(300)
  async findAll() {
    return this.userService.findAll();
  }
}

// Redis 缓存
import * as redisStore from 'cache-manager-redis-store';

@Module({
  imports: [
    CacheModule.register({
      store: redisStore,
      host: 'localhost',
      port: 6379,
      ttl: 60
    })
  ]
})
export class AppModule {}
```

## 配置管理

```typescript
// npm i @nestjs/config

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ['.env.local', '.env'],
      ignoreEnvFile: process.env.NODE_ENV === 'production',
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test')
          .default('development'),
        PORT: Joi.number().default(3000),
        DATABASE_URL: Joi.string().required()
      })
    })
  ]
})
export class AppModule {}

// 使用配置
@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}
  
  getDbConfig() {
    const url = this.configService.get<string>('DATABASE_URL');
    const port = this.configService.get<number>('PORT', 3000);
    
    return { url, port };
  }
}

// 自定义配置
export default () => ({
  database: {
    host: process.env.DB_HOST,
    port: parseInt(process.env.DB_PORT, 10) || 5432
  },
  redis: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT, 10) || 6379
  }
});

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration]
    })
  ]
})
export class AppModule {}

// 使用
const dbHost = this.configService.get<string>('database.host');
```

## 常见面试题

### 1. NestJS 微服务架构的优缺点？

<details>
<summary>点击查看答案</summary>

**优点**：
- 服务解耦，独立部署
- 技术栈灵活
- 易于扩展
- 故障隔离
- 支持多种传输层（TCP、Redis、RabbitMQ、Kafka）

**缺点**：
- 复杂度增加
- 分布式事务难处理
- 服务间通信开销
- 调试困难
- 运维成本高

**适用场景**：
- 大型分布式系统
- 需要独立扩展的服务
- 团队独立开发
- 高可用要求
</details>

### 2. 如何处理微服务间的分布式事务？

<details>
<summary>点击查看答案</summary>

**方案**：

1. **Saga 模式**：
   - 本地事务 + 补偿机制
   - 适合长事务

2. **2PC/3PC**：
   - 两阶段/三阶段提交
   - 强一致性，性能差

3. **事件溯源**：
   - 记录事件流
   - 最终一致性

4. **TCC（Try-Confirm-Cancel）**：
   - 预留-确认-取消
   - 业务侵入性强

**推荐**：
- 大多数场景使用 Saga
- 金融场景考虑 TCC
- 能接受最终一致性优先使用事件驱动
</details>

### 3. WebSocket 与 HTTP 的区别？何时使用？

<details>
<summary>点击查看答案</summary>

**HTTP**：
- 请求-响应模式
- 短连接（或长轮询）
- 单向通信

**WebSocket**：
- 全双工通信
- 长连接
- 实时双向数据传输
- 低延迟

**使用场景**：

WebSocket：
- 实时聊天
- 实时通知
- 协同编辑
- 在线游戏
- 实时数据看板

HTTP：
- RESTful API
- 传统 Web 应用
- 文件下载
- 简单的数据获取
</details>

### 4. GraphQL vs REST 的优缺点？

<details>
<summary>点击查看答案</summary>

**GraphQL 优点**：
- 按需查询，避免过度获取
- 单个端点
- 强类型系统
- 实时订阅
- 自动文档

**GraphQL 缺点**：
- 学习曲线陡峭
- 缓存复杂
- 查询复杂度控制
- N+1 问题

**REST 优点**：
- 简单易用
- 缓存友好
- 广泛支持
- 成熟的工具链

**REST 缺点**：
- 过度获取/不足获取
- 多个端点
- 版本管理复杂

**选择建议**：
- 移动应用、复杂查询 → GraphQL
- 简单 CRUD、公共 API → REST
</details>

### 5. Bull 队列的应用场景和最佳实践？

<details>
<summary>点击查看答案</summary>

**应用场景**：
- 邮件发送
- 图片处理
- 视频转码
- 数据导出
- 定时任务
- 批量操作

**最佳实践**：

1. **设置重试策略**：
```typescript
{
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 1000
  }
}
```

2. **控制并发**：
```typescript
@Process({ name: 'heavy', concurrency: 2 })
```

3. **监控队列**：
```typescript
@OnQueueFailed()
async onFailed(job: Job, error: Error) {
  // 记录失败，发送告警
}
```

4. **使用优先级**：
```typescript
await queue.add(data, { priority: 1 });
```

5. **定期清理**：
```typescript
await queue.clean(86400000, 'completed');
```
</details>

