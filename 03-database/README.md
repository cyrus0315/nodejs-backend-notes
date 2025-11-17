# 数据库知识

本模块聚焦于高级特性、性能优化和面试重点，已创建详细的实战文档。

## 📚 学习文档

### ✅ 已完成

1. **[PostgreSQL](./01-postgresql.md)** (35KB)
   - **PostgreSQL vs MySQL 深度对比**（8 大优势）
   - 丰富的数据类型（JSONB, 数组, UUID, 范围类型, 几何类型）
   - 窗口函数和复杂查询（ROW_NUMBER, RANK, 递归 CTE）
   - 全文搜索（GIN 索引, ts_vector）
   - 7 种高级索引类型（GiST, GIN, BRIN, 部分索引）
   - 事务和 MVCC（真正的 SERIALIZABLE）
   - 表继承和声明式分区
   - 物化视图
   - 扩展功能（PostGIS, pg_trgm, uuid-ossp）
   - Node.js (pg) 最佳实践
   - 连接池和性能优化
   - 5 道高级面试题

2. **[Redis](./02-redis.md)** (38KB)
   - 5 种数据类型深度解析
   - Node.js (ioredis) 高级用法
   - **5 个实战场景**：
     - 缓存系统（Cache-Aside, Read-Through, Write-Behind）
     - 分布式锁（Redlock 模式）
     - 限流器（固定窗口, 滑动窗口, 令牌桶）
     - 会话管理
     - 排行榜系统
   - 缓存三大问题及解决方案：
     - 缓存穿透（布隆过滤器）
     - 缓存击穿（互斥锁）
     - 缓存雪崩（随机过期时间）
   - Pipeline 和事务
   - Lua 脚本原子操作
   - 发布订阅
   - 5 道深度面试题

3. **[MongoDB](./03-mongodb.md)** (32KB)
   - 文档模型 vs 关系模型
   - 索引策略（复合索引, ESR 规则, 地理空间索引）
   - **聚合框架**（Pipeline, $lookup, $facet）
   - 事务（ACID 支持）
   - **5 种数据建模模式**：
     - 嵌套文档模式
     - 引用模式
     - 桶模式（时序数据）
     - 扩展引用模式
     - 预聚合模式
   - Mongoose 最佳实践
   - Change Streams（实时监听）
   - 分片（Sharding）策略
   - 性能优化技巧
   - 5 道架构设计面试题

4. **[DynamoDB](./04-dynamodb.md)** (30KB)
   - **主键设计原则**（分区键选择, 避免热分区）
   - GSI vs LSI 深度对比
   - **3 种高级查询模式**：
     - 单表设计（Single Table Design）
     - 分层数据建模
     - 邻接列表模式（社交网络）
   - 事务操作（25 项限制）
   - 条件表达式和更新表达式
   - DynamoDB Streams（实时处理）
   - 批量操作（BatchGet, BatchWrite）
   - 容量模式（按需 vs 预置）
   - PartiQL 查询
   - AWS 生态集成
   - 5 道 NoSQL 设计面试题

## 📝 知识点清单

### PostgreSQL ⭐⭐⭐
- [x] JSONB 和高级数据类型
- [x] 窗口函数
- [x] 全文搜索
- [x] 高级索引类型
- [x] MVCC 和事务隔离
- [x] 表分区
- [x] 物化视图
- [x] 扩展功能

### Redis ⭐⭐⭐
- [x] 5 种数据类型
- [x] 缓存策略
- [x] 分布式锁
- [x] 限流实现
- [x] 缓存三大问题
- [x] Pipeline 和事务
- [x] Lua 脚本
- [x] 发布订阅

### MongoDB ⭐⭐
- [x] 文档模型设计
- [x] 索引优化
- [x] 聚合框架
- [x] 数据建模模式
- [x] 事务支持
- [x] Change Streams
- [x] 分片策略
- [x] 性能优化

### DynamoDB ⭐⭐
- [x] 主键设计
- [x] GSI/LSI 使用
- [x] 单表设计
- [x] 事务操作
- [x] DynamoDB Streams
- [x] 批量操作
- [x] 容量优化
- [x] 最佳实践

## 💡 核心要点

### 必须掌握 ⭐⭐⭐

**PostgreSQL**：
- 为什么选择 PostgreSQL 而不是 MySQL
- JSONB 的使用场景和索引
- 窗口函数的实际应用
- 事务隔离级别的区别

**Redis**：
- 5 种数据类型的应用场景
- 缓存三大问题的解决方案
- 分布式锁的实现
- 如何实现限流

**MongoDB**：
- 嵌套 vs 引用的选择
- 聚合框架的使用
- 索引优化策略
- 数据建模最佳实践

**DynamoDB**：
- 主键设计原则
- GSI vs LSI 的区别
- 单表设计模式
- 如何避免热分区

## 🔥 高频面试题

### PostgreSQL
1. PostgreSQL 比 MySQL 有哪些优势？
2. JSONB 和 JSON 的区别？
3. 如何优化 PostgreSQL 查询性能？
4. PostgreSQL 的事务隔离级别？
5. 如何在 PostgreSQL 中实现分页？

### Redis
1. Redis 为什么这么快？
2. Redis 的持久化方式？
3. 如何实现分布式锁？
4. Redis 集群方案？
5. 缓存和数据库一致性？

### MongoDB
1. MongoDB 的优势和适用场景？
2. MongoDB 的索引原理？
3. 如何避免 MongoDB 性能问题？
4. 复制集（Replica Set）工作原理？
5. 嵌套文档 vs 引用，如何选择？

### DynamoDB
1. DynamoDB 的主键设计原则？
2. 何时使用 GSI vs LSI？
3. 单表设计的优缺点？
4. 如何处理热分区问题？
5. DynamoDB 事务的限制？

## 📊 对比总结

### 何时使用哪个数据库？

**PostgreSQL**：
- 复杂查询和分析
- 需要事务和一致性
- 地理信息系统（PostGIS）
- 全文搜索

**MySQL**：
- 简单的 CRUD 操作
- 读多写少
- 需要简单易用

**Redis**：
- 缓存
- 会话存储
- 实时计数
- 排行榜
- 分布式锁

**MongoDB**：
- 灵活的 Schema
- 快速迭代
- 内容管理系统
- 实时分析
- 物联网数据

**DynamoDB**：
- 无服务器架构
- 需要自动扩展
- 可预测的性能
- AWS 生态
- 高可用要求

## 🎯 学习路径

**第一阶段**（必学）：
1. PostgreSQL 基础和优势
2. Redis 数据类型和缓存策略
3. 索引优化原理

**第二阶段**（深入）：
1. PostgreSQL 窗口函数和 JSONB
2. Redis 分布式锁和限流
3. MongoDB 聚合框架
4. DynamoDB 主键设计

**第三阶段**（实战）：
1. 数据建模最佳实践
2. 性能优化技巧
3. 缓存策略设计
4. 分布式系统数据一致性

## 🚀 实战技能

每个文档都包含实战代码和场景：

1. **PostgreSQL**：
   - 复杂聚合查询
   - 全文搜索实现
   - 地理位置查询
   - 性能优化案例

2. **Redis**：
   - 缓存系统设计
   - 分布式锁实现
   - 三种限流算法
   - 排行榜系统

3. **MongoDB**：
   - 数据建模模式
   - 聚合管道设计
   - 索引优化
   - Change Streams 应用

4. **DynamoDB**：
   - 单表设计实践
   - 事务处理
   - Streams 处理
   - 批量操作优化

## 📚 推荐资源

- [PostgreSQL 官方文档](https://www.postgresql.org/docs/)
- [Redis 官方文档](https://redis.io/documentation)
- [MongoDB University](https://university.mongodb.com/)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/)
- [Database Internals](https://www.databass.dev/) - 深入理解数据库原理

## 💪 面试准备建议

1. **理解原理**：不仅知道怎么用，更要知道为什么
2. **实战练习**：运行文档中的所有代码示例
3. **性能优化**：重点掌握查询优化和索引设计
4. **方案对比**：能够说出不同数据库的优劣和适用场景
5. **真实案例**：准备 2-3 个实际项目中的数据库使用案例

