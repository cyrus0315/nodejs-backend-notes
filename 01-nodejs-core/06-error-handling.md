# 错误处理

## 错误类型

### 1. 标准错误类型

```javascript
// Error - 基础错误类型
throw new Error('Something went wrong');

// TypeError - 类型错误
const num = null;
num.toFixed(2); // TypeError: Cannot read property 'toFixed' of null

// ReferenceError - 引用错误
console.log(undefinedVariable); // ReferenceError: undefinedVariable is not defined

// SyntaxError - 语法错误
eval('var a = ;'); // SyntaxError: Unexpected token ';'

// RangeError - 范围错误
new Array(-1); // RangeError: Invalid array length

// URIError - URI 错误
decodeURIComponent('%'); // URIError: URI malformed
```

### 2. 自定义错误类型

```javascript
// 基础自定义错误
class CustomError extends Error {
  constructor(message) {
    super(message);
    this.name = 'CustomError';
    Error.captureStackTrace(this, this.constructor);
  }
}

// 业务错误
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
    this.statusCode = 400;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends Error {
  constructor(resource) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
    this.statusCode = 404;
    Error.captureStackTrace(this, this.constructor);
  }
}

class UnauthorizedError extends Error {
  constructor(message = 'Unauthorized') {
    super(message);
    this.name = 'UnauthorizedError';
    this.statusCode = 401;
    Error.captureStackTrace(this, this.constructor);
  }
}

// 使用
function validateUser(user) {
  if (!user.email) {
    throw new ValidationError('Email is required', 'email');
  }
  if (!user.password) {
    throw new ValidationError('Password is required', 'password');
  }
}

async function getUser(id) {
  const user = await db.users.findById(id);
  if (!user) {
    throw new NotFoundError('User');
  }
  return user;
}
```

### 3. 操作错误 vs 程序错误

```javascript
// 操作错误（Operational Errors）- 可预期的错误
// - 网络连接失败
// - 数据库查询失败
// - 用户输入无效
// - 文件不存在
// 应该被捕获和处理

async function fetchData(url) {
  try {
    const response = await fetch(url);
    return await response.json();
  } catch (error) {
    // 操作错误：网络问题
    logger.error('Failed to fetch data:', error);
    throw new Error('Service temporarily unavailable');
  }
}

// 程序错误（Programmer Errors）- 代码 bug
// - 读取 undefined 的属性
// - 传递错误类型的参数
// - 数组越界
// 应该修复代码，而不是捕获

function calculateTotal(items) {
  // ❌ 程序错误：没有验证参数
  return items.reduce((sum, item) => sum + item.price, 0);
}

function calculateTotal(items) {
  // ✅ 防御性编程
  if (!Array.isArray(items)) {
    throw new TypeError('items must be an array');
  }
  
  return items.reduce((sum, item) => {
    if (typeof item.price !== 'number') {
      throw new TypeError('item.price must be a number');
    }
    return sum + item.price;
  }, 0);
}
```

## 同步错误处理

### Try-Catch

```javascript
// 基本用法
try {
  const result = riskyOperation();
  console.log(result);
} catch (error) {
  console.error('Error:', error.message);
} finally {
  // 无论是否有错误都会执行
  cleanup();
}

// 捕获特定错误类型
try {
  JSON.parse(invalidJSON);
} catch (error) {
  if (error instanceof SyntaxError) {
    console.error('Invalid JSON');
  } else {
    throw error; // 重新抛出未知错误
  }
}

// 嵌套 try-catch
try {
  const data = readFile();
  
  try {
    const parsed = JSON.parse(data);
    return parsed;
  } catch (parseError) {
    console.error('Failed to parse JSON:', parseError);
    return null;
  }
} catch (fileError) {
  console.error('Failed to read file:', fileError);
  throw fileError;
}
```

## 异步错误处理

### Callback 错误处理

```javascript
// Node.js 回调约定：error-first callback
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error reading file:', err);
    return;
  }
  console.log(data);
});

// 错误传播
function processFile(callback) {
  fs.readFile('file.txt', 'utf8', (err, data) => {
    if (err) {
      return callback(err); // 传播错误
    }
    
    try {
      const parsed = JSON.parse(data);
      callback(null, parsed); // 成功
    } catch (parseError) {
      callback(parseError); // 传播解析错误
    }
  });
}

// 使用
processFile((err, result) => {
  if (err) {
    console.error('Failed to process file:', err);
    return;
  }
  console.log('Result:', result);
});
```

### Promise 错误处理

```javascript
// 1. .catch() 方法
fetchData()
  .then(data => processData(data))
  .then(result => saveResult(result))
  .catch(error => {
    console.error('Error in promise chain:', error);
  });

// 2. .then() 的第二个参数
fetchData()
  .then(
    data => console.log('Success:', data),
    error => console.error('Error:', error)
  );

// 3. 错误恢复
fetchData()
  .catch(error => {
    console.error('Primary source failed:', error);
    return fetchBackupData(); // 尝试备用数据源
  })
  .then(data => {
    console.log('Got data:', data);
  });

// 4. finally
fetchData()
  .then(processData)
  .catch(handleError)
  .finally(() => {
    // 清理资源
    cleanup();
  });

// 5. 捕获特定错误
fetchData()
  .catch(error => {
    if (error.code === 'ENOTFOUND') {
      throw new Error('Network error');
    }
    if (error.statusCode === 404) {
      return null; // 404 不算错误
    }
    throw error; // 重新抛出其他错误
  });
```

### Async/Await 错误处理

```javascript
// 1. 基本 try-catch
async function fetchUser(id) {
  try {
    const user = await db.users.findById(id);
    return user;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
}

// 2. 多个 await 的错误处理
async function processData() {
  try {
    const user = await fetchUser(1);
    const posts = await fetchPosts(user.id);
    const comments = await fetchComments(posts[0].id);
    return { user, posts, comments };
  } catch (error) {
    console.error('Error in data processing:', error);
    throw error;
  }
}

// 3. 错误恢复
async function fetchDataWithFallback() {
  try {
    return await fetchPrimarySource();
  } catch (primaryError) {
    console.warn('Primary source failed, trying backup');
    try {
      return await fetchBackupSource();
    } catch (backupError) {
      console.error('Both sources failed');
      throw new Error('All data sources unavailable');
    }
  }
}

// 4. 部分错误处理
async function processMultipleItems(items) {
  const results = [];
  const errors = [];
  
  for (const item of items) {
    try {
      const result = await processItem(item);
      results.push(result);
    } catch (error) {
      errors.push({ item, error });
      // 继续处理其他项目
    }
  }
  
  return { results, errors };
}

// 5. Promise.allSettled 处理多个 Promise
async function fetchAllData() {
  const results = await Promise.allSettled([
    fetchUsers(),
    fetchPosts(),
    fetchComments()
  ]);
  
  results.forEach((result, index) => {
    if (result.status === 'fulfilled') {
      console.log(`Task ${index} succeeded:`, result.value);
    } else {
      console.error(`Task ${index} failed:`, result.reason);
    }
  });
}
```

### 错误处理工具函数

```javascript
// 1. 包装函数自动捕获错误
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

// Express 路由中使用
app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.users.findById(req.params.id);
  res.json(user);
}));

// 2. 安全的 JSON 解析
function safeJsonParse(str, defaultValue = null) {
  try {
    return JSON.parse(str);
  } catch (error) {
    console.error('JSON parse error:', error);
    return defaultValue;
  }
}

// 3. 重试机制
async function retry(fn, maxAttempts = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) {
        throw error;
      }
      console.log(`Attempt ${attempt} failed, retrying in ${delay}ms...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}

// 使用
const data = await retry(
  () => fetchData('https://api.example.com'),
  3,
  2000
);

// 4. 超时处理
function timeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), ms)
    )
  ]);
}

// 使用
try {
  const data = await timeout(fetchData(), 5000);
} catch (error) {
  if (error.message === 'Timeout') {
    console.error('Request timed out');
  }
}
```

## 全局错误处理

### 未捕获的异常

```javascript
// 同步代码中的未捕获异常
process.on('uncaughtException', (error, origin) => {
  console.error('Uncaught Exception:', error);
  console.error('Origin:', origin);
  
  // 记录错误
  logger.error({
    type: 'uncaughtException',
    error: error.message,
    stack: error.stack
  });
  
  // 优雅关闭
  gracefulShutdown();
});

// ⚠️ 注意：uncaughtException 后应该退出进程
// 应用状态可能已损坏
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error);
  process.exit(1);
});
```

### 未处理的 Promise Rejection

```javascript
// Promise rejection 未被捕获
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise);
  console.error('Reason:', reason);
  
  // 记录错误
  logger.error({
    type: 'unhandledRejection',
    reason: reason,
    stack: reason?.stack
  });
  
  // 未来的 Node.js 版本会自动退出
  // 建议主动处理
});

// Node.js 15+ 会在 unhandledRejection 时退出
// 可以通过参数禁用
// node --unhandled-rejections=warn app.js
```

### Express 错误处理中间件

```javascript
const express = require('express');
const app = express();

// 路由
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await db.users.findById(req.params.id);
    if (!user) {
      throw new NotFoundError('User');
    }
    res.json(user);
  } catch (error) {
    next(error); // 传递给错误处理中间件
  }
});

// 404 处理
app.use((req, res, next) => {
  res.status(404).json({
    error: 'Not Found',
    path: req.path
  });
});

// 错误处理中间件（必须有 4 个参数）
app.use((err, req, res, next) => {
  // 记录错误
  logger.error({
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method
  });
  
  // 发送错误响应
  const statusCode = err.statusCode || 500;
  
  res.status(statusCode).json({
    error: err.message,
    ...(process.env.NODE_ENV === 'development' && {
      stack: err.stack
    })
  });
});

// 集中错误处理
class ErrorHandler {
  handleError(error, req, res) {
    logger.error({
      error: error.message,
      stack: error.stack,
      url: req?.url,
      method: req?.method,
      userId: req?.user?.id
    });
    
    if (error instanceof ValidationError) {
      return res.status(400).json({
        error: error.message,
        field: error.field
      });
    }
    
    if (error instanceof NotFoundError) {
      return res.status(404).json({
        error: error.message
      });
    }
    
    if (error instanceof UnauthorizedError) {
      return res.status(401).json({
        error: error.message
      });
    }
    
    // 默认 500 错误
    res.status(500).json({
      error: 'Internal Server Error'
    });
  }
  
  isTrustedError(error) {
    return error instanceof CustomError;
  }
}

const errorHandler = new ErrorHandler();

app.use((err, req, res, next) => {
  errorHandler.handleError(err, req, res);
});
```

## 错误日志

### 使用 Winston

```javascript
const winston = require('winston');

// 创建 logger
const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    // 错误日志文件
    new winston.transports.File({
      filename: 'error.log',
      level: 'error'
    }),
    // 所有日志文件
    new winston.transports.File({
      filename: 'combined.log'
    })
  ]
});

// 开发环境输出到控制台
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

// 使用
try {
  await riskyOperation();
} catch (error) {
  logger.error('Operation failed', {
    error: error.message,
    stack: error.stack,
    context: { userId: 123 }
  });
}
```

### 结构化日志

```javascript
// 创建上下文 logger
function createLogger(context) {
  return {
    info: (message, meta = {}) => {
      logger.info(message, { ...context, ...meta });
    },
    error: (message, error, meta = {}) => {
      logger.error(message, {
        ...context,
        ...meta,
        error: error.message,
        stack: error.stack
      });
    },
    warn: (message, meta = {}) => {
      logger.warn(message, { ...context, ...meta });
    }
  };
}

// 使用
const requestLogger = createLogger({
  requestId: req.id,
  userId: req.user.id,
  ip: req.ip
});

requestLogger.error('Failed to process request', error, {
  action: 'updateUser'
});
```

## 错误监控

### Sentry 集成

```javascript
const Sentry = require('@sentry/node');

Sentry.init({
  dsn: 'your-sentry-dsn',
  environment: process.env.NODE_ENV,
  tracesSampleRate: 1.0
});

// Express 集成
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// 路由...

// 错误处理
app.use(Sentry.Handlers.errorHandler());

// 手动捕获错误
try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error, {
    tags: {
      section: 'payment'
    },
    user: {
      id: userId,
      username: username
    }
  });
  throw error;
}
```

## 最佳实践

### 1. 使用自定义错误类型

```javascript
// ✅ 好：清晰的错误类型
class DatabaseError extends Error {
  constructor(message, query) {
    super(message);
    this.name = 'DatabaseError';
    this.query = query;
  }
}

throw new DatabaseError('Query failed', 'SELECT * FROM users');

// ❌ 不好：通用错误
throw new Error('Something went wrong');
```

### 2. 错误应该包含足够的上下文

```javascript
// ✅ 好：包含上下文
throw new ValidationError(
  `Invalid email format: ${email}`,
  'email'
);

// ❌ 不好：信息不足
throw new Error('Invalid email');
```

### 3. 区分操作错误和程序错误

```javascript
// 操作错误：捕获并处理
try {
  await fetchUserData();
} catch (error) {
  logger.error('Failed to fetch user data', error);
  return defaultUserData;
}

// 程序错误：让它崩溃，修复代码
function calculateDiscount(price) {
  if (typeof price !== 'number') {
    throw new TypeError('price must be a number');
  }
  return price * 0.9;
}
```

### 4. 不要吞掉错误

```javascript
// ❌ 不好：吞掉错误
try {
  await riskyOperation();
} catch (error) {
  // 什么都不做
}

// ✅ 好：至少记录错误
try {
  await riskyOperation();
} catch (error) {
  logger.error('Operation failed:', error);
  // 或者重新抛出
  throw error;
}
```

### 5. 优雅关闭

```javascript
async function gracefulShutdown() {
  console.log('Starting graceful shutdown...');
  
  // 1. 停止接受新请求
  server.close(() => {
    console.log('HTTP server closed');
  });
  
  // 2. 完成正在处理的请求（给 30 秒）
  await new Promise((resolve) => {
    setTimeout(() => {
      console.log('Forcing shutdown');
      resolve();
    }, 30000);
  });
  
  // 3. 关闭数据库连接
  try {
    await db.close();
    console.log('Database connections closed');
  } catch (error) {
    console.error('Error closing database:', error);
  }
  
  // 4. 关闭其他资源
  await redis.quit();
  await messageQueue.close();
  
  // 5. 退出进程
  process.exit(0);
}

// 监听信号
process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

// 监听未捕获的错误
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error);
  gracefulShutdown();
});

process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled Rejection:', reason);
  gracefulShutdown();
});
```

## 常见面试题

### 1. Error-first callback 是什么？

<details>
<summary>点击查看答案</summary>

**定义**：Node.js 的回调函数约定，第一个参数是错误对象，如果没有错误则为 `null`。

```javascript
// 标准格式
function callback(err, result) {
  if (err) {
    // 处理错误
    return;
  }
  // 处理结果
}

// 示例
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Error:', err);
    return;
  }
  console.log('Data:', data);
});
```

**为什么这样设计？**
1. 强制开发者先处理错误
2. 统一的错误处理模式
3. 错误可以通过回调链传播
</details>

### 2. 如何处理 async/await 中的多个错误？

<details>
<summary>点击查看答案</summary>

```javascript
// 方法 1: 多个 try-catch
async function method1() {
  let user, posts;
  
  try {
    user = await fetchUser();
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
  
  try {
    posts = await fetchPosts(user.id);
  } catch (error) {
    console.error('Failed to fetch posts:', error);
    posts = []; // 使用默认值
  }
  
  return { user, posts };
}

// 方法 2: Promise.allSettled
async function method2() {
  const [userResult, postsResult] = await Promise.allSettled([
    fetchUser(),
    fetchPosts()
  ]);
  
  const user = userResult.status === 'fulfilled' 
    ? userResult.value 
    : null;
    
  const posts = postsResult.status === 'fulfilled'
    ? postsResult.value
    : [];
  
  return { user, posts };
}

// 方法 3: 包装函数
function to(promise) {
  return promise
    .then(data => [null, data])
    .catch(err => [err, null]);
}

async function method3() {
  const [userErr, user] = await to(fetchUser());
  if (userErr) {
    console.error('Failed to fetch user:', userErr);
    throw userErr;
  }
  
  const [postsErr, posts] = await to(fetchPosts(user.id));
  if (postsErr) {
    console.error('Failed to fetch posts:', postsErr);
    // 继续执行，使用默认值
  }
  
  return { user, posts: posts || [] };
}
```
</details>

### 3. process.on('uncaughtException') 后应该做什么？

<details>
<summary>点击查看答案</summary>

**应该退出进程**，因为：

1. 应用状态可能已损坏
2. 可能存在内存泄漏
3. 资源可能未正确清理

```javascript
process.on('uncaughtException', (error) => {
  // 1. 记录错误
  logger.error('Uncaught Exception:', {
    error: error.message,
    stack: error.stack
  });
  
  // 2. 发送告警
  alerting.send('Critical: Uncaught Exception', error);
  
  // 3. 优雅关闭（如果可能）
  gracefulShutdown().finally(() => {
    // 4. 退出进程
    process.exit(1);
  });
  
  // 5. 强制退出（如果优雅关闭失败）
  setTimeout(() => {
    process.abort();
  }, 10000);
});
```

**更好的做法**：
- 使用 Cluster 或 PM2，让进程管理器重启应用
- 修复代码，避免未捕获的异常
</details>

### 4. 如何在 Promise 链中传播错误？

<details>
<summary>点击查看答案</summary>

```javascript
// 自动传播：不处理就会传给下一个 .catch()
fetchData()
  .then(data => processData(data))  // 如果出错，跳过后续 .then()
  .then(result => saveResult(result))
  .catch(error => {
    // 捕获所有错误
    console.error('Error in chain:', error);
  });

// 手动传播：重新抛出
fetchData()
  .catch(error => {
    logger.error('Fetch failed:', error);
    throw error; // 传播给下一个 .catch()
  })
  .then(processData)
  .catch(error => {
    console.error('Final error handler:', error);
  });

// 转换错误
fetchData()
  .catch(error => {
    if (error.code === 'ENOTFOUND') {
      throw new Error('Network error');
    }
    throw error;
  });

// 恢复错误（不传播）
fetchData()
  .catch(error => {
    console.error('Error:', error);
    return defaultData; // 返回默认值，不再是错误
  })
  .then(data => {
    // data 是 fetchData 的结果或 defaultData
  });
```
</details>

### 5. 如何处理 EventEmitter 的错误？

<details>
<summary>点击查看答案</summary>

```javascript
const EventEmitter = require('events');

const emitter = new EventEmitter();

// 方法 1: 监听 'error' 事件
emitter.on('error', (error) => {
  console.error('Error event:', error);
});

// 如果没有 'error' 监听器，会抛出未捕获的异常
emitter.emit('error', new Error('Something went wrong'));

// 方法 2: 使用 try-catch（在监听器内部）
emitter.on('data', (data) => {
  try {
    processData(data);
  } catch (error) {
    emitter.emit('error', error);
  }
});

// 方法 3: 使用 async 监听器
emitter.on('async-data', async (data) => {
  try {
    await processDataAsync(data);
  } catch (error) {
    emitter.emit('error', error);
  }
});

// 方法 4: 包装 EventEmitter
class SafeEmitter extends EventEmitter {
  emit(event, ...args) {
    if (event === 'error') {
      return super.emit(event, ...args);
    }
    
    try {
      return super.emit(event, ...args);
    } catch (error) {
      this.emit('error', error);
    }
  }
}
```
</details>

## 练习题

1. 实现一个支持重试和超时的 fetch 函数
2. 实现一个错误处理中间件，支持不同的错误类型
3. 实现一个 Promise 包装器，将 error-first callback 转为 Promise
4. 实现一个断路器（Circuit Breaker）模式
5. 实现一个全局错误收集器

<details>
<summary>点击查看部分答案</summary>

```javascript
// 1. 支持重试和超时的 fetch
async function resilientFetch(url, options = {}) {
  const {
    maxRetries = 3,
    timeout = 5000,
    retryDelay = 1000
  } = options;
  
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(() => controller.abort(), timeout);
      
      const response = await fetch(url, {
        ...options,
        signal: controller.signal
      });
      
      clearTimeout(timeoutId);
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }
      
      return await response.json();
    } catch (error) {
      const isLastAttempt = attempt === maxRetries;
      
      if (isLastAttempt) {
        throw error;
      }
      
      console.log(`Attempt ${attempt} failed, retrying...`);
      await new Promise(resolve => setTimeout(resolve, retryDelay));
    }
  }
}

// 4. 断路器模式
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.nextAttempt = Date.now();
  }
  
  async call(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failureCount++;
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.log('Circuit breaker is now OPEN');
    }
  }
}

// 使用
const breaker = new CircuitBreaker(fetchData, {
  failureThreshold: 3,
  resetTimeout: 30000
});

try {
  const data = await breaker.call();
} catch (error) {
  console.error('Call failed:', error);
}
```
</details>

