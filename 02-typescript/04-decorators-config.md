# 装饰器与配置

## 装饰器 (Decorators)

装饰器是一种特殊的声明，可以附加到类、方法、访问器、属性或参数上。

### 启用装饰器

```json
// tsconfig.json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

### 类装饰器

```typescript
// 基本类装饰器
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

@sealed
class Greeter {
  greeting: string;
  constructor(message: string) {
    this.greeting = message;
  }
}

// 装饰器工厂
function classDecorator(value: string) {
  return function (constructor: Function) {
    console.log(`Class decorated with: ${value}`);
  };
}

@classDecorator("MyClass")
class MyClass {}

// 修改类定义
function addTimestamp<T extends { new (...args: any[]): {} }>(
  constructor: T
) {
  return class extends constructor {
    timestamp = new Date();
  };
}

@addTimestamp
class User {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

const user = new User("Alice");
console.log((user as any).timestamp); // Date对象

// 实战：日志装饰器
function Logger(prefix: string = "LOG") {
  return function<T extends { new (...args: any[]): {} }>(
    constructor: T
  ) {
    return class extends constructor {
      constructor(...args: any[]) {
        super(...args);
        console.log(`[${prefix}] Creating instance of ${constructor.name}`);
      }
    };
  };
}

@Logger("USER")
class Admin {
  constructor(public username: string) {}
}

const admin = new Admin("admin");
// 输出: [USER] Creating instance of Admin
```

### 方法装饰器

```typescript
// 基本方法装饰器
function log(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;
  
  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey} with args:`, args);
    const result = originalMethod.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };
  
  return descriptor;
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(2, 3);
// 输出:
// Calling add with args: [2, 3]
// Result: 5

// 性能监控装饰器
function measure(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;
  
  descriptor.value = async function (...args: any[]) {
    const start = performance.now();
    const result = await originalMethod.apply(this, args);
    const end = performance.now();
    console.log(`${propertyKey} took ${end - start}ms`);
    return result;
  };
  
  return descriptor;
}

class DataService {
  @measure
  async fetchData() {
    // 模拟异步操作
    await new Promise(resolve => setTimeout(resolve, 100));
    return "data";
  }
}

// 错误处理装饰器
function catchErrors(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalMethod = descriptor.value;
  
  descriptor.value = async function (...args: any[]) {
    try {
      return await originalMethod.apply(this, args);
    } catch (error) {
      console.error(`Error in ${propertyKey}:`, error);
      throw error;
    }
  };
  
  return descriptor;
}

class UserService {
  @catchErrors
  @measure
  async getUser(id: string) {
    // 可能抛出错误的操作
    return { id, name: "Alice" };
  }
}
```

### 访问器装饰器

```typescript
// 访问器装饰器
function configurable(value: boolean) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    descriptor.configurable = value;
  };
}

class Point {
  private _x: number;
  private _y: number;
  
  constructor(x: number, y: number) {
    this._x = x;
    this._y = y;
  }
  
  @configurable(false)
  get x() {
    return this._x;
  }
  
  @configurable(false)
  get y() {
    return this._y;
  }
}

// 验证装饰器
function validate(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  const originalSet = descriptor.set;
  
  if (originalSet) {
    descriptor.set = function (value: any) {
      if (typeof value !== "number" || value < 0) {
        throw new Error(`${propertyKey} must be a positive number`);
      }
      originalSet.call(this, value);
    };
  }
}

class Product {
  private _price: number = 0;
  
  @validate
  set price(value: number) {
    this._price = value;
  }
  
  get price() {
    return this._price;
  }
}
```

### 属性装饰器

```typescript
// 属性装饰器
function format(formatString: string) {
  return function (target: any, propertyKey: string) {
    let value: string;
    
    const getter = function () {
      return value;
    };
    
    const setter = function (newVal: string) {
      value = formatString.replace("%s", newVal);
    };
    
    Object.defineProperty(target, propertyKey, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true
    });
  };
}

class Greeter {
  @format("Hello, %s!")
  greeting: string;
}

const greeter = new Greeter();
greeter.greeting = "World";
console.log(greeter.greeting); // "Hello, World!"

// 验证装饰器
import "reflect-metadata";

const requiredMetadataKey = Symbol("required");

function required(target: Object, propertyKey: string | symbol) {
  let existingRequiredProperties: string[] =
    Reflect.getOwnMetadata(requiredMetadataKey, target) || [];
  
  existingRequiredProperties.push(propertyKey as string);
  
  Reflect.defineMetadata(
    requiredMetadataKey,
    existingRequiredProperties,
    target
  );
}

function validate(target: any) {
  let requiredProperties: string[] =
    Reflect.getOwnMetadata(requiredMetadataKey, target) || [];
  
  for (let propertyKey of requiredProperties) {
    if (!target[propertyKey]) {
      throw new Error(`Property ${propertyKey} is required`);
    }
  }
}

class User {
  @required
  name: string;
  
  @required
  email: string;
  
  age?: number;
}

const user = new User();
// validate(user); // 会抛出错误
```

### 参数装饰器

```typescript
// 参数装饰器
const requiredMetadataKey = Symbol("required");

function required(
  target: Object,
  propertyKey: string | symbol,
  parameterIndex: number
) {
  let existingRequiredParameters: number[] =
    Reflect.getOwnMetadata(requiredMetadataKey, target, propertyKey) || [];
  
  existingRequiredParameters.push(parameterIndex);
  
  Reflect.defineMetadata(
    requiredMetadataKey,
    existingRequiredParameters,
    target,
    propertyKey
  );
}

function validateParams(
  target: any,
  propertyName: string,
  descriptor: PropertyDescriptor
) {
  const method = descriptor.value;
  
  descriptor.value = function (...args: any[]) {
    const requiredParameters: number[] =
      Reflect.getOwnMetadata(requiredMetadataKey, target, propertyName) || [];
    
    for (let parameterIndex of requiredParameters) {
      if (
        parameterIndex >= args.length ||
        args[parameterIndex] === undefined
      ) {
        throw new Error(
          `Missing required argument at position ${parameterIndex}`
        );
      }
    }
    
    return method.apply(this, args);
  };
  
  return descriptor;
}

class Greeter {
  @validateParams
  greet(@required name: string) {
    return `Hello, ${name}!`;
  }
}

const greeter = new Greeter();
greeter.greet("World"); // ✅ OK
// greeter.greet(); // ❌ Error
```

### NestJS 风格的装饰器

```typescript
// 路由装饰器
function Controller(prefix: string = "") {
  return function (target: Function) {
    Reflect.defineMetadata("prefix", prefix, target);
  };
}

function Get(path: string = "") {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    Reflect.defineMetadata("path", path, target, propertyKey);
    Reflect.defineMetadata("method", "GET", target, propertyKey);
  };
}

function Post(path: string = "") {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    Reflect.defineMetadata("path", path, target, propertyKey);
    Reflect.defineMetadata("method", "POST", target, propertyKey);
  };
}

// 参数装饰器
function Param(name: string) {
  return function (
    target: Object,
    propertyKey: string | symbol,
    parameterIndex: number
  ) {
    const existingParams =
      Reflect.getOwnMetadata("params", target, propertyKey) || {};
    
    existingParams[parameterIndex] = { type: "param", name };
    
    Reflect.defineMetadata("params", existingParams, target, propertyKey);
  };
}

function Body() {
  return function (
    target: Object,
    propertyKey: string | symbol,
    parameterIndex: number
  ) {
    const existingParams =
      Reflect.getOwnMetadata("params", target, propertyKey) || {};
    
    existingParams[parameterIndex] = { type: "body" };
    
    Reflect.defineMetadata("params", existingParams, target, propertyKey);
  };
}

// 使用
@Controller("/users")
class UserController {
  @Get("/:id")
  getUser(@Param("id") id: string) {
    return { id, name: "Alice" };
  }
  
  @Post()
  createUser(@Body() data: any) {
    return { id: "1", ...data };
  }
}
```

## TypeScript 配置

### tsconfig.json 详解

```json
{
  "compilerOptions": {
    /* 基本选项 */
    "target": "ES2020",                      // 目标 JavaScript 版本
    "module": "commonjs",                    // 模块系统
    "lib": ["ES2020"],                       // 包含的库文件
    "outDir": "./dist",                      // 输出目录
    "rootDir": "./src",                      // 源代码目录
    
    /* 严格检查选项 */
    "strict": true,                          // 启用所有严格检查
    "noImplicitAny": true,                   // 不允许隐式 any
    "strictNullChecks": true,                // 严格的 null 检查
    "strictFunctionTypes": true,             // 严格的函数类型检查
    "strictBindCallApply": true,             // 严格的 bind/call/apply 检查
    "strictPropertyInitialization": true,    // 严格的属性初始化
    "noImplicitThis": true,                  // 不允许隐式 this
    "alwaysStrict": true,                    // 使用严格模式
    
    /* 额外检查 */
    "noUnusedLocals": true,                  // 检查未使用的局部变量
    "noUnusedParameters": true,              // 检查未使用的参数
    "noImplicitReturns": true,               // 函数必须有返回值
    "noFallthroughCasesInSwitch": true,      // switch 必须有 break
    "noUncheckedIndexedAccess": true,        // 索引访问可能 undefined
    
    /* 模块解析选项 */
    "moduleResolution": "node",              // Node.js 模块解析
    "baseUrl": "./",                         // 基础路径
    "paths": {                               // 路径映射
      "@/*": ["src/*"],
      "@utils/*": ["src/utils/*"]
    },
    "esModuleInterop": true,                 // ES 模块互操作
    "allowSyntheticDefaultImports": true,    // 允许默认导入
    "resolveJsonModule": true,               // 导入 JSON 文件
    
    /* Source Map */
    "sourceMap": true,                       // 生成 source map
    "inlineSourceMap": false,                // 内联 source map
    "declarationMap": true,                  // 生成声明文件的 source map
    
    /* 实验性选项 */
    "experimentalDecorators": true,          // 启用装饰器
    "emitDecoratorMetadata": true,           // 生成装饰器元数据
    
    /* 高级选项 */
    "skipLibCheck": true,                    // 跳过库文件检查
    "forceConsistentCasingInFileNames": true,// 强制文件名大小写一致
    "declaration": true,                     // 生成声明文件
    "declarationDir": "./dist/types",        // 声明文件输出目录
    "removeComments": true,                  // 移除注释
    "importHelpers": true,                   // 从 tslib 导入辅助函数
    "incremental": true,                     // 增量编译
    "tsBuildInfoFile": "./.tsbuildinfo"      // 增量编译信息文件
  },
  
  /* 包含和排除 */
  "include": [
    "src/**/*"
  ],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.spec.ts"
  ],
  
  /* 项目引用 */
  "references": [
    { "path": "./tsconfig.node.json" }
  ]
}
```

### 不同场景的配置

#### Node.js 后端配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

#### ESM 配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ES2020",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true
  }
}
```

#### 库开发配置

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "module": "commonjs",
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "removeComments": true,
    "importHelpers": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

### 路径映射

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"],
      "@config/*": ["src/config/*"]
    }
  }
}
```

```typescript
// 使用路径映射
import { User } from "@types/user";
import { logger } from "@utils/logger";
import { config } from "@config/database";
```

### 项目引用 (Project References)

```json
// tsconfig.base.json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true
  }
}

// packages/core/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}

// packages/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "references": [
    { "path": "../core" }
  ],
  "include": ["src/**/*"]
}
```

## 常见面试题

### 1. 装饰器的执行顺序是什么？

<details>
<summary>点击查看答案</summary>

**执行顺序**：

1. **参数装饰器**
2. **方法/访问器/属性装饰器**
3. **类装饰器**

在同一个声明上有多个装饰器时，从下到上执行。

```typescript
function first() {
  console.log("first(): factory evaluated");
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    console.log("first(): called");
  };
}

function second() {
  console.log("second(): factory evaluated");
  return function (target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    console.log("second(): called");
  };
}

class ExampleClass {
  @first()
  @second()
  method() {}
}

// 输出:
// first(): factory evaluated
// second(): factory evaluated
// second(): called
// first(): called
```

**完整顺序示例**：

```typescript
function classDecorator(target: Function) {
  console.log("4. Class decorator");
}

function methodDecorator(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  console.log("3. Method decorator");
}

function propertyDecorator(target: any, propertyKey: string) {
  console.log("2. Property decorator");
}

function parameterDecorator(target: Object, propertyKey: string | symbol, parameterIndex: number) {
  console.log("1. Parameter decorator");
}

@classDecorator
class Example {
  @propertyDecorator
  prop: string;
  
  @methodDecorator
  method(@parameterDecorator param: string) {}
}

// 输出顺序：
// 1. Parameter decorator
// 2. Property decorator
// 3. Method decorator
// 4. Class decorator
```
</details>

### 2. tsconfig.json 中 strict 包含哪些选项？

<details>
<summary>点击查看答案</summary>

`"strict": true` 等价于启用以下所有选项：

```json
{
  "compilerOptions": {
    "noImplicitAny": true,              // 不允许隐式 any
    "strictNullChecks": true,           // 严格 null 检查
    "strictFunctionTypes": true,        // 严格函数类型
    "strictBindCallApply": true,        // 严格 bind/call/apply
    "strictPropertyInitialization": true, // 严格属性初始化
    "noImplicitThis": true,             // 不允许隐式 this
    "alwaysStrict": true                // 使用严格模式
  }
}
```

**建议**：生产环境始终启用 `strict: true`
</details>

### 3. 如何实现一个日志装饰器？

<details>
<summary>点击查看答案</summary>

```typescript
// 方法日志装饰器
function Log(options?: { prefix?: string; logArgs?: boolean; logResult?: boolean }) {
  const { prefix = "LOG", logArgs = true, logResult = true } = options || {};
  
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const className = target.constructor.name;
      
      if (logArgs) {
        console.log(`[${prefix}] ${className}.${propertyKey} called with:`, args);
      }
      
      try {
        const result = await originalMethod.apply(this, args);
        
        if (logResult) {
          console.log(`[${prefix}] ${className}.${propertyKey} returned:`, result);
        }
        
        return result;
      } catch (error) {
        console.error(`[${prefix}] ${className}.${propertyKey} threw error:`, error);
        throw error;
      }
    };
    
    return descriptor;
  };
}

// 使用
class UserService {
  @Log({ prefix: "USER_SERVICE", logArgs: true, logResult: true })
  async getUser(id: string) {
    return { id, name: "Alice" };
  }
  
  @Log({ logResult: false })
  async updateUser(id: string, data: any) {
    // 更新用户
    return true;
  }
}
```
</details>

### 4. baseUrl 和 paths 的区别？

<details>
<summary>点击查看答案</summary>

**baseUrl**：
- 设置模块解析的基础目录
- 所有非相对路径导入都相对于此目录

**paths**：
- 路径映射，将模块路径映射到物理路径
- 必须与 `baseUrl` 一起使用

```json
{
  "compilerOptions": {
    "baseUrl": "./",           // 基础路径为项目根目录
    "paths": {
      "@/*": ["src/*"],        // @ 映射到 src 目录
      "@utils/*": ["src/utils/*"]
    }
  }
}
```

```typescript
// 使用
import { logger } from "@utils/logger";
// 实际路径: ./src/utils/logger

import { User } from "@/types/user";
// 实际路径: ./src/types/user
```

**注意**：运行时还需要配置模块解析器（如 ts-node, webpack, etc.）
</details>

### 5. 如何在运行时使用装饰器元数据？

<details>
<summary>点击查看答案</summary>

需要使用 `reflect-metadata` 库：

```typescript
import "reflect-metadata";

// 定义元数据键
const formatMetadataKey = Symbol("format");

// 装饰器
function format(formatString: string) {
  return Reflect.metadata(formatMetadataKey, formatString);
}

// 使用
class Greeter {
  @format("Hello, %s")
  greeting: string;
}

// 读取元数据
const formatString = Reflect.getMetadata(
  formatMetadataKey,
  new Greeter(),
  "greeting"
);

console.log(formatString); // "Hello, %s"
```

**常见元数据**：

```typescript
// 设计时类型信息
Reflect.getMetadata("design:type", target, propertyKey);       // 属性类型
Reflect.getMetadata("design:paramtypes", target, propertyKey); // 参数类型
Reflect.getMetadata("design:returntype", target, propertyKey); // 返回类型
```
</details>

## 最佳实践

### 1. 始终启用 strict 模式

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

### 2. 合理使用装饰器

```typescript
// ✅ 好：清晰的装饰器组合
@Controller("/users")
class UserController {
  @Get("/:id")
  @UseGuards(AuthGuard)
  @UsePipes(ValidationPipe)
  getUser(@Param("id") id: string) {
    return this.userService.findById(id);
  }
}
```

### 3. 使用路径映射简化导入

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### 4. 为不同环境创建不同配置

```bash
tsconfig.json           # 基础配置
tsconfig.build.json     # 生产构建
tsconfig.dev.json       # 开发环境
tsconfig.test.json      # 测试环境
```

