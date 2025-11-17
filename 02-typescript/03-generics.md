# TypeScript 泛型

## 泛型基础

### 什么是泛型？

泛型允许我们创建可重用的组件，这些组件可以支持多种类型而不失去类型安全。

```typescript
// 不使用泛型
function identityNumber(arg: number): number {
  return arg;
}

function identityString(arg: string): string {
  return arg;
}

// 使用 any（失去类型安全）
function identity(arg: any): any {
  return arg;
}

// ✅ 使用泛型
function identity<T>(arg: T): T {
  return arg;
}

// 使用
const num = identity<number>(42); // number
const str = identity<string>("hello"); // string

// 类型推断
const num2 = identity(42); // TypeScript 自动推断为 number
const str2 = identity("hello"); // 自动推断为 string
```

### 泛型函数

```typescript
// 基本泛型函数
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = firstElement([1, 2, 3]); // number | undefined
const str = firstElement(["a", "b", "c"]); // string | undefined

// 多个类型参数
function map<Input, Output>(
  arr: Input[],
  func: (arg: Input) => Output
): Output[] {
  return arr.map(func);
}

const numbers = [1, 2, 3];
const strings = map(numbers, (n) => n.toString()); // string[]

// 泛型函数类型
type MapFunction = <T, U>(arr: T[], func: (arg: T) => U) => U[];

const myMap: MapFunction = (arr, func) => arr.map(func);

// 调用签名泛型
interface GenericIdentityFn<T> {
  (arg: T): T;
}

const myIdentity: GenericIdentityFn<number> = (arg) => arg;
```

### 泛型约束

```typescript
// 基本约束
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(arg: T): T {
  console.log(arg.length);
  return arg;
}

logLength("hello"); // ✅ OK
logLength([1, 2, 3]); // ✅ OK
logLength({ length: 10, value: 3 }); // ✅ OK
// logLength(3); // ❌ Error: number 没有 length 属性

// 在泛型约束中使用类型参数
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = {
  name: "Alice",
  age: 25
};

const name = getProperty(person, "name"); // string
const age = getProperty(person, "age"); // number
// getProperty(person, "email"); // ❌ Error

// 多个约束
function copyFields<T extends U, U>(target: T, source: U): T {
  for (const id in source) {
    target[id] = source[id] as any;
  }
  return target;
}

// 使用类类型约束
function create<T>(c: new () => T): T {
  return new c();
}

class Person {
  name = "Alice";
}

const person = create(Person); // Person
```

## 泛型接口

```typescript
// 基本泛型接口
interface GenericIdentityFn<T> {
  (arg: T): T;
}

const myIdentity: GenericIdentityFn<number> = (x) => x;

// 泛型接口描述对象
interface Box<T> {
  value: T;
}

const numberBox: Box<number> = { value: 42 };
const stringBox: Box<string> = { value: "hello" };

// 泛型数组接口
interface Array<T> {
  length: number;
  push(...items: T[]): number;
  pop(): T | undefined;
  [n: number]: T;
}

// 复杂泛型接口
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(item: Omit<T, "id">): Promise<T>;
  update(id: string, item: Partial<T>): Promise<T>;
  delete(id: string): Promise<boolean>;
}

interface User {
  id: string;
  name: string;
  email: string;
}

const userRepository: Repository<User> = {
  async findById(id) {
    // 实现...
    return null;
  },
  async findAll() {
    return [];
  },
  async create(user) {
    // user 类型是 Omit<User, "id">
    return { id: "1", ...user };
  },
  async update(id, user) {
    // user 类型是 Partial<User>
    return {} as User;
  },
  async delete(id) {
    return true;
  }
};
```

## 泛型类

```typescript
// 基本泛型类
class GenericNumber<T> {
  zeroValue: T;
  add: (x: T, y: T) => T;
  
  constructor(zero: T, addFn: (x: T, y: T) => T) {
    this.zeroValue = zero;
    this.add = addFn;
  }
}

const numberCalc = new GenericNumber<number>(
  0,
  (x, y) => x + y
);

const stringCalc = new GenericNumber<string>(
  "",
  (x, y) => x + y
);

// 实用泛型类：Stack
class Stack<T> {
  private items: T[] = [];
  
  push(item: T): void {
    this.items.push(item);
  }
  
  pop(): T | undefined {
    return this.items.pop();
  }
  
  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }
  
  isEmpty(): boolean {
    return this.items.length === 0;
  }
  
  size(): number {
    return this.items.length;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
console.log(numberStack.pop()); // 2

// 泛型类约束
class Animal {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

class Dog extends Animal {
  bark() {
    console.log("Woof!");
  }
}

class Zoo<T extends Animal> {
  private animals: T[] = [];
  
  addAnimal(animal: T): void {
    this.animals.push(animal);
  }
  
  getAnimals(): T[] {
    return this.animals;
  }
}

const dogZoo = new Zoo<Dog>();
dogZoo.addAnimal(new Dog("Buddy"));
```

## 高级泛型模式

### 泛型工厂模式

```typescript
// 工厂函数
function createInstance<T>(c: new () => T): T {
  return new c();
}

class Person {
  constructor(public name: string = "Alice") {}
}

const person = createInstance(Person);

// 带参数的工厂
function createInstanceWithArgs<T>(
  c: new (...args: any[]) => T,
  ...args: any[]
): T {
  return new c(...args);
}

class User {
  constructor(public name: string, public age: number) {}
}

const user = createInstanceWithArgs(User, "Bob", 30);
```

### 泛型默认值

```typescript
// 泛型默认类型
interface Response<T = any> {
  data: T;
  status: number;
  message: string;
}

const response1: Response = {
  data: "anything",
  status: 200,
  message: "OK"
};

const response2: Response<User> = {
  data: { id: "1", name: "Alice", email: "alice@example.com" },
  status: 200,
  message: "OK"
};

// 条件默认类型
type DefaultTo<T, D> = T extends undefined ? D : T;

type A = DefaultTo<string, number>; // string
type B = DefaultTo<undefined, number>; // number
```

### 泛型协变和逆变

```typescript
// 协变（Covariance）- 类型参数在输出位置
interface Producer<out T> {
  produce(): T;
}

// 子类型可以赋值给父类型
const animalProducer: Producer<Animal> = {} as Producer<Dog>; // ✅ OK

// 逆变（Contravariance）- 类型参数在输入位置
interface Consumer<in T> {
  consume(value: T): void;
}

// 父类型可以赋值给子类型
const dogConsumer: Consumer<Dog> = {} as Consumer<Animal>; // ✅ OK

// 不变（Invariance）- 既是输入又是输出
interface Box<T> {
  value: T;
  setValue(value: T): void;
}

// 不能互相赋值
// const animalBox: Box<Animal> = {} as Box<Dog>; // ❌ Error
```

## 实战案例

### 1. 类型安全的事件发射器

```typescript
type EventMap = {
  [key: string]: any;
};

class TypedEventEmitter<Events extends EventMap> {
  private listeners: {
    [K in keyof Events]?: Array<(data: Events[K]) => void>;
  } = {};
  
  on<K extends keyof Events>(
    event: K,
    listener: (data: Events[K]) => void
  ): void {
    if (!this.listeners[event]) {
      this.listeners[event] = [];
    }
    this.listeners[event]!.push(listener);
  }
  
  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    const eventListeners = this.listeners[event];
    if (eventListeners) {
      eventListeners.forEach((listener) => listener(data));
    }
  }
  
  off<K extends keyof Events>(
    event: K,
    listener: (data: Events[K]) => void
  ): void {
    const eventListeners = this.listeners[event];
    if (eventListeners) {
      this.listeners[event] = eventListeners.filter(
        (l) => l !== listener
      ) as any;
    }
  }
}

// 使用
interface MyEvents {
  userLogin: { userId: string; timestamp: Date };
  userLogout: { userId: string };
  message: { from: string; text: string };
}

const emitter = new TypedEventEmitter<MyEvents>();

emitter.on("userLogin", (data) => {
  // data 类型是 { userId: string; timestamp: Date }
  console.log(`User ${data.userId} logged in`);
});

emitter.emit("userLogin", {
  userId: "123",
  timestamp: new Date()
});
```

### 2. 类型安全的状态管理

```typescript
type StateSlice<T> = {
  value: T;
  setValue: (value: T | ((prev: T) => T)) => void;
};

class Store<State extends Record<string, any>> {
  private state: State;
  private listeners: Set<(state: State) => void> = new Set();
  
  constructor(initialState: State) {
    this.state = initialState;
  }
  
  getState(): Readonly<State> {
    return this.state;
  }
  
  setState<K extends keyof State>(
    key: K,
    value: State[K] | ((prev: State[K]) => State[K])
  ): void {
    const newValue =
      typeof value === "function" ? value(this.state[key]) : value;
    
    this.state = {
      ...this.state,
      [key]: newValue
    };
    
    this.notify();
  }
  
  subscribe(listener: (state: State) => void): () => void {
    this.listeners.add(listener);
    
    return () => {
      this.listeners.delete(listener);
    };
  }
  
  private notify(): void {
    this.listeners.forEach((listener) => listener(this.state));
  }
  
  select<K extends keyof State>(key: K): State[K] {
    return this.state[key];
  }
}

// 使用
interface AppState {
  user: { name: string; email: string } | null;
  count: number;
  theme: "light" | "dark";
}

const store = new Store<AppState>({
  user: null,
  count: 0,
  theme: "light"
});

// 类型安全的更新
store.setState("count", (prev) => prev + 1);
store.setState("theme", "dark");
store.setState("user", {
  name: "Alice",
  email: "alice@example.com"
});

// 订阅变化
const unsubscribe = store.subscribe((state) => {
  console.log("State changed:", state);
});
```

### 3. 类型安全的 API 客户端

```typescript
type HTTPMethod = "GET" | "POST" | "PUT" | "DELETE" | "PATCH";

type APIEndpoint<Method extends HTTPMethod, Req, Res> = {
  method: Method;
  request: Req;
  response: Res;
};

type APISchema = {
  [path: string]: APIEndpoint<any, any, any>;
};

class TypedAPIClient<Schema extends APISchema> {
  constructor(private baseURL: string) {}
  
  async request<Path extends keyof Schema>(
    path: Path,
    ...args: Schema[Path]["request"] extends void
      ? []
      : [data: Schema[Path]["request"]]
  ): Promise<Schema[Path]["response"]> {
    const endpoint = path as string;
    const data = args[0];
    
    const response = await fetch(`${this.baseURL}${endpoint}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data)
    });
    
    return response.json();
  }
}

// 定义 API Schema
interface MyAPI {
  "/users": APIEndpoint<
    "GET",
    void,
    { id: string; name: string }[]
  >;
  "/users/:id": APIEndpoint<
    "GET",
    void,
    { id: string; name: string; email: string }
  >;
  "/users/create": APIEndpoint<
    "POST",
    { name: string; email: string },
    { id: string; name: string; email: string }
  >;
}

// 使用
const api = new TypedAPIClient<MyAPI>("https://api.example.com");

async function example() {
  // 类型安全的请求
  const users = await api.request("/users"); // { id: string; name: string }[]
  
  const newUser = await api.request("/users/create", {
    name: "Alice",
    email: "alice@example.com"
  }); // { id: string; name: string; email: string }
}
```

### 4. 泛型构建器模式

```typescript
class QueryBuilder<T> {
  private conditions: Array<(item: T) => boolean> = [];
  private sortFn?: (a: T, b: T) => number;
  private limitValue?: number;
  
  where(condition: (item: T) => boolean): this {
    this.conditions.push(condition);
    return this;
  }
  
  orderBy<K extends keyof T>(
    key: K,
    direction: "asc" | "desc" = "asc"
  ): this {
    this.sortFn = (a, b) => {
      const aVal = a[key];
      const bVal = b[key];
      
      if (aVal < bVal) return direction === "asc" ? -1 : 1;
      if (aVal > bVal) return direction === "asc" ? 1 : -1;
      return 0;
    };
    
    return this;
  }
  
  limit(value: number): this {
    this.limitValue = value;
    return this;
  }
  
  execute(data: T[]): T[] {
    let result = data.filter((item) =>
      this.conditions.every((condition) => condition(item))
    );
    
    if (this.sortFn) {
      result = result.sort(this.sortFn);
    }
    
    if (this.limitValue !== undefined) {
      result = result.slice(0, this.limitValue);
    }
    
    return result;
  }
}

// 使用
interface User {
  id: string;
  name: string;
  age: number;
  email: string;
}

const users: User[] = [
  { id: "1", name: "Alice", age: 25, email: "alice@example.com" },
  { id: "2", name: "Bob", age: 30, email: "bob@example.com" },
  { id: "3", name: "Charlie", age: 35, email: "charlie@example.com" }
];

const result = new QueryBuilder<User>()
  .where((user) => user.age > 25)
  .orderBy("name", "asc")
  .limit(10)
  .execute(users);
```

## 常见面试题

### 1. 什么时候需要使用泛型？

<details>
<summary>点击查看答案</summary>

**使用泛型的场景**：

1. **需要支持多种类型**
```typescript
// ❌ 不好：为每种类型写函数
function wrapInArrayNumber(value: number): number[] {
  return [value];
}

function wrapInArrayString(value: string): string[] {
  return [value];
}

// ✅ 好：使用泛型
function wrapInArray<T>(value: T): T[] {
  return [value];
}
```

2. **需要保持类型关系**
```typescript
// ✅ 输入和输出类型相关
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

3. **创建可重用的数据结构**
```typescript
// ✅ 泛型类
class Stack<T> {
  private items: T[] = [];
  push(item: T): void {
    this.items.push(item);
  }
  pop(): T | undefined {
    return this.items.pop();
  }
}
```

4. **类型安全的工具函数**
```typescript
// ✅ 保持对象结构
function pick<T, K extends keyof T>(obj: T, keys: K[]): Pick<T, K> {
  const result = {} as Pick<T, K>;
  keys.forEach((key) => {
    result[key] = obj[key];
  });
  return result;
}
```
</details>

### 2. 泛型约束有什么作用？

<details>
<summary>点击查看答案</summary>

**泛型约束**用于限制泛型类型参数必须满足某些条件。

```typescript
// 1. 确保有特定属性
interface Lengthwise {
  length: number;
}

function logLength<T extends Lengthwise>(arg: T): void {
  console.log(arg.length); // ✅ 可以访问 length
}

logLength("hello"); // ✅ string 有 length
logLength([1, 2, 3]); // ✅ array 有 length
// logLength(42); // ❌ number 没有 length

// 2. 约束为某个类型的子集
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person = { name: "Alice", age: 25 };
getProperty(person, "name"); // ✅ OK
// getProperty(person, "email"); // ❌ Error

// 3. 多个约束
interface Printable {
  print(): void;
}

interface Saveable {
  save(): void;
}

function process<T extends Printable & Saveable>(item: T): void {
  item.print();
  item.save();
}

// 4. 使用类作为约束
class Animal {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

function createAnimal<T extends Animal>(
  c: new (name: string) => T,
  name: string
): T {
  return new c(name);
}
```
</details>

### 3. 如何实现一个类型安全的深度克隆函数？

<details>
<summary>点击查看答案</summary>

```typescript
type Primitive = string | number | boolean | null | undefined | symbol | bigint;

function deepClone<T>(value: T): T {
  // 基本类型直接返回
  if (value === null || typeof value !== "object") {
    return value;
  }
  
  // Date
  if (value instanceof Date) {
    return new Date(value.getTime()) as any;
  }
  
  // RegExp
  if (value instanceof RegExp) {
    return new RegExp(value.source, value.flags) as any;
  }
  
  // Array
  if (Array.isArray(value)) {
    return value.map((item) => deepClone(item)) as any;
  }
  
  // Object
  const cloned = {} as T;
  for (const key in value) {
    if (value.hasOwnProperty(key)) {
      cloned[key] = deepClone(value[key]);
    }
  }
  
  return cloned;
}

// 使用
interface User {
  name: string;
  age: number;
  address: {
    city: string;
    country: string;
  };
  hobbies: string[];
}

const user: User = {
  name: "Alice",
  age: 25,
  address: {
    city: "New York",
    country: "USA"
  },
  hobbies: ["reading", "coding"]
};

const cloned = deepClone(user);
cloned.address.city = "Boston"; // 不影响原对象
```
</details>

### 4. 如何实现一个类型安全的 Omit？

<details>
<summary>点击查看答案</summary>

```typescript
// 方式 1: 使用 Pick 和 Exclude
type MyOmit1<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// 方式 2: 使用映射类型
type MyOmit2<T, K extends keyof any> = {
  [P in keyof T as P extends K ? never : P]: T[P];
};

// 测试
interface User {
  id: string;
  name: string;
  email: string;
  password: string;
}

type PublicUser = MyOmit1<User, "password">;
// {
//   id: string;
//   name: string;
//   email: string;
// }

// 支持多个属性
type MinimalUser = MyOmit2<User, "password" | "email">;
// {
//   id: string;
//   name: string;
// }
```
</details>

### 5. 泛型的协变和逆变是什么？

<details>
<summary>点击查看答案</summary>

**协变（Covariance）**：类型参数在输出位置，子类型可以赋值给父类型

**逆变（Contravariance）**：类型参数在输入位置，父类型可以赋值给子类型

```typescript
class Animal {
  name: string = "";
}

class Dog extends Animal {
  bark() {}
}

// 1. 协变 - 在返回类型位置
interface Producer<out T> {
  produce(): T;
}

const dogProducer: Producer<Dog> = {
  produce: () => new Dog()
};

// ✅ 协变：Dog 是 Animal 的子类型
const animalProducer: Producer<Animal> = dogProducer;

// 2. 逆变 - 在参数位置
interface Consumer<in T> {
  consume(value: T): void;
}

const animalConsumer: Consumer<Animal> = {
  consume: (animal) => console.log(animal.name)
};

// ✅ 逆变：Animal 的处理器可以处理 Dog
const dogConsumer: Consumer<Dog> = animalConsumer;

// 3. 不变 - 既是输入又是输出
interface Box<T> {
  value: T;
  setValue(value: T): void;
}

const dogBox: Box<Dog> = {
  value: new Dog(),
  setValue(value) {}
};

// ❌ 不能互相赋值
// const animalBox: Box<Animal> = dogBox; // Error
```

**实际意义**：

- 数组是协变的（返回元素）
- 函数参数是逆变的（接收参数）
- 对象属性既协变又逆变（不变）
</details>

## 最佳实践

### 1. 尽量使用类型推断

```typescript
// ❌ 不必要的泛型声明
const arr = identity<number>(42);

// ✅ 让 TypeScript 推断
const arr = identity(42);
```

### 2. 使用有意义的泛型参数名

```typescript
// ❌ 单字母不清晰（除非很简单）
function map<T, U>(arr: T[], fn: (item: T) => U): U[] {
  return arr.map(fn);
}

// ✅ 描述性的名称
function transformArray<Input, Output>(
  arr: Input[],
  transformer: (item: Input) => Output
): Output[] {
  return arr.map(transformer);
}
```

### 3. 适当使用泛型约束

```typescript
// ✅ 明确约束
function sortByKey<T, K extends keyof T>(
  arr: T[],
  key: K
): T[] {
  return [...arr].sort((a, b) => {
    if (a[key] < b[key]) return -1;
    if (a[key] > b[key]) return 1;
    return 0;
  });
}
```

### 4. 避免过度使用泛型

```typescript
// ❌ 过度泛型
function add<T extends number>(a: T, b: T): T {
  return (a + b) as T;
}

// ✅ 简单直接
function add(a: number, b: number): number {
  return a + b;
}
```

