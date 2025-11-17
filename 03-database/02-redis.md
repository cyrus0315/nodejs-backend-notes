# Redis

Redis（Remote Dictionary Server）是一个开源的内存数据结构存储系统，可用作数据库、缓存和消息代理。

## Redis 基础

### 数据类型

#### 1. String（字符串）

```bash
# 设置值
SET key "value"
SET user:1:name "Alice"
SET counter 0

# 获取值
GET user:1:name

# 设置过期时间
SETEX session:abc 3600 "user_data"  # 3600 秒后过期
SET key "value" EX 3600              # 同上

# 仅当键不存在时设置
SETNX lock:resource "locked"

# 批量操作
MSET key1 "value1" key2 "value2"
MGET key1 key2

# 数值操作
INCR counter           # 加 1
INCRBY counter 5       # 加 5
DECR counter           # 减 1
DECRBY counter 3       # 减 3

# 追加
APPEND key "more_data"

# 获取部分字符串
GETRANGE key 0 10
```

#### 2. Hash（哈希表）

```bash
# 设置哈希字段
HSET user:1 name "Alice"
HSET user:1 age 25
HSET user:1 email "alice@example.com"

# 批量设置
HMSET user:2 name "Bob" age 30 email "bob@example.com"

# 获取字段
HGET user:1 name

# 获取多个字段
HMGET user:1 name age

# 获取所有字段
HGETALL user:1

# 检查字段是否存在
HEXISTS user:1 name

# 获取所有字段名
HKEYS user:1

# 获取所有值
HVALS user:1

# 数值操作
HINCRBY user:1 age 1
```

#### 3. List（列表）

```bash
# 从左侧插入
LPUSH queue:tasks "task1"
LPUSH queue:tasks "task2"

# 从右侧插入
RPUSH queue:tasks "task3"

# 从左侧弹出
LPOP queue:tasks

# 从右侧弹出
RPOP queue:tasks

# 阻塞弹出（用于消息队列）
BLPOP queue:tasks 10  # 10 秒超时

# 获取列表长度
LLEN queue:tasks

# 获取范围
LRANGE queue:tasks 0 -1  # 获取所有

# 修剪列表
LTRIM queue:tasks 0 99   # 只保留前 100 个

# 获取指定索引的元素
LINDEX queue:tasks 0

# 设置指定索引的元素
LSET queue:tasks 0 "new_task"
```

#### 4. Set（集合）

```bash
# 添加成员
SADD tags:article:1 "redis" "database" "nosql"

# 获取所有成员
SMEMBERS tags:article:1

# 检查成员是否存在
SISMEMBER tags:article:1 "redis"

# 获取成员数量
SCARD tags:article:1

# 随机获取成员
SRANDMEMBER tags:article:1 2

# 移除成员
SREM tags:article:1 "nosql"

# 弹出成员
SPOP tags:article:1

# 集合运算
SINTER set1 set2        # 交集
SUNION set1 set2        # 并集
SDIFF set1 set2         # 差集
```

#### 5. Sorted Set（有序集合）

```bash
# 添加成员（带分数）
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"

# 获取成员分数
ZSCORE leaderboard "player1"

# 增加分数
ZINCRBY leaderboard 10 "player1"

# 获取排名（从 0 开始）
ZRANK leaderboard "player1"      # 正序排名
ZREVRANK leaderboard "player1"   # 倒序排名

# 按分数范围获取
ZRANGE leaderboard 0 -1 WITHSCORES        # 全部，正序
ZREVRANGE leaderboard 0 9 WITHSCORES      # 前 10，倒序
ZRANGEBYSCORE leaderboard 100 200         # 分数范围

# 获取成员数量
ZCARD leaderboard

# 获取分数范围内的成员数量
ZCOUNT leaderboard 100 200

# 删除成员
ZREM leaderboard "player1"

# 按排名范围删除
ZREMRANGEBYRANK leaderboard 0 9

# 按分数范围删除
ZREMRANGEBYSCORE leaderboard 0 100
```

### 通用命令

```bash
# 检查键是否存在
EXISTS key

# 删除键
DEL key

# 设置过期时间
EXPIRE key 3600          # 秒
PEXPIRE key 3600000      # 毫秒
EXPIREAT key 1609459200  # Unix 时间戳

# 查看剩余过期时间
TTL key                  # 秒
PTTL key                 # 毫秒

# 移除过期时间
PERSIST key

# 重命名键
RENAME oldkey newkey

# 获取键的类型
TYPE key

# 扫描键（用于生产环境）
SCAN 0 MATCH user:* COUNT 100
```

## Node.js 中使用 Redis

### ioredis（推荐）

```typescript
import Redis from 'ioredis';

// 单机连接
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'password',
  db: 0,
  keyPrefix: 'myapp:',
  retryStrategy: (times) => {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
});

// 基本操作
async function basicOperations() {
  // String
  await redis.set('user:1:name', 'Alice');
  const name = await redis.get('user:1:name');
  
  // 带过期时间
  await redis.setex('session:abc', 3600, 'session_data');
  
  // Hash
  await redis.hset('user:1', 'name', 'Alice');
  await redis.hset('user:1', 'age', '25');
  const user = await redis.hgetall('user:1');
  
  // List
  await redis.lpush('queue:tasks', 'task1', 'task2');
  const task = await redis.rpop('queue:tasks');
  
  // Set
  await redis.sadd('tags:1', 'redis', 'database');
  const tags = await redis.smembers('tags:1');
  
  // Sorted Set
  await redis.zadd('leaderboard', 100, 'player1');
  const topPlayers = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
}

// Pipeline（批量操作）
async function usePipeline() {
  const pipeline = redis.pipeline();
  
  pipeline.set('key1', 'value1');
  pipeline.set('key2', 'value2');
  pipeline.get('key1');
  pipeline.get('key2');
  
  const results = await pipeline.exec();
  // results: [[null, 'OK'], [null, 'OK'], [null, 'value1'], [null, 'value2']]
}

// 事务
async function useTransaction() {
  const multi = redis.multi();
  
  multi.set('key1', 'value1');
  multi.set('key2', 'value2');
  multi.get('key1');
  
  const results = await multi.exec();
}

// Lua 脚本
async function useLuaScript() {
  // 原子性增加库存并返回
  const script = `
    local stock = redis.call('get', KEYS[1])
    if tonumber(stock) >= tonumber(ARGV[1]) then
      redis.call('decrby', KEYS[1], ARGV[1])
      return 1
    else
      return 0
    end
  `;
  
  const result = await redis.eval(script, 1, 'product:1:stock', 5);
  
  // 或使用 evalsha（更高效）
  const sha = await redis.script('load', script);
  const result2 = await redis.evalsha(sha, 1, 'product:1:stock', 5);
}

// 发布订阅
async function pubSub() {
  // 订阅者
  const subscriber = new Redis();
  
  subscriber.subscribe('news', 'music', (err, count) => {
    console.log(`Subscribed to ${count} channels`);
  });
  
  subscriber.on('message', (channel, message) => {
    console.log(`Received ${message} from ${channel}`);
  });
  
  // 发布者
  const publisher = new Redis();
  await publisher.publish('news', 'Hello World!');
}

// 集群连接
const cluster = new Redis.Cluster([
  { host: '127.0.0.1', port: 6379 },
  { host: '127.0.0.1', port: 6380 },
]);

// 关闭连接
await redis.quit();
```

## 常见使用场景

### 1. 缓存

```typescript
class CacheService {
  constructor(private redis: Redis) {}
  
  async get<T>(key: string): Promise<T | null> {
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }
  
  async set(key: string, value: any, ttl: number = 3600): Promise<void> {
    await this.redis.setex(key, ttl, JSON.stringify(value));
  }
  
  async del(key: string): Promise<void> {
    await this.redis.del(key);
  }
  
  // 缓存模式：Cache-Aside
  async getWithFallback<T>(
    key: string,
    fallback: () => Promise<T>,
    ttl: number = 3600
  ): Promise<T> {
    // 先查缓存
    const cached = await this.get<T>(key);
    if (cached) {
      return cached;
    }
    
    // 缓存未命中，查数据库
    const data = await fallback();
    
    // 写入缓存
    await this.set(key, data, ttl);
    
    return data;
  }
  
  // 防止缓存穿透
  async getWithNullCache<T>(
    key: string,
    fallback: () => Promise<T | null>,
    ttl: number = 3600
  ): Promise<T | null> {
    const cached = await this.redis.get(key);
    
    if (cached === 'null') {
      return null;
    }
    
    if (cached) {
      return JSON.parse(cached);
    }
    
    const data = await fallback();
    
    if (data === null) {
      // 缓存空值，但设置较短过期时间
      await this.redis.setex(key, 60, 'null');
    } else {
      await this.set(key, data, ttl);
    }
    
    return data;
  }
}

// 使用
const cache = new CacheService(redis);

async function getUser(id: string) {
  return cache.getWithFallback(
    `user:${id}`,
    async () => {
      const user = await db.users.findById(id);
      return user;
    },
    3600
  );
}
```

### 2. 分布式锁

```typescript
class RedisLock {
  constructor(private redis: Redis) {}
  
  async acquire(
    resource: string,
    ttl: number = 10000,
    retries: number = 3,
    retryDelay: number = 100
  ): Promise<string | null> {
    const identifier = Math.random().toString(36).substring(7);
    const key = `lock:${resource}`;
    
    for (let i = 0; i < retries; i++) {
      // 使用 SET NX EX 原子操作
      const result = await this.redis.set(
        key,
        identifier,
        'PX',
        ttl,
        'NX'
      );
      
      if (result === 'OK') {
        return identifier;
      }
      
      // 等待后重试
      await new Promise(resolve => setTimeout(resolve, retryDelay));
    }
    
    return null;
  }
  
  async release(resource: string, identifier: string): Promise<boolean> {
    const key = `lock:${resource}`;
    
    // 使用 Lua 脚本保证原子性
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(script, 1, key, identifier);
    return result === 1;
  }
  
  async withLock<T>(
    resource: string,
    callback: () => Promise<T>,
    ttl: number = 10000
  ): Promise<T> {
    const lock = await this.acquire(resource, ttl);
    
    if (!lock) {
      throw new Error('Failed to acquire lock');
    }
    
    try {
      return await callback();
    } finally {
      await this.release(resource, lock);
    }
  }
}

// 使用
const lock = new RedisLock(redis);

async function processOrder(orderId: string) {
  return lock.withLock(`order:${orderId}`, async () => {
    // 处理订单的关键逻辑
    const order = await db.orders.findById(orderId);
    order.status = 'processing';
    await db.orders.update(orderId, order);
    return order;
  });
}
```

### 3. 限流

```typescript
class RateLimiter {
  constructor(private redis: Redis) {}
  
  // 固定窗口限流
  async checkFixedWindow(
    key: string,
    limit: number,
    window: number // 秒
  ): Promise<boolean> {
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, window);
    }
    
    return current <= limit;
  }
  
  // 滑动窗口限流
  async checkSlidingWindow(
    key: string,
    limit: number,
    window: number // 秒
  ): Promise<boolean> {
    const now = Date.now();
    const windowStart = now - window * 1000;
    
    const pipe = this.redis.pipeline();
    
    // 删除过期的请求
    pipe.zremrangebyscore(key, 0, windowStart);
    
    // 获取当前窗口内的请求数
    pipe.zcard(key);
    
    // 添加当前请求
    pipe.zadd(key, now, `${now}-${Math.random()}`);
    
    // 设置过期时间
    pipe.expire(key, window);
    
    const results = await pipe.exec();
    const count = results[1][1] as number;
    
    return count < limit;
  }
  
  // 令牌桶算法
  async checkTokenBucket(
    key: string,
    capacity: number,
    refillRate: number // 每秒补充的令牌数
  ): Promise<boolean> {
    const now = Date.now();
    const script = `
      local capacity = tonumber(ARGV[1])
      local refillRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      local tokens = tonumber(redis.call('hget', KEYS[1], 'tokens') or capacity)
      local lastRefill = tonumber(redis.call('hget', KEYS[1], 'lastRefill') or now)
      
      local timePassed = (now - lastRefill) / 1000
      local tokensToAdd = timePassed * refillRate
      tokens = math.min(capacity, tokens + tokensToAdd)
      
      if tokens >= 1 then
        tokens = tokens - 1
        redis.call('hset', KEYS[1], 'tokens', tokens)
        redis.call('hset', KEYS[1], 'lastRefill', now)
        return 1
      else
        return 0
      end
    `;
    
    const result = await this.redis.eval(
      script,
      1,
      key,
      capacity,
      refillRate,
      now
    );
    
    return result === 1;
  }
}

// 使用
const limiter = new RateLimiter(redis);

async function handleRequest(userId: string) {
  const allowed = await limiter.checkSlidingWindow(
    `rate:user:${userId}`,
    100,  // 100 次
    60    // 60 秒
  );
  
  if (!allowed) {
    throw new Error('Rate limit exceeded');
  }
  
  // 处理请求
}
```

### 4. 会话管理

```typescript
interface Session {
  userId: string;
  data: any;
  expiresAt: number;
}

class SessionManager {
  constructor(private redis: Redis) {}
  
  async create(userId: string, data: any, ttl: number = 3600): Promise<string> {
    const sessionId = Math.random().toString(36).substring(7);
    const key = `session:${sessionId}`;
    
    const session: Session = {
      userId,
      data,
      expiresAt: Date.now() + ttl * 1000
    };
    
    await this.redis.setex(key, ttl, JSON.stringify(session));
    
    return sessionId;
  }
  
  async get(sessionId: string): Promise<Session | null> {
    const key = `session:${sessionId}`;
    const data = await this.redis.get(key);
    
    if (!data) {
      return null;
    }
    
    return JSON.parse(data);
  }
  
  async refresh(sessionId: string, ttl: number = 3600): Promise<boolean> {
    const key = `session:${sessionId}`;
    const exists = await this.redis.exists(key);
    
    if (!exists) {
      return false;
    }
    
    await this.redis.expire(key, ttl);
    return true;
  }
  
  async destroy(sessionId: string): Promise<void> {
    const key = `session:${sessionId}`;
    await this.redis.del(key);
  }
  
  async getUserSessions(userId: string): Promise<string[]> {
    const pattern = 'session:*';
    const sessions: string[] = [];
    
    let cursor = '0';
    do {
      const [newCursor, keys] = await this.redis.scan(
        cursor,
        'MATCH',
        pattern,
        'COUNT',
        100
      );
      
      cursor = newCursor;
      
      for (const key of keys) {
        const data = await this.redis.get(key);
        if (data) {
          const session: Session = JSON.parse(data);
          if (session.userId === userId) {
            sessions.push(key.replace('session:', ''));
          }
        }
      }
    } while (cursor !== '0');
    
    return sessions;
  }
}
```

### 5. 排行榜

```typescript
class Leaderboard {
  constructor(
    private redis: Redis,
    private key: string
  ) {}
  
  async addScore(userId: string, score: number): Promise<void> {
    await this.redis.zadd(this.key, score, userId);
  }
  
  async incrementScore(userId: string, increment: number): Promise<number> {
    return await this.redis.zincrby(this.key, increment, userId);
  }
  
  async getScore(userId: string): Promise<number | null> {
    const score = await this.redis.zscore(this.key, userId);
    return score ? parseFloat(score) : null;
  }
  
  async getRank(userId: string): Promise<number | null> {
    // 倒序排名（分数高的排前面）
    const rank = await this.redis.zrevrank(this.key, userId);
    return rank !== null ? rank + 1 : null;
  }
  
  async getTopPlayers(count: number = 10): Promise<Array<{ userId: string; score: number; rank: number }>> {
    const results = await this.redis.zrevrange(
      this.key,
      0,
      count - 1,
      'WITHSCORES'
    );
    
    const players: Array<{ userId: string; score: number; rank: number }> = [];
    
    for (let i = 0; i < results.length; i += 2) {
      players.push({
        userId: results[i],
        score: parseFloat(results[i + 1]),
        rank: i / 2 + 1
      });
    }
    
    return players;
  }
  
  async getPlayersInRange(start: number, end: number): Promise<Array<{ userId: string; score: number; rank: number }>> {
    const results = await this.redis.zrevrange(
      this.key,
      start - 1,
      end - 1,
      'WITHSCORES'
    );
    
    const players: Array<{ userId: string; score: number; rank: number }> = [];
    
    for (let i = 0; i < results.length; i += 2) {
      players.push({
        userId: results[i],
        score: parseFloat(results[i + 1]),
        rank: start + i / 2
      });
    }
    
    return players;
  }
  
  async removePlayer(userId: string): Promise<void> {
    await this.redis.zrem(this.key, userId);
  }
  
  async getTotalPlayers(): Promise<number> {
    return await this.redis.zcard(this.key);
  }
}

// 使用
const leaderboard = new Leaderboard(redis, 'game:leaderboard');

await leaderboard.addScore('player1', 1000);
await leaderboard.incrementScore('player1', 50);

const rank = await leaderboard.getRank('player1');
const topPlayers = await leaderboard.getTopPlayers(10);
```

## 缓存策略

### 1. Cache-Aside（旁路缓存）

```typescript
async function cacheAside(key: string) {
  // 读取
  let data = await redis.get(key);
  
  if (!data) {
    // 缓存未命中，查数据库
    data = await db.query(key);
    
    // 写入缓存
    await redis.setex(key, 3600, JSON.stringify(data));
  } else {
    data = JSON.parse(data);
  }
  
  return data;
}

// 更新
async function updateData(key: string, newData: any) {
  // 更新数据库
  await db.update(key, newData);
  
  // 删除缓存
  await redis.del(key);
}
```

### 2. Read-Through / Write-Through

```typescript
class CacheProxy {
  async get(key: string) {
    // 先查缓存
    let data = await redis.get(key);
    
    if (!data) {
      // 查数据库
      data = await db.query(key);
      
      // 自动写入缓存
      await redis.setex(key, 3600, data);
    }
    
    return data;
  }
  
  async set(key: string, value: any) {
    // 同时写入数据库和缓存
    await db.update(key, value);
    await redis.setex(key, 3600, value);
  }
}
```

### 3. Write-Behind（异步写入）

```typescript
class WriteBehindCache {
  private writeQueue: Map<string, any> = new Map();
  
  constructor(private redis: Redis) {
    // 定期刷新到数据库
    setInterval(() => this.flush(), 5000);
  }
  
  async set(key: string, value: any) {
    // 立即写入缓存
    await this.redis.setex(key, 3600, JSON.stringify(value));
    
    // 加入写入队列
    this.writeQueue.set(key, value);
  }
  
  private async flush() {
    if (this.writeQueue.size === 0) return;
    
    const batch = Array.from(this.writeQueue.entries());
    this.writeQueue.clear();
    
    // 批量写入数据库
    await db.batchUpdate(batch);
  }
}
```

## 常见问题

### 缓存穿透

**问题**：查询不存在的数据，导致每次都打到数据库。

**解决方案**：

```typescript
// 1. 缓存空值
async function getUser(id: string) {
  const cached = await redis.get(`user:${id}`);
  
  if (cached === 'null') {
    return null;
  }
  
  if (cached) {
    return JSON.parse(cached);
  }
  
  const user = await db.users.findById(id);
  
  if (!user) {
    // 缓存空值，但设置较短过期时间
    await redis.setex(`user:${id}`, 60, 'null');
    return null;
  }
  
  await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
  return user;
}

// 2. 布隆过滤器
import { BloomFilter } from 'bloom-filters';

const filter = new BloomFilter(10000, 4);

// 初始化：将所有存在的 ID 加入过滤器
const allIds = await db.users.getAllIds();
allIds.forEach(id => filter.add(id));

async function getUserWithBloom(id: string) {
  // 先检查过滤器
  if (!filter.has(id)) {
    return null; // 一定不存在
  }
  
  // 可能存在，继续查询
  return getUser(id);
}
```

### 缓存击穿

**问题**：热点数据过期，大量请求同时打到数据库。

**解决方案**：

```typescript
// 互斥锁
async function getUserWithLock(id: string) {
  const cacheKey = `user:${id}`;
  const lockKey = `lock:user:${id}`;
  
  // 先查缓存
  let user = await redis.get(cacheKey);
  if (user) {
    return JSON.parse(user);
  }
  
  // 尝试获取锁
  const lock = await redis.set(lockKey, '1', 'EX', 10, 'NX');
  
  if (lock === 'OK') {
    try {
      // 获取到锁，查询数据库
      user = await db.users.findById(id);
      await redis.setex(cacheKey, 3600, JSON.stringify(user));
      return user;
    } finally {
      await redis.del(lockKey);
    }
  } else {
    // 没有获取到锁，等待后重试
    await new Promise(resolve => setTimeout(resolve, 50));
    return getUserWithLock(id);
  }
}
```

### 缓存雪崩

**问题**：大量缓存同时过期，导致数据库压力骤增。

**解决方案**：

```typescript
// 1. 随机过期时间
async function setWithRandomExpire(key: string, value: any, baseExpire: number) {
  const randomExpire = baseExpire + Math.floor(Math.random() * 300);
  await redis.setex(key, randomExpire, JSON.stringify(value));
}

// 2. 永不过期（后台更新）
class CacheWarmer {
  async warmCache(key: string, fetchData: () => Promise<any>) {
    const data = await fetchData();
    await redis.set(key, JSON.stringify(data)); // 不设置过期时间
    
    // 后台定期更新
    setInterval(async () => {
      const newData = await fetchData();
      await redis.set(key, JSON.stringify(newData));
    }, 3000000); // 50 分钟更新一次
  }
}
```

## 常见面试题

### 1. Redis 为什么这么快？

<details>
<summary>点击查看答案</summary>

1. **内存存储**：所有数据都在内存中
2. **单线程模型**：避免上下文切换和竞态条件
3. **I/O 多路复用**：使用 epoll/kqueue 处理并发连接
4. **高效的数据结构**：针对不同场景优化的数据结构
5. **简单的协议**：RESP 协议简单高效
</details>

### 2. Redis 的持久化方式？

<details>
<summary>点击查看答案</summary>

**RDB（快照）**：
- 定期保存数据快照
- 恢复快
- 可能丢失最后一次快照后的数据

**AOF（追加文件）**：
- 记录所有写操作
- 数据更安全
- 文件更大，恢复慢

**混合持久化**（推荐）：
- RDB + AOF
- 兼顾性能和数据安全
</details>

### 3. 如何实现分布式锁？

<details>
<summary>点击查看答案</summary>

使用 `SET key value NX EX seconds` 原子操作：

```typescript
async function acquireLock(key: string, ttl: number = 10) {
  const identifier = Math.random().toString(36);
  const result = await redis.set(key, identifier, 'EX', ttl, 'NX');
  return result === 'OK' ? identifier : null;
}

async function releaseLock(key: string, identifier: string) {
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  return await redis.eval(script, 1, key, identifier);
}
```
</details>

### 4. Redis 集群方案？

<details>
<summary>点击查看答案</summary>

1. **主从复制**：一主多从，读写分离
2. **哨兵模式**：自动故障转移
3. **Redis Cluster**：官方集群方案，分片存储
4. **代理方案**：Twemproxy, Codis

**选择建议**：
- 小规模 → 主从复制
- 中等规模 → 哨兵模式
- 大规模 → Redis Cluster
</details>

### 5. 缓存和数据库一致性？

<details>
<summary>点击查看答案</summary>

**方案**：

1. **先删缓存，再更新数据库**（推荐）
```typescript
await redis.del(key);
await db.update(data);
// 可能的延迟双删
setTimeout(() => redis.del(key), 1000);
```

2. **先更新数据库，再删缓存**
```typescript
await db.update(data);
await redis.del(key);
```

3. **使用消息队列保证最终一致性**
```typescript
await db.update(data);
await mq.publish('cache.invalidate', { key });
```

**最佳实践**：
- 设置合理的过期时间
- 使用延迟双删
- 允许短时间的不一致
</details>

## 最佳实践

1. **使用连接池**
2. **设置合理的过期时间**
3. **使用 Pipeline 批量操作**
4. **避免大 Key**
5. **使用合适的数据结构**
6. **监控 Redis 性能**
7. **定期备份数据**
8. **使用 Lua 脚本保证原子性**

