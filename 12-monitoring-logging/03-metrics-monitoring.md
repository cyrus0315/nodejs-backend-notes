# 监控指标

监控指标是观察系统健康状况的重要手段。本文深入讲解 Prometheus、Grafana 和监控方法论。

## 目录
- [监控方法论](#监控方法论)
- [Prometheus](#prometheus)
- [Grafana](#grafana)
- [Node.js 指标](#nodejs-指标)
- [业务指标](#业务指标)
- [CloudWatch Metrics](#cloudwatch-metrics)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## 监控方法论

### 四大黄金信号

```typescript
// Google SRE 提出的四大黄金信号

1. 延迟（Latency）
   - 请求响应时间
   - 示例：P50=50ms, P95=200ms, P99=500ms

2. 流量（Traffic）
   - 请求量
   - 示例：1000 req/s

3. 错误（Errors）
   - 错误率
   - 示例：0.1% error rate

4. 饱和度（Saturation）
   - 资源使用率
   - 示例：CPU 70%, Memory 80%
```

### RED 方法

```typescript
// 面向请求的监控方法

1. Rate（速率）
   - 每秒请求数
   - 示例：requests_per_second

2. Errors（错误）
   - 错误请求数/率
   - 示例：error_rate

3. Duration（持续时间）
   - 请求延迟
   - 示例：request_duration_seconds
```

### USE 方法

```typescript
// 面向资源的监控方法

1. Utilization（利用率）
   - CPU、内存、磁盘、网络使用率
   - 示例：cpu_utilization

2. Saturation（饱和度）
   - 队列长度、等待时间
   - 示例：queue_length

3. Errors（错误）
   - 资源错误
   - 示例：disk_errors
```

---

## Prometheus

### 安装

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

volumes:
  prometheus_data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# 告警规则
rule_files:
  - "alerts.yml"

# 抓取配置
scrape_configs:
  # Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter（系统指标）
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # 应用指标
  - job_name: 'my-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'
```

### Node.js 集成

```bash
npm install prom-client
```

```typescript
import express from 'express';
import client from 'prom-client';

const app = express();

// 创建 Registry
const register = new client.Registry();

// 默认指标（进程、系统）
client.collectDefaultMetrics({ register });

// 自定义指标
// 1. Counter（计数器）：只增不减
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

// 2. Gauge（仪表盘）：可增可减
const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register]
});

// 3. Histogram（直方图）：观察值分布
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5], // 分桶
  registers: [register]
});

// 4. Summary（摘要）：观察值统计
const httpRequestSize = new client.Summary({
  name: 'http_request_size_bytes',
  help: 'Size of HTTP requests in bytes',
  labelNames: ['method', 'route'],
  percentiles: [0.5, 0.9, 0.99], // P50, P90, P99
  registers: [register]
});

// 中间件：记录请求指标
app.use((req, res, next) => {
  const start = Date.now();

  // 记录活跃连接
  activeConnections.inc();

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path;

    // 记录请求总数
    httpRequestsTotal.inc({
      method: req.method,
      route,
      status_code: res.statusCode
    });

    // 记录请求延迟
    httpRequestDuration.observe(
      {
        method: req.method,
        route,
        status_code: res.statusCode
      },
      duration
    );

    // 记录请求大小
    const requestSize = parseInt(req.headers['content-length'] || '0', 10);
    httpRequestSize.observe(
      {
        method: req.method,
        route
      },
      requestSize
    );

    // 减少活跃连接
    activeConnections.dec();
  });

  next();
});

// 暴露指标端点
app.get('/metrics', async (req, res) => {
  res.setHeader('Content-Type', register.contentType);
  res.send(await register.metrics());
});

app.listen(3000);
```

### 自定义指标

```typescript
// 业务指标
const ordersTotal = new client.Counter({
  name: 'orders_total',
  help: 'Total number of orders',
  labelNames: ['status'], // 'created', 'paid', 'shipped', 'cancelled'
  registers: [register]
});

const orderAmount = new client.Histogram({
  name: 'order_amount_usd',
  help: 'Order amount in USD',
  buckets: [10, 50, 100, 500, 1000, 5000],
  registers: [register]
});

const queueSize = new client.Gauge({
  name: 'queue_size',
  help: 'Current queue size',
  labelNames: ['queue_name'],
  registers: [register]
});

// 使用
app.post('/api/orders', async (req, res) => {
  try {
    const order = await createOrder(req.body);

    // 记录订单创建
    ordersTotal.inc({ status: 'created' });

    // 记录订单金额
    orderAmount.observe(order.amount);

    res.json(order);
  } catch (error) {
    res.status(500).json({ error: 'Failed to create order' });
  }
});

// 定期更新队列大小
setInterval(() => {
  queueSize.set({ queue_name: 'email' }, emailQueue.length);
  queueSize.set({ queue_name: 'notification' }, notificationQueue.length);
}, 5000);
```

### PromQL 查询

```promql
# 1. 请求速率（每秒）
rate(http_requests_total[5m])

# 2. 按路由分组的请求速率
sum(rate(http_requests_total[5m])) by (route)

# 3. 错误率
sum(rate(http_requests_total{status_code=~"5.."}[5m])) 
/ 
sum(rate(http_requests_total[5m]))

# 4. P95 延迟
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# 5. 按路由分组的 P95 延迟
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (route, le)
)

# 6. CPU 使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 7. 内存使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 8. 磁盘使用率
(node_filesystem_size_bytes - node_filesystem_avail_bytes) 
/ 
node_filesystem_size_bytes * 100

# 9. 最近 5 分钟的错误总数
sum(increase(http_requests_total{status_code=~"5.."}[5m]))

# 10. 活跃连接数变化趋势
delta(active_connections[1h])
```

### 告警规则

```yaml
# alerts.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # 高错误率
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            / 
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # 高延迟
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s"

      # CPU 使用率高
      - alert: HighCPU
        expr: |
          100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage"
          description: "CPU usage is {{ $value | humanize }}%"

      # 内存使用率高
      - alert: HighMemory
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanize }}%"

      # 磁盘空间不足
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space"
          description: "Disk space is {{ $value | humanize }}% free"

      # 服务不可用
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} is down"
```

---

## Grafana

### Docker 启动

```yaml
# docker-compose.yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SECURITY_ADMIN_USER=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

volumes:
  grafana_data:
```

### 数据源配置

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
```

### Dashboard JSON

```json
{
  "dashboard": {
    "title": "API Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (route)",
            "legendFormat": "{{route}}"
          }
        ],
        "yaxes": [
          {
            "format": "reqps",
            "label": "Requests per second"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
            "legendFormat": "Error Rate"
          }
        ],
        "yaxes": [
          {
            "format": "percentunit",
            "label": "Error Rate"
          }
        ]
      },
      {
        "title": "Response Time (P95)",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (route, le))",
            "legendFormat": "{{route}}"
          }
        ],
        "yaxes": [
          {
            "format": "s",
            "label": "Duration"
          }
        ]
      }
    ]
  }
}
```

### Grafana API

```typescript
// 通过 API 创建 Dashboard
import fetch from 'node-fetch';

const GRAFANA_URL = 'http://localhost:3000';
const API_KEY = process.env.GRAFANA_API_KEY;

async function createDashboard(dashboard: any) {
  const response = await fetch(`${GRAFANA_URL}/api/dashboards/db`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      dashboard,
      overwrite: true
    })
  });

  return await response.json();
}

// 创建告警
async function createAlert(alert: any) {
  const response = await fetch(`${GRAFANA_URL}/api/alerts`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(alert)
  });

  return await response.json();
}
```

---

## Node.js 指标

### 进程指标

```typescript
import client from 'prom-client';
import v8 from 'v8';
import process from 'process';

// 默认指标（包含进程指标）
client.collectDefaultMetrics();

// 自定义 Node.js 指标
const heapSizeGauge = new client.Gauge({
  name: 'nodejs_heap_size_total_bytes',
  help: 'Total heap size'
});

const heapUsedGauge = new client.Gauge({
  name: 'nodejs_heap_size_used_bytes',
  help: 'Used heap size'
});

const eventLoopLag = new client.Histogram({
  name: 'nodejs_eventloop_lag_seconds',
  help: 'Event loop lag in seconds',
  buckets: [0.001, 0.01, 0.1, 1, 10]
});

// 定期更新
setInterval(() => {
  const heapStats = v8.getHeapStatistics();
  heapSizeGauge.set(heapStats.total_heap_size);
  heapUsedGauge.set(heapStats.used_heap_size);

  // 测量 Event Loop Lag
  const start = Date.now();
  setImmediate(() => {
    const lag = (Date.now() - start) / 1000;
    eventLoopLag.observe(lag);
  });
}, 5000);
```

### Event Loop 监控

```typescript
import { performance, PerformanceObserver } from 'perf_hooks';

// 监控 Event Loop 延迟
const eventLoopDelayHistogram = new client.Histogram({
  name: 'nodejs_eventloop_delay_seconds',
  help: 'Event loop delay',
  buckets: [0.001, 0.01, 0.05, 0.1, 0.5, 1]
});

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((entry) => {
    eventLoopDelayHistogram.observe(entry.duration / 1000);
  });
});

obs.observe({ entryTypes: ['measure'] });

// 定期测量
setInterval(() => {
  performance.mark('A');
  setImmediate(() => {
    performance.mark('B');
    performance.measure('EventLoop', 'A', 'B');
  });
}, 1000);
```

### GC 监控

```typescript
import { PerformanceObserver } from 'perf_hooks';

const gcDuration = new client.Histogram({
  name: 'nodejs_gc_duration_seconds',
  help: 'GC duration in seconds',
  labelNames: ['kind'],
  buckets: [0.001, 0.01, 0.1, 1, 10]
});

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((entry) => {
    const gcKind = entry.detail?.kind || 'unknown';
    gcDuration.observe({ kind: gcKind }, entry.duration / 1000);
  });
});

obs.observe({ entryTypes: ['gc'] });
```

---

## 业务指标

### 用户行为指标

```typescript
// 用户注册
const userRegistrations = new client.Counter({
  name: 'user_registrations_total',
  help: 'Total user registrations',
  labelNames: ['source'] // 'web', 'mobile', 'api'
});

// 用户登录
const userLogins = new client.Counter({
  name: 'user_logins_total',
  help: 'Total user logins',
  labelNames: ['method'] // 'password', 'oauth', '2fa'
});

// 活跃用户
const activeUsers = new client.Gauge({
  name: 'active_users',
  help: 'Number of active users',
  labelNames: ['timeframe'] // '5m', '1h', '1d'
});

// 使用
app.post('/api/register', async (req, res) => {
  await createUser(req.body);
  userRegistrations.inc({ source: 'web' });
  res.json({ success: true });
});

app.post('/api/login', async (req, res) => {
  await authenticateUser(req.body);
  userLogins.inc({ method: 'password' });
  res.json({ success: true });
});

// 定期更新活跃用户
setInterval(async () => {
  const count5m = await getActiveUsers('5m');
  const count1h = await getActiveUsers('1h');
  const count1d = await getActiveUsers('1d');

  activeUsers.set({ timeframe: '5m' }, count5m);
  activeUsers.set({ timeframe: '1h' }, count1h);
  activeUsers.set({ timeframe: '1d' }, count1d);
}, 60000);
```

### 业务操作指标

```typescript
// 订单指标
const ordersCreated = new client.Counter({
  name: 'orders_created_total',
  help: 'Total orders created'
});

const ordersCompleted = new client.Counter({
  name: 'orders_completed_total',
  help: 'Total orders completed'
});

const ordersCancelled = new client.Counter({
  name: 'orders_cancelled_total',
  help: 'Total orders cancelled'
});

const orderProcessingDuration = new client.Histogram({
  name: 'order_processing_duration_seconds',
  help: 'Order processing duration',
  buckets: [1, 5, 10, 30, 60, 300]
});

const orderRevenue = new client.Counter({
  name: 'order_revenue_usd_total',
  help: 'Total order revenue in USD'
});

// 使用
app.post('/api/orders', async (req, res) => {
  const start = Date.now();

  const order = await createOrder(req.body);
  ordersCreated.inc();

  // 处理订单...
  await processOrder(order);

  const duration = (Date.now() - start) / 1000;
  orderProcessingDuration.observe(duration);

  ordersCompleted.inc();
  orderRevenue.inc(order.amount);

  res.json(order);
});
```

### 资源使用指标

```typescript
// 数据库连接池
const dbPoolSize = new client.Gauge({
  name: 'db_pool_size',
  help: 'Database connection pool size',
  labelNames: ['state'] // 'idle', 'active'
});

const dbQueryDuration = new client.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query duration',
  labelNames: ['operation'], // 'select', 'insert', 'update', 'delete'
  buckets: [0.001, 0.01, 0.1, 1, 10]
});

// 缓存指标
const cacheHits = new client.Counter({
  name: 'cache_hits_total',
  help: 'Total cache hits',
  labelNames: ['cache'] // 'redis', 'memory'
});

const cacheMisses = new client.Counter({
  name: 'cache_misses_total',
  help: 'Total cache misses',
  labelNames: ['cache']
});

// 使用
async function getCachedData(key: string) {
  const cached = await redis.get(key);

  if (cached) {
    cacheHits.inc({ cache: 'redis' });
    return JSON.parse(cached);
  }

  cacheMisses.inc({ cache: 'redis' });

  const data = await fetchFromDatabase(key);
  await redis.setex(key, 3600, JSON.stringify(data));

  return data;
}

// 定期更新连接池指标
setInterval(async () => {
  const poolStats = await prisma.$metrics.json();
  dbPoolSize.set({ state: 'idle' }, poolStats.pool.idle);
  dbPoolSize.set({ state: 'active' }, poolStats.pool.active);
}, 5000);
```

---

## CloudWatch Metrics

### 发送自定义指标

```typescript
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cloudwatch = new CloudWatchClient({ region: 'us-east-1' });

async function putMetric(
  metricName: string,
  value: number,
  unit: string = 'Count',
  dimensions: Record<string, string> = {}
) {
  const params = {
    Namespace: 'MyApp',
    MetricData: [
      {
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: Object.entries(dimensions).map(([Name, Value]) => ({
          Name,
          Value
        }))
      }
    ]
  };

  await cloudwatch.send(new PutMetricDataCommand(params));
}

// 使用
await putMetric('OrdersCreated', 1, 'Count', { Environment: 'production' });
await putMetric('ResponseTime', 150, 'Milliseconds', { Endpoint: '/api/users' });
```

### CloudWatch Embedded Metrics

```typescript
import { MetricsLogger } from 'aws-embedded-metrics';

const metrics = new MetricsLogger();

app.get('/api/users', async (req, res) => {
  const start = Date.now();

  const users = await prisma.user.findMany();

  const duration = Date.now() - start;

  // 记录指标
  metrics.putDimensions({ Service: 'API', Endpoint: '/users' });
  metrics.putMetric('Duration', duration, 'Milliseconds');
  metrics.putMetric('UserCount', users.length, 'Count');
  metrics.setProperty('RequestId', req.id);

  await metrics.flush();

  res.json(users);
});
```

---

## 最佳实践

### 1. 指标命名规范

```typescript
// ✅ 好的命名
http_requests_total          // 清晰、描述性
http_request_duration_seconds // 包含单位
db_query_errors_total        // 前缀表示组件

// ❌ 不好的命名
requests                     // 太笼统
time                        // 不清楚是什么时间
error                       // 缺少上下文
```

### 2. 合理使用标签

```typescript
// ✅ 好的标签使用
httpRequestsTotal.inc({
  method: 'GET',
  route: '/api/users',
  status_code: '200'
});

// ❌ 不好的标签使用（高基数）
httpRequestsTotal.inc({
  user_id: '123456',  // 会产生大量时间序列
  request_id: 'abc',  // 每个请求都不同
  timestamp: Date.now() // 无限增长
});
```

### 3. 设置合理的 bucket

```typescript
// ✅ API 响应时间（秒）
const apiDuration = new client.Histogram({
  name: 'api_duration_seconds',
  help: 'API response time',
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5, 10] // 10ms 到 10s
});

// ✅ 数据库查询时间（秒）
const dbDuration = new client.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query time',
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1] // 1ms 到 1s
});

// ✅ 订单金额（美元）
const orderAmount = new client.Histogram({
  name: 'order_amount_usd',
  help: 'Order amount',
  buckets: [10, 50, 100, 500, 1000, 5000, 10000]
});
```

### 4. 控制指标数量

```typescript
// 定期清理不活跃的指标
const metrics = new Map<string, client.Counter>();

function getOrCreateMetric(name: string) {
  if (!metrics.has(name)) {
    const metric = new client.Counter({ name, help: name });
    metrics.set(name, metric);
  }
  return metrics.get(name)!;
}

// 定期清理（如果指标过多）
setInterval(() => {
  if (metrics.size > 1000) {
    console.warn('Too many metrics, consider reducing');
  }
}, 60000);
```

---

## 常见面试题

### 1. 四大黄金信号是什么？

**答案**：

1. **延迟（Latency）**：请求响应时间
2. **流量（Traffic）**：请求量
3. **错误（Errors）**：错误率
4. **饱和度（Saturation）**：资源使用率

### 2. Counter、Gauge、Histogram、Summary 的区别？

| 类型 | 特点 | 适用场景 | 示例 |
|------|------|---------|------|
| **Counter** | 只增不减 | 累计值 | 请求总数、错误总数 |
| **Gauge** | 可增可减 | 当前值 | 内存使用、活跃连接数 |
| **Histogram** | 分桶统计 | 分布情况 | 响应时间、请求大小 |
| **Summary** | 分位数 | 精确分位 | P50、P95、P99 |

### 3. Histogram vs Summary？

| 特性 | Histogram | Summary |
|------|-----------|---------|
| **计算位置** | 服务端（Prometheus） | 客户端（应用） |
| **灵活性** | 可聚合 | 不可聚合 |
| **精度** | 近似 | 精确 |
| **性能** | 更好 | 较差 |
| **推荐** | ✅ 首选 | 特定场景 |

### 4. 如何监控微服务？

**方法**：

1. **服务级监控**：
   - 请求速率、错误率、延迟（RED）
   - 服务可用性

2. **基础设施监控**：
   - CPU、内存、磁盘、网络（USE）
   - 容器资源使用

3. **业务监控**：
   - 关键业务指标
   - 用户行为

4. **分布式追踪**：
   - 完整调用链
   - 性能瓶颈

5. **日志聚合**：
   - 集中日志
   - 关联追踪

### 5. 如何设置告警阈值？

**方法**：

1. **基于历史数据**：
   - P95 + 20%
   - 平均值 + 2σ

2. **基于 SLA**：
   - 可用性 99.9%
   - 响应时间 < 200ms

3. **动态阈值**：
   - 机器学习预测
   - 异常检测

4. **分级告警**：
   - Warning：超过 70%
   - Critical：超过 90%

---

## 总结

### 监控工具选型

| 工具 | 适用场景 | 优势 | 劣势 |
|------|---------|------|------|
| **Prometheus** | 开源、自建 | 灵活、强大 | 需要维护 |
| **Grafana** | 可视化 | 美观、丰富 | 需配合数据源 |
| **CloudWatch** | AWS 环境 | 集成好 | 功能有限 |
| **Datadog** | 全栈监控 | 全面、易用 | 昂贵 |

### 实践检查清单

- [ ] 监控四大黄金信号
- [ ] 设置合理的指标和标签
- [ ] 配置告警规则
- [ ] 创建可视化 Dashboard
- [ ] 监控业务关键指标
- [ ] 监控基础设施
- [ ] 定期审查和优化
- [ ] 文档化监控方案

---

**上一篇**：[APM 和分布式追踪](./02-apm-tracing.md)  
**下一篇**：[错误追踪](./04-error-tracking.md)

