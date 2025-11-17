# DevOps 开发运维

DevOps 是开发（Development）和运维（Operations）的结合，强调自动化、持续集成和快速迭代。

## 📚 学习路径

### 第一阶段：容器化（必修）

- [ ] **Docker 基础**
  - [ ] 容器 vs 虚拟机
  - [ ] Dockerfile 编写
  - [ ] 镜像构建与优化
  - [ ] 多阶段构建
  - [ ] Docker Compose

- [ ] **Docker 进阶**
  - [ ] 容器网络
  - [ ] 数据持久化（Volumes）
  - [ ] 安全最佳实践
  - [ ] 镜像优化技巧

### 第二阶段：CI/CD（必修）

- [ ] **持续集成**
  - [ ] CI/CD 概念
  - [ ] GitHub Actions
  - [ ] 自动化测试
  - [ ] 代码质量检查

- [ ] **持续部署**
  - [ ] 部署策略（蓝绿、金丝雀、滚动）
  - [ ] 自动化部署
  - [ ] 回滚策略
  - [ ] Secrets 管理

### 第三阶段：容器编排（选修）

- [ ] **Kubernetes 基础**
  - [ ] K8s 架构
  - [ ] Pod、Deployment、Service
  - [ ] ConfigMap 和 Secret
  - [ ] Ingress

- [ ] **Kubernetes 进阶**
  - [ ] 自动扩缩容
  - [ ] 健康检查
  - [ ] 资源限制
  - [ ] Helm

### 第四阶段：基础设施即代码（选修）

- [ ] **Infrastructure as Code**
  - [ ] Terraform 基础
  - [ ] CloudFormation
  - [ ] IaC 最佳实践

### 第五阶段：监控与日志

- [ ] **监控**
  - [ ] 详见监控模块
  - [ ] DevOps 监控指标

- [ ] **日志**
  - [ ] 集中式日志
  - [ ] ELK/EFK Stack

## 📖 模块内容

### [01-docker.md](./01-docker.md)

Docker 容器化技术。

**核心内容**：
- Docker 基础概念
- Dockerfile 编写（基础、多阶段、优化）
- 镜像优化策略
- Docker Compose 配置
- 容器管理命令
- 数据持久化（Volumes）
- 网络配置
- 安全最佳实践
- 调试和监控

**面试重点**：
- Docker vs 虚拟机
- 如何优化镜像大小
- 层缓存工作原理
- Docker Compose vs Kubernetes
- 数据持久化方案

### [02-cicd.md](./02-cicd.md)

CI/CD 持续集成与部署。

**核心内容**：
- CI/CD 概念和价值
- GitHub Actions 完整 Pipeline
- 多环境部署
- 部署策略（蓝绿、金丝雀、滚动）
- Secrets 管理
- 自动化测试集成
- 性能测试
- 回滚策略
- 最佳实践

**面试重点**：
- CI/CD 的好处
- 蓝绿部署 vs 金丝雀部署
- 如何确保 CI/CD 安全性
- Pipeline 设计原则

## 🎯 面试高频问题

### 1. Docker 容器与虚拟机的区别？

**架构差异**：

| 特性 | Docker 容器 | 虚拟机 |
|------|------------|--------|
| 虚拟化层级 | 操作系统级 | 硬件级 |
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | MB | GB |
| 隔离性 | 进程级 | 系统级 |
| 性能 | 接近原生 | 有损耗 |
| 可移植性 | 强 | 一般 |

**使用场景**：
- 微服务架构 → 容器
- 需要完全隔离 → 虚拟机
- 快速部署 → 容器
- 运行不同 OS → 虚拟机

### 2. 如何优化 Docker 镜像大小？

**优化策略**：

1. **选择小基础镜像**：
```dockerfile
# ❌ node:18 (~1GB)
# ✅ node:18-alpine (~170MB)
# ✅ node:18-slim (~250MB)
FROM node:18-alpine
```

2. **多阶段构建**：
```dockerfile
FROM node:18-alpine AS builder
RUN npm ci && npm run build

FROM node:18-alpine AS production
COPY --from=builder /app/dist ./dist
```

3. **清理缓存**：
```dockerfile
RUN npm ci --only=production && \
    npm cache clean --force
```

4. **使用 .dockerignore**：
```
node_modules
dist
.git
```

**效果**：
- 未优化：~1.2GB
- 优化后：~150MB

### 3. 什么是 CI/CD？为什么重要？

**定义**：

- **CI（持续集成）**：频繁将代码集成到主分支，自动构建和测试
- **CD（持续交付）**：确保代码随时可以部署
- **CD（持续部署）**：自动部署到生产环境

**重要性**：

1. **更快交付**：
   - 自动化减少手动时间
   - 缩短上线周期

2. **更高质量**：
   - 自动化测试
   - 早期发现问题

3. **更低风险**：
   - 小批量频繁发布
   - 快速回滚

4. **更好协作**：
   - 统一流程
   - 可视化状态

### 4. 部署策略对比？

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **蓝绿部署** | 快速切换和回滚、零停机 | 需要双倍资源 | 关键系统 |
| **金丝雀部署** | 风险小、渐进式 | 时间长、监控复杂 | 大规模变更 |
| **滚动更新** | 节省资源 | 回滚复杂 | 常规更新 |

**蓝绿部署**：
```
旧版本（蓝）运行 → 部署新版本（绿）→ 测试绿环境 → 切换流量 → 保留蓝作备份
```

**金丝雀部署**：
```
部署到 10% → 监控 → 50% → 监控 → 100%
```

### 5. GitHub Actions vs Jenkins？

| 特性 | GitHub Actions | Jenkins |
|------|----------------|---------|
| 托管 | 云端 | 自托管 |
| 配置 | YAML | UI + Groovy |
| 集成 | 原生 GitHub | 需要插件 |
| 成本 | 按分钟收费 | 服务器成本 |
| 学习曲线 | 简单 | 复杂 |
| 灵活性 | 中等 | 高 |

**选择**：
- GitHub 项目 → GitHub Actions
- 复杂流程 → Jenkins
- 快速上手 → GitHub Actions
- 自定义需求高 → Jenkins

### 6. Docker Compose vs Kubernetes？

| 特性 | Docker Compose | Kubernetes |
|------|----------------|------------|
| 复杂度 | 简单 | 复杂 |
| 规模 | 单机 | 集群 |
| 自动恢复 | 无 | 有 |
| 负载均衡 | 基础 | 高级 |
| 学习成本 | 低 | 高 |
| 适用场景 | 开发/小项目 | 生产/大规模 |

**使用建议**：
- 本地开发 → Docker Compose
- 小型项目 → Docker Compose
- 生产环境 → Kubernetes
- 微服务集群 → Kubernetes

### 7. 如何确保 CI/CD 安全？

**安全措施**：

1. **密钥管理**：
   - GitHub Secrets
   - AWS Secrets Manager
   - 定期轮换
   ```yaml
   env:
     API_KEY: ${{ secrets.API_KEY }}
   ```

2. **权限控制**：
   - 最小权限原则
   - 使用 IAM 角色
   - 环境分离

3. **代码扫描**：
   ```yaml
   - run: npm audit
   - uses: snyk/actions/node@master
   ```

4. **环境保护**：
   - 生产环境需审批
   - 分支保护规则

5. **审计日志**：
   - 记录所有部署
   - 可追溯性

### 8. 多阶段构建的好处？

**优势**：

1. **更小的镜像**：
   - 只包含运行时依赖
   - 不包含构建工具

2. **更安全**：
   - 不暴露源代码
   - 不包含开发依赖

3. **更快的部署**：
   - 镜像小，传输快

**示例**：
```dockerfile
# 构建阶段（包含 TypeScript、构建工具）
FROM node:18-alpine AS builder
RUN npm ci && npm run build

# 生产阶段（只包含运行时）
FROM node:18-alpine
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production
```

**效果**：
- 构建镜像：~800MB
- 生产镜像：~150MB

## 📝 实践项目

### 项目 1：Node.js 应用容器化

**目标**：将 NestJS 应用容器化

**步骤**：
1. 编写优化的 Dockerfile
2. 配置 Docker Compose
3. 实现多环境配置
4. 优化镜像大小

**验收标准**：
- [ ] 镜像大小 < 200MB
- [ ] 构建时间 < 2分钟
- [ ] 支持热重载

### 项目 2：CI/CD Pipeline

**目标**：完整的 CI/CD 流程

**步骤**：
1. 设置 GitHub Actions
2. 自动化测试（单元、集成、E2E）
3. 代码质量检查
4. 自动部署到 AWS ECS
5. 实现蓝绿部署

**验收标准**：
- [ ] 自动化测试覆盖率 > 80%
- [ ] 部署时间 < 10分钟
- [ ] 支持一键回滚

### 项目 3：监控和告警

**目标**：完善的监控体系

**步骤**：
1. 集成 CloudWatch
2. 配置自定义指标
3. 设置告警规则
4. 创建 Dashboard

**验收标准**：
- [ ] 关键指标全覆盖
- [ ] 告警及时（< 1分钟）
- [ ] 可视化 Dashboard

## ✅ 技能检查清单

完成以下内容后，你应该能够：

### Docker
- [ ] 编写生产级 Dockerfile
- [ ] 优化 Docker 镜像大小
- [ ] 使用 Docker Compose 管理多容器
- [ ] 理解层缓存机制
- [ ] 处理数据持久化
- [ ] 配置容器网络
- [ ] 实现安全最佳实践

### CI/CD
- [ ] 设计完整的 CI/CD Pipeline
- [ ] 配置 GitHub Actions
- [ ] 实现自动化测试
- [ ] 管理 Secrets 和环境变量
- [ ] 实现多种部署策略
- [ ] 配置自动回滚
- [ ] 集成安全扫描

### 运维
- [ ] 监控应用性能
- [ ] 配置日志收集
- [ ] 设置告警规则
- [ ] 处理故障恢复
- [ ] 优化成本

### 面试准备
- [ ] 解释 DevOps 理念
- [ ] 对比不同工具和方案
- [ ] 分享实践经验
- [ ] 讨论最佳实践
- [ ] 解决实际问题

## 🔗 相关资源

### 官方文档
- [Docker 官方文档](https://docs.docker.com/)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Kubernetes 文档](https://kubernetes.io/docs/)

### 学习资源
- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [12-Factor App](https://12factor.net/)
- [DevOps Roadmap](https://roadmap.sh/devops)

### 工具
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [k9s](https://k9scli.io/) - Kubernetes CLI
- [Lens](https://k8slens.dev/) - Kubernetes IDE

## 🎓 面试评分标准

### 初级（0-2 年）
- 了解 Docker 基础概念
- 能编写基础 Dockerfile
- 了解 CI/CD 概念
- 能使用 GitHub Actions

### 中级（2-4 年）
- 熟练使用 Docker 和 Compose
- 能优化 Docker 镜像
- 设计 CI/CD Pipeline
- 实现多种部署策略
- 了解 Kubernetes 基础

### 高级（4+ 年）
- 精通容器化和编排
- 设计大规模 DevOps 方案
- 优化 CI/CD 效率
- 实现复杂部署策略
- 有 Kubernetes 生产经验
- 能处理复杂故障

## 💡 学习建议

### 第一周：Docker 基础
1. 学习 Docker 概念
2. 编写 Dockerfile
3. 使用 Docker Compose
4. 实践项目容器化

### 第二周：Docker 进阶
1. 多阶段构建
2. 镜像优化
3. 网络和存储
4. 安全实践

### 第三周：CI/CD
1. GitHub Actions 基础
2. 自动化测试
3. 构建 Pipeline
4. 部署到云平台

### 第四周：实战
1. 完整项目实践
2. 优化和监控
3. 故障演练
4. 面试准备

---

**上一模块**: [10-aws（AWS）](../10-aws/README.md)  
**下一模块**: [12-monitoring-logging（监控与日志）](../12-monitoring-logging/README.md)
