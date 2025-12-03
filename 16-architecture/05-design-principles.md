# 设计原则

设计原则是编写可维护、可扩展代码的指导方针。本文讲解 SOLID 原则和其他重要的设计原则。

## 目录
- [SOLID 原则](#solid-原则)
- [其他设计原则](#其他设计原则)
- [设计模式](#设计模式)
- [架构决策](#架构决策)
- [面试题](#常见面试题)

---

## SOLID 原则

### S - 单一职责原则（SRP）

```typescript
// Single Responsibility Principle
// 一个类只应该有一个引起它变化的原因

// ❌ 违反 SRP：User 类承担了多个职责
class User {
  constructor(
    public name: string,
    public email: string
  ) {}

  // 职责1：用户数据管理
  save(): void {
    // 保存到数据库
    db.save(this);
  }

  // 职责2：邮件发送
  sendEmail(subject: string, body: string): void {
    emailClient.send(this.email, subject, body);
  }

  // 职责3：日志记录
  log(message: string): void {
    fs.appendFile('log.txt', message);
  }

  // 职责4：数据验证
  validate(): boolean {
    return this.email.includes('@');
  }
}

// ✅ 遵循 SRP：每个类只有一个职责

// 职责1：用户实体
class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: Email
  ) {}

  changeName(name: string): void {
    this.name = name;
  }
}

// 职责2：用户持久化
class UserRepository {
  async save(user: User): Promise<void> {
    await this.prisma.user.upsert({
      where: { id: user.id },
      update: { name: user.name, email: user.email.value },
      create: { id: user.id, name: user.name, email: user.email.value }
    });
  }
}

// 职责3：邮件服务
class EmailService {
  async send(to: Email, subject: string, body: string): Promise<void> {
    await this.emailClient.send(to.value, subject, body);
  }
}

// 职责4：日志服务
class Logger {
  log(level: string, message: string): void {
    console.log(`[${level}] ${message}`);
  }
}

// 职责5：验证（通过值对象）
class Email {
  constructor(public readonly value: string) {
    if (!this.isValid(value)) {
      throw new InvalidEmailError();
    }
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}
```

### O - 开闭原则（OCP）

```typescript
// Open/Closed Principle
// 对扩展开放，对修改关闭

// ❌ 违反 OCP：每增加一种支付方式，都需要修改 PaymentProcessor
class PaymentProcessor {
  process(type: string, amount: number): void {
    if (type === 'credit_card') {
      // 处理信用卡支付
    } else if (type === 'paypal') {
      // 处理 PayPal 支付
    } else if (type === 'apple_pay') {
      // 新增 Apple Pay，需要修改这个类
    }
  }
}

// ✅ 遵循 OCP：通过接口扩展，无需修改现有代码

// 定义支付策略接口
interface PaymentStrategy {
  pay(amount: Money): Promise<PaymentResult>;
}

// 具体策略实现
class CreditCardPayment implements PaymentStrategy {
  constructor(private cardDetails: CardDetails) {}

  async pay(amount: Money): Promise<PaymentResult> {
    // 信用卡支付逻辑
    return { success: true, transactionId: 'xxx' };
  }
}

class PayPalPayment implements PaymentStrategy {
  constructor(private paypalAccount: string) {}

  async pay(amount: Money): Promise<PaymentResult> {
    // PayPal 支付逻辑
    return { success: true, transactionId: 'xxx' };
  }
}

// 新增 Apple Pay，只需添加新类，不修改现有代码
class ApplePayPayment implements PaymentStrategy {
  async pay(amount: Money): Promise<PaymentResult> {
    // Apple Pay 支付逻辑
    return { success: true, transactionId: 'xxx' };
  }
}

// 支付处理器使用策略
class PaymentProcessor {
  constructor(private strategy: PaymentStrategy) {}

  async process(amount: Money): Promise<PaymentResult> {
    return this.strategy.pay(amount);
  }
}

// 使用
const processor = new PaymentProcessor(new CreditCardPayment(cardDetails));
await processor.process(Money.of(100));
```

### L - 里氏替换原则（LSP）

```typescript
// Liskov Substitution Principle
// 子类型必须能够替换其基类型

// ❌ 违反 LSP：正方形不是长方形的合适子类
class Rectangle {
  constructor(
    protected width: number,
    protected height: number
  ) {}

  setWidth(width: number): void {
    this.width = width;
  }

  setHeight(height: number): void {
    this.height = height;
  }

  getArea(): number {
    return this.width * this.height;
  }
}

class Square extends Rectangle {
  setWidth(width: number): void {
    this.width = width;
    this.height = width; // 违反！改变了父类的行为
  }

  setHeight(height: number): void {
    this.height = height;
    this.width = height; // 违反！
  }
}

// 测试（使用父类引用）
function testRectangle(rect: Rectangle): void {
  rect.setWidth(5);
  rect.setHeight(4);
  console.assert(rect.getArea() === 20); // Square 会失败！
}

// ✅ 遵循 LSP：正确的继承关系

interface Shape {
  getArea(): number;
}

class Rectangle implements Shape {
  constructor(
    private width: number,
    private height: number
  ) {}

  getArea(): number {
    return this.width * this.height;
  }
}

class Square implements Shape {
  constructor(private side: number) {}

  getArea(): number {
    return this.side * this.side;
  }
}

// 两者都可以正确替换 Shape
function printArea(shape: Shape): void {
  console.log(shape.getArea());
}

printArea(new Rectangle(5, 4)); // 20
printArea(new Square(5));       // 25
```

### I - 接口隔离原则（ISP）

```typescript
// Interface Segregation Principle
// 客户端不应该被迫依赖它不使用的方法

// ❌ 违反 ISP：胖接口
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Human implements Worker {
  work(): void { /* ... */ }
  eat(): void { /* ... */ }
  sleep(): void { /* ... */ }
}

class Robot implements Worker {
  work(): void { /* ... */ }
  eat(): void { throw new Error('Robots do not eat'); }  // 强制实现不需要的方法
  sleep(): void { throw new Error('Robots do not sleep'); }
}

// ✅ 遵循 ISP：拆分接口

interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

interface Sleepable {
  sleep(): void;
}

class Human implements Workable, Eatable, Sleepable {
  work(): void { /* ... */ }
  eat(): void { /* ... */ }
  sleep(): void { /* ... */ }
}

class Robot implements Workable {
  work(): void { /* ... */ }
  // 只实现需要的接口
}

// 实际应用示例
// ❌ 胖仓储接口
interface Repository<T> {
  findById(id: string): Promise<T>;
  findAll(): Promise<T[]>;
  save(entity: T): Promise<void>;
  delete(id: string): Promise<void>;
  findByEmail(email: string): Promise<T>;  // 不是所有实体都有 email
  findByStatus(status: string): Promise<T[]>;  // 不是所有实体都有 status
}

// ✅ 分离的仓储接口
interface ReadRepository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
}

interface WriteRepository<T> {
  save(entity: T): Promise<void>;
  delete(id: string): Promise<void>;
}

interface UserRepository extends ReadRepository<User>, WriteRepository<User> {
  findByEmail(email: string): Promise<User | null>;
}

interface OrderRepository extends ReadRepository<Order>, WriteRepository<Order> {
  findByStatus(status: OrderStatus): Promise<Order[]>;
  findByCustomerId(customerId: string): Promise<Order[]>;
}
```

### D - 依赖倒置原则（DIP）

```typescript
// Dependency Inversion Principle
// 高层模块不应该依赖低层模块，两者都应该依赖抽象

// ❌ 违反 DIP：高层直接依赖低层
class MySQLUserRepository {
  async findById(id: string): Promise<User> {
    return mysql.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}

class UserService {
  private repository = new MySQLUserRepository(); // 直接依赖具体实现

  async getUser(id: string): Promise<User> {
    return this.repository.findById(id);
  }
}

// ✅ 遵循 DIP：依赖抽象

// 抽象（接口）
interface UserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
}

// 低层实现
class MySQLUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    const row = await mysql.query('SELECT * FROM users WHERE id = ?', [id]);
    return row ? this.mapToUser(row) : null;
  }

  async save(user: User): Promise<void> {
    await mysql.query('INSERT INTO users ...', [user]);
  }
}

class MongoUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    return this.collection.findOne({ _id: id });
  }

  async save(user: User): Promise<void> {
    await this.collection.insertOne(user);
  }
}

// 高层依赖抽象
class UserService {
  constructor(private repository: UserRepository) {} // 依赖接口

  async getUser(id: string): Promise<User> {
    const user = await this.repository.findById(id);
    if (!user) throw new UserNotFoundError();
    return user;
  }
}

// 依赖注入
const mysqlRepo = new MySQLUserRepository();
const mongoRepo = new MongoUserRepository();

const serviceWithMySQL = new UserService(mysqlRepo);
const serviceWithMongo = new UserService(mongoRepo);

// 测试时可以注入 Mock
class MockUserRepository implements UserRepository {
  private users: User[] = [];

  async findById(id: string): Promise<User | null> {
    return this.users.find(u => u.id === id) || null;
  }

  async save(user: User): Promise<void> {
    this.users.push(user);
  }
}

const testService = new UserService(new MockUserRepository());
```

---

## 其他设计原则

### DRY（Don't Repeat Yourself）

```typescript
// 不要重复自己

// ❌ 违反 DRY
class OrderService {
  async createOrder(data: any) {
    // 验证邮箱
    if (!data.email.includes('@')) {
      throw new Error('Invalid email');
    }
    // ...
  }
}

class UserService {
  async createUser(data: any) {
    // 重复的验证逻辑
    if (!data.email.includes('@')) {
      throw new Error('Invalid email');
    }
    // ...
  }
}

// ✅ 遵循 DRY
class Email {
  constructor(public readonly value: string) {
    if (!this.isValid(value)) {
      throw new InvalidEmailError();
    }
  }

  private isValid(email: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }
}

class OrderService {
  async createOrder(data: any) {
    const email = new Email(data.email); // 复用验证逻辑
    // ...
  }
}

class UserService {
  async createUser(data: any) {
    const email = new Email(data.email); // 复用验证逻辑
    // ...
  }
}
```

### KISS（Keep It Simple, Stupid）

```typescript
// 保持简单

// ❌ 过度复杂
class UserValidator {
  private static instance: UserValidator;
  private strategies: Map<string, ValidationStrategy>;

  private constructor() {
    this.strategies = new Map();
  }

  static getInstance(): UserValidator {
    if (!UserValidator.instance) {
      UserValidator.instance = new UserValidator();
    }
    return UserValidator.instance;
  }

  registerStrategy(name: string, strategy: ValidationStrategy): void {
    this.strategies.set(name, strategy);
  }

  validate(user: User): ValidationResult {
    const results: ValidationError[] = [];
    this.strategies.forEach(strategy => {
      const result = strategy.validate(user);
      if (!result.isValid) {
        results.push(...result.errors);
      }
    });
    return new ValidationResult(results);
  }
}

// ✅ 保持简单
function validateUser(user: User): string[] {
  const errors: string[] = [];

  if (!user.name || user.name.length < 2) {
    errors.push('Name must be at least 2 characters');
  }

  if (!user.email || !user.email.includes('@')) {
    errors.push('Invalid email');
  }

  return errors;
}
```

### YAGNI（You Aren't Gonna Need It）

```typescript
// 你不会需要它

// ❌ 过度设计（假设未来需求）
interface DatabaseAdapter {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  query(sql: string): Promise<any>;
  transaction(fn: () => Promise<any>): Promise<any>;
  backup(): Promise<void>;          // 现在不需要
  restore(): Promise<void>;         // 现在不需要
  replicate(): Promise<void>;       // 现在不需要
  sharding(): Promise<void>;        // 现在不需要
  clustering(): Promise<void>;      // 现在不需要
}

// ✅ 只实现当前需要的
interface DatabaseAdapter {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  query(sql: string): Promise<any>;
  transaction(fn: () => Promise<any>): Promise<any>;
}

// 当真正需要时再添加
```

### 高内聚低耦合

```typescript
// 高内聚：相关功能放在一起
// 低耦合：减少模块间依赖

// ❌ 低内聚：用户相关逻辑分散各处
// user-validation.ts
export function validateUser(user: User) { /* ... */ }

// user-email.ts
export function sendUserEmail(user: User) { /* ... */ }

// user-db.ts
export function saveUser(user: User) { /* ... */ }

// 使用时需要引入多个模块
import { validateUser } from './user-validation';
import { sendUserEmail } from './user-email';
import { saveUser } from './user-db';

// ✅ 高内聚：用户相关功能放在一起
// user/
// ├── User.ts          (实体)
// ├── UserService.ts   (业务逻辑)
// ├── UserRepository.ts(持久化)
// └── index.ts         (模块导出)

class UserService {
  constructor(
    private repository: UserRepository,
    private emailService: EmailService
  ) {}

  async createUser(data: CreateUserDto): Promise<User> {
    const user = User.create(data);
    await this.repository.save(user);
    await this.emailService.sendWelcome(user);
    return user;
  }
}

// ❌ 高耦合：直接依赖具体实现
class OrderService {
  private userService = new UserService();
  private productService = new ProductService();
  private paymentService = new PaymentService();
  private emailService = new EmailService();
  // ...
}

// ✅ 低耦合：依赖注入接口
class OrderService {
  constructor(
    private userRepository: UserRepository,        // 接口
    private productRepository: ProductRepository,  // 接口
    private paymentGateway: PaymentGateway,       // 接口
    private notificationService: NotificationService // 接口
  ) {}
}
```

### 组合优于继承

```typescript
// 组合比继承更灵活

// ❌ 继承的问题
class Animal {
  eat(): void { /* ... */ }
  sleep(): void { /* ... */ }
}

class Bird extends Animal {
  fly(): void { /* ... */ }
}

class Penguin extends Bird {
  fly(): void {
    throw new Error('Penguins cannot fly'); // 违反 LSP
  }
}

// ✅ 使用组合
interface Eater {
  eat(): void;
}

interface Sleeper {
  sleep(): void;
}

interface Flyer {
  fly(): void;
}

class Bird implements Eater, Sleeper, Flyer {
  constructor(
    private eatingBehavior: EatingBehavior,
    private sleepingBehavior: SleepingBehavior,
    private flyingBehavior: FlyingBehavior
  ) {}

  eat(): void { this.eatingBehavior.eat(); }
  sleep(): void { this.sleepingBehavior.sleep(); }
  fly(): void { this.flyingBehavior.fly(); }
}

class Penguin implements Eater, Sleeper {
  constructor(
    private eatingBehavior: EatingBehavior,
    private sleepingBehavior: SleepingBehavior
    // 不需要 FlyingBehavior
  ) {}

  eat(): void { this.eatingBehavior.eat(); }
  sleep(): void { this.sleepingBehavior.sleep(); }
  // 不需要 fly 方法
}
```

---

## 设计模式

### 创建型模式

```typescript
// 1. 工厂模式
interface Notification {
  send(message: string): void;
}

class EmailNotification implements Notification {
  send(message: string): void {
    console.log(`Sending email: ${message}`);
  }
}

class SMSNotification implements Notification {
  send(message: string): void {
    console.log(`Sending SMS: ${message}`);
  }
}

class PushNotification implements Notification {
  send(message: string): void {
    console.log(`Sending push: ${message}`);
  }
}

class NotificationFactory {
  create(type: 'email' | 'sms' | 'push'): Notification {
    switch (type) {
      case 'email': return new EmailNotification();
      case 'sms': return new SMSNotification();
      case 'push': return new PushNotification();
      default: throw new Error(`Unknown notification type: ${type}`);
    }
  }
}

// 2. 单例模式
class Database {
  private static instance: Database;
  private connection: any;

  private constructor() {
    this.connection = this.connect();
  }

  static getInstance(): Database {
    if (!Database.instance) {
      Database.instance = new Database();
    }
    return Database.instance;
  }

  private connect(): any {
    // 建立数据库连接
  }
}

// 3. 建造者模式
class QueryBuilder {
  private query: string[] = [];
  private params: any[] = [];

  select(...columns: string[]): this {
    this.query.push(`SELECT ${columns.join(', ')}`);
    return this;
  }

  from(table: string): this {
    this.query.push(`FROM ${table}`);
    return this;
  }

  where(condition: string, ...params: any[]): this {
    this.query.push(`WHERE ${condition}`);
    this.params.push(...params);
    return this;
  }

  orderBy(column: string, direction: 'ASC' | 'DESC' = 'ASC'): this {
    this.query.push(`ORDER BY ${column} ${direction}`);
    return this;
  }

  limit(count: number): this {
    this.query.push(`LIMIT ${count}`);
    return this;
  }

  build(): { sql: string; params: any[] } {
    return {
      sql: this.query.join(' '),
      params: this.params
    };
  }
}

// 使用
const { sql, params } = new QueryBuilder()
  .select('id', 'name', 'email')
  .from('users')
  .where('status = ?', 'active')
  .orderBy('created_at', 'DESC')
  .limit(10)
  .build();
```

### 结构型模式

```typescript
// 1. 适配器模式
// 旧的支付接口
interface OldPaymentGateway {
  makePayment(amount: number, currency: string): boolean;
}

// 新的支付接口
interface PaymentGateway {
  pay(money: Money): Promise<PaymentResult>;
}

// 适配器
class PaymentGatewayAdapter implements PaymentGateway {
  constructor(private oldGateway: OldPaymentGateway) {}

  async pay(money: Money): Promise<PaymentResult> {
    const success = this.oldGateway.makePayment(
      money.amount,
      money.currency.code
    );
    return {
      success,
      transactionId: success ? crypto.randomUUID() : null
    };
  }
}

// 2. 装饰器模式
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(message);
  }
}

// 装饰器：添加时间戳
class TimestampLogger implements Logger {
  constructor(private logger: Logger) {}

  log(message: string): void {
    this.logger.log(`[${new Date().toISOString()}] ${message}`);
  }
}

// 装饰器：添加日志级别
class LevelLogger implements Logger {
  constructor(
    private logger: Logger,
    private level: string
  ) {}

  log(message: string): void {
    this.logger.log(`[${this.level}] ${message}`);
  }
}

// 组合装饰器
const logger = new TimestampLogger(
  new LevelLogger(
    new ConsoleLogger(),
    'INFO'
  )
);
logger.log('Hello'); // [2024-01-01T00:00:00.000Z] [INFO] Hello

// 3. 代理模式
interface UserService {
  getUser(id: string): Promise<User>;
}

class RealUserService implements UserService {
  async getUser(id: string): Promise<User> {
    // 实际的数据库查询
    return db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
}

// 缓存代理
class CachedUserService implements UserService {
  private cache = new Map<string, User>();

  constructor(private realService: UserService) {}

  async getUser(id: string): Promise<User> {
    if (this.cache.has(id)) {
      return this.cache.get(id)!;
    }

    const user = await this.realService.getUser(id);
    this.cache.set(id, user);
    return user;
  }
}
```

### 行为型模式

```typescript
// 1. 策略模式
interface PricingStrategy {
  calculate(basePrice: number): number;
}

class RegularPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice;
  }
}

class VIPPricing implements PricingStrategy {
  calculate(basePrice: number): number {
    return basePrice * 0.9; // 9折
  }
}

class PromotionPricing implements PricingStrategy {
  constructor(private discount: number) {}

  calculate(basePrice: number): number {
    return basePrice * (1 - this.discount);
  }
}

class PriceCalculator {
  constructor(private strategy: PricingStrategy) {}

  setStrategy(strategy: PricingStrategy): void {
    this.strategy = strategy;
  }

  calculate(basePrice: number): number {
    return this.strategy.calculate(basePrice);
  }
}

// 2. 观察者模式
interface Observer {
  update(event: any): void;
}

class EventEmitter {
  private observers: Map<string, Observer[]> = new Map();

  subscribe(event: string, observer: Observer): void {
    if (!this.observers.has(event)) {
      this.observers.set(event, []);
    }
    this.observers.get(event)!.push(observer);
  }

  unsubscribe(event: string, observer: Observer): void {
    const observers = this.observers.get(event);
    if (observers) {
      const index = observers.indexOf(observer);
      if (index > -1) {
        observers.splice(index, 1);
      }
    }
  }

  emit(event: string, data: any): void {
    const observers = this.observers.get(event);
    if (observers) {
      observers.forEach(observer => observer.update(data));
    }
  }
}

// 3. 责任链模式
interface Handler {
  setNext(handler: Handler): Handler;
  handle(request: Request): Response | null;
}

abstract class BaseHandler implements Handler {
  private nextHandler: Handler | null = null;

  setNext(handler: Handler): Handler {
    this.nextHandler = handler;
    return handler;
  }

  handle(request: Request): Response | null {
    if (this.nextHandler) {
      return this.nextHandler.handle(request);
    }
    return null;
  }
}

class AuthenticationHandler extends BaseHandler {
  handle(request: Request): Response | null {
    if (!request.headers.authorization) {
      return { status: 401, body: 'Unauthorized' };
    }
    return super.handle(request);
  }
}

class RateLimitHandler extends BaseHandler {
  handle(request: Request): Response | null {
    if (this.isRateLimited(request.ip)) {
      return { status: 429, body: 'Too Many Requests' };
    }
    return super.handle(request);
  }
}

class ValidationHandler extends BaseHandler {
  handle(request: Request): Response | null {
    if (!this.isValid(request.body)) {
      return { status: 400, body: 'Bad Request' };
    }
    return super.handle(request);
  }
}

// 使用
const handler = new AuthenticationHandler();
handler
  .setNext(new RateLimitHandler())
  .setNext(new ValidationHandler());

const response = handler.handle(request);
```

---

## 架构决策

### ADR（架构决策记录）

```markdown
# ADR 001: 选择 PostgreSQL 作为主数据库

## 状态
已接受

## 上下文
我们需要选择一个主数据库来存储核心业务数据。
候选方案：MySQL, PostgreSQL, MongoDB

## 决策
选择 PostgreSQL

## 理由
1. 支持复杂查询和事务
2. JSON 支持（JSONB）满足灵活数据需求
3. 强大的扩展性（分区、复制）
4. 团队有 PostgreSQL 经验
5. 开源、社区活跃

## 后果
- 正面：强一致性、丰富的功能
- 负面：运维复杂度高于 MySQL
- 风险：需要 DBA 支持

## 参考
- [PostgreSQL vs MySQL](...)
- [团队数据库评估报告](...)
```

### 技术选型原则

```typescript
const selectionCriteria = {
  // 1. 团队因素
  团队: {
    技术栈熟悉度: '团队是否有经验',
    学习曲线: '学习成本',
    招聘难度: '是否容易招到人'
  },

  // 2. 技术因素
  技术: {
    成熟度: '是否经过生产验证',
    生态系统: '库和工具是否丰富',
    性能: '是否满足性能需求',
    可扩展性: '是否支持扩展'
  },

  // 3. 业务因素
  业务: {
    功能匹配: '是否满足业务需求',
    时间约束: '是否能按时交付',
    成本: '许可、运维成本'
  },

  // 4. 运维因素
  运维: {
    监控: '是否容易监控',
    部署: '是否容易部署',
    故障恢复: '是否容易恢复'
  }
};
```

---

## 常见面试题

### 1. 什么是 SOLID 原则？

| 原则 | 全称 | 含义 |
|------|------|------|
| **S** | 单一职责 | 一个类只有一个职责 |
| **O** | 开闭原则 | 对扩展开放，对修改关闭 |
| **L** | 里氏替换 | 子类可以替换父类 |
| **I** | 接口隔离 | 接口应该小而专一 |
| **D** | 依赖倒置 | 依赖抽象而非具体 |

### 2. DRY vs WET？

- **DRY**：Don't Repeat Yourself，不要重复
- **WET**：Write Everything Twice，写两遍才抽象

**建议**：Rule of Three，重复三次再抽象

### 3. 组合和继承的区别？

| 特性 | 继承 | 组合 |
|------|------|------|
| **关系** | is-a | has-a |
| **耦合** | 紧耦合 | 松耦合 |
| **灵活性** | 低 | 高 |
| **复用** | 白盒复用 | 黑盒复用 |

**建议**：优先使用组合

### 4. 什么时候用单例模式？

**适用场景**：
- 数据库连接池
- 配置管理
- 日志服务
- 缓存

**注意**：
- 单例使测试困难
- 可能引入全局状态
- 考虑依赖注入替代

### 5. 如何评估架构决策？

**评估维度**：
1. 满足当前需求
2. 支持未来扩展
3. 团队可维护
4. 成本可接受
5. 风险可控

---

## 总结

### 设计原则速查

| 原则 | 一句话 |
|------|--------|
| **SRP** | 一个类一个职责 |
| **OCP** | 扩展不修改 |
| **LSP** | 子类可替换父类 |
| **ISP** | 小接口好过大接口 |
| **DIP** | 依赖抽象 |
| **DRY** | 不重复 |
| **KISS** | 保持简单 |
| **YAGNI** | 不要过度设计 |

### 实践检查清单

- [ ] 理解 SOLID 原则
- [ ] 掌握常用设计模式
- [ ] 能识别代码坏味道
- [ ] 能进行合理重构
- [ ] 会记录架构决策

---

**上一篇**：[微服务深入](./04-microservices.md)  
**返回目录**：[架构设计](./README.md)

