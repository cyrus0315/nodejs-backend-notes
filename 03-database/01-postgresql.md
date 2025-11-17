# PostgreSQL

PostgreSQL 是一个功能强大的开源对象关系型数据库系统，被称为"世界上最先进的开源数据库"。

## PostgreSQL vs MySQL

### PostgreSQL 的优势

#### 1. 更强大的数据类型支持

```sql
-- PostgreSQL 支持丰富的数据类型
-- JSON/JSONB（二进制 JSON，性能更好）
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  metadata JSONB,  -- 二进制 JSON，支持索引
  tags TEXT[]      -- 数组类型
);

INSERT INTO users (name, metadata, tags) VALUES
  ('Alice', '{"age": 25, "city": "NYC"}', ARRAY['developer', 'blogger']);

-- JSONB 查询
SELECT * FROM users WHERE metadata->>'city' = 'NYC';
SELECT * FROM users WHERE metadata @> '{"age": 25}';

-- 数组操作
SELECT * FROM users WHERE 'developer' = ANY(tags);

-- UUID 类型
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  amount DECIMAL(10, 2)
);

-- 范围类型
CREATE TABLE bookings (
  id SERIAL PRIMARY KEY,
  room_id INT,
  during TSRANGE  -- 时间范围
);

INSERT INTO bookings (room_id, during) VALUES
  (101, '[2024-01-01 14:00, 2024-01-01 16:00)');

-- 检查重叠
SELECT * FROM bookings 
WHERE during && '[2024-01-01 15:00, 2024-01-01 17:00)';

-- 几何类型
CREATE TABLE locations (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  coordinates POINT
);

-- 网络地址类型
CREATE TABLE connections (
  id SERIAL PRIMARY KEY,
  ip_address INET,
  mac_address MACADDR
);
```

#### 2. 复杂查询和窗口函数

```sql
-- 窗口函数
CREATE TABLE sales (
  id SERIAL PRIMARY KEY,
  employee_id INT,
  sale_date DATE,
  amount DECIMAL(10, 2)
);

-- 计算每个员工的排名
SELECT 
  employee_id,
  sale_date,
  amount,
  RANK() OVER (PARTITION BY employee_id ORDER BY amount DESC) as rank,
  SUM(amount) OVER (PARTITION BY employee_id) as total_sales,
  AVG(amount) OVER (PARTITION BY employee_id) as avg_sales
FROM sales;

-- ROW_NUMBER, LEAD, LAG
SELECT 
  employee_id,
  sale_date,
  amount,
  ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY sale_date) as row_num,
  LEAD(amount) OVER (PARTITION BY employee_id ORDER BY sale_date) as next_sale,
  LAG(amount) OVER (PARTITION BY employee_id ORDER BY sale_date) as prev_sale
FROM sales;

-- CTE (Common Table Expressions) - 递归查询
WITH RECURSIVE employee_hierarchy AS (
  -- 基础情况：顶层员工
  SELECT id, name, manager_id, 1 as level
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- 递归情况：下级员工
  SELECT e.id, e.name, e.manager_id, eh.level + 1
  FROM employees e
  INNER JOIN employee_hierarchy eh ON e.manager_id = eh.id
)
SELECT * FROM employee_hierarchy ORDER BY level, id;
```

#### 3. 全文搜索 (Full-Text Search)

```sql
-- 创建全文搜索索引
CREATE TABLE articles (
  id SERIAL PRIMARY KEY,
  title TEXT,
  content TEXT,
  search_vector TSVECTOR
);

-- 创建 GIN 索引加速搜索
CREATE INDEX articles_search_idx ON articles USING GIN(search_vector);

-- 插入数据并生成搜索向量
INSERT INTO articles (title, content, search_vector) VALUES
  ('PostgreSQL Tutorial', 'Learn PostgreSQL database...', 
   to_tsvector('english', 'PostgreSQL Tutorial Learn PostgreSQL database'));

-- 更新触发器自动维护搜索向量
CREATE TRIGGER articles_search_update BEFORE INSERT OR UPDATE
ON articles FOR EACH ROW EXECUTE FUNCTION
tsvector_update_trigger(search_vector, 'pg_catalog.english', title, content);

-- 全文搜索
SELECT title, content,
       ts_rank(search_vector, query) AS rank
FROM articles, 
     to_tsquery('english', 'PostgreSQL & database') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- 高亮搜索结果
SELECT title,
       ts_headline('english', content, 
                   to_tsquery('english', 'PostgreSQL'),
                   'MaxWords=50, MinWords=30')
FROM articles
WHERE search_vector @@ to_tsquery('english', 'PostgreSQL');
```

#### 4. 高级索引类型

```sql
-- B-tree 索引（默认）
CREATE INDEX idx_users_email ON users(email);

-- Hash 索引（等值查询）
CREATE INDEX idx_users_id_hash ON users USING HASH(id);

-- GiST 索引（几何数据、全文搜索）
CREATE INDEX idx_locations_coords ON locations USING GIST(coordinates);

-- GIN 索引（JSONB、数组、全文搜索）
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);
CREATE INDEX idx_users_tags ON users USING GIN(tags);

-- 部分索引（条件索引）
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- 表达式索引
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 多列索引
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- 唯一索引
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- BRIN 索引（大表，有序数据）
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(created_at);
```

#### 5. 事务和 MVCC

```sql
-- PostgreSQL 使用 MVCC（多版本并发控制）
-- 读操作不会阻塞写操作，写操作不会阻塞读操作

-- 事务隔离级别
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- PostgreSQL 支持真正的 SERIALIZABLE 隔离级别
-- MySQL InnoDB 的 REPEATABLE READ 存在幻读问题

-- 保存点
BEGIN;
  INSERT INTO accounts (name, balance) VALUES ('Alice', 1000);
  SAVEPOINT my_savepoint;
  
  UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
  -- 出错了，回滚到保存点
  ROLLBACK TO SAVEPOINT my_savepoint;
  
  -- 继续其他操作
  UPDATE accounts SET balance = balance + 50 WHERE name = 'Alice';
COMMIT;

-- 咨询锁（Advisory Locks）
SELECT pg_advisory_lock(12345);
-- 执行关键操作
SELECT pg_advisory_unlock(12345);

-- 行级锁
SELECT * FROM users WHERE id = 1 FOR UPDATE;  -- 排他锁
SELECT * FROM users WHERE id = 1 FOR SHARE;   -- 共享锁
```

#### 6. 继承和表分区

```sql
-- 表继承
CREATE TABLE cities (
  name TEXT,
  population REAL,
  elevation INT
);

CREATE TABLE capitals (
  state CHAR(2)
) INHERITS (cities);

-- 查询包括子表
SELECT name, elevation FROM cities;

-- 只查询父表
SELECT name, elevation FROM ONLY cities;

-- 声明式分区（PostgreSQL 10+）
CREATE TABLE measurements (
  city_id INT NOT NULL,
  logdate DATE NOT NULL,
  peaktemp INT,
  unitsales INT
) PARTITION BY RANGE (logdate);

-- 创建分区
CREATE TABLE measurements_y2023 PARTITION OF measurements
  FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE measurements_y2024 PARTITION OF measurements
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- 自动路由到正确的分区
INSERT INTO measurements VALUES (1, '2023-06-15', 85, 120);
```

#### 7. 视图和物化视图

```sql
-- 普通视图
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE active = true;

-- 可更新视图
CREATE VIEW user_emails AS
SELECT id, email FROM users;

UPDATE user_emails SET email = 'new@example.com' WHERE id = 1;

-- 物化视图（预计算并存储结果）
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
  DATE_TRUNC('month', sale_date) as month,
  employee_id,
  SUM(amount) as total_sales,
  AVG(amount) as avg_sale
FROM sales
GROUP BY DATE_TRUNC('month', sale_date), employee_id;

-- 创建索引
CREATE INDEX ON sales_summary(employee_id, month);

-- 刷新物化视图
REFRESH MATERIALIZED VIEW sales_summary;

-- 并发刷新（不阻塞查询）
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

#### 8. 扩展功能

```sql
-- PostGIS（地理信息系统）
CREATE EXTENSION postgis;

CREATE TABLE locations (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  geom GEOMETRY(Point, 4326)
);

-- 插入地理坐标
INSERT INTO locations (name, geom) VALUES
  ('New York', ST_SetSRID(ST_MakePoint(-74.006, 40.7128), 4326));

-- 查询距离
SELECT name, 
       ST_Distance(
         geom,
         ST_SetSRID(ST_MakePoint(-73.935242, 40.730610), 4326)
       ) as distance
FROM locations
ORDER BY distance;

-- pg_trgm（相似度搜索）
CREATE EXTENSION pg_trgm;

CREATE INDEX idx_users_name_trgm ON users USING GIN(name gin_trgm_ops);

-- 模糊搜索
SELECT name, similarity(name, 'Alice') as sim
FROM users
WHERE name % 'Alice'  -- % 是相似度操作符
ORDER BY sim DESC;

-- uuid-ossp
CREATE EXTENSION "uuid-ossp";
SELECT uuid_generate_v4();

-- hstore（键值对）
CREATE EXTENSION hstore;

CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  attributes HSTORE
);

INSERT INTO products (name, attributes) VALUES
  ('Laptop', 'brand=>Dell, ram=>16GB, ssd=>512GB');

SELECT name FROM products WHERE attributes->'brand' = 'Dell';
```

### MySQL 的优势

虽然 PostgreSQL 功能更强大，但 MySQL 也有其优势：

1. **更简单易用**：配置和管理相对简单
2. **更快的读操作**：在某些简单查询场景下性能更好
3. **更广泛的应用**：生态系统更成熟，更多的工具和资源
4. **复制更简单**：主从复制配置相对简单

## PostgreSQL 基础

### 数据类型

```sql
-- 数值类型
SMALLINT          -- 2 字节，-32768 到 32767
INTEGER (INT)     -- 4 字节，约 -21 亿到 21 亿
BIGINT            -- 8 字节，非常大的整数
DECIMAL(p, s)     -- 精确数值
NUMERIC(p, s)     -- 同 DECIMAL
REAL              -- 4 字节浮点数
DOUBLE PRECISION  -- 8 字节浮点数
SERIAL            -- 自增整数（1 到 21 亿）
BIGSERIAL         -- 自增大整数

-- 字符类型
CHAR(n)           -- 定长字符串
VARCHAR(n)        -- 变长字符串
TEXT              -- 无限长文本

-- 日期时间
DATE              -- 日期
TIME              -- 时间
TIMESTAMP         -- 时间戳
TIMESTAMPTZ       -- 带时区时间戳
INTERVAL          -- 时间间隔

-- 布尔
BOOLEAN           -- true/false/null

-- 二进制
BYTEA             -- 二进制数据

-- JSON
JSON              -- JSON 文本
JSONB             -- 二进制 JSON（推荐）

-- 数组
INTEGER[]         -- 整数数组
TEXT[]            -- 文本数组

-- 范围类型
INT4RANGE         -- 整数范围
TSRANGE           -- 时间戳范围
DATERANGE         -- 日期范围

-- UUID
UUID              -- 通用唯一标识符

-- 网络地址
INET              -- IP 地址
CIDR              -- 网络地址
MACADDR           -- MAC 地址

-- 几何
POINT             -- 平面点
LINE              -- 无限线
LSEG              -- 线段
BOX               -- 矩形框
PATH              -- 几何路径
POLYGON           -- 多边形
CIRCLE            -- 圆
```

### 基本操作

```sql
-- 创建数据库
CREATE DATABASE myapp;

-- 连接数据库
\c myapp;

-- 创建表
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 插入数据
INSERT INTO users (email, username, password_hash)
VALUES ('alice@example.com', 'alice', 'hashed_password');

-- 批量插入
INSERT INTO users (email, username, password_hash) VALUES
  ('bob@example.com', 'bob', 'hash1'),
  ('charlie@example.com', 'charlie', 'hash2');

-- 查询
SELECT * FROM users WHERE email = 'alice@example.com';

-- 更新
UPDATE users SET updated_at = CURRENT_TIMESTAMP WHERE id = 1;

-- 删除
DELETE FROM users WHERE id = 1;

-- RETURNING 子句（PostgreSQL 特性）
INSERT INTO users (email, username, password_hash)
VALUES ('dave@example.com', 'dave', 'hash')
RETURNING id, created_at;

UPDATE users SET username = 'new_name' WHERE id = 1
RETURNING *;

DELETE FROM users WHERE id = 1 RETURNING *;
```

### 约束

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT CHECK (quantity > 0),
  total_amount DECIMAL(10, 2) CHECK (total_amount >= 0),
  status VARCHAR(20) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- 外键约束
  CONSTRAINT fk_user FOREIGN KEY (user_id) 
    REFERENCES users(id) 
    ON DELETE CASCADE,
  
  CONSTRAINT fk_product FOREIGN KEY (product_id)
    REFERENCES products(id)
    ON DELETE RESTRICT,
  
  -- 唯一约束
  CONSTRAINT unique_user_product UNIQUE (user_id, product_id),
  
  -- 检查约束
  CONSTRAINT check_status CHECK (status IN ('pending', 'paid', 'shipped', 'delivered'))
);

-- 添加约束
ALTER TABLE orders ADD CONSTRAINT check_quantity CHECK (quantity <= 100);

-- 删除约束
ALTER TABLE orders DROP CONSTRAINT check_quantity;
```

### 索引优化

```sql
-- 查看查询计划
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';

-- 详细执行计划
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';

-- 常用索引策略
-- 1. 频繁查询的列
CREATE INDEX idx_users_email ON users(email);

-- 2. 外键列
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 3. 排序和分组的列
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- 4. JSONB 字段
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- 5. 部分索引（减小索引大小）
CREATE INDEX idx_active_orders ON orders(user_id) 
WHERE status IN ('pending', 'paid');

-- 6. 表达式索引
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 7. 覆盖索引（INCLUDE）
CREATE INDEX idx_users_email_include ON users(email) INCLUDE (username);

-- 维护索引
REINDEX INDEX idx_users_email;
REINDEX TABLE users;

-- 查看索引使用情况
SELECT 
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- 查找未使用的索引
SELECT 
  schemaname,
  tablename,
  indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_toast%';
```

## Node.js 中使用 PostgreSQL

### pg 驱动

```typescript
import { Pool, Client } from 'pg';

// 连接池（推荐）
const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'myapp',
  user: 'postgres',
  password: 'password',
  max: 20, // 最大连接数
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// 查询
async function getUser(id: number) {
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [id]
  );
  return result.rows[0];
}

// 参数化查询（防止 SQL 注入）
async function createUser(email: string, username: string, passwordHash: string) {
  const result = await pool.query(
    'INSERT INTO users (email, username, password_hash) VALUES ($1, $2, $3) RETURNING *',
    [email, username, passwordHash]
  );
  return result.rows[0];
}

// 事务
async function transferMoney(fromId: number, toId: number, amount: number) {
  const client = await pool.connect();
  
  try {
    await client.query('BEGIN');
    
    // 扣款
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );
    
    // 加款
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    
    await client.query('COMMIT');
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// 批量操作
async function batchInsert(users: Array<{ email: string; username: string }>) {
  const values = users.map((user, i) => 
    `($${i * 2 + 1}, $${i * 2 + 2})`
  ).join(',');
  
  const params = users.flatMap(user => [user.email, user.username]);
  
  const query = `
    INSERT INTO users (email, username)
    VALUES ${values}
    RETURNING *
  `;
  
  const result = await pool.query(query, params);
  return result.rows;
}

// 流式查询（大数据量）
import { QueryStream } from 'pg-query-stream';

async function streamUsers() {
  const client = await pool.connect();
  
  const query = new QueryStream('SELECT * FROM users');
  const stream = client.query(query);
  
  stream.on('data', (row) => {
    console.log(row);
  });
  
  stream.on('end', () => {
    client.release();
  });
  
  stream.on('error', (error) => {
    console.error(error);
    client.release();
  });
}

// 监听通知（LISTEN/NOTIFY）
async function listenForChanges() {
  const client = await pool.connect();
  
  await client.query('LISTEN user_changes');
  
  client.on('notification', (msg) => {
    console.log('Notification:', msg.channel, msg.payload);
  });
}

// 发送通知
async function notifyChange(userId: number) {
  await pool.query(
    "SELECT pg_notify('user_changes', $1)",
    [JSON.stringify({ userId })]
  );
}

// 优雅关闭
process.on('SIGTERM', async () => {
  await pool.end();
  process.exit(0);
});
```

## 性能优化

### 查询优化

```sql
-- 1. 使用 EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM orders WHERE user_id = 123;

-- 2. 避免 SELECT *
-- ❌ 不好
SELECT * FROM users WHERE id = 1;

-- ✅ 好
SELECT id, email, username FROM users WHERE id = 1;

-- 3. 使用索引
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 4. 批量操作
-- ❌ 不好：多次查询
FOR id IN ids:
  SELECT * FROM users WHERE id = id;

-- ✅ 好：一次查询
SELECT * FROM users WHERE id = ANY($1);

-- 5. JOIN 优化
-- 确保 JOIN 的列有索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);

-- 6. 使用 CTE 优化复杂查询
WITH recent_orders AS (
  SELECT user_id, COUNT(*) as order_count
  FROM orders
  WHERE created_at > NOW() - INTERVAL '30 days'
  GROUP BY user_id
)
SELECT u.*, ro.order_count
FROM users u
LEFT JOIN recent_orders ro ON u.id = ro.user_id;

-- 7. 分页优化
-- ❌ 不好：OFFSET 性能差
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 10000;

-- ✅ 好：使用游标分页
SELECT * FROM orders WHERE id > $1 ORDER BY id LIMIT 10;
```

### 连接池配置

```typescript
const pool = new Pool({
  // 基本配置
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || '5432'),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  
  // 连接池配置
  max: 20,                        // 最大连接数
  min: 5,                         // 最小连接数
  idleTimeoutMillis: 30000,       // 空闲连接超时
  connectionTimeoutMillis: 2000,  // 连接超时
  
  // SSL 配置
  ssl: process.env.NODE_ENV === 'production' ? {
    rejectUnauthorized: false
  } : false,
  
  // 语句超时
  statement_timeout: 30000,       // 30 秒
  query_timeout: 30000,
});

// 监控连接池
pool.on('connect', () => {
  console.log('New client connected');
});

pool.on('error', (err) => {
  console.error('Unexpected error on idle client', err);
});

// 获取连接池统计
console.log({
  total: pool.totalCount,
  idle: pool.idleCount,
  waiting: pool.waitingCount
});
```

## 常见面试题

### 1. PostgreSQL 比 MySQL 有哪些优势？

<details>
<summary>点击查看答案</summary>

**核心优势**：

1. **更丰富的数据类型**
   - JSONB、数组、范围类型、UUID、几何类型等
   - MySQL 只有基本的数据类型

2. **更强大的查询功能**
   - 窗口函数（RANK, ROW_NUMBER, LEAD, LAG等）
   - 递归 CTE（WITH RECURSIVE）
   - 完整的子查询支持

3. **全文搜索**
   - 内置全文搜索引擎
   - MySQL 需要额外的工具（Elasticsearch）

4. **更多的索引类型**
   - B-tree, Hash, GiST, GIN, BRIN
   - 支持部分索引、表达式索引

5. **真正的 SERIALIZABLE**
   - 完整的 ACID 支持
   - 更好的并发控制（MVCC）

6. **扩展性**
   - PostGIS（地理信息）
   - pg_trgm（模糊搜索）
   - 丰富的扩展生态

7. **高级特性**
   - 表继承和分区
   - 物化视图
   - LISTEN/NOTIFY
   - 外部数据包装器（FDW）
</details>

### 2. JSONB 和 JSON 的区别？

<details>
<summary>点击查看答案</summary>

**JSON**：
- 存储为文本
- 保留原始格式（包括空格）
- 插入快
- 查询慢

**JSONB**（推荐）：
- 存储为二进制
- 不保留格式
- 插入稍慢（需要解析）
- 查询快
- 支持索引（GIN）
- 支持更多操作符

```sql
-- JSONB 操作
SELECT metadata->>'name' FROM users;           -- 提取文本
SELECT metadata->'address'->>'city' FROM users; -- 嵌套提取
SELECT * FROM users WHERE metadata @> '{"age": 25}';  -- 包含
SELECT * FROM users WHERE metadata ? 'age';     -- 键存在

-- JSONB 索引
CREATE INDEX idx_metadata ON users USING GIN(metadata);
```

**选择**：
- 需要保留格式 → JSON
- 需要查询和索引 → JSONB（推荐）
</details>

### 3. 如何优化 PostgreSQL 查询性能？

<details>
<summary>点击查看答案</summary>

**优化策略**：

1. **使用 EXPLAIN ANALYZE**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 123;
```

2. **创建合适的索引**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

3. **使用覆盖索引**
```sql
CREATE INDEX idx_orders_user_include 
ON orders(user_id) INCLUDE (created_at, total);
```

4. **避免 SELECT ***
```sql
-- ✅ 只选择需要的列
SELECT id, email FROM users WHERE id = 1;
```

5. **批量操作**
```sql
-- ✅ 使用 IN 或 ANY
SELECT * FROM users WHERE id = ANY($1);
```

6. **使用连接池**
```typescript
const pool = new Pool({ max: 20 });
```

7. **分页优化**
```sql
-- ✅ 使用游标分页
SELECT * FROM orders WHERE id > $1 ORDER BY id LIMIT 10;
```

8. **定期 VACUUM**
```sql
VACUUM ANALYZE orders;
```
</details>

### 4. PostgreSQL 的事务隔离级别？

<details>
<summary>点击查看答案</summary>

PostgreSQL 支持 4 种隔离级别：

1. **READ UNCOMMITTED**（实际上是 READ COMMITTED）
2. **READ COMMITTED**（默认）
3. **REPEATABLE READ**
4. **SERIALIZABLE**

```sql
-- 设置隔离级别
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- READ COMMITTED（默认）
-- - 可以读取已提交的数据
-- - 可能有不可重复读

-- REPEATABLE READ
-- - 同一事务中读取的数据一致
-- - PostgreSQL 使用快照隔离
-- - 没有幻读问题（与 MySQL 不同）

-- SERIALIZABLE
-- - 完全隔离
-- - 如果检测到冲突会回滚
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM accounts WHERE id = 1;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

**PostgreSQL vs MySQL**：
- PostgreSQL 的 REPEATABLE READ 没有幻读
- MySQL 的 REPEATABLE READ 可能有幻读
- PostgreSQL 的 SERIALIZABLE 是真正的可串行化
</details>

### 5. 如何在 PostgreSQL 中实现分页？

<details>
<summary>点击查看答案</summary>

**方法 1：OFFSET/LIMIT**（简单但性能差）

```sql
SELECT * FROM orders 
ORDER BY id 
LIMIT 10 OFFSET 20;
```

缺点：
- OFFSET 越大越慢
- 需要扫描前面所有行

**方法 2：游标分页**（推荐）

```sql
-- 第一页
SELECT * FROM orders 
ORDER BY id 
LIMIT 10;

-- 下一页（使用上一页最后一条的 id）
SELECT * FROM orders 
WHERE id > $last_id 
ORDER BY id 
LIMIT 10;
```

优点：
- 性能稳定
- 不受数据量影响

**方法 3：Keyset Pagination**（最佳性能）

```typescript
interface PaginationParams {
  limit: number;
  cursor?: {
    id: number;
    created_at: Date;
  };
}

async function getOrders(params: PaginationParams) {
  const { limit, cursor } = params;
  
  let query = 'SELECT * FROM orders ';
  const values: any[] = [];
  
  if (cursor) {
    query += 'WHERE (created_at, id) < ($1, $2) ';
    values.push(cursor.created_at, cursor.id);
  }
  
  query += 'ORDER BY created_at DESC, id DESC LIMIT $' + (values.length + 1);
  values.push(limit);
  
  const result = await pool.query(query, values);
  
  return {
    items: result.rows,
    nextCursor: result.rows.length > 0 
      ? {
          id: result.rows[result.rows.length - 1].id,
          created_at: result.rows[result.rows.length - 1].created_at
        }
      : null
  };
}
```
</details>

## 最佳实践

### 1. 使用参数化查询

```typescript
// ✅ 好：防止 SQL 注入
await pool.query('SELECT * FROM users WHERE email = $1', [email]);

// ❌ 不好：SQL 注入风险
await pool.query(`SELECT * FROM users WHERE email = '${email}'`);
```

### 2. 使用连接池

```typescript
// ✅ 好：重用连接
const pool = new Pool({ max: 20 });
await pool.query('SELECT * FROM users');

// ❌ 不好：每次创建新连接
const client = new Client();
await client.connect();
```

### 3. 使用事务

```typescript
// ✅ 好：保证数据一致性
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, fromId]);
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, toId]);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');
  throw e;
} finally {
  client.release();
}
```

### 4. 定期维护

```sql
-- 定期 VACUUM
VACUUM ANALYZE;

-- 重建索引
REINDEX TABLE users;

-- 更新统计信息
ANALYZE users;
```

