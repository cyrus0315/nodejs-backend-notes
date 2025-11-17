# TypeScript 高级类型

## 联合类型和交叉类型

### 联合类型 (Union Types)

```typescript
// 基本联合类型
type StringOrNumber = string | number;

function printId(id: StringOrNumber) {
  console.log(`ID: ${id}`);
}

printId("abc"); // ✅ OK
printId(123); // ✅ OK

// 类型收窄
function printId2(id: string | number) {
  if (typeof id === "string") {
    // 此处 id 是 string
    console.log(id.toUpperCase());
  } else {
    // 此处 id 是 number
    console.log(id.toFixed(2));
  }
}

// 对象联合类型
type Success = {
  success: true;
  data: any;
};

type Error = {
  success: false;
  error: string;
};

type Response = Success | Error;

function handleResponse(response: Response) {
  if (response.success) {
    console.log(response.data);
  } else {
    console.error(response.error);
  }
}
```

### 交叉类型 (Intersection Types)

```typescript
// 组合多个类型
type Person = {
  name: string;
  age: number;
};

type Employee = {
  employeeId: number;
  department: string;
};

type EmployeePerson = Person & Employee;

const employee: EmployeePerson = {
  name: "Alice",
  age: 30,
  employeeId: 12345,
  department: "Engineering"
};

// Mixin 模式
interface Timestamped {
  timestamp: Date;
}

interface Tagged {
  tags: string[];
}

type TimestampedTagged = Timestamped & Tagged;

const item: TimestampedTagged = {
  timestamp: new Date(),
  tags: ["typescript", "advanced"]
};

// 函数类型交叉
type Logger = {
  log: (message: string) => void;
};

type Formatter = {
  format: (data: any) => string;
};

type FormattedLogger = Logger & Formatter;

const logger: FormattedLogger = {
  log(message) {
    console.log(message);
  },
  format(data) {
    return JSON.stringify(data);
  }
};
```

## 条件类型 (Conditional Types)

```typescript
// 基本语法：T extends U ? X : Y
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false

// 实用示例：提取返回类型
type ReturnTypeOf<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { name: "Alice", age: 25 };
}

type User = ReturnTypeOf<typeof getUser>; // { name: string; age: number; }

// NonNullable
type NonNullable<T> = T extends null | undefined ? never : T;

type C = NonNullable<string | null>; // string
type D = NonNullable<string | undefined>; // string

// 分布式条件类型
type ToArray<T> = T extends any ? T[] : never;

type E = ToArray<string | number>; // string[] | number[]

// 排除类型
type Exclude<T, U> = T extends U ? never : T;

type F = Exclude<"a" | "b" | "c", "a">; // "b" | "c"
type G = Exclude<string | number | boolean, string>; // number | boolean

// 提取类型
type Extract<T, U> = T extends U ? T : never;

type H = Extract<"a" | "b" | "c", "a" | "b">; // "a" | "b"

// 复杂示例：函数参数类型
type FunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

type NonFunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? never : K;
}[keyof T];

interface Part {
  id: number;
  name: string;
  subparts: Part[];
  updatePart(newName: string): void;
}

type T1 = FunctionPropertyNames<Part>; // "updatePart"
type T2 = NonFunctionPropertyNames<Part>; // "id" | "name" | "subparts"
```

## 映射类型 (Mapped Types)

```typescript
// 基本映射类型
type Readonly<T> = {
  readonly [P in keyof T]: T[P];
};

type Partial<T> = {
  [P in keyof T]?: T[P];
};

type Required<T> = {
  [P in keyof T]-?: T[P];
};

// 使用示例
interface User {
  name: string;
  age: number;
  email?: string;
}

type ReadonlyUser = Readonly<User>;
// {
//   readonly name: string;
//   readonly age: number;
//   readonly email?: string;
// }

type PartialUser = Partial<User>;
// {
//   name?: string;
//   age?: string;
//   email?: string;
// }

// Pick - 选择部分属性
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};

type UserPreview = Pick<User, "name" | "email">;
// {
//   name: string;
//   email?: string;
// }

// Omit - 排除部分属性
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

type UserWithoutEmail = Omit<User, "email">;
// {
//   name: string;
//   age: number;
// }

// Record - 创建对象类型
type Record<K extends keyof any, T> = {
  [P in K]: T;
};

type PageInfo = {
  title: string;
  content: string;
};

type Pages = Record<"home" | "about" | "contact", PageInfo>;
// {
//   home: PageInfo;
//   about: PageInfo;
//   contact: PageInfo;
// }

// 自定义映射类型
type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
// {
//   name: string | null;
//   age: number | null;
//   email?: string | null;
// }

// 深度 Partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

interface Config {
  database: {
    host: string;
    port: number;
    credentials: {
      username: string;
      password: string;
    };
  };
}

type PartialConfig = DeepPartial<Config>;
// 所有属性都变为可选
```

## 索引类型 (Index Types)

```typescript
// keyof 操作符
interface Person {
  name: string;
  age: number;
  email: string;
}

type PersonKeys = keyof Person; // "name" | "age" | "email"

// 索引访问类型
type NameType = Person["name"]; // string
type AgeType = Person["age"]; // number

// 索引类型查询
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const person: Person = {
  name: "Alice",
  age: 25,
  email: "alice@example.com"
};

const name = getProperty(person, "name"); // string
const age = getProperty(person, "age"); // number

// 索引签名
interface StringMap {
  [key: string]: string;
}

interface NumberMap {
  [key: string]: number;
}

// 混合索引签名
interface Dictionary {
  [key: string]: any;
  length: number;
}

// 实用工具：Getter 类型
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface State {
  name: string;
  age: number;
}

type StateGetters = Getters<State>;
// {
//   getName: () => string;
//   getAge: () => number;
// }
```

## 模板字面量类型 (Template Literal Types)

```typescript
// 基本用法
type World = "world";
type Greeting = `hello ${World}`; // "hello world"

// 组合多个字符串
type Direction = "left" | "right" | "top" | "bottom";
type Padding = `padding-${Direction}`; 
// "padding-left" | "padding-right" | "padding-top" | "padding-bottom"

// 多个模板变量
type Size = "small" | "medium" | "large";
type Color = "red" | "green" | "blue";
type Style = `${Size}-${Color}`;
// "small-red" | "small-green" | "small-blue" | ...

// 内置字符串操作
type Uppercase<S extends string> = intrinsic;
type Lowercase<S extends string> = intrinsic;
type Capitalize<S extends string> = intrinsic;
type Uncapitalize<S extends string> = intrinsic;

type LoudGreeting = Uppercase<"Hello">; // "HELLO"
type QuietGreeting = Lowercase<"HELLO">; // "hello"
type PropName = Capitalize<"userName">; // "UserName"

// 实战：事件监听器类型
type EventName = "click" | "scroll" | "mousemove";
type EventHandler = `on${Capitalize<EventName>}`;
// "onClick" | "onScroll" | "onMousemove"

interface Events {
  onClick: (e: MouseEvent) => void;
  onScroll: (e: Event) => void;
  onMousemove: (e: MouseEvent) => void;
}

// 实战：API 路由类型
type HTTPMethod = "get" | "post" | "put" | "delete";
type Route = "/users" | "/posts" | "/comments";
type API = `${HTTPMethod} ${Route}`;
// "get /users" | "post /users" | ...

// 推断模板字面量类型
type ExtractRouteParams<T extends string> =
  T extends `${infer Start}/:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractRouteParams<Rest>]: string }
    : T extends `${infer Start}/:${infer Param}`
    ? { [K in Param]: string }
    : {};

type Params = ExtractRouteParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string; }
```

## infer 关键字

```typescript
// 基本用法：推断类型
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { name: "Alice", age: 25 };
}

type UserType = ReturnType<typeof getUser>; // { name: string; age: number; }

// 推断参数类型
type Parameters<T> = T extends (...args: infer P) => any ? P : never;

function createUser(name: string, age: number) {
  return { name, age };
}

type CreateUserParams = Parameters<typeof createUser>; // [string, number]

// 推断数组元素类型
type ArrayElement<T> = T extends (infer U)[] ? U : never;

type StringArray = string[];
type Element = ArrayElement<StringArray>; // string

// 推断 Promise 类型
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;

type A = UnwrapPromise<Promise<string>>; // string
type B = UnwrapPromise<string>; // string

// 复杂示例：提取函数第一个参数类型
type FirstParameter<T> = T extends (first: infer F, ...args: any[]) => any
  ? F
  : never;

function example(str: string, num: number, bool: boolean) {}

type First = FirstParameter<typeof example>; // string

// 提取构造函数参数类型
type ConstructorParameters<T> = T extends new (...args: infer P) => any
  ? P
  : never;

class User {
  constructor(name: string, age: number) {}
}

type UserParams = ConstructorParameters<typeof User>; // [string, number]

// 推断对象属性类型
type PropertyType<T, K extends keyof T> = T extends { [key in K]: infer U }
  ? U
  : never;

interface Person {
  name: string;
  age: number;
}

type NameType = PropertyType<Person, "name">; // string
```

## 工具类型 (Utility Types)

```typescript
// 1. Partial<T> - 所有属性可选
interface User {
  name: string;
  age: number;
  email: string;
}

type PartialUser = Partial<User>;
// { name?: string; age?: number; email?: string; }

function updateUser(user: User, updates: Partial<User>): User {
  return { ...user, ...updates };
}

// 2. Required<T> - 所有属性必需
interface Config {
  host?: string;
  port?: number;
}

type RequiredConfig = Required<Config>;
// { host: string; port: number; }

// 3. Readonly<T> - 所有属性只读
type ReadonlyUser = Readonly<User>;

const user: ReadonlyUser = {
  name: "Alice",
  age: 25,
  email: "alice@example.com"
};

// user.name = "Bob"; // ❌ Error

// 4. Record<K, T> - 创建对象类型
type Role = "admin" | "user" | "guest";

type Permissions = Record<Role, string[]>;
// {
//   admin: string[];
//   user: string[];
//   guest: string[];
// }

const permissions: Permissions = {
  admin: ["read", "write", "delete"],
  user: ["read", "write"],
  guest: ["read"]
};

// 5. Pick<T, K> - 选择属性
type UserPreview = Pick<User, "name" | "email">;
// { name: string; email: string; }

// 6. Omit<T, K> - 排除属性
type UserWithoutEmail = Omit<User, "email">;
// { name: string; age: number; }

// 7. Exclude<T, U> - 从联合类型中排除
type T1 = Exclude<"a" | "b" | "c", "a">; // "b" | "c"
type T2 = Exclude<string | number | boolean, string>; // number | boolean

// 8. Extract<T, U> - 从联合类型中提取
type T3 = Extract<"a" | "b" | "c", "a" | "f">; // "a"
type T4 = Extract<string | number, number>; // number

// 9. NonNullable<T> - 排除 null 和 undefined
type T5 = NonNullable<string | null | undefined>; // string

// 10. ReturnType<T> - 函数返回类型
function getUser() {
  return { name: "Alice", age: 25 };
}

type UserType = ReturnType<typeof getUser>; 
// { name: string; age: number; }

// 11. Parameters<T> - 函数参数类型
function createUser(name: string, age: number) {}

type Params = Parameters<typeof createUser>; // [string, number]

// 12. ConstructorParameters<T> - 构造函数参数
class Person {
  constructor(name: string, age: number) {}
}

type PersonParams = ConstructorParameters<typeof Person>; // [string, number]

// 13. InstanceType<T> - 实例类型
type PersonInstance = InstanceType<typeof Person>; // Person

// 14. Awaited<T> - 解包 Promise
type T6 = Awaited<Promise<string>>; // string
type T7 = Awaited<Promise<Promise<number>>>; // number

// 自定义工具类型
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};

type WriteableUser = Mutable<ReadonlyUser>;

type Nullable<T> = {
  [P in keyof T]: T[P] | null;
};

type NullableUser = Nullable<User>;
```

## 类型推断和类型断言

```typescript
// 类型推断
let x = 3; // 推断为 number
let arr = [1, 2, 3]; // 推断为 number[]
let tuple = ["hello", 10]; // 推断为 (string | number)[]

// 最佳通用类型
let mixed = [1, "two", true]; // (string | number | boolean)[]

// 上下文类型推断
window.onmousedown = function(event) {
  // event 被推断为 MouseEvent
  console.log(event.button);
};

// const 断言
let obj1 = { name: "Alice" }; // { name: string }
let obj2 = { name: "Alice" } as const; // { readonly name: "Alice" }

let arr1 = [1, 2, 3]; // number[]
let arr2 = [1, 2, 3] as const; // readonly [1, 2, 3]

// 类型断言
let someValue: unknown = "this is a string";
let strLength = (someValue as string).length;

// 双重断言（不推荐）
let value = "string" as unknown as number;

// 非空断言
function getValue(): string | null {
  return "value";
}

let value1 = getValue()!; // 断言不为 null
```

## 常见面试题

### 1. 什么是类型收窄(Type Narrowing)？

<details>
<summary>点击查看答案</summary>

**类型收窄**是 TypeScript 根据代码逻辑自动缩小类型范围的能力。

```typescript
function printId(id: string | number) {
  // 初始：id 是 string | number
  
  if (typeof id === "string") {
    // 收窄为 string
    console.log(id.toUpperCase());
  } else {
    // 收窄为 number
    console.log(id.toFixed(2));
  }
}

// 其他收窄方式
function example(value: string | null) {
  // 1. typeof 检查
  if (typeof value === "string") {
    console.log(value.toUpperCase());
  }
  
  // 2. 真值检查
  if (value) {
    console.log(value.toUpperCase());
  }
  
  // 3. 相等性检查
  if (value !== null) {
    console.log(value.toUpperCase());
  }
}

// 判别联合
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number };

function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.size ** 2;
  }
}
```
</details>

### 2. 联合类型和交叉类型的区别？

<details>
<summary>点击查看答案</summary>

**联合类型 (Union)**：值可以是多个类型之一

```typescript
type StringOrNumber = string | number;

let value: StringOrNumber;
value = "hello"; // ✅ OK
value = 42; // ✅ OK
```

**交叉类型 (Intersection)**：值必须同时满足多个类型

```typescript
type Person = {
  name: string;
  age: number;
};

type Employee = {
  employeeId: number;
};

type EmployeePerson = Person & Employee;

const employee: EmployeePerson = {
  name: "Alice",
  age: 30,
  employeeId: 12345 // 必须包含所有属性
};
```

**对比**：

| 特性 | 联合类型 | 交叉类型 |
|------|---------|---------|
| 符号 | `\|` | `&` |
| 含义 | "或" | "和" |
| 属性 | 只能访问共有属性 | 包含所有属性 |
| 使用 | 类型选择 | 类型组合 |

```typescript
// 联合类型：只能访问共有属性
type A = { a: string; shared: string };
type B = { b: number; shared: string };

function useUnion(value: A | B) {
  console.log(value.shared); // ✅ OK
  // console.log(value.a); // ❌ Error
}

// 交叉类型：包含所有属性
function useIntersection(value: A & B) {
  console.log(value.a); // ✅ OK
  console.log(value.b); // ✅ OK
  console.log(value.shared); // ✅ OK
}
```
</details>

### 3. 如何实现深度 Readonly？

<details>
<summary>点击查看答案</summary>

```typescript
// 递归实现深度 Readonly
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object
    ? T[P] extends Function
      ? T[P]
      : DeepReadonly<T[P]>
    : T[P];
};

interface Config {
  server: {
    host: string;
    port: number;
    ssl: {
      enabled: boolean;
      cert: string;
    };
  };
  features: string[];
}

type ReadonlyConfig = DeepReadonly<Config>;

const config: ReadonlyConfig = {
  server: {
    host: "localhost",
    port: 3000,
    ssl: {
      enabled: true,
      cert: "cert.pem"
    }
  },
  features: ["auth", "api"]
};

// config.server.host = ""; // ❌ Error
// config.server.ssl.enabled = false; // ❌ Error
// config.features.push("new"); // ❌ Error
```
</details>

### 4. Partial 和 Required 如何实现？

<details>
<summary>点击查看答案</summary>

```typescript
// Partial 实现：所有属性变为可选
type MyPartial<T> = {
  [P in keyof T]?: T[P];
};

// Required 实现：所有属性变为必需
type MyRequired<T> = {
  [P in keyof T]-?: T[P]; // -? 移除可选修饰符
};

// Readonly 实现
type MyReadonly<T> = {
  readonly [P in keyof T]: T[P];
};

// Mutable 实现：移除 readonly
type Mutable<T> = {
  -readonly [P in keyof T]: T[P]; // -readonly 移除只读修饰符
};

// 测试
interface User {
  name: string;
  age?: number;
}

type PartialUser = MyPartial<User>;
// { name?: string; age?: number; }

type RequiredUser = MyRequired<User>;
// { name: string; age: number; }

type ReadonlyUser = MyReadonly<User>;
// { readonly name: string; readonly age?: number; }

type MutableUser = Mutable<ReadonlyUser>;
// { name: string; age?: number; }
```
</details>

### 5. 如何提取函数的参数和返回类型？

<details>
<summary>点击查看答案</summary>

```typescript
// 提取参数类型
type Parameters<T extends (...args: any) => any> = T extends (
  ...args: infer P
) => any
  ? P
  : never;

// 提取返回类型
type ReturnType<T extends (...args: any) => any> = T extends (
  ...args: any
) => infer R
  ? R
  : never;

// 使用示例
function createUser(name: string, age: number, email: string) {
  return {
    id: Math.random(),
    name,
    age,
    email,
    createdAt: new Date()
  };
}

type CreateUserParams = Parameters<typeof createUser>;
// [string, number, string]

type User = ReturnType<typeof createUser>;
// {
//   id: number;
//   name: string;
//   age: number;
//   email: string;
//   createdAt: Date;
// }

// 提取第一个参数
type FirstParameter<T extends (...args: any) => any> = T extends (
  first: infer F,
  ...rest: any
) => any
  ? F
  : never;

type FirstParam = FirstParameter<typeof createUser>; // string

// 提取除第一个外的参数
type RestParameters<T extends (...args: any) => any> = T extends (
  first: any,
  ...rest: infer R
) => any
  ? R
  : never;

type RestParams = RestParameters<typeof createUser>; // [number, string]
```
</details>

## 最佳实践

### 1. 优先使用联合类型而不是枚举

```typescript
// ❌ 不推荐
enum Status {
  Pending,
  Approved,
  Rejected
}

// ✅ 推荐
type Status = "pending" | "approved" | "rejected";
```

### 2. 使用工具类型简化代码

```typescript
// ❌ 不好
interface UpdateUser {
  name?: string;
  age?: number;
  email?: string;
}

// ✅ 好
interface User {
  name: string;
  age: number;
  email: string;
}

type UpdateUser = Partial<User>;
```

### 3. 使用模板字面量类型提高类型安全

```typescript
// ✅ 类型安全的 CSS 属性
type CSSProperty = 
  | "margin"
  | "padding"
  | "border";

type CSSPropertyWithSide = `${CSSProperty}-${"top" | "right" | "bottom" | "left"}`;

function setStyle(property: CSSPropertyWithSide, value: string) {
  // ...
}

setStyle("margin-top", "10px"); // ✅ OK
// setStyle("margin", "10px"); // ❌ Error
```

### 4. 使用判别联合类型处理复杂状态

```typescript
// ✅ 类型安全的状态管理
type RequestState =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: any }
  | { status: "error"; error: string };

function handleRequest(state: RequestState) {
  switch (state.status) {
    case "idle":
      return "Not started";
    case "loading":
      return "Loading...";
    case "success":
      return state.data; // ✅ 可以访问 data
    case "error":
      return state.error; // ✅ 可以访问 error
  }
}
```

