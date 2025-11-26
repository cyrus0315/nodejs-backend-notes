# APM 和分布式追踪

APM（Application Performance Monitoring）和分布式追踪是监控微服务应用性能的关键技术。本文深入讲解 APM 原理和实践。

## 目录
- [APM 基础](#apm-基础)
- [OpenTelemetry](#opentelemetry)
- [Jaeger 分布式追踪](#jaeger-分布式追踪)
- [New Relic APM](#new-relic-apm)
- [Datadog APM](#datadog-apm)
- [AWS X-Ray](#aws-x-ray)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## APM 基础

### 三大支柱

```
1. 日志（Logs）
   - 离散的事件记录
   - 示例：User logged in, Order created

2. 指标（Metrics）
   - 数值测量
   - 示例：响应时间、吞吐量、错误率

3. 追踪（Traces）
   - 请求的完整路径
   - 示例：API → Service A → Database → Service B
```

### 核心概念

```typescript
// Trace（追踪）：一个完整的请求流程
{
  traceId: "abc123",
  spans: [...]
}

// Span（跨度）：一个操作单元
{
  spanId: "span1",
  traceId: "abc123",
  parentSpanId: null,
  name: "GET /api/users",
  startTime: 1640000000000,
  endTime: 1640000000100,
  duration: 100,
  tags: {
    "http.method": "GET",
    "http.url": "/api/users",
    "http.status_code": 200
  },
  logs: [
    {
      timestamp: 1640000000050,
      message: "Query executed"
    }
  ]
}

// 调用链示例
Trace: abc123
  └─ Span1: GET /api/users (100ms)
      ├─ Span2: Query users table (50ms)
      └─ Span3: POST /api/notifications (30ms)
          └─ Span4: Send email (20ms)
```

---

## OpenTelemetry

### 简介

OpenTelemetry 是 CNCF 的可观测性标准，统一了追踪、指标和日志。

### 安装

```bash
npm install @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-trace-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http
```

### 基础配置

```typescript
// tracing.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-http';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';

// 配置资源
const resource = new Resource({
  [SemanticResourceAttributes.SERVICE_NAME]: 'my-app',
  [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
  [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV
});

// 追踪导出器
const traceExporter = new OTLPTraceExporter({
  url: 'http://localhost:4318/v1/traces' // Jaeger/Tempo
});

// 指标导出器
const metricReader = new PeriodicExportingMetricReader({
  exporter: new OTLPMetricExporter({
    url: 'http://localhost:4318/v1/metrics'
  }),
  exportIntervalMillis: 60000 // 每 60 秒导出一次
});

// 初始化 SDK
const sdk = new NodeSDK({
  resource,
  traceExporter,
  metricReader,
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-fs': {
        enabled: false // 禁用文件系统追踪（噪音太多）
      },
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingPaths: ['/health', '/metrics'] // 忽略健康检查
      }
    })
  ]
});

// 启动
sdk.start();

// 优雅关闭
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});

export default sdk;
```

### 手动追踪

```typescript
// app.ts
import './tracing'; // 必须在应用代码之前导入
import express from 'express';
import { trace, context, SpanStatusCode } from '@opentelemetry/api';

const app = express();
const tracer = trace.getTracer('my-app', '1.0.0');

app.get('/api/users/:id', async (req, res) => {
  // 创建 span
  const span = tracer.startSpan('GET /api/users/:id', {
    attributes: {
      'http.method': 'GET',
      'http.route': '/api/users/:id',
      'user.id': req.params.id
    }
  });

  try {
    // 在 span 上下文中执行
    await context.with(trace.setSpan(context.active(), span), async () => {
      // 子 span：数据库查询
      const dbSpan = tracer.startSpan('db.query', {
        attributes: {
          'db.system': 'postgresql',
          'db.statement': 'SELECT * FROM users WHERE id = $1'
        }
      });

      const user = await prisma.user.findUnique({
        where: { id: parseInt(req.params.id) }
      });

      dbSpan.end();

      if (!user) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: 'User not found'
        });
        res.status(404).json({ error: 'User not found' });
        return;
      }

      // 添加事件
      span.addEvent('User found', {
        'user.email': user.email
      });

      res.json(user);
    });

    span.setStatus({ code: SpanStatusCode.OK });
  } catch (error) {
    span.recordException(error);
    span.setStatus({
      code: SpanStatusCode.ERROR,
      message: error.message
    });
    res.status(500).json({ error: 'Internal server error' });
  } finally {
    span.end();
  }
});
```

### 自动追踪（推荐）

```typescript
// tracing.ts 已经配置了自动追踪
// 自动追踪 HTTP、Express、数据库、Redis 等

// app.ts
import './tracing'; // 导入即可，无需手动创建 span
import express from 'express';

const app = express();

// HTTP 请求自动被追踪
app.get('/api/users', async (req, res) => {
  // 数据库查询自动被追踪
  const users = await prisma.user.findMany();
  
  // Redis 操作自动被追踪
  await redis.set('users:count', users.length);
  
  res.json(users);
});

// Trace 自动包含：
// └─ GET /api/users
//     ├─ prisma.user.findMany
//     └─ redis.set
```

### 传播追踪上下文

```typescript
// 调用下游服务时自动传播 trace context
import fetch from 'node-fetch';
import { context, propagation } from '@opentelemetry/api';

async function callDownstreamService(data: any) {
  const headers: any = {
    'Content-Type': 'application/json'
  };

  // 注入追踪上下文到 headers
  propagation.inject(context.active(), headers);

  const response = await fetch('http://downstream-service/api', {
    method: 'POST',
    headers,
    body: JSON.stringify(data)
  });

  return await response.json();
}

// headers 自动包含：
// traceparent: 00-abc123...-def456...-01
// tracestate: ...
```

---

## Jaeger 分布式追踪

### Docker 启动 Jaeger

```yaml
# docker-compose.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "5775:5775/udp"   # Agent (Deprecated)
      - "6831:6831/udp"   # Agent (Compact)
      - "6832:6832/udp"   # Agent (Binary)
      - "5778:5778"       # Agent (Config)
      - "16686:16686"     # UI
      - "14268:14268"     # Collector HTTP
      - "14250:14250"     # Collector gRPC
      - "9411:9411"       # Zipkin compatible
      - "4317:4317"       # OTLP gRPC
      - "4318:4318"       # OTLP HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
```

```bash
docker-compose up -d

# 访问 UI
open http://localhost:16686
```

### Jaeger 客户端

```typescript
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const exporter = new JaegerExporter({
  endpoint: 'http://localhost:14268/api/traces',
  // 或使用 Agent
  // host: 'localhost',
  // port: 6832
});

const sdk = new NodeSDK({
  traceExporter: exporter,
  instrumentations: [getNodeAutoInstrumentations()]
});

sdk.start();
```

### 查询追踪

```bash
# 通过 API 查询
curl "http://localhost:16686/api/traces?service=my-app&limit=10"

# 查询特定 trace
curl "http://localhost:16686/api/traces/abc123..."
```

---

## New Relic APM

### 安装

```bash
npm install newrelic
```

### 配置

```javascript
// newrelic.js
'use strict';

exports.config = {
  app_name: ['My App'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  logging: {
    level: 'info'
  },
  
  // 分布式追踪
  distributed_tracing: {
    enabled: true
  },
  
  // 事务追踪
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 'apdex_f',
    record_sql: 'obfuscated',
    explain_threshold: 500 // 慢查询阈值（ms）
  },
  
  // 错误收集
  error_collector: {
    enabled: true,
    ignore_status_codes: [404]
  },
  
  // 自定义属性
  attributes: {
    enabled: true,
    include: ['request.headers.user-agent']
  }
};
```

### 使用

```typescript
// app.ts
// 必须在第一行导入
import newrelic from 'newrelic';
import express from 'express';

const app = express();

app.get('/api/users', async (req, res) => {
  // 自动追踪 HTTP 请求和数据库查询
  const users = await prisma.user.findMany();
  res.json(users);
});

// 自定义事务
app.post('/api/process', async (req, res) => {
  const transaction = newrelic.getTransaction();
  
  // 添加自定义属性
  newrelic.addCustomAttribute('jobType', req.body.type);
  newrelic.addCustomAttribute('userId', req.user.id);
  
  // 记录自定义事件
  newrelic.recordCustomEvent('JobProcessed', {
    jobId: job.id,
    duration: job.duration,
    status: 'success'
  });
  
  // 自定义 segment（子操作）
  await newrelic.startSegment('processPayment', true, async () => {
    return await processPayment(req.body);
  });
  
  res.json({ success: true });
});

// 记录错误
app.use((err, req, res, next) => {
  newrelic.noticeError(err, {
    userId: req.user?.id,
    requestId: req.id
  });
  
  res.status(500).json({ error: 'Internal server error' });
});
```

### 自定义指标

```typescript
// 业务指标
newrelic.recordMetric('Custom/Orders/Created', 1);
newrelic.recordMetric('Custom/Revenue', order.amount);

// 定时记录
setInterval(() => {
  const queueSize = queue.length;
  newrelic.recordMetric('Custom/Queue/Size', queueSize);
}, 10000);
```

---

## Datadog APM

### 安装

```bash
npm install dd-trace --save
```

### 配置

```typescript
// tracer.ts
import tracer from 'dd-trace';

tracer.init({
  hostname: process.env.DD_AGENT_HOST || 'localhost',
  port: process.env.DD_AGENT_PORT || 8126,
  
  service: 'my-app',
  env: process.env.NODE_ENV,
  version: process.env.APP_VERSION,
  
  // 日志注入（关联日志和追踪）
  logInjection: true,
  
  // 运行时指标
  runtimeMetrics: true,
  
  // 采样率
  sampleRate: 1.0, // 100%
  
  // 插件配置
  plugins: {
    http: {
      enabled: true
    },
    express: {
      enabled: true
    },
    pg: {
      enabled: true,
      service: 'postgres'
    },
    redis: {
      enabled: true,
      service: 'redis'
    }
  }
});

export default tracer;
```

### 使用

```typescript
// app.ts
import './tracer'; // 必须在第一行导入
import express from 'express';
import tracer from 'dd-trace';

const app = express();

app.get('/api/users', async (req, res) => {
  // 自动追踪
  const users = await prisma.user.findMany();
  
  // 手动 span
  const span = tracer.startSpan('custom.operation', {
    resource: 'processUsers',
    type: 'web'
  });
  
  span.setTag('user.count', users.length);
  
  try {
    // 处理...
    span.setTag('status', 'success');
  } catch (error) {
    span.setTag('error', error);
  } finally {
    span.finish();
  }
  
  res.json(users);
});

// 子 span
async function fetchUserData(userId: number) {
  return tracer.trace('fetchUserData', async (span) => {
    span.setTag('user.id', userId);
    
    const user = await prisma.user.findUnique({
      where: { id: userId }
    });
    
    span.setTag('user.found', !!user);
    
    return user;
  });
}
```

### 自定义指标

```typescript
import { StatsD } from 'hot-shots';

const dogstatsd = new StatsD({
  host: 'localhost',
  port: 8125,
  prefix: 'myapp.',
  globalTags: {
    env: process.env.NODE_ENV,
    service: 'my-app'
  }
});

// 计数器
dogstatsd.increment('orders.created', 1, ['status:success']);

// 计时器
const startTime = Date.now();
await processOrder();
const duration = Date.now() - startTime;
dogstatsd.timing('orders.process_time', duration);

// 仪表盘
dogstatsd.gauge('queue.size', queue.length);

// 直方图
dogstatsd.histogram('response.size', responseBody.length);
```

---

## AWS X-Ray

### 安装

```bash
npm install aws-xray-sdk-core aws-xray-sdk-express
```

### 配置

```typescript
import AWSXRay from 'aws-xray-sdk-core';
import express from 'express';

const app = express();

// X-Ray 中间件（必须在其他中间件之前）
app.use(AWSXRay.express.openSegment('MyApp'));

// 捕获 AWS SDK 调用
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

// 捕获 HTTP 调用
const http = AWSXRay.captureHTTPs(require('http'));
const https = AWSXRay.captureHTTPs(require('https'));

// 捕获数据库查询
AWSXRay.capturePostgres(require('pg'));

// 路由
app.get('/api/users', async (req, res) => {
  // 自定义 subsegment
  const segment = AWSXRay.getSegment();
  const subsegment = segment.addNewSubsegment('fetchUsers');
  
  try {
    subsegment.addAnnotation('userId', req.user.id);
    subsegment.addMetadata('query', { limit: 10 });
    
    const users = await prisma.user.findMany({ take: 10 });
    
    subsegment.close();
    res.json(users);
  } catch (error) {
    subsegment.addError(error);
    subsegment.close();
    throw error;
  }
});

// 关闭 segment（必须在最后）
app.use(AWSXRay.express.closeSegment());
```

### Lambda 集成

```typescript
// Lambda handler with X-Ray
import AWSXRay from 'aws-xray-sdk-core';
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

export const handler = async (event, context) => {
  // Lambda 自动创建 segment
  const segment = AWSXRay.getSegment();
  
  // 添加注解（可索引）
  segment.addAnnotation('userId', event.userId);
  
  // 添加元数据（不可索引）
  segment.addMetadata('event', event);
  
  // 子 segment
  const subsegment = segment.addNewSubsegment('processEvent');
  
  try {
    const result = await processEvent(event);
    subsegment.close();
    return result;
  } catch (error) {
    subsegment.addError(error);
    subsegment.close();
    throw error;
  }
};

async function processEvent(event) {
  return AWSXRay.captureAsyncFunc('processEvent', async (subsegment) => {
    // DynamoDB 调用自动被追踪
    const dynamodb = new AWS.DynamoDB.DocumentClient();
    const result = await dynamodb.get({
      TableName: 'Users',
      Key: { id: event.userId }
    }).promise();
    
    subsegment.close();
    return result;
  });
}
```

---

## 最佳实践

### 1. 采样策略

```typescript
// 基于规则的采样
import { ParentBasedSampler, TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-base';

const sampler = new ParentBasedSampler({
  root: new TraceIdRatioBasedSampler(0.1) // 10% 采样率
});

// 动态采样（根据错误、慢请求等）
class AdaptiveSampler {
  private baseSampleRate = 0.1; // 基础 10%
  
  shouldSample(span) {
    // 错误请求：100% 采样
    if (span.status === 'error') {
      return true;
    }
    
    // 慢请求：100% 采样
    if (span.duration > 1000) {
      return true;
    }
    
    // 特定路径：100% 采样
    if (span.name.includes('/api/critical')) {
      return true;
    }
    
    // 其他请求：基础采样率
    return Math.random() < this.baseSampleRate;
  }
}
```

### 2. 减少开销

```typescript
// 1. 忽略健康检查等高频端点
instrumentations: [
  getNodeAutoInstrumentations({
    '@opentelemetry/instrumentation-http': {
      ignoreIncomingPaths: [
        '/health',
        '/metrics',
        '/favicon.ico'
      ]
    }
  })
]

// 2. 批量导出
const exporter = new OTLPTraceExporter({
  url: 'http://localhost:4318/v1/traces',
  maxQueueSize: 100,
  maxExportBatchSize: 50,
  scheduledDelayMillis: 5000 // 5 秒批量导出
});

// 3. 禁用不必要的插件
instrumentations: [
  getNodeAutoInstrumentations({
    '@opentelemetry/instrumentation-fs': {
      enabled: false
    }
  })
]
```

### 3. 关联日志和追踪

```typescript
import { trace, context } from '@opentelemetry/api';
import winston from 'winston';

// 在日志中添加 trace 信息
const format = winston.format((info) => {
  const span = trace.getSpan(context.active());
  if (span) {
    const spanContext = span.spanContext();
    info.traceId = spanContext.traceId;
    info.spanId = spanContext.spanId;
    info.traceFlags = spanContext.traceFlags;
  }
  return info;
});

const logger = winston.createLogger({
  format: winston.format.combine(
    format(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console()
  ]
});

// 日志输出
logger.info('Processing request', { userId: 123 });
// 输出：{ level: 'info', message: 'Processing request', 
//        userId: 123, traceId: '...', spanId: '...' }
```

---

## 常见面试题

### 1. APM 的三大支柱是什么？

**答案**：

1. **日志（Logs）**：离散的事件记录
2. **指标（Metrics）**：数值测量
3. **追踪（Traces）**：请求的完整路径

它们的关系：
- 日志记录具体事件
- 指标聚合数值数据
- 追踪展示调用链

### 2. Span 和 Trace 的区别？

**答案**：

- **Trace**：一个完整的请求流程，包含多个 Span
- **Span**：一个操作单元（如一次 HTTP 请求、一次数据库查询）

```
Trace (traceId: abc123)
  └─ Span1 (API request)
      ├─ Span2 (DB query)
      └─ Span3 (External API call)
```

### 3. 如何减少 APM 的性能开销？

**方法**：

1. **采样**：不追踪所有请求（10-20%）
2. **忽略高频端点**：健康检查、静态资源
3. **批量导出**：批量发送追踪数据
4. **异步处理**：不阻塞主线程
5. **禁用不必要的插件**：文件系统追踪等

### 4. 如何选择 APM 工具？

| 工具 | 优势 | 适用场景 |
|------|------|---------|
| **OpenTelemetry** | 开源、标准化、供应商中立 | 自建、灵活 |
| **Jaeger** | 开源、轻量、分布式追踪 | 微服务追踪 |
| **New Relic** | 全面、易用、AI 分析 | 商业应用 |
| **Datadog** | 全栈监控、集成好 | 云原生应用 |
| **AWS X-Ray** | AWS 集成好 | AWS 环境 |

### 5. 分布式追踪如何传播上下文？

**答案**：

通过 HTTP headers 传播 trace context：

```
traceparent: 00-{traceId}-{spanId}-{flags}

示例：
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01

格式：
00 - 版本
0af765... - trace ID
b7ad6b... - parent span ID
01 - flags（sampled）
```

服务 A 调用服务 B 时：
1. A 在请求头中注入 `traceparent`
2. B 从请求头中提取 `traceparent`
3. B 创建新的 span，使用相同的 trace ID
4. 整个调用链共享同一个 trace ID

---

## 总结

### APM 工具选型

| 特性 | OpenTelemetry | Jaeger | New Relic | Datadog | X-Ray |
|------|--------------|--------|-----------|---------|-------|
| **成本** | 免费 | 免费 | 付费 | 付费 | 按量付费 |
| **易用性** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **功能** | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **集成** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |

### 实践检查清单

- [ ] 集成 APM 工具
- [ ] 配置自动追踪
- [ ] 设置采样策略
- [ ] 忽略高频端点
- [ ] 关联日志和追踪
- [ ] 监控关键指标
- [ ] 设置告警阈值
- [ ] 定期分析慢请求
- [ ] 优化性能瓶颈

---

**上一篇**：[日志系统](./01-logging.md)  
**下一篇**：[监控指标](./03-metrics-monitoring.md)

