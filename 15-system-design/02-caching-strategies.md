# 缓存策略

缓存是提升系统性能的最有效手段之一。合理的缓存策略可以显著降低数据库负载、提升响应速度、提高系统吞吐量。

## 目录
- [缓存基础](#缓存基础)
- [缓存模式](#缓存模式)
- [缓存更新策略](#缓存更新策略)
- [缓存三大问题](#缓存三大问题)
- [多级缓存](#多级缓存)
- [Node.js 实战](#nodejs-实战)
- [面试题](#常见面试题)

---

## 缓存基础

### 什么时候需要缓存？

**适合缓存的场景**：
- ✅ 读多写少（读写比 > 3:1）
- ✅ 计算成本高（复杂查询、聚合）
- ✅ 数据一致性要求不高（允许短暂延迟）
- ✅ 热点数据（20% 数据占 80% 访问）

**不适合缓存的场景**：
- ❌ 写多读少
- ❌ 强一致性要求（金融交易）
- ❌ 数据访问分散（没有热点）
- ❌ 数据量太大（缓存命中率低）

### 缓存指标

```typescript
interface CacheMetrics {
  hits: number;        // 命中次数
  misses: number;      // 未命中次数
  hitRate: number;     // 命中率 = hits / (hits + misses)
  avgLatency: number;  // 平均延迟
  size: number;        // 缓存大小
  evictions: number;   // 驱逐次数
}

class CacheMonitor {
  private metrics: CacheMetrics = {
    hits: 0,
    misses: 0,
    hitRate: 0,
    avgLatency: 0,
    size: 0,
    evictions: 0
  };
  
  private latencies: number[] = [];

  recordHit(latency: number) {
    this.metrics.hits++;
    this.latencies.push(latency);
    this.updateMetrics();
  }

  recordMiss(latency: number) {
    this.metrics.misses++;
    this.latencies.push(latency);
    this.updateMetrics();
  }

  recordEviction() {
    this.metrics.evictions++;
  }

  private updateMetrics() {
    const total = this.metrics.hits + this.metrics.misses;
    this.metrics.hitRate = total > 0 ? this.metrics.hits / total : 0;
    
    if (this.latencies.length > 0) {
      this.metrics.avgLatency = 
        this.latencies.reduce((a, b) => a + b, 0) / this.latencies.length;
    }
    
    // 保留最近 1000 个延迟记录
    if (this.latencies.length > 1000) {
      this.latencies = this.latencies.slice(-1000);
    }
  }

  getMetrics(): CacheMetrics {
    return { ...this.metrics };
  }

  reset() {
    this.metrics = {
      hits: 0,
      misses: 0,
      hitRate: 0,
      avgLatency: 0,
      size: 0,
      evictions: 0
    };
    this.latencies = [];
  }
}
```

---

## 缓存模式

### 1. Cache-Aside（旁路缓存）

**最常用的模式**，应用负责读写缓存和数据库。

```typescript
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';

const prisma = new PrismaClient();
const redis = new Redis();

class UserRepository {
  private readonly CACHE_TTL = 3600; // 1 小时
  private readonly CACHE_PREFIX = 'user:';

  // 读取：先查缓存，未命中查数据库
  async getUser(id: number): Promise<User | null> {
    const cacheKey = `${this.CACHE_PREFIX}${id}`;
    
    // 1. 查缓存
    const cached = await redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // 2. 缓存未命中，查数据库
    const user = await prisma.user.findUnique({ where: { id } });
    
    if (user) {
      // 3. 写入缓存
      await redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(user));
    }
    
    return user;
  }

  // 更新：先更新数据库，再删除缓存
  async updateUser(id: number, data: Partial<User>): Promise<User> {
    // 1. 更新数据库
    const user = await prisma.user.update({
      where: { id },
      data
    });
    
    // 2. 删除缓存（让下次读取时重新加载）
    const cacheKey = `${this.CACHE_PREFIX}${id}`;
    await redis.del(cacheKey);
    
    return user;
  }

  // 删除：先删除数据库，再删除缓存
  async deleteUser(id: number): Promise<void> {
    // 1. 删除数据库
    await prisma.user.delete({ where: { id } });
    
    // 2. 删除缓存
    const cacheKey = `${this.CACHE_PREFIX}${id}`;
    await redis.del(cacheKey);
  }
}
```

**优点**：
- ✅ 简单易懂
- ✅ 缓存失败不影响系统
- ✅ 适用于大多数场景

**缺点**：
- ❌ 首次请求较慢（缓存未命中）
- ❌ 缓存和数据库可能短暂不一致

### 2. Read-Through（直读缓存）

缓存层负责加载数据，应用只与缓存交互。

```typescript
// 抽象缓存层
abstract class ReadThroughCache<K, V> {
  constructor(
    protected redis: Redis,
    protected ttl: number,
    protected keyPrefix: string
  ) {}

  async get(key: K): Promise<V | null> {
    const cacheKey = this.getCacheKey(key);
    
    // 1. 查缓存
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // 2. 缓存未命中，调用子类的加载方法
    const value = await this.loadFromSource(key);
    
    if (value) {
      // 3. 写入缓存
      await this.redis.setex(cacheKey, this.ttl, JSON.stringify(value));
    }
    
    return value;
  }

  protected abstract loadFromSource(key: K): Promise<V | null>;
  
  protected getCacheKey(key: K): string {
    return `${this.keyPrefix}${String(key)}`;
  }
}

// 具体实现
class UserCache extends ReadThroughCache<number, User> {
  constructor(redis: Redis, private prisma: PrismaClient) {
    super(redis, 3600, 'user:');
  }

  protected async loadFromSource(id: number): Promise<User | null> {
    return await this.prisma.user.findUnique({ where: { id } });
  }
}

// 使用
const userCache = new UserCache(redis, prisma);
const user = await userCache.get(123); // 自动处理缓存和数据库
```

**优点**：
- ✅ 应用逻辑简单（不用关心加载）
- ✅ 一致的数据访问接口

**缺点**：
- ❌ 缓存层复杂
- ❌ 缓存失败可能影响系统

### 3. Write-Through（直写缓存）

写入时同时更新缓存和数据库。

```typescript
class WriteThroughCache<K, V> {
  constructor(
    private redis: Redis,
    private ttl: number,
    private keyPrefix: string
  ) {}

  async set(key: K, value: V): Promise<void> {
    const cacheKey = this.getCacheKey(key);
    
    // 1. 写入数据库（子类实现）
    await this.writeToSource(key, value);
    
    // 2. 同步写入缓存
    await this.redis.setex(cacheKey, this.ttl, JSON.stringify(value));
  }

  protected getCacheKey(key: K): string {
    return `${this.keyPrefix}${String(key)}`;
  }

  protected async writeToSource(key: K, value: V): Promise<void> {
    // 由子类实现
    throw new Error('Must implement writeToSource');
  }
}

class UserWriteThroughCache extends WriteThroughCache<number, Partial<User>> {
  constructor(redis: Redis, private prisma: PrismaClient) {
    super(redis, 3600, 'user:');
  }

  protected async writeToSource(id: number, data: Partial<User>): Promise<void> {
    await this.prisma.user.update({ where: { id }, data });
  }
}
```

**优点**：
- ✅ 缓存和数据库强一致
- ✅ 读取性能好（总是命中）

**缺点**：
- ❌ 写入延迟高（同步写两个地方）
- ❌ 写入的数据可能不常读（浪费）

### 4. Write-Behind（回写缓存）

先写缓存，异步批量写数据库。

```typescript
class WriteBehindCache {
  private writeQueue = new Map<string, any>();
  private flushInterval: NodeJS.Timeout;

  constructor(
    private redis: Redis,
    private prisma: PrismaClient,
    private flushIntervalMs = 5000 // 5 秒批量刷新
  ) {
    this.startFlushTimer();
  }

  async set(key: string, value: any): Promise<void> {
    // 1. 立即写入缓存
    await this.redis.set(key, JSON.stringify(value));
    
    // 2. 加入写队列
    this.writeQueue.set(key, value);
  }

  private startFlushTimer() {
    this.flushInterval = setInterval(async () => {
      await this.flush();
    }, this.flushIntervalMs);
  }

  private async flush() {
    if (this.writeQueue.size === 0) return;

    console.log(`Flushing ${this.writeQueue.size} items to database...`);

    // 批量写入数据库
    const entries = Array.from(this.writeQueue.entries());
    this.writeQueue.clear();

    try {
      await prisma.$transaction(
        entries.map(([key, value]) => {
          const userId = parseInt(key.replace('user:', ''));
          return prisma.user.update({
            where: { id: userId },
            data: value
          });
        })
      );
      console.log('Flush completed');
    } catch (error) {
      console.error('Flush failed:', error);
      // 重新加入队列
      entries.forEach(([key, value]) => {
        this.writeQueue.set(key, value);
      });
    }
  }

  async shutdown() {
    clearInterval(this.flushInterval);
    await this.flush(); // 最后一次刷新
  }
}

// 使用
const writeCache = new WriteBehindCache(redis, prisma);

// 快速写入（只写缓存）
await writeCache.set('user:123', { name: 'John', email: 'john@example.com' });

// 优雅关闭时刷新所有数据
process.on('SIGTERM', async () => {
  await writeCache.shutdown();
  process.exit(0);
});
```

**优点**：
- ✅ 写入性能极高（只写缓存）
- ✅ 批量操作（减少数据库负载）

**缺点**：
- ❌ 数据可能丢失（缓存宕机）
- ❌ 实现复杂
- ❌ 缓存和数据库可能不一致

### 模式对比

| 模式 | 读性能 | 写性能 | 一致性 | 复杂度 | 适用场景 |
|------|-------|-------|--------|--------|---------|
| Cache-Aside | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ | **通用（推荐）** |
| Read-Through | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | 读多写少 |
| Write-Through | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | 强一致性 |
| Write-Behind | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | 高吞吐写入 |

---

## 缓存更新策略

### 1. 更新数据库 + 删除缓存（推荐）

```typescript
async function updateUser(id: number, data: Partial<User>) {
  // 1. 更新数据库
  const user = await prisma.user.update({ where: { id }, data });
  
  // 2. 删除缓存
  await redis.del(`user:${id}`);
  
  return user;
}
```

**优点**：简单、安全
**缺点**：下次读取会缓存未命中

### 2. 更新数据库 + 更新缓存

```typescript
async function updateUser(id: number, data: Partial<User>) {
  // 1. 更新数据库
  const user = await prisma.user.update({ where: { id }, data });
  
  // 2. 更新缓存
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  
  return user;
}
```

**优点**：下次读取直接命中
**缺点**：可能导致不一致（更新失败、并发问题）

### 3. 延迟双删（解决并发问题）

```typescript
async function updateUser(id: number, data: Partial<User>) {
  const cacheKey = `user:${id}`;
  
  // 1. 删除缓存
  await redis.del(cacheKey);
  
  // 2. 更新数据库
  const user = await prisma.user.update({ where: { id }, data });
  
  // 3. 延迟 500ms 再删除一次
  setTimeout(async () => {
    await redis.del(cacheKey);
  }, 500);
  
  return user;
}
```

**为什么需要延迟双删？**

防止以下并发场景：
1. 线程 A 删除缓存
2. 线程 B 读取（缓存未命中）→ 读数据库（旧数据）→ 写缓存
3. 线程 A 更新数据库
4. **问题**：缓存是旧数据

**解决**：第二次删除清理掉线程 B 写入的旧缓存。

### 4. 使用版本号

```typescript
interface CachedUser extends User {
  _cacheVersion: number;
}

async function updateUser(id: number, data: Partial<User>) {
  const cacheKey = `user:${id}`;
  
  // 1. 读取当前版本
  const cached = await redis.get(cacheKey);
  const currentVersion = cached ? JSON.parse(cached)._cacheVersion || 0 : 0;
  
  // 2. 更新数据库
  const user = await prisma.user.update({ where: { id }, data });
  
  // 3. 更新缓存（新版本）
  const cachedUser: CachedUser = {
    ...user,
    _cacheVersion: currentVersion + 1
  };
  await redis.setex(cacheKey, 3600, JSON.stringify(cachedUser));
  
  return user;
}

async function getUser(id: number): Promise<User | null> {
  const cacheKey = `user:${id}`;
  
  // 读取缓存
  const cached = await redis.get(cacheKey);
  if (cached) {
    const data = JSON.parse(cached);
    delete data._cacheVersion; // 移除版本号
    return data;
  }
  
  // 从数据库加载
  const user = await prisma.user.findUnique({ where: { id } });
  if (user) {
    const cachedUser: CachedUser = { ...user, _cacheVersion: 1 };
    await redis.setex(cacheKey, 3600, JSON.stringify(cachedUser));
  }
  
  return user;
}
```

---

## 缓存三大问题

### 1. 缓存穿透

**问题**：查询不存在的数据，缓存和数据库都没有，导致每次都查数据库。

```typescript
// 攻击示例
for (let i = -1000000; i < 0; i--) {
  await fetch(`/api/users/${i}`); // 这些用户 ID 不存在
}
// 每次请求都打到数据库，数据库崩溃
```

**解决方案 1：缓存空值**

```typescript
async function getUser(id: number): Promise<User | null> {
  const cacheKey = `user:${id}`;
  
  // 1. 查缓存
  const cached = await redis.get(cacheKey);
  if (cached) {
    if (cached === 'NULL') return null; // 空值
    return JSON.parse(cached);
  }
  
  // 2. 查数据库
  const user = await prisma.user.findUnique({ where: { id } });
  
  if (user) {
    // 存在：缓存 1 小时
    await redis.setex(cacheKey, 3600, JSON.stringify(user));
  } else {
    // 不存在：缓存空值，较短的 TTL（5 分钟）
    await redis.setex(cacheKey, 300, 'NULL');
  }
  
  return user;
}
```

**解决方案 2：布隆过滤器（Bloom Filter）**

```typescript
import { BloomFilter } from 'bloomfilter';

class UserBloomFilter {
  private filter: BloomFilter;
  
  constructor(private redis: Redis, private prisma: PrismaClient) {
    // 初始化布隆过滤器（1000万容量，0.01 误判率）
    this.filter = new BloomFilter(
      10000000, // 位数组大小
      10        // 哈希函数数量
    );
    this.initFilter();
  }

  private async initFilter() {
    // 启动时加载所有用户 ID
    const users = await this.prisma.user.findMany({ select: { id: true } });
    users.forEach(user => this.filter.add(String(user.id)));
    console.log(`Bloom filter initialized with ${users.length} users`);
  }

  async getUser(id: number): Promise<User | null> {
    // 1. 布隆过滤器检查
    if (!this.filter.test(String(id))) {
      // 肯定不存在，直接返回
      return null;
    }
    
    // 2. 可能存在，查缓存
    const cached = await this.redis.get(`user:${id}`);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // 3. 查数据库
    const user = await this.prisma.user.findUnique({ where: { id } });
    
    if (user) {
      await this.redis.setex(`user:${id}`, 3600, JSON.stringify(user));
    }
    
    return user;
  }

  async createUser(data: any): Promise<User> {
    const user = await this.prisma.user.create({ data });
    
    // 添加到布隆过滤器
    this.filter.add(String(user.id));
    
    return user;
  }
}
```

**优点**：
- 空间效率高（10M 条记录只需几 MB）
- 查询速度快（O(1)）

**缺点**：
- 有误判（可能说存在实际不存在，但不会说不存在实际存在）
- 不支持删除

### 2. 缓存击穿

**问题**：热点 Key 突然过期，大量请求同时查数据库。

```typescript
// 场景：热门商品缓存过期
// 瞬间 10000 个请求同时查数据库
```

**解决方案 1：互斥锁**

```typescript
class CacheWithLock {
  private locks = new Map<string, Promise<any>>();

  async get(key: string, loader: () => Promise<any>, ttl: number) {
    // 1. 查缓存
    let cached = await redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    // 2. 尝试获取锁
    const lockKey = `lock:${key}`;
    const locked = await redis.set(lockKey, '1', 'NX', 'EX', 10); // 10 秒锁

    if (locked) {
      // 获得锁，加载数据
      try {
        const data = await loader();
        await redis.setex(key, ttl, JSON.stringify(data));
        return data;
      } finally {
        // 释放锁
        await redis.del(lockKey);
      }
    } else {
      // 未获得锁，等待一段时间后重试
      await new Promise(resolve => setTimeout(resolve, 50));
      
      // 重新查缓存
      cached = await redis.get(key);
      if (cached) {
        return JSON.parse(cached);
      }
      
      // 还是没有，递归重试
      return this.get(key, loader, ttl);
    }
  }
}

// 使用
const cache = new CacheWithLock();

async function getProduct(id: number) {
  return await cache.get(
    `product:${id}`,
    () => prisma.product.findUnique({ where: { id } }),
    3600
  );
}
```

**解决方案 2：永不过期**

```typescript
interface CachedData<T> {
  data: T;
  expireAt: number;
}

async function getWithLogicalExpire<T>(
  key: string,
  loader: () => Promise<T>,
  ttl: number
): Promise<T> {
  // 1. 查缓存（物理上不过期）
  const cached = await redis.get(key);
  
  if (cached) {
    const cachedData: CachedData<T> = JSON.parse(cached);
    
    // 逻辑上未过期
    if (Date.now() < cachedData.expireAt) {
      return cachedData.data;
    }
    
    // 逻辑上已过期，异步更新
    setImmediate(async () => {
      const lockKey = `lock:${key}`;
      const locked = await redis.set(lockKey, '1', 'NX', 'EX', 10);
      
      if (locked) {
        try {
          const data = await loader();
          const newCached: CachedData<T> = {
            data,
            expireAt: Date.now() + ttl * 1000
          };
          await redis.set(key, JSON.stringify(newCached));
        } finally {
          await redis.del(lockKey);
        }
      }
    });
    
    // 先返回旧数据
    return cachedData.data;
  }
  
  // 2. 缓存不存在，加载数据
  const data = await loader();
  const cachedData: CachedData<T> = {
    data,
    expireAt: Date.now() + ttl * 1000
  };
  await redis.set(key, JSON.stringify(cachedData));
  
  return data;
}
```

### 3. 缓存雪崩

**问题**：大量缓存同时过期，数据库瞬间压力巨大。

**解决方案 1：随机 TTL**

```typescript
function getRandomTTL(baseTTL: number): number {
  // 基础 TTL ± 20% 随机偏移
  const offset = baseTTL * 0.2;
  return baseTTL + Math.random() * offset * 2 - offset;
}

async function cacheUser(id: number, user: User) {
  const ttl = getRandomTTL(3600); // 3600 ± 720 秒
  await redis.setex(`user:${id}`, Math.floor(ttl), JSON.stringify(user));
}
```

**解决方案 2：缓存预热**

```typescript
async function warmUpCache() {
  console.log('Starting cache warm-up...');
  
  // 预热热点数据
  const hotProducts = await prisma.product.findMany({
    where: { isHot: true },
    take: 100
  });
  
  for (const product of hotProducts) {
    await redis.setex(
      `product:${product.id}`,
      getRandomTTL(3600),
      JSON.stringify(product)
    );
  }
  
  console.log(`Warmed up ${hotProducts.length} products`);
}

// 应用启动时预热
warmUpCache();

// 定时预热
setInterval(warmUpCache, 30 * 60 * 1000); // 每 30 分钟
```

**解决方案 3：多级缓存**

```typescript
// 本地缓存 + Redis
import NodeCache from 'node-cache';

const localCache = new NodeCache({ stdTTL: 60 }); // 本地缓存 1 分钟

async function getUser(id: number): Promise<User | null> {
  // 1. L1: 本地缓存
  let user = localCache.get<User>(`user:${id}`);
  if (user) {
    return user;
  }
  
  // 2. L2: Redis
  const cached = await redis.get(`user:${id}`);
  if (cached) {
    user = JSON.parse(cached);
    localCache.set(`user:${id}`, user);
    return user;
  }
  
  // 3. L3: 数据库
  user = await prisma.user.findUnique({ where: { id } });
  if (user) {
    localCache.set(`user:${id}`, user);
    await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  }
  
  return user;
}
```

---

## 多级缓存

### 缓存层级

```
用户请求
   ↓
Browser Cache (浏览器缓存)
   ↓
CDN Cache (边缘缓存)
   ↓
API Gateway Cache (网关缓存)
   ↓
Local Cache (应用本地缓存)
   ↓
Redis Cache (分布式缓存)
   ↓
Database (数据库)
```

### 完整的多级缓存实现

```typescript
import NodeCache from 'node-cache';
import Redis from 'ioredis';
import { PrismaClient } from '@prisma/client';

interface CacheOptions {
  localTTL?: number;
  redisTTL?: number;
}

class MultiLevelCache {
  private localCache: NodeCache;
  private redis: Redis;
  private prisma: PrismaClient;
  private monitor: CacheMonitor;

  constructor() {
    this.localCache = new NodeCache({
      stdTTL: 60,           // 1 分钟
      checkperiod: 120,     // 2 分钟检查一次过期
      useClones: false      // 不克隆对象（性能）
    });
    
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: Number(process.env.REDIS_PORT),
      retryStrategy: (times) => Math.min(times * 50, 2000)
    });
    
    this.prisma = new PrismaClient();
    this.monitor = new CacheMonitor();
  }

  async get<T>(
    key: string,
    loader: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const { localTTL = 60, redisTTL = 3600 } = options;
    const start = Date.now();

    // L1: 本地缓存
    let value = this.localCache.get<T>(key);
    if (value !== undefined) {
      this.monitor.recordHit(Date.now() - start);
      console.log(`[L1 HIT] ${key}`);
      return value;
    }

    // L2: Redis
    try {
      const cached = await this.redis.get(key);
      if (cached) {
        value = JSON.parse(cached) as T;
        
        // 写回 L1
        this.localCache.set(key, value, localTTL);
        
        this.monitor.recordHit(Date.now() - start);
        console.log(`[L2 HIT] ${key}`);
        return value;
      }
    } catch (error) {
      console.error('[Redis Error]', error);
      // Redis 故障，降级到数据库
    }

    // L3: 数据库
    this.monitor.recordMiss(Date.now() - start);
    console.log(`[MISS] ${key}, loading from source...`);
    
    value = await loader();

    // 写回缓存
    this.localCache.set(key, value, localTTL);
    
    try {
      await this.redis.setex(key, redisTTL, JSON.stringify(value));
    } catch (error) {
      console.error('[Redis Set Error]', error);
    }

    return value;
  }

  async invalidate(key: string) {
    // 删除所有层级的缓存
    this.localCache.del(key);
    
    try {
      await this.redis.del(key);
    } catch (error) {
      console.error('[Redis Delete Error]', error);
    }
  }

  async invalidatePattern(pattern: string) {
    // 删除匹配的缓存
    const keys = this.localCache.keys().filter(k => k.startsWith(pattern));
    keys.forEach(k => this.localCache.del(k));
    
    try {
      const redisKeys = await this.redis.keys(pattern);
      if (redisKeys.length > 0) {
        await this.redis.del(...redisKeys);
      }
    } catch (error) {
      console.error('[Redis Pattern Delete Error]', error);
    }
  }

  getMetrics() {
    return {
      cache: this.monitor.getMetrics(),
      local: {
        size: this.localCache.keys().length,
        stats: this.localCache.getStats()
      }
    };
  }
}

// 使用示例
const cache = new MultiLevelCache();

async function getUser(id: number): Promise<User | null> {
  return await cache.get(
    `user:${id}`,
    () => prisma.user.findUnique({ where: { id } }),
    { localTTL: 30, redisTTL: 1800 }
  );
}

async function updateUser(id: number, data: Partial<User>): Promise<User> {
  const user = await prisma.user.update({ where: { id }, data });
  
  // 清理缓存
  await cache.invalidate(`user:${id}`);
  
  return user;
}

// 监控端点
app.get('/metrics/cache', (req, res) => {
  res.json(cache.getMetrics());
});
```

---

## Node.js 实战

### 缓存工具类

```typescript
import Redis from 'ioredis';
import { promisify } from 'util';
import { gzip, gunzip } from 'zlib';

const gzipAsync = promisify(gzip);
const gunzipAsync = promisify(gunzip);

export class CacheManager {
  constructor(
    private redis: Redis,
    private options = {
      compress: true,           // 是否压缩
      compressThreshold: 1024,  // 超过 1KB 才压缩
      serializeJson: true       // 自动序列化 JSON
    }
  ) {}

  async get<T>(key: string): Promise<T | null> {
    const value = await this.redis.getBuffer(key);
    if (!value) return null;

    try {
      // 检查是否压缩
      const isCompressed = value[0] === 0x1f && value[1] === 0x8b;
      
      let data = value;
      if (isCompressed) {
        data = await gunzipAsync(value);
      }

      if (this.options.serializeJson) {
        return JSON.parse(data.toString()) as T;
      }

      return data.toString() as any;
    } catch (error) {
      console.error('Cache get error:', error);
      return null;
    }
  }

  async set(
    key: string,
    value: any,
    ttl?: number
  ): Promise<void> {
    try {
      let data: Buffer;

      // 序列化
      if (this.options.serializeJson) {
        data = Buffer.from(JSON.stringify(value));
      } else {
        data = Buffer.from(String(value));
      }

      // 压缩（如果超过阈值）
      if (
        this.options.compress &&
        data.length > this.options.compressThreshold
      ) {
        data = await gzipAsync(data);
      }

      if (ttl) {
        await this.redis.setex(key, ttl, data);
      } else {
        await this.redis.set(key, data);
      }
    } catch (error) {
      console.error('Cache set error:', error);
      throw error;
    }
  }

  async getOrSet<T>(
    key: string,
    loader: () => Promise<T>,
    ttl: number
  ): Promise<T> {
    // 尝试获取缓存
    const cached = await this.get<T>(key);
    if (cached !== null) {
      return cached;
    }

    // 加载数据
    const value = await loader();

    // 写入缓存（异步，不阻塞返回）
    setImmediate(() => {
      this.set(key, value, ttl).catch(console.error);
    });

    return value;
  }

  async remember<T>(
    key: string,
    ttl: number,
    loader: () => Promise<T>
  ): Promise<T> {
    return this.getOrSet(key, loader, ttl);
  }

  async forget(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async forgetPattern(pattern: string): Promise<number> {
    const keys = await this.redis.keys(pattern);
    if (keys.length === 0) return 0;
    return await this.redis.del(...keys);
  }

  async tags(tags: string[]): CacheTagManager {
    return new CacheTagManager(this.redis, tags);
  }
}

// 标签缓存
class CacheTagManager {
  constructor(
    private redis: Redis,
    private tags: string[]
  ) {}

  async remember<T>(
    key: string,
    ttl: number,
    loader: () => Promise<T>
  ): Promise<T> {
    // 记录 key 属于哪些标签
    for (const tag of this.tags) {
      await this.redis.sadd(`tag:${tag}`, key);
    }

    const cache = new CacheManager(this.redis);
    return cache.remember(key, ttl, loader);
  }

  async flush(): Promise<void> {
    // 清空所有标签关联的缓存
    for (const tag of this.tags) {
      const keys = await this.redis.smembers(`tag:${tag}`);
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
      await this.redis.del(`tag:${tag}`);
    }
  }
}

// 使用示例
const cache = new CacheManager(redis);

// 1. 简单缓存
const user = await cache.remember(
  'user:123',
  3600,
  () => prisma.user.findUnique({ where: { id: 123 } })
);

// 2. 标签缓存
const posts = await cache.tags(['user:123', 'posts']).remember(
  'user:123:posts',
  1800,
  () => prisma.post.findMany({ where: { authorId: 123 } })
);

// 3. 清空某个用户的所有缓存
await cache.tags(['user:123']).flush();
```

---

## 常见面试题

### 1. 缓存击穿、穿透、雪崩的区别和解决方案？

| 问题 | 现象 | 原因 | 解决方案 |
|------|------|------|---------|
| **穿透** | 大量请求查不存在的数据 | 恶意攻击或业务缺陷 | 布隆过滤器、缓存空值 |
| **击穿** | 热点 Key 过期瞬间压垮数据库 | 热点数据突然过期 | 互斥锁、永不过期 |
| **雪崩** | 大量缓存同时失效 | 大量 Key 同时过期 | 随机 TTL、多级缓存、缓存预热 |

### 2. 缓存和数据库一致性如何保证？

**方案对比**：

| 方案 | 一致性 | 性能 | 复杂度 | 推荐 |
|------|-------|------|--------|------|
| 先更新数据库，再删除缓存 | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ✅ **推荐** |
| 先删除缓存，再更新数据库 | ⭐⭐ | ⭐⭐⭐⭐ | ⭐ | ❌ 不推荐 |
| 延迟双删 | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ✅ 高并发 |
| 先更新数据库，再更新缓存 | ⭐⭐ | ⭐⭐⭐ | ⭐ | ❌ 不推荐 |
| 使用消息队列异步更新 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ 大数据量 |

**推荐方案**：先更新数据库，再删除缓存

理由：
- 简单可靠
- 缓存失败不影响数据更新
- 下次读取时自动更新缓存

### 3. 什么时候用 Redis，什么时候用本地缓存？

**本地缓存（NodeCache、LRU Cache）**：
- ✅ 单机应用
- ✅ 不需要共享
- ✅ 数据量小
- ✅ 极低延迟要求（微秒级）

**Redis**：
- ✅ 分布式应用（多实例）
- ✅ 需要共享缓存
- ✅ 数据量大
- ✅ 持久化需求
- ✅ 复杂数据结构（List、Set、ZSet）

**最佳实践**：多级缓存（本地 + Redis）

### 4. 缓存 Key 如何设计？

**规范**：

```typescript
// 格式：业务:对象:ID[:字段]
const keys = {
  user: (id: number) => `user:${id}`,
  userPosts: (userId: number) => `user:${userId}:posts`,
  userProfile: (userId: number) => `user:${userId}:profile`,
  
  post: (id: number) => `post:${id}`,
  postComments: (postId: number) => `post:${postId}:comments`,
  
  // 带版本号
  userV2: (id: number) => `user:v2:${id}`,
  
  // 带环境
  userDev: (id: number) => `dev:user:${id}`,
  userProd: (id: number) => `prod:user:${id}`
};
```

**原则**：
1. 有意义、易读
2. 使用冒号分隔层级
3. 加入版本号（便于迁移）
4. 区分环境
5. 不要太长（节省内存）

### 5. 如何监控缓存性能？

**关键指标**：

```typescript
interface CacheMetrics {
  hitRate: number;        // 命中率（最重要）
  qps: number;            // 每秒查询数
  avgLatency: number;     // 平均延迟
  p99Latency: number;     // P99 延迟
  memoryUsage: number;    // 内存使用
  evictions: number;      // 驱逐次数
  errors: number;         // 错误次数
}

// 监控端点
app.get('/metrics/cache', async (req, res) => {
  const info = await redis.info('stats');
  const keyspaceHits = parseInt(info.match(/keyspace_hits:(\d+)/)?.[1] || '0');
  const keyspaceMisses = parseInt(info.match(/keyspace_misses:(\d+)/)?.[1] || '0');
  
  const metrics: CacheMetrics = {
    hitRate: keyspaceHits / (keyspaceHits + keyspaceMisses),
    qps: await getQPS(),
    avgLatency: await getAvgLatency(),
    p99Latency: await getP99Latency(),
    memoryUsage: await getMemoryUsage(),
    evictions: await getEvictions(),
    errors: await getErrors()
  };
  
  res.json(metrics);
});
```

**告警阈值**：
- 命中率 < 80%：需要优化
- P99 延迟 > 100ms：性能问题
- 驱逐次数增长：内存不足

---

## 总结

### 缓存策略选择

| 场景 | 推荐方案 |
|------|---------|
| **通用场景** | Cache-Aside + 先更新后删除 |
| **读多写少** | Read-Through + 永不过期 |
| **强一致性** | Write-Through |
| **高吞吐写** | Write-Behind |
| **防缓存穿透** | 布隆过滤器 + 缓存空值 |
| **防缓存击穿** | 互斥锁 + 永不过期 |
| **防缓存雪崩** | 随机 TTL + 多级缓存 |
| **分布式应用** | Redis + 本地缓存 |
| **单机应用** | 本地缓存（NodeCache） |

### 实践检查清单

- [ ] 是否选择了合适的缓存模式？
- [ ] 是否设置了合理的 TTL？
- [ ] 是否处理了缓存穿透？
- [ ] 是否处理了缓存击穿？
- [ ] 是否防止了缓存雪崩？
- [ ] 缓存更新策略是否正确？
- [ ] 是否实现了多级缓存？
- [ ] 是否监控缓存性能？
- [ ] 缓存 Key 命名是否规范？
- [ ] 是否处理了缓存失败？
- [ ] 是否有缓存预热？
- [ ] 是否有缓存降级方案？

---

**下一篇**：[分布式系统](./03-distributed-systems.md)

