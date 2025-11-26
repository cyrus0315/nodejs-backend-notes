# 错误追踪

错误追踪是快速定位和修复问题的关键。本文深入讲解 Sentry 等错误追踪工具的使用。

## 目录
- [Sentry 详解](#sentry-详解)
- [错误分组与去重](#错误分组与去重)
- [Source Maps](#source-maps)
- [性能监控](#性能监控)
- [告警与集成](#告警与集成)
- [其他工具](#其他工具)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## Sentry 详解

### 安装

```bash
npm install @sentry/node @sentry/profiling-node
```

### 基础配置

```typescript
// sentry.ts
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  release: process.env.APP_VERSION,
  
  // 采样率
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  
  // 集成
  integrations: [
    new ProfilingIntegration(),
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app: true }),
    new Sentry.Integrations.Postgres(),
    new Sentry.Integrations.Mongo()
  ],
  
  // 过滤敏感数据
  beforeSend(event, hint) {
    // 过滤本地环境
    if (process.env.NODE_ENV === 'development') {
      return null;
    }
    
    // 过滤敏感信息
    if (event.request?.headers) {
      delete event.request.headers['authorization'];
      delete event.request.headers['cookie'];
    }
    
    if (event.request?.data) {
      const data = event.request.data;
      if (typeof data === 'object') {
        delete data.password;
        delete data.token;
        delete data.apiKey;
      }
    }
    
    return event;
  },
  
  // 过滤错误
  ignoreErrors: [
    // 网络错误
    'Network Error',
    'NetworkError',
    
    // 用户取消
    'AbortError',
    'CanceledError',
    
    // 404 错误
    'NotFoundError',
    
    // 正则匹配
    /timeout of \d+ms exceeded/
  ],
  
  // 忽略 URL
  denyUrls: [
    /\/health$/,
    /\/metrics$/,
    /\/favicon.ico$/
  ]
});

export default Sentry;
```

### Express 集成

```typescript
// app.ts
import express from 'express';
import Sentry from './sentry';

const app = express();

// 必须在所有中间件之前
app.use(Sentry.Handlers.requestHandler());

// 追踪中间件
app.use(Sentry.Handlers.tracingHandler());

// 你的路由...
app.get('/api/users', async (req, res) => {
  const users = await prisma.user.findMany();
  res.json(users);
});

// 错误处理器（必须在所有中间件之后，但在其他错误处理器之前）
app.use(Sentry.Handlers.errorHandler({
  shouldHandleError(error) {
    // 只捕获 5xx 错误
    return error.status >= 500;
  }
}));

// 自定义错误处理
app.use((err, req, res, next) => {
  // Sentry 已经记录了错误
  
  res.status(err.status || 500).json({
    error: process.env.NODE_ENV === 'production'
      ? 'Internal server error'
      : err.message
  });
});

app.listen(3000);
```

### 手动捕获错误

```typescript
import Sentry from './sentry';

// 捕获异常
try {
  throw new Error('Something went wrong');
} catch (error) {
  Sentry.captureException(error);
}

// 捕获消息
Sentry.captureMessage('User action completed', 'info');

// 带上下文
Sentry.captureException(error, {
  tags: {
    section: 'payment',
    action: 'process'
  },
  user: {
    id: user.id,
    email: user.email,
    username: user.name
  },
  extra: {
    orderId: order.id,
    amount: order.amount
  },
  level: 'error' // 'fatal' | 'error' | 'warning' | 'info' | 'debug'
});

// 添加面包屑（用户操作路径）
Sentry.addBreadcrumb({
  category: 'auth',
  message: 'User logged in',
  level: 'info',
  data: {
    userId: user.id
  }
});

// 设置用户
Sentry.setUser({
  id: user.id,
  email: user.email,
  username: user.name,
  ip_address: req.ip
});

// 设置标签
Sentry.setTag('environment', process.env.NODE_ENV);
Sentry.setTag('feature', 'payment');

// 设置上下文
Sentry.setContext('order', {
  id: order.id,
  amount: order.amount,
  status: order.status
});
```

### Scope 管理

```typescript
// 使用 Scope 隔离上下文
app.get('/api/orders/:id', async (req, res) => {
  Sentry.withScope((scope) => {
    // 设置标签
    scope.setTag('route', '/api/orders/:id');
    scope.setTag('orderId', req.params.id);
    
    // 设置用户
    scope.setUser({
      id: req.user.id,
      email: req.user.email
    });
    
    // 设置上下文
    scope.setContext('request', {
      method: req.method,
      url: req.url,
      headers: req.headers
    });
    
    try {
      const order = await getOrder(req.params.id);
      res.json(order);
    } catch (error) {
      // 错误会自动包含 scope 中的信息
      Sentry.captureException(error);
      res.status(500).json({ error: 'Failed to fetch order' });
    }
  });
});

// 配置 Scope（全局）
Sentry.configureScope((scope) => {
  scope.setTag('server', os.hostname());
  scope.setTag('version', process.env.APP_VERSION);
});
```

---

## 错误分组与去重

### 自定义 Fingerprint

```typescript
// 默认情况下，Sentry 根据堆栈分组错误
// 自定义分组规则
Sentry.captureException(error, {
  fingerprint: [
    '{{ default }}',  // 使用默认规则
    'custom-group'    // 添加自定义规则
  ]
});

// 示例：按错误类型和路由分组
app.use((err, req, res, next) => {
  Sentry.withScope((scope) => {
    scope.setFingerprint([
      err.name,           // 错误类型
      req.route?.path     // 路由
    ]);
    
    Sentry.captureException(err);
  });
  
  next(err);
});

// 示例：按业务逻辑分组
try {
  await processPayment(order);
} catch (error) {
  Sentry.captureException(error, {
    fingerprint: [
      'payment-processing',
      order.paymentMethod
    ]
  });
}

// 完全自定义分组
Sentry.captureException(error, {
  fingerprint: ['my-custom-group'] // 所有使用此 fingerprint 的错误都会分到一组
});
```

### 错误级别

```typescript
// Sentry 错误级别
enum Level {
  Fatal = 'fatal',     // 致命错误，应用崩溃
  Error = 'error',     // 错误，需要处理
  Warning = 'warning', // 警告，潜在问题
  Info = 'info',       // 信息
  Debug = 'debug'      // 调试
}

// 使用
Sentry.captureMessage('User action', 'info');
Sentry.captureException(error, { level: 'warning' });

// 根据错误类型设置级别
app.use((err, req, res, next) => {
  let level: Level;
  
  if (err.name === 'ValidationError') {
    level = 'warning';
  } else if (err.status >= 500) {
    level = 'error';
  } else {
    level = 'info';
  }
  
  Sentry.withScope((scope) => {
    scope.setLevel(level);
    Sentry.captureException(err);
  });
  
  next(err);
});
```

---

## Source Maps

### 生成 Source Maps

```json
// tsconfig.json
{
  "compilerOptions": {
    "sourceMap": true,
    "inlineSources": true,
    "sourceRoot": "/"
  }
}
```

```javascript
// webpack.config.js
module.exports = {
  devtool: 'source-map',
  output: {
    filename: '[name].[contenthash].js',
    sourceMapFilename: '[name].[contenthash].js.map'
  }
};
```

### 上传 Source Maps

```bash
# 安装 Sentry CLI
npm install @sentry/cli --save-dev

# 配置
npx sentry-cli login

# 上传 Source Maps
npx sentry-cli releases files <release-version> upload-sourcemaps ./dist \
  --url-prefix '~/' \
  --validate \
  --strip-prefix /path/to/project
```

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "release": "npm run build && npm run sentry:release",
    "sentry:release": "sentry-cli releases new $npm_package_version && sentry-cli releases files $npm_package_version upload-sourcemaps ./dist && sentry-cli releases finalize $npm_package_version"
  }
}
```

### 自动上传（Webpack 插件）

```javascript
// webpack.config.js
const SentryWebpackPlugin = require('@sentry/webpack-plugin');

module.exports = {
  devtool: 'source-map',
  plugins: [
    new SentryWebpackPlugin({
      org: 'my-org',
      project: 'my-project',
      authToken: process.env.SENTRY_AUTH_TOKEN,
      
      include: './dist',
      ignore: ['node_modules'],
      
      release: process.env.APP_VERSION,
      
      // 上传后删除 Source Maps
      cleanArtifacts: true,
      
      // 设置 commit
      setCommits: {
        auto: true
      }
    })
  ]
};
```

---

## 性能监控

### 启用性能监控

```typescript
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  
  // 性能监控采样率
  tracesSampleRate: 0.1, // 10%
  
  // 启用性能监控
  integrations: [
    new Sentry.Integrations.Http({ tracing: true })
  ]
});
```

### 手动创建 Transaction

```typescript
// 创建事务
const transaction = Sentry.startTransaction({
  op: 'task',
  name: 'Process Order'
});

try {
  // 创建 Span（子操作）
  const span1 = transaction.startChild({
    op: 'db',
    description: 'Fetch user'
  });
  const user = await prisma.user.findUnique({ where: { id: userId } });
  span1.finish();

  const span2 = transaction.startChild({
    op: 'payment',
    description: 'Process payment'
  });
  await processPayment(order);
  span2.finish();

  const span3 = transaction.startChild({
    op: 'notification',
    description: 'Send email'
  });
  await sendEmail(user.email, order);
  span3.finish();

  transaction.setStatus('ok');
} catch (error) {
  transaction.setStatus('internal_error');
  Sentry.captureException(error);
} finally {
  transaction.finish();
}
```

### 自动追踪

```typescript
// HTTP 请求自动追踪
Sentry.init({
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app: true })
  ]
});

// 数据库查询自动追踪
Sentry.init({
  integrations: [
    new Sentry.Integrations.Postgres(),
    new Sentry.Integrations.Mongo()
  ]
});

// Prisma 追踪
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// 中间件
prisma.$use(async (params, next) => {
  const transaction = Sentry.getCurrentHub().getScope()?.getTransaction();
  
  if (transaction) {
    const span = transaction.startChild({
      op: 'db.query',
      description: `${params.model}.${params.action}`
    });
    
    try {
      const result = await next(params);
      span.setStatus('ok');
      return result;
    } catch (error) {
      span.setStatus('internal_error');
      throw error;
    } finally {
      span.finish();
    }
  }
  
  return next(params);
});
```

---

## 告警与集成

### Slack 集成

```typescript
// Sentry 可以直接集成 Slack（在 Sentry 控制台配置）
// 或者自定义 Webhook

import Sentry from './sentry';

Sentry.init({
  beforeSend(event, hint) {
    // 发送到 Slack
    if (event.level === 'error' || event.level === 'fatal') {
      sendToSlack(event);
    }
    
    return event;
  }
});

async function sendToSlack(event: any) {
  await fetch(process.env.SLACK_WEBHOOK_URL!, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      text: `🚨 Error in ${event.environment}`,
      attachments: [{
        color: 'danger',
        title: event.exception?.values?.[0]?.type || 'Error',
        text: event.exception?.values?.[0]?.value || 'Unknown error',
        fields: [
          {
            title: 'Environment',
            value: event.environment,
            short: true
          },
          {
            title: 'Release',
            value: event.release,
            short: true
          },
          {
            title: 'User',
            value: event.user?.email || 'Anonymous',
            short: true
          }
        ],
        footer: 'Sentry',
        ts: Date.now() / 1000
      }]
    })
  });
}
```

### PagerDuty 集成

```typescript
// 严重错误触发 PagerDuty
Sentry.init({
  beforeSend(event, hint) {
    if (event.level === 'fatal') {
      triggerPagerDuty(event);
    }
    return event;
  }
});

async function triggerPagerDuty(event: any) {
  await fetch('https://events.pagerduty.com/v2/enqueue', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      routing_key: process.env.PAGERDUTY_ROUTING_KEY,
      event_action: 'trigger',
      payload: {
        summary: event.exception?.values?.[0]?.value || 'Critical error',
        severity: 'critical',
        source: event.server_name,
        custom_details: {
          environment: event.environment,
          release: event.release,
          user: event.user?.email
        }
      }
    })
  });
}
```

### 告警规则

```typescript
// 在代码中设置告警逻辑
class ErrorRateMonitor {
  private errors = 0;
  private requests = 0;
  private lastAlert = 0;

  recordError() {
    this.errors++;
    this.checkErrorRate();
  }

  recordRequest() {
    this.requests++;
  }

  private checkErrorRate() {
    if (this.requests < 100) return; // 至少 100 个请求

    const errorRate = this.errors / this.requests;

    // 错误率超过 5%
    if (errorRate > 0.05) {
      const now = Date.now();
      
      // 5 分钟内只告警一次
      if (now - this.lastAlert > 5 * 60 * 1000) {
        Sentry.captureMessage(`High error rate: ${(errorRate * 100).toFixed(2)}%`, {
          level: 'critical',
          tags: {
            alert_type: 'error_rate'
          }
        });
        
        this.lastAlert = now;
      }
    }
  }

  // 每分钟重置
  reset() {
    this.errors = 0;
    this.requests = 0;
  }
}

const monitor = new ErrorRateMonitor();
setInterval(() => monitor.reset(), 60000);

app.use((req, res, next) => {
  monitor.recordRequest();
  
  res.on('finish', () => {
    if (res.statusCode >= 500) {
      monitor.recordError();
    }
  });
  
  next();
});
```

---

## 其他工具

### Rollbar

```bash
npm install rollbar
```

```typescript
import Rollbar from 'rollbar';

const rollbar = new Rollbar({
  accessToken: process.env.ROLLBAR_ACCESS_TOKEN,
  environment: process.env.NODE_ENV,
  captureUncaught: true,
  captureUnhandledRejections: true,
  
  payload: {
    server: {
      root: __dirname,
      branch: 'main'
    }
  }
});

// Express 集成
app.use(rollbar.errorHandler());

// 手动记录
rollbar.error('Something went wrong', { userId: 123 });
rollbar.warning('Potential issue', { action: 'payment' });
rollbar.info('User action', { event: 'login' });
```

### Bugsnag

```bash
npm install @bugsnag/js @bugsnag/plugin-express
```

```typescript
import Bugsnag from '@bugsnag/js';
import BugsnagPluginExpress from '@bugsnag/plugin-express';

Bugsnag.start({
  apiKey: process.env.BUGSNAG_API_KEY,
  releaseStage: process.env.NODE_ENV,
  plugins: [BugsnagPluginExpress]
});

const middleware = Bugsnag.getPlugin('express');

// Express 集成
app.use(middleware.requestHandler);
app.use(middleware.errorHandler);

// 手动记录
Bugsnag.notify(new Error('Something went wrong'), (event) => {
  event.addMetadata('user', {
    id: user.id,
    email: user.email
  });
});
```

---

## 最佳实践

### 1. 错误上下文

```typescript
// ✅ 好的做法：丰富的上下文
try {
  await processOrder(order);
} catch (error) {
  Sentry.withScope((scope) => {
    scope.setTag('operation', 'process_order');
    scope.setUser({ id: user.id, email: user.email });
    scope.setContext('order', {
      id: order.id,
      amount: order.amount,
      items: order.items.length
    });
    scope.setContext('payment', {
      method: order.paymentMethod,
      provider: order.paymentProvider
    });
    
    Sentry.captureException(error);
  });
}

// ❌ 不好的做法：缺少上下文
try {
  await processOrder(order);
} catch (error) {
  Sentry.captureException(error);
}
```

### 2. 面包屑

```typescript
// 记录用户操作路径
app.use((req, res, next) => {
  Sentry.addBreadcrumb({
    category: 'http',
    message: `${req.method} ${req.url}`,
    level: 'info',
    data: {
      method: req.method,
      url: req.url,
      status_code: res.statusCode
    }
  });
  
  next();
});

// 业务操作
async function createOrder(userId: number, items: any[]) {
  Sentry.addBreadcrumb({
    category: 'order',
    message: 'Creating order',
    level: 'info',
    data: { userId, itemCount: items.length }
  });
  
  const order = await prisma.order.create({ data: { userId, items } });
  
  Sentry.addBreadcrumb({
    category: 'order',
    message: 'Order created',
    level: 'info',
    data: { orderId: order.id }
  });
  
  return order;
}
```

### 3. 采样策略

```typescript
// 动态采样
Sentry.init({
  beforeSend(event, hint) {
    // 总是发送错误和致命错误
    if (event.level === 'error' || event.level === 'fatal') {
      return event;
    }
    
    // 警告：50% 采样
    if (event.level === 'warning') {
      return Math.random() < 0.5 ? event : null;
    }
    
    // 信息：10% 采样
    return Math.random() < 0.1 ? event : null;
  }
});
```

### 4. 性能监控采样

```typescript
Sentry.init({
  tracesSampler(samplingContext) {
    // 总是追踪关键端点
    if (samplingContext.request?.url?.includes('/api/payment')) {
      return 1.0; // 100%
    }
    
    // 健康检查不追踪
    if (samplingContext.request?.url?.includes('/health')) {
      return 0;
    }
    
    // 其他端点：10% 采样
    return 0.1;
  }
});
```

---

## 常见面试题

### 1. 为什么需要错误追踪工具？

**答案**：

1. **快速定位**：精确的错误堆栈和上下文
2. **错误分组**：相同错误归为一组，避免重复
3. **影响分析**：了解错误影响的用户数
4. **趋势分析**：错误趋势、新旧错误对比
5. **告警通知**：实时告警
6. **Release 追踪**：关联到具体版本

### 2. Sentry vs 日志，有什么区别？

| 特性 | Sentry | 日志 |
|------|--------|------|
| **目的** | 错误追踪 | 记录事件 |
| **结构** | 错误分组、堆栈 | 文本流 |
| **可视化** | Dashboard、图表 | 需要额外工具 |
| **告警** | 内置 | 需要配置 |
| **Source Maps** | 支持 | 不支持 |
| **用户影响** | 追踪受影响用户 | 难以统计 |

**结论**：两者互补，都需要。

### 3. 如何减少 Sentry 成本？

**方法**：

1. **采样**：
   - 生产环境 10-20% 采样
   - 关键端点 100% 采样

2. **过滤**：
   - 忽略已知错误
   - 过滤 4xx 错误
   - 忽略健康检查

3. **去重**：
   - 合理设置 fingerprint
   - 避免重复上报

4. **数据保留**：
   - 减少数据保留时间

5. **环境隔离**：
   - 开发环境不上报

### 4. 如何调试生产环境错误？

**步骤**：

1. **查看错误堆栈**：
   - 使用 Source Maps 还原
   - 定位具体代码行

2. **分析上下文**：
   - 用户信息
   - 请求参数
   - 环境变量

3. **查看面包屑**：
   - 用户操作路径
   - 导致错误的步骤

4. **对比版本**：
   - 是否是新版本引入
   - 对比不同版本的错误率

5. **本地复现**：
   - 根据上下文在本地复现
   - 编写测试用例

### 5. Source Maps 为什么重要？

**原因**：

1. **还原代码**：
   - 生产环境代码压缩/混淆
   - Source Maps 还原为原始代码

2. **精确定位**：
   - 准确的文件名和行号
   - 可读的变量名

3. **调试效率**：
   - 快速理解错误上下文
   - 无需猜测对应的源代码

**注意**：
- Source Maps 不要部署到生产
- 只上传到 Sentry
- 设置访问权限

---

## 总结

### 错误追踪工具对比

| 工具 | 价格 | 功能 | 易用性 | 推荐度 |
|------|------|------|--------|--------|
| **Sentry** | $$ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Rollbar** | $$ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **Bugsnag** | $$ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **自建** | $ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |

### 实践检查清单

- [ ] 集成错误追踪工具
- [ ] 配置环境和版本
- [ ] 设置用户上下文
- [ ] 过滤敏感信息
- [ ] 添加面包屑
- [ ] 上传 Source Maps
- [ ] 配置告警规则
- [ ] 设置采样策略
- [ ] 集成 Slack/PagerDuty
- [ ] 定期审查错误

---

**上一篇**：[监控指标](./03-metrics-monitoring.md)  
**下一篇**：[告警与健康检查](./05-alerting.md)

