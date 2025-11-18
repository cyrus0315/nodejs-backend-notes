# 可扩展性与高可用设计

系统设计的核心目标是构建**可扩展**和**高可用**的系统。本文深入讲解如何设计和实现可扩展、高可用的后端系统。

## 目录
- [可扩展性设计](#可扩展性设计)
- [高可用设计](#高可用设计)
- [负载均衡](#负载均衡)
- [容错与降级](#容错与降级)
- [Node.js 实战](#nodejs-实战)
- [面试题](#常见面试题)

---

## 可扩展性设计

### 什么是可扩展性？

**可扩展性（Scalability）** 是系统处理增长负载的能力。包括：
- 用户增长
- 数据增长
- 流量增长
- 功能增长

### 垂直扩展 vs 水平扩展

#### 垂直扩展（Scale Up）

升级单台服务器的硬件配置。

**优点**：
- ✅ 实现简单，无需改代码
- ✅ 无分布式复杂度
- ✅ 数据一致性容易保证

**缺点**：
- ❌ 成本高（非线性增长）
- ❌ 有物理上限
- ❌ 单点故障风险
- ❌ 停机时间（升级需要重启）

**适用场景**：
- 传统关系型数据库
- 缓存服务器（Redis）
- 计算密集型应用

```typescript
// 垂直扩展示例配置
// 从 2 核 4GB → 8 核 32GB
const serverConfig = {
  cpu: '8 cores',
  memory: '32GB',
  disk: '500GB SSD'
};
```

#### 水平扩展（Scale Out）

增加服务器数量，分散负载。

**优点**：
- ✅ 成本较低（线性增长）
- ✅ 无理论上限
- ✅ 高可用（无单点故障）
- ✅ 灵活（按需增减）

**缺点**：
- ❌ 架构复杂（分布式）
- ❌ 数据一致性挑战
- ❌ 需要负载均衡
- ❌ 运维复杂度高

**适用场景**：
- Web 服务器
- API 服务器
- 微服务架构
- 无状态应用

```typescript
// 水平扩展示例
// PM2 启动多个实例
module.exports = {
  apps: [{
    name: 'api-server',
    script: './dist/main.js',
    instances: 4,  // 4 个实例
    exec_mode: 'cluster',
    max_memory_restart: '1G'
  }]
};
```

### 无状态设计

**关键原则**：应用服务器不保存会话状态，所有状态存储在外部。

#### ❌ 有状态设计（不推荐）

```typescript
// 问题：状态存储在内存中
const sessions = new Map<string, UserSession>();

app.post('/login', (req, res) => {
  const sessionId = generateId();
  sessions.set(sessionId, { 
    userId: req.body.userId,
    loginTime: Date.now()
  });
  res.json({ sessionId });
});

app.get('/profile', (req, res) => {
  const session = sessions.get(req.headers.sessionid);
  if (!session) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  // 问题：如果请求被负载均衡到另一台服务器，session 不存在
  res.json({ userId: session.userId });
});
```

**问题**：
- 无法水平扩展（会话绑定到特定服务器）
- 服务器重启会话丢失
- 需要 Sticky Session（增加复杂度）

#### ✅ 无状态设计（推荐）

```typescript
import Redis from 'ioredis';
import jwt from 'jsonwebtoken';

const redis = new Redis();

// 方案 1：JWT（推荐用于认证）
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);
  
  const token = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
  
  res.json({ token });
});

app.get('/profile', authenticateJWT, (req, res) => {
  // req.user 由 JWT 中间件解析
  res.json({ userId: req.user.userId });
});

// 方案 2：Redis Session（推荐用于复杂会话）
app.post('/login', async (req, res) => {
  const user = await authenticateUser(req.body);
  const sessionId = generateId();
  
  // 存储到 Redis
  await redis.setex(
    `session:${sessionId}`,
    86400, // 24 小时
    JSON.stringify({
      userId: user.id,
      email: user.email,
      loginTime: Date.now()
    })
  );
  
  res.json({ sessionId });
});

app.get('/profile', async (req, res) => {
  const sessionData = await redis.get(
    `session:${req.headers.sessionid}`
  );
  
  if (!sessionData) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  
  const session = JSON.parse(sessionData);
  res.json({ userId: session.userId });
});
```

**优势**：
- ✅ 任何服务器都能处理任何请求
- ✅ 服务器重启不影响会话
- ✅ 易于水平扩展

### 数据分片（Sharding）

将数据分散到多个数据库实例。

#### 分片策略

**1. 范围分片（Range-based）**

```typescript
// 按用户 ID 范围分片
function getShardByUserId(userId: number): string {
  if (userId < 1_000_000) return 'shard-1';
  if (userId < 2_000_000) return 'shard-2';
  if (userId < 3_000_000) return 'shard-3';
  return 'shard-4';
}

// 使用示例
const shard = getShardByUserId(1_234_567);
const user = await db[shard].user.findUnique({ 
  where: { id: userId } 
});
```

**优点**：
- ✅ 范围查询高效
- ✅ 易于理解和实现

**缺点**：
- ❌ 数据可能不均匀（热点问题）
- ❌ 难以重新平衡

**2. 哈希分片（Hash-based）**

```typescript
import crypto from 'crypto';

// 一致性哈希
class ConsistentHash {
  private ring: Map<number, string> = new Map();
  private sortedKeys: number[] = [];
  private virtualNodes = 150; // 虚拟节点数

  addNode(node: string) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.set(hash, node);
    }
    this.sortedKeys = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  removeNode(node: string) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const hash = this.hash(`${node}:${i}`);
      this.ring.delete(hash);
    }
    this.sortedKeys = Array.from(this.ring.keys()).sort((a, b) => a - b);
  }

  getNode(key: string): string | undefined {
    if (this.sortedKeys.length === 0) return undefined;

    const hash = this.hash(key);
    
    // 找到第一个 >= hash 的节点
    for (const nodeHash of this.sortedKeys) {
      if (nodeHash >= hash) {
        return this.ring.get(nodeHash);
      }
    }
    
    // 如果没找到，返回第一个节点（环形）
    return this.ring.get(this.sortedKeys[0]);
  }

  private hash(key: string): number {
    const hash = crypto.createHash('md5').update(key).digest();
    return hash.readUInt32BE(0);
  }
}

// 使用示例
const hashRing = new ConsistentHash();
hashRing.addNode('shard-1');
hashRing.addNode('shard-2');
hashRing.addNode('shard-3');

async function getUserFromShard(userId: number) {
  const shard = hashRing.getNode(`user:${userId}`);
  return await db[shard].user.findUnique({ where: { id: userId } });
}

// 新增节点，只影响部分数据
hashRing.addNode('shard-4');
```

**优点**：
- ✅ 数据分布均匀
- ✅ 易于动态扩展（一致性哈希）

**缺点**：
- ❌ 范围查询困难
- ❌ 跨分片查询复杂

**3. 地理位置分片**

```typescript
// 按地理位置分片
function getShardByRegion(country: string): string {
  const regionMap: Record<string, string> = {
    'US': 'shard-us-east',
    'CN': 'shard-cn-north',
    'EU': 'shard-eu-west',
    'JP': 'shard-jp-east'
  };
  return regionMap[country] || 'shard-default';
}

// 减少跨区域延迟
const shard = getShardByRegion(user.country);
const data = await db[shard].query(/* ... */);
```

### 读写分离

将读操作和写操作分散到不同的数据库实例。

```typescript
import { PrismaClient } from '@prisma/client';

// 主库（写）
const primary = new PrismaClient({
  datasources: {
    db: { url: process.env.PRIMARY_DB_URL }
  }
});

// 从库（读）
const replicas = [
  new PrismaClient({
    datasources: { db: { url: process.env.REPLICA_1_URL } }
  }),
  new PrismaClient({
    datasources: { db: { url: process.env.REPLICA_2_URL } }
  }),
  new PrismaClient({
    datasources: { db: { url: process.env.REPLICA_3_URL } }
  })
];

// 轮询策略选择从库
let replicaIndex = 0;
function getReadConnection() {
  const replica = replicas[replicaIndex];
  replicaIndex = (replicaIndex + 1) % replicas.length;
  return replica;
}

// 数据库管理器
class DatabaseManager {
  // 写操作：使用主库
  async createUser(data: any) {
    return await primary.user.create({ data });
  }

  async updateUser(id: number, data: any) {
    return await primary.user.update({ where: { id }, data });
  }

  // 读操作：使用从库
  async findUser(id: number) {
    const replica = getReadConnection();
    return await replica.user.findUnique({ where: { id } });
  }

  async findUsers(query: any) {
    const replica = getReadConnection();
    return await replica.user.findMany(query);
  }

  // 强制读主库（需要最新数据）
  async findUserFromPrimary(id: number) {
    return await primary.user.findUnique({ where: { id } });
  }
}

const db = new DatabaseManager();

// 使用示例
app.post('/users', async (req, res) => {
  // 写操作：主库
  const user = await db.createUser(req.body);
  res.json(user);
});

app.get('/users/:id', async (req, res) => {
  // 读操作：从库
  const user = await db.findUser(Number(req.params.id));
  res.json(user);
});

app.post('/users/:id/update-and-get', async (req, res) => {
  // 更新后立即读取：强制读主库
  await db.updateUser(Number(req.params.id), req.body);
  const user = await db.findUserFromPrimary(Number(req.params.id));
  res.json(user);
});
```

**注意事项**：
- **主从延迟**：写入主库后，从库可能需要几百毫秒同步
- **解决方案**：写入后立即读取时，从主库读取

---

## 高可用设计

### 什么是高可用？

**高可用性（High Availability）** 是系统持续提供服务的能力。通常用 **SLA**（Service Level Agreement）衡量。

| SLA | 年停机时间 | 月停机时间 | 周停机时间 |
|-----|-----------|-----------|-----------|
| 99% | 3.65 天 | 7.2 小时 | 1.68 小时 |
| 99.9% (3个9) | 8.76 小时 | 43.8 分钟 | 10.1 分钟 |
| 99.99% (4个9) | 52.56 分钟 | 4.38 分钟 | 1.01 分钟 |
| 99.999% (5个9) | 5.26 分钟 | 25.9 秒 | 6.05 秒 |

### 故障转移（Failover）

当主服务器故障时，自动切换到备用服务器。

```typescript
import Redis from 'ioredis';

// Redis Sentinel（哨兵）模式
const redis = new Redis({
  sentinels: [
    { host: 'sentinel-1', port: 26379 },
    { host: 'sentinel-2', port: 26379 },
    { host: 'sentinel-3', port: 26379 }
  ],
  name: 'mymaster', // 主节点名称
  sentinelRetryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  }
});

// 监听故障转移事件
redis.on('+switch-master', (info) => {
  console.log('Master switched:', info);
  // 主节点切换，通知监控系统
});

redis.on('error', (err) => {
  console.error('Redis error:', err);
});
```

### 健康检查

定期检查服务健康状态。

```typescript
import express from 'express';
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';

const app = express();
const prisma = new PrismaClient();
const redis = new Redis();

interface HealthStatus {
  status: 'healthy' | 'unhealthy' | 'degraded';
  timestamp: string;
  checks: {
    database: { status: string; latency?: number; error?: string };
    redis: { status: string; latency?: number; error?: string };
    memory: { status: string; usage: number; limit: number };
    cpu: { status: string; usage: number };
  };
}

app.get('/health', async (req, res) => {
  const health: HealthStatus = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    checks: {
      database: { status: 'unknown' },
      redis: { status: 'unknown' },
      memory: { status: 'unknown', usage: 0, limit: 0 },
      cpu: { status: 'unknown', usage: 0 }
    }
  };

  // 1. 检查数据库
  try {
    const start = Date.now();
    await prisma.$queryRaw`SELECT 1`;
    const latency = Date.now() - start;
    
    health.checks.database = {
      status: latency < 100 ? 'healthy' : 'degraded',
      latency
    };
  } catch (error) {
    health.checks.database = {
      status: 'unhealthy',
      error: error.message
    };
    health.status = 'unhealthy';
  }

  // 2. 检查 Redis
  try {
    const start = Date.now();
    await redis.ping();
    const latency = Date.now() - start;
    
    health.checks.redis = {
      status: latency < 50 ? 'healthy' : 'degraded',
      latency
    };
  } catch (error) {
    health.checks.redis = {
      status: 'unhealthy',
      error: error.message
    };
    health.status = 'degraded'; // Redis 故障可降级
  }

  // 3. 检查内存
  const memUsage = process.memoryUsage();
  const totalMem = memUsage.heapTotal;
  const usedMem = memUsage.heapUsed;
  const memPercentage = (usedMem / totalMem) * 100;

  health.checks.memory = {
    status: memPercentage < 80 ? 'healthy' : 'degraded',
    usage: Math.round(memPercentage),
    limit: 80
  };

  // 4. 检查 CPU
  const cpuUsage = process.cpuUsage();
  const cpuPercentage = (cpuUsage.user + cpuUsage.system) / 1000000 / 10; // 简化计算

  health.checks.cpu = {
    status: cpuPercentage < 70 ? 'healthy' : 'degraded',
    usage: Math.round(cpuPercentage)
  };

  // 确定整体状态
  if (health.status === 'healthy') {
    const hasDegraded = Object.values(health.checks)
      .some(check => check.status === 'degraded');
    if (hasDegraded) {
      health.status = 'degraded';
    }
  }

  const statusCode = health.status === 'healthy' ? 200 :
                     health.status === 'degraded' ? 200 : 503;

  res.status(statusCode).json(health);
});

// 简单的存活检查（liveness probe）
app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'alive' });
});

// 就绪检查（readiness probe）
app.get('/health/ready', async (req, res) => {
  try {
    // 检查关键依赖
    await Promise.all([
      prisma.$queryRaw`SELECT 1`,
      redis.ping()
    ]);
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: error.message });
  }
});
```

**Kubernetes 配置**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-server
spec:
  containers:
  - name: api
    image: api-server:latest
    ports:
    - containerPort: 3000
    livenessProbe:
      httpGet:
        path: /health/live
        port: 3000
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 2
```

### 熔断器（Circuit Breaker）

防止故障传播，快速失败。

```typescript
enum CircuitState {
  CLOSED = 'CLOSED',   // 正常
  OPEN = 'OPEN',       // 熔断
  HALF_OPEN = 'HALF_OPEN' // 半开（尝试恢复）
}

interface CircuitBreakerOptions {
  failureThreshold: number;  // 失败阈值
  successThreshold: number;  // 成功阈值（用于半开状态）
  timeout: number;           // 熔断时长
  monitoringPeriod: number;  // 监控周期
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private nextAttempt = Date.now();
  private failures: number[] = [];

  constructor(private options: CircuitBreakerOptions) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    // 熔断状态：直接拒绝
    if (this.state === CircuitState.OPEN) {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      // 超过熔断时长，进入半开状态
      this.state = CircuitState.HALF_OPEN;
      this.successCount = 0;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      // 半开状态：连续成功达到阈值，恢复
      if (this.successCount >= this.options.successThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
        console.log('Circuit breaker: HALF_OPEN -> CLOSED');
      }
    }
  }

  private onFailure() {
    this.failures.push(Date.now());
    
    // 清理过期的失败记录
    const cutoff = Date.now() - this.options.monitoringPeriod;
    this.failures = this.failures.filter(time => time > cutoff);
    
    this.failureCount = this.failures.length;

    // 关闭或半开状态：失败次数达到阈值，熔断
    if (
      this.failureCount >= this.options.failureThreshold &&
      this.state !== CircuitState.OPEN
    ) {
      this.state = CircuitState.OPEN;
      this.nextAttempt = Date.now() + this.options.timeout;
      console.log(`Circuit breaker: ${this.state} -> OPEN`);
    }
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount
    };
  }
}

// 使用示例
const paymentServiceBreaker = new CircuitBreaker({
  failureThreshold: 5,    // 5 次失败触发熔断
  successThreshold: 2,    // 半开状态 2 次成功恢复
  timeout: 60000,         // 熔断 60 秒
  monitoringPeriod: 10000 // 10 秒内的失败次数
});

async function callPaymentService(orderId: string) {
  try {
    return await paymentServiceBreaker.execute(async () => {
      // 调用支付服务
      const response = await fetch(`${PAYMENT_SERVICE_URL}/process`, {
        method: 'POST',
        body: JSON.stringify({ orderId }),
        headers: { 'Content-Type': 'application/json' },
        signal: AbortSignal.timeout(5000) // 5 秒超时
      });

      if (!response.ok) {
        throw new Error(`Payment service error: ${response.status}`);
      }

      return await response.json();
    });
  } catch (error) {
    console.error('Payment service call failed:', error.message);
    
    // 熔断时的降级逻辑
    if (error.message.includes('Circuit breaker is OPEN')) {
      // 记录订单，稍后重试
      await queueFailedPayment(orderId);
      return { status: 'queued', message: 'Payment will be processed later' };
    }
    
    throw error;
  }
}
```

### 降级策略

当系统压力过大或部分服务不可用时，降低服务质量以保证核心功能。

```typescript
import { Request, Response, NextFunction } from 'express';

// 功能开关
class FeatureFlags {
  private flags = new Map<string, boolean>();

  constructor(private redis: Redis) {}

  async isEnabled(feature: string): Promise<boolean> {
    // 先查缓存
    if (this.flags.has(feature)) {
      return this.flags.get(feature)!;
    }

    // 从 Redis 读取
    const value = await this.redis.get(`feature:${feature}`);
    const enabled = value === 'true';
    this.flags.set(feature, enabled);
    return enabled;
  }

  async enable(feature: string) {
    await this.redis.set(`feature:${feature}`, 'true');
    this.flags.set(feature, true);
  }

  async disable(feature: string) {
    await this.redis.set(`feature:${feature}`, 'false');
    this.flags.set(feature, false);
  }
}

const featureFlags = new FeatureFlags(redis);

// 降级中间件
function degradeGracefully(feature: string, fallback: any) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const enabled = await featureFlags.isEnabled(feature);
    
    if (!enabled) {
      console.log(`Feature ${feature} is disabled, using fallback`);
      return res.json(fallback);
    }
    
    next();
  };
}

// 示例：推荐系统降级
app.get('/api/recommendations',
  degradeGracefully('recommendations', {
    items: [],
    message: 'Recommendations temporarily unavailable'
  }),
  async (req, res) => {
    // 复杂的推荐算法（高负载）
    const recommendations = await getPersonalizedRecommendations(req.user.id);
    res.json({ items: recommendations });
  }
);

// 示例：搜索功能降级
app.get('/api/search', async (req, res) => {
  const query = req.query.q as string;
  
  try {
    // 尝试使用 Elasticsearch（可能不可用）
    if (await featureFlags.isEnabled('elasticsearch-search')) {
      const results = await elasticsearchClient.search({
        index: 'products',
        body: { query: { match: { name: query } } }
      });
      return res.json({ results, source: 'elasticsearch' });
    }
  } catch (error) {
    console.error('Elasticsearch failed, falling back to database');
  }

  // 降级到数据库查询（功能简化）
  const results = await prisma.product.findMany({
    where: { 
      name: { contains: query, mode: 'insensitive' } 
    },
    take: 20
  });
  
  res.json({ results, source: 'database' });
});

// 监控端点：手动切换功能开关
app.post('/admin/features/:name/:action', async (req, res) => {
  const { name, action } = req.params;
  
  if (action === 'enable') {
    await featureFlags.enable(name);
  } else if (action === 'disable') {
    await featureFlags.disable(name);
  }
  
  res.json({ feature: name, action });
});
```

### 限流（Rate Limiting）

防止系统过载。

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

// 基于 IP 的限流
const apiLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:api:'
  }),
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 最多 100 个请求
  message: 'Too many requests, please try again later',
  standardHeaders: true, // 返回 RateLimit-* headers
  legacyHeaders: false
});

// 应用到所有 API
app.use('/api/', apiLimiter);

// 更严格的限流（登录接口）
const loginLimiter = rateLimit({
  store: new RedisStore({
    client: redis,
    prefix: 'rl:login:'
  }),
  windowMs: 60 * 60 * 1000, // 1 小时
  max: 5, // 最多 5 次尝试
  skipSuccessfulRequests: true // 成功的请求不计数
});

app.post('/api/login', loginLimiter, async (req, res) => {
  // 登录逻辑
});

// 滑动窗口限流（更精确）
class SlidingWindowLimiter {
  constructor(
    private redis: Redis,
    private maxRequests: number,
    private windowMs: number
  ) {}

  async isAllowed(key: string): Promise<boolean> {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    
    const multi = this.redis.multi();
    
    // 删除窗口之外的记录
    multi.zremrangebyscore(key, 0, windowStart);
    
    // 添加当前请求
    multi.zadd(key, now, `${now}`);
    
    // 计数
    multi.zcard(key);
    
    // 设置过期时间
    multi.expire(key, Math.ceil(this.windowMs / 1000));
    
    const results = await multi.exec();
    const count = results[2][1] as number;
    
    return count <= this.maxRequests;
  }
}

const limiter = new SlidingWindowLimiter(redis, 100, 60000); // 100 req/min

app.use(async (req, res, next) => {
  const key = `rate:${req.ip}`;
  const allowed = await limiter.isAllowed(key);
  
  if (!allowed) {
    return res.status(429).json({ 
      error: 'Too many requests' 
    });
  }
  
  next();
});
```

---

## 负载均衡

### 负载均衡算法

```typescript
// 1. 轮询（Round Robin）
class RoundRobinBalancer {
  private current = 0;

  constructor(private servers: string[]) {}

  getServer(): string {
    const server = this.servers[this.current];
    this.current = (this.current + 1) % this.servers.length;
    return server;
  }
}

// 2. 加权轮询（Weighted Round Robin）
class WeightedRoundRobinBalancer {
  private current = 0;
  private currentWeight = 0;

  constructor(
    private servers: Array<{ host: string; weight: number }>
  ) {}

  getServer(): string {
    let totalWeight = 0;
    let maxWeight = 0;
    let selectedServer: string | null = null;

    for (const server of this.servers) {
      server.weight += server.weight; // 当前权重增加
      totalWeight += server.weight;

      if (server.weight > maxWeight) {
        maxWeight = server.weight;
        selectedServer = server.host;
      }
    }

    // 选中的服务器权重减少
    for (const server of this.servers) {
      if (server.host === selectedServer) {
        server.weight -= totalWeight;
        break;
      }
    }

    return selectedServer!;
  }
}

// 3. 最少连接（Least Connections）
class LeastConnectionsBalancer {
  private connections = new Map<string, number>();

  constructor(private servers: string[]) {
    servers.forEach(s => this.connections.set(s, 0));
  }

  getServer(): string {
    let minConn = Infinity;
    let selectedServer = this.servers[0];

    for (const server of this.servers) {
      const conn = this.connections.get(server) || 0;
      if (conn < minConn) {
        minConn = conn;
        selectedServer = server;
      }
    }

    this.connections.set(selectedServer, minConn + 1);
    return selectedServer;
  }

  releaseConnection(server: string) {
    const conn = this.connections.get(server) || 0;
    this.connections.set(server, Math.max(0, conn - 1));
  }
}

// 4. IP Hash（会话粘性）
class IPHashBalancer {
  constructor(private servers: string[]) {}

  getServer(clientIP: string): string {
    let hash = 0;
    for (let i = 0; i < clientIP.length; i++) {
      hash = ((hash << 5) - hash) + clientIP.charCodeAt(i);
      hash = hash & hash; // Convert to 32bit integer
    }
    const index = Math.abs(hash) % this.servers.length;
    return this.servers[index];
  }
}

// 使用示例
const servers = [
  'http://server1:3000',
  'http://server2:3000',
  'http://server3:3000'
];

const balancer = new RoundRobinBalancer(servers);

app.use(async (req, res, next) => {
  const targetServer = balancer.getServer();
  
  try {
    const response = await fetch(`${targetServer}${req.url}`, {
      method: req.method,
      headers: req.headers,
      body: req.method !== 'GET' ? JSON.stringify(req.body) : undefined
    });
    
    const data = await response.json();
    res.json(data);
  } catch (error) {
    console.error(`Server ${targetServer} failed:`, error);
    next(error);
  }
});
```

### Nginx 负载均衡配置

```nginx
upstream backend {
    # 轮询（默认）
    server server1:3000;
    server server2:3000;
    server server3:3000;
    
    # 加权轮询
    # server server1:3000 weight=3;
    # server server2:3000 weight=2;
    # server server3:3000 weight=1;
    
    # 最少连接
    # least_conn;
    
    # IP Hash
    # ip_hash;
    
    # 健康检查
    server server1:3000 max_fails=3 fail_timeout=30s;
    server server2:3000 max_fails=3 fail_timeout=30s;
    server server3:3000 backup; # 备用服务器
    
    # 保持连接
    keepalive 32;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
        
        # 超时设置
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

---

## Node.js 实战

### 完整的高可用 API 服务

```typescript
import express from 'express';
import helmet from 'helmet';
import compression from 'compression';
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';
import pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info'
});

const app = express();
const prisma = new PrismaClient();
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  retryStrategy: (times) => Math.min(times * 50, 2000),
  maxRetriesPerRequest: 3
});

// 中间件
app.use(helmet()); // 安全头
app.use(compression()); // 压缩
app.use(express.json({ limit: '1mb' }));

// 请求 ID
app.use((req, res, next) => {
  req.id = crypto.randomUUID();
  res.setHeader('X-Request-ID', req.id);
  next();
});

// 请求日志
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info({
      requestId: req.id,
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      userAgent: req.get('user-agent')
    });
  });
  
  next();
});

// 超时中间件
app.use((req, res, next) => {
  req.setTimeout(30000); // 30 秒
  res.setTimeout(30000);
  next();
});

// 健康检查（前面已实现）
app.get('/health', healthCheckHandler);
app.get('/health/live', livenessHandler);
app.get('/health/ready', readinessHandler);

// 优雅关闭
let isShuttingDown = false;

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);

async function gracefulShutdown() {
  if (isShuttingDown) return;
  isShuttingDown = true;
  
  logger.info('Received shutdown signal, starting graceful shutdown...');
  
  // 1. 停止接受新请求
  server.close(() => {
    logger.info('HTTP server closed');
  });
  
  // 2. 等待现有请求完成（最多 30 秒）
  await new Promise(resolve => setTimeout(resolve, 30000));
  
  // 3. 关闭数据库连接
  await prisma.$disconnect();
  logger.info('Database disconnected');
  
  // 4. 关闭 Redis 连接
  await redis.quit();
  logger.info('Redis disconnected');
  
  logger.info('Graceful shutdown completed');
  process.exit(0);
}

// 全局错误处理
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error({
    requestId: req.id,
    error: err.message,
    stack: err.stack
  });
  
  if (res.headersSent) {
    return next(err);
  }
  
  res.status(500).json({
    error: 'Internal Server Error',
    requestId: req.id
  });
});

const PORT = process.env.PORT || 3000;
const server = app.listen(PORT, () => {
  logger.info(`Server started on port ${PORT}`);
});
```

---

## 常见面试题

### 1. 如何设计一个高可用系统？

**回答要点**：

1. **消除单点故障**
   - 服务多实例部署
   - 数据库主从/集群
   - 负载均衡

2. **故障检测与恢复**
   - 健康检查（Liveness/Readiness）
   - 自动故障转移（Failover）
   - 快速重启机制

3. **容错设计**
   - 超时和重试
   - 熔断器
   - 降级策略

4. **数据冗余**
   - 数据库复制
   - 定期备份
   - 异地容灾

5. **监控和告警**
   - 实时监控关键指标
   - 及时告警通知
   - 日志聚合和分析

**示例架构**：

```
┌─────────────┐
│   用户      │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  CDN/WAF    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Load Balancer│ (Nginx/HAProxy)
└──────┬──────┘
       │
       ├────────┬────────┬────────┐
       ▼        ▼        ▼        ▼
    [API-1]  [API-2]  [API-3]  [API-4]
       │        │        │        │
       └────────┴────────┴────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
   [Redis]   [主库]    [从库1]
   Sentinel          [从库2]
```

### 2. 垂直扩展 vs 水平扩展的选择？

| 场景 | 推荐 | 理由 |
|------|------|------|
| 数据库（OLTP） | 垂直 + 读写分离 | 保证一致性 |
| Web/API 服务器 | 水平 | 无状态，易扩展 |
| 缓存服务器 | 垂直 | 减少网络开销 |
| 消息队列 | 水平（分区） | 提高吞吐量 |
| 静态文件服务 | 水平 + CDN | 就近访问 |

**混合方案**（最常见）：
- 应用层：水平扩展（多实例）
- 数据库层：垂直扩展 + 读写分离
- 缓存层：垂直扩展 + 主从复制

### 3. 如何处理数据库的主从延迟？

**问题**：写入主库后，从库可能需要几百毫秒才能同步。

**解决方案**：

1. **写后读主库**（最简单）
```typescript
async function updateAndGet(userId: number, data: any) {
  // 写入主库
  await primary.user.update({ where: { id: userId }, data });
  
  // 强制从主库读取
  return await primary.user.findUnique({ where: { id: userId } });
}
```

2. **延迟读取**
```typescript
async function updateUser(userId: number, data: any) {
  await primary.user.update({ where: { id: userId }, data });
  
  // 等待主从同步（通常 < 100ms）
  await new Promise(resolve => setTimeout(resolve, 100));
  
  // 从从库读取
  return await replica.user.findUnique({ where: { id: userId } });
}
```

3. **版本号/时间戳**
```typescript
// 写入时记录版本号
await primary.user.update({
  where: { id: userId },
  data: { ...data, version: { increment: 1 } }
});

// 读取时检查版本号
const user = await replica.user.findUnique({ where: { id: userId } });
if (user.version < expectedVersion) {
  // 版本过旧，从主库读取
  return await primary.user.findUnique({ where: { id: userId } });
}
```

4. **会话一致性**（推荐）
```typescript
// 同一会话内，短时间内强制读主库
const sessionCache = new Map<string, number>();

app.use((req, res, next) => {
  const sessionId = req.headers.sessionid;
  const lastWrite = sessionCache.get(sessionId);
  
  // 写入后 5 秒内，强制读主库
  if (lastWrite && Date.now() - lastWrite < 5000) {
    req.forceReadPrimary = true;
  }
  
  next();
});

app.post('/users/:id', async (req, res) => {
  await primary.user.update(/* ... */);
  sessionCache.set(req.headers.sessionid, Date.now());
  res.json({ success: true });
});

app.get('/users/:id', async (req, res) => {
  const db = req.forceReadPrimary ? primary : replica;
  const user = await db.user.findUnique(/* ... */);
  res.json(user);
});
```

### 4. 如何实现无状态服务？

**关键原则**：服务器不保存任何会话状态。

**存储位置**：
1. **客户端**：JWT（适用于认证信息）
2. **外部存储**：Redis/数据库（适用于复杂会话）
3. **数据库**：用户数据

**反模式**（避免）：
```typescript
// ❌ 状态存储在内存
const sessions = new Map();
const uploadProgress = new Map();
const websocketConnections = new Map();
```

**正确做法**：
```typescript
// ✅ 使用 Redis
await redis.set(`session:${sessionId}`, JSON.stringify(data));
await redis.set(`upload:${uploadId}`, progress);
await redis.sadd(`ws:connections`, connectionId);
```

### 5. 如何选择负载均衡算法？

| 算法 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **轮询** | 服务器性能相同 | 简单、公平 | 不考虑负载 |
| **加权轮询** | 服务器性能不同 | 按能力分配 | 需要配置权重 |
| **最少连接** | 长连接、WebSocket | 动态平衡负载 | 实现复杂 |
| **IP Hash** | 需要会话粘性 | 同一用户到同一服务器 | 分布可能不均 |
| **一致性哈希** | 缓存服务器 | 节点变化影响小 | 实现复杂 |

**选择建议**：
- 无状态 API：轮询或加权轮询
- WebSocket/长连接：最少连接
- 有状态应用（不推荐）：IP Hash
- 缓存集群：一致性哈希

---

## 总结

### 可扩展性关键点

1. ✅ **无状态设计**：状态外部化（Redis/JWT）
2. ✅ **水平扩展**：应用层多实例
3. ✅ **读写分离**：数据库层优化
4. ✅ **数据分片**：大数据量时必须
5. ✅ **异步处理**：消息队列削峰

### 高可用关键点

1. ✅ **消除单点**：多实例 + 负载均衡
2. ✅ **健康检查**：及时发现故障
3. ✅ **自动恢复**：故障转移、自动重启
4. ✅ **容错设计**：超时、重试、熔断、降级
5. ✅ **监控告警**：实时监控 + 及时响应

### 实践检查清单

- [ ] 服务是否无状态？
- [ ] 是否有单点故障？
- [ ] 是否实现健康检查？
- [ ] 是否有超时和重试机制？
- [ ] 是否实现熔断器？
- [ ] 是否有降级方案？
- [ ] 是否实现限流？
- [ ] 数据库是否有主从？
- [ ] 是否有负载均衡？
- [ ] 是否能优雅关闭？
- [ ] 是否有监控和告警？
- [ ] 日志是否聚合？

---

**下一篇**：[缓存策略](./02-caching-strategies.md)

