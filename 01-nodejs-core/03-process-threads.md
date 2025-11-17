# 进程与线程

## Node.js 的单线程模型

### 单线程的含义

Node.js 的 JavaScript 代码运行在单个线程中，但这不意味着 Node.js 本身是单线程的。

```javascript
// JavaScript 代码在主线程执行
console.log('Main thread:', process.pid);

// I/O 操作在 libuv 的线程池中执行
const fs = require('fs');
fs.readFile('file.txt', (err, data) => {
  // 回调在主线程执行
  console.log('Callback in main thread');
});
```

### 单线程的优势和劣势

**优势**：
- 避免多线程的复杂性（锁、竞态条件）
- 减少上下文切换开销
- 简化编程模型

**劣势**：
- 无法充分利用多核 CPU
- CPU 密集型任务会阻塞整个应用
- 一个未捕获的异常可能导致整个进程崩溃

## Process 对象

`process` 是一个全局对象，提供有关当前 Node.js 进程的信息和控制能力。

### 基本信息

```javascript
// 进程 ID
console.log('PID:', process.pid);

// 父进程 ID
console.log('PPID:', process.ppid);

// 平台信息
console.log('Platform:', process.platform); // 'darwin', 'linux', 'win32'
console.log('Architecture:', process.arch); // 'x64', 'arm64'

// Node.js 版本
console.log('Node version:', process.version);
console.log('Versions:', process.versions); // node, v8, openssl 等

// 当前工作目录
console.log('CWD:', process.cwd());

// 改变工作目录
process.chdir('/tmp');

// 执行时间
console.log('Uptime:', process.uptime(), 'seconds');

// 命令行参数
console.log('Arguments:', process.argv);
// node app.js arg1 arg2
// process.argv = ['node', '/path/to/app.js', 'arg1', 'arg2']
```

### 环境变量

```javascript
// 读取环境变量
console.log('NODE_ENV:', process.env.NODE_ENV);
console.log('HOME:', process.env.HOME);

// 设置环境变量
process.env.CUSTOM_VAR = 'value';

// 使用 dotenv 加载 .env 文件
require('dotenv').config();
console.log(process.env.DATABASE_URL);

// 最佳实践：提供默认值
const port = process.env.PORT || 3000;
const env = process.env.NODE_ENV || 'development';
```

### 内存使用

```javascript
// 查看内存使用
const memoryUsage = process.memoryUsage();
console.log({
  rss: `${Math.round(memoryUsage.rss / 1024 / 1024)} MB`,        // 常驻内存
  heapTotal: `${Math.round(memoryUsage.heapTotal / 1024 / 1024)} MB`, // 堆总量
  heapUsed: `${Math.round(memoryUsage.heapUsed / 1024 / 1024)} MB`,   // 已使用堆
  external: `${Math.round(memoryUsage.external / 1024 / 1024)} MB`,   // C++ 对象
  arrayBuffers: `${Math.round(memoryUsage.arrayBuffers / 1024 / 1024)} MB`
});

// CPU 使用时间
const cpuUsage = process.cpuUsage();
console.log({
  user: cpuUsage.user / 1000000,    // 用户 CPU 时间（秒）
  system: cpuUsage.system / 1000000 // 系统 CPU 时间（秒）
});
```

### 进程事件

```javascript
// 退出事件
process.on('exit', (code) => {
  console.log(`Process exiting with code: ${code}`);
  // 注意：这里只能执行同步操作
});

// 未捕获的异常
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  // 记录错误后应该退出进程
  process.exit(1);
});

// 未处理的 Promise rejection
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
});

// 警告
process.on('warning', (warning) => {
  console.warn(warning.name);
  console.warn(warning.message);
  console.warn(warning.stack);
});

// 信号处理
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

process.on('SIGINT', () => {
  console.log('SIGINT received (Ctrl+C)');
  process.exit(0);
});
```

### 退出进程

```javascript
// 正常退出
process.exit(0);

// 错误退出
process.exit(1);

// 异步操作完成后退出
setTimeout(() => {
  process.exit(0);
}, 1000);

// 优雅退出
async function gracefulShutdown() {
  console.log('Shutting down gracefully...');
  
  // 停止接受新请求
  server.close();
  
  // 关闭数据库连接
  await db.close();
  
  // 清理资源
  await cleanup();
  
  process.exit(0);
}

process.on('SIGTERM', gracefulShutdown);
```

## Child Process（子进程）

Node.js 提供了 `child_process` 模块来创建子进程。

### spawn - 启动子进程

```javascript
const { spawn } = require('child_process');

// 执行命令
const ls = spawn('ls', ['-lh', '/usr']);

// 监听标准输出
ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

// 监听标准错误
ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

// 监听关闭事件
ls.on('close', (code) => {
  console.log(`Child process exited with code ${code}`);
});

// 监听错误
ls.on('error', (err) => {
  console.error('Failed to start child process:', err);
});
```

### exec - 执行 shell 命令

```javascript
const { exec } = require('child_process');

// 执行命令并获取完整输出
exec('cat *.js | wc -l', (error, stdout, stderr) => {
  if (error) {
    console.error(`exec error: ${error}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
  console.error(`stderr: ${stderr}`);
});

// 带选项
exec('cat *.js', {
  cwd: '/path/to/directory',
  env: { ...process.env, CUSTOM: 'value' },
  maxBuffer: 1024 * 1024, // 1MB
  timeout: 5000 // 5 秒超时
}, (error, stdout, stderr) => {
  // 处理结果
});

// Promise 版本
const { promisify } = require('util');
const execPromise = promisify(exec);

async function runCommand() {
  try {
    const { stdout, stderr } = await execPromise('ls -lh');
    console.log(stdout);
  } catch (error) {
    console.error(error);
  }
}
```

### execFile - 执行文件

```javascript
const { execFile } = require('child_process');

// 直接执行文件（不通过 shell）
execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    console.error(`execFile error: ${error}`);
    return;
  }
  console.log(`Node version: ${stdout}`);
});

// 执行 Node.js 脚本
execFile('node', ['script.js', 'arg1', 'arg2'], (error, stdout, stderr) => {
  // 处理结果
});
```

### fork - 创建 Node.js 子进程

```javascript
// parent.js
const { fork } = require('child_process');

const child = fork('child.js');

// 发送消息给子进程
child.send({ type: 'START', data: 'Hello' });

// 接收子进程消息
child.on('message', (message) => {
  console.log('Message from child:', message);
});

// 子进程退出
child.on('exit', (code) => {
  console.log(`Child process exited with code ${code}`);
});

// child.js
process.on('message', (message) => {
  console.log('Message from parent:', message);
  
  // 处理消息
  if (message.type === 'START') {
    // 执行任务
    const result = doWork(message.data);
    
    // 发送结果回父进程
    process.send({ type: 'RESULT', data: result });
  }
});

function doWork(data) {
  // CPU 密集型任务
  return data.toUpperCase();
}
```

### 子进程通信示例

```javascript
// master.js - 计算斐波那契数列
const { fork } = require('child_process');

function calculateFibonacci(n) {
  return new Promise((resolve, reject) => {
    const worker = fork('fibonacci-worker.js');
    
    worker.send({ n });
    
    worker.on('message', (message) => {
      resolve(message.result);
      worker.kill();
    });
    
    worker.on('error', reject);
    
    // 超时处理
    setTimeout(() => {
      worker.kill();
      reject(new Error('Worker timeout'));
    }, 10000);
  });
}

// 并行计算
async function main() {
  const results = await Promise.all([
    calculateFibonacci(40),
    calculateFibonacci(41),
    calculateFibonacci(42)
  ]);
  
  console.log('Results:', results);
}

main();

// fibonacci-worker.js
process.on('message', ({ n }) => {
  const result = fibonacci(n);
  process.send({ result });
});

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
```

## Cluster（集群）

Cluster 模块允许轻松创建共享服务器端口的子进程。

### 基本使用

```javascript
const cluster = require('cluster');
const http = require('http');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  console.log(`Master ${process.pid} is running`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  // 监听 worker 退出
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    // 重启 worker
    cluster.fork();
  });
  
} else {
  // Workers 共享 TCP 连接
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Hello from worker ${process.pid}\n`);
  }).listen(8000);
  
  console.log(`Worker ${process.pid} started`);
}
```

### 完整的集群实现

```javascript
const cluster = require('cluster');
const http = require('http');
const os = require('os');

class ClusterManager {
  constructor(workerCount = os.cpus().length) {
    this.workerCount = workerCount;
    this.workers = new Map();
  }
  
  start() {
    if (cluster.isMaster) {
      this.startMaster();
    } else {
      this.startWorker();
    }
  }
  
  startMaster() {
    console.log(`Master ${process.pid} starting...`);
    console.log(`Starting ${this.workerCount} workers`);
    
    // 创建 workers
    for (let i = 0; i < this.workerCount; i++) {
      this.forkWorker();
    }
    
    // Worker 退出时重启
    cluster.on('exit', (worker, code, signal) => {
      console.log(`Worker ${worker.process.pid} died (${signal || code})`);
      this.workers.delete(worker.id);
      
      // 如果不是正常退出，重启 worker
      if (code !== 0 && !worker.exitedAfterDisconnect) {
        console.log('Starting a new worker...');
        this.forkWorker();
      }
    });
    
    // 优雅重启
    process.on('SIGUSR2', () => {
      console.log('Restarting workers...');
      this.gracefulRestart();
    });
    
    // 优雅关闭
    process.on('SIGTERM', () => {
      console.log('Master received SIGTERM, shutting down...');
      this.shutdown();
    });
  }
  
  forkWorker() {
    const worker = cluster.fork();
    this.workers.set(worker.id, worker);
    
    // Worker 消息
    worker.on('message', (message) => {
      console.log(`Message from worker ${worker.process.pid}:`, message);
    });
    
    return worker;
  }
  
  startWorker() {
    const server = http.createServer((req, res) => {
      // CPU 密集型任务模拟
      if (req.url === '/heavy') {
        const start = Date.now();
        while (Date.now() - start < 5000) {} // 阻塞 5 秒
      }
      
      res.writeHead(200);
      res.end(`Handled by worker ${process.pid}\n`);
    });
    
    server.listen(8000, () => {
      console.log(`Worker ${process.pid} listening on port 8000`);
    });
    
    // 优雅关闭
    process.on('SIGTERM', () => {
      console.log(`Worker ${process.pid} shutting down...`);
      server.close(() => {
        process.exit(0);
      });
    });
  }
  
  async gracefulRestart() {
    const workers = Array.from(this.workers.values());
    
    // 逐个重启 worker
    for (const worker of workers) {
      await this.restartWorker(worker);
      // 等待一段时间再重启下一个
      await this.sleep(1000);
    }
    
    console.log('All workers restarted');
  }
  
  restartWorker(worker) {
    return new Promise((resolve) => {
      // 创建新 worker
      const newWorker = this.forkWorker();
      
      // 等待新 worker 准备就绪
      newWorker.on('listening', () => {
        // 关闭旧 worker
        worker.disconnect();
        
        setTimeout(() => {
          worker.kill();
          resolve();
        }, 5000); // 给 5 秒时间处理现有请求
      });
    });
  }
  
  shutdown() {
    // 关闭所有 workers
    for (const worker of this.workers.values()) {
      worker.disconnect();
      setTimeout(() => worker.kill(), 5000);
    }
    
    // 等待所有 workers 关闭
    setTimeout(() => {
      process.exit(0);
    }, 10000);
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// 使用
const manager = new ClusterManager();
manager.start();
```

## Worker Threads（工作线程）

Worker Threads 提供了真正的多线程能力，适合 CPU 密集型任务。

### 基本使用

```javascript
// main.js
const { Worker } = require('worker_threads');

function runWorker(workerData) {
  return new Promise((resolve, reject) => {
    const worker = new Worker('./worker.js', { workerData });
    
    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

// 使用
async function main() {
  const result = await runWorker({ number: 42 });
  console.log('Result:', result);
}

main();

// worker.js
const { workerData, parentPort } = require('worker_threads');

// 执行 CPU 密集型任务
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

const result = fibonacci(workerData.number);
parentPort.postMessage(result);
```

### Worker Pool 实现

```javascript
const { Worker } = require('worker_threads');
const os = require('os');

class WorkerPool {
  constructor(workerScript, poolSize = os.cpus().length) {
    this.workerScript = workerScript;
    this.poolSize = poolSize;
    this.workers = [];
    this.freeWorkers = [];
    this.queue = [];
    
    // 初始化 worker 池
    for (let i = 0; i < poolSize; i++) {
      this.addWorker();
    }
  }
  
  addWorker() {
    const worker = new Worker(this.workerScript);
    
    worker.on('message', (result) => {
      // Worker 完成任务
      const { resolve } = worker.currentTask;
      resolve(result);
      
      // 处理队列中的下一个任务
      this.freeWorkers.push(worker);
      this.processQueue();
    });
    
    worker.on('error', (error) => {
      const { reject } = worker.currentTask;
      reject(error);
      
      // 移除错误的 worker 并创建新的
      const index = this.workers.indexOf(worker);
      if (index !== -1) {
        this.workers.splice(index, 1);
      }
      
      this.addWorker();
      this.processQueue();
    });
    
    this.workers.push(worker);
    this.freeWorkers.push(worker);
  }
  
  exec(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processQueue();
    });
  }
  
  processQueue() {
    if (this.queue.length === 0 || this.freeWorkers.length === 0) {
      return;
    }
    
    const worker = this.freeWorkers.pop();
    const task = this.queue.shift();
    
    worker.currentTask = task;
    worker.postMessage(task.data);
  }
  
  async destroy() {
    for (const worker of this.workers) {
      await worker.terminate();
    }
    this.workers = [];
    this.freeWorkers = [];
  }
}

// 使用
const pool = new WorkerPool('./worker.js', 4);

async function main() {
  const tasks = Array.from({ length: 10 }, (_, i) => i + 40);
  
  const results = await Promise.all(
    tasks.map(n => pool.exec({ number: n }))
  );
  
  console.log('Results:', results);
  
  await pool.destroy();
}

main();
```

### Worker Threads vs Child Process vs Cluster

| 特性 | Worker Threads | Child Process | Cluster |
|------|----------------|---------------|---------|
| **共享内存** | 可以（通过 SharedArrayBuffer） | 不可以 | 不可以 |
| **通信方式** | 消息传递 | IPC 或消息传递 | IPC |
| **启动开销** | 低 | 高 | 高 |
| **内存开销** | 低 | 高 | 高 |
| **适用场景** | CPU 密集型任务 | 运行其他程序 | 负载均衡 |
| **隔离性** | 共享同一进程 | 完全隔离 | 完全隔离 |

## 常见面试题

### 1. Node.js 是单线程的吗？

<details>
<summary>点击查看答案</summary>

**不完全是**。更准确的说法是：

1. **JavaScript 执行是单线程的**：你的 JS 代码在主线程中执行
2. **Node.js 本身是多线程的**：
   - libuv 使用线程池处理 I/O 操作
   - 默认线程池大小为 4（可通过 `UV_THREADPOOL_SIZE` 环境变量调整）
   - V8 的垃圾回收在单独的线程中进行

```javascript
// 这些操作会使用线程池
fs.readFile('file.txt', callback);    // 文件 I/O
crypto.pbkdf2('password', 'salt', ...); // CPU 密集型加密
dns.lookup('google.com', callback);   // DNS 查询
```

**优势**：
- 简单的编程模型
- 避免线程同步问题
- 高效的 I/O 处理

**劣势**：
- 无法充分利用多核 CPU
- CPU 密集型任务会阻塞事件循环
</details>

### 2. 如何充分利用多核 CPU？

<details>
<summary>点击查看答案</summary>

有三种主要方式：

**1. Cluster 模块**（推荐用于 HTTP 服务器）

```javascript
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
} else {
  // 启动服务器
}
```

**2. Worker Threads**（推荐用于 CPU 密集型任务）

```javascript
const { Worker } = require('worker_threads');

const worker = new Worker('./heavy-task.js', {
  workerData: { task: 'compute' }
});
```

**3. Child Process**（运行其他程序）

```javascript
const { fork } = require('child_process');
const child = fork('child.js');
```

**选择建议**：
- HTTP 服务器 → Cluster
- CPU 密集型计算 → Worker Threads
- 运行外部程序 → Child Process
</details>

### 3. 如何实现优雅关闭（Graceful Shutdown）？

<details>
<summary>点击查看答案</summary>

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.end('Hello');
});

server.listen(3000);

// 跟踪活跃连接
const connections = new Set();

server.on('connection', (conn) => {
  connections.add(conn);
  
  conn.on('close', () => {
    connections.delete(conn);
  });
});

// 优雅关闭函数
async function gracefulShutdown(signal) {
  console.log(`Received ${signal}, starting graceful shutdown...`);
  
  // 1. 停止接受新连接
  server.close(() => {
    console.log('Server closed');
  });
  
  // 2. 关闭现有连接（给 30 秒时间）
  setTimeout(() => {
    console.log('Forcing shutdown');
    connections.forEach(conn => conn.destroy());
    process.exit(0);
  }, 30000);
  
  // 3. 关闭数据库连接
  try {
    await db.close();
    console.log('Database closed');
  } catch (err) {
    console.error('Error closing database:', err);
  }
  
  // 4. 清理其他资源
  await cleanup();
  
  // 5. 所有活跃连接结束后退出
  if (connections.size === 0) {
    process.exit(0);
  }
}

// 监听关闭信号
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// 处理未捕获的异常
process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  gracefulShutdown('uncaughtException');
});

process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Rejection:', reason);
  gracefulShutdown('unhandledRejection');
});
```
</details>

### 4. Cluster 和 PM2 的区别？

<details>
<summary>点击查看答案</summary>

**Cluster**（Node.js 内置）：
- 需要自己编写集群管理代码
- 灵活性高，可以自定义行为
- 需要处理 worker 重启、监控等

**PM2**（进程管理器）：
- 开箱即用，无需编写集群代码
- 自动重启、负载均衡、日志管理
- 提供监控界面和 CLI 工具
- 支持零停机重载

```bash
# PM2 使用
pm2 start app.js -i max  # 使用所有 CPU 核心
pm2 reload app           # 零停机重载
pm2 logs                 # 查看日志
pm2 monit                # 监控
```

**选择建议**：
- 生产环境 → PM2（功能更完整）
- 需要自定义逻辑 → Cluster
- 学习 Node.js → 两者都了解
</details>

### 5. 什么时候使用 Worker Threads？

<details>
<summary>点击查看答案</summary>

**适用场景**：

1. **CPU 密集型任务**
```javascript
// 图像处理
// 视频编码
// 数据加密/解密
// 复杂计算
```

2. **大量数据处理**
```javascript
// 解析大型 JSON/XML
// 数据转换
// 文件处理
```

**不适用场景**：

1. **I/O 密集型任务**（已经是异步的）
```javascript
// 数据库查询
// 文件读写
// 网络请求
```

2. **简单任务**（线程开销大于收益）

**示例：何时需要**

```javascript
// ❌ 不需要 Worker Thread（I/O 操作）
app.get('/users', async (req, res) => {
  const users = await db.query('SELECT * FROM users');
  res.json(users);
});

// ✅ 需要 Worker Thread（CPU 密集）
app.post('/encrypt', async (req, res) => {
  const encrypted = await runInWorker({
    task: 'encrypt',
    data: req.body.data,
    algorithm: 'aes-256-gcm'
  });
  res.json({ encrypted });
});
```
</details>

## 最佳实践

### 1. 使用 PM2 管理生产环境进程

```json
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'app',
    script: './app.js',
    instances: 'max',
    exec_mode: 'cluster',
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'production'
    }
  }]
};
```

### 2. 避免阻塞事件循环

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
app.get('/fib/:n', async (req, res) => {
  const result = await runInWorker({
    task: 'fibonacci',
    n: req.params.n
  });
  res.json({ result });
});
```

### 3. 正确处理进程信号

```javascript
// 优雅关闭
['SIGTERM', 'SIGINT'].forEach(signal => {
  process.on(signal, async () => {
    console.log(`${signal} received`);
    await gracefulShutdown();
    process.exit(0);
  });
});

// 错误处理
process.on('uncaughtException', (err) => {
  logger.error('Uncaught Exception:', err);
  process.exit(1);
});

process.on('unhandledRejection', (reason) => {
  logger.error('Unhandled Rejection:', reason);
  process.exit(1);
});
```

### 4. 使用环境变量管理配置

```javascript
// config.js
module.exports = {
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || 'development',
  db: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT) || 5432,
    name: process.env.DB_NAME || 'myapp',
  },
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379'
  }
};
```

