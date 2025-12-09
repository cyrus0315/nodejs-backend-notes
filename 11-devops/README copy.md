# DevOps 开发运维

DevOps 是开发（Development）和运维（Operations）的结合，强调自动化、持续集成和快速迭代。本模块全面覆盖从容器化到云原生的完整 DevOps 技术栈。

## 📚 学习路径

### 第一阶段：容器化（必修）⭐⭐⭐⭐⭐

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

### 第二阶段：CI/CD（必修）⭐⭐⭐⭐⭐

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

### 第三阶段：容器编排（必修）⭐⭐⭐⭐⭐

- [ ] **Kubernetes 基础**
  - [ ] K8s 架构
  - [ ] Pod、Deployment、Service
  - [ ] ConfigMap 和 Secret
  - [ ] Ingress
  - [ ] 健康检查（Liveness/Readiness Probe）

- [ ] **Kubernetes 进阶**
  - [ ] 自动扩缩容（HPA/VPA/KEDA）
  - [ ] 资源管理（Requests/Limits）
  - [ ] 网络策略
  - [ ] Helm Charts
  - [ ] Operator 模式

### 第四阶段：基础设施即代码（必修）⭐⭐⭐⭐

- [ ] **Terraform**
  - [ ] HCL 语法
  - [ ] 状态管理
  - [ ] 模块化
  - [ ] 工作空间

- [ ] **其他 IaC 工具**
  - [ ] Pulumi（TypeScript）
  - [ ] AWS CloudFormation
  - [ ] IaC 最佳实践

### 第五阶段：GitOps（进阶）⭐⭐⭐⭐

- [ ] **ArgoCD**
  - [ ] Application 定义
  - [ ] ApplicationSet（多环境）
  - [ ] Sync Waves 和 Hooks
  - [ ] 通知配置

- [ ] **Flux**
  - [ ] GitRepository 和 Kustomization
  - [ ] HelmRelease
  - [ ] 镜像自动更新

- [ ] **GitOps 工作流**
  - [ ] 仓库结构设计
  - [ ] PR 审批流程
  - [ ] 渐进式发布

### 第六阶段：云原生（进阶）⭐⭐⭐⭐⭐

- [ ] **12-Factor App**
  - [ ] 配置外部化
  - [ ] 无状态进程
  - [ ] 易处理（优雅关闭）
  - [ ] 日志作为事件流
  - [ ] 开发/生产等价

- [ ] **Service Mesh**
  - [ ] Istio 架构
  - [ ] 流量管理
  - [ ] 安全（mTLS）
  - [ ] 熔断器

- [ ] **Serverless**
  - [ ] AWS Lambda
  - [ ] Serverless Framework
  - [ ] 冷启动优化

- [ ] **云原生模式**
  - [ ] Sidecar 模式
  - [ ] 熔断器模式
  - [ ] 重试模式
  - [ ] 舱壁模式

---

## 📖 模块内容

### [01-docker.md](./01-docker.md) ⭐ 基础

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

---

### [02-cicd.md](./02-cicd.md) ⭐ 基础

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

---

### [03-kubernetes.md](./03-kubernetes.md) ⭐ 核心

Kubernetes 容器编排。

**核心内容**：
- K8s 架构和核心组件
- Pod 与 Deployment
- Service 类型（ClusterIP/NodePort/LoadBalancer）
- Ingress（NGINX/ALB）
- ConfigMap 和 Secret
- 自动扩缩容（HPA/VPA/KEDA）
- Helm Charts 开发
- 网络策略
- Pod 安全上下文
- 生产最佳实践

**面试重点**：
- Pod 生命周期
- Service 如何实现负载均衡
- 零停机部署实现
- ConfigMap 更新机制
- 资源限制三层模型

---

### [04-infrastructure-as-code.md](./04-infrastructure-as-code.md) ⭐ 核心

基础设施即代码。

**核心内容**：
- IaC 概念和价值
- Terraform 完整教程
  - HCL 语法
  - 状态管理
  - 模块化设计
  - 工作空间
- Pulumi（TypeScript）
- AWS CloudFormation
- Node.js 应用基础设施示例
  - VPC/子网
  - ECS/Fargate
  - RDS Aurora
  - ALB
  - Auto Scaling
- IaC 最佳实践

**面试重点**：
- Terraform state 的作用
- 如何处理状态漂移
- Terraform vs Pulumi 选择
- IaC 安全最佳实践

---

### [05-gitops.md](./05-gitops.md) ⭐ 进阶

GitOps 实践。

**核心内容**：
- GitOps 原理和优势
- ArgoCD 完整教程
  - Application 定义
  - ApplicationSet（多环境/多集群）
  - Project 和 RBAC
  - Sync Waves 和 Hooks
  - 通知配置
- Flux 基础
- GitOps 工作流设计
- Kustomize 配置管理
- 渐进式发布（Argo Rollouts）
- 秘钥管理（Sealed Secrets/External Secrets）
- 策略即代码（Kyverno）

**面试重点**：
- GitOps vs 传统 CI/CD
- 如何处理敏感数据
- 多环境部署方案
- 灾难恢复

---

### [06-cloud-native.md](./06-cloud-native.md) ⭐ 进阶

云原生架构。

**核心内容**：
- 云原生定义和特征
- 12-Factor App 完整解读
- Service Mesh（Istio）
  - 流量管理
  - 安全（mTLS）
  - 可观测性
  - 熔断器
- Serverless
  - AWS Lambda + Node.js
  - Serverless Framework
  - 冷启动优化
- 云原生设计模式
  - Sidecar 模式
  - Ambassador 模式
  - Circuit Breaker 模式
  - Retry 模式
  - Bulkhead 模式
- 可观测性三大支柱
- OpenTelemetry 集成

**面试重点**：
- 云原生核心特征
- 12-Factor App 最重要的几条
- Service Mesh 解决什么问题
- Serverless 优缺点

---

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

### 2. 如何实现零停机部署？

**关键技术**：
1. **RollingUpdate**：maxSurge=1, maxUnavailable=0
2. **健康检查**：Liveness + Readiness Probe
3. **优雅关闭**：preStop hook + terminationGracePeriodSeconds
4. **PodDisruptionBudget**：保证最小可用副本数
5. **蓝绿/金丝雀**：流量切换策略

### 3. 什么是 GitOps？

**核心原则**：
1. **声明式**：系统状态以声明式配置描述
2. **版本控制**：Git 作为唯一事实源
3. **自动拉取**：Agent 自动检测并应用变更
4. **持续协调**：持续确保实际状态与期望状态一致

**优势**：
- 完整审计日志
- Git revert 即回滚
- 更安全（Pull 模式）

### 4. 12-Factor App 最重要的几条？

1. **配置（Config）**：环境变量存储配置
2. **无状态进程（Processes）**：应用无状态
3. **易处理（Disposability）**：快速启动、优雅关闭
4. **日志（Logs）**：日志作为事件流
5. **开发/生产等价**：环境一致性

### 5. Service Mesh vs API Gateway？

| 特性 | Service Mesh | API Gateway |
|------|-------------|-------------|
| 位置 | 服务间（东西向） | 入口（南北向） |
| 协议 | 所有 | HTTP/REST/GraphQL |
| 功能 | 通信基础设施 | API 管理 |
| 部署 | Sidecar | 集中式 |
| 侵入性 | 低 | 中 |

---

## 📝 实践项目

### 项目 1：Node.js 应用容器化

**目标**：将 NestJS 应用容器化

**步骤**：
1. 编写优化的 Dockerfile（多阶段构建）
2. 配置 Docker Compose（含 PostgreSQL、Redis）
3. 实现多环境配置
4. 优化镜像大小

**验收标准**：
- [ ] 镜像大小 < 200MB
- [ ] 构建时间 < 2 分钟
- [ ] 支持本地开发热重载

---

### 项目 2：完整 CI/CD Pipeline

**目标**：GitHub Actions 完整流程

**步骤**：
1. 设置代码质量检查（ESLint、TypeScript）
2. 自动化测试（单元、集成、E2E）
3. Docker 镜像构建和推送
4. 自动部署到 AWS ECS/K8s
5. 实现蓝绿部署

**验收标准**：
- [ ] 测试覆盖率 > 80%
- [ ] 部署时间 < 10 分钟
- [ ] 支持一键回滚

---

### 项目 3：Kubernetes 生产部署

**目标**：生产级 K8s 部署配置

**步骤**：
1. 创建 Deployment（资源限制、健康检查）
2. 配置 Service 和 Ingress
3. 实现 HPA 自动扩缩容
4. 创建 Helm Chart
5. 配置网络策略

**验收标准**：
- [ ] 完整的健康检查配置
- [ ] HPA 正确响应负载
- [ ] 网络策略限制流量

---

### 项目 4：GitOps 工作流

**目标**：ArgoCD GitOps 部署

**步骤**：
1. 设置配置仓库（Kustomize）
2. 安装和配置 ArgoCD
3. 创建 Application 和 ApplicationSet
4. 实现 CI 更新配置仓库
5. 配置通知

**验收标准**：
- [ ] Git push 自动部署
- [ ] 多环境配置管理
- [ ] 完整的审计日志

---

### 项目 5：完整云原生基础设施

**目标**：Terraform + K8s + GitOps

**步骤**：
1. Terraform 创建 AWS 基础设施（VPC、EKS）
2. 安装 ArgoCD 和 Istio
3. 部署 Node.js 微服务
4. 配置 Service Mesh 流量管理
5. 实现可观测性（日志、指标、追踪）

**验收标准**：
- [ ] 基础设施代码化
- [ ] 服务间 mTLS
- [ ] 完整可观测性

---

## ✅ 技能检查清单

完成以下内容后，你应该能够：

### Docker
- [ ] 编写生产级 Dockerfile
- [ ] 优化 Docker 镜像大小（<200MB）
- [ ] 使用 Docker Compose 管理多容器
- [ ] 理解层缓存机制
- [ ] 实现安全最佳实践（非 root 用户）

### CI/CD
- [ ] 设计完整的 CI/CD Pipeline
- [ ] 配置 GitHub Actions
- [ ] 实现多种部署策略
- [ ] 管理 Secrets 和环境变量
- [ ] 配置自动回滚

### Kubernetes
- [ ] 部署生产级工作负载
- [ ] 配置 Ingress 和 TLS
- [ ] 实现 HPA/VPA 自动扩缩容
- [ ] 创建和使用 Helm Charts
- [ ] 配置网络策略和安全

### IaC
- [ ] 使用 Terraform 管理基础设施
- [ ] 理解状态管理和锁
- [ ] 创建可复用模块
- [ ] 实现多环境部署

### GitOps
- [ ] 配置 ArgoCD Application
- [ ] 实现多环境 GitOps
- [ ] 处理敏感数据
- [ ] 设计 PR 审批流程

### 云原生
- [ ] 遵循 12-Factor 原则
- [ ] 理解 Service Mesh 概念
- [ ] 实现云原生设计模式
- [ ] 配置完整可观测性

---

## 🔗 相关资源

### 官方文档
- [Docker 官方文档](https://docs.docker.com/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Terraform 官方文档](https://developer.hashicorp.com/terraform/docs)
- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)
- [Istio 官方文档](https://istio.io/latest/docs/)

### 学习资源
- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Patterns](https://k8spatterns.io/)
- [12-Factor App](https://12factor.net/)
- [GitOps Principles](https://opengitops.dev/)
- [CNCF Landscape](https://landscape.cncf.io/)

### 工具
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [k9s](https://k9scli.io/) - Kubernetes CLI
- [Lens](https://k8slens.dev/) - Kubernetes IDE
- [tfenv](https://github.com/tfutils/tfenv) - Terraform 版本管理
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - 上下文切换

---

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
- 熟练使用 IaC 工具
- 实现 GitOps 工作流
- 有 Kubernetes 生产经验
- 理解云原生架构

### 架构师
- 设计企业级 DevOps 平台
- 多云/混合云架构
- Service Mesh 和 GitOps
- DevSecOps 安全实践
- 成本优化和容量规划
- 技术决策和团队建设

---

## 💡 学习建议

### 第一周：Docker 基础
1. 学习 Docker 概念
2. 编写 Dockerfile
3. 使用 Docker Compose
4. 实践项目容器化

### 第二周：CI/CD
1. GitHub Actions 基础
2. 自动化测试集成
3. 构建 Pipeline
4. 部署策略

### 第三周：Kubernetes
1. K8s 核心概念
2. 部署工作负载
3. Service 和 Ingress
4. 健康检查和扩缩容

### 第四周：IaC 和 GitOps
1. Terraform 基础
2. ArgoCD 配置
3. GitOps 工作流
4. 多环境管理

### 第五周：云原生
1. 12-Factor App
2. Service Mesh 基础
3. 云原生模式
4. 可观测性

### 第六周：实战
1. 完整项目实践
2. 性能优化
3. 故障演练
4. 面试准备

---

## 📊 技术雷达

```
                        评估                采用
                          ↑                  ↑
  ┌───────────────────────┼──────────────────┤
  │         KEDA          │    ArgoCD        │ ← 试验
  │     Crossplane        │    Istio         │
  ├───────────────────────┼──────────────────┤
  │       Pulumi          │  Kubernetes      │ ← 评估
  │       Flux            │  Terraform       │
  ├───────────────────────┼──────────────────┤
  │                       │  GitHub Actions  │ ← 采用
  │                       │  Docker          │
  └───────────────────────┴──────────────────┘
           暂缓                 采用
```

---

**上一模块**: [10-aws（AWS）](../10-aws/README.md)  
**下一模块**: [12-monitoring-logging（监控与日志）](../12-monitoring-logging/README.md)
