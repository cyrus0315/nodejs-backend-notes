# 性能优化

## 内存管理

### V8 内存结构

Node.js 使用 V8 引擎，内存分为几个区域：

```
┌─────────────────────────────┐
│     新生代 (New Space)        │  小对象，快速分配和回收
│  - From Space (Survivor)    │
│  - To Space (Survivor)      │
├─────────────────────────────┤
│     老生代 (Old Space)        │  存活时间长的对象
├─────────────────────────────┤
│     大对象空间 (Large Object) │  大于 1MB 的对象
├─────────────────────────────┤
│     代码空间 (Code Space)     │  JIT 编译后的代码
└─────────────────────────────┘
```

### 垃圾回收机制

```javascript
// 查看垃圾回收
node --expose-gc app.js

if (global.gc) {
  console.log('Before GC:', process.memoryUsage());
  global.gc();
  console.log('After GC:', process.memoryUsage());
}

// 监听 GC 事件
const v8 = require('v8');

v8.setFlagsFromString('--expose_gc');
const perf = require('perf_hooks').performance;

const obs = new perf.PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`GC ${entry.kind}: ${entry.duration}ms`);
  });
});

obs.observe({ entryTypes: ['gc'] });
```

### 内存泄漏检测

**常见内存泄漏原因**：

1. **全局变量**
```javascript
// ❌ 不好：意外的全局变量
function leak() {
  data = 'large data'; // 没有 var/let/const
}

// ✅ 好：使用严格模式
'use strict';
function noLeak() {
  const data = 'large data';
}
```

2. **闭包引用**
```javascript
// ❌ 不好：闭包持有大对象
function createLeak() {
  const largeData = new Array(1000000);
  
  return function() {
    console.log('closure');
    // largeData 无法被回收
  };
}

// ✅ 好：只保留需要的数据
function noLeak() {
  const largeData = new Array(1000000);
  const needed = largeData[0];
  
  return function() {
    console.log(needed);
  };
}
```

3. **未清理的定时器**
```javascript
// ❌ 不好：忘记清理定时器
class Component {
  constructor() {
    this.timer = setInterval(() => {
      this.update();
    }, 1000);
  }
}

// ✅ 好：清理定时器
class Component {
  constructor() {
    this.timer = setInterval(() => {
      this.update();
    }, 1000);
  }
  
  destroy() {
    clearInterval(this.timer);
  }
}
```

4. **事件监听器未移除**
```javascript
// ❌ 不好：事件监听器累积
function leak() {
  const emitter = getEventEmitter();
  emitter.on('event', handler);
}

// ✅ 好：移除监听器
function noLeak() {
  const emitter = getEventEmitter();
  emitter.on('event', handler);
  
  // 清理
  return () => {
    emitter.removeListener('event', handler);
  };
}

// ✅ 更好：使用 once
emitter.once('event', handler);
```

5. **缓存无限增长**
```javascript
// ❌ 不好：缓存无限增长
const cache = {};

function getData(key) {
  if (cache[key]) return cache[key];
  
  const data = fetchData(key);
  cache[key] = data; // 永远不会删除
  return data;
}

// ✅ 好：使用 LRU 缓存
const LRU = require('lru-cache');
const cache = new LRU({
  max: 500,
  maxAge: 1000 * 60 * 60 // 1 小时
});

function getData(key) {
  if (cache.has(key)) return cache.get(key);
  
  const data = fetchData(key);
  cache.set(key, data);
  return data;
}
```

### 内存泄漏检测工具

```javascript
// 1. heapdump - 堆快照
const heapdump = require('heapdump');

// 生成堆快照
heapdump.writeSnapshot('./heap-' + Date.now() + '.heapsnapshot');

// 2. memwatch-next - 内存监控
const memwatch = require('@airbnb/node-memwatch');

memwatch.on('leak', (info) => {
  console.error('Memory leak detected:', info);
});

memwatch.on('stats', (stats) => {
  console.log('GC stats:', stats);
});

// 3. clinic.js - 性能诊断
// npm install -g clinic
// clinic doctor -- node app.js
// clinic flame -- node app.js
// clinic bubbleprof -- node app.js

// 4. 0x - 火焰图
// npm install -g 0x
// 0x app.js
```

### 内存优化技巧

```javascript
// 1. 及时释放大对象
function processLargeData() {
  let largeArray = new Array(1000000).fill(0);
  
  // 处理数据
  const result = computeResult(largeArray);
  
  // 释放引用
  largeArray = null;
  
  return result;
}

// 2. 使用流处理大文件
const fs = require('fs');

// ❌ 不好：一次性读取
fs.readFile('large-file.txt', (err, data) => {
  // 整个文件加载到内存
});

// ✅ 好：使用流
const stream = fs.createReadStream('large-file.txt');
stream.on('data', (chunk) => {
  // 分块处理
});

// 3. 使用 Buffer.allocUnsafe 需谨慎
// ❌ 可能不安全：内容未初始化
const buf1 = Buffer.allocUnsafe(10);

// ✅ 安全：内容初始化为 0
const buf2 = Buffer.alloc(10);

// 4. 避免创建大量小对象
// ❌ 不好：大量小对象
for (let i = 0; i < 1000000; i++) {
  const obj = { id: i, name: 'user' + i };
  process(obj);
}

// ✅ 好：重用对象
const obj = { id: 0, name: '' };
for (let i = 0; i < 1000000; i++) {
  obj.id = i;
  obj.name = 'user' + i;
  process(obj);
}
```

## CPU 优化

### 避免阻塞事件循环

```javascript
// ❌ 不好：阻塞事件循环
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

app.get('/fib/:n', (req, res) => {
  const result = fibonacci(req.params.n); // 阻塞！
  res.json({ result });
});

// ✅ 好：使用 Worker Thread
const { Worker } = require('worker_threads');

app.get('/fib/:n', async (req, res) => {
  const result = await runInWorker({
    task: 'fibonacci',
    n: req.params.n
  });
  res.json({ result });
});

// ✅ 或使用分片处理
async function fibonacciAsync(n) {
  if (n <= 1) return n;
  
  let a = 0, b = 1;
  for (let i = 2; i <= n; i++) {
    [a, b] = [b, a + b];
    
    // 每 10000 次迭代让出控制权
    if (i % 10000 === 0) {
      await setImmediate(() => {});
    }
  }
  return b;
}
```

### 缓存计算结果

```javascript
// 1. 简单缓存
const cache = new Map();

function expensiveOperation(key) {
  if (cache.has(key)) {
    return cache.get(key);
  }
  
  const result = doExpensiveComputation(key);
  cache.set(key, result);
  return result;
}

// 2. 记忆化（Memoization）
function memoize(fn) {
  const cache = new Map();
  
  return function(...args) {
    const key = JSON.stringify(args);
    
    if (cache.has(key)) {
      return cache.get(key);
    }
    
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// 使用
const fibonacci = memoize(function(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

// 3. LRU 缓存
const LRU = require('lru-cache');

const cache = new LRU({
  max: 500,
  maxAge: 1000 * 60 * 60,
  updateAgeOnGet: true
});
```

### 优化正则表达式

```javascript
// ❌ 不好：灾难性回溯
const regex = /^(a+)+$/;
const input = 'a'.repeat(100) + 'b'; // 会非常慢

// ✅ 好：避免嵌套量词
const regex = /^a+$/;

// ✅ 提前编译正则
// ❌ 不好：每次都编译
function validate(str) {
  return /^[a-z]+$/.test(str);
}

// ✅ 好：编译一次
const PATTERN = /^[a-z]+$/;
function validate(str) {
  return PATTERN.test(str);
}
```

## 数据库优化

### 连接池

```javascript
// ✅ 使用连接池
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  database: 'mydb',
  max: 20,                    // 最大连接数
  idleTimeoutMillis: 30000,   // 空闲超时
  connectionTimeoutMillis: 2000
});

async function query(sql, params) {
  const client = await pool.connect();
  try {
    return await client.query(sql, params);
  } finally {
    client.release();
  }
}

// 优雅关闭
process.on('SIGTERM', async () => {
  await pool.end();
});
```

### 批量操作

```javascript
// ❌ 不好：逐个插入
for (const user of users) {
  await db.insert('users', user);
}

// ✅ 好：批量插入
await db.batchInsert('users', users, 100); // 每批 100 条

// ✅ PostgreSQL 批量插入
const values = users.map((u, i) => 
  `($${i*2+1}, $${i*2+2})`
).join(',');

await pool.query(
  `INSERT INTO users (name, email) VALUES ${values}`,
  users.flatMap(u => [u.name, u.email])
);
```

### 查询优化

```javascript
// 1. 只查询需要的字段
// ❌ 不好
const users = await db.select('*').from('users');

// ✅ 好
const users = await db
  .select('id', 'name', 'email')
  .from('users');

// 2. 使用索引
// ❌ 不好：未使用索引
await db.query('SELECT * FROM users WHERE email = ?', [email]);

// ✅ 好：确保 email 字段有索引
// CREATE INDEX idx_users_email ON users(email);

// 3. 避免 N+1 查询
// ❌ 不好：N+1 查询
const posts = await db.select('*').from('posts');
for (const post of posts) {
  post.author = await db
    .select('*')
    .from('users')
    .where('id', post.userId)
    .first();
}

// ✅ 好：使用 JOIN 或批量查询
const posts = await db
  .select('posts.*', 'users.name as authorName')
  .from('posts')
  .join('users', 'users.id', 'posts.userId');
```

## HTTP 性能优化

### 启用压缩

```javascript
const express = require('express');
const compression = require('compression');

const app = express();

// 启用 gzip 压缩
app.use(compression({
  level: 6,
  threshold: 1024,  // 只压缩大于 1KB 的响应
  filter: (req, res) => {
    // 自定义过滤逻辑
    return compression.filter(req, res);
  }
}));
```

### HTTP/2 Server Push

```javascript
const http2 = require('http2');
const fs = require('fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('key.pem'),
  cert: fs.readFileSync('cert.pem')
});

server.on('stream', (stream, headers) => {
  if (headers[':path'] === '/') {
    // Server Push
    stream.pushStream({ ':path': '/style.css' }, (err, pushStream) => {
      if (err) throw err;
      pushStream.respondWithFile('style.css');
    });
    
    stream.respondWithFile('index.html');
  }
});

server.listen(3000);
```

### 使用缓存

```javascript
const express = require('express');
const app = express();

// 1. HTTP 缓存头
app.get('/static/*', (req, res) => {
  res.set({
    'Cache-Control': 'public, max-age=31536000', // 1 年
    'ETag': generateETag(req.path)
  });
  
  res.sendFile(req.path);
});

// 2. 应用层缓存
const redis = require('redis');
const client = redis.createClient();

app.get('/api/users/:id', async (req, res) => {
  const cacheKey = `user:${req.params.id}`;
  
  // 先查缓存
  const cached = await client.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }
  
  // 查数据库
  const user = await db.users.findById(req.params.id);
  
  // 写入缓存
  await client.setex(cacheKey, 3600, JSON.stringify(user));
  
  res.json(user);
});

// 3. 内存缓存
const LRU = require('lru-cache');
const cache = new LRU({ max: 500, maxAge: 1000 * 60 * 5 });

app.get('/api/expensive', async (req, res) => {
  const key = 'expensive-data';
  
  if (cache.has(key)) {
    return res.json(cache.get(key));
  }
  
  const result = await expensiveOperation();
  cache.set(key, result);
  res.json(result);
});
```

### 连接复用

```javascript
const http = require('http');
const https = require('https');

// 启用 Keep-Alive
const agent = new https.Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10,
  timeout: 60000,
  keepAliveMsecs: 30000
});

// 使用
const axios = require('axios');
const client = axios.create({
  httpsAgent: agent
});

// 所有请求都会复用连接
await client.get('https://api.example.com/data');
```

## 性能监控

### 内置性能钩子

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// 1. 标记和测量
performance.mark('start');

// 执行操作
doSomething();

performance.mark('end');
performance.measure('operation', 'start', 'end');

const measure = performance.getEntriesByName('operation')[0];
console.log(`Duration: ${measure.duration}ms`);

// 2. 性能观察器
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`${entry.name}: ${entry.duration}ms`);
  });
});

obs.observe({ entryTypes: ['measure', 'function'] });

// 3. 函数性能测量
const wrapped = performance.timerify(expensiveFunction);
wrapped();
```

### 自定义性能指标

```javascript
class PerformanceMonitor {
  constructor() {
    this.metrics = new Map();
  }
  
  start(name) {
    this.metrics.set(name, {
      startTime: Date.now(),
      startCpu: process.cpuUsage(),
      startMem: process.memoryUsage()
    });
  }
  
  end(name) {
    const metric = this.metrics.get(name);
    if (!metric) return;
    
    const duration = Date.now() - metric.startTime;
    const cpu = process.cpuUsage(metric.startCpu);
    const mem = process.memoryUsage();
    
    const result = {
      duration,
      cpuUser: cpu.user / 1000,
      cpuSystem: cpu.system / 1000,
      memoryDelta: mem.heapUsed - metric.startMem.heapUsed
    };
    
    this.metrics.delete(name);
    return result;
  }
  
  async measure(name, fn) {
    this.start(name);
    try {
      return await fn();
    } finally {
      const metrics = this.end(name);
      console.log(`${name}:`, metrics);
    }
  }
}

// 使用
const monitor = new PerformanceMonitor();

await monitor.measure('database-query', async () => {
  return await db.query('SELECT * FROM users');
});
```

### APM 集成

```javascript
// 1. New Relic
require('newrelic');

// 2. Datadog
const tracer = require('dd-trace').init();

tracer.trace('database.query', () => {
  return db.query('SELECT * FROM users');
});

// 3. Elastic APM
const apm = require('elastic-apm-node').start({
  serviceName: 'my-app',
  serverUrl: 'http://localhost:8200'
});

// 4. AWS X-Ray
const AWSXRay = require('aws-xray-sdk-core');
const http = AWSXRay.captureHTTPs(require('http'));
```

## 常见面试题

### 1. 如何检测和修复内存泄漏？

<details>
<summary>点击查看答案</summary>

**检测方法**：

1. **监控内存使用**
```javascript
setInterval(() => {
  const usage = process.memoryUsage();
  console.log(`Heap Used: ${usage.heapUsed / 1024 / 1024} MB`);
}, 10000);
```

2. **使用 heapdump**
```javascript
const heapdump = require('heapdump');
heapdump.writeSnapshot();
// 在 Chrome DevTools 中分析 .heapsnapshot 文件
```

3. **使用 memwatch**
```javascript
const memwatch = require('@airbnb/node-memwatch');

memwatch.on('leak', (info) => {
  console.error('Memory leak detected:', info);
});
```

**常见原因和修复**：

1. 全局变量 → 使用严格模式
2. 闭包 → 只保留需要的数据
3. 定时器 → 清理定时器
4. 事件监听器 → 移除监听器
5. 缓存 → 使用 LRU 缓存
</details>

### 2. Node.js 有哪些性能优化手段？

<details>
<summary>点击查看答案</summary>

**代码层面**：
1. 避免阻塞事件循环（使用 Worker Threads）
2. 使用异步 I/O
3. 缓存计算结果
4. 使用流处理大数据
5. 优化算法和数据结构

**数据库层面**：
1. 使用连接池
2. 批量操作
3. 添加索引
4. 避免 N+1 查询
5. 使用缓存（Redis）

**HTTP 层面**：
1. 启用压缩（gzip）
2. 使用 HTTP/2
3. CDN 加速
4. 启用缓存
5. 连接复用

**架构层面**：
1. 使用 Cluster 充分利用多核
2. 负载均衡
3. 微服务拆分
4. 消息队列异步处理
5. 读写分离

**监控层面**：
1. APM 监控
2. 日志分析
3. 性能追踪
4. 告警机制
</details>

### 3. 如何优化 Node.js 的启动时间？

<details>
<summary>点击查看答案</summary>

```javascript
// 1. 延迟加载模块
// ❌ 不好：启动时加载所有模块
const heavyModule = require('heavy-module');

// ✅ 好：按需加载
let heavyModule;
function getHeavyModule() {
  if (!heavyModule) {
    heavyModule = require('heavy-module');
  }
  return heavyModule;
}

// 2. 使用 V8 快照
// 生成快照
v8.writeHeapSnapshot();

// 3. 减少依赖数量
// 审查 package.json，移除不必要的依赖

// 4. 使用 esbuild 等打包工具
// 将多个模块打包成单个文件

// 5. 预编译 TypeScript
// 部署编译后的 JS 而不是源码
```
</details>

### 4. 什么是背压（Backpressure）？如何处理？

<details>
<summary>点击查看答案</summary>

**定义**：当数据生产速度超过消费速度时，会产生背压。

**问题示例**：
```javascript
// ❌ 不好：没有处理背压
const readStream = fs.createReadStream('large-file.txt');
const writeStream = fs.createWriteStream('output.txt');

readStream.on('data', (chunk) => {
  writeStream.write(chunk); // 可能导致内存积压
});
```

**解决方案**：

1. **使用 pipe**（自动处理）
```javascript
// ✅ 好：pipe 自动处理背压
readStream.pipe(writeStream);
```

2. **手动处理**
```javascript
// ✅ 好：手动处理背压
readStream.on('data', (chunk) => {
  const canContinue = writeStream.write(chunk);
  
  if (!canContinue) {
    // 缓冲区满，暂停读取
    readStream.pause();
  }
});

writeStream.on('drain', () => {
  // 缓冲区清空，恢复读取
  readStream.resume();
});
```

3. **使用 pipeline**
```javascript
const { pipeline } = require('stream');

pipeline(
  readStream,
  transformStream,
  writeStream,
  (err) => {
    if (err) console.error('Pipeline failed:', err);
  }
);
```
</details>

### 5. 如何监控 Node.js 应用的性能？

<details>
<summary>点击查看答案</summary>

**1. 基础监控**
```javascript
// 内存监控
const usage = process.memoryUsage();
console.log({
  rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,
  heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`
});

// CPU 监控
const startUsage = process.cpuUsage();
// ... 执行操作
const cpuUsage = process.cpuUsage(startUsage);
console.log(`CPU: ${cpuUsage.user / 1000}ms`);

// Event Loop 延迟
const start = Date.now();
setImmediate(() => {
  const lag = Date.now() - start;
  console.log(`Event Loop Lag: ${lag}ms`);
});
```

**2. APM 工具**
- New Relic
- Datadog
- Elastic APM
- AWS X-Ray

**3. 日志和追踪**
- Winston/Pino 日志
- Jaeger 分布式追踪
- Prometheus + Grafana

**4. 性能指标**
- 响应时间（P50, P95, P99）
- 吞吐量（RPS）
- 错误率
- CPU/内存使用率
- Event Loop 延迟
</details>

## 最佳实践

### 1. 使用性能分析工具

```bash
# Clinic.js
clinic doctor -- node app.js
clinic flame -- node app.js

# 0x 火焰图
0x app.js

# Node.js 内置 profiler
node --prof app.js
node --prof-process isolate-*.log > processed.txt
```

### 2. 定期监控和告警

```javascript
// 监控 Event Loop 延迟
const lagMonitor = require('event-loop-lag')();

setInterval(() => {
  const lag = lagMonitor();
  if (lag > 100) {
    console.warn(`High event loop lag: ${lag}ms`);
  }
}, 10000);
```

### 3. 使用负载测试

```bash
# Artillery
artillery quick --count 10 -n 100 http://localhost:3000

# Apache Bench
ab -n 1000 -c 100 http://localhost:3000/

# autocannon
autocannon -c 100 -d 10 http://localhost:3000
```

### 4. 实施性能预算

```javascript
// 设置性能阈值
const PERFORMANCE_BUDGET = {
  maxResponseTime: 200,    // ms
  maxMemory: 500 * 1024 * 1024,  // 500MB
  maxCpu: 80              // %
};

// 监控并告警
if (responseTime > PERFORMANCE_BUDGET.maxResponseTime) {
  alert('Response time exceeded budget');
}
```

