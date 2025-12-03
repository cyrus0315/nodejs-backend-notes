# 领域驱动设计（DDD）

DDD 是处理复杂业务领域的架构方法论。本文深入讲解 DDD 的核心概念和实践。

## 目录
- [DDD 核心概念](#ddd-核心概念)
- [战略设计](#战略设计)
- [战术设计](#战术设计)
- [实体与值对象](#实体与值对象)
- [聚合](#聚合)
- [领域服务](#领域服务)
- [领域事件](#领域事件)
- [仓储模式](#仓储模式)
- [实战案例](#实战案例)
- [面试题](#常见面试题)

---

## DDD 核心概念

### 什么是 DDD

```
DDD（Domain-Driven Design）领域驱动设计

核心思想：
1. 以领域为中心进行软件设计
2. 让代码反映业务语言（通用语言）
3. 通过限界上下文划分系统边界
4. 使用战术模式实现领域模型

DDD 分为两个层面：
- 战略设计：划分领域边界、定义上下文关系
- 战术设计：实体、值对象、聚合、领域服务等
```

### 通用语言

```typescript
// 通用语言（Ubiquitous Language）：团队共同使用的业务术语

// ❌ 技术语言
class User {
  setStatus(status: number) {}  // status: 1=active, 2=inactive
  addRecord(record: Record) {}   // 什么记录？
}

// ✅ 领域语言
class Customer {                 // 客户，而非 User
  activate(): void {}           // 激活客户
  suspend(): void {}            // 暂停客户
  placeOrder(order: Order) {}   // 下单
}

// 代码应该使用业务人员能理解的术语
// 这样开发人员和业务人员可以使用同一套语言沟通
```

---

## 战略设计

### 限界上下文

```
限界上下文（Bounded Context）：明确的语义边界

电商系统的限界上下文：

┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  销售上下文     │  │  库存上下文     │  │  物流上下文     │
│               │  │               │  │               │
│  - 订单       │  │  - 库存       │  │  - 发货       │
│  - 客户       │  │  - 预留       │  │  - 配送       │
│  - 商品(售卖) │  │  - 商品(存储) │  │  - 包裹       │
└────────────────┘  └────────────────┘  └────────────────┘

注意：同一个概念（如"商品"）在不同上下文中有不同含义
- 销售上下文：商品的价格、描述、分类
- 库存上下文：商品的数量、位置、批次
- 物流上下文：商品的重量、体积、包装
```

```typescript
// 销售上下文中的商品
// sales/domain/Product.ts
export class Product {
  constructor(
    public readonly id: string,
    public name: string,
    public description: string,
    public price: Money,
    public category: Category
  ) {}

  applyDiscount(discount: Discount): void {
    this.price = this.price.subtract(discount.amount);
  }
}

// 库存上下文中的商品
// inventory/domain/StockItem.ts
export class StockItem {
  constructor(
    public readonly productId: string,
    public quantity: number,
    public location: WarehouseLocation,
    public batchNumber: string
  ) {}

  reserve(amount: number): void {
    if (amount > this.quantity) {
      throw new InsufficientStockError();
    }
    this.quantity -= amount;
  }

  restock(amount: number): void {
    this.quantity += amount;
  }
}
```

### 上下文映射

```
上下文之间的关系模式：

1. 共享内核（Shared Kernel）
   两个上下文共享一部分模型
   
2. 客户-供应商（Customer-Supplier）
   下游上下文依赖上游上下文
   
3. 遵奉者（Conformist）
   下游完全接受上游的模型
   
4. 防腐层（Anti-Corruption Layer）
   下游通过翻译层隔离上游
   
5. 开放主机服务（Open Host Service）
   提供公开的 API 供其他上下文调用
   
6. 发布语言（Published Language）
   使用通用的数据格式（如 JSON Schema）
```

```typescript
// 防腐层示例：销售上下文调用库存上下文

// 1. 库存上下文提供的 API（开放主机服务）
// inventory-service/api/InventoryController.ts
@Controller('inventory')
export class InventoryController {
  @Post('reserve')
  async reserve(@Body() dto: ReserveDto) {
    // 返回库存上下文的数据结构
    return this.inventoryService.reserve(dto);
  }
}

// 2. 销售上下文的防腐层
// sales/infrastructure/acl/InventoryACL.ts
export class InventoryACL {
  constructor(private httpClient: HttpClient) {}

  async reserveStock(items: OrderItem[]): Promise<ReservationResult> {
    // 转换为库存上下文需要的格式
    const dto = items.map(item => ({
      productId: item.productId,
      quantity: item.quantity
    }));

    // 调用库存服务
    const response = await this.httpClient.post('/inventory/reserve', dto);

    // 将响应转换为销售上下文的领域对象
    return new ReservationResult(
      response.reservationId,
      response.success,
      response.failedItems?.map(item => new FailedReservation(item.productId, item.reason))
    );
  }
}

// 3. 在销售上下文中使用
// sales/application/CreateOrderUseCase.ts
export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private inventoryACL: InventoryACL  // 通过防腐层访问
  ) {}

  async execute(input: CreateOrderInput): Promise<Order> {
    const order = Order.create(input);
    
    // 通过防腐层预留库存
    const reservation = await this.inventoryACL.reserveStock(order.items);
    
    if (!reservation.success) {
      throw new InsufficientStockError(reservation.failedItems);
    }
    
    await this.orderRepository.save(order);
    return order;
  }
}
```

---

## 战术设计

### DDD 构建块

```
┌─────────────────────────────────────────────────────────┐
│                    聚合 (Aggregate)                      │
│  ┌─────────────┐                                       │
│  │  聚合根      │ ←── 唯一入口点                        │
│  │(Aggregate   │                                       │
│  │   Root)     │                                       │
│  └──────┬──────┘                                       │
│         │                                              │
│    ┌────┴────┐                                         │
│    ▼         ▼                                         │
│  ┌─────┐  ┌─────┐                                     │
│  │实体 │  │值对象│                                     │
│  │Entity│  │Value │                                     │
│  │     │  │Object│                                     │
│  └─────┘  └─────┘                                     │
└─────────────────────────────────────────────────────────┘

其他构建块：
- 领域服务（Domain Service）
- 领域事件（Domain Event）
- 仓储（Repository）
- 工厂（Factory）
```

---

## 实体与值对象

### 实体（Entity）

```typescript
// 实体特点：
// 1. 有唯一标识
// 2. 标识不变，属性可变
// 3. 有生命周期

// 订单实体
export class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus = OrderStatus.DRAFT;

  constructor(
    public readonly id: OrderId,           // 唯一标识
    public readonly customerId: CustomerId,
    public readonly createdAt: Date
  ) {}

  // 业务行为
  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.ensureDraft();
    
    const existing = this._items.find(item => item.productId.equals(productId));
    if (existing) {
      existing.increaseQuantity(quantity);
    } else {
      this._items.push(new OrderItem(productId, quantity, price));
    }
  }

  removeItem(productId: ProductId): void {
    this.ensureDraft();
    this._items = this._items.filter(item => !item.productId.equals(productId));
  }

  submit(): void {
    this.ensureDraft();
    if (this._items.length === 0) {
      throw new EmptyOrderError();
    }
    this._status = OrderStatus.SUBMITTED;
  }

  confirm(): void {
    if (this._status !== OrderStatus.SUBMITTED) {
      throw new InvalidOrderStateError();
    }
    this._status = OrderStatus.CONFIRMED;
  }

  // 计算属性
  get total(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero()
    );
  }

  get itemCount(): number {
    return this._items.length;
  }

  get items(): ReadonlyArray<OrderItem> {
    return [...this._items];
  }

  get status(): OrderStatus {
    return this._status;
  }

  // 私有方法
  private ensureDraft(): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new OrderNotDraftError();
    }
  }

  // 实体相等性基于 ID
  equals(other: Order): boolean {
    return this.id.equals(other.id);
  }
}
```

### 值对象（Value Object）

```typescript
// 值对象特点：
// 1. 无唯一标识
// 2. 不可变
// 3. 相等性基于所有属性

// 金额值对象
export class Money {
  private constructor(
    private readonly amount: number,
    private readonly currency: Currency
  ) {
    if (amount < 0) {
      throw new NegativeAmountError();
    }
  }

  static of(amount: number, currency: Currency = Currency.USD): Money {
    return new Money(amount, currency);
  }

  static zero(currency: Currency = Currency.USD): Money {
    return new Money(0, currency);
  }

  add(other: Money): Money {
    this.ensureSameCurrency(other);
    return new Money(this.amount + other.amount, this.currency);
  }

  subtract(other: Money): Money {
    this.ensureSameCurrency(other);
    const result = this.amount - other.amount;
    if (result < 0) {
      throw new InsufficientFundsError();
    }
    return new Money(result, this.currency);
  }

  multiply(factor: number): Money {
    return new Money(this.amount * factor, this.currency);
  }

  // 值对象相等性基于所有属性
  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }

  toString(): string {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }

  private ensureSameCurrency(other: Money): void {
    if (this.currency !== other.currency) {
      throw new CurrencyMismatchError();
    }
  }
}

// 地址值对象
export class Address {
  constructor(
    public readonly street: string,
    public readonly city: string,
    public readonly state: string,
    public readonly zipCode: string,
    public readonly country: string
  ) {
    this.validate();
  }

  private validate(): void {
    if (!this.street || !this.city || !this.country) {
      throw new InvalidAddressError();
    }
  }

  equals(other: Address): boolean {
    return (
      this.street === other.street &&
      this.city === other.city &&
      this.state === other.state &&
      this.zipCode === other.zipCode &&
      this.country === other.country
    );
  }

  // 返回新的值对象
  withStreet(street: string): Address {
    return new Address(street, this.city, this.state, this.zipCode, this.country);
  }

  format(): string {
    return `${this.street}, ${this.city}, ${this.state} ${this.zipCode}, ${this.country}`;
  }
}

// ID 值对象
export class OrderId {
  constructor(public readonly value: string) {
    if (!value || value.length === 0) {
      throw new InvalidIdError();
    }
  }

  static generate(): OrderId {
    return new OrderId(crypto.randomUUID());
  }

  equals(other: OrderId): boolean {
    return this.value === other.value;
  }

  toString(): string {
    return this.value;
  }
}
```

---

## 聚合

### 聚合设计原则

```typescript
// 聚合（Aggregate）：
// 1. 定义清晰的边界
// 2. 通过聚合根访问内部对象
// 3. 保持一致性边界
// 4. 设计小聚合

// 订单聚合
//
// Order (聚合根)
//   │
//   ├── OrderItem (实体/值对象)
//   │     └── ProductId (值对象)
//   │     └── Quantity (值对象)
//   │     └── Money (值对象)
//   │
//   ├── ShippingAddress (值对象)
//   │
//   └── OrderStatus (值对象/枚举)

export class Order {  // 聚合根
  private _items: OrderItem[] = [];
  private _shippingAddress: Address | null = null;
  private _status: OrderStatus = OrderStatus.DRAFT;

  constructor(
    public readonly id: OrderId,
    public readonly customerId: CustomerId,
    public readonly createdAt: Date
  ) {}

  // 所有修改都通过聚合根
  addItem(productId: ProductId, quantity: Quantity, price: Money): void {
    this.ensureModifiable();
    // ...
  }

  setShippingAddress(address: Address): void {
    this.ensureModifiable();
    this._shippingAddress = address;
  }

  submit(): void {
    if (!this._shippingAddress) {
      throw new MissingShippingAddressError();
    }
    if (this._items.length === 0) {
      throw new EmptyOrderError();
    }
    this._status = OrderStatus.SUBMITTED;
  }

  // 内部实体不暴露可变引用
  get items(): ReadonlyArray<OrderItem> {
    return Object.freeze([...this._items]);
  }

  private ensureModifiable(): void {
    if (this._status !== OrderStatus.DRAFT) {
      throw new OrderNotModifiableError();
    }
  }
}

// 聚合之间通过 ID 引用
export class Order {
  constructor(
    public readonly id: OrderId,
    public readonly customerId: CustomerId,  // 引用 Customer 聚合的 ID
    // 不直接引用 Customer 对象
  ) {}
}
```

### 聚合设计原则

```typescript
// 原则 1：设计小聚合
// ❌ 大聚合
class Customer {
  orders: Order[];        // 可能有很多订单
  addresses: Address[];
  paymentMethods: PaymentMethod[];
}

// ✅ 小聚合
class Customer {
  defaultAddressId: AddressId;
  defaultPaymentMethodId: PaymentMethodId;
}

class Order {
  customerId: CustomerId;  // 通过 ID 引用
}

// 原则 2：通过唯一标识引用其他聚合
// ❌ 直接引用
class Order {
  customer: Customer;  // 直接引用对象
}

// ✅ ID 引用
class Order {
  customerId: CustomerId;  // 通过 ID 引用
}

// 原则 3：一个事务只修改一个聚合
// ❌ 一个事务修改多个聚合
async createOrder(orderData, customerId) {
  const order = new Order(orderData);
  const customer = await customerRepo.findById(customerId);
  customer.addOrder(order);  // 修改 Customer 聚合
  await orderRepo.save(order);
  await customerRepo.save(customer);
}

// ✅ 只修改一个聚合，通过领域事件通知其他聚合
async createOrder(orderData, customerId) {
  const order = new Order(orderData, customerId);
  await orderRepo.save(order);
  await eventBus.publish(new OrderCreatedEvent(order.id, customerId));
}
```

---

## 领域服务

### 什么时候使用领域服务

```typescript
// 领域服务用于：
// 1. 业务逻辑不属于任何实体
// 2. 需要多个聚合协作
// 3. 无状态的领域操作

// 示例：转账服务
export class TransferService {
  async transfer(
    fromAccountId: AccountId,
    toAccountId: AccountId,
    amount: Money
  ): Promise<void> {
    // 领域服务协调多个聚合
    const fromAccount = await this.accountRepository.findById(fromAccountId);
    const toAccount = await this.accountRepository.findById(toAccountId);

    if (!fromAccount || !toAccount) {
      throw new AccountNotFoundError();
    }

    // 执行领域逻辑
    fromAccount.withdraw(amount);
    toAccount.deposit(amount);

    // 保存（理想情况下在一个事务中）
    await this.accountRepository.save(fromAccount);
    await this.accountRepository.save(toAccount);
  }
}

// 示例：订单价格计算服务
export class OrderPricingService {
  constructor(
    private discountPolicy: DiscountPolicy,
    private taxCalculator: TaxCalculator
  ) {}

  calculateTotal(order: Order, customer: Customer): OrderPricing {
    const subtotal = order.items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero()
    );

    // 应用折扣
    const discount = this.discountPolicy.calculate(order, customer);
    const afterDiscount = subtotal.subtract(discount);

    // 计算税费
    const tax = this.taxCalculator.calculate(afterDiscount, order.shippingAddress);

    // 计算总价
    const total = afterDiscount.add(tax);

    return new OrderPricing(subtotal, discount, tax, total);
  }
}
```

### 领域服务 vs 应用服务

```typescript
// 领域服务：封装领域逻辑
// domain/services/TransferService.ts
export class TransferService {
  transfer(from: Account, to: Account, amount: Money): void {
    // 纯领域逻辑，无基础设施依赖
    from.withdraw(amount);
    to.deposit(amount);
  }
}

// 应用服务：编排用例
// application/services/TransferApplicationService.ts
export class TransferApplicationService {
  constructor(
    private accountRepository: AccountRepository,
    private transferService: TransferService,
    private eventBus: EventBus
  ) {}

  async execute(command: TransferCommand): Promise<void> {
    // 1. 获取聚合
    const from = await this.accountRepository.findById(command.fromAccountId);
    const to = await this.accountRepository.findById(command.toAccountId);

    // 2. 执行领域服务
    this.transferService.transfer(from, to, command.amount);

    // 3. 持久化
    await this.accountRepository.save(from);
    await this.accountRepository.save(to);

    // 4. 发布事件
    await this.eventBus.publish(new TransferCompletedEvent(
      command.fromAccountId,
      command.toAccountId,
      command.amount
    ));
  }
}
```

---

## 领域事件

### 领域事件定义

```typescript
// 领域事件（Domain Event）：记录领域中发生的重要事件

// 基类
export abstract class DomainEvent {
  public readonly occurredAt: Date = new Date();
  public readonly eventId: string = crypto.randomUUID();
  
  abstract get eventType(): string;
}

// 具体事件
export class OrderCreatedEvent extends DomainEvent {
  get eventType(): string {
    return 'order.created';
  }

  constructor(
    public readonly orderId: OrderId,
    public readonly customerId: CustomerId,
    public readonly items: ReadonlyArray<OrderItemData>,
    public readonly total: Money
  ) {
    super();
  }
}

export class OrderConfirmedEvent extends DomainEvent {
  get eventType(): string {
    return 'order.confirmed';
  }

  constructor(
    public readonly orderId: OrderId,
    public readonly confirmedAt: Date
  ) {
    super();
  }
}

export class PaymentReceivedEvent extends DomainEvent {
  get eventType(): string {
    return 'payment.received';
  }

  constructor(
    public readonly orderId: OrderId,
    public readonly paymentId: PaymentId,
    public readonly amount: Money
  ) {
    super();
  }
}
```

### 聚合发布事件

```typescript
// 聚合根收集领域事件
export abstract class AggregateRoot {
  private _domainEvents: DomainEvent[] = [];

  get domainEvents(): ReadonlyArray<DomainEvent> {
    return [...this._domainEvents];
  }

  protected addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  clearDomainEvents(): void {
    this._domainEvents = [];
  }
}

// 订单聚合
export class Order extends AggregateRoot {
  private _status: OrderStatus = OrderStatus.DRAFT;

  submit(): void {
    this.ensureDraft();
    this._status = OrderStatus.SUBMITTED;
    
    // 发布领域事件
    this.addDomainEvent(new OrderSubmittedEvent(
      this.id,
      this.customerId,
      this.items.map(item => ({
        productId: item.productId.value,
        quantity: item.quantity.value,
        price: item.price.amount
      })),
      this.total
    ));
  }

  confirm(): void {
    this._status = OrderStatus.CONFIRMED;
    
    this.addDomainEvent(new OrderConfirmedEvent(
      this.id,
      new Date()
    ));
  }
}

// 仓储保存后发布事件
export class OrderRepository {
  constructor(
    private prisma: PrismaClient,
    private eventBus: EventBus
  ) {}

  async save(order: Order): Promise<void> {
    // 保存聚合
    await this.prisma.order.upsert({
      where: { id: order.id.value },
      update: this.toPersistence(order),
      create: this.toPersistence(order)
    });

    // 发布领域事件
    for (const event of order.domainEvents) {
      await this.eventBus.publish(event);
    }

    // 清除已发布的事件
    order.clearDomainEvents();
  }
}
```

### 事件处理器

```typescript
// 事件处理器接口
export interface EventHandler<T extends DomainEvent> {
  handle(event: T): Promise<void>;
}

// 订单创建后的处理器
// 1. 发送确认邮件
export class SendOrderConfirmationEmailHandler implements EventHandler<OrderCreatedEvent> {
  constructor(private emailService: EmailService) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    await this.emailService.sendOrderConfirmation(
      event.customerId,
      event.orderId,
      event.total
    );
  }
}

// 2. 预留库存
export class ReserveInventoryHandler implements EventHandler<OrderCreatedEvent> {
  constructor(private inventoryService: InventoryService) {}

  async handle(event: OrderCreatedEvent): Promise<void> {
    for (const item of event.items) {
      await this.inventoryService.reserve(item.productId, item.quantity);
    }
  }
}

// 事件总线
export class EventBus {
  private handlers: Map<string, EventHandler<any>[]> = new Map();

  register<T extends DomainEvent>(
    eventType: string,
    handler: EventHandler<T>
  ): void {
    const handlers = this.handlers.get(eventType) || [];
    handlers.push(handler);
    this.handlers.set(eventType, handlers);
  }

  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) || [];
    await Promise.all(handlers.map(h => h.handle(event)));
  }
}
```

---

## 仓储模式

### 仓储接口

```typescript
// 仓储（Repository）：提供聚合的持久化抽象

// 通用仓储接口
export interface Repository<T extends AggregateRoot, ID> {
  findById(id: ID): Promise<T | null>;
  save(aggregate: T): Promise<void>;
  delete(id: ID): Promise<void>;
}

// 订单仓储接口
export interface OrderRepository extends Repository<Order, OrderId> {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: CustomerId): Promise<Order[]>;
  findByStatus(status: OrderStatus): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
}

// 仓储只操作聚合根
// ❌ 错误：直接操作内部实体
interface OrderItemRepository {
  findByOrderId(orderId: OrderId): Promise<OrderItem[]>;
}

// ✅ 正确：通过聚合根访问
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;  // Order 包含 items
}
```

### 仓储实现

```typescript
// Prisma 仓储实现
export class PrismaOrderRepository implements OrderRepository {
  constructor(
    private prisma: PrismaClient,
    private eventBus: EventBus
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    const data = await this.prisma.order.findUnique({
      where: { id: id.value },
      include: { items: true }
    });

    if (!data) return null;
    
    return this.toDomain(data);
  }

  async findByCustomerId(customerId: CustomerId): Promise<Order[]> {
    const data = await this.prisma.order.findMany({
      where: { customerId: customerId.value },
      include: { items: true },
      orderBy: { createdAt: 'desc' }
    });

    return data.map(d => this.toDomain(d));
  }

  async save(order: Order): Promise<void> {
    const data = this.toPersistence(order);

    await this.prisma.$transaction(async (tx) => {
      // 保存订单
      await tx.order.upsert({
        where: { id: order.id.value },
        update: {
          status: data.status,
          shippingAddress: data.shippingAddress,
          updatedAt: new Date()
        },
        create: data
      });

      // 删除旧的订单项
      await tx.orderItem.deleteMany({
        where: { orderId: order.id.value }
      });

      // 创建新的订单项
      if (data.items.length > 0) {
        await tx.orderItem.createMany({
          data: data.items.map(item => ({
            ...item,
            orderId: order.id.value
          }))
        });
      }
    });

    // 发布领域事件
    for (const event of order.domainEvents) {
      await this.eventBus.publish(event);
    }
    order.clearDomainEvents();
  }

  async delete(id: OrderId): Promise<void> {
    await this.prisma.$transaction([
      this.prisma.orderItem.deleteMany({ where: { orderId: id.value } }),
      this.prisma.order.delete({ where: { id: id.value } })
    ]);
  }

  // 领域模型 ↔ 持久化模型 转换
  private toDomain(data: any): Order {
    const order = new Order(
      new OrderId(data.id),
      new CustomerId(data.customerId),
      data.createdAt
    );

    // 恢复内部状态
    for (const item of data.items) {
      order['_items'].push(new OrderItem(
        new ProductId(item.productId),
        new Quantity(item.quantity),
        Money.of(item.price)
      ));
    }
    order['_status'] = data.status as OrderStatus;
    
    if (data.shippingAddress) {
      order['_shippingAddress'] = new Address(
        data.shippingAddress.street,
        data.shippingAddress.city,
        data.shippingAddress.state,
        data.shippingAddress.zipCode,
        data.shippingAddress.country
      );
    }

    return order;
  }

  private toPersistence(order: Order): any {
    return {
      id: order.id.value,
      customerId: order.customerId.value,
      status: order.status,
      shippingAddress: order['_shippingAddress'] ? {
        street: order['_shippingAddress'].street,
        city: order['_shippingAddress'].city,
        state: order['_shippingAddress'].state,
        zipCode: order['_shippingAddress'].zipCode,
        country: order['_shippingAddress'].country
      } : null,
      items: order.items.map(item => ({
        productId: item.productId.value,
        quantity: item.quantity.value,
        price: item.price['amount']
      })),
      createdAt: order.createdAt
    };
  }
}
```

---

## 实战案例

### 电商订单系统

```typescript
// 完整的订单限界上下文结构

// domain/
//   ├── entities/
//   │   ├── Order.ts
//   │   └── OrderItem.ts
//   ├── value-objects/
//   │   ├── OrderId.ts
//   │   ├── Money.ts
//   │   ├── Quantity.ts
//   │   └── Address.ts
//   ├── events/
//   │   ├── OrderCreatedEvent.ts
//   │   └── OrderConfirmedEvent.ts
//   ├── repositories/
//   │   └── OrderRepository.ts
//   └── services/
//       └── OrderPricingService.ts

// application/
//   ├── use-cases/
//   │   ├── CreateOrderUseCase.ts
//   │   ├── ConfirmOrderUseCase.ts
//   │   └── CancelOrderUseCase.ts
//   └── dto/
//       └── OrderDto.ts

// infrastructure/
//   ├── persistence/
//   │   └── PrismaOrderRepository.ts
//   ├── acl/
//   │   └── InventoryACL.ts
//   └── events/
//       └── KafkaEventBus.ts

// 创建订单用例
export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private customerRepository: CustomerRepository,
    private inventoryACL: InventoryACL,
    private pricingService: OrderPricingService
  ) {}

  async execute(input: CreateOrderInput): Promise<OrderDto> {
    // 1. 获取客户
    const customer = await this.customerRepository.findById(
      new CustomerId(input.customerId)
    );
    if (!customer) {
      throw new CustomerNotFoundError();
    }

    // 2. 创建订单
    const order = Order.create(
      new CustomerId(input.customerId)
    );

    // 3. 添加商品
    for (const item of input.items) {
      order.addItem(
        new ProductId(item.productId),
        new Quantity(item.quantity),
        Money.of(item.price)
      );
    }

    // 4. 设置配送地址
    order.setShippingAddress(new Address(
      input.shippingAddress.street,
      input.shippingAddress.city,
      input.shippingAddress.state,
      input.shippingAddress.zipCode,
      input.shippingAddress.country
    ));

    // 5. 计算价格
    const pricing = this.pricingService.calculateTotal(order, customer);
    order.setPricing(pricing);

    // 6. 预留库存（通过防腐层）
    const reservation = await this.inventoryACL.reserveStock(order.items);
    if (!reservation.success) {
      throw new InsufficientStockError(reservation.failedItems);
    }

    // 7. 提交订单
    order.submit();

    // 8. 保存
    await this.orderRepository.save(order);

    // 9. 返回 DTO
    return OrderDto.fromDomain(order);
  }
}
```

---

## 常见面试题

### 1. 实体和值对象的区别？

| 特性 | 实体 | 值对象 |
|------|------|--------|
| **标识** | 有唯一标识 | 无标识 |
| **可变性** | 可变 | 不可变 |
| **相等性** | 基于 ID | 基于所有属性 |
| **生命周期** | 有 | 无 |
| **示例** | Order, Customer | Money, Address |

### 2. 什么是聚合？设计原则是什么？

**聚合**：一组相关对象的集合，对外只能通过聚合根访问。

**原则**：
1. 设计小聚合
2. 通过 ID 引用其他聚合
3. 一个事务只修改一个聚合
4. 聚合根是唯一入口

### 3. 领域服务和应用服务的区别？

- **领域服务**：封装领域逻辑，无基础设施依赖
- **应用服务**：编排用例，协调领域对象和基础设施

### 4. 什么是限界上下文？

**限界上下文**：明确的语义边界，同一概念在不同上下文有不同含义。

**作用**：
- 划分系统边界
- 避免模型污染
- 支持团队分工

### 5. DDD 适用于什么场景？

**适用**：
- 复杂业务领域
- 领域专家参与
- 长期维护的项目

**不适用**：
- 简单 CRUD
- 技术驱动项目
- 短期项目

---

## 总结

### DDD 构建块总结

| 构建块 | 职责 | 示例 |
|--------|------|------|
| **实体** | 有标识的领域对象 | Order, Customer |
| **值对象** | 无标识的不可变对象 | Money, Address |
| **聚合** | 一致性边界 | Order + OrderItems |
| **仓储** | 聚合持久化 | OrderRepository |
| **领域服务** | 跨聚合领域逻辑 | TransferService |
| **领域事件** | 领域中的重要事件 | OrderCreated |

### 实践检查清单

- [ ] 理解通用语言
- [ ] 识别限界上下文
- [ ] 区分实体和值对象
- [ ] 设计合理的聚合
- [ ] 实现领域事件
- [ ] 使用仓储模式
- [ ] 识别领域服务

---

**上一篇**：[分层架构](./02-layered-architecture.md)  
**下一篇**：[微服务深入](./04-microservices.md)

