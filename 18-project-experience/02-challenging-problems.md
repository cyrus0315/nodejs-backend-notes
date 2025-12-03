# 面试必杀题：最有挑战的问题 & 最难的 Bug

## 目录

- [一、回答策略](#一回答策略)
- [二、系统设计类挑战](#二系统设计类挑战)
- [三、内存与性能类 Bug](#三内存与性能类-bug)
- [四、数据库类 Bug](#四数据库类-bug)
- [五、缓存类故障](#五缓存类故障)
- [六、消息队列类问题](#六消息队列类问题)
- [七、分布式系统类挑战](#七分布式系统类挑战)
- [八、数据精度与边界类 Bug](#八数据精度与边界类-bug)
- [九、网络与协议类问题](#九网络与协议类问题)
- [十、并发与幂等类问题](#十并发与幂等类问题)
- [十一、运维与稳定性类](#十一运维与稳定性类)
- [十二、场景选择指南](#十二场景选择指南)

---

## 一、回答策略

### 1.1 STAR 法则

无论选择哪个故事，都用这个结构：

```
S - Situation（背景）：简述业务场景和技术环境（15秒）
T - Task（任务）：什么问题？影响多大？（15秒）
A - Action（行动）：排查过程 + 解决方案 + 为什么这样设计（90秒）
R - Result（结果）：量化成果 + 沉淀了什么规范（15秒）
```

### 1.2 回答要点

1. **有数据支撑**：响应时间降低 60%、错误率从 5% 降到 0.01%
2. **有技术深度**：不只说"怎么做"，还要说"为什么这样做"
3. **有复盘思考**：这个问题教会了你什么？沉淀了什么规范？
4. **准备追问**：面试官一定会深挖细节

### 1.3 不同岗位侧重

| 岗位 | 推荐类型 | 示例 |
|------|---------|------|
| 高级工程师 | 技术深度型 | 内存泄漏、Event Loop 阻塞 |
| 架构师 | 系统设计型 | WFQ+DAG 调度、分布式事务 |
| Tech Lead | 故障处理型 | 缓存雪崩、支付可靠性 |
| SRE/DevOps | 稳定性保障型 | 日志爆盘、熔断降级 |

---

## 二、系统设计类挑战

### 2.1 WFQ + DAG 双层调度（⭐⭐⭐⭐⭐ 首选）

#### 问题背景

```
多租户 AI 平台，不同等级租户（VIP、标准、免费）同时提交任务
AI 任务执行时间长（10s-2min），且有依赖关系（抠图→生成→合成）

挑战：
1. VIP 要优先，但免费用户不能饿死
2. 任务有依赖，需要按序执行
3. GPU 是稀缺资源，要最大化利用率
```

#### 面试话术

> 我在做 AI 电商平台时，遇到的最有挑战的问题是**多租户场景下 AI 任务的公平调度与资源隔离**。
>
> **问题的复杂性**在于：AI 任务执行时间长（10秒到2分钟），而且有依赖关系。我们需要同时满足：VIP 租户优先保障、普通租户不被饿死、系统吞吐量最大化。传统的优先级队列行不通——VIP 请求多的时候，免费用户可能永远排不上。
>
> 我设计了**双层调度架构**：
>
> **第一层是 WFQ（加权公平队列）**，解决租户间资源分配。借鉴网络 QoS 的思想，通过虚拟时间戳机制，权重越高的租户 VFT 增长越慢，因此获得更多调度机会。数学上可以证明，长期资源分配比例精确等于权重比。
>
> **第二层是 DAG 调度器**，解决单个请求内的任务依赖。用拓扑排序管理依赖关系，入度为 0 的任务才能执行，实现最大并行度。
>
> 两者的协作方式是：请求到达时构建 DAG，将入口任务推送到 WFQ；WFQ 调度任务执行；任务完成后更新 DAG 入度，新就绪的任务再推回 WFQ。
>
> 上线后，VIP 租户的平均响应时间降低了 60%，同时普通租户的任务也能在合理时间内完成。

#### 追问应对

| 追问 | 回答要点 |
|------|---------|
| WFQ 和简单优先级有什么区别？ | 优先级队列会饿死低优先级；WFQ 通过虚拟时间保证比例公平 |
| 虚拟时间怎么理解？ | VFT = 成本/权重，权重高的 VFT 增长慢，排名靠前 |
| DAG 任务失败怎么处理？ | 级联取消所有下游依赖任务，同时支持重试策略 |
| 分布式环境下怎么同步状态？ | 队列存在 Redis，用 Lua 脚本保证原子性 |

#### 核心代码

```typescript
interface Task {
  id: string;
  tenantId: string;
  cost: number;
  virtualFinishTime: number;
}

class WFQScheduler {
  private globalVirtualTime = 0;
  private tenantStates = new Map<string, { weight: number; lastVFT: number }>();
  private taskQueue: Task[] = [];

  enqueue(task: Task): void {
    const tenant = this.tenantStates.get(task.tenantId)!;
    
    // 核心公式
    const startTime = Math.max(this.globalVirtualTime, tenant.lastVFT);
    task.virtualFinishTime = startTime + task.cost / tenant.weight;
    tenant.lastVFT = task.virtualFinishTime;
    
    // 按 VFT 排序插入
    this.insertSorted(task);
  }

  dequeue(): Task | null {
    if (this.taskQueue.length === 0) return null;
    const task = this.taskQueue.shift()!;
    this.globalVirtualTime = task.virtualFinishTime;
    return task;
  }
}
```

---

### 2.2 分布式事务：Saga 模式

#### 问题背景

```
微服务架构，创建订单需要：
1. 库存服务扣库存
2. 钱包服务扣余额
3. 订单服务创建订单

任何一步失败，数据就不一致
```

#### 面试话术

> 我们是微服务架构，创建订单需要调用三个服务。有次库存扣了、余额扣了，但订单服务超时了，重试又创建了重复订单。
>
> **解决方案是 Saga 模式 + 本地消息表**：
>
> 1. 订单服务先写"待确认"订单 + 本地消息表（同一个事务）
> 2. 异步发消息给库存服务扣库存
> 3. 库存成功后发消息给钱包服务扣余额
> 4. 全部成功后，订单状态改为"已确认"
> 5. 任何一步失败，执行补偿事务（加库存、加余额）
>
> **关键设计**：
> - 本地消息表保证"发消息"和"业务操作"原子性
> - 每个服务都要实现正向操作和补偿操作
> - 消息必须幂等

#### 核心代码

```typescript
// 本地消息表
model OutboxMessage {
  id          String   @id
  aggregateId String   // 订单ID
  eventType   String   // ORDER_CREATED, STOCK_DEDUCTED
  payload     Json
  status      String   // PENDING, SENT, PROCESSED
  createdAt   DateTime
}

// 创建订单（本地事务）
async function createOrder(data: OrderData) {
  await prisma.$transaction([
    // 1. 创建待确认订单
    prisma.order.create({ 
      data: { ...data, status: 'PENDING' } 
    }),
    // 2. 写本地消息表
    prisma.outboxMessage.create({
      data: {
        aggregateId: orderId,
        eventType: 'ORDER_CREATED',
        payload: data,
        status: 'PENDING'
      }
    })
  ]);
}

// 定时任务：发送待处理消息
@Cron('*/5 * * * * *')
async function processOutbox() {
  const messages = await prisma.outboxMessage.findMany({
    where: { status: 'PENDING' }
  });
  
  for (const msg of messages) {
    await kafka.send(msg.eventType, msg.payload);
    await prisma.outboxMessage.update({
      where: { id: msg.id },
      data: { status: 'SENT' }
    });
  }
}
```

---

## 三、内存与性能类 Bug

### 3.1 Node.js 内存泄漏（⭐⭐⭐⭐⭐）

#### 问题现象

```
服务运行 2-3 天后必崩，重启后恢复，循环往复
K8s 显示 OOM Killed
```

#### 面试话术

> 我遇到过一个隐蔽的内存泄漏问题。线上 Node.js 服务每隔 2-3 天就会 OOM 被 K8s 杀掉重启。
>
> **排查过程**：
> 1. 先用 `process.memoryUsage()` 确认内存确实在持续增长
> 2. 用 `heapdump` 模块在内存增长时自动 dump 堆快照
> 3. 对比两个时间点的快照，发现 `Closure` 类型对象持续增长
> 4. 追踪到一个 EventEmitter 的监听器没有正确移除
>
> **根因**：一个中间件在每次请求时给全局 EventEmitter 添加监听器，但请求结束后没移除，导致闭包持有请求上下文，GC 无法回收。
>
> **解决**：把 `on` 改成 `once`，或在请求结束时显式 `removeListener`。
>
> **复盘**：这让我养成习惯——用 `process.memoryUsage()` 做监控，CI 里加内存增长测试。

#### 问题代码 vs 修复代码

```typescript
// ❌ 问题代码：每次请求都添加监听器，但从不移除
app.use((req, res, next) => {
  globalEmitter.on('event', (data) => {
    // 闭包持有 req, res
    console.log(req.url, data);
  });
  next();
});

// ✅ 修复方案1：使用 once
app.use((req, res, next) => {
  globalEmitter.once('event', (data) => {
    console.log(req.url, data);
  });
  next();
});

// ✅ 修复方案2：请求结束时移除
app.use((req, res, next) => {
  const handler = (data) => console.log(req.url, data);
  globalEmitter.on('event', handler);
  
  res.on('finish', () => {
    globalEmitter.removeListener('event', handler);
  });
  next();
});
```

#### 排查工具

```typescript
// 1. 监控内存使用
setInterval(() => {
  const used = process.memoryUsage();
  console.log({
    rss: `${Math.round(used.rss / 1024 / 1024)}MB`,
    heapUsed: `${Math.round(used.heapUsed / 1024 / 1024)}MB`,
    heapTotal: `${Math.round(used.heapTotal / 1024 / 1024)}MB`
  });
}, 10000);

// 2. 自动 dump 堆快照
import heapdump from 'heapdump';

let lastHeapUsed = 0;
setInterval(() => {
  const { heapUsed } = process.memoryUsage();
  // 内存增长超过 100MB 时 dump
  if (heapUsed - lastHeapUsed > 100 * 1024 * 1024) {
    heapdump.writeSnapshot(`./heap-${Date.now()}.heapsnapshot`);
    lastHeapUsed = heapUsed;
  }
}, 60000);
```

---

### 3.2 Event Loop 阻塞（⭐⭐⭐⭐⭐）

#### 问题现象

```
99% 请求正常，P50 = 50ms
但偶尔有请求卡 30 秒才返回，P99 飙到 30s
```

#### 面试话术

> 有段时间线上告警频繁：接口 P99 延迟飙到 30 秒，但 P50 只有 50ms，非常诡异。
>
> **排查过程**：
> 1. APM 看调用链，数据库、Redis 都正常
> 2. 怀疑是 Event Loop 被阻塞，用 `blocked-at` 库定位
> 3. 发现是一个 `JSON.parse()` 在处理超大 JSON（10MB+）时阻塞了主线程
> 4. 这个 JSON 来自第三方回调，正常情况 10KB，偶尔会有 10MB 的异常数据
>
> **解决方案**：
> 1. 短期：加请求体大小限制，超过 1MB 直接拒绝
> 2. 长期：大 JSON 解析改用流式处理或 Worker Threads
>
> **收获**：Node.js 单线程模型下，任何 CPU 密集操作都是定时炸弹。后来我在代码规范里加了一条：**超过 1ms 的同步操作必须 Review**。

#### 排查与修复

```typescript
// 排查：使用 blocked-at 定位阻塞代码
import blocked from 'blocked-at';

blocked((time, stack) => {
  console.log(`Event loop blocked for ${time}ms`);
  console.log(stack);
}, { threshold: 100 }); // 超过 100ms 就报警

// ❌ 问题代码
app.post('/webhook', (req, res) => {
  const data = JSON.parse(req.body); // 大 JSON 阻塞
  // ...
});

// ✅ 修复方案1：限制请求体大小
app.use(express.json({ limit: '1mb' }));

// ✅ 修复方案2：Worker Threads 处理
import { Worker } from 'worker_threads';

async function parseJSONInWorker(jsonStr: string) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(`
      const { parentPort } = require('worker_threads');
      parentPort.on('message', (data) => {
        parentPort.postMessage(JSON.parse(data));
      });
    `, { eval: true });
    
    worker.postMessage(jsonStr);
    worker.on('message', resolve);
    worker.on('error', reject);
  });
}

// ✅ 修复方案3：流式解析
import { parser } from 'stream-json';
import { streamArray } from 'stream-json/streamers/StreamArray';

const pipeline = fs.createReadStream('large.json')
  .pipe(parser())
  .pipe(streamArray());

pipeline.on('data', ({ value }) => {
  // 处理每一项
});
```

---

### 3.3 正则表达式 ReDoS（⭐⭐⭐⭐）

#### 问题现象

```
有人提交了一个特殊字符串，服务直接卡住 30 秒不响应
CPU 100%
```

#### 面试话术

> 有个表单校验邮箱的正则，有人输入了 `aaaaaaaaaaaaaaaaaaaaaaaaaaaa!`，CPU 直接 100%，30 秒后才返回。
>
> **原因**：正则里的 `([a-zA-Z0-9]+)+` 嵌套量词导致**灾难性回溯**，时间复杂度指数级增长。
>
> **解决**：
> 1. 用 `safe-regex` 库检测危险正则
> 2. 重写正则，避免嵌套量词
> 3. 加输入长度限制
>
> **CI 集成**：lint 规则扫描所有正则，发现危险模式直接报错。

#### 问题代码 vs 修复代码

```typescript
// ❌ 危险正则：嵌套量词导致灾难性回溯
const emailRegex = /^([a-zA-Z0-9]+)+@[a-zA-Z0-9]+\.[a-zA-Z]+$/;

// 测试
console.time('regex');
emailRegex.test('aaaaaaaaaaaaaaaaaaaaaaaaaaaa!'); // 30秒+
console.timeEnd('regex');

// ✅ 安全正则：移除嵌套量词
const safeEmailRegex = /^[a-zA-Z0-9]+@[a-zA-Z0-9]+\.[a-zA-Z]+$/;

// 防护措施
import safeRegex from 'safe-regex';

function validateRegex(pattern: RegExp) {
  if (!safeRegex(pattern)) {
    throw new Error('Unsafe regex pattern detected');
  }
}

// 限制输入长度
function validateEmail(email: string) {
  if (email.length > 254) {  // RFC 5321
    return false;
  }
  return safeEmailRegex.test(email);
}
```

---

## 四、数据库类 Bug

### 4.1 数据库死锁（⭐⭐⭐⭐）

#### 问题现象

```
秒杀活动期间，数据库频繁死锁告警
大量请求失败，用户投诉下单失败
```

#### 面试话术

> 秒杀活动期间，数据库疯狂告警死锁，查 `pg_stat_activity` 发现两个事务互相等待。
>
> **根因分析**：
> - 事务 A：先锁订单表 row1，再锁库存表 row1
> - 事务 B：先锁库存表 row1，再锁订单表 row1
> - 经典的锁顺序不一致导致死锁
>
> **解决**：
> 1. **统一加锁顺序**：所有事务按"库存→订单→支付"顺序加锁
> 2. **减小事务粒度**：拆分大事务，减少锁持有时间
> 3. **使用乐观锁**：库存扣减改用 `UPDATE ... WHERE stock >= amount`
> 4. **预扣库存**：用 Redis 做库存预扣，数据库只做最终确认

#### 代码示例

```typescript
// ❌ 问题代码：锁顺序不一致
// 事务A
await prisma.$transaction([
  prisma.order.update({ where: { id: orderId }, data: {...} }),
  prisma.stock.update({ where: { id: stockId }, data: {...} })
]);

// 事务B
await prisma.$transaction([
  prisma.stock.update({ where: { id: stockId }, data: {...} }),
  prisma.order.update({ where: { id: orderId }, data: {...} })
]);

// ✅ 修复方案1：统一加锁顺序
// 所有事务都按 stock -> order 顺序
await prisma.$transaction([
  prisma.stock.update({ where: { id: stockId }, data: {...} }),
  prisma.order.update({ where: { id: orderId }, data: {...} })
]);

// ✅ 修复方案2：乐观锁
const result = await prisma.stock.updateMany({
  where: {
    id: stockId,
    quantity: { gte: amount }  // 乐观锁条件
  },
  data: {
    quantity: { decrement: amount }
  }
});

if (result.count === 0) {
  throw new Error('库存不足');
}

// ✅ 修复方案3：Redis 预扣
async function deductStockWithRedis(productId: string, amount: number) {
  const key = `stock:${productId}`;
  const remaining = await redis.decrby(key, amount);
  
  if (remaining < 0) {
    await redis.incrby(key, amount); // 回滚
    throw new Error('库存不足');
  }
  
  // 异步同步到数据库
  await queue.add('sync-stock', { productId, amount });
}
```

---

### 4.2 连接池泄漏（⭐⭐⭐⭐）

#### 问题现象

```
服务跑着跑着就报 "connection pool exhausted"
重启后又好了，过几小时又挂
```

#### 面试话术

> 服务运行几小时后开始报错：无法获取数据库连接。查 `pg_stat_activity`，发现有大量 idle 连接。
>
> **排查发现**：代码中有些分支提前 return 了，没有释放连接。
>
> **解决**：用 `try-finally` 保证释放，或者更好的方案是用 ORM 自动管理连接。

#### 问题代码 vs 修复代码

```typescript
// ❌ 问题代码：提前返回没释放
async function getData() {
  const client = await pool.connect();
  const result = await client.query('SELECT ...');
  if (result.rows.length === 0) {
    return null;  // 💥 提前返回，没有 release！
  }
  client.release();
  return result.rows;
}

// ✅ 修复方案1：try-finally
async function getData() {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT ...');
    return result.rows.length ? result.rows : null;
  } finally {
    client.release();  // 无论如何都释放
  }
}

// ✅ 修复方案2：使用 pool.query（自动管理）
async function getData() {
  const result = await pool.query('SELECT ...');
  return result.rows.length ? result.rows : null;
}

// ✅ 修复方案3：使用 ORM
async function getData() {
  return await prisma.user.findMany();  // Prisma 自动管理连接
}

// 监控连接池状态
setInterval(() => {
  console.log({
    total: pool.totalCount,
    idle: pool.idleCount,
    waiting: pool.waitingCount
  });
}, 10000);
```

---

### 4.3 慢 SQL 锁表（⭐⭐⭐）

#### 问题现象

```
有人执行了一个没 WHERE 的 UPDATE
锁了 5 分钟，所有业务瘫痪
```

#### 面试话术

> 运营同学在后台执行 `UPDATE users SET status = 'active'`，没加 WHERE！
>
> 这是一张 500 万行的表，全表更新锁了 5 分钟，期间所有涉及 users 表的请求全部超时。
>
> **解决**：
> 1. 数据库权限分离：普通用户不能执行没 WHERE 的 UPDATE/DELETE
> 2. SQL 审计工具：上线 SQL 要经过审批
> 3. 分批更新

#### 分批更新示例

```typescript
// ❌ 危险：全表更新
await prisma.$executeRaw`UPDATE users SET status = 'active'`;

// ✅ 安全：分批更新
async function batchUpdate(batchSize = 1000) {
  let updated = 0;
  
  while (true) {
    const result = await prisma.$executeRaw`
      UPDATE users 
      SET status = 'active' 
      WHERE id IN (
        SELECT id FROM users 
        WHERE status != 'active' 
        LIMIT ${batchSize}
      )
    `;
    
    if (result === 0) break;
    updated += result;
    
    // 每批之间休息一下，给其他查询机会
    await sleep(100);
    console.log(`Updated ${updated} rows`);
  }
  
  return updated;
}

// PostgreSQL 权限控制
-- CREATE ROLE readonly;
-- GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- CREATE ROLE readwrite;
-- GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
-- 禁止没有 WHERE 的 UPDATE/DELETE（需要通过应用层或触发器实现）
```

---

## 五、缓存类故障

### 5.1 缓存雪崩（⭐⭐⭐⭐⭐）

#### 问题现象

```
Redis 主节点宕机，哨兵切换花了 30 秒
这 30 秒内所有请求打到数据库
数据库连接池耗尽，全站 502
```

#### 面试话术

> 有次 Redis 主节点意外宕机，哨兵切换花了 30 秒，这 30 秒内所有缓存请求都 fallback 到了 PostgreSQL。
>
> 结果数据库连接池瞬间被打满，查询排队，上游超时重试，形成雪崩，5 分钟内全站挂了。
>
> **事后复盘**：
> 1. **二级缓存**：加本地缓存做兜底
> 2. **熔断器**：Redis 失败率超阈值直接熔断，返回降级数据
> 3. **连接池隔离**：核心接口和非核心接口用不同的数据库连接池
> 4. **预热机制**：服务启动时主动加载热点数据

#### 代码示例

```typescript
import CircuitBreaker from 'opossum';
import NodeCache from 'node-cache';

// 本地缓存（二级缓存）
const localCache = new NodeCache({ stdTTL: 60 });

// 熔断器
const redisBreaker = new CircuitBreaker(
  async (key: string) => redis.get(key),
  {
    timeout: 3000,           // 3秒超时
    errorThresholdPercentage: 50,  // 50% 错误率触发熔断
    resetTimeout: 30000      // 30秒后尝试恢复
  }
);

// 降级策略
redisBreaker.fallback((key: string) => {
  console.warn(`Redis circuit open, falling back for key: ${key}`);
  return localCache.get(key) || null;
});

// 多级缓存读取
async function getWithCache(key: string, fetchFn: () => Promise<any>) {
  // 1. 先查本地缓存
  const local = localCache.get(key);
  if (local) return local;

  // 2. 查 Redis（带熔断）
  try {
    const cached = await redisBreaker.fire(key);
    if (cached) {
      localCache.set(key, cached);
      return cached;
    }
  } catch (err) {
    console.error('Redis error:', err);
  }

  // 3. 查数据库
  const data = await fetchFn();
  
  // 4. 写入缓存（异步，不阻塞响应）
  setImmediate(async () => {
    localCache.set(key, data);
    try {
      await redis.setex(key, 3600, JSON.stringify(data));
    } catch (err) {
      console.error('Redis write error:', err);
    }
  });

  return data;
}
```

---

### 5.2 热点 Key 压垮单节点（⭐⭐⭐⭐）

#### 问题现象

```
秒杀活动，Redis 集群某个节点 CPU 100%
其他节点很闲，无法水平扩展
```

#### 面试话术

> 秒杀库存放在 Redis 里：`stock:activity:123`
>
> 所有请求都打这一个 Key，而 Redis Cluster 是按 Key hash 分片的，这个 Key 只落在一个节点上。
>
> 结果：一个节点 CPU 100%，其他节点空闲。
>
> **解决：Key 分片**，把一个 Key 拆成多个，分散到不同节点。

#### 代码示例

```typescript
class ShardedCounter {
  private shardCount = 10;
  private keyPrefix: string;

  constructor(keyPrefix: string, shardCount = 10) {
    this.keyPrefix = keyPrefix;
    this.shardCount = shardCount;
  }

  // 扣减（随机选一个分片）
  async decrement(amount = 1): Promise<boolean> {
    const shard = Math.floor(Math.random() * this.shardCount);
    const key = `${this.keyPrefix}:shard:${shard}`;
    
    const result = await redis.decrby(key, amount);
    if (result < 0) {
      // 这个分片不够了，尝试其他分片
      await redis.incrby(key, amount);  // 回滚
      return this.decrementFromOtherShards(shard, amount);
    }
    return true;
  }

  // 获取总数
  async getTotal(): Promise<number> {
    const keys = Array(this.shardCount)
      .fill(0)
      .map((_, i) => `${this.keyPrefix}:shard:${i}`);
    
    const values = await redis.mget(keys);
    return values.reduce((sum, val) => sum + (Number(val) || 0), 0);
  }

  // 初始化库存
  async init(totalStock: number): Promise<void> {
    const perShard = Math.floor(totalStock / this.shardCount);
    const remainder = totalStock % this.shardCount;
    
    const pipeline = redis.pipeline();
    for (let i = 0; i < this.shardCount; i++) {
      const stock = perShard + (i < remainder ? 1 : 0);
      pipeline.set(`${this.keyPrefix}:shard:${i}`, stock);
    }
    await pipeline.exec();
  }
}

// 使用
const stockCounter = new ShardedCounter('stock:activity:123');
await stockCounter.init(10000);  // 初始化 1 万库存
await stockCounter.decrement();   // 扣库存
const remaining = await stockCounter.getTotal();  // 查总库存
```

---

## 六、消息队列类问题

### 6.1 Kafka 消息丢失（⭐⭐⭐⭐⭐）

#### 问题现象

```
订单消息偶尔丢失
用户付了钱但订单状态没更新
```

#### 面试话术

> 我们用 Kafka 处理订单状态变更。有用户投诉付款后订单没变化，查日志发现消息确实发了，但消费者没处理。
>
> **排查发现**：
> - 消费者配置的是 `autoCommit: true`
> - 收到消息后自动提交 offset，然后开始处理业务逻辑
> - 业务逻辑执行到一半服务重启了，offset 已提交，但消息没处理完
>
> **解决**：改成手动提交 offset，业务逻辑成功后再 commit。但这引入新问题——重复消费，所以还要加幂等处理。

#### 代码示例

```typescript
// ❌ 问题配置：自动提交
const consumer = kafka.consumer({
  groupId: 'order-group',
  autoCommit: true,          // 自动提交
  autoCommitInterval: 5000   // 每 5 秒提交
});

// ✅ 修复：手动提交 + 幂等
const consumer = kafka.consumer({
  groupId: 'order-group',
  autoCommit: false  // 关闭自动提交
});

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const messageId = message.headers?.messageId?.toString();
    
    // 幂等检查
    const processed = await redis.get(`processed:${messageId}`);
    if (processed) {
      console.log(`Message ${messageId} already processed, skipping`);
      await commitOffset(topic, partition, message.offset);
      return;
    }

    try {
      // 处理业务逻辑
      await processOrder(JSON.parse(message.value!.toString()));
      
      // 标记已处理（设置过期时间，比如 7 天）
      await redis.setex(`processed:${messageId}`, 7 * 24 * 3600, '1');
      
      // 手动提交 offset
      await commitOffset(topic, partition, message.offset);
    } catch (err) {
      console.error('Process failed:', err);
      // 不提交 offset，消息会重新消费
    }
  }
});

async function commitOffset(topic: string, partition: number, offset: string) {
  await consumer.commitOffsets([{
    topic,
    partition,
    offset: (Number(offset) + 1).toString()
  }]);
}
```

---

### 6.2 消息重复消费（⭐⭐⭐⭐）

#### 问题现象

```
同一个订单被处理了两次
用户扣了两次钱
```

#### 面试话术

> 为了保证消息不丢失，我们改成了手动提交 offset。但有时候业务处理完了，提交 offset 之前服务重启了，消息就会被重新消费。
>
> **解决**：业务逻辑必须幂等。用唯一键（订单号/消息ID）做去重。

#### 幂等设计模式

```typescript
// 方案1：Redis 去重
async function processOrderIdempotent(orderId: string, handler: () => Promise<void>) {
  const lockKey = `order:process:${orderId}`;
  
  // 尝试获取锁（7天过期）
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 7 * 24 * 3600);
  if (!acquired) {
    console.log(`Order ${orderId} already processed`);
    return;
  }

  try {
    await handler();
  } catch (err) {
    // 处理失败，删除锁，允许重试
    await redis.del(lockKey);
    throw err;
  }
}

// 方案2：数据库唯一键
model OrderEvent {
  id        String   @id @default(uuid())
  orderId   String
  eventType String
  eventId   String   @unique  // 消息ID，唯一约束
  createdAt DateTime @default(now())
}

async function processOrderEvent(eventId: string, orderId: string, handler: () => Promise<void>) {
  try {
    await prisma.orderEvent.create({
      data: { eventId, orderId, eventType: 'PAYMENT' }
    });
  } catch (err) {
    if (err.code === 'P2002') {  // 唯一键冲突
      console.log(`Event ${eventId} already processed`);
      return;
    }
    throw err;
  }

  await handler();
}

// 方案3：状态机防重
async function updateOrderStatus(orderId: string, fromStatus: string, toStatus: string) {
  const result = await prisma.order.updateMany({
    where: {
      id: orderId,
      status: fromStatus  // 只有当前状态符合才更新
    },
    data: {
      status: toStatus
    }
  });

  if (result.count === 0) {
    console.log(`Order ${orderId} not in ${fromStatus} status, skipping`);
    return false;
  }
  return true;
}
```

---

## 七、分布式系统类挑战

### 7.1 分布式锁失效（⭐⭐⭐⭐）

#### 问题现象

```
同一任务被多个 Worker 同时执行
数据处理重复或冲突
```

#### 面试话术

> 我们用 Redis 分布式锁保证任务只被一个 Worker 执行。但有次发现同一个任务被两个 Worker 同时执行了。
>
> **排查发现**：
> - 锁的 TTL 是 30 秒
> - 任务实际执行了 35 秒
> - 锁过期后被另一个 Worker 抢到了
>
> **解决**：实现锁续期机制（看门狗），任务执行期间定期延长锁的过期时间。

#### 代码示例

```typescript
import Redlock from 'redlock';

class DistributedLock {
  private redlock: Redlock;

  constructor(redisClients: Redis[]) {
    this.redlock = new Redlock(redisClients, {
      retryCount: 10,
      retryDelay: 200,
      retryJitter: 200
    });
  }

  // 带看门狗的锁
  async acquireWithWatchdog<T>(
    resource: string,
    ttl: number,
    fn: () => Promise<T>
  ): Promise<T> {
    const lock = await this.redlock.acquire([resource], ttl);
    
    // 启动看门狗，定期续期
    const watchdog = setInterval(async () => {
      try {
        await lock.extend(ttl);
        console.log(`Extended lock for ${resource}`);
      } catch (error) {
        console.error(`Failed to extend lock for ${resource}:`, error);
        clearInterval(watchdog);
      }
    }, ttl / 3);  // 每 1/3 TTL 续期一次

    try {
      return await fn();
    } finally {
      clearInterval(watchdog);
      try {
        await lock.release();
      } catch (err) {
        console.error(`Failed to release lock for ${resource}:`, err);
      }
    }
  }
}

// 使用
const lock = new DistributedLock([redis]);

await lock.acquireWithWatchdog(
  `task:${taskId}`,
  30000,  // 30秒 TTL
  async () => {
    // 即使执行超过 30 秒也没问题，看门狗会续期
    await processLongRunningTask(taskId);
  }
);
```

---

### 7.2 点数并发扣减（⭐⭐⭐⭐⭐）

#### 问题现象

```
用户账户有 100 点
同时发起 3 个任务，每个消耗 50 点
结果 3 个任务都成功了，余额变成 -50
```

#### 面试话术

> 我遇到过一个隐蔽的并发超卖 bug，用户点数明明不够，但任务还是执行成功了，导致点数变成负数。
>
> **排查发现**：原先是先查余额 `if (balance >= cost)` 再扣减，典型的"检查-执行时间差"问题（TOCTOU）。
>
> **解决**：采用 TCC 模式 + 乐观锁。Try 阶段冻结点数，用原子操作保证检查和扣减在同一个操作里。

#### 代码示例

```typescript
// ❌ 问题代码：检查和执行不是原子的
async function deductPoints(userId: string, amount: number) {
  const user = await prisma.user.findUnique({ where: { userId } });
  
  if (user.points < amount) {  // 检查
    throw new Error('余额不足');
  }
  
  // 这里可能有其他请求也通过了检查
  
  await prisma.user.update({   // 执行
    where: { userId },
    data: { points: { decrement: amount } }
  });
}

// ✅ 修复：TCC + 乐观锁
class PointsService {
  
  // Try: 冻结点数
  async freeze(userId: string, amount: number, taskId: string): Promise<boolean> {
    const result = await prisma.userPoints.updateMany({
      where: {
        userId,
        balance: { gte: amount },  // 检查和更新在同一个原子操作
        version: currentVersion     // 乐观锁
      },
      data: {
        balance: { decrement: amount },
        frozen: { increment: amount },
        version: { increment: 1 }
      }
    });
    
    if (result.count === 0) {
      return false;  // 余额不足或版本冲突
    }
    
    // 记录冻结流水
    await prisma.pointsLog.create({
      data: { userId, taskId, amount, type: 'FREEZE' }
    });
    
    return true;
  }

  // Confirm: 确认扣除
  async confirm(userId: string, amount: number, taskId: string): Promise<void> {
    await prisma.userPoints.update({
      where: { userId },
      data: { frozen: { decrement: amount } }
    });
    
    await prisma.pointsLog.create({
      data: { userId, taskId, amount, type: 'CONFIRM' }
    });
  }

  // Cancel: 回滚
  async cancel(userId: string, amount: number, taskId: string): Promise<void> {
    await prisma.userPoints.update({
      where: { userId },
      data: {
        frozen: { decrement: amount },
        balance: { increment: amount }
      }
    });
    
    await prisma.pointsLog.create({
      data: { userId, taskId, amount, type: 'CANCEL' }
    });
  }
}
```

---

## 八、数据精度与边界类 Bug

### 8.1 浮点数精度问题（⭐⭐⭐⭐⭐）

#### 问题现象

```
用户充值 100 元，分 3 次消费 33.33 元
余额显示 0.01 元，但实际对账差了几万块
```

#### 面试话术

> 财务对账发现每天都有几分钱的差异，累计下来一个月差了好几万。
>
> **根因**：JavaScript 浮点数精度问题，`0.1 + 0.2 = 0.30000000000000004`
>
> **解决**：所有金额用整数存储（以"分"为单位），计算用 Decimal.js。

#### 代码示例

```typescript
// ❌ 问题代码
const price = 33.33;
const total = price * 3;  // 99.99000000000001

// ✅ 修复方案1：整数存储（以分为单位）
const priceInCents = 3333;  // 33.33 元
const totalInCents = priceInCents * 3;  // 9999 分 = 99.99 元

// 显示时转换
function formatMoney(cents: number): string {
  return (cents / 100).toFixed(2);  // "99.99"
}

// ✅ 修复方案2：使用 Decimal.js
import Decimal from 'decimal.js';

const price = new Decimal('33.33');
const total = price.times(3);  // Decimal { 99.99 }

// 比较
const a = new Decimal('0.1');
const b = new Decimal('0.2');
console.log(a.plus(b).toString());  // "0.3" ✅

// ✅ 数据库使用 DECIMAL 类型
model Order {
  id     String  @id
  amount Decimal @db.Decimal(10, 2)  // 10位数字，2位小数
}
```

---

### 8.2 时区问题（⭐⭐⭐⭐）

#### 问题现象

```
海外用户投诉订单时间不对
凌晨 1 点下单，显示昨天 17 点
```

#### 面试话术

> 东南亚用户反馈时间不对。**根因**：服务器时区是 UTC，前端直接显示数据库返回的时间，没做时区转换。
>
> **规范**：存储统一 UTC，传输用 ISO 8601，显示按用户时区转换。

#### 代码示例

```typescript
import dayjs from 'dayjs';
import utc from 'dayjs/plugin/utc';
import timezone from 'dayjs/plugin/timezone';

dayjs.extend(utc);
dayjs.extend(timezone);

// 存储：统一 UTC
const createdAt = dayjs.utc().toDate();

// 传输：ISO 8601 格式
const isoString = dayjs(createdAt).toISOString();
// "2024-01-15T08:00:00.000Z"

// 显示：转换为用户时区
function formatForUser(utcTime: Date, userTimezone: string): string {
  return dayjs(utcTime).tz(userTimezone).format('YYYY-MM-DD HH:mm:ss');
}

// 使用
formatForUser(createdAt, 'Asia/Shanghai');  // "2024-01-15 16:00:00"
formatForUser(createdAt, 'Asia/Singapore'); // "2024-01-15 16:00:00"
formatForUser(createdAt, 'America/New_York'); // "2024-01-15 03:00:00"

// 从请求中获取用户时区
@Get('/orders')
async getOrders(@Req() req: Request) {
  const userTimezone = req.headers['x-timezone'] || 'UTC';
  const orders = await this.orderService.findAll();
  
  return orders.map(order => ({
    ...order,
    createdAt: formatForUser(order.createdAt, userTimezone)
  }));
}
```

---

## 九、网络与协议类问题

### 9.1 WebSocket 重连风暴（⭐⭐⭐⭐）

#### 问题现象

```
网络抖动恢复后，几万个客户端同时重连
服务直接 OOM
```

#### 面试话术

> 我们有个实时推送服务，维护了 5 万个 WebSocket 长连接。网络抖动 10 秒后恢复，5 万客户端同时重连，服务 OOM 重启，客户端又重连，恶性循环。
>
> **解决**：客户端实现指数退避 + 随机抖动的重连策略。

#### 代码示例

```typescript
// 客户端重连逻辑
class WebSocketClient {
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private ws: WebSocket | null = null;

  connect() {
    this.ws = new WebSocket('wss://example.com/ws');
    
    this.ws.onopen = () => {
      this.reconnectAttempts = 0;  // 重置计数
    };

    this.ws.onclose = () => {
      this.reconnect();
    };
  }

  private reconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnect attempts reached');
      return;
    }

    // 指数退避 + 随机抖动
    const delay = Math.min(
      1000 * Math.pow(2, this.reconnectAttempts) +  // 指数: 1s, 2s, 4s, 8s...
      Math.random() * 1000,  // 随机抖动 0-1s
      30000  // 最大 30 秒
    );

    console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts + 1})`);
    
    setTimeout(() => {
      this.reconnectAttempts++;
      this.connect();
    }, delay);
  }
}

// 服务端保护
class WebSocketServer {
  private connectionQueue: Queue;
  private maxConcurrentHandshakes = 100;

  async handleConnection(socket: WebSocket) {
    // 连接数限流
    const currentConnections = await this.getConnectionCount();
    if (currentConnections >= this.maxConnections) {
      socket.close(1013, 'Server overloaded');
      return;
    }

    // 握手队列化
    await this.connectionQueue.add(async () => {
      await this.authenticateAndSetup(socket);
    });
  }
}
```

---

### 9.2 第三方超时拖垮全站（⭐⭐⭐⭐）

#### 问题现象

```
调用第三方 API 没设超时
第三方变慢后连接池耗尽，服务雪崩
```

#### 面试话术

> 有次第三方短信服务响应变慢（从 200ms 变成 60s），我们没设超时，导致 HTTP 连接被占满，内存飙升，最终 OOM。
>
> **解决**：所有外部调用必须有超时 + 重试 + 熔断。封装统一的 HTTP Client，强制超时配置。

#### 代码示例

```typescript
import axios, { AxiosInstance } from 'axios';
import axiosRetry from 'axios-retry';
import CircuitBreaker from 'opossum';

// 创建带超时和重试的 HTTP Client
function createHttpClient(baseURL: string): AxiosInstance {
  const client = axios.create({
    baseURL,
    timeout: 5000,  // 强制 5 秒超时
  });

  // 自动重试
  axiosRetry(client, {
    retries: 3,
    retryDelay: axiosRetry.exponentialDelay,
    retryCondition: (error) => {
      return axiosRetry.isNetworkOrIdempotentRequestError(error) ||
             error.code === 'ECONNABORTED';  // 超时也重试
    }
  });

  return client;
}

// 熔断器包装
class ResilientHttpClient {
  private client: AxiosInstance;
  private breaker: CircuitBreaker;

  constructor(baseURL: string) {
    this.client = createHttpClient(baseURL);
    
    this.breaker = new CircuitBreaker(
      (config: any) => this.client.request(config),
      {
        timeout: 10000,
        errorThresholdPercentage: 50,
        resetTimeout: 30000
      }
    );

    this.breaker.fallback(() => {
      throw new Error('Service temporarily unavailable');
    });
  }

  async get<T>(url: string): Promise<T> {
    const response = await this.breaker.fire({ method: 'GET', url });
    return response.data;
  }

  async post<T>(url: string, data: any): Promise<T> {
    const response = await this.breaker.fire({ method: 'POST', url, data });
    return response.data;
  }
}

// 使用
const smsClient = new ResilientHttpClient('https://sms-provider.com');

try {
  await smsClient.post('/send', { phone, message });
} catch (err) {
  // 降级处理：记录日志，后续重试
  await failedSmsQueue.add({ phone, message });
}
```

---

## 十、并发与幂等类问题

### 10.1 幂等失效导致重复下单（⭐⭐⭐⭐⭐）

#### 问题现象

```
用户点了一次下单按钮，收到 3 个订单
网络重试 + 用户重复点击
```

#### 面试话术

> 用户网络不好，点下单后 loading 很久，又点了两次。加上前端超时重试，一共发了 5 个请求，后端每个请求都创建了订单。
>
> **解决**：幂等 Token。前端进入下单页生成唯一 key，后端用 Redis 保证同一个 key 只处理一次。

#### 代码示例

```typescript
// 前端：生成幂等 key
const idempotencyKey = crypto.randomUUID();

await fetch('/api/orders', {
  method: 'POST',
  headers: {
    'X-Idempotency-Key': idempotencyKey
  },
  body: JSON.stringify(orderData)
});

// 后端：幂等处理
@Post('/orders')
async createOrder(
  @Body() data: CreateOrderDto,
  @Headers('X-Idempotency-Key') idempotencyKey: string
) {
  if (!idempotencyKey) {
    throw new BadRequestException('Idempotency key required');
  }

  const cacheKey = `idempotency:${idempotencyKey}`;
  
  // 尝试获取锁
  const locked = await redis.set(cacheKey, 'processing', 'NX', 'EX', 300);
  
  if (!locked) {
    // 已有相同请求在处理或已处理完
    const result = await this.waitForResult(cacheKey);
    if (result) {
      return JSON.parse(result);
    }
    throw new ConflictException('Request is being processed');
  }

  try {
    // 处理业务
    const order = await this.orderService.create(data);
    
    // 缓存结果（24小时）
    await redis.setex(`${cacheKey}:result`, 86400, JSON.stringify(order));
    
    return order;
  } catch (err) {
    await redis.del(cacheKey);  // 释放锁，允许重试
    throw err;
  }
}

private async waitForResult(cacheKey: string, maxWait = 5000): Promise<string | null> {
  const start = Date.now();
  while (Date.now() - start < maxWait) {
    const result = await redis.get(`${cacheKey}:result`);
    if (result) return result;
    await sleep(100);
  }
  return null;
}
```

---

## 十一、运维与稳定性类

### 11.1 日志打爆磁盘（⭐⭐⭐）

#### 问题现象

```
服务突然不响应，但没有任何错误日志
查了半天发现磁盘满了
```

#### 面试话术

> 凌晨告警服务全部 502，登上服务器发现磁盘 100%。原来有个 debug 日志没关，高峰期一天打了 500GB。
>
> **解决**：日志分级、日志轮转、磁盘监控告警、高频日志采样。

#### 代码示例

```typescript
import winston from 'winston';
import 'winston-daily-rotate-file';

const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'warn' : 'debug',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    // 日志轮转
    new winston.transports.DailyRotateFile({
      filename: 'logs/app-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '100m',    // 单文件最大 100MB
      maxFiles: '7d',      // 保留 7 天
      zippedArchive: true  // 压缩旧文件
    }),
    // 错误日志单独存储
    new winston.transports.DailyRotateFile({
      filename: 'logs/error-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      level: 'error',
      maxSize: '100m',
      maxFiles: '30d'
    })
  ]
});

// 高频日志采样
function sampledLog(level: string, message: string, meta?: any, sampleRate = 0.01) {
  if (Math.random() < sampleRate) {
    logger.log(level, message, { ...meta, sampled: true });
  }
}

// 使用
app.use((req, res, next) => {
  // 每个请求都打日志太多了，采样 1%
  sampledLog('debug', 'Request received', {
    method: req.method,
    url: req.url
  });
  next();
});
```

---

### 11.2 微信支付可靠性（⭐⭐⭐⭐⭐）

#### 问题现象

```
用户付了钱但系统没收到通知
同一笔支付通知多次，导致重复充值
```

#### 面试话术

> 我处理过一个微信支付回调丢失导致的资损问题。服务器滚动重启时，微信回调正好打到重启的 Pod，重试几次后放弃，导致订单悬挂。
>
> **解决**：三重保障——回调幂等、主动查询补偿、每日对账。

#### 代码示例

```typescript
class PaymentService {
  
  // 保障1：回调幂等
  async handleCallback(notification: WechatPayNotification): Promise<void> {
    const { orderId, transactionId } = notification;
    
    // 分布式锁
    const lock = await this.redlock.acquire([`pay:${orderId}`], 10000);
    
    try {
      const order = await this.prisma.order.findUnique({ where: { orderId } });
      
      // 幂等检查
      if (order.status === 'PAID' || order.status === 'COMPLETED') {
        return;  // 已处理，直接返回成功
      }
      
      await this.processPayment(order, notification);
    } finally {
      await lock.release();
    }
  }

  // 保障2：主动查询补偿
  @Cron('*/5 * * * *')  // 每 5 分钟
  async reconcilePendingOrders(): Promise<void> {
    const pendingOrders = await this.prisma.order.findMany({
      where: {
        status: 'PAYING',
        createdAt: { lt: new Date(Date.now() - 5 * 60 * 1000) }  // 超过 5 分钟
      }
    });

    for (const order of pendingOrders) {
      const wechatResult = await this.wechatClient.queryOrder(order.orderId);
      
      if (wechatResult.trade_state === 'SUCCESS') {
        await this.processPayment(order, wechatResult);
      } else if (wechatResult.trade_state === 'CLOSED') {
        await this.closeOrder(order);
      }
    }
  }

  // 保障3：每日对账
  @Cron('0 2 * * *')  // 每天凌晨 2 点
  async dailyReconciliation(): Promise<void> {
    const yesterday = dayjs().subtract(1, 'day').format('YYYY-MM-DD');
    
    // 下载微信账单
    const wechatBill = await this.wechatClient.downloadBill(yesterday);
    
    // 获取我们的订单
    const ourOrders = await this.getOrdersByDate(yesterday);
    
    // 对比
    const discrepancies = this.compareOrders(wechatBill, ourOrders);
    
    if (discrepancies.length > 0) {
      await this.alertService.send('支付对账异常', {
        date: yesterday,
        count: discrepancies.length,
        details: discrepancies
      });
    }
  }
}
```

---

## 十二、场景选择指南

### 12.1 按面试官问法选择

| 问法 | 推荐场景 |
|------|---------|
| "最有技术深度的挑战" | WFQ+DAG 调度、Saga 分布式事务、内存泄漏 |
| "解决过最难的 bug" | 内存泄漏、Event Loop 阻塞、消息丢失、浮点数精度 |
| "处理过什么线上故障" | 缓存雪崩、第三方超时、死锁、日志爆盘 |
| "并发问题怎么处理" | 点数超卖、消息重复消费、幂等失效 |
| "支付怎么保证可靠" | 微信支付三重保障 |

### 12.2 按岗位级别选择

| 级别 | 推荐场景 | 侧重点 |
|------|---------|--------|
| 中级工程师 | 内存泄漏、连接池泄漏、幂等失效 | 排查过程、代码修复 |
| 高级工程师 | Event Loop 阻塞、消息丢失、死锁 | 原理分析、方案对比 |
| 架构师 | WFQ+DAG、分布式事务、缓存雪崩 | 系统设计、权衡取舍 |
| Tech Lead | 支付可靠性、线上故障 | 流程规范、团队沉淀 |

### 12.3 完整场景地图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           技术难点/Bug 场景地图                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  内存/CPU   │  │   数据库    │  │    缓存     │  │  消息队列   │       │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤       │
│  │ 内存泄漏    │  │ 死锁        │  │ 雪崩        │  │ 消息丢失    │       │
│  │ EventLoop  │  │ 连接池泄漏  │  │ 热点Key     │  │ 重复消费    │       │
│  │ ReDoS      │  │ 慢SQL锁表   │  │ 穿透        │  │             │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  分布式     │  │  数据精度   │  │   网络      │  │  并发/幂等  │       │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤       │
│  │ 分布式事务  │  │ 浮点数精度  │  │ WS重连风暴  │  │ 点数超卖    │       │
│  │ 分布式锁    │  │ 时区问题    │  │ 第三方超时  │  │ 重复下单    │       │
│  │ WFQ+DAG    │  │             │  │             │  │             │       │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐                                          │
│  │   运维      │  │    支付     │                                          │
│  ├─────────────┤  ├─────────────┤                                          │
│  │ 日志爆盘    │  │ 回调丢失    │                                          │
│  │ 磁盘满      │  │ 重复充值    │                                          │
│  │             │  │ 对账异常    │                                          │
│  └─────────────┘  └─────────────┘                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 附录：快速记忆卡片

### A. 内存泄漏

```
现象：服务 2-3 天后 OOM
工具：heapdump + Chrome DevTools
常见原因：EventEmitter 监听器未移除、闭包持有引用、全局变量累积
解决：once/removeListener、WeakMap、定期检查
```

### B. Event Loop 阻塞

```
现象：P99 延迟飙高，P50 正常
工具：blocked-at
常见原因：大 JSON 解析、复杂正则、同步加密
解决：Worker Threads、流式处理、输入限制
```

### C. 缓存雪崩

```
现象：Redis 故障后数据库被打爆
解决：二级缓存 + 熔断器 + 连接池隔离 + 预热
```

### D. 消息丢失

```
现象：消息发了但没处理
原因：自动提交 offset + 处理中断
解决：手动提交 + 幂等处理
```

### E. 并发超卖

```
现象：余额变负数
原因：检查-执行不是原子操作（TOCTOU）
解决：TCC + 乐观锁，检查和更新在同一个原子操作
```

