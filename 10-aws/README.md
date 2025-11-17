# AWS 服务

Amazon Web Services (AWS) 是最流行的云计算平台，本模块专注于 Node.js 后端开发中常用的 AWS 服务。

## 📚 学习路径

### 第一阶段：Serverless 核心（必修）

- [ ] **Lambda**
  - [ ] Serverless 概念
  - [ ] 函数创建和配置
  - [ ] 触发器类型（API Gateway、SQS、S3等）
  - [ ] 冷启动优化
  - [ ] 错误处理和重试
  - [ ] 与其他服务集成

- [ ] **API Gateway**
  - [ ] REST API vs HTTP API
  - [ ] 路由和集成
  - [ ] 授权和认证
  - [ ] 限流和 CORS
  - [ ] 部署和阶段

- [ ] **DynamoDB**
  - [ ] 详见数据库模块
  - [ ] 单表设计
  - [ ] GSI/LSI
  - [ ] Streams + Lambda

### 第二阶段：存储和数据库（必修）

- [ ] **S3**
  - [ ] Bucket 和对象操作
  - [ ] 预签名 URL
  - [ ] 生命周期策略
  - [ ] 事件通知
  - [ ] 存储类别

- [ ] **RDS**
  - [ ] PostgreSQL/MySQL
  - [ ] 连接池管理
  - [ ] 备份和恢复
  - [ ] Multi-AZ 部署

- [ ] **ElastiCache**
  - [ ] Redis 集群
  - [ ] 缓存策略
  - [ ] 与 Lambda 集成

### 第三阶段：消息和事件（必修）

- [ ] **SQS**
  - [ ] Standard vs FIFO
  - [ ] 消息可见性
  - [ ] 死信队列
  - [ ] 与 Lambda 批量处理

- [ ] **SNS**
  - [ ] Topic 和订阅
  - [ ] 消息过滤
  - [ ] SNS + SQS Fanout

- [ ] **EventBridge**
  - [ ] 事件总线
  - [ ] 事件规则和目标
  - [ ] 事件驱动架构

### 第四阶段：安全和监控

- [ ] **IAM**
  - [ ] 用户、组、角色
  - [ ] 策略管理
  - [ ] 最小权限原则
  - [ ] 临时凭证（STS）

- [ ] **Secrets Manager**
  - [ ] 密钥管理
  - [ ] 自动轮换
  - [ ] 与应用集成

- [ ] **CloudWatch**
  - [ ] 日志管理
  - [ ] 自定义指标
  - [ ] 告警配置
  - [ ] Insights 查询

### 第五阶段：高级服务

- [ ] **Step Functions**
  - [ ] 状态机设计
  - [ ] 工作流编排
  - [ ] 错误处理和重试

- [ ] **ECS/EKS**
  - [ ] 容器编排
  - [ ] Fargate
  - [ ] 任务定义和服务

- [ ] **CloudFormation**
  - [ ] Infrastructure as Code
  - [ ] 堆栈管理
  - [ ] 参数和输出

## 📖 模块内容

### [01-lambda-serverless.md](./01-lambda-serverless.md)

AWS Lambda 和 Serverless 架构。

**核心内容**：
- Serverless 概念和优劣势
- Lambda 函数创建和配置
- 各种触发器（API Gateway、SQS、S3、DynamoDB、EventBridge）
- 环境变量和 Lambda Layers
- 冷启动优化策略
- 并发控制和错误处理
- 监控和日志
- Serverless Framework 配置
- 与其他 AWS 服务集成

**面试重点**：
- 冷启动优化
- Lambda 限制
- 幂等性实现

### [02-core-services.md](./02-core-services.md)

Node.js 开发中常用的 AWS 核心服务。

**核心内容**：
- IAM（角色、策略、临时凭证）
- S3（基础操作、预签名 URL、事件通知）
- RDS（PostgreSQL 连接、连接池、Secrets Manager）
- ElastiCache（Redis 使用、缓存模式）
- CloudWatch（日志、自定义指标）
- Secrets Manager（密钥管理和缓存）
- EventBridge（事件发布和订阅）
- Step Functions（状态机工作流）

**面试重点**：
- IAM 角色 vs 用户
- S3 预签名 URL 原理
- RDS vs DynamoDB 选择

## 🎯 面试高频问题

### 1. Lambda 冷启动如何优化？

**优化策略**：

1. **减小部署包**：
   - 使用 esbuild/webpack
   - 只导入需要的依赖
   - Tree-shaking

2. **运行时选择**：
   - ARM64 架构（快 20%）
   - 新版本 Node.js

3. **Provisioned Concurrency**：
   - 预留实例
   - 消除冷启动

4. **代码优化**：
   - 在 handler 外初始化
   - 延迟加载依赖
   - Lambda Layers

5. **配置优化**：
   - 增加内存（CPU 也增加）
   - VPC 优化

**冷启动时间**：
- Node.js 18：~200-300ms
- Node.js 18 (ARM64)：~150-250ms
- Provisioned Concurrency：0ms

### 2. AWS SDK v2 vs v3？

| 特性 | SDK v2 | SDK v3 |
|------|--------|--------|
| 模块化 | 全量导入 | 按需导入 |
| 包大小 | 大（~40MB） | 小（~100KB） |
| 性能 | 较慢 | 快 |
| TypeScript | 需要 @types | 原生支持 |
| 异步 | Promise/Callback | Promise only |

**迁移示例**：
```typescript
// v2
import AWS from 'aws-sdk';
const s3 = new AWS.S3();
await s3.putObject({...}).promise();

// v3
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
const s3Client = new S3Client({});
await s3Client.send(new PutObjectCommand({...}));
```

### 3. SQS Standard vs FIFO？

| 特性 | Standard | FIFO |
|------|----------|------|
| 吞吐量 | 无限 | 3000 TPS |
| 顺序 | 尽力而为 | 严格有序 |
| 重复 | 可能重复 | 精确一次 |
| 使用场景 | 高吞吐 | 顺序关键 |

**选择建议**：
- 需要严格顺序 → FIFO
- 高吞吐量 → Standard
- 金融交易 → FIFO
- 日志处理 → Standard

### 4. RDS 连接池最佳实践？

```typescript
import { Pool } from 'pg';

// ❌ 每次创建新连接
export const handler = async () => {
  const client = new Client({...});
  await client.connect();
  // ...
  await client.end();
};

// ✅ 复用连接池
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

export const handler = async () => {
  const client = await pool.connect();
  try {
    // ...
  } finally {
    client.release();
  }
};
```

**配置建议**：
- `max`: min(RDS最大连接数 / Lambda并发数, 20)
- Lambda 并发控制：避免耗尽数据库连接

### 5. S3 成本优化策略？

1. **生命周期策略**：
```json
{
  "Rules": [{
    "Transitions": [
      {
        "Days": 30,
        "StorageClass": "STANDARD_IA"
      },
      {
        "Days": 90,
        "StorageClass": "GLACIER"
      }
    ],
    "Expiration": {
      "Days": 365
    }
  }]
}
```

2. **存储类别选择**：
   - 频繁访问 → S3 Standard
   - 偶尔访问 → S3 Standard-IA
   - 归档 → S3 Glacier

3. **压缩和去重**

4. **CloudFront 缓存**

### 6. 如何设计高可用架构？

**关键策略**：

1. **多 AZ 部署**：
   - RDS Multi-AZ
   - ELB 跨 AZ
   - Lambda 自动多 AZ

2. **故障转移**：
   - Route 53 健康检查
   - 自动故障转移

3. **备份策略**：
   - RDS 自动备份
   - S3 版本控制
   - 跨区域复制

4. **监控告警**：
   - CloudWatch Alarms
   - SNS 通知

5. **弹性伸缩**：
   - Auto Scaling
   - Lambda 自动扩展

### 7. EventBridge vs SNS vs SQS？

| 服务 | 用途 | 特点 |
|------|------|------|
| EventBridge | 事件路由 | 规则匹配、多目标 |
| SNS | 发布/订阅 | Fanout、推送 |
| SQS | 消息队列 | 解耦、拉取 |

**使用场景**：
- 事件驱动架构 → EventBridge
- 消息广播 → SNS
- 任务队列 → SQS
- 组合：EventBridge → SNS → SQS

### 8. Lambda 最佳实践？

1. **函数设计**：
   - 单一职责
   - 无状态
   - 幂等性

2. **性能优化**：
   - 减小包大小
   - 预初始化连接
   - 选择合适内存

3. **错误处理**：
   - 区分可重试错误
   - 使用 DLQ
   - 结构化日志

4. **安全**：
   - 最小权限 IAM
   - 使用 Secrets Manager
   - 加密环境变量

5. **监控**：
   - CloudWatch Logs
   - 自定义指标
   - X-Ray 追踪

## 📝 学习建议

### 基础阶段
1. 从 Lambda 开始，理解 Serverless
2. 掌握 S3、DynamoDB 基础操作
3. 学习 IAM 权限管理
4. 理解 SQS/SNS 消息服务

### 进阶阶段
1. 深入 Lambda 优化
2. 设计事件驱动架构
3. 使用 Step Functions 编排
4. 掌握 CloudWatch 监控

### 实战阶段
1. 构建完整 Serverless 应用
2. 实现高可用架构
3. 优化成本和性能
4. 自动化部署（CloudFormation）

### 面试准备
1. 熟记各服务的使用场景
2. 准备架构设计案例
3. 理解成本优化策略
4. 掌握常见问题解决方案

## ✅ 检查清单

完成以下内容后，你应该能够：

- [ ] 创建和部署 Lambda 函数
- [ ] 使用 AWS SDK 操作各种服务
- [ ] 设计 Serverless 架构
- [ ] 实现事件驱动系统
- [ ] 配置 IAM 权限
- [ ] 优化 Lambda 性能
- [ ] 设置监控和告警
- [ ] 实现高可用架构
- [ ] 控制和优化成本
- [ ] 回答常见面试问题

## 🔗 相关资源

### 官方文档
- [AWS 官方文档](https://docs.aws.amazon.com/)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
- [Serverless Framework](https://www.serverless.com/framework/docs)

### 学习资源
- [AWS 架构中心](https://aws.amazon.com/architecture/)
- [AWS 最佳实践](https://aws.amazon.com/architecture/well-architected/)
- [Serverless Patterns](https://serverlessland.com/patterns)

### 工具
- [LocalStack](https://localstack.cloud/) - 本地 AWS 模拟
- [AWS SAM](https://aws.amazon.com/serverless/sam/) - Serverless 开发框架
- [CDK](https://aws.amazon.com/cdk/) - Infrastructure as Code

## 🎓 面试评分标准

### 初级（0-2 年）
- 会使用基础 AWS 服务（S3、Lambda）
- 理解 Serverless 概念
- 能创建简单的 Lambda 函数
- 了解 IAM 基础

### 中级（2-4 年）
- 熟练使用多种 AWS 服务
- 能设计 Serverless 架构
- 掌握性能优化技巧
- 理解安全最佳实践
- 能解决常见问题

### 高级（4+ 年）
- 精通 AWS 服务和架构设计
- 能设计高可用系统
- 有大规模项目经验
- 能优化成本和性能
- 理解 AWS 底层原理

---

**下一模块**: [11-devops（DevOps）](../11-devops/README.md)
