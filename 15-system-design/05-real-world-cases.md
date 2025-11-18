# 系统设计实战案例

本文通过经典的系统设计案例，展示如何综合运用前面学到的知识解决实际问题。

## 目录
- [秒杀系统](#秒杀系统)
- [短链接系统](#短链接系统)
- [分布式限流系统](#分布式限流系统)
- [实时消息系统（IM）](#实时消息系统)
- [新闻推送系统](#新闻推送系统)
- [面试技巧](#系统设计面试技巧)

---

## 秒杀系统

### 需求分析

**场景**：1000 件商品，100 万用户抢购。

**挑战**：
- 超高并发（10 万+ QPS）
- 超卖问题
- 数据库压力
- 库存一致性

### 架构设计

```
用户 → CDN → WAF → 网关 → 秒杀服务 → Redis → 数据库
              ↓              ↓          ↓
           限流器        消息队列    库存服务
```

### 核心技术方案

#### 1. 前端限流

```typescript
// 防止重复点击
let isSubmitting = false;
const countdown = 5; // 5 秒倒计时

async function seckill() {
  if (isSubmitting) return;
  
  isSubmitting = true;
  button.disabled = true;
  
  try {
    const response = await fetch('/api/seckill', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-Request-ID': generateRequestId() // 防重复提交
      },
      body: JSON.stringify({ productId: 123 })
    });
    
    const result = await response.json();
    alert(result.message);
  } finally {
    // 5 秒后才能再次点击
    setTimeout(() => {
      isSubmitting = false;
      button.disabled = false;
    }, countdown * 1000);
  }
}
```

#### 2. 网关层限流

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';

// 全局限流
const globalLimiter = rateLimit({
  store: new RedisStore({ client: redis }),
  windowMs: 1000, // 1 秒
  max: 10000, // 最多 10000 个请求
  message: 'Too many requests'
});

// 用户维度限流
const userLimiter = rateLimit({
  store: new RedisStore({ client: redis, prefix: 'rl:user:' }),
  windowMs: 60 * 1000, // 1 分钟
  max: 5, // 每个用户最多 5 次
  keyGenerator: (req) => req.user.id,
  message: 'Too many attempts, please try again later'
});

app.post('/api/seckill', globalLimiter, userLimiter, seckillHandler);
```

#### 3. Redis 预扣库存

```typescript
class SeckillService {
  private redis: Redis;
  private productId: number;
  private stock: number;

  constructor(redis: Redis, productId: number, stock: number) {
    this.redis = redis;
    this.productId = productId;
    this.stock = stock;
  }

  // 初始化：将库存放入 Redis
  async init() {
    await this.redis.set(`seckill:stock:${this.productId}`, this.stock);
  }

  // 秒杀：Redis 原子扣库存
  async buy(userId: number): Promise<{ success: boolean; message: string }> {
    const requestId = `${userId}-${Date.now()}`;
    
    // Lua 脚本：原子性检查和扣减库存
    const script = `
      local stock = redis.call('get', KEYS[1])
      if tonumber(stock) <= 0 then
        return 0  -- 库存不足
      end
      
      -- 检查用户是否已购买
      local purchased = redis.call('sismember', KEYS[2], ARGV[1])
      if purchased == 1 then
        return -1  -- 已购买
      end
      
      -- 扣减库存
      redis.call('decr', KEYS[1])
      
      -- 记录购买用户
      redis.call('sadd', KEYS[2], ARGV[1])
      
      -- 加入订单队列
      redis.call('lpush', KEYS[3], ARGV[2])
      
      return 1  -- 成功
    `;

    const result = await this.redis.eval(
      script,
      3,
      `seckill:stock:${this.productId}`,
      `seckill:users:${this.productId}`,
      `seckill:orders:${this.productId}`,
      userId.toString(),
      JSON.stringify({ userId, productId: this.productId, requestId })
    );

    if (result === 0) {
      return { success: false, message: '商品已售罄' };
    } else if (result === -1) {
      return { success: false, message: '您已购买过该商品' };
    }

    return { success: true, message: '抢购成功，请等待订单生成' };
  }
}

// 使用
const seckillService = new SeckillService(redis, 123, 1000);

// 秒杀开始前初始化
await seckillService.init();

// 秒杀接口
app.post('/api/seckill', authenticateUser, async (req, res) => {
  const result = await seckillService.buy(req.user.id);
  
  if (result.success) {
    res.status(200).json(result);
  } else {
    res.status(400).json(result);
  }
});
```

#### 4. 异步创建订单

```typescript
// 订单处理 Worker
class OrderWorker {
  constructor(
    private redis: Redis,
    private prisma: PrismaClient
  ) {}

  async start() {
    console.log('Order worker started');
    
    while (true) {
      try {
        // 从队列中取出订单
        const orderData = await this.redis.rpop('seckill:orders:123');
        
        if (!orderData) {
          await new Promise(resolve => setTimeout(resolve, 100));
          continue;
        }

        const order = JSON.parse(orderData);
        await this.processOrder(order);
      } catch (error) {
        console.error('Order processing error:', error);
      }
    }
  }

  private async processOrder(orderData: any) {
    const { userId, productId, requestId } = orderData;

    try {
      // 创建订单
      await this.prisma.order.create({
        data: {
          userId,
          productId,
          quantity: 1,
          status: 'pending',
          requestId,
          createdAt: new Date()
        }
      });

      // 扣减数据库库存
      await this.prisma.product.update({
        where: { id: productId },
        data: { stock: { decrement: 1 } }
      });

      console.log(`Order created for user ${userId}`);
      
      // 发送通知
      await this.sendNotification(userId, '抢购成功，订单已生成');
    } catch (error) {
      console.error('Failed to create order:', error);
      
      // 失败：恢复 Redis 库存
      await this.redis.incr(`seckill:stock:${productId}`);
      await this.redis.srem(`seckill:users:${productId}`, userId);
    }
  }

  private async sendNotification(userId: number, message: string) {
    // 发送通知（邮件、短信、推送等）
    console.log(`Notification sent to user ${userId}: ${message}`);
  }
}

// 启动多个 Worker
for (let i = 0; i < 5; i++) {
  new OrderWorker(redis, prisma).start();
}
```

#### 5. 库存补偿机制

```typescript
// 定时任务：检查 Redis 和数据库库存是否一致
async function syncStock() {
  const productId = 123;
  
  // Redis 库存
  const redisStock = parseInt(await redis.get(`seckill:stock:${productId}`) || '0');
  
  // 数据库库存
  const product = await prisma.product.findUnique({
    where: { id: productId }
  });
  const dbStock = product?.stock || 0;

  if (redisStock !== dbStock) {
    console.warn(`Stock mismatch: Redis=${redisStock}, DB=${dbStock}`);
    
    // 以数据库为准，修正 Redis
    await redis.set(`seckill:stock:${productId}`, dbStock);
  }
}

// 每分钟同步一次
setInterval(syncStock, 60000);
```

### 完整流程

```typescript
// 秒杀系统完整实现
import express from 'express';
import Redis from 'ioredis';
import { PrismaClient } from '@prisma/client';

const app = express();
const redis = new Redis();
const prisma = new PrismaClient();

// 秒杀配置
const SECKILL_CONFIG = {
  productId: 123,
  stock: 1000,
  startTime: new Date('2024-12-12 20:00:00'),
  endTime: new Date('2024-12-12 20:05:00')
};

// 初始化秒杀
async function initSeckill() {
  const { productId, stock } = SECKILL_CONFIG;
  
  // 设置库存
  await redis.set(`seckill:stock:${productId}`, stock);
  
  // 清空已购买用户
  await redis.del(`seckill:users:${productId}`);
  
  // 清空订单队列
  await redis.del(`seckill:orders:${productId}`);
  
  console.log('Seckill initialized');
}

// 秒杀接口
app.post('/api/seckill', 
  authenticateUser,
  checkSeckillTime,
  userLimiter,
  async (req, res) => {
    const userId = req.user.id;
    const { productId } = SECKILL_CONFIG;

    // 秒杀
    const seckill = new SeckillService(redis, productId, 0);
    const result = await seckill.buy(userId);

    res.json(result);
  }
);

// 检查秒杀时间
function checkSeckillTime(req, res, next) {
  const now = new Date();
  const { startTime, endTime } = SECKILL_CONFIG;

  if (now < startTime) {
    return res.status(400).json({ message: '秒杀还未开始' });
  }

  if (now > endTime) {
    return res.status(400).json({ message: '秒杀已结束' });
  }

  next();
}

// 秒杀开始前初始化
initSeckill();

// 启动 Worker
for (let i = 0; i < 5; i++) {
  new OrderWorker(redis, prisma).start();
}

app.listen(3000);
```

---

## 短链接系统

### 需求分析

将长 URL 转换为短 URL，如：
- `https://example.com/articles/123456?utm_source=abc` 
- → `https://short.link/aB3cD`

**需求**：
- 生成唯一短链接
- 高性能（百万级 QPS）
- 统计点击量

### 架构设计

```
用户 → CDN → 短链接服务 → Redis → 数据库
                ↓           ↓
            分析服务      布隆过滤器
```

### 核心实现

#### 1. 短链接生成算法

```typescript
import crypto from 'crypto';

class ShortUrlGenerator {
  private readonly chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  private readonly base = this.chars.length; // 62

  // 方案 1：Base62 编码（自增 ID）
  encode(id: number): string {
    if (id === 0) return this.chars[0];

    let short = '';
    while (id > 0) {
      short = this.chars[id % this.base] + short;
      id = Math.floor(id / this.base);
    }
    return short;
  }

  decode(short: string): number {
    let id = 0;
    for (let i = 0; i < short.length; i++) {
      id = id * this.base + this.chars.indexOf(short[i]);
    }
    return id;
  }

  // 方案 2：哈希算法
  hash(longUrl: string): string {
    const hash = crypto.createHash('md5').update(longUrl).digest('hex');
    
    // 取前 7 个字符
    let short = '';
    for (let i = 0; i < 7; i++) {
      const index = parseInt(hash.substring(i * 4, i * 4 + 4), 16) % this.base;
      short += this.chars[index];
    }
    
    return short;
  }

  // 方案 3：随机生成 + 冲突检测
  async random(length = 6): Promise<string> {
    const randomBytes = crypto.randomBytes(length);
    let short = '';
    
    for (let i = 0; i < length; i++) {
      short += this.chars[randomBytes[i] % this.base];
    }
    
    return short;
  }
}

const generator = new ShortUrlGenerator();

// 使用自增 ID
const short1 = generator.encode(123456); // "W7e"
const id1 = generator.decode('W7e'); // 123456

// 使用哈希
const short2 = generator.hash('https://example.com/articles/123456');
```

#### 2. 短链接服务

```typescript
import { PrismaClient } from '@prisma/client';
import Redis from 'ioredis';
import { BloomFilter } from 'bloomfilter';

class ShortUrlService {
  private prisma: PrismaClient;
  private redis: Redis;
  private generator: ShortUrlGenerator;
  private bloomFilter: BloomFilter;

  constructor() {
    this.prisma = new PrismaClient();
    this.redis = new Redis();
    this.generator = new ShortUrlGenerator();
    
    // 布隆过滤器（1000 万容量）
    this.bloomFilter = new BloomFilter(10000000, 10);
    this.initBloomFilter();
  }

  private async initBloomFilter() {
    // 加载所有短链接到布隆过滤器
    const urls = await this.prisma.shortUrl.findMany({
      select: { shortCode: true }
    });
    
    for (const url of urls) {
      this.bloomFilter.add(url.shortCode);
    }
    
    console.log(`Loaded ${urls.length} short URLs into bloom filter`);
  }

  // 创建短链接
  async create(longUrl: string, userId?: number): Promise<string> {
    // 1. 检查是否已存在
    const existing = await this.findByLongUrl(longUrl);
    if (existing) {
      return existing;
    }

    // 2. 生成短链接（重试机制）
    let shortCode: string;
    let retries = 0;
    const maxRetries = 5;

    do {
      shortCode = await this.generator.random(6);
      retries++;
      
      // 布隆过滤器快速检查
      if (!this.bloomFilter.test(shortCode)) {
        break; // 不存在，可以使用
      }
      
      // 可能存在，查数据库确认
      const exists = await this.redis.exists(`short:${shortCode}`);
      if (!exists) {
        break;
      }
    } while (retries < maxRetries);

    if (retries >= maxRetries) {
      throw new Error('Failed to generate unique short code');
    }

    // 3. 保存到数据库
    await this.prisma.shortUrl.create({
      data: {
        shortCode,
        longUrl,
        userId,
        createdAt: new Date()
      }
    });

    // 4. 写入缓存
    await this.redis.setex(`short:${shortCode}`, 86400, longUrl);

    // 5. 加入布隆过滤器
    this.bloomFilter.add(shortCode);

    return shortCode;
  }

  // 查询长链接（高性能）
  async getLongUrl(shortCode: string): Promise<string | null> {
    // 1. 布隆过滤器：快速判断不存在
    if (!this.bloomFilter.test(shortCode)) {
      return null;
    }

    // 2. Redis 缓存
    let longUrl = await this.redis.get(`short:${shortCode}`);
    if (longUrl) {
      // 异步统计点击量
      this.incrementClicks(shortCode);
      return longUrl;
    }

    // 3. 数据库
    const record = await this.prisma.shortUrl.findUnique({
      where: { shortCode }
    });

    if (record) {
      // 写入缓存
      await this.redis.setex(`short:${shortCode}`, 86400, record.longUrl);
      this.incrementClicks(shortCode);
      return record.longUrl;
    }

    return null;
  }

  // 统计点击量
  private async incrementClicks(shortCode: string) {
    // Redis 计数（异步，不阻塞）
    this.redis.incr(`clicks:${shortCode}`).catch(console.error);

    // 定期同步到数据库（批量）
    this.redis.lpush('clicks:queue', shortCode).catch(console.error);
  }

  private async findByLongUrl(longUrl: string): Promise<string | null> {
    const record = await this.prisma.shortUrl.findFirst({
      where: { longUrl }
    });
    return record?.shortCode || null;
  }
}

// 使用
const shortUrlService = new ShortUrlService();

// 创建短链接
app.post('/api/shorten', async (req, res) => {
  const { url } = req.body;
  
  if (!isValidUrl(url)) {
    return res.status(400).json({ error: 'Invalid URL' });
  }

  const shortCode = await shortUrlService.create(url, req.user?.id);
  const shortUrl = `https://short.link/${shortCode}`;
  
  res.json({ shortUrl, shortCode });
});

// 重定向
app.get('/:shortCode', async (req, res) => {
  const { shortCode } = req.params;
  
  const longUrl = await shortUrlService.getLongUrl(shortCode);
  
  if (!longUrl) {
    return res.status(404).send('Short URL not found');
  }

  res.redirect(301, longUrl);
});
```

#### 3. 点击统计（批量同步）

```typescript
// Worker: 批量同步点击量到数据库
async function syncClicksWorker() {
  while (true) {
    try {
      // 批量获取点击记录
      const clicks = await redis.rpop('clicks:queue', 100);
      
      if (!clicks || clicks.length === 0) {
        await new Promise(resolve => setTimeout(resolve, 1000));
        continue;
      }

      // 聚合
      const clickCounts = new Map<string, number>();
      for (const shortCode of clicks) {
        clickCounts.set(shortCode, (clickCounts.get(shortCode) || 0) + 1);
      }

      // 批量更新
      await prisma.$transaction(
        Array.from(clickCounts.entries()).map(([shortCode, count]) =>
          prisma.shortUrl.update({
            where: { shortCode },
            data: { clicks: { increment: count } }
          })
        )
      );

      console.log(`Synced ${clicks.length} clicks`);
    } catch (error) {
      console.error('Sync clicks error:', error);
    }
  }
}

syncClicksWorker();
```

---

## 分布式限流系统

### 需求分析

限制 API 调用频率，防止滥用和过载。

**需求**：
- 用户维度限流（每分钟 100 次）
- IP 维度限流（每分钟 1000 次）
- 全局限流（每秒 10000 次）
- 分布式环境下一致

### 限流算法

#### 1. 固定窗口

```typescript
class FixedWindowLimiter {
  constructor(
    private redis: Redis,
    private maxRequests: number,
    private windowMs: number
  ) {}

  async isAllowed(key: string): Promise<boolean> {
    const now = Date.now();
    const windowKey = `${key}:${Math.floor(now / this.windowMs)}`;

    const count = await this.redis.incr(windowKey);
    
    if (count === 1) {
      // 第一次请求，设置过期时间
      await this.redis.pexpire(windowKey, this.windowMs);
    }

    return count <= this.maxRequests;
  }
}

// 使用
const limiter = new FixedWindowLimiter(redis, 100, 60000); // 100 req/min

app.use(async (req, res, next) => {
  const allowed = await limiter.isAllowed(`user:${req.user.id}`);
  
  if (!allowed) {
    return res.status(429).json({ error: 'Too many requests' });
  }
  
  next();
});
```

**问题**：窗口边界流量突刺。

#### 2. 滑动窗口

```typescript
class SlidingWindowLimiter {
  constructor(
    private redis: Redis,
    private maxRequests: number,
    private windowMs: number
  ) {}

  async isAllowed(key: string): Promise<boolean> {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Lua 脚本：原子操作
    const script = `
      local key = KEYS[1]
      local now = tonumber(ARGV[1])
      local window = tonumber(ARGV[2])
      local limit = tonumber(ARGV[3])
      
      -- 删除窗口外的记录
      redis.call('zremrangebyscore', key, 0, now - window)
      
      -- 计数
      local count = redis.call('zcard', key)
      
      if count < limit then
        -- 允许请求，记录时间戳
        redis.call('zadd', key, now, now)
        redis.call('pexpire', key, window)
        return 1
      else
        return 0
      end
    `;

    const result = await this.redis.eval(
      script,
      1,
      key,
      now.toString(),
      this.windowMs.toString(),
      this.maxRequests.toString()
    );

    return result === 1;
  }
}
```

#### 3. 令牌桶

```typescript
class TokenBucketLimiter {
  constructor(
    private redis: Redis,
    private capacity: number,      // 桶容量
    private refillRate: number,    // 每秒补充令牌数
    private refillInterval = 1000  // 补充间隔（毫秒）
  ) {}

  async isAllowed(key: string, tokens = 1): Promise<boolean> {
    const now = Date.now();

    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local refill_rate = tonumber(ARGV[2])
      local refill_interval = tonumber(ARGV[3])
      local now = tonumber(ARGV[4])
      local tokens_requested = tonumber(ARGV[5])
      
      -- 获取当前令牌数和最后补充时间
      local tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)
      local last_refill = tonumber(redis.call('hget', key, 'last_refill') or now)
      
      -- 计算需要补充的令牌数
      local time_passed = now - last_refill
      local refills = math.floor(time_passed / refill_interval)
      local tokens_to_add = refills * refill_rate
      
      -- 补充令牌（不超过容量）
      tokens = math.min(capacity, tokens + tokens_to_add)
      local new_last_refill = last_refill + refills * refill_interval
      
      -- 检查是否有足够的令牌
      if tokens >= tokens_requested then
        tokens = tokens - tokens_requested
        redis.call('hset', key, 'tokens', tokens)
        redis.call('hset', key, 'last_refill', new_last_refill)
        redis.call('expire', key, 3600)
        return 1
      else
        redis.call('hset', key, 'tokens', tokens)
        redis.call('hset', key, 'last_refill', new_last_refill)
        return 0
      end
    `;

    const result = await this.redis.eval(
      script,
      1,
      key,
      this.capacity.toString(),
      this.refillRate.toString(),
      this.refillInterval.toString(),
      now.toString(),
      tokens.toString()
    );

    return result === 1;
  }
}

// 使用
const limiter = new TokenBucketLimiter(redis, 100, 10, 1000);
// 容量 100，每秒补充 10 个令牌

app.use(async (req, res, next) => {
  const allowed = await limiter.isAllowed(`user:${req.user.id}`);
  
  if (!allowed) {
    return res.status(429).json({ error: 'Rate limit exceeded' });
  }
  
  next();
});
```

### 多级限流

```typescript
class MultiLevelRateLimiter {
  private limiters: Array<{
    name: string;
    limiter: SlidingWindowLimiter;
    keyPrefix: string;
  }>;

  constructor(redis: Redis) {
    this.limiters = [
      {
        name: 'global',
        limiter: new SlidingWindowLimiter(redis, 10000, 1000), // 10k/s
        keyPrefix: 'limit:global'
      },
      {
        name: 'ip',
        limiter: new SlidingWindowLimiter(redis, 1000, 60000), // 1k/min
        keyPrefix: 'limit:ip'
      },
      {
        name: 'user',
        limiter: new SlidingWindowLimiter(redis, 100, 60000), // 100/min
        keyPrefix: 'limit:user'
      }
    ];
  }

  async isAllowed(req: Request): Promise<{ allowed: boolean; reason?: string }> {
    // 全局限流
    const globalAllowed = await this.limiters[0].limiter.isAllowed(
      this.limiters[0].keyPrefix
    );
    if (!globalAllowed) {
      return { allowed: false, reason: 'Global rate limit exceeded' };
    }

    // IP 限流
    const ipAllowed = await this.limiters[1].limiter.isAllowed(
      `${this.limiters[1].keyPrefix}:${req.ip}`
    );
    if (!ipAllowed) {
      return { allowed: false, reason: 'IP rate limit exceeded' };
    }

    // 用户限流
    if (req.user) {
      const userAllowed = await this.limiters[2].limiter.isAllowed(
        `${this.limiters[2].keyPrefix}:${req.user.id}`
      );
      if (!userAllowed) {
        return { allowed: false, reason: 'User rate limit exceeded' };
      }
    }

    return { allowed: true };
  }
}

// 使用
const rateLimiter = new MultiLevelRateLimiter(redis);

app.use(async (req, res, next) => {
  const { allowed, reason } = await rateLimiter.isAllowed(req);
  
  if (!allowed) {
    return res.status(429).json({ error: reason });
  }
  
  next();
});
```

---

## 实时消息系统

### 需求分析

类似微信/Slack 的实时聊天系统。

**需求**：
- 实时消息推送
- 消息持久化
- 消息状态（已读/未读）
- 群聊支持
- 历史消息查询

### 架构设计

```
客户端 ←→ WebSocket Gateway ←→ Redis Pub/Sub ←→ 消息服务 → 数据库
                ↓                                          ↓
          连接管理器                                  消息队列
```

### 核心实现

#### 1. WebSocket 服务器

```typescript
import { Server as WebSocketServer } from 'ws';
import { IncomingMessage } from 'http';
import Redis from 'ioredis';

class IMServer {
  private wss: WebSocketServer;
  private redis: Redis;
  private redisSub: Redis;
  private connections = new Map<number, Set<WebSocket>>();

  constructor(port: number) {
    this.wss = new WebSocketServer({ port });
    this.redis = new Redis();
    this.redisSub = new Redis();
    
    this.initWebSocket();
    this.initRedisSubscription();
  }

  private initWebSocket() {
    this.wss.on('connection', (ws: WebSocket, req: IncomingMessage) => {
      this.handleConnection(ws, req);
    });
  }

  private async handleConnection(ws: WebSocket, req: IncomingMessage) {
    // 从 URL 解析用户 ID
    const url = new URL(req.url!, `http://${req.headers.host}`);
    const token = url.searchParams.get('token');
    
    // 验证 token
    const userId = await this.verifyToken(token);
    if (!userId) {
      ws.close(1008, 'Unauthorized');
      return;
    }

    // 保存连接
    if (!this.connections.has(userId)) {
      this.connections.set(userId, new Set());
    }
    this.connections.get(userId)!.add(ws);

    console.log(`User ${userId} connected`);

    // 订阅用户频道
    await this.redisSub.subscribe(`user:${userId}`);

    // 处理消息
    ws.on('message', async (data: string) => {
      try {
        const message = JSON.parse(data);
        await this.handleMessage(userId, message);
      } catch (error) {
        console.error('Message handling error:', error);
      }
    });

    // 处理断开
    ws.on('close', () => {
      this.connections.get(userId)?.delete(ws);
      if (this.connections.get(userId)?.size === 0) {
        this.connections.delete(userId);
        this.redisSub.unsubscribe(`user:${userId}`);
      }
      console.log(`User ${userId} disconnected`);
    });

    // 发送离线消息
    await this.sendOfflineMessages(userId);
  }

  private initRedisSubscription() {
    this.redisSub.on('message', (channel: string, message: string) => {
      // channel: user:123
      const userId = parseInt(channel.split(':')[1]);
      const connections = this.connections.get(userId);

      if (connections) {
        const data = JSON.parse(message);
        connections.forEach(ws => {
          if (ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(data));
          }
        });
      }
    });
  }

  private async handleMessage(fromUserId: number, message: any) {
    const { type, to, content, groupId } = message;

    switch (type) {
      case 'text':
        if (groupId) {
          await this.sendGroupMessage(fromUserId, groupId, content);
        } else {
          await this.sendPrivateMessage(fromUserId, to, content);
        }
        break;
      
      case 'read':
        await this.markAsRead(fromUserId, message.messageId);
        break;
    }
  }

  // 发送私聊消息
  private async sendPrivateMessage(
    fromUserId: number,
    toUserId: number,
    content: string
  ) {
    // 保存到数据库
    const message = await prisma.message.create({
      data: {
        fromUserId,
        toUserId,
        content,
        type: 'private',
        status: 'sent',
        createdAt: new Date()
      }
    });

    // 通过 Redis 发布消息
    const payload = {
      type: 'message',
      data: {
        id: message.id,
        from: fromUserId,
        content,
        timestamp: message.createdAt
      }
    };

    // 发给发送者（多端同步）
    await this.redis.publish(`user:${fromUserId}`, JSON.stringify(payload));
    
    // 发给接收者
    await this.redis.publish(`user:${toUserId}`, JSON.stringify(payload));
  }

  // 发送群聊消息
  private async sendGroupMessage(
    fromUserId: number,
    groupId: number,
    content: string
  ) {
    // 保存到数据库
    const message = await prisma.message.create({
      data: {
        fromUserId,
        groupId,
        content,
        type: 'group',
        status: 'sent',
        createdAt: new Date()
      }
    });

    // 获取群成员
    const members = await prisma.groupMember.findMany({
      where: { groupId },
      select: { userId: true }
    });

    const payload = {
      type: 'message',
      data: {
        id: message.id,
        from: fromUserId,
        groupId,
        content,
        timestamp: message.createdAt
      }
    };

    // 发给所有群成员
    await Promise.all(
      members.map(member =>
        this.redis.publish(`user:${member.userId}`, JSON.stringify(payload))
      )
    );
  }

  // 标记已读
  private async markAsRead(userId: number, messageId: number) {
    await prisma.message.update({
      where: { id: messageId, toUserId: userId },
      data: { status: 'read', readAt: new Date() }
    });
  }

  // 发送离线消息
  private async sendOfflineMessages(userId: number) {
    const messages = await prisma.message.findMany({
      where: {
        toUserId: userId,
        status: 'sent'
      },
      orderBy: { createdAt: 'asc' },
      take: 100
    });

    const connections = this.connections.get(userId);
    if (!connections) return;

    for (const message of messages) {
      const payload = {
        type: 'message',
        data: {
          id: message.id,
          from: message.fromUserId,
          content: message.content,
          timestamp: message.createdAt
        }
      };

      connections.forEach(ws => {
        if (ws.readyState === WebSocket.OPEN) {
          ws.send(JSON.stringify(payload));
        }
      });
    }
  }

  private async verifyToken(token: string | null): Promise<number | null> {
    if (!token) return null;
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      return decoded.userId;
    } catch {
      return null;
    }
  }
}

// 启动
const imServer = new IMServer(8080);
```

---

## 新闻推送系统

### 需求分析

类似今日头条的个性化推送系统。

**需求**：
- 关注模式（推模式）
- 推荐模式（拉模式）
- 实时性
- 去重

### 架构设计（推拉结合）

```
发布 → 写入数据库 → 消息队列 → Fan-out 服务 → Redis Timeline
                                                      ↓
用户 ← API 服务 ← 推荐引擎 ← Redis Timeline + 数据库
```

### 实现

```typescript
class FeedSystem {
  private redis: Redis;
  private prisma: PrismaClient;

  constructor() {
    this.redis = new Redis();
    this.prisma = new PrismaClient();
  }

  // 发布内容
  async publish(userId: number, content: string): Promise<number> {
    // 1. 保存到数据库
    const post = await this.prisma.post.create({
      data: {
        userId,
        content,
        createdAt: new Date()
      }
    });

    // 2. Fan-out 到粉丝的 Timeline
    await this.fanout(userId, post.id);

    return post.id;
  }

  // Fan-out：推送到所有粉丝的 Timeline
  private async fanout(userId: number, postId: number) {
    // 获取粉丝列表
    const followers = await this.prisma.follow.findMany({
      where: { followingId: userId },
      select: { followerId: true }
    });

    // 批量写入 Redis
    const pipeline = this.redis.pipeline();
    
    for (const follower of followers) {
      const key = `timeline:${follower.followerId}`;
      
      // 添加到 Timeline（Sorted Set）
      pipeline.zadd(key, Date.now(), `post:${postId}`);
      
      // 只保留最近 1000 条
      pipeline.zremrangebyrank(key, 0, -1001);
    }

    await pipeline.exec();
  }

  // 获取 Timeline（推模式）
  async getTimeline(userId: number, page = 1, pageSize = 20): Promise<Post[]> {
    const offset = (page - 1) * pageSize;
    const key = `timeline:${userId}`;

    // 从 Redis 获取 Post ID
    const postIds = await this.redis.zrevrange(key, offset, offset + pageSize - 1);

    if (postIds.length === 0) {
      return [];
    }

    // 批量查询 Post
    const posts = await this.prisma.post.findMany({
      where: {
        id: { in: postIds.map(id => parseInt(id.replace('post:', ''))) }
      },
      include: {
        user: {
          select: { id: true, name: true, avatar: true }
        }
      }
    });

    return posts;
  }

  // 获取推荐内容（拉模式）
  async getRecommendations(userId: number, page = 1, pageSize = 20): Promise<Post[]> {
    // 1. 从推荐服务获取推荐 Post ID
    const recommendedIds = await this.getRecommendedPostIds(userId, page, pageSize);

    // 2. 查询 Post
    const posts = await this.prisma.post.findMany({
      where: { id: { in: recommendedIds } },
      include: {
        user: {
          select: { id: true, name: true, avatar: true }
        }
      }
    });

    return posts;
  }

  // 混合模式：Timeline + 推荐
  async getFeed(userId: number, page = 1, pageSize = 20): Promise<Post[]> {
    // 70% Timeline + 30% 推荐
    const timelineCount = Math.floor(pageSize * 0.7);
    const recommendCount = pageSize - timelineCount;

    const [timeline, recommended] = await Promise.all([
      this.getTimeline(userId, page, timelineCount),
      this.getRecommendations(userId, page, recommendCount)
    ]);

    // 混合并去重
    const postMap = new Map<number, Post>();
    
    for (const post of [...timeline, ...recommended]) {
      if (!postMap.has(post.id)) {
        postMap.set(post.id, post);
      }
    }

    return Array.from(postMap.values()).slice(0, pageSize);
  }

  private async getRecommendedPostIds(
    userId: number,
    page: number,
    count: number
  ): Promise<number[]> {
    // 简化版：返回热门内容
    // 实际应该调用推荐引擎
    const posts = await this.prisma.post.findMany({
      orderBy: [
        { likes: 'desc' },
        { createdAt: 'desc' }
      ],
      skip: (page - 1) * count,
      take: count,
      select: { id: true }
    });

    return posts.map(p => p.id);
  }
}
```

---

## 系统设计面试技巧

### 面试流程

```
1. 需求澄清 (5分钟)
   - 功能需求
   - 非功能需求（性能、可用性）
   - 规模估算

2. 高层设计 (10-15分钟)
   - 画架构图
   - 说明组件职责
   - 数据流

3. 详细设计 (20-25分钟)
   - API 设计
   - 数据模型
   - 核心算法
   - 扩展性

4. 深入讨论 (10分钟)
   - 性能优化
   - 容错处理
   - 权衡取舍
```

### 需求澄清模板

```
Q: 每天多少用户？每秒多少请求？
Q: 读写比例？
Q: 数据量级？需要存储多久？
Q: 延迟要求？（实时 / 秒级 / 分钟级）
Q: 可用性要求？（99.9% / 99.99%）
Q: 一致性要求？（强一致 / 最终一致）
Q: 需要支持哪些平台？
```

### 容量估算示例

```
场景：短视频 App

DAU：5000 万
每人每天观看：20 个视频
每个视频：2MB

计算：
- QPS：5000万 * 20 / 86400 ≈ 11,500 QPS
- 峰值 QPS：11,500 * 3 = 34,500 QPS
- 存储：5000万 * 20 * 2MB = 2PB/天
- 带宽：34,500 * 2MB = 69GB/s
```

### 架构图模板

```
┌─────────────────────────────────────────────┐
│              Client (Web/Mobile)            │
└───────────────────┬─────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
     [CDN]      [WAF]      [Load Balancer]
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
  [API Gateway]        [WebSocket]          [Job Queue]
        │                     │                     │
        ├─────────────────────┼─────────────────────┤
        │                     │                     │
        ▼                     ▼                     ▼
   [Service A]           [Service B]          [Service C]
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
    [Redis]             [PostgreSQL]          [MongoDB]
                        (Primary + Replicas)
```

### 常见权衡

| 权衡 | 方案 A | 方案 B | 选择建议 |
|------|--------|--------|---------|
| **一致性 vs 性能** | 强一致 | 最终一致 | 根据业务需求 |
| **SQL vs NoSQL** | 关系型 | 文档型 | 关系复杂→SQL，灵活→NoSQL |
| **同步 vs 异步** | 实时响应 | 异步处理 | 实时性要求高→同步 |
| **规范化 vs 冗余** | 3NF | 反范式 | 读多→冗余，写多→规范化 |
| **微服务 vs 单体** | 分布式 | 集中式 | 团队大、业务复杂→微服务 |

---

## 总结

### 系统设计核心要点

1. **需求分析**
   - 功能需求
   - 非功能需求
   - 容量估算

2. **架构设计**
   - 分层架构
   - 服务拆分
   - 数据流向

3. **关键技术**
   - 缓存（Redis）
   - 消息队列（Kafka）
   - 负载均衡
   - 数据库优化

4. **高可用**
   - 无单点故障
   - 降级熔断
   - 限流保护

5. **扩展性**
   - 水平扩展
   - 分库分表
   - 读写分离

### 学习建议

1. **多画架构图**：培养系统思维
2. **动手实现**：理解细节和坑
3. **分析开源项目**：学习最佳实践
4. **模拟面试**：训练表达能力
5. **关注权衡**：没有银弹，只有取舍

---

**上一篇**：[数据库设计](./04-database-design.md)
**返回目录**：[README](./README.md)

