# 消息队列（Message Queue）

消息队列是现代分布式系统的核心组件，用于解耦服务、异步处理、削峰填谷、提高系统可用性和扩展性。

## 📚 学习路径

### 第一阶段：核心概念（必修）

- [ ] **消息队列基础**
  - [ ] 消息队列的作用和使用场景
  - [ ] 消息模型：点对点 vs 发布/订阅
  - [ ] 消息可靠性：At Most Once、At Least Once、Exactly Once
  - [ ] 消息顺序性保证
  - [ ] 消息持久化

- [ ] **Kafka 核心**
  - [ ] 架构：Broker、Topic、Partition、Replication
  - [ ] Producer：分区策略、ACK 机制、幂等性、事务
  - [ ] Consumer：消费者组、偏移量管理、再平衡
  - [ ] 高性能原理：零拷贝、顺序写、批处理
  - [ ] Node.js 集成（kafkajs）

- [ ] **RabbitMQ 核心**
  - [ ] AMQP 协议和架构
  - [ ] Exchange 类型：Direct、Topic、Fanout、Headers
  - [ ] 消息可靠性：Publisher Confirms、消息持久化、ACK
  - [ ] 死信队列（DLQ）
  - [ ] Node.js 集成（amqplib）

- [ ] **AWS SQS & SNS**
  - [ ] SQS：Standard vs FIFO Queue
  - [ ] 可见性超时和消息生命周期
  - [ ] 死信队列配置
  - [ ] SNS：Topic、Subscription、消息过滤
  - [ ] SNS + SQS Fanout 架构

### 第二阶段：高级特性（深入）

- [ ] **Kafka 高级**
  - [ ] Exactly Once 语义实现
  - [ ] 事件溯源（Event Sourcing）
  - [ ] CQRS 模式
  - [ ] Saga 分布式事务
  - [ ] Kafka Streams

- [ ] **RabbitMQ 高级**
  - [ ] 优先级队列
  - [ ] 延迟队列/定时消息
  - [ ] RPC 模式
  - [ ] 集群和高可用（镜像队列、Quorum 队列）
  - [ ] 消息路由模式

- [ ] **性能优化**
  - [ ] 批量操作优化
  - [ ] 连接池管理
  - [ ] 背压（Backpressure）处理
  - [ ] 消息压缩
  - [ ] 分区/队列数量选择

- [ ] **可靠性保证**
  - [ ] 消息丢失防护
  - [ ] 消息重复处理（幂等性）
  - [ ] 消息堆积处理
  - [ ] 错误处理和重试策略
  - [ ] 死信队列设计

### 第三阶段：实战应用（必会）

- [ ] **常见架构模式**
  - [ ] 异步任务队列
  - [ ] 事件驱动架构
  - [ ] Fanout 广播
  - [ ] 消息过滤和路由
  - [ ] 请求/响应（RPC）模式

- [ ] **Node.js 集成实践**
  - [ ] Producer/Consumer 封装
  - [ ] 连接管理和重连
  - [ ] 优雅关闭
  - [ ] 错误处理
  - [ ] 监控和日志

- [ ] **监控和运维**
  - [ ] 关键指标监控
  - [ ] 消费延迟（Lag）监控
  - [ ] 告警配置
  - [ ] 性能调优
  - [ ] 故障排查

## 📖 模块内容

### [01-kafka.md](./01-kafka.md)

Kafka 是分布式流处理平台，以高吞吐量和强大的可扩展性著称。

**核心内容**：
- 架构：Topic、Partition、Replication、ISR
- Producer：分区策略、ACK 配置、幂等性、事务
- Consumer：消费者组、偏移量、再平衡、处理模式
- 性能优化：批处理、压缩、零拷贝
- 实战场景：事件溯源、CQRS、Saga 模式
- 监控和运维

**面试重点**：
- Kafka 为什么快？
- 如何保证消息不丢失？
- 如何保证消息顺序？
- 消费者再平衡机制
- Exactly Once 语义实现
- Kafka vs RabbitMQ 对比

### [02-rabbitmq.md](./02-rabbitmq.md)

RabbitMQ 是成熟的消息代理，实现 AMQP 协议，以灵活的路由和可靠性著称。

**核心内容**：
- AMQP 协议和架构组件
- Exchange 类型：Direct、Topic、Fanout、Headers
- 消息可靠性：Publisher Confirms、持久化、ACK
- 高级特性：优先级队列、延迟队列、RPC 模式
- 死信队列（DLQ）设计
- 集群和高可用

**面试重点**：
- Exchange 类型和使用场景
- 消息可靠性保证
- 事务 vs 确认模式
- 如何处理消息堆积？
- 集群和镜像队列
- RabbitMQ vs Kafka 选择

### [03-aws-sqs-sns.md](./03-aws-sqs-sns.md)

AWS 托管消息服务，SQS 是队列服务，SNS 是发布/订阅服务。

**核心内容**：
- SQS：Standard vs FIFO Queue
- 消息生命周期和可见性超时
- 死信队列配置
- SNS：Topic、Subscription、消息过滤
- SNS + SQS Fanout 架构
- 性能优化和成本控制

**面试重点**：
- Standard vs FIFO Queue 选择
- 如何处理消息重复？
- 可见性超时机制
- SNS vs SQS 使用场景
- 长轮询 vs 短轮询
- 成本优化策略

## 🎯 面试高频问题

### 1. 消息队列的作用和使用场景？

**作用**：
- **解耦**：服务之间松耦合，降低依赖
- **异步**：提高响应速度，优化用户体验
- **削峰**：流量高峰时缓冲请求
- **可靠性**：消息持久化，系统容错
- **扩展性**：水平扩展，提高吞吐量

**使用场景**：
- 异步任务处理（邮件发送、数据导出）
- 服务解耦（订单系统 → 库存、支付、物流）
- 流量削峰（秒杀、抢购）
- 日志收集和分析
- 事件驱动架构
- 数据同步和 CDC

### 2. At Most Once、At Least Once、Exactly Once 的区别？

**At Most Once（最多一次）**：
- 消息可能丢失，不会重复
- Producer 不重试
- Consumer 先提交后处理
- 场景：可容忍丢失（日志、监控指标）

**At Least Once（至少一次）**：✅ 最常用
- 消息不丢失，可能重复
- Producer 重试
- Consumer 先处理后提交
- 需要幂等性处理
- 场景：大多数业务场景

**Exactly Once（精确一次）**：
- 消息不丢失、不重复
- 需要分布式事务支持
- 性能开销大
- 场景：金融交易、账务系统

### 3. 如何保证消息顺序？

**Kafka**：
- 单分区内有序
- 相同 key 的消息发到同一分区
- Consumer 单线程消费

**RabbitMQ**：
- 单队列有序
- Consumer 单线程消费
- 消息设置顺序号

**注意**：
- 全局顺序会牺牲性能
- 大多数场景只需局部顺序（按业务 key）

### 4. 消息丢失的原因和预防？

**Producer 端**：
- 问题：网络故障、Broker 宕机
- 预防：重试机制、ACK 确认、持久化

**Broker 端**：
- 问题：磁盘故障、未刷盘
- 预防：多副本、同步复制、持久化配置

**Consumer 端**：
- 问题：处理失败、自动提交偏移量
- 预防：手动提交、先处理后确认

### 5. 消息堆积如何处理？

**原因**：
- 消费速度慢
- 消费者数量不足
- 消息处理失败重试

**解决**：
1. **增加消费者**：水平扩展
2. **提高消费速度**：批量处理、异步处理
3. **增加分区/队列**：提高并行度
4. **优先级处理**：重要消息优先
5. **临时降级**：跳过非关键消息
6. **扩容 Broker**：提高吞吐量

### 6. 如何实现消息幂等性？

**方法 1：唯一 ID**
```typescript
const processedIds = new Set();

if (processedIds.has(message.id)) {
  return;  // 已处理
}
processedIds.add(message.id);
```

**方法 2：数据库唯一约束**
```typescript
await db.orders.insertOne({
  _id: message.orderId,  // 唯一约束
  ...message
});
```

**方法 3：版本号/时间戳**
```typescript
await db.orders.updateOne(
  { id: orderId, version: message.version - 1 },
  { $set: { ...data, version: message.version } }
);
```

**方法 4：状态机**
```typescript
if (order.status !== 'pending') {
  return;  // 不是待处理状态，跳过
}
```

### 7. Kafka vs RabbitMQ 如何选择？

| 特性 | Kafka | RabbitMQ |
|------|-------|----------|
| 吞吐量 | 极高（百万级） | 中等（数万级） |
| 延迟 | 较低（ms 级） | 极低（μs 级） |
| 消息顺序 | 分区内有序 | 队列有序 |
| 消息持久化 | 默认持久化 | 可选 |
| 消息重放 | 支持 | 不支持 |
| 路由复杂度 | 简单 | 灵活 |
| 集群 | 原生分布式 | 主从/镜像 |
| 使用难度 | 中等 | 简单 |

**选择建议**：
- **高吞吐、消息重放** → Kafka（日志、事件流、大数据）
- **低延迟、复杂路由** → RabbitMQ（任务队列、RPC、通知）
- **简单任务队列** → RabbitMQ 或 AWS SQS
- **事件驱动架构** → Kafka 或 SNS+SQS

## 📝 学习建议

### 基础阶段
1. 理解消息队列的核心概念和作用
2. 熟悉 Kafka 和 RabbitMQ 的基本使用
3. 掌握 Node.js 客户端的基本操作
4. 理解消息可靠性保证机制

### 进阶阶段
1. 深入 Kafka 和 RabbitMQ 的架构原理
2. 掌握性能优化技巧
3. 学习高级特性（事务、幂等性、CQRS）
4. 理解不同消息队列的适用场景

### 实战阶段
1. 实现完整的生产者/消费者系统
2. 处理各种边界情况（网络故障、消息重复、堆积）
3. 集成监控和告警
4. 实践常见架构模式（事件驱动、Saga）

### 面试准备
1. 熟记核心概念和原理
2. 准备实际项目中的使用案例
3. 理解技术选型的权衡
4. 掌握常见问题的解决方案

## 🔗 相关资源

### Kafka
- [Kafka 官方文档](https://kafka.apache.org/documentation/)
- [KafkaJS 文档](https://kafka.js.org/)
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)

### RabbitMQ
- [RabbitMQ 官方文档](https://www.rabbitmq.com/documentation.html)
- [AMQP 协议](https://www.amqp.org/)
- [amqplib 文档](https://amqp-node.github.io/amqplib/)

### AWS
- [AWS SQS 文档](https://docs.aws.amazon.com/sqs/)
- [AWS SNS 文档](https://docs.aws.amazon.com/sns/)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)

## ✅ 检查清单

完成以下内容后，你应该能够：

- [ ] 解释消息队列的作用和适用场景
- [ ] 对比 Kafka、RabbitMQ、SQS 的差异
- [ ] 实现可靠的消息生产和消费
- [ ] 处理消息丢失、重复、堆积问题
- [ ] 设计事件驱动架构
- [ ] 实现分布式事务（Saga）
- [ ] 配置死信队列和错误处理
- [ ] 优化消息队列性能
- [ ] 监控消息队列健康状态
- [ ] 回答常见面试问题

## 🎓 面试评分标准

### 初级（0-2 年）
- 理解消息队列的基本概念
- 会使用 Kafka 或 RabbitMQ
- 知道如何发送和接收消息
- 了解基本的可靠性保证

### 中级（2-4 年）
- 深入理解消息队列原理
- 熟练使用多种消息队列
- 能处理常见问题（丢失、重复、堆积）
- 掌握性能优化技巧
- 理解不同场景的技术选型

### 高级（4+ 年）
- 精通消息队列架构和实现
- 能设计复杂的事件驱动架构
- 实现过分布式事务和 Saga
- 有大规模系统的实战经验
- 能从业务角度权衡技术方案

---

**下一模块**: [05-frameworks（框架）](../05-frameworks/README.md)
