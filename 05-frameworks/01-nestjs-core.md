# NestJS 核心架构

NestJS 是一个用于构建高效、可扩展的 Node.js 服务端应用的框架，基于 TypeScript，借鉴了 Angular 的架构设计。

## 核心概念

### 模块（Modules）

模块是组织应用结构的基本单元，每个应用至少有一个根模块。

```typescript
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';

@Module({
  imports: [
    // 导入其他模块
    TypeOrmModule.forFeature([User])
  ],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService]  // 导出供其他模块使用
})
export class UserModule {}
```

**模块类型**：

```typescript
// 1. 功能模块
@Module({
  imports: [DatabaseModule],
  controllers: [OrderController],
  providers: [OrderService],
  exports: [OrderService]
})
export class OrderModule {}

// 2. 共享模块
@Module({
  providers: [ConfigService],
  exports: [ConfigService]
})
export class SharedModule {}

// 3. 全局模块（@Global）
@Global()
@Module({
  providers: [LoggerService],
  exports: [LoggerService]
})
export class LoggerModule {}

// 4. 动态模块
@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options
        },
        DatabaseService
      ],
      exports: [DatabaseService]
    };
  }
  
  static forFeature(entities: Entity[]): DynamicModule {
    const providers = entities.map(entity => ({
      provide: `${entity.name}_REPOSITORY`,
      useFactory: (connection: Connection) => 
        connection.getRepository(entity),
      inject: [Connection]
    }));
    
    return {
      module: DatabaseModule,
      providers,
      exports: providers
    };
  }
}

// 使用
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432
    })
  ]
})
export class AppModule {}
```

### 依赖注入（Dependency Injection）

NestJS 使用强大的 IoC 容器管理依赖。

```typescript
// 1. 类提供者（默认）
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
    private emailService: EmailService
  ) {}
  
  async createUser(dto: CreateUserDto) {
    const user = this.userRepository.create(dto);
    await this.userRepository.save(user);
    await this.emailService.sendWelcome(user.email);
    return user;
  }
}

// 2. 值提供者
@Module({
  providers: [
    {
      provide: 'CONFIG',
      useValue: {
        apiUrl: process.env.API_URL,
        timeout: 5000
      }
    }
  ]
})
export class AppModule {}

// 使用
@Injectable()
export class ApiService {
  constructor(@Inject('CONFIG') private config: any) {}
}

// 3. 工厂提供者
@Module({
  providers: [
    {
      provide: 'DATABASE_CONNECTION',
      useFactory: async (configService: ConfigService) => {
        const config = configService.get('database');
        const connection = await createConnection(config);
        return connection;
      },
      inject: [ConfigService]
    }
  ]
})
export class DatabaseModule {}

// 4. 类提供者（别名）
@Module({
  providers: [
    {
      provide: 'IUserService',
      useClass: process.env.NODE_ENV === 'test' 
        ? MockUserService 
        : UserService
    }
  ]
})
export class AppModule {}

// 5. 现有提供者（别名）
@Module({
  providers: [
    LoggerService,
    {
      provide: 'AliasedLogger',
      useExisting: LoggerService
    }
  ]
})
export class LoggerModule {}
```

**作用域（Scope）**：

```typescript
// 1. DEFAULT（单例，默认）
@Injectable()
export class UserService {}

// 2. REQUEST（每个请求一个实例）
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {
  constructor(@Inject(REQUEST) private request: Request) {}
}

// 3. TRANSIENT（每次注入创建新实例）
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {}

// 注意：REQUEST 作用域会影响性能
// 使用场景：需要访问请求上下文、多租户系统
```

### 控制器（Controllers）

```typescript
@Controller('users')
@UseGuards(AuthGuard)
@UseInterceptors(LoggingInterceptor)
export class UserController {
  constructor(
    private readonly userService: UserService,
    private readonly logger: LoggerService
  ) {}
  
  @Get()
  @UseGuards(RolesGuard)
  @Roles('admin')
  @ApiOperation({ summary: 'Get all users' })
  @ApiResponse({ status: 200, type: [User] })
  async findAll(
    @Query() query: FindUsersDto,
    @Req() req: Request
  ): Promise<User[]> {
    return this.userService.findAll(query);
  }
  
  @Get(':id')
  @ApiParam({ name: 'id', type: 'string' })
  async findOne(
    @Param('id', ParseIntPipe) id: number
  ): Promise<User> {
    const user = await this.userService.findOne(id);
    if (!user) {
      throw new NotFoundException(`User #${id} not found`);
    }
    return user;
  }
  
  @Post()
  @HttpCode(201)
  @UsePipes(ValidationPipe)
  async create(
    @Body() createUserDto: CreateUserDto
  ): Promise<User> {
    return this.userService.create(createUserDto);
  }
  
  @Put(':id')
  async update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto
  ): Promise<User> {
    return this.userService.update(id, updateUserDto);
  }
  
  @Delete(':id')
  @HttpCode(204)
  async remove(
    @Param('id', ParseIntPipe) id: number
  ): Promise<void> {
    await this.userService.remove(id);
  }
  
  // 子路由
  @Get(':id/orders')
  async getUserOrders(
    @Param('id', ParseIntPipe) id: number,
    @Query('page', new DefaultValuePipe(1), ParseIntPipe) page: number
  ) {
    return this.userService.getOrders(id, page);
  }
}

// 路径版本控制
@Controller({
  path: 'users',
  version: '1'
})
export class UserV1Controller {}

@Controller({
  path: 'users',
  version: '2'
})
export class UserV2Controller {}

// 启用版本控制
const app = await NestFactory.create(AppModule);
app.enableVersioning({
  type: VersioningType.URI
});
// 访问：/v1/users, /v2/users
```

### 提供者（Providers）

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
    private emailService: EmailService,
    private cacheService: CacheService,
    @Inject('LOGGER') private logger: Logger
  ) {}
  
  async findAll(query: FindUsersDto): Promise<User[]> {
    const cacheKey = `users:${JSON.stringify(query)}`;
    
    // 先查缓存
    const cached = await this.cacheService.get(cacheKey);
    if (cached) {
      return cached;
    }
    
    // 查询数据库
    const users = await this.userRepository.find({
      where: query,
      relations: ['profile', 'orders']
    });
    
    // 写入缓存
    await this.cacheService.set(cacheKey, users, 300);
    
    return users;
  }
  
  async create(dto: CreateUserDto): Promise<User> {
    const user = this.userRepository.create(dto);
    await this.userRepository.save(user);
    
    // 发送欢迎邮件（异步）
    this.emailService.sendWelcome(user.email)
      .catch(err => this.logger.error('Failed to send email', err));
    
    return user;
  }
  
  async update(id: number, dto: UpdateUserDto): Promise<User> {
    const user = await this.findOne(id);
    Object.assign(user, dto);
    return this.userRepository.save(user);
  }
  
  async findOne(id: number): Promise<User> {
    return this.userRepository.findOne({
      where: { id },
      relations: ['profile']
    });
  }
}

// 自定义提供者示例
@Injectable()
export class CacheService {
  private cache = new Map<string, { value: any; expiry: number }>();
  
  async get<T>(key: string): Promise<T | null> {
    const item = this.cache.get(key);
    
    if (!item) {
      return null;
    }
    
    if (Date.now() > item.expiry) {
      this.cache.delete(key);
      return null;
    }
    
    return item.value;
  }
  
  async set(key: string, value: any, ttl: number): Promise<void> {
    this.cache.set(key, {
      value,
      expiry: Date.now() + ttl * 1000
    });
  }
  
  async del(key: string): Promise<void> {
    this.cache.delete(key);
  }
  
  async clear(): Promise<void> {
    this.cache.clear();
  }
}
```

## 中间件（Middleware）

```typescript
// 1. 函数式中间件
export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`${req.method} ${req.url}`);
  next();
}

// 2. 类中间件
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  constructor(private logger: LoggerService) {}
  
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();
    
    res.on('finish', () => {
      const duration = Date.now() - start;
      this.logger.log({
        method: req.method,
        url: req.url,
        statusCode: res.statusCode,
        duration
      });
    });
    
    next();
  }
}

// 3. 应用中间件
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware, AuthMiddleware)
      .exclude(
        { path: 'health', method: RequestMethod.GET },
        'metrics'
      )
      .forRoutes(
        { path: 'users', method: RequestMethod.ALL },
        OrderController
      );
  }
}

// 4. 全局中间件
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // 函数式中间件
  app.use(helmet());
  app.use(compression());
  app.use(logger);
  
  await app.listen(3000);
}
```

## 异常过滤器（Exception Filters）

```typescript
// 1. 内置异常
@Get(':id')
async findOne(@Param('id') id: string) {
  const user = await this.userService.findOne(id);
  
  if (!user) {
    throw new NotFoundException('User not found');
  }
  
  return user;
}

// 常用异常：
// BadRequestException (400)
// UnauthorizedException (401)
// ForbiddenException (403)
// NotFoundException (404)
// ConflictException (409)
// InternalServerErrorException (500)

// 2. 自定义异常
export class BusinessException extends HttpException {
  constructor(message: string, code: string) {
    super(
      {
        statusCode: HttpStatus.BAD_REQUEST,
        message,
        code,
        timestamp: new Date().toISOString()
      },
      HttpStatus.BAD_REQUEST
    );
  }
}

// 使用
throw new BusinessException('Insufficient balance', 'INSUFFICIENT_BALANCE');

// 3. 异常过滤器
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(private logger: LoggerService) {}
  
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();
    
    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message: 
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message
    };
    
    // 记录错误
    this.logger.error(
      `${request.method} ${request.url}`,
      JSON.stringify(errorResponse)
    );
    
    response.status(status).json(errorResponse);
  }
}

// 4. 捕获所有异常
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    
    const status =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;
    
    const message =
      exception instanceof HttpException
        ? exception.message
        : 'Internal server error';
    
    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message
    });
  }
}

// 应用过滤器
// 方法级别
@Get()
@UseFilters(HttpExceptionFilter)
async findAll() {}

// 控制器级别
@Controller()
@UseFilters(HttpExceptionFilter)
export class UserController {}

// 全局级别
app.useGlobalFilters(new AllExceptionsFilter());
```

## 管道（Pipes）

```typescript
// 1. 内置管道
@Get(':id')
async findOne(
  @Param('id', ParseIntPipe) id: number
) {
  return this.userService.findOne(id);
}

// 内置管道：
// - ValidationPipe: 验证和转换 DTO
// - ParseIntPipe: 转换为整数
// - ParseFloatPipe: 转换为浮点数
// - ParseBoolPipe: 转换为布尔值
// - ParseArrayPipe: 转换为数组
// - ParseUUIDPipe: 验证 UUID
// - ParseEnumPipe: 验证枚举
// - DefaultValuePipe: 设置默认值

// 2. ValidationPipe 配置
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,  // 移除 DTO 中未定义的属性
  forbidNonWhitelisted: true,  // 存在未定义属性时抛出错误
  transform: true,  // 自动转换类型
  transformOptions: {
    enableImplicitConversion: true
  },
  disableErrorMessages: process.env.NODE_ENV === 'production'
}));

// 3. DTO 验证
import { IsString, IsEmail, IsInt, Min, Max, IsOptional } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @IsNotEmpty()
  name: string;
  
  @IsEmail()
  email: string;
  
  @IsInt()
  @Min(18)
  @Max(100)
  age: number;
  
  @IsOptional()
  @IsString()
  bio?: string;
}

// 4. 自定义管道
@Injectable()
export class ParseDatePipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    const date = new Date(value);
    
    if (isNaN(date.getTime())) {
      throw new BadRequestException('Invalid date format');
    }
    
    return date;
  }
}

// 使用
@Get()
async findByDate(
  @Query('date', ParseDatePipe) date: Date
) {
  return this.service.findByDate(date);
}

// 5. 高级验证管道
@Injectable()
export class UniqueEmailPipe implements PipeTransform {
  constructor(private userService: UserService) {}
  
  async transform(value: any, metadata: ArgumentMetadata) {
    if (metadata.type === 'body' && value.email) {
      const existing = await this.userService.findByEmail(value.email);
      
      if (existing) {
        throw new ConflictException('Email already exists');
      }
    }
    
    return value;
  }
}

// 6. 自定义装饰器 + 管道
export const ParseJsonPipe = () => 
  new ParseArrayPipe({ 
    items: Object,
    separator: ',',
    optional: true 
  });

@Get()
async findAll(
  @Query('filters', ParseJsonPipe()) filters: any[]
) {
  return this.service.findAll(filters);
}
```

## 守卫（Guards）

```typescript
// 1. 认证守卫
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private userService: UserService
  ) {}
  
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);
    
    if (!token) {
      throw new UnauthorizedException('Missing token');
    }
    
    try {
      const payload = await this.jwtService.verifyAsync(token);
      const user = await this.userService.findOne(payload.sub);
      
      if (!user) {
        throw new UnauthorizedException('User not found');
      }
      
      // 将用户信息附加到请求
      request.user = user;
      return true;
      
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
  
  private extractToken(request: Request): string | null {
    const authHeader = request.headers.authorization;
    
    if (!authHeader) {
      return null;
    }
    
    const [type, token] = authHeader.split(' ');
    return type === 'Bearer' ? token : null;
  }
}

// 2. 角色守卫
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      'roles',
      [context.getHandler(), context.getClass()]
    );
    
    if (!requiredRoles) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    if (!user) {
      return false;
    }
    
    return requiredRoles.some(role => user.roles?.includes(role));
  }
}

// 自定义装饰器
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);

// 使用
@Controller('admin')
@UseGuards(AuthGuard, RolesGuard)
export class AdminController {
  @Get()
  @Roles('admin', 'superadmin')
  async findAll() {
    return 'Admin panel';
  }
}

// 3. 权限守卫
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private permissionService: PermissionService
  ) {}
  
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions',
      context.getHandler()
    );
    
    if (!requiredPermissions) {
      return true;
    }
    
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    
    const hasPermission = await this.permissionService.hasPermissions(
      user.id,
      requiredPermissions
    );
    
    return hasPermission;
  }
}

// 4. 限流守卫
@Injectable()
export class ThrottlerGuard implements CanActivate {
  private requests = new Map<string, number[]>();
  
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const key = this.getKey(request);
    
    const now = Date.now();
    const windowMs = 60000;  // 1 分钟
    const maxRequests = 100;
    
    // 获取请求记录
    const timestamps = this.requests.get(key) || [];
    
    // 过滤掉过期的请求
    const validTimestamps = timestamps.filter(
      ts => now - ts < windowMs
    );
    
    if (validTimestamps.length >= maxRequests) {
      throw new HttpException(
        'Too many requests',
        HttpStatus.TOO_MANY_REQUESTS
      );
    }
    
    // 记录本次请求
    validTimestamps.push(now);
    this.requests.set(key, validTimestamps);
    
    return true;
  }
  
  private getKey(request: Request): string {
    return request.ip || 'unknown';
  }
}
```

## 拦截器（Interceptors）

```typescript
// 1. 日志拦截器
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: LoggerService) {}
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url } = request;
    const start = Date.now();
    
    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        this.logger.log(`${method} ${url} - ${duration}ms`);
      }),
      catchError(err => {
        const duration = Date.now() - start;
        this.logger.error(`${method} ${url} - ${duration}ms - Error: ${err.message}`);
        return throwError(() => err);
      })
    );
  }
}

// 2. 转换拦截器
@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString()
      }))
    );
  }
}

interface Response<T> {
  success: boolean;
  data: T;
  timestamp: string;
}

// 3. 缓存拦截器
@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(private cacheService: CacheService) {}
  
  async intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const key = `cache:${request.method}:${request.url}`;
    
    // 只缓存 GET 请求
    if (request.method !== 'GET') {
      return next.handle();
    }
    
    // 检查缓存
    const cached = await this.cacheService.get(key);
    if (cached) {
      return of(cached);
    }
    
    // 执行请求并缓存结果
    return next.handle().pipe(
      tap(async response => {
        await this.cacheService.set(key, response, 300);
      })
    );
  }
}

// 4. 超时拦截器
@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeout: number = 5000) {}
  
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeout),
      catchError(err => {
        if (err instanceof TimeoutError) {
          throw new RequestTimeoutException('Request timeout');
        }
        return throwError(() => err);
      })
    );
  }
}

// 5. 错误拦截器
@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError(err => {
        // 转换内部错误为 HTTP 异常
        if (err instanceof BusinessException) {
          throw new BadRequestException(err.message);
        }
        
        if (err instanceof QueryFailedError) {
          throw new InternalServerErrorException('Database error');
        }
        
        return throwError(() => err);
      })
    );
  }
}

// 使用拦截器
@Controller()
@UseInterceptors(LoggingInterceptor, TransformInterceptor)
export class UserController {}

// 全局拦截器
app.useGlobalInterceptors(
  new LoggingInterceptor(logger),
  new TransformInterceptor()
);
```

## 装饰器组合

```typescript
// 自定义装饰器
export const Auth = (...roles: string[]) => 
  applyDecorators(
    UseGuards(AuthGuard, RolesGuard),
    Roles(...roles),
    ApiBearerAuth(),
    ApiUnauthorizedResponse({ description: 'Unauthorized' })
  );

export const ApiPaginatedResponse = <T>(model: Type<T>) =>
  applyDecorators(
    ApiOkResponse({
      schema: {
        allOf: [
          {
            properties: {
              data: {
                type: 'array',
                items: { $ref: getSchemaPath(model) }
              },
              meta: {
                type: 'object',
                properties: {
                  total: { type: 'number' },
                  page: { type: 'number' },
                  limit: { type: 'number' }
                }
              }
            }
          }
        ]
      }
    })
  );

// 使用
@Controller('users')
@ApiTags('users')
export class UserController {
  @Get()
  @Auth('admin')
  @ApiPaginatedResponse(User)
  async findAll() {
    return this.userService.findAll();
  }
}

// 参数装饰器
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.user;
  }
);

// 使用
@Get('profile')
async getProfile(@CurrentUser() user: User) {
  return user;
}
```

## 应用启动配置

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log', 'debug', 'verbose'],
    cors: true,
    bodyParser: true
  });
  
  // 全局前缀
  app.setGlobalPrefix('api/v1');
  
  // 版本控制
  app.enableVersioning({
    type: VersioningType.URI,
    defaultVersion: '1'
  });
  
  // 全局管道
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,
    forbidNonWhitelisted: true,
    transform: true
  }));
  
  // 全局过滤器
  app.useGlobalFilters(new AllExceptionsFilter());
  
  // 全局拦截器
  app.useGlobalInterceptors(new LoggingInterceptor());
  
  // 全局守卫
  app.useGlobalGuards(new ThrottlerGuard());
  
  // Swagger 文档
  const config = new DocumentBuilder()
    .setTitle('API Documentation')
    .setDescription('The API description')
    .setVersion('1.0')
    .addBearerAuth()
    .build();
  
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api/docs', app, document);
  
  // 优雅关闭
  app.enableShutdownHooks();
  
  await app.listen(3000);
}

bootstrap();
```

## 常见面试题

### 1. NestJS 的核心架构是什么？

<details>
<summary>点击查看答案</summary>

**核心概念**：
1. **模块（Modules）**：组织代码的基本单元
2. **控制器（Controllers）**：处理 HTTP 请求
3. **提供者（Providers）**：业务逻辑和服务
4. **依赖注入（DI）**：管理依赖关系

**请求生命周期**：
```
Middleware → Guards → Interceptors (before)
  → Pipes → Controller → Service
  → Interceptors (after) → Exception Filters
```

**优势**：
- 模块化架构
- 强大的依赖注入
- TypeScript 支持
- 装饰器元编程
- 可测试性强
</details>

### 2. 依赖注入的作用域有哪些？

<details>
<summary>点击查看答案</summary>

**三种作用域**：

1. **DEFAULT（单例）**：
   - 整个应用共享一个实例
   - 最高性能
   - 适用于无状态服务

2. **REQUEST**：
   - 每个请求创建新实例
   - 可访问请求上下文
   - 性能开销大
   - 适用于需要请求信息的服务

3. **TRANSIENT**：
   - 每次注入创建新实例
   - 不共享状态
   - 最大灵活性，最低性能

**注意事项**：
- REQUEST 作用域会传播到整个依赖链
- 性能敏感场景优先使用 DEFAULT
- 多租户系统可能需要 REQUEST 作用域
</details>

### 3. 中间件、守卫、拦截器、管道的区别和执行顺序？

<details>
<summary>点击查看答案</summary>

**执行顺序**：
```
Middleware → Guards → Interceptors (before) 
  → Pipes → Controller Handler 
  → Interceptors (after) → Exception Filters
```

**区别**：

| 组件 | 作用 | 访问内容 | 示例 |
|------|------|----------|------|
| Middleware | 请求预处理 | Request/Response | 日志、CORS |
| Guards | 权限控制 | ExecutionContext | 认证、授权 |
| Interceptors | AOP | 请求前后 | 日志、转换、缓存 |
| Pipes | 数据转换和验证 | 参数 | 验证、类型转换 |
| Filters | 异常处理 | Exception | 错误处理 |

**选择建议**：
- 通用逻辑 → Middleware
- 权限检查 → Guards
- 响应转换 → Interceptors
- 参数验证 → Pipes
- 错误处理 → Filters
</details>

### 4. 如何实现全局异常处理？

<details>
<summary>点击查看答案</summary>

```typescript
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();
    
    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    
    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();
      message = 
        typeof exceptionResponse === 'string'
          ? exceptionResponse
          : (exceptionResponse as any).message;
    }
    
    // 记录日志
    console.error(exception);
    
    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
      path: request.url
    });
  }
}

// 应用
app.useGlobalFilters(new GlobalExceptionFilter());
```
</details>

### 5. 动态模块是什么？如何实现？

<details>
<summary>点击查看答案</summary>

动态模块允许在运行时根据配置创建模块。

```typescript
@Module({})
export class ConfigModule {
  static forRoot(options: ConfigOptions): DynamicModule {
    return {
      module: ConfigModule,
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useValue: options
        },
        ConfigService
      ],
      exports: [ConfigService],
      global: options.isGlobal || false
    };
  }
  
  static forRootAsync(options: ConfigAsyncOptions): DynamicModule {
    return {
      module: ConfigModule,
      imports: options.imports || [],
      providers: [
        {
          provide: 'CONFIG_OPTIONS',
          useFactory: options.useFactory,
          inject: options.inject || []
        },
        ConfigService
      ],
      exports: [ConfigService]
    };
  }
}

// 使用
@Module({
  imports: [
    ConfigModule.forRoot({
      folder: './config',
      isGlobal: true
    })
  ]
})
export class AppModule {}
```

**使用场景**：
- 数据库连接配置
- 第三方服务集成
- 功能模块配置
</details>

