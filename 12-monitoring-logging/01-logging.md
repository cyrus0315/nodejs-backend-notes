# 日志系统

日志是应用程序的黑匣子，是排查问题、监控系统的重要手段。本文深入讲解 Node.js 日志系统的设计与实现。

## 目录
- [日志基础](#日志基础)
- [Winston 详解](#winston-详解)
- [Pino 高性能日志](#pino-高性能日志)
- [日志聚合](#日志聚合)
- [日志最佳实践](#日志最佳实践)
- [面试题](#常见面试题)

---

## 日志基础

### 日志级别

```typescript
enum LogLevel {
  ERROR = 0,   // 错误：需要立即处理
  WARN = 1,    // 警告：潜在问题
  INFO = 2,    // 信息：重要的业务流程
  HTTP = 3,    // HTTP 请求
  VERBOSE = 4, // 详细信息
  DEBUG = 5,   // 调试信息
  SILLY = 6    // 最详细的信息
}

// 使用示例
logger.error('Database connection failed', { error: err.message });
logger.warn('Cache miss rate high', { rate: 0.85 });
logger.info('User logged in', { userId: 123 });
logger.debug('Query executed', { sql: query, duration: 45 });
```

### 结构化日志

```typescript
// ❌ 不好：字符串拼接
logger.info(`User ${userId} logged in from ${ip}`);

// ✅ 好：结构化数据
logger.info('User logged in', {
  userId,
  ip,
  userAgent: req.headers['user-agent'],
  timestamp: new Date().toISOString()
});

// JSON 输出（易于解析）
{
  "level": "info",
  "message": "User logged in",
  "userId": 123,
  "ip": "192.168.1.1",
  "userAgent": "Mozilla/5.0...",
  "timestamp": "2024-01-01T10:00:00.000Z"
}
```

### 日志上下文

```typescript
// 使用 AsyncLocalStorage 传递上下文
import { AsyncLocalStorage } from 'async_hooks';

const asyncLocalStorage = new AsyncLocalStorage();

// 中间件：设置请求上下文
app.use((req, res, next) => {
  const requestId = req.headers['x-request-id'] || crypto.randomUUID();
  const userId = req.user?.id;

  asyncLocalStorage.run({ requestId, userId }, () => {
    next();
  });
});

// 日志工具：自动添加上下文
function log(level: string, message: string, meta: any = {}) {
  const context = asyncLocalStorage.getStore() || {};
  
  logger[level](message, {
    ...meta,
    ...context // 自动添加 requestId, userId
  });
}

// 使用
log('info', 'Processing payment', { amount: 100 });
// 输出：{ level: 'info', message: 'Processing payment', amount: 100, requestId: '...', userId: 123 }
```

---

## Winston 详解

### 基础配置

```typescript
import winston from 'winston';
import DailyRotateFile from 'winston-daily-rotate-file';

// 创建 logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  
  // 格式化
  format: winston.format.combine(
    winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
    winston.format.errors({ stack: true }), // 捕获错误堆栈
    winston.format.splat(), // 支持字符串插值
    winston.format.json()
  ),
  
  // 默认元数据
  defaultMeta: {
    service: 'my-app',
    environment: process.env.NODE_ENV,
    hostname: os.hostname()
  },
  
  // 传输器
  transports: [
    // 错误日志
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5
    }),
    
    // 所有日志
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10
    })
  ],
  
  // 异常处理
  exceptionHandlers: [
    new winston.transports.File({ filename: 'logs/exceptions.log' })
  ],
  
  // Promise rejection 处理
  rejectionHandlers: [
    new winston.transports.File({ filename: 'logs/rejections.log' })
  ]
});

// 开发环境：彩色输出到控制台
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.printf(({ level, message, timestamp, ...meta }) => {
        return `${timestamp} [${level}]: ${message} ${
          Object.keys(meta).length ? JSON.stringify(meta, null, 2) : ''
        }`;
      })
    )
  }));
}

export default logger;
```

### 日志轮转

```typescript
import DailyRotateFile from 'winston-daily-rotate-file';

// 按日期轮转
const transport = new DailyRotateFile({
  filename: 'logs/application-%DATE%.log',
  datePattern: 'YYYY-MM-DD',
  zippedArchive: true, // 压缩旧日志
  maxSize: '20m',      // 单个文件最大 20MB
  maxFiles: '14d'      // 保留 14 天
});

transport.on('rotate', (oldFilename, newFilename) => {
  logger.info('Log file rotated', { oldFilename, newFilename });
});

logger.add(transport);

// 按大小轮转
const sizeRotateTransport = new DailyRotateFile({
  filename: 'logs/app-%DATE%.log',
  datePattern: 'YYYY-MM-DD-HH',
  maxSize: '10m',
  maxFiles: '7d'
});
```

### 自定义格式

```typescript
// 自定义格式化函数
const customFormat = winston.format.printf(({ level, message, timestamp, ...meta }) => {
  // 提取错误堆栈
  const stack = meta.stack;
  delete meta.stack;

  let log = `${timestamp} [${level.toUpperCase()}] ${message}`;

  // 添加元数据
  if (Object.keys(meta).length > 0) {
    log += ` ${JSON.stringify(meta)}`;
  }

  // 添加堆栈
  if (stack) {
    log += `\n${stack}`;
  }

  return log;
});

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    customFormat
  ),
  transports: [
    new winston.transports.Console()
  ]
});

// 使用
logger.error('Something went wrong', new Error('Test error'));
// 输出：
// 2024-01-01 10:00:00 [ERROR] Something went wrong
// Error: Test error
//     at Object.<anonymous> (/path/to/file.js:10:15)
//     ...
```

### Child Logger

```typescript
// 创建子 logger（带有额外的上下文）
const childLogger = logger.child({ 
  module: 'payment',
  version: '1.0.0'
});

childLogger.info('Processing payment', { orderId: '12345' });
// 输出：{ level: 'info', message: 'Processing payment', 
//        module: 'payment', version: '1.0.0', orderId: '12345' }

// 为每个请求创建 child logger
app.use((req, res, next) => {
  req.logger = logger.child({
    requestId: req.id,
    path: req.path,
    method: req.method
  });
  next();
});

// 在路由中使用
app.post('/api/orders', (req, res) => {
  req.logger.info('Creating order', { userId: req.user.id });
  // 自动包含 requestId, path, method
});
```

### 多个 Transport

```typescript
// 根据日志级别使用不同的 transport
const logger = winston.createLogger({
  transports: [
    // 1. 错误日志 -> 文件
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error'
    }),
    
    // 2. 警告和错误 -> 另一个文件
    new winston.transports.File({
      filename: 'logs/warn.log',
      level: 'warn'
    }),
    
    // 3. 所有日志 -> 文件
    new winston.transports.File({
      filename: 'logs/combined.log'
    }),
    
    // 4. 错误日志 -> Slack（自定义 transport）
    new SlackTransport({
      level: 'error',
      webhookUrl: process.env.SLACK_WEBHOOK_URL
    })
  ]
});

// 自定义 Transport
import Transport from 'winston-transport';

class SlackTransport extends Transport {
  constructor(opts) {
    super(opts);
    this.webhookUrl = opts.webhookUrl;
  }

  async log(info, callback) {
    setImmediate(() => {
      this.emit('logged', info);
    });

    try {
      await fetch(this.webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          text: `[${info.level.toUpperCase()}] ${info.message}`,
          attachments: [{
            color: 'danger',
            fields: Object.entries(info.metadata || {}).map(([key, value]) => ({
              title: key,
              value: String(value),
              short: true
            }))
          }]
        })
      });
    } catch (error) {
      console.error('Failed to send log to Slack:', error);
    }

    callback();
  }
}
```

---

## Pino 高性能日志

### 为什么 Pino 更快？

- **异步写入**：不阻塞主线程
- **最小化对象创建**
- **JSON 序列化优化**
- **Worker 线程**

### 基础使用

```typescript
import pino from 'pino';

// 创建 logger
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  
  // 序列化器（过滤敏感信息）
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      // 不记录 headers 中的敏感信息
      headers: {
        ...req.headers,
        authorization: undefined,
        cookie: undefined
      }
    }),
    res: (res) => ({
      statusCode: res.statusCode
    }),
    err: pino.stdSerializers.err
  },
  
  // 基础字段
  base: {
    service: 'my-app',
    environment: process.env.NODE_ENV
  },
  
  // 时间戳格式
  timestamp: pino.stdTimeFunctions.isoTime
});

// 使用
logger.info('Server started', { port: 3000 });
logger.error({ err: new Error('Failed') }, 'Operation failed');
```

### Pretty Print（开发环境）

```typescript
import pino from 'pino';

const logger = pino({
  transport: process.env.NODE_ENV !== 'production'
    ? {
        target: 'pino-pretty',
        options: {
          colorize: true,
          translateTime: 'SYS:standard',
          ignore: 'pid,hostname'
        }
      }
    : undefined
});

// 输出（开发环境）：
// [2024-01-01 10:00:00] INFO: Server started
//     port: 3000
```

### Child Logger

```typescript
// 创建 child logger
const childLogger = logger.child({ module: 'auth' });

childLogger.info('User login attempt', { userId: 123 });
// 输出：{ level: 30, module: 'auth', msg: 'User login attempt', userId: 123 }

// 为每个请求创建 child logger
app.use((req, res, next) => {
  req.log = logger.child({
    requestId: crypto.randomUUID(),
    path: req.path
  });
  
  req.log.info('Request started');
  next();
});
```

### Pino HTTP

```typescript
import express from 'express';
import pino from 'pino';
import pinoHttp from 'pino-http';

const logger = pino();
const app = express();

// HTTP 日志中间件
app.use(pinoHttp({
  logger,
  
  // 自定义请求日志
  customLogLevel: (res, err) => {
    if (res.statusCode >= 400 && res.statusCode < 500) {
      return 'warn';
    } else if (res.statusCode >= 500 || err) {
      return 'error';
    }
    return 'info';
  },
  
  // 自定义成功消息
  customSuccessMessage: (res) => {
    return `Request completed with status ${res.statusCode}`;
  },
  
  // 自定义错误消息
  customErrorMessage: (err, res) => {
    return `Request failed with status ${res.statusCode}`;
  },
  
  // 自定义属性
  customAttributeKeys: {
    req: 'request',
    res: 'response',
    err: 'error',
    responseTime: 'duration'
  }
}));

// 自动记录请求和响应
app.get('/api/users', (req, res) => {
  req.log.info('Fetching users');
  res.json({ users: [] });
});

// 输出：
// {"level":30,"request":{...},"msg":"Request completed with status 200","duration":5}
```

### 日志轮转（Pino）

```bash
# 使用 pino-roll
npm install pino-roll

# 启动应用
node app.js | pino-roll --size 10m --frequency daily
```

```typescript
// 或使用 rotating-file-stream
import pino from 'pino';
import rfs from 'rotating-file-stream';

const stream = rfs.createStream('app.log', {
  size: '10M',
  interval: '1d',
  path: './logs',
  compress: 'gzip'
});

const logger = pino(stream);
```

### Winston vs Pino 对比

```typescript
// 性能测试
import Benchmark from 'benchmark';
import winston from 'winston';
import pino from 'pino';

const winstonLogger = winston.createLogger({
  transports: [
    new winston.transports.File({ filename: 'winston.log' })
  ]
});

const pinoLogger = pino(pino.destination('pino.log'));

const suite = new Benchmark.Suite();

suite
  .add('Winston', () => {
    winstonLogger.info('Test message', { userId: 123 });
  })
  .add('Pino', () => {
    pinoLogger.info({ userId: 123 }, 'Test message');
  })
  .on('cycle', (event) => {
    console.log(String(event.target));
  })
  .on('complete', function() {
    console.log('Fastest is ' + this.filter('fastest').map('name'));
  })
  .run();

// 结果：
// Winston x 10,000 ops/sec ±1%
// Pino x 50,000 ops/sec ±1%
// Fastest is Pino
```

| 特性 | Winston | Pino |
|------|---------|------|
| **性能** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **易用性** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **生态系统** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **自定义** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **适用场景** | 功能丰富的应用 | 高性能应用 |

---

## 日志聚合

### ELK Stack

#### 架构

```
应用 → Filebeat → Logstash → Elasticsearch → Kibana
```

#### Filebeat 配置

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    
    # JSON 解析
    json.keys_under_root: true
    json.add_error_key: true
    
    # 添加字段
    fields:
      app: my-app
      environment: production

# 输出到 Logstash
output.logstash:
  hosts: ["localhost:5044"]
  
# 或直接输出到 Elasticsearch
output.elasticsearch:
  hosts: ["localhost:9200"]
  index: "app-logs-%{+yyyy.MM.dd}"
```

#### Logstash 配置

```ruby
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # 解析 JSON
  json {
    source => "message"
  }
  
  # 添加时间戳
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
  
  # 添加地理位置（根据 IP）
  geoip {
    source => "ip"
  }
  
  # 过滤敏感信息
  mutate {
    remove_field => ["password", "token"]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "app-logs-%{+YYYY.MM.dd}"
  }
  
  # 同时输出到控制台（调试用）
  stdout {
    codec => rubydebug
  }
}
```

#### Elasticsearch 索引模板

```json
PUT _index_template/app-logs-template
{
  "index_patterns": ["app-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "app-logs-policy"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "userId": { "type": "keyword" },
        "requestId": { "type": "keyword" },
        "duration": { "type": "long" },
        "error": {
          "properties": {
            "message": { "type": "text" },
            "stack": { "type": "text" }
          }
        }
      }
    }
  }
}
```

#### Kibana 可视化

```javascript
// 在 Kibana Dev Tools 中创建查询
GET app-logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "level": "error" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "timestamp",
        "fixed_interval": "5m"
      }
    }
  }
}
```

### CloudWatch Logs

```typescript
import winston from 'winston';
import WinstonCloudWatch from 'winston-cloudwatch';

const logger = winston.createLogger({
  transports: [
    new WinstonCloudWatch({
      logGroupName: '/aws/app/my-app',
      logStreamName: `${process.env.NODE_ENV}-${new Date().toISOString().split('T')[0]}`,
      awsRegion: 'us-east-1',
      jsonMessage: true,
      
      // 批量上传
      uploadRate: 2000, // 2 秒
      
      // 错误处理
      errorHandler: (err) => {
        console.error('CloudWatch logging error:', err);
      }
    })
  ]
});

// 使用
logger.info('User action', { userId: 123, action: 'login' });
```

### Loki（Grafana）

```typescript
import pino from 'pino';

// Pino + Loki
const logger = pino({
  transport: {
    target: 'pino-loki',
    options: {
      batching: true,
      interval: 5, // 5 秒批量发送
      host: 'http://localhost:3100',
      labels: {
        app: 'my-app',
        environment: process.env.NODE_ENV
      }
    }
  }
});
```

```yaml
# Promtail 配置
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: app-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: app-logs
          __path__: /var/log/app/*.log
    
    # JSON 解析
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            timestamp: timestamp
      
      - timestamp:
          source: timestamp
          format: RFC3339
      
      - labels:
          level:
```

---

## 日志最佳实践

### 1. 日志规范

```typescript
// ✅ 好的日志
logger.info('Order created', {
  orderId: order.id,
  userId: user.id,
  amount: order.amount,
  currency: order.currency,
  timestamp: new Date().toISOString()
});

// ❌ 不好的日志
logger.info('order created'); // 缺少上下文
logger.info(`Order ${order.id} created`); // 非结构化
logger.info('Order created', order); // 可能包含敏感信息
```

### 2. 敏感信息脱敏

```typescript
// 脱敏工具
function sanitize(obj: any): any {
  const sensitiveKeys = ['password', 'token', 'secret', 'apiKey', 'ssn', 'creditCard'];
  
  if (typeof obj !== 'object' || obj === null) {
    return obj;
  }
  
  const sanitized = Array.isArray(obj) ? [] : {};
  
  for (const [key, value] of Object.entries(obj)) {
    if (sensitiveKeys.some(sensitive => key.toLowerCase().includes(sensitive))) {
      sanitized[key] = '***REDACTED***';
    } else if (typeof value === 'object') {
      sanitized[key] = sanitize(value);
    } else {
      sanitized[key] = value;
    }
  }
  
  return sanitized;
}

// 包装 logger
function safeLog(level: string, message: string, meta: any = {}) {
  logger[level](message, sanitize(meta));
}

// 使用
safeLog('info', 'User registered', {
  email: 'user@example.com',
  password: '123456' // 会被脱敏
});
// 输出：{ email: 'user@example.com', password: '***REDACTED***' }
```

### 3. 日志采样

```typescript
// 高频日志采样（避免日志洪水）
class SamplingLogger {
  private counters = new Map<string, number>();
  private sampleRate = 0.1; // 10% 采样率

  log(level: string, message: string, meta: any = {}) {
    const key = `${level}:${message}`;
    const count = (this.counters.get(key) || 0) + 1;
    this.counters.set(key, count);

    // 每 N 次记录一次
    if (count === 1 || count % Math.floor(1 / this.sampleRate) === 0) {
      logger[level](message, {
        ...meta,
        sampledCount: count // 记录实际发生次数
      });
    }
  }

  // 定期清理计数器
  reset() {
    this.counters.clear();
  }
}

const samplingLogger = new SamplingLogger();
setInterval(() => samplingLogger.reset(), 60000); // 每分钟重置

// 使用
for (let i = 0; i < 1000; i++) {
  samplingLogger.log('debug', 'Cache hit'); // 只记录约 100 次
}
```

### 4. 日志关联

```typescript
// 使用 Correlation ID 关联分布式系统的日志
import { v4 as uuidv4 } from 'uuid';

app.use((req, res, next) => {
  // 从 header 获取或生成新的 correlation ID
  const correlationId = req.headers['x-correlation-id'] || uuidv4();
  
  // 传递给下游服务
  req.correlationId = correlationId;
  res.setHeader('X-Correlation-ID', correlationId);
  
  // 添加到日志上下文
  req.log = logger.child({ correlationId });
  
  next();
});

// 调用下游服务时传递
async function callDownstreamService(req, data) {
  return await fetch('http://downstream-service/api', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Correlation-ID': req.correlationId // 传递
    },
    body: JSON.stringify(data)
  });
}
```

### 5. 日志保留策略

```typescript
// Elasticsearch ILM（Index Lifecycle Management）
PUT _ilm/policy/app-logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 常见面试题

### 1. Winston vs Pino，如何选择？

| 场景 | 推荐 | 理由 |
|------|------|------|
| **高性能要求** | Pino | 5x 更快 |
| **需要多种 transport** | Winston | 生态系统丰富 |
| **微服务** | Pino | 异步、低开销 |
| **复杂格式化** | Winston | 灵活的格式化 |
| **已有 Winston 项目** | Winston | 迁移成本 |

### 2. 如何设计日志系统？

**要点**：

1. **日志级别**：合理使用 ERROR/WARN/INFO/DEBUG
2. **结构化**：JSON 格式，易于解析
3. **上下文**：requestId、userId、correlationId
4. **脱敏**：过滤敏感信息
5. **轮转**：避免单个文件过大
6. **聚合**：集中管理（ELK/Loki）
7. **采样**：高频日志采样
8. **保留**：定期清理旧日志

### 3. 生产环境日志应该记录什么？

**必须记录**：
- 错误和异常（堆栈）
- 重要业务操作（订单、支付）
- 性能指标（慢查询、API 响应时间）
- 安全事件（登录、权限变更）
- 外部服务调用

**不要记录**：
- 敏感信息（密码、token）
- 过于详细的调试信息
- 高频的正常操作

### 4. 如何处理日志性能问题？

**方法**：

1. **异步写入**：使用 Pino 或 Winston 异步 transport
2. **采样**：高频日志采样
3. **批量发送**：批量上传到日志系统
4. **缓冲**：本地缓冲后批量写入
5. **压缩**：压缩日志文件
6. **分级存储**：热数据内存，冷数据磁盘

### 5. 分布式系统如何关联日志？

**方案**：

1. **Correlation ID**：整个调用链使用同一个 ID
2. **Trace ID**：分布式追踪（OpenTelemetry）
3. **Session ID**：用户会话 ID
4. **Request ID**：单个请求 ID

```typescript
// 示例
logger.info('Processing request', {
  correlationId: 'abc123',  // 跨服务
  traceId: 'def456',       // 分布式追踪
  spanId: 'ghi789',        // 当前 span
  requestId: 'jkl012',     // 当前请求
  userId: 123
});
```

---

## 总结

### 日志系统选型

| 库 | 性能 | 易用性 | 生态 | 适用场景 |
|----|------|--------|------|---------|
| **Winston** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 通用应用 |
| **Pino** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 高性能应用 |
| **Bunyan** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 结构化日志 |

### 日志聚合选型

| 方案 | 适用场景 | 成本 |
|------|---------|------|
| **ELK** | 大规模、自建 | 高 |
| **Loki** | 中小规模、轻量 | 中 |
| **CloudWatch** | AWS 环境 | 中 |
| **Datadog** | 全栈监控 | 高 |

### 实践检查清单

- [ ] 使用结构化日志
- [ ] 设置合理的日志级别
- [ ] 添加请求上下文（requestId、userId）
- [ ] 脱敏敏感信息
- [ ] 配置日志轮转
- [ ] 集中化日志管理
- [ ] 设置日志保留策略
- [ ] 监控日志错误率
- [ ] 设置告警（错误日志激增）

---

**下一篇**：[APM 和分布式追踪](./02-apm-tracing.md)

