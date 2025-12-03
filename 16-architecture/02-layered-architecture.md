# 分层架构

分层架构是组织代码的基本方式。本文深入讲解各种分层架构模式及其实践。

## 目录
- [传统三层架构](#传统三层架构)
- [MVC 架构](#mvc-架构)
- [洋葱架构](#洋葱架构)
- [六边形架构](#六边形架构)
- [Clean Architecture](#clean-architecture)
- [实战对比](#实战对比)
- [面试题](#常见面试题)

---

## 传统三层架构

### 架构图

```
┌─────────────────────────────────────────┐
│           表示层 (Presentation)          │
│         Controller / API                │
├─────────────────────────────────────────┤
│           业务逻辑层 (Business)           │
│              Service                    │
├─────────────────────────────────────────┤
│           数据访问层 (Data Access)        │
│           Repository / DAO              │
└─────────────────────────────────────────┘
                    │
                    ▼
              ┌───────────┐
              │  Database │
              └───────────┘

依赖方向：上层依赖下层
```

### 代码结构

```typescript
// 项目结构
src/
├── controllers/          # 表示层
│   └── userController.ts
├── services/             # 业务逻辑层
│   └── userService.ts
├── repositories/         # 数据访问层
│   └── userRepository.ts
├── models/               # 数据模型
│   └── User.ts
├── dto/                  # 数据传输对象
│   └── userDto.ts
└── app.ts

// controllers/userController.ts
import { Request, Response } from 'express';
import { UserService } from '../services/userService';

export class UserController {
  constructor(private userService: UserService) {}

  async getUser(req: Request, res: Response) {
    try {
      const user = await this.userService.getUserById(req.params.id);
      res.json(user);
    } catch (error) {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  }

  async createUser(req: Request, res: Response) {
    try {
      const user = await this.userService.createUser(req.body);
      res.status(201).json(user);
    } catch (error) {
      if (error instanceof ValidationError) {
        res.status(400).json({ error: error.message });
      } else {
        res.status(500).json({ error: 'Internal Server Error' });
      }
    }
  }
}

// services/userService.ts
import { UserRepository } from '../repositories/userRepository';
import { CreateUserDto, UserDto } from '../dto/userDto';

export class UserService {
  constructor(private userRepository: UserRepository) {}

  async getUserById(id: string): Promise<UserDto> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundError('User not found');
    }
    return this.toDto(user);
  }

  async createUser(dto: CreateUserDto): Promise<UserDto> {
    // 业务逻辑
    await this.validateEmail(dto.email);
    
    const hashedPassword = await this.hashPassword(dto.password);
    
    const user = await this.userRepository.create({
      ...dto,
      password: hashedPassword
    });
    
    return this.toDto(user);
  }

  private toDto(user: User): UserDto {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      createdAt: user.createdAt
    };
  }
}

// repositories/userRepository.ts
import { PrismaClient, User } from '@prisma/client';

export class UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async create(data: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): Promise<User> {
    return this.prisma.user.create({ data });
  }

  async update(id: string, data: Partial<User>): Promise<User> {
    return this.prisma.user.update({ where: { id }, data });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }
}
```

### 优缺点

**优点**：
- 简单易懂
- 职责分明
- 易于测试（可 mock 下层）

**缺点**：
- 业务逻辑层依赖数据访问层
- 难以替换基础设施
- 随着复杂度增加难以维护

---

## MVC 架构

### 架构图

```
┌─────────┐       ┌─────────┐       ┌─────────┐
│  View   │ <──── │Controller│ ───> │  Model  │
└─────────┘       └─────────┘       └─────────┘
     │                 │                 │
     │                 ▼                 │
     │            ┌─────────┐            │
     └──────────> │ Service │ <──────────┘
                  └─────────┘

MVC 在后端 API 中的变体：
View = JSON Response
```

### Express MVC

```typescript
// models/User.ts
import { Schema, model } from 'mongoose';

const userSchema = new Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true }
}, { timestamps: true });

export const User = model('User', userSchema);

// controllers/userController.ts
import { Request, Response, NextFunction } from 'express';
import { User } from '../models/User';

export const userController = {
  async index(req: Request, res: Response, next: NextFunction) {
    try {
      const users = await User.find().select('-password');
      res.json(users);
    } catch (error) {
      next(error);
    }
  },

  async show(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await User.findById(req.params.id).select('-password');
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      res.json(user);
    } catch (error) {
      next(error);
    }
  },

  async create(req: Request, res: Response, next: NextFunction) {
    try {
      const user = new User(req.body);
      await user.save();
      res.status(201).json(user);
    } catch (error) {
      next(error);
    }
  },

  async update(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await User.findByIdAndUpdate(
        req.params.id,
        req.body,
        { new: true, runValidators: true }
      ).select('-password');
      
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      res.json(user);
    } catch (error) {
      next(error);
    }
  },

  async destroy(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await User.findByIdAndDelete(req.params.id);
      if (!user) {
        return res.status(404).json({ error: 'User not found' });
      }
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
};

// routes/userRoutes.ts
import { Router } from 'express';
import { userController } from '../controllers/userController';

const router = Router();

router.get('/users', userController.index);
router.get('/users/:id', userController.show);
router.post('/users', userController.create);
router.put('/users/:id', userController.update);
router.delete('/users/:id', userController.destroy);

export default router;
```

---

## 洋葱架构

### 架构图

```
         ┌─────────────────────────────────────┐
         │       Infrastructure / UI           │
         │  ┌─────────────────────────────┐   │
         │  │    Interface Adapters        │   │
         │  │  ┌─────────────────────┐    │   │
         │  │  │   Application        │    │   │
         │  │  │  ┌─────────────┐    │    │   │
         │  │  │  │   Domain     │    │    │   │
         │  │  │  │  (Entities)  │    │    │   │
         │  │  │  └─────────────┘    │    │   │
         │  │  └─────────────────────┘    │   │
         │  └─────────────────────────────┘   │
         └─────────────────────────────────────┘

依赖方向：由外向内
核心（Domain）不依赖任何外层
```

### 核心原则

```typescript
// 1. 依赖倒置（DIP）：内层定义接口，外层实现

// domain/repositories/IUserRepository.ts（内层定义接口）
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<void>;
}

// infrastructure/repositories/PrismaUserRepository.ts（外层实现）
import { IUserRepository } from '../../domain/repositories/IUserRepository';

export class PrismaUserRepository implements IUserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async save(user: User): Promise<User> {
    return this.prisma.user.upsert({
      where: { id: user.id },
      update: user,
      create: user
    });
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({ where: { id } });
  }
}

// 2. 核心领域不依赖框架

// domain/entities/User.ts（纯 TypeScript，无框架依赖）
export class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: string,
    private passwordHash: string,
    public readonly createdAt: Date
  ) {}

  static create(name: string, email: string, password: string): User {
    const id = crypto.randomUUID();
    const passwordHash = this.hashPassword(password);
    return new User(id, name, email, passwordHash, new Date());
  }

  changeName(newName: string): void {
    if (newName.length < 2) {
      throw new DomainError('Name must be at least 2 characters');
    }
    this.name = newName;
  }

  verifyPassword(password: string): boolean {
    return this.passwordHash === User.hashPassword(password);
  }

  private static hashPassword(password: string): string {
    // 简化示例
    return require('crypto').createHash('sha256').update(password).digest('hex');
  }
}
```

### 项目结构

```
src/
├── domain/                      # 核心层（最内层）
│   ├── entities/
│   │   └── User.ts
│   ├── value-objects/
│   │   └── Email.ts
│   ├── repositories/            # 接口定义
│   │   └── IUserRepository.ts
│   └── services/                # 领域服务
│       └── UserDomainService.ts
│
├── application/                 # 应用层
│   ├── use-cases/
│   │   ├── CreateUserUseCase.ts
│   │   └── GetUserUseCase.ts
│   ├── dto/
│   │   └── UserDto.ts
│   └── services/                # 接口定义
│       └── IEmailService.ts
│
├── infrastructure/              # 基础设施层（最外层）
│   ├── repositories/            # 接口实现
│   │   └── PrismaUserRepository.ts
│   ├── services/
│   │   └── SendGridEmailService.ts
│   └── persistence/
│       └── prisma.ts
│
└── presentation/                # 表示层（最外层）
    ├── controllers/
    │   └── UserController.ts
    └── routes/
        └── userRoutes.ts
```

---

## 六边形架构

### 架构图

```
                        Primary Adapters
                    (Controllers, CLI, Tests)
                              │
              ┌───────────────┼───────────────┐
              │               ▼               │
              │   ┌───────────────────────┐   │
    Ports ────│───│                       │───│──── Ports
              │   │    Application Core   │   │
   (Input)    │   │                       │   │   (Output)
              │   │   ┌───────────────┐   │   │
              │   │   │    Domain     │   │   │
              │   │   └───────────────┘   │   │
              │   └───────────────────────┘   │
              │               │               │
              └───────────────┼───────────────┘
                              │
                    Secondary Adapters
              (Database, Message Queue, External APIs)
```

### 核心概念

```typescript
// Port = 接口（应用核心定义）
// Adapter = 实现（外部适配）

// 1. 输入端口（Primary/Driving Port）
// application/ports/input/CreateUserPort.ts
export interface CreateUserPort {
  execute(command: CreateUserCommand): Promise<UserDto>;
}

// 2. 输出端口（Secondary/Driven Port）
// application/ports/output/UserRepositoryPort.ts
export interface UserRepositoryPort {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// application/ports/output/EmailServicePort.ts
export interface EmailServicePort {
  sendWelcomeEmail(to: string, name: string): Promise<void>;
}

// 3. 输入适配器（Controller）
// adapters/input/http/UserController.ts
export class UserController {
  constructor(private createUserPort: CreateUserPort) {}

  async create(req: Request, res: Response) {
    const command: CreateUserCommand = {
      name: req.body.name,
      email: req.body.email,
      password: req.body.password
    };
    
    const user = await this.createUserPort.execute(command);
    res.status(201).json(user);
  }
}

// 4. 输出适配器（Repository）
// adapters/output/persistence/PrismaUserRepository.ts
export class PrismaUserRepository implements UserRepositoryPort {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const data = await this.prisma.user.findUnique({ where: { id } });
    return data ? this.toDomain(data) : null;
  }

  async save(user: User): Promise<void> {
    await this.prisma.user.upsert({
      where: { id: user.id },
      update: this.toPersistence(user),
      create: this.toPersistence(user)
    });
  }

  private toDomain(data: any): User {
    return new User(data.id, data.name, data.email, data.passwordHash, data.createdAt);
  }

  private toPersistence(user: User): any {
    return { id: user.id, name: user.name, email: user.email, passwordHash: user['passwordHash'] };
  }
}

// 5. 应用服务（实现输入端口）
// application/services/CreateUserService.ts
export class CreateUserService implements CreateUserPort {
  constructor(
    private userRepository: UserRepositoryPort,
    private emailService: EmailServicePort
  ) {}

  async execute(command: CreateUserCommand): Promise<UserDto> {
    // 创建领域对象
    const user = User.create(command.name, command.email, command.password);
    
    // 保存用户
    await this.userRepository.save(user);
    
    // 发送欢迎邮件
    await this.emailService.sendWelcomeEmail(user.email, user.name);
    
    // 返回 DTO
    return UserDto.from(user);
  }
}
```

### 项目结构

```
src/
├── domain/                      # 领域核心
│   ├── entities/
│   ├── value-objects/
│   └── services/
│
├── application/                 # 应用核心
│   ├── ports/
│   │   ├── input/              # 输入端口
│   │   │   └── CreateUserPort.ts
│   │   └── output/             # 输出端口
│   │       ├── UserRepositoryPort.ts
│   │       └── EmailServicePort.ts
│   ├── services/               # 端口实现
│   │   └── CreateUserService.ts
│   └── dto/
│
└── adapters/                    # 适配器
    ├── input/                   # 输入适配器
    │   ├── http/
    │   │   └── UserController.ts
    │   └── cli/
    │       └── UserCLI.ts
    └── output/                  # 输出适配器
        ├── persistence/
        │   └── PrismaUserRepository.ts
        └── email/
            └── SendGridEmailService.ts
```

---

## Clean Architecture

### 架构图

```
┌──────────────────────────────────────────────────────────┐
│                 Frameworks & Drivers                      │
│                   (Web, UI, DB, Devices)                  │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Interface Adapters                   │   │
│  │        (Controllers, Gateways, Presenters)        │   │
│  │  ┌──────────────────────────────────────────┐    │   │
│  │  │           Application Business Rules      │    │   │
│  │  │               (Use Cases)                 │    │   │
│  │  │  ┌──────────────────────────────────┐    │    │   │
│  │  │  │    Enterprise Business Rules      │    │    │   │
│  │  │  │           (Entities)              │    │    │   │
│  │  │  └──────────────────────────────────┘    │    │   │
│  │  └──────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘

依赖规则：依赖只能从外向内
内层对外层一无所知
```

### 各层职责

```typescript
// 1. Entities（实体层）- 企业业务规则
// domain/entities/Order.ts
export class Order {
  private items: OrderItem[] = [];
  private status: OrderStatus = OrderStatus.PENDING;

  constructor(
    public readonly id: string,
    public readonly userId: string,
    public readonly createdAt: Date
  ) {}

  addItem(productId: string, quantity: number, price: number): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new DomainError('Cannot modify a confirmed order');
    }
    
    const existing = this.items.find(item => item.productId === productId);
    if (existing) {
      existing.quantity += quantity;
    } else {
      this.items.push(new OrderItem(productId, quantity, price));
    }
  }

  removeItem(productId: string): void {
    if (this.status !== OrderStatus.PENDING) {
      throw new DomainError('Cannot modify a confirmed order');
    }
    this.items = this.items.filter(item => item.productId !== productId);
  }

  confirm(): void {
    if (this.items.length === 0) {
      throw new DomainError('Cannot confirm an empty order');
    }
    this.status = OrderStatus.CONFIRMED;
  }

  get total(): number {
    return this.items.reduce((sum, item) => sum + item.subtotal, 0);
  }

  get itemCount(): number {
    return this.items.length;
  }
}

// 2. Use Cases（用例层）- 应用业务规则
// application/use-cases/CreateOrderUseCase.ts
export interface CreateOrderInput {
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
}

export interface CreateOrderOutput {
  orderId: string;
  total: number;
  status: string;
}

export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private productRepository: ProductRepository,
    private paymentGateway: PaymentGateway
  ) {}

  async execute(input: CreateOrderInput): Promise<CreateOrderOutput> {
    // 1. 创建订单
    const order = new Order(
      crypto.randomUUID(),
      input.userId,
      new Date()
    );

    // 2. 添加商品
    for (const item of input.items) {
      const product = await this.productRepository.findById(item.productId);
      if (!product) {
        throw new NotFoundError(`Product ${item.productId} not found`);
      }
      order.addItem(product.id, item.quantity, product.price);
    }

    // 3. 保存订单
    await this.orderRepository.save(order);

    // 4. 返回结果
    return {
      orderId: order.id,
      total: order.total,
      status: 'pending'
    };
  }
}

// 3. Interface Adapters（接口适配器层）
// adapters/controllers/OrderController.ts
export class OrderController {
  constructor(
    private createOrderUseCase: CreateOrderUseCase,
    private getOrderUseCase: GetOrderUseCase
  ) {}

  async create(req: Request, res: Response): Promise<void> {
    const input: CreateOrderInput = {
      userId: req.user.id,
      items: req.body.items
    };

    try {
      const output = await this.createOrderUseCase.execute(input);
      res.status(201).json(output);
    } catch (error) {
      if (error instanceof NotFoundError) {
        res.status(404).json({ error: error.message });
      } else if (error instanceof DomainError) {
        res.status(400).json({ error: error.message });
      } else {
        res.status(500).json({ error: 'Internal Server Error' });
      }
    }
  }
}

// adapters/gateways/PrismaOrderRepository.ts
export class PrismaOrderRepository implements OrderRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true }
    });
    
    if (!data) return null;
    
    return this.toDomain(data);
  }

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id },
      update: this.toPersistence(order),
      create: this.toPersistence(order)
    });
  }
}

// 4. Frameworks & Drivers（框架和驱动层）
// infrastructure/express/app.ts
import express from 'express';
import { OrderController } from '../../adapters/controllers/OrderController';

const app = express();

// 依赖注入
const prisma = new PrismaClient();
const orderRepository = new PrismaOrderRepository(prisma);
const productRepository = new PrismaProductRepository(prisma);
const paymentGateway = new StripePaymentGateway();

const createOrderUseCase = new CreateOrderUseCase(
  orderRepository,
  productRepository,
  paymentGateway
);
const orderController = new OrderController(createOrderUseCase);

// 路由
app.post('/orders', (req, res) => orderController.create(req, res));
```

### NestJS 实现 Clean Architecture

```typescript
// 项目结构
src/
├── domain/
│   ├── entities/
│   │   └── order.entity.ts
│   └── repositories/
│       └── order.repository.interface.ts
│
├── application/
│   ├── use-cases/
│   │   └── create-order.use-case.ts
│   └── dto/
│       └── create-order.dto.ts
│
├── infrastructure/
│   ├── persistence/
│   │   └── prisma-order.repository.ts
│   └── modules/
│       └── database.module.ts
│
└── presentation/
    └── http/
        ├── controllers/
        │   └── order.controller.ts
        └── modules/
            └── order.module.ts

// order.module.ts（依赖注入配置）
@Module({
  imports: [DatabaseModule],
  controllers: [OrderController],
  providers: [
    CreateOrderUseCase,
    {
      provide: 'OrderRepository',
      useClass: PrismaOrderRepository
    }
  ]
})
export class OrderModule {}

// create-order.use-case.ts
@Injectable()
export class CreateOrderUseCase {
  constructor(
    @Inject('OrderRepository')
    private orderRepository: OrderRepository
  ) {}

  async execute(input: CreateOrderInput): Promise<CreateOrderOutput> {
    const order = new Order(input.userId);
    // ...
    await this.orderRepository.save(order);
    return { orderId: order.id };
  }
}

// order.controller.ts
@Controller('orders')
export class OrderController {
  constructor(private createOrderUseCase: CreateOrderUseCase) {}

  @Post()
  async create(@Body() dto: CreateOrderDto, @Req() req) {
    return this.createOrderUseCase.execute({
      userId: req.user.id,
      items: dto.items
    });
  }
}
```

---

## 实战对比

### 不同架构的代码对比

```typescript
// 场景：创建用户

// ===== 1. 传统三层架构 =====
// controller 直接依赖 service，service 直接依赖 repository
class UserController {
  constructor(private userService: UserService) {}
  
  async createUser(req, res) {
    const user = await this.userService.createUser(req.body);
    res.json(user);
  }
}

class UserService {
  constructor(private userRepository: UserRepository) {} // 直接依赖具体实现
  
  async createUser(data) {
    return this.userRepository.create(data);
  }
}

// ===== 2. 洋葱/六边形架构 =====
// 依赖倒置：内层定义接口，外层实现

// 内层：定义接口
interface IUserRepository {
  save(user: User): Promise<void>;
}

// 应用层：依赖接口
class CreateUserUseCase {
  constructor(private userRepository: IUserRepository) {} // 依赖接口
  
  async execute(data) {
    const user = User.create(data);
    await this.userRepository.save(user);
    return user;
  }
}

// 外层：实现接口
class PrismaUserRepository implements IUserRepository {
  async save(user: User) {
    await this.prisma.user.create({ data: user });
  }
}

// ===== 3. Clean Architecture =====
// 严格的层级划分

// Entity 层
class User {
  constructor(public id: string, public name: string) {}
  static create(name: string) {
    return new User(crypto.randomUUID(), name);
  }
}

// Use Case 层
class CreateUserUseCase {
  constructor(private userGateway: UserGateway) {}
  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    const user = User.create(input.name);
    await this.userGateway.save(user);
    return { id: user.id, name: user.name };
  }
}

// Interface Adapter 层
class UserController {
  constructor(private createUserUseCase: CreateUserUseCase) {}
  async create(req, res) {
    const output = await this.createUserUseCase.execute(req.body);
    res.json(output);
  }
}

// Framework 层
// Express、Prisma 等
```

### 何时使用哪种架构

| 场景 | 推荐架构 |
|------|---------|
| **简单 CRUD** | 三层架构 |
| **中等复杂度** | 洋葱/六边形架构 |
| **复杂领域逻辑** | Clean Architecture + DDD |
| **需要高测试覆盖** | 六边形/Clean |
| **需要替换基础设施** | 六边形/Clean |
| **快速原型** | 三层架构/MVC |

---

## 常见面试题

### 1. 什么是依赖倒置原则？

**依赖倒置原则（DIP）**：
- 高层模块不应该依赖低层模块，两者都应该依赖抽象
- 抽象不应该依赖细节，细节应该依赖抽象

```typescript
// ❌ 违反 DIP
class UserService {
  private repository = new PrismaUserRepository(); // 直接依赖具体实现
}

// ✅ 遵循 DIP
interface UserRepository {
  save(user: User): Promise<void>;
}

class UserService {
  constructor(private repository: UserRepository) {} // 依赖抽象
}
```

### 2. 洋葱架构和六边形架构的区别？

**相似点**：
- 都强调依赖倒置
- 核心业务不依赖外部

**区别**：
- **洋葱架构**：强调同心圆分层
- **六边形架构**：强调端口和适配器，更关注输入输出

### 3. Clean Architecture 的核心原则？

1. **依赖规则**：依赖只能从外向内
2. **实体**：封装企业业务规则
3. **用例**：封装应用业务规则
4. **接口适配器**：转换数据格式
5. **框架和驱动**：外部工具和框架

### 4. 为什么要分层？

**好处**：
- **关注点分离**：每层专注一个职责
- **可测试性**：可以独立测试每层
- **可维护性**：改动影响范围小
- **可替换性**：可以替换某一层实现

---

## 总结

### 架构对比

| 架构 | 复杂度 | 灵活性 | 测试性 | 适用场景 |
|------|--------|--------|--------|---------|
| **三层** | 低 | 低 | 中 | 简单应用 |
| **MVC** | 低 | 中 | 中 | Web 应用 |
| **洋葱** | 中 | 高 | 高 | 中型项目 |
| **六边形** | 中 | 高 | 高 | 需要适配器 |
| **Clean** | 高 | 高 | 高 | 复杂领域 |

### 实践检查清单

- [ ] 理解各种分层架构
- [ ] 掌握依赖倒置原则
- [ ] 能实现端口和适配器
- [ ] 理解 Clean Architecture 层级
- [ ] 能根据项目选择合适架构

---

**上一篇**：[架构模式](./01-architecture-patterns.md)  
**下一篇**：[领域驱动设计](./03-ddd.md)

