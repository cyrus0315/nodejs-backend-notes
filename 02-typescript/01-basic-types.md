# TypeScript 基础类型

## 基本类型

### 原始类型

```typescript
// 1. boolean
let isDone: boolean = false;
let isActive: boolean = true;

// 2. number
let decimal: number = 6;
let hex: number = 0xf00d;
let binary: number = 0b1010;
let octal: number = 0o744;
let big: bigint = 100n;

// 3. string
let color: string = "blue";
let fullName: string = `Bob Smith`;
let sentence: string = `Hello, my name is ${fullName}`;

// 4. null 和 undefined
let u: undefined = undefined;
let n: null = null;

// strictNullChecks: true 时
let name1: string = "Alice";
// name1 = null; // ❌ Error

let name2: string | null = "Alice";
name2 = null; // ✅ OK

// 5. symbol
let sym1: symbol = Symbol("key");
let sym2: symbol = Symbol("key");
console.log(sym1 === sym2); // false

// 6. void
function warnUser(): void {
  console.log("This is a warning");
}

// 7. never - 永远不会返回的函数
function error(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}
```

### any 和 unknown

```typescript
// any - 关闭类型检查（尽量避免使用）
let notSure: any = 4;
notSure = "maybe a string";
notSure = false;

notSure.ifItExists(); // ✅ OK（编译通过，但可能运行时错误）
notSure.toFixed(); // ✅ OK

// unknown - 类型安全的 any
let value: unknown = 4;
value = "string";
value = false;

// value.toFixed(); // ❌ Error: unknown 类型不能直接调用方法

// 需要类型检查或断言
if (typeof value === "number") {
  console.log(value.toFixed(2)); // ✅ OK
}

// 类型断言
console.log((value as string).toUpperCase());

// ✅ 推荐：使用 unknown 而不是 any
function processValue(value: unknown) {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  if (typeof value === "number") {
    return value.toFixed(2);
  }
  return value;
}
```

## 数组和元组

### 数组

```typescript
// 方式 1: type[]
let list1: number[] = [1, 2, 3];
let names1: string[] = ["Alice", "Bob"];

// 方式 2: Array<type>
let list2: Array<number> = [1, 2, 3];
let names2: Array<string> = ["Alice", "Bob"];

// 只读数组
let readonlyList: ReadonlyArray<number> = [1, 2, 3];
// readonlyList.push(4); // ❌ Error
// readonlyList[0] = 0; // ❌ Error

// 简写形式
let readonlyList2: readonly number[] = [1, 2, 3];

// 多维数组
let matrix: number[][] = [
  [1, 2, 3],
  [4, 5, 6]
];

// 混合类型数组
let mixed: (number | string)[] = [1, "two", 3];
```

### 元组 (Tuple)

```typescript
// 固定长度和类型的数组
let tuple: [string, number];
tuple = ["hello", 10]; // ✅ OK
// tuple = [10, "hello"]; // ❌ Error

// 访问元素
console.log(tuple[0].toUpperCase()); // HELLO
console.log(tuple[1].toFixed(2)); // 10.00

// 可选元素
let optionalTuple: [string, number?];
optionalTuple = ["hello"];
optionalTuple = ["hello", 10];

// 剩余元素
let restTuple: [string, ...number[]];
restTuple = ["hello"];
restTuple = ["hello", 1, 2, 3];

// 只读元组
let readonlyTuple: readonly [string, number] = ["hello", 10];
// readonlyTuple[0] = "world"; // ❌ Error

// 实战：函数返回值
function useState<T>(initial: T): [T, (value: T) => void] {
  let state = initial;
  
  const setState = (value: T) => {
    state = value;
  };
  
  return [state, setState];
}

const [count, setCount] = useState(0);
```

## 对象类型

### Object、object 和 {}

```typescript
// Object - 所有对象的基类型（包括原始类型的包装对象）
let obj1: Object;
obj1 = {};
obj1 = [];
obj1 = () => {};
obj1 = new String("hello");

// object - 非原始类型
let obj2: object;
obj2 = {};
obj2 = [];
obj2 = () => {};
// obj2 = 1; // ❌ Error
// obj2 = "hello"; // ❌ Error

// {} - 空对象类型（类似 Object）
let obj3: {};
obj3 = {};
obj3 = 1;
obj3 = "hello";

// ✅ 推荐：使用接口或类型别名定义对象
interface User {
  name: string;
  age: number;
}

let user: User = {
  name: "Alice",
  age: 25
};
```

### 接口 (Interface)

```typescript
// 基本接口
interface User {
  name: string;
  age: number;
  email: string;
}

const user: User = {
  name: "Alice",
  age: 25,
  email: "alice@example.com"
};

// 可选属性
interface Config {
  host: string;
  port?: number; // 可选
  timeout?: number;
}

const config1: Config = { host: "localhost" }; // ✅ OK
const config2: Config = { host: "localhost", port: 3000 }; // ✅ OK

// 只读属性
interface Point {
  readonly x: number;
  readonly y: number;
}

const point: Point = { x: 10, y: 20 };
// point.x = 5; // ❌ Error

// 索引签名
interface StringArray {
  [index: number]: string;
}

const myArray: StringArray = ["Alice", "Bob"];

interface StringMap {
  [key: string]: string;
}

const myMap: StringMap = {
  name: "Alice",
  email: "alice@example.com"
};

// 混合索引签名
interface Dictionary {
  [key: string]: any;
  length: number; // 明确的属性
  name: string;
}

// 函数类型
interface SearchFunc {
  (source: string, subString: string): boolean;
}

const mySearch: SearchFunc = (src, sub) => {
  return src.includes(sub);
};

// 类类型
interface ClockInterface {
  currentTime: Date;
  setTime(d: Date): void;
}

class Clock implements ClockInterface {
  currentTime: Date = new Date();
  
  setTime(d: Date) {
    this.currentTime = d;
  }
}

// 接口继承
interface Shape {
  color: string;
}

interface Square extends Shape {
  sideLength: number;
}

const square: Square = {
  color: "blue",
  sideLength: 10
};

// 多重继承
interface Shape {
  color: string;
}

interface PenStroke {
  penWidth: number;
}

interface Square extends Shape, PenStroke {
  sideLength: number;
}
```

## 类型别名 (Type Alias)

```typescript
// 基本类型别名
type Name = string;
type Age = number;
type User = {
  name: Name;
  age: Age;
};

// 联合类型
type StringOrNumber = string | number;
let value: StringOrNumber;
value = "hello";
value = 42;

// 交叉类型
type Admin = {
  name: string;
  privileges: string[];
};

type Employee = {
  name: string;
  startDate: Date;
};

type ElevatedEmployee = Admin & Employee;

const employee: ElevatedEmployee = {
  name: "Alice",
  privileges: ["server-access"],
  startDate: new Date()
};

// 函数类型
type GreetFunction = (name: string) => void;

const greet: GreetFunction = (name) => {
  console.log(`Hello, ${name}!`);
};

// 条件类型
type IsString<T> = T extends string ? true : false;

type A = IsString<string>; // true
type B = IsString<number>; // false
```

## 字面量类型

```typescript
// 字符串字面量
type Direction = "north" | "south" | "east" | "west";

let direction: Direction;
direction = "north"; // ✅ OK
// direction = "up"; // ❌ Error

// 数字字面量
type DiceRoll = 1 | 2 | 3 | 4 | 5 | 6;

let roll: DiceRoll = 3; // ✅ OK
// let roll: DiceRoll = 7; // ❌ Error

// 布尔字面量
type Yes = true;
type No = false;

// 混合字面量
type Status = "success" | "error" | 404 | 500;

// 实战：配置类型
type LogLevel = "debug" | "info" | "warn" | "error";

interface LoggerConfig {
  level: LogLevel;
  format: "json" | "text";
}

const config: LoggerConfig = {
  level: "info",
  format: "json"
};
```

## 枚举 (Enum)

```typescript
// 数字枚举
enum Direction {
  Up = 1,
  Down,
  Left,
  Right
}

console.log(Direction.Up); // 1
console.log(Direction.Down); // 2

// 反向映射
console.log(Direction[1]); // "Up"

// 字符串枚举
enum LogLevel {
  Debug = "DEBUG",
  Info = "INFO",
  Warn = "WARN",
  Error = "ERROR"
}

const level: LogLevel = LogLevel.Info;

// 常量枚举（编译后会被内联）
const enum Color {
  Red,
  Green,
  Blue
}

let c: Color = Color.Red;
// 编译后: let c = 0 /* Red */;

// 异构枚举（不推荐）
enum Mixed {
  No = 0,
  Yes = "YES"
}

// ✅ 推荐：使用联合类型替代枚举
type LogLevel2 = "DEBUG" | "INFO" | "WARN" | "ERROR";

// 或使用 as const
const LogLevel3 = {
  Debug: "DEBUG",
  Info: "INFO",
  Warn: "WARN",
  Error: "ERROR"
} as const;

type LogLevel3Type = typeof LogLevel3[keyof typeof LogLevel3];
```

## 类型断言

```typescript
// 方式 1: as 语法（推荐）
let someValue: unknown = "this is a string";
let strLength: number = (someValue as string).length;

// 方式 2: 尖括号语法（在 JSX 中不可用）
let someValue2: unknown = "this is a string";
let strLength2: number = (<string>someValue2).length;

// 双重断言（不推荐，除非确实需要）
const value = "string" as unknown as number; // ⚠️ 危险

// const 断言
let x = "hello" as const; // 类型: "hello"
// x = "world"; // ❌ Error

let arr = [10, 20] as const; // 类型: readonly [10, 20]
// arr[0] = 30; // ❌ Error

let obj = { name: "Alice", age: 25 } as const;
// obj.name = "Bob"; // ❌ Error

// 非空断言
function getValue(): string | null {
  return "value";
}

let value1 = getValue();
// let length1 = value1.length; // ❌ Error: 可能为 null

let value2 = getValue()!; // 断言不为 null
let length2 = value2.length; // ✅ OK（但要确保不为 null）

// 实战：DOM 操作
const button = document.querySelector("button"); // HTMLButtonElement | null
// button.click(); // ❌ Error

// 方式 1: 类型断言
(button as HTMLButtonElement).click();

// 方式 2: 非空断言
button!.click();

// 方式 3: 类型守卫（推荐）
if (button) {
  button.click();
}
```

## 类型守卫

```typescript
// 1. typeof 类型守卫
function padLeft(value: string, padding: string | number) {
  if (typeof padding === "number") {
    return " ".repeat(padding) + value;
  }
  if (typeof padding === "string") {
    return padding + value;
  }
}

// 2. instanceof 类型守卫
class Bird {
  fly() {
    console.log("Flying");
  }
}

class Fish {
  swim() {
    console.log("Swimming");
  }
}

function move(animal: Bird | Fish) {
  if (animal instanceof Bird) {
    animal.fly();
  } else {
    animal.swim();
  }
}

// 3. in 操作符
interface Bird2 {
  fly: () => void;
}

interface Fish2 {
  swim: () => void;
}

function move2(animal: Bird2 | Fish2) {
  if ("fly" in animal) {
    animal.fly();
  } else {
    animal.swim();
  }
}

// 4. 自定义类型守卫
interface Bird3 {
  type: "bird";
  fly: () => void;
}

interface Fish3 {
  type: "fish";
  swim: () => void;
}

function isBird(animal: Bird3 | Fish3): animal is Bird3 {
  return animal.type === "bird";
}

function move3(animal: Bird3 | Fish3) {
  if (isBird(animal)) {
    animal.fly();
  } else {
    animal.swim();
  }
}

// 5. 判别联合类型（Discriminated Union）
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; sideLength: number }
  | { kind: "rectangle"; width: number; height: number };

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    case "rectangle":
      return shape.width * shape.height;
  }
}
```

## 常见面试题

### 1. any 和 unknown 的区别？

<details>
<summary>点击查看答案</summary>

**any**：
- 关闭类型检查
- 可以赋值给任何类型
- 可以调用任何方法（不安全）

**unknown**：
- 类型安全的 any
- 不能直接赋值给其他类型（除了 any 和 unknown）
- 不能直接调用方法，需要类型检查或断言

```typescript
// any
let a: any = 10;
a = "string";
a.toFixed(); // ✅ 编译通过（可能运行时错误）

let b: string = a; // ✅ OK

// unknown
let x: unknown = 10;
x = "string";
// x.toFixed(); // ❌ Error

// let y: string = x; // ❌ Error

// 需要类型检查
if (typeof x === "string") {
  let y: string = x; // ✅ OK
}
```

**选择建议**：
- 尽量避免使用 `any`
- 如果不确定类型，使用 `unknown`
- 使用类型守卫进行类型检查
</details>

### 2. interface 和 type 的区别？

<details>
<summary>点击查看答案</summary>

**相同点**：
- 都可以描述对象的形状
- 都支持继承/扩展

**主要区别**：

| 特性 | Interface | Type |
|------|-----------|------|
| 扩展方式 | extends | & (交叉类型) |
| 声明合并 | ✅ 支持 | ❌ 不支持 |
| 联合类型 | ❌ 不支持 | ✅ 支持 |
| 元组 | ❌ 不方便 | ✅ 支持 |
| 映射类型 | ❌ 不支持 | ✅ 支持 |

```typescript
// 声明合并（只有 interface 支持）
interface User {
  name: string;
}

interface User {
  age: number;
}

const user: User = {
  name: "Alice",
  age: 25
};

// 联合类型（只有 type 支持）
type Status = "success" | "error";

// 映射类型（只有 type 支持）
type ReadonlyUser = {
  readonly [K in keyof User]: User[K];
};
```

**选择建议**：
- 定义对象结构 → 优先使用 `interface`
- 联合类型、工具类型 → 使用 `type`
- 需要声明合并 → 使用 `interface`
</details>

### 3. never 类型的使用场景？

<details>
<summary>点击查看答案</summary>

**never 表示永远不会发生的值**

使用场景：

1. **永远不返回的函数**
```typescript
function error(message: string): never {
  throw new Error(message);
}

function infiniteLoop(): never {
  while (true) {}
}
```

2. **穷尽性检查**
```typescript
type Shape = Circle | Square;

function getArea(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    default:
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
  }
}
```

3. **过滤联合类型**
```typescript
type NonNullable<T> = T extends null | undefined ? never : T;

type A = NonNullable<string | null>; // string
```

4. **不可能的交叉类型**
```typescript
type A = string & number; // never
```
</details>

### 4. 什么时候使用类型断言？

<details>
<summary>点击查看答案</summary>

**适用场景**：

1. **你比 TypeScript 更了解类型**
```typescript
const canvas = document.getElementById("canvas") as HTMLCanvasElement;
```

2. **从 any 类型转换**
```typescript
const data: any = JSON.parse(jsonString);
const user = data as User;
```

3. **const 断言**
```typescript
const config = {
  host: "localhost",
  port: 3000
} as const;
```

**⚠️ 注意**：
- 类型断言不做运行时检查
- 滥用可能导致运行时错误
- 优先使用类型守卫

```typescript
// ❌ 不好：滥用类型断言
const value: unknown = "string";
const num = value as number;
num.toFixed(); // 运行时错误！

// ✅ 好：使用类型守卫
if (typeof value === "number") {
  num.toFixed();
}
```
</details>

### 5. 如何定义一个只读对象？

<details>
<summary>点击查看答案</summary>

```typescript
// 方式 1: readonly 修饰符
interface User {
  readonly id: number;
  readonly name: string;
}

// 方式 2: Readonly 工具类型
type ReadonlyUser = Readonly<User>;

// 方式 3: as const 断言
const user = {
  id: 1,
  name: "Alice"
} as const;

// 方式 4: ReadonlyArray
const numbers: ReadonlyArray<number> = [1, 2, 3];
// 或
const numbers2: readonly number[] = [1, 2, 3];

// 深度只读
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object
    ? DeepReadonly<T[K]>
    : T[K];
};

interface Config {
  server: {
    host: string;
    port: number;
  };
}

type ReadonlyConfig = DeepReadonly<Config>;
```
</details>

## 最佳实践

### 1. 优先使用类型推断

```typescript
// ❌ 不必要的类型注解
const name: string = "Alice";
const age: number = 25;

// ✅ 让 TypeScript 推断
const name = "Alice";
const age = 25;

// ✅ 在函数参数和返回值使用类型注解
function greet(name: string): string {
  return `Hello, ${name}`;
}
```

### 2. 使用 strictNullChecks

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "strictNullChecks": true
  }
}

// ✅ 明确处理 null 和 undefined
function getLength(str: string | null): number {
  if (str === null) {
    return 0;
  }
  return str.length;
}
```

### 3. 避免使用 any

```typescript
// ❌ 不好
function process(data: any) {
  return data.value;
}

// ✅ 好：使用泛型
function process<T extends { value: unknown }>(data: T) {
  return data.value;
}

// ✅ 或使用 unknown
function process(data: unknown) {
  if (isValidData(data)) {
    return data.value;
  }
}
```

### 4. 使用字面量类型而不是枚举

```typescript
// ❌ 不推荐：枚举
enum Status {
  Pending,
  Approved,
  Rejected
}

// ✅ 推荐：字面量类型
type Status = "pending" | "approved" | "rejected";

// 或使用 const 对象
const STATUS = {
  Pending: "pending",
  Approved: "approved",
  Rejected: "rejected"
} as const;

type Status = typeof STATUS[keyof typeof STATUS];
```

