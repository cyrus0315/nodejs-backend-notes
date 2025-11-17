# Node.js 现代特性

Node.js 近年来引入了许多现代特性，极大提升了开发体验和性能。本文重点介绍 Node.js 16+ 的重要新特性。

## Fetch API（Node.js 18+）

Node.js 18 引入了全局 `fetch()` API，无需第三方库即可发起 HTTP 请求。

### 基础使用

```typescript
// GET 请求
const response = await fetch('https://api.example.com/data');
const data = await response.json();

// POST 请求
const response = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com'
  })
});

// 处理响应
if (response.ok) {
  const result = await response.json();
  console.log(result);
} else {
  console.error('Request failed:', response.status);
}
```

### 高级用法

```typescript
// 设置超时
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

try {
  const response = await fetch('https://api.example.com/slow', {
    signal: controller.signal
  });
  clearTimeout(timeoutId);
  
  const data = await response.json();
} catch (error) {
  if (error.name === 'AbortError') {
    console.error('Request timeout');
  }
}

// 流式响应
const response = await fetch('https://api.example.com/stream');
const reader = response.body?.getReader();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  
  console.log('Received chunk:', new TextDecoder().decode(value));
}

// 自定义 Headers
const response = await fetch('https://api.example.com/data', {
  headers: {
    'Authorization': 'Bearer token',
    'Accept': 'application/json',
    'User-Agent': 'MyApp/1.0'
  }
});

// FormData
const formData = new FormData();
formData.append('file', fileBlob);
formData.append('name', 'document.pdf');

const response = await fetch('https://api.example.com/upload', {
  method: 'POST',
  body: formData
});
```

### 与 axios 对比

```typescript
// axios（需要安装）
import axios from 'axios';
const { data } = await axios.get('https://api.example.com/data');

// fetch（内置）
const response = await fetch('https://api.example.com/data');
const data = await response.json();

// fetch 优势：
// ✅ 内置，无需依赖
// ✅ 标准 Web API
// ✅ 支持流式响应

// axios 优势：
// ✅ 更简洁的 API
// ✅ 自动 JSON 转换
// ✅ 请求/响应拦截器
// ✅ 更好的错误处理
```

## Test Runner（Node.js 18+）

内置测试运行器，无需 Jest 或 Mocha。

### 基础测试

```typescript
// test.js
import { test, describe } from 'node:test';
import assert from 'node:assert';

describe('Array', () => {
  test('should add element', () => {
    const arr = [1, 2, 3];
    arr.push(4);
    assert.strictEqual(arr.length, 4);
    assert.strictEqual(arr[3], 4);
  });
  
  test('should remove element', () => {
    const arr = [1, 2, 3];
    const removed = arr.pop();
    assert.strictEqual(removed, 3);
    assert.strictEqual(arr.length, 2);
  });
});

// 运行测试
// node --test test.js
```

### 异步测试

```typescript
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('async operation', async () => {
  const result = await fetchData();
  assert.equal(result.status, 'success');
});

test('promise rejection', async () => {
  await assert.rejects(
    async () => {
      throw new Error('Expected error');
    },
    {
      name: 'Error',
      message: 'Expected error'
    }
  );
});

test('with timeout', { timeout: 5000 }, async () => {
  await longRunningOperation();
});
```

### 测试钩子

```typescript
import { test, before, after, beforeEach, afterEach } from 'node:test';

before(() => {
  console.log('Run once before all tests');
});

after(() => {
  console.log('Run once after all tests');
});

beforeEach(() => {
  console.log('Run before each test');
});

afterEach(() => {
  console.log('Run after each test');
});

test('test 1', () => {
  // ...
});

test('test 2', () => {
  // ...
});
```

### Mock 和 Spy

```typescript
import { test, mock } from 'node:test';
import assert from 'node:assert/strict';

test('mock function', () => {
  const fn = mock.fn();
  
  fn('arg1', 'arg2');
  fn('arg3');
  
  assert.strictEqual(fn.mock.calls.length, 2);
  assert.deepStrictEqual(fn.mock.calls[0].arguments, ['arg1', 'arg2']);
  assert.deepStrictEqual(fn.mock.calls[1].arguments, ['arg3']);
});

test('mock method', () => {
  const obj = {
    method: () => 'original'
  };
  
  mock.method(obj, 'method', () => 'mocked');
  
  assert.strictEqual(obj.method(), 'mocked');
});

test('mock timer', () => {
  mock.timers.enable({ apis: ['setTimeout'] });
  
  let called = false;
  setTimeout(() => {
    called = true;
  }, 1000);
  
  mock.timers.tick(1000);
  assert.strictEqual(called, true);
  
  mock.timers.reset();
});
```

### 覆盖率报告

```bash
# 运行测试并生成覆盖率报告
node --test --experimental-test-coverage

# 指定测试文件
node --test tests/**/*.test.js

# Watch 模式
node --test --watch
```

## Watch Mode（Node.js 18.11+）

自动监听文件变化并重启应用。

```bash
# 开发模式
node --watch server.js

# 监听特定文件
node --watch --watch-path=./src server.js

# 结合 test runner
node --test --watch
```

```typescript
// 实用配置
// package.json
{
  "scripts": {
    "dev": "node --watch src/index.js",
    "test:watch": "node --test --watch"
  }
}
```

## Web Streams API（Node.js 18+）

标准的 Web Streams API，更好的流处理。

### ReadableStream

```typescript
// 创建可读流
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue('chunk 1');
    controller.enqueue('chunk 2');
    controller.close();
  }
});

// 读取流
const reader = stream.getReader();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  console.log('Received:', value);
}

// 从 fetch 响应获取流
const response = await fetch('https://api.example.com/large-file');
const stream = response.body;

for await (const chunk of stream) {
  console.log('Chunk size:', chunk.length);
}
```

### WritableStream

```typescript
import { createWriteStream } from 'node:fs';

// 创建可写流
const fileStream = createWriteStream('output.txt');

const writableStream = new WritableStream({
  write(chunk) {
    fileStream.write(chunk);
  },
  close() {
    fileStream.end();
  },
  abort(reason) {
    fileStream.destroy(reason);
  }
});

// 写入数据
const writer = writableStream.getWriter();
await writer.write('Hello ');
await writer.write('World');
await writer.close();
```

### TransformStream

```typescript
// 转换流
const transformStream = new TransformStream({
  transform(chunk, controller) {
    // 转换为大写
    const transformed = chunk.toString().toUpperCase();
    controller.enqueue(transformed);
  }
});

// 使用管道
const response = await fetch('https://api.example.com/data');
const transformedStream = response.body
  .pipeThrough(transformStream);

// 读取转换后的数据
for await (const chunk of transformedStream) {
  console.log(chunk);
}
```

## Web Crypto API（Node.js 15+）

标准的 Web Crypto API。

```typescript
import { webcrypto } from 'node:crypto';
const { crypto } = webcrypto;

// 生成随机数
const randomBytes = crypto.getRandomValues(new Uint8Array(16));

// 生成密钥对
const keyPair = await crypto.subtle.generateKey(
  {
    name: 'RSASSA-PKCS1-v1_5',
    modulusLength: 2048,
    publicExponent: new Uint8Array([1, 0, 1]),
    hash: 'SHA-256'
  },
  true,
  ['sign', 'verify']
);

// 签名
const data = new TextEncoder().encode('Hello World');
const signature = await crypto.subtle.sign(
  'RSASSA-PKCS1-v1_5',
  keyPair.privateKey,
  data
);

// 验证签名
const isValid = await crypto.subtle.verify(
  'RSASSA-PKCS1-v1_5',
  keyPair.publicKey,
  signature,
  data
);

// 哈希
const hashBuffer = await crypto.subtle.digest('SHA-256', data);
const hashArray = Array.from(new Uint8Array(hashBuffer));
const hashHex = hashArray.map(b => b.toString(16).padStart(2, '0')).join('');

// AES 加密
const key = await crypto.subtle.generateKey(
  { name: 'AES-GCM', length: 256 },
  true,
  ['encrypt', 'decrypt']
);

const iv = crypto.getRandomValues(new Uint8Array(12));
const encrypted = await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv },
  key,
  data
);

// 解密
const decrypted = await crypto.subtle.decrypt(
  { name: 'AES-GCM', iv },
  key,
  encrypted
);
```

## Single Executable Applications（Node.js 19+）

将 Node.js 应用打包为单个可执行文件。

```bash
# 创建可执行文件
node --experimental-sea-config sea-config.json
```

```json
// sea-config.json
{
  "main": "app.js",
  "output": "sea-prep.blob",
  "disableExperimentalSEAWarning": true
}
```

```bash
# 1. 生成 blob
node --experimental-sea-config sea-config.json

# 2. 复制 node 二进制
cp $(command -v node) myapp

# 3. 注入 blob
npx postject myapp NODE_SEA_BLOB sea-prep.blob \
  --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2

# 4. 运行
./myapp
```

## 权限模型（Node.js 20+）

限制应用的文件系统和子进程访问。

```bash
# 限制文件系统访问
node --experimental-permission --allow-fs-read=/tmp app.js

# 限制子进程
node --experimental-permission --allow-child-process app.js

# 组合权限
node --experimental-permission \
  --allow-fs-read=/tmp \
  --allow-fs-write=/tmp/logs \
  --allow-child-process \
  app.js
```

```typescript
// 检查权限
import { permission } from 'node:process';

if (permission.has('fs.read', '/etc/passwd')) {
  // 可以读取
} else {
  console.error('No permission to read /etc/passwd');
}

// 运行时检查
try {
  require('fs').readFileSync('/etc/passwd');
} catch (error) {
  if (error.code === 'ERR_ACCESS_DENIED') {
    console.error('Permission denied');
  }
}
```

## 其他重要特性

### 1. 内置 .env 支持（Node.js 20.6+）

```bash
# 不再需要 dotenv 包
node --env-file=.env app.js
```

```typescript
// .env
DATABASE_URL=postgresql://localhost/mydb
API_KEY=secret123

// app.js
console.log(process.env.DATABASE_URL);  // 自动加载
console.log(process.env.API_KEY);
```

### 2. 导入 JSON 模块（Node.js 17.5+）

```typescript
// import assertions
import data from './data.json' assert { type: 'json' };

// Node.js 20.10+ (import attributes)
import data from './data.json' with { type: 'json' };

console.log(data);
```

### 3. Array.prototype 新方法（Node.js 20+）

```typescript
// findLast 和 findLastIndex
const array = [1, 2, 3, 4, 5];
const lastEven = array.findLast(x => x % 2 === 0);  // 4
const lastEvenIndex = array.findLastIndex(x => x % 2 === 0);  // 3

// toReversed, toSorted, toSpliced（不改变原数组）
const original = [3, 1, 4, 1, 5];
const reversed = original.toReversed();  // [5, 1, 4, 1, 3]
const sorted = original.toSorted();  // [1, 1, 3, 4, 5]
console.log(original);  // [3, 1, 4, 1, 5] 未改变

// with（替换元素）
const replaced = original.with(2, 99);  // [3, 1, 99, 1, 5]
```

### 4. V8 编译缓存（Node.js 18.8+）

```typescript
// 自动启用，提升启动速度
// 缓存存储在 node_modules/.cache/node

// 检查缓存
console.log(process.features.cached_builtins);
```

### 5. import.meta 增强

```typescript
// import.meta.url
console.log(import.meta.url);
// file:///path/to/module.js

// import.meta.resolve
const resolvedPath = import.meta.resolve('./relative/path');
console.log(resolvedPath);

// import.meta.dirname 和 filename（Node.js 20.11+）
console.log(import.meta.dirname);
console.log(import.meta.filename);
```

### 6. 网络检查（Node.js 19+）

```typescript
import { isIP, isIPv4, isIPv6 } from 'node:net';

console.log(isIP('127.0.0.1'));  // 4
console.log(isIP('::1'));  // 6
console.log(isIP('invalid'));  // 0

console.log(isIPv4('192.168.1.1'));  // true
console.log(isIPv6('2001:db8::1'));  // true
```

### 7. 原生 TypeScript 支持（实验性，Node.js 22+）

```bash
# 直接运行 TypeScript
node --experimental-strip-types app.ts

# 不需要 ts-node 或编译步骤
```

## 性能改进

### Node.js 18 性能改进

- ✅ V8 10.1：更快的解析和编译
- ✅ 更快的 HTTP server（基于 undici）
- ✅ 改进的垃圾回收

### Node.js 20 性能改进

- ✅ V8 11.3：更好的性能
- ✅ 更快的 URL 解析
- ✅ 优化的 Buffer 操作

### 基准测试

```typescript
// HTTP Server 性能对比
// Node.js 16: ~50,000 req/s
// Node.js 18: ~65,000 req/s
// Node.js 20: ~75,000 req/s
```

## 迁移建议

### 从 Node.js 16 → 18

```typescript
// 1. 使用内置 fetch 替代 axios/node-fetch
// ❌ Before
import axios from 'axios';
const { data } = await axios.get(url);

// ✅ After
const response = await fetch(url);
const data = await response.json();

// 2. 使用内置 test runner
// ❌ Before
import { expect } from 'chai';
import { describe, it } from 'mocha';

// ✅ After
import { test, describe } from 'node:test';
import assert from 'node:assert/strict';

// 3. 使用 Web Streams
// ❌ Before
const { Readable } = require('stream');

// ✅ After
const stream = new ReadableStream({...});
```

### 从 Node.js 18 → 20

```typescript
// 1. 使用 .env 支持
// ❌ Before
import 'dotenv/config';

// ✅ After
// node --env-file=.env app.js

// 2. 使用新的数组方法
// ❌ Before
const reversed = [...array].reverse();

// ✅ After
const reversed = array.toReversed();

// 3. 使用权限模型
// node --experimental-permission --allow-fs-read=. app.js
```

## 最佳实践

### 1. 使用现代 API

```typescript
// ✅ 使用 fetch 而不是 axios（如果功能足够）
// ✅ 使用内置 test runner（小项目）
// ✅ 使用 Web Streams API
// ✅ 使用 Web Crypto API
```

### 2. 开发工具配置

```json
// package.json
{
  "scripts": {
    "dev": "node --watch --env-file=.env src/index.js",
    "test": "node --test tests/**/*.test.js",
    "test:watch": "node --test --watch",
    "test:coverage": "node --test --experimental-test-coverage"
  }
}
```

### 3. tsconfig.json 配置

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022", "DOM"],  // 包含 DOM 以支持 fetch
    "types": ["node"]
  }
}
```

## 常见面试题

### 1. Node.js 18 引入了哪些重要特性？

<details>
<summary>点击查看答案</summary>

**主要特性**：

1. **Fetch API**：
   - 内置 HTTP 客户端
   - 基于 Web 标准
   - 无需第三方依赖

2. **Test Runner**：
   - 内置测试框架
   - 支持 describe/test/mock
   - 无需 Jest/Mocha

3. **Watch Mode**：
   - 自动重启
   - 提升开发体验

4. **Web Streams API**：
   - 标准流处理
   - 更好的性能

5. **性能改进**：
   - V8 10.1
   - 更快的 HTTP server
   - 优化的启动时间

**影响**：
- 减少外部依赖
- 更好的 Web 兼容性
- 提升开发体验
</details>

### 2. fetch() vs axios 如何选择？

<details>
<summary>点击查看答案</summary>

**fetch 优势**：
- ✅ 内置，无需依赖
- ✅ Web 标准 API
- ✅ 更小的包体积
- ✅ 支持流式响应

**axios 优势**：
- ✅ 更简洁的 API
- ✅ 自动 JSON 转换
- ✅ 请求/响应拦截器
- ✅ 更好的错误处理
- ✅ 进度跟踪
- ✅ 取消请求更方便

**选择建议**：
- 简单 HTTP 请求 → fetch
- 复杂场景（拦截器、进度）→ axios
- 新项目，功能足够 → fetch
- 已有 axios 依赖 → 保持 axios

**fetch 限制**：
```typescript
// fetch 不会对 HTTP 错误抛出异常
const response = await fetch(url);
if (!response.ok) {
  throw new Error(`HTTP ${response.status}`);
}

// axios 自动抛出异常
try {
  const { data } = await axios.get(url);
} catch (error) {
  if (error.response) {
    console.error('HTTP', error.response.status);
  }
}
```
</details>

### 3. Node.js 内置 test runner vs Jest？

<details>
<summary>点击查看答案</summary>

**内置 Test Runner 优势**：
- ✅ 无需安装依赖
- ✅ 更快的启动速度
- ✅ 原生 ESM 支持
- ✅ 更轻量

**Jest 优势**：
- ✅ 功能更丰富
- ✅ 更好的 mock 系统
- ✅ 快照测试
- ✅ 并行测试
- ✅ 更成熟的生态

**对比**：

| 特性 | Node Test | Jest |
|------|-----------|------|
| 安装 | 内置 | 需要安装 |
| 启动速度 | 快 | 较慢 |
| 功能丰富度 | 基础 | 丰富 |
| 生态 | 新 | 成熟 |
| 配置 | 简单 | 复杂 |

**选择建议**：
- 小型项目 → Node Test Runner
- 大型项目 → Jest
- 简单测试 → Node Test Runner
- 需要快照/复杂 mock → Jest
</details>

### 4. 如何理解 Single Executable Applications？

<details>
<summary>点击查看答案</summary>

**概念**：
将 Node.js 应用打包为单个可执行文件，无需安装 Node.js 运行时。

**优势**：
- ✅ 简化部署
- ✅ 保护源码
- ✅ 统一环境
- ✅ 独立运行

**使用场景**：
- CLI 工具
- 桌面应用
- 嵌入式系统
- 商业软件

**限制**：
- ❌ 文件大小较大（~50MB+）
- ❌ 功能仍在实验阶段
- ❌ 不支持动态 require
- ❌ 调试困难

**替代方案**：
- pkg（第三方）
- nexe
- vercel/ncc（打包为单文件）
</details>

### 5. Node.js 20 的权限模型有什么用？

<details>
<summary>点击查看答案</summary>

**目的**：
增强安全性，限制应用的系统访问。

**使用场景**：

1. **限制文件访问**：
```bash
node --experimental-permission \
  --allow-fs-read=/app/data \
  --allow-fs-write=/app/logs \
  app.js
```

2. **限制子进程**：
```bash
node --experimental-permission \
  --allow-child-process \
  app.js
```

3. **沙箱环境**：
```bash
# 运行不受信任的代码
node --experimental-permission \
  --allow-fs-read=. \
  untrusted.js
```

**好处**：
- ✅ 防止恶意代码访问敏感文件
- ✅ 限制第三方包的权限
- ✅ 符合最小权限原则

**类似机制**：
- Deno 的权限系统
- 浏览器的沙箱机制
</details>

### 6. Web Streams vs Node.js Streams 的区别？

<details>
<summary>点击查看答案</summary>

**Web Streams**（标准）：
```typescript
const stream = new ReadableStream({
  start(controller) {
    controller.enqueue('data');
    controller.close();
  }
});

const reader = stream.getReader();
const { value, done } = await reader.read();
```

**Node.js Streams**（传统）：
```typescript
const { Readable } = require('stream');

const stream = new Readable({
  read() {
    this.push('data');
    this.push(null);
  }
});

stream.on('data', chunk => {
  console.log(chunk);
});
```

**主要区别**：

| 特性 | Web Streams | Node Streams |
|------|-------------|--------------|
| 标准 | Web 标准 | Node.js 专有 |
| API | Promise-based | Event-based |
| 浏览器兼容 | ✅ | ❌ |
| 性能 | 较好 | 很好 |
| 生态 | 新 | 成熟 |

**选择建议**：
- 新代码，需要浏览器兼容 → Web Streams
- 已有 Node.js 代码 → Node Streams
- 文件操作 → Node Streams
- fetch 响应 → Web Streams
</details>

## 总结

Node.js 近年来的发展：

1. **更接近 Web 标准**：
   - Fetch API
   - Web Streams
   - Web Crypto

2. **内置更多功能**：
   - Test Runner
   - .env 支持
   - Watch Mode

3. **提升性能**：
   - 更快的 V8
   - 优化的 HTTP
   - 编译缓存

4. **增强安全**：
   - 权限模型
   - 更好的沙箱

5. **改善开发体验**：
   - 更快的启动
   - 更好的错误信息
   - TypeScript 支持

**建议**：
- 新项目使用 Node.js 20+
- 优先使用内置特性
- 减少外部依赖
- 关注性能和安全

