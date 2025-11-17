# NestJS 底层原理

深入理解 NestJS 的底层实现机制，包括依赖注入、装饰器、模块加载等核心原理。

## 装饰器（Decorators）原理

### 装饰器基础

装饰器是一种特殊的声明，可以附加到类、方法、访问器、属性或参数上。

```typescript
// 类装饰器
function Injectable(): ClassDecorator {
  return (target: Function) => {
    // target 是被装饰的类
    Reflect.defineMetadata('injectable', true, target);
  };
}

// 方法装饰器
function Get(path: string): MethodDecorator {
  return (
    target: Object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) => {
    // target: 类的原型
    // propertyKey: 方法名
    // descriptor: 属性描述符
    Reflect.defineMetadata('path', path, target.constructor, propertyKey);
    Reflect.defineMetadata('method', 'GET', target.constructor, propertyKey);
  };
}

// 参数装饰器
function Body(): ParameterDecorator {
  return (
    target: Object,
    propertyKey: string | symbol,
    parameterIndex: number
  ) => {
    const existingParameters = Reflect.getMetadata('parameters', target.constructor, propertyKey) || [];
    existingParameters[parameterIndex] = { type: 'body' };
    Reflect.defineMetadata('parameters', existingParameters, target.constructor, propertyKey);
  };
}
```

### NestJS 装饰器实现

```typescript
// @Controller 装饰器实现
export function Controller(prefix: string = ''): ClassDecorator {
  return (target: Function) => {
    // 存储路由前缀
    Reflect.defineMetadata(PATH_METADATA, prefix, target);
    
    // 标记为控制器
    Reflect.defineMetadata('__controller__', true, target);
  };
}

// @Get 装饰器实现
export function Get(path: string = ''): MethodDecorator {
  return (
    target: Object,
    propertyKey: string | symbol,
    descriptor: PropertyDescriptor
  ) => {
    // 存储路由路径
    Reflect.defineMetadata(PATH_METADATA, path, descriptor.value);
    
    // 存储 HTTP 方法
    Reflect.defineMetadata(METHOD_METADATA, RequestMethod.GET, descriptor.value);
    
    return descriptor;
  };
}

// @Param 装饰器实现
export function Param(property?: string): ParameterDecorator {
  return (target: Object, propertyKey: string | symbol, parameterIndex: number) => {
    const existingParameters = Reflect.getMetadata(ROUTE_ARGS_METADATA, target.constructor, propertyKey) || {};
    
    existingParameters[parameterIndex] = {
      index: parameterIndex,
      type: 'param',
      data: property
    };
    
    Reflect.defineMetadata(ROUTE_ARGS_METADATA, existingParameters, target.constructor, propertyKey);
  };
}

// 使用示例
@Controller('users')
export class UserController {
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `User ${id}`;
  }
}

// NestJS 内部读取元数据
const controllerPath = Reflect.getMetadata(PATH_METADATA, UserController);
// 'users'

const methods = Object.getOwnPropertyNames(UserController.prototype);
methods.forEach(methodName => {
  const method = UserController.prototype[methodName];
  const httpMethod = Reflect.getMetadata(METHOD_METADATA, method);
  const path = Reflect.getMetadata(PATH_METADATA, method);
  
  console.log(`${httpMethod} /${controllerPath}/${path}`);
  // GET /users/:id
});
```

## 依赖注入（DI）原理

### IoC 容器实现

```typescript
// 简化版 IoC 容器
class IoCContainer {
  private providers = new Map<any, any>();
  private instances = new Map<any, any>();
  
  // 注册提供者
  register(token: any, provider: any) {
    this.providers.set(token, provider);
  }
  
  // 解析依赖
  resolve<T>(token: any): T {
    // 1. 检查是否已有实例（单例）
    if (this.instances.has(token)) {
      return this.instances.get(token);
    }
    
    // 2. 获取提供者
    const provider = this.providers.get(token);
    
    if (!provider) {
      throw new Error(`No provider found for ${token}`);
    }
    
    // 3. 解析依赖
    const instance = this.createInstance(provider);
    
    // 4. 缓存实例
    this.instances.set(token, instance);
    
    return instance;
  }
  
  private createInstance(target: any) {
    // 获取构造函数参数类型
    const paramTypes = Reflect.getMetadata('design:paramtypes', target) || [];
    
    // 递归解析依赖
    const dependencies = paramTypes.map((paramType: any) => {
      return this.resolve(paramType);
    });
    
    // 创建实例
    return new target(...dependencies);
  }
}

// 使用示例
@Injectable()
class UserRepository {
  findAll() {
    return ['user1', 'user2'];
  }
}

@Injectable()
class UserService {
  constructor(private userRepository: UserRepository) {}
  
  getUsers() {
    return this.userRepository.findAll();
  }
}

// 注册和解析
const container = new IoCContainer();
container.register(UserRepository, UserRepository);
container.register(UserService, UserService);

const userService = container.resolve(UserService);
console.log(userService.getUsers());
// ['user1', 'user2']
```

### NestJS 依赖注入完整实现

```typescript
// 作用域类型
enum Scope {
  DEFAULT = 'DEFAULT',    // 单例
  REQUEST = 'REQUEST',    // 请求作用域
  TRANSIENT = 'TRANSIENT' // 瞬态
}

// 提供者接口
interface Provider {
  provide: any;
  useClass?: any;
  useValue?: any;
  useFactory?: (...args: any[]) => any;
  inject?: any[];
  scope?: Scope;
}

class NestContainer {
  private modules = new Map<any, ModuleRef>();
  
  addModule(module: any) {
    const moduleRef = new ModuleRef(module);
    this.modules.set(module, moduleRef);
    return moduleRef;
  }
  
  getModule(module: any): ModuleRef {
    return this.modules.get(module);
  }
}

class ModuleRef {
  private providers = new Map<any, ProviderWrapper>();
  private controllers = new Map<any, ControllerWrapper>();
  private imports = new Set<ModuleRef>();
  private exports = new Set<any>();
  
  constructor(private module: any) {
    this.scanModule();
  }
  
  private scanModule() {
    // 读取模块元数据
    const imports = Reflect.getMetadata('imports', this.module) || [];
    const providers = Reflect.getMetadata('providers', this.module) || [];
    const controllers = Reflect.getMetadata('controllers', this.module) || [];
    const exports = Reflect.getMetadata('exports', this.module) || [];
    
    // 处理导入
    imports.forEach((importedModule: any) => {
      // ... 处理模块导入
    });
    
    // 注册提供者
    providers.forEach((provider: any) => {
      this.addProvider(provider);
    });
    
    // 注册控制器
    controllers.forEach((controller: any) => {
      this.addController(controller);
    });
    
    // 标记导出
    exports.forEach((token: any) => {
      this.exports.add(token);
    });
  }
  
  addProvider(provider: Provider) {
    const wrapper = new ProviderWrapper(provider);
    this.providers.set(provider.provide || provider, wrapper);
  }
  
  addController(controller: any) {
    const wrapper = new ControllerWrapper(controller);
    this.controllers.set(controller, wrapper);
  }
  
  // 解析实例
  get<T>(token: any, scope: Scope = Scope.DEFAULT): T {
    const wrapper = this.providers.get(token);
    
    if (!wrapper) {
      throw new Error(`Provider ${token} not found`);
    }
    
    return wrapper.getInstance(this, scope);
  }
}

class ProviderWrapper {
  private instance: any = null;
  private scope: Scope;
  
  constructor(private provider: Provider) {
    this.scope = provider.scope || Scope.DEFAULT;
  }
  
  getInstance(moduleRef: ModuleRef, requestScope?: Scope): any {
    // 单例模式
    if (this.scope === Scope.DEFAULT && this.instance) {
      return this.instance;
    }
    
    // 创建实例
    const instance = this.createInstance(moduleRef);
    
    // 缓存单例
    if (this.scope === Scope.DEFAULT) {
      this.instance = instance;
    }
    
    return instance;
  }
  
  private createInstance(moduleRef: ModuleRef): any {
    const provider = this.provider;
    
    // useValue
    if ('useValue' in provider) {
      return provider.useValue;
    }
    
    // useFactory
    if ('useFactory' in provider) {
      const factory = provider.useFactory!;
      const deps = (provider.inject || []).map(token => 
        moduleRef.get(token)
      );
      return factory(...deps);
    }
    
    // useClass
    const targetClass = provider.useClass || provider.provide;
    
    // 获取构造函数参数
    const paramTypes = Reflect.getMetadata('design:paramtypes', targetClass) || [];
    
    // 解析依赖
    const dependencies = paramTypes.map((paramType: any) => {
      return moduleRef.get(paramType);
    });
    
    // 创建实例
    return new targetClass(...dependencies);
  }
}

class ControllerWrapper {
  private instance: any = null;
  
  constructor(private controller: any) {}
  
  getInstance(moduleRef: ModuleRef): any {
    if (this.instance) {
      return this.instance;
    }
    
    // 解析控制器依赖
    const paramTypes = Reflect.getMetadata('design:paramtypes', this.controller) || [];
    const dependencies = paramTypes.map((paramType: any) => 
      moduleRef.get(paramType)
    );
    
    this.instance = new this.controller(...dependencies);
    return this.instance;
  }
}
```

### 循环依赖处理

```typescript
// NestJS 使用 forwardRef 处理循环依赖
@Injectable()
class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))
    private serviceB: ServiceB
  ) {}
}

@Injectable()
class ServiceB {
  constructor(
    @Inject(forwardRef(() => ServiceA))
    private serviceA: ServiceA
  ) {}
}

// forwardRef 实现原理
function forwardRef(fn: () => any) {
  return {
    forwardRef: fn
  };
}

// 容器处理 forwardRef
class Container {
  resolve(token: any) {
    // 检查是否是 forwardRef
    if (token && typeof token.forwardRef === 'function') {
      // 延迟获取真实类型
      token = token.forwardRef();
    }
    
    // 继续正常解析
    return this.createInstance(token);
  }
}
```

## 模块系统原理

### 模块加载流程

```typescript
class NestApplication {
  private container: NestContainer;
  
  async init(appModule: any) {
    // 1. 创建容器
    this.container = new NestContainer();
    
    // 2. 扫描模块树
    await this.scanModules(appModule);
    
    // 3. 初始化提供者
    await this.initializeProviders();
    
    // 4. 创建控制器实例
    await this.createControllers();
    
    // 5. 注册路由
    await this.registerRoutes();
    
    // 6. 调用生命周期钩子
    await this.callModuleInitHooks();
  }
  
  private async scanModules(module: any, parent?: ModuleRef) {
    // 检查是否已扫描
    if (this.container.getModule(module)) {
      return;
    }
    
    // 创建模块引用
    const moduleRef = this.container.addModule(module);
    
    // 读取导入的模块
    const imports = Reflect.getMetadata('imports', module) || [];
    
    // 递归扫描导入的模块
    for (const importedModule of imports) {
      // 处理动态模块
      if (this.isDynamicModule(importedModule)) {
        await this.scanModules(importedModule.module, moduleRef);
      } else {
        await this.scanModules(importedModule, moduleRef);
      }
    }
  }
  
  private isDynamicModule(module: any): boolean {
    return module && module.module;
  }
  
  private async initializeProviders() {
    // 遍历所有模块
    for (const moduleRef of this.container.getAllModules()) {
      const providers = moduleRef.getProviders();
      
      // 实例化所有单例提供者
      for (const provider of providers) {
        if (provider.scope === Scope.DEFAULT) {
          await provider.getInstance(moduleRef);
        }
      }
    }
  }
  
  private async createControllers() {
    for (const moduleRef of this.container.getAllModules()) {
      const controllers = moduleRef.getControllers();
      
      // 创建控制器实例
      for (const controller of controllers) {
        await controller.getInstance(moduleRef);
      }
    }
  }
  
  private async registerRoutes() {
    const router = express.Router();
    
    for (const moduleRef of this.container.getAllModules()) {
      const controllers = moduleRef.getControllers();
      
      for (const controllerWrapper of controllers) {
        const controller = controllerWrapper.getTarget();
        const instance = controllerWrapper.getInstance(moduleRef);
        
        // 读取控制器路径
        const basePath = Reflect.getMetadata(PATH_METADATA, controller) || '';
        
        // 遍历控制器方法
        const prototype = Object.getPrototypeOf(instance);
        const methodNames = Object.getOwnPropertyNames(prototype)
          .filter(name => name !== 'constructor');
        
        for (const methodName of methodNames) {
          const method = prototype[methodName];
          
          // 读取方法元数据
          const httpMethod = Reflect.getMetadata(METHOD_METADATA, method);
          const path = Reflect.getMetadata(PATH_METADATA, method) || '';
          
          if (!httpMethod) continue;
          
          // 完整路径
          const fullPath = this.normalizePath(`${basePath}/${path}`);
          
          // 注册路由
          router[httpMethod.toLowerCase()](fullPath, async (req, res, next) => {
            try {
              // 解析参数
              const args = await this.resolveParameters(req, res, next, controller, methodName);
              
              // 调用处理器
              const result = await instance[methodName](...args);
              
              // 发送响应
              res.json(result);
            } catch (error) {
              next(error);
            }
          });
        }
      }
    }
    
    return router;
  }
  
  private async resolveParameters(
    req: any,
    res: any,
    next: any,
    controller: any,
    methodName: string
  ): Promise<any[]> {
    const parametersMetadata = Reflect.getMetadata(
      ROUTE_ARGS_METADATA,
      controller,
      methodName
    ) || {};
    
    const paramTypes = Reflect.getMetadata(
      'design:paramtypes',
      controller.prototype,
      methodName
    ) || [];
    
    const args = new Array(paramTypes.length);
    
    // 解析每个参数
    for (const [index, param] of Object.entries(parametersMetadata)) {
      const paramIndex = parseInt(index);
      
      switch (param.type) {
        case 'body':
          args[paramIndex] = req.body;
          break;
        case 'param':
          args[paramIndex] = param.data 
            ? req.params[param.data] 
            : req.params;
          break;
        case 'query':
          args[paramIndex] = param.data 
            ? req.query[param.data] 
            : req.query;
          break;
        case 'req':
          args[paramIndex] = req;
          break;
        case 'res':
          args[paramIndex] = res;
          break;
        case 'next':
          args[paramIndex] = next;
          break;
      }
    }
    
    return args;
  }
}
```

## 请求处理流程

### 完整的请求管道

```typescript
class RequestPipeline {
  async handle(req: Request, res: Response, next: NextFunction) {
    try {
      // 1. 执行中间件
      await this.executeMiddleware(req, res);
      
      // 2. 执行守卫
      const canActivate = await this.executeGuards(req);
      if (!canActivate) {
        throw new ForbiddenException();
      }
      
      // 3. 执行拦截器（前置）
      await this.executeInterceptorsBefore(req);
      
      // 4. 执行管道（参数转换和验证）
      const transformedArgs = await this.executePipes(req);
      
      // 5. 调用路由处理器
      let result = await this.executeHandler(transformedArgs);
      
      // 6. 执行拦截器（后置）
      result = await this.executeInterceptorsAfter(result);
      
      // 7. 发送响应
      res.json(result);
      
    } catch (error) {
      // 8. 执行异常过滤器
      this.executeExceptionFilters(error, req, res);
    }
  }
  
  private async executeMiddleware(req: Request, res: Response) {
    // 执行全局中间件
    for (const middleware of this.globalMiddleware) {
      await new Promise((resolve, reject) => {
        middleware(req, res, (error?: any) => {
          if (error) reject(error);
          else resolve(undefined);
        });
      });
    }
    
    // 执行路由中间件
    for (const middleware of this.routeMiddleware) {
      await middleware.use(req, res, () => {});
    }
  }
  
  private async executeGuards(req: Request): Promise<boolean> {
    const context = this.createExecutionContext(req);
    
    // 执行全局守卫
    for (const guard of this.globalGuards) {
      const canActivate = await guard.canActivate(context);
      if (!canActivate) return false;
    }
    
    // 执行路由守卫
    for (const guard of this.routeGuards) {
      const canActivate = await guard.canActivate(context);
      if (!canActivate) return false;
    }
    
    return true;
  }
  
  private async executeInterceptorsBefore(req: Request) {
    const context = this.createExecutionContext(req);
    
    for (const interceptor of this.interceptors) {
      await interceptor.intercept(context, {
        handle: () => of(null)
      });
    }
  }
  
  private async executePipes(req: Request): Promise<any[]> {
    const args = this.extractArguments(req);
    const transformedArgs = [];
    
    for (let i = 0; i < args.length; i++) {
      let arg = args[i];
      
      // 执行参数级管道
      for (const pipe of this.paramPipes[i] || []) {
        arg = await pipe.transform(arg, {
          type: 'param',
          metatype: this.paramTypes[i]
        });
      }
      
      // 执行全局管道
      for (const pipe of this.globalPipes) {
        arg = await pipe.transform(arg, {
          type: 'param',
          metatype: this.paramTypes[i]
        });
      }
      
      transformedArgs.push(arg);
    }
    
    return transformedArgs;
  }
  
  private createExecutionContext(req: Request): ExecutionContext {
    return {
      switchToHttp: () => ({
        getRequest: () => req,
        getResponse: () => res,
        getNext: () => next
      }),
      getClass: () => this.controller,
      getHandler: () => this.handler
    };
  }
}
```

## 生命周期钩子

### 生命周期顺序

```typescript
// 生命周期接口
interface OnModuleInit {
  onModuleInit(): Promise<void> | void;
}

interface OnApplicationBootstrap {
  onApplicationBootstrap(): Promise<void> | void;
}

interface OnModuleDestroy {
  onModuleDestroy(): Promise<void> | void;
}

interface BeforeApplicationShutdown {
  beforeApplicationShutdown(signal?: string): Promise<void> | void;
}

interface OnApplicationShutdown {
  onApplicationShutdown(signal?: string): Promise<void> | void;
}

// 生命周期管理器
class LifecycleManager {
  async callInitHooks(instances: any[]) {
    // 1. onModuleInit
    for (const instance of instances) {
      if (this.hasHook(instance, 'onModuleInit')) {
        await instance.onModuleInit();
      }
    }
    
    // 2. onApplicationBootstrap
    for (const instance of instances) {
      if (this.hasHook(instance, 'onApplicationBootstrap')) {
        await instance.onApplicationBootstrap();
      }
    }
  }
  
  async callDestroyHooks(instances: any[], signal?: string) {
    // 1. beforeApplicationShutdown
    for (const instance of instances) {
      if (this.hasHook(instance, 'beforeApplicationShutdown')) {
        await instance.beforeApplicationShutdown(signal);
      }
    }
    
    // 2. onModuleDestroy
    for (const instance of instances) {
      if (this.hasHook(instance, 'onModuleDestroy')) {
        await instance.onModuleDestroy();
      }
    }
    
    // 3. onApplicationShutdown
    for (const instance of instances) {
      if (this.hasHook(instance, 'onApplicationShutdown')) {
        await instance.onApplicationShutdown(signal);
      }
    }
  }
  
  private hasHook(instance: any, hookName: string): boolean {
    return instance && typeof instance[hookName] === 'function';
  }
}

// 使用示例
@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy {
  private connection: Connection;
  
  async onModuleInit() {
    console.log('Connecting to database...');
    this.connection = await createConnection();
  }
  
  async onModuleDestroy() {
    console.log('Closing database connection...');
    await this.connection.close();
  }
}
```

## AOP（面向切面编程）实现

### 拦截器的 AOP 实现

```typescript
// 拦截器接口
interface NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler
  ): Observable<any>;
}

// CallHandler 实现
class CallHandler {
  constructor(private handler: Function, private args: any[]) {}
  
  handle(): Observable<any> {
    return from(this.handler(...this.args));
  }
}

// 拦截器执行器
class InterceptorsExecutor {
  async execute(
    interceptors: NestInterceptor[],
    context: ExecutionContext,
    handler: Function,
    args: any[]
  ): Promise<any> {
    if (interceptors.length === 0) {
      return handler(...args);
    }
    
    // 构建拦截器链
    const callHandler = new CallHandler(handler, args);
    
    // 从后往前包装
    let stream$ = callHandler.handle();
    
    for (let i = interceptors.length - 1; i >= 0; i--) {
      const interceptor = interceptors[i];
      const nextHandler = {
        handle: () => stream$
      };
      stream$ = interceptor.intercept(context, nextHandler);
    }
    
    // 执行拦截器链
    return firstValueFrom(stream$);
  }
}

// 示例：日志拦截器
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    
    return next.handle().pipe(
      tap(() => {
        const elapsed = Date.now() - now;
        console.log(`Request took ${elapsed}ms`);
      })
    );
  }
}
```

## 反射元数据（Reflect Metadata）

### 元数据存储和读取

```typescript
// NestJS 使用 reflect-metadata 库
import 'reflect-metadata';

// 定义元数据键
const METADATA_KEY = {
  PATH: 'path',
  METHOD: 'method',
  PARAMS: 'params',
  GUARDS: 'guards',
  INTERCEPTORS: 'interceptors'
};

// 存储元数据
function setMetadata(key: string, value: any, target: any, propertyKey?: string) {
  if (propertyKey) {
    Reflect.defineMetadata(key, value, target, propertyKey);
  } else {
    Reflect.defineMetadata(key, value, target);
  }
}

// 读取元数据
function getMetadata(key: string, target: any, propertyKey?: string): any {
  if (propertyKey) {
    return Reflect.getMetadata(key, target, propertyKey);
  } else {
    return Reflect.getMetadata(key, target);
  }
}

// 合并元数据（装饰器组合）
function mergeMetadata(key: string, value: any[], target: any, propertyKey?: string) {
  const existing = getMetadata(key, target, propertyKey) || [];
  setMetadata(key, [...existing, ...value], target, propertyKey);
}

// 示例：自定义装饰器
function UseGuards(...guards: any[]): MethodDecorator {
  return (target: Object, propertyKey: string | symbol, descriptor: PropertyDescriptor) => {
    mergeMetadata(METADATA_KEY.GUARDS, guards, target.constructor, propertyKey as string);
    return descriptor;
  };
}

// 读取守卫
function getGuards(target: any, methodName: string): any[] {
  // 方法级守卫
  const methodGuards = getMetadata(METADATA_KEY.GUARDS, target, methodName) || [];
  
  // 类级守卫
  const classGuards = getMetadata(METADATA_KEY.GUARDS, target) || [];
  
  // 合并
  return [...classGuards, ...methodGuards];
}
```

### 自动类型推断

```typescript
// TypeScript 自动生成的元数据
// 需要在 tsconfig.json 中启用
// "emitDecoratorMetadata": true

@Injectable()
class UserService {
  constructor(
    private userRepository: UserRepository,
    private emailService: EmailService
  ) {}
}

// TypeScript 编译后会自动添加：
Reflect.defineMetadata('design:paramtypes', [UserRepository, EmailService], UserService);

// NestJS 可以读取这个元数据来解析依赖
const paramTypes = Reflect.getMetadata('design:paramtypes', UserService);
// [UserRepository, EmailService]

// 方法参数类型
class UserController {
  findOne(@Param('id') id: string) {}
}

// TypeScript 生成：
Reflect.defineMetadata('design:paramtypes', [String], UserController.prototype, 'findOne');

const methodParamTypes = Reflect.getMetadata('design:paramtypes', UserController.prototype, 'findOne');
// [String]
```

## 性能优化原理

### 1. 单例模式

```typescript
// 默认所有提供者都是单例
@Injectable()
class ExpensiveService {
  private cache = new Map();
  
  constructor() {
    // 只执行一次
    this.initializeExpensiveOperation();
  }
  
  private initializeExpensiveOperation() {
    // 耗时操作
  }
}

// 整个应用共享一个实例
```

### 2. 懒加载模块

```typescript
// 动态导入模块
@Module({})
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(async (req, res, next) => {
        if (req.path.startsWith('/admin')) {
          // 懒加载 admin 模块
          const { AdminModule } = await import('./admin/admin.module');
          // 注册模块
          await app.registerModule(AdminModule);
        }
        next();
      })
      .forRoutes('*');
  }
}
```

### 3. 请求作用域优化

```typescript
// 避免不必要的 REQUEST 作用域
@Injectable({ scope: Scope.DEFAULT })  // ✅ 单例
class FastService {}

@Injectable({ scope: Scope.REQUEST })  // ❌ 每请求创建
class SlowService {}

// 只在真正需要请求上下文时使用 REQUEST 作用域
@Injectable({ scope: Scope.REQUEST })
class RequestContextService {
  constructor(@Inject(REQUEST) private request: Request) {}
  
  getUserFromRequest() {
    return this.request.user;
  }
}
```

## 常见面试题

### 1. NestJS 的依赖注入是如何实现的？

<details>
<summary>点击查看答案</summary>

**核心原理**：

1. **装饰器收集元数据**：
```typescript
@Injectable()  // 标记为可注入
class UserService {
  constructor(private repo: UserRepository) {}
  // TypeScript 自动生成 design:paramtypes 元数据
}
```

2. **容器管理实例**：
- IoC 容器存储所有提供者
- 通过 token 查找提供者

3. **解析依赖**：
```typescript
// 读取构造函数参数类型
const paramTypes = Reflect.getMetadata('design:paramtypes', UserService);
// [UserRepository]

// 递归解析依赖
const dependencies = paramTypes.map(type => container.get(type));

// 创建实例
const instance = new UserService(...dependencies);
```

4. **作用域管理**：
- DEFAULT: 单例，缓存实例
- REQUEST: 每个请求创建新实例
- TRANSIENT: 每次注入创建新实例

**关键点**：
- 依赖 TypeScript 的 `emitDecoratorMetadata`
- 使用 `reflect-metadata` 库
- 支持循环依赖（forwardRef）
</details>

### 2. 装饰器的执行顺序是什么？

<details>
<summary>点击查看答案</summary>

**执行顺序**：

1. **参数装饰器**（从右到左，从下到上）
2. **方法/访问器/属性装饰器**（从下到上）
3. **类装饰器**

```typescript
function ClassDec() {
  console.log('4. 类装饰器');
  return (target: any) => {};
}

function MethodDec() {
  console.log('3. 方法装饰器');
  return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {};
}

function ParamDec1() {
  console.log('1. 参数装饰器1');
  return (target: any, propertyKey: string, parameterIndex: number) => {};
}

function ParamDec2() {
  console.log('2. 参数装饰器2');
  return (target: any, propertyKey: string, parameterIndex: number) => {};
}

@ClassDec()
class Example {
  @MethodDec()
  method(@ParamDec1() param1: string, @ParamDec2() param2: string) {}
}

// 输出：
// 1. 参数装饰器1
// 2. 参数装饰器2
// 3. 方法装饰器
// 4. 类装饰器
```

**多个同类型装饰器**：
```typescript
@Guard1()
@Guard2()
class Controller {
  @Interceptor1()
  @Interceptor2()
  method() {}
}

// 守卫执行顺序：Guard1 → Guard2
// 拦截器执行顺序：Interceptor1 → Interceptor2
```
</details>

### 3. NestJS 的模块是如何加载的？

<details>
<summary>点击查看答案</summary>

**加载流程**：

1. **扫描根模块**：
```typescript
const app = await NestFactory.create(AppModule);
```

2. **递归扫描导入**：
```typescript
@Module({
  imports: [UserModule, OrderModule],
  // ...
})
class AppModule {}

// 深度优先遍历模块树
```

3. **构建模块图**：
- 创建 ModuleRef 对象
- 建立模块间依赖关系
- 处理模块导入导出

4. **实例化提供者**：
- 按依赖顺序实例化
- 处理循环依赖
- 缓存单例实例

5. **创建控制器**：
- 实例化控制器
- 解析路由元数据
- 注册路由处理器

6. **调用生命周期钩子**：
```typescript
onModuleInit → onApplicationBootstrap
```

**动态模块处理**：
```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ folder: './config' })
  ]
})
class AppModule {}

// forRoot 返回 DynamicModule
// { module: ConfigModule, providers: [...], exports: [...] }
```
</details>

### 4. 请求在 NestJS 中是如何处理的？

<details>
<summary>点击查看答案</summary>

**完整流程**：

```
HTTP Request
  ↓
1. Middleware（全局 → 路由）
  ↓
2. Guards（全局 → 控制器 → 路由）
  ↓
3. Interceptors Before（全局 → 控制器 → 路由）
  ↓
4. Pipes（全局 → 控制器 → 路由 → 参数）
  ↓
5. Route Handler（控制器方法）
  ↓
6. Interceptors After（路由 → 控制器 → 全局）
  ↓
7. Exception Filters（路由 → 控制器 → 全局）
  ↓
HTTP Response
```

**示例**：
```typescript
@Controller('users')
@UseGuards(AuthGuard)  // 2
@UseInterceptors(LoggingInterceptor)  // 3, 6
export class UserController {
  @Get(':id')
  @UseGuards(RolesGuard)  // 2
  @UsePipes(ValidationPipe)  // 4
  findOne(
    @Param('id', ParseIntPipe) id: number  // 4
  ) {
    return this.userService.findOne(id);  // 5
  }
}
```

**关键点**：
- 守卫决定是否执行
- 管道转换和验证参数
- 拦截器可以修改请求/响应
- 异常过滤器处理错误
</details>

### 5. forwardRef 是如何解决循环依赖的？

<details>
<summary>点击查看答案</summary>

**问题**：
```typescript
// 循环依赖
@Injectable()
class ServiceA {
  constructor(private serviceB: ServiceB) {}  // ❌ ServiceB 未定义
}

@Injectable()
class ServiceB {
  constructor(private serviceA: ServiceA) {}  // ❌ ServiceA 未定义
}
```

**解决方案**：
```typescript
@Injectable()
class ServiceA {
  constructor(
    @Inject(forwardRef(() => ServiceB))  // ✅ 延迟引用
    private serviceB: ServiceB
  ) {}
}

@Injectable()
class ServiceB {
  constructor(
    @Inject(forwardRef(() => ServiceA))
    private serviceA: ServiceA
  ) {}
}
```

**forwardRef 实现原理**：
```typescript
function forwardRef(fn: () => any) {
  // 返回一个包装对象
  return {
    forwardRef: fn,
    __forward_ref__: true
  };
}

// 容器解析时
function resolve(token: any) {
  // 检查是否是 forwardRef
  if (token && token.__forward_ref__) {
    // 调用函数获取真实类型
    token = token.forwardRef();
  }
  
  return this.getInstance(token);
}
```

**注意**：
- 只在必要时使用
- 优先通过架构设计避免循环依赖
- 模块级别循环依赖用 `forwardRef(() => Module)`
</details>

### 6. NestJS 如何实现 AOP？

<details>
<summary>点击查看答案</summary>

**AOP（面向切面编程）实现**：

主要通过 **拦截器（Interceptors）** 实现：

```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const now = Date.now();
    
    // 前置逻辑
    console.log('Before...');
    
    return next.handle().pipe(
      // 后置逻辑
      tap(() => {
        console.log(`After... ${Date.now() - now}ms`);
      }),
      
      // 转换响应
      map(data => ({ data, timestamp: new Date() })),
      
      // 错误处理
      catchError(err => {
        console.error('Error:', err);
        return throwError(() => err);
      })
    );
  }
}
```

**实现原理**：

1. **包装处理器**：
```typescript
const handler = () => controller.method();
const wrappedHandler = interceptor.intercept(context, { handle: handler });
```

2. **RxJS 管道**：
```typescript
// 多个拦截器形成管道
handler
  .pipe(interceptor1)
  .pipe(interceptor2)
  .pipe(interceptor3)
```

3. **执行顺序**：
- 前置：Interceptor1 → Interceptor2 → Handler
- 后置：Handler → Interceptor2 → Interceptor1

**应用场景**：
- 日志记录
- 性能监控
- 缓存
- 响应转换
- 超时处理
- 错误处理
</details>

## 总结

NestJS 底层原理的核心：

1. **装饰器 + 元数据**：声明式配置
2. **依赖注入**：IoC 容器管理依赖
3. **模块系统**：组织代码结构
4. **请求管道**：标准化请求处理
5. **AOP**：横切关注点分离

理解这些原理有助于：
- 更好地使用框架
- 调试问题
- 性能优化
- 架构设计

