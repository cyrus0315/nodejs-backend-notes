# GitOps 实践

GitOps 是一种使用 Git 作为单一事实源来管理基础设施和应用部署的运维模式。本文深入讲解 GitOps 原理、ArgoCD 实践和最佳实践。

## 目录
- [GitOps 概述](#gitops-概述)
- [ArgoCD](#argocd)
- [Flux](#flux)
- [GitOps 工作流](#gitops-工作流)
- [高级模式](#高级模式)
- [最佳实践](#最佳实践)
- [常见面试题](#常见面试题)

---

## GitOps 概述

### 什么是 GitOps？

```
传统 CI/CD（Push 模式）           GitOps（Pull 模式）
┌────────────────────────┐      ┌────────────────────────┐
│ 开发者提交代码           │      │ 开发者提交代码           │
│        ↓               │      │        ↓               │
│ CI 构建和测试           │      │ CI 构建和测试           │
│        ↓               │      │        ↓               │
│ CI 推送到集群           │      │ 更新 Git 仓库配置        │
│   (kubectl apply)      │      │        ↓               │
│        ↓               │      │ GitOps Operator 检测    │
│ 集群状态改变            │      │        ↓               │
└────────────────────────┘      │ 自动同步到集群           │
                                │        ↓               │
                                │ 集群状态改变            │
                                └────────────────────────┘

问题：                           优势：
- CI 需要集群访问权限             - Git 是唯一事实源
- 难以追溯变更                   - 完整审计日志
- 手动回滚复杂                   - Git revert 即回滚
- 状态漂移难以检测               - 自动漂移检测和修复
```

### GitOps 四大原则

1. **声明式（Declarative）**
   - 系统状态以声明式方式描述
   - YAML/JSON 配置文件

2. **版本控制（Versioned）**
   - Git 作为唯一事实源
   - 所有变更可追溯

3. **自动拉取（Automatically Pulled）**
   - GitOps Agent 自动检测 Git 变更
   - 拉取并应用到集群

4. **持续协调（Continuously Reconciled）**
   - 持续比较期望状态和实际状态
   - 自动修复漂移

### GitOps 工具对比

| 特性 | ArgoCD | Flux | Jenkins X |
|------|--------|------|-----------|
| 界面 | Web UI | CLI | Web UI |
| 多集群 | ✅ 原生支持 | ✅ 支持 | ✅ 支持 |
| Helm | ✅ | ✅ | ✅ |
| Kustomize | ✅ | ✅ | ⚠️ |
| 通知 | ✅ | ✅ | ✅ |
| RBAC | ✅ 丰富 | ✅ | ✅ |
| 学习曲线 | 中等 | 低 | 高 |
| 社区 | 大 | 大 | 中 |

---

## ArgoCD

### 安装 ArgoCD

```bash
# 创建 namespace
kubectl create namespace argocd

# 安装 ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 等待就绪
kubectl wait --for=condition=available --timeout=600s deployment/argocd-server -n argocd

# 获取初始密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 端口转发访问 UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 安装 CLI
brew install argocd

# 登录
argocd login localhost:8080
```

### Application 定义

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nodejs-app
  namespace: argocd
  # 自动清理
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  # 源：Git 仓库
  source:
    repoURL: https://github.com/myorg/nodejs-app-config.git
    targetRevision: main
    path: overlays/production
    
    # Kustomize 配置
    kustomize:
      images:
        - ghcr.io/myorg/nodejs-app:v1.0.0
    
    # 或 Helm 配置
    # helm:
    #   releaseName: nodejs-app
    #   valueFiles:
    #     - values-production.yaml
    #   parameters:
    #     - name: image.tag
    #       value: v1.0.0
  
  # 目标：K8s 集群
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # 同步策略
  syncPolicy:
    # 自动同步
    automated:
      prune: true       # 删除不在 Git 中的资源
      selfHeal: true    # 自动修复漂移
      allowEmpty: false # 不允许空应用
    
    # 同步选项
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    
    # 重试策略
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # 忽略差异
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA 管理的副本数
```

### ApplicationSet（多环境/多集群）

```yaml
# applicationset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nodejs-app
  namespace: argocd
spec:
  generators:
    # 列表生成器
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster.example.com
            namespace: development
          - cluster: staging
            url: https://staging-cluster.example.com
            namespace: staging
          - cluster: prod
            url: https://prod-cluster.example.com
            namespace: production
    
    # 或 Git 目录生成器
    # - git:
    #     repoURL: https://github.com/myorg/nodejs-app-config.git
    #     revision: main
    #     directories:
    #       - path: overlays/*
  
  template:
    metadata:
      name: 'nodejs-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/nodejs-app-config.git
        targetRevision: main
        path: 'overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true

---
# 使用 Cluster Generator（自动发现集群）
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nodejs-app-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: 'nodejs-app-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/nodejs-app-config.git
        targetRevision: main
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: production
```

### 项目（Project）

```yaml
# project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: nodejs-apps
  namespace: argocd
spec:
  description: Node.js Applications
  
  # 允许的源仓库
  sourceRepos:
    - 'https://github.com/myorg/*'
    - 'https://charts.helm.sh/stable'
  
  # 允许的目标集群和命名空间
  destinations:
    - namespace: 'development'
      server: https://kubernetes.default.svc
    - namespace: 'staging'
      server: https://kubernetes.default.svc
    - namespace: 'production'
      server: https://kubernetes.default.svc
  
  # 允许的资源类型
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: 'rbac.authorization.k8s.io'
      kind: ClusterRole
    - group: 'rbac.authorization.k8s.io'
      kind: ClusterRoleBinding
  
  # 命名空间资源黑名单
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
    - group: ''
      kind: LimitRange
  
  # 角色
  roles:
    - name: developer
      description: Developer access
      policies:
        - p, proj:nodejs-apps:developer, applications, get, nodejs-apps/*, allow
        - p, proj:nodejs-apps:developer, applications, sync, nodejs-apps/*, allow
      groups:
        - developers
    
    - name: admin
      description: Admin access
      policies:
        - p, proj:nodejs-apps:admin, applications, *, nodejs-apps/*, allow
      groups:
        - platform-team
```

### Sync Waves 和 Hooks

```yaml
# 使用 Sync Waves 控制部署顺序
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    argocd.argoproj.io/sync-wave: "-2"  # 先创建

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # 然后创建配置

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # 最后部署应用

---
# Pre-Sync Hook（同步前执行）
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migration
          image: myorg/nodejs-app:v1.0.0
          command: ["npm", "run", "db:migrate"]
      restartPolicy: Never

---
# Post-Sync Hook（同步后执行）
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: test
          image: myorg/nodejs-app:v1.0.0
          command: ["npm", "run", "test:smoke"]
      restartPolicy: Never
```

### 通知

```yaml
# argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack 服务
  service.slack: |
    token: $slack-token

  # 触发器
  trigger.on-sync-succeeded: |
    - when: app.status.sync.status == 'Synced'
      send: [app-sync-succeeded]

  trigger.on-sync-failed: |
    - when: app.status.sync.status == 'Failed'
      send: [app-sync-failed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  # 模板
  template.app-sync-succeeded: |
    slack:
      attachments: |
        [{
          "color": "#18be52",
          "title": "{{.app.metadata.name}} synced successfully",
          "text": "Application {{.app.metadata.name}} is now synced.\nRevision: {{.app.status.sync.revision}}"
        }]

  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "color": "#ff0000",
          "title": "{{.app.metadata.name}} sync failed",
          "text": "Application {{.app.metadata.name}} failed to sync.\n{{.app.status.conditions}}"
        }]

  # 订阅
  subscriptions: |
    - recipients:
        - slack:deployments
      triggers:
        - on-sync-succeeded
        - on-sync-failed
        - on-health-degraded
```

### ArgoCD CLI 常用命令

```bash
# 登录
argocd login argocd.example.com --grpc-web

# 查看应用
argocd app list
argocd app get nodejs-app

# 同步应用
argocd app sync nodejs-app
argocd app sync nodejs-app --prune --force

# 回滚
argocd app rollback nodejs-app <revision>

# 查看历史
argocd app history nodejs-app

# 差异
argocd app diff nodejs-app

# 删除应用
argocd app delete nodejs-app

# 集群管理
argocd cluster list
argocd cluster add my-cluster --name production

# 仓库管理
argocd repo add https://github.com/myorg/config.git \
  --username myuser \
  --password mypass
```

---

## Flux

### 安装 Flux

```bash
# 安装 CLI
brew install fluxcd/tap/flux

# 检查前置条件
flux check --pre

# 引导安装（GitHub）
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --personal
```

### GitRepository 和 Kustomization

```yaml
# flux-system/gotk-sync.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: nodejs-app
  namespace: flux-system
spec:
  interval: 1m0s
  url: https://github.com/myorg/nodejs-app-config
  ref:
    branch: main
  secretRef:
    name: flux-system

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: nodejs-app
  namespace: flux-system
spec:
  interval: 10m0s
  targetNamespace: production
  sourceRef:
    kind: GitRepository
    name: nodejs-app
  path: ./overlays/production
  prune: true
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: nodejs-app
      namespace: production
```

### HelmRelease

```yaml
# helm-release.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 1h0m0s
  url: https://charts.bitnami.com/bitnami

---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: postgresql
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: postgresql
      version: "12.x"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: flux-system
  values:
    auth:
      postgresPassword: ${POSTGRES_PASSWORD}
    primary:
      persistence:
        size: 20Gi
  valuesFrom:
    - kind: Secret
      name: postgresql-values
      valuesKey: values.yaml
```

### 镜像自动更新

```yaml
# image-repository.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: nodejs-app
  namespace: flux-system
spec:
  image: ghcr.io/myorg/nodejs-app
  interval: 1m0s
  secretRef:
    name: ghcr-credentials

---
# image-policy.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: nodejs-app
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: nodejs-app
  policy:
    semver:
      range: 1.x.x

---
# image-update.yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
        name: Flux
      messageTemplate: 'Update image to {{.NewTag}}'
    push:
      branch: main
  update:
    path: ./clusters/production
    strategy: Setters
```

---

## GitOps 工作流

### 仓库结构

```
# 应用代码仓库
nodejs-app/
├── src/
├── Dockerfile
├── package.json
└── .github/
    └── workflows/
        └── ci.yaml

# 配置仓库（GitOps）
nodejs-app-config/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── kustomization.yaml
│   └── configmap.yaml
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── patch-replicas.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── patch-replicas.yaml
│       └── patch-resources.yaml
└── argocd/
    ├── application.yaml
    └── project.yaml
```

### Kustomize 配置

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app: nodejs-app

---
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
        - name: app
          image: ghcr.io/myorg/nodejs-app:latest
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: nodejs-app-config
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

images:
  - name: ghcr.io/myorg/nodejs-app
    newTag: v1.2.3

replicas:
  - name: nodejs-app
    count: 5

patches:
  - path: patch-resources.yaml

---
# overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  template:
    spec:
      containers:
        - name: app
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

### CI 流水线（更新配置仓库）

```yaml
# .github/workflows/ci.yaml（应用仓库）
name: CI/CD

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  update-config:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref_type == 'tag'
    
    steps:
      - name: Checkout config repo
        uses: actions/checkout@v3
        with:
          repository: myorg/nodejs-app-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}
      
      - name: Update image tag
        run: |
          cd overlays/production
          kustomize edit set image \
            ghcr.io/myorg/nodejs-app=ghcr.io/myorg/nodejs-app:${{ needs.build.outputs.image-tag }}
      
      - name: Commit and push
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Update image to ${{ needs.build.outputs.image-tag }}"
          git push
```

### 渐进式发布（Progressive Delivery）

```yaml
# Argo Rollouts 配置
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: nodejs-app
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
        - name: app
          image: ghcr.io/myorg/nodejs-app:v1.0.0
          ports:
            - containerPort: 3000
  strategy:
    canary:
      # 金丝雀步骤
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - setWeight: 30
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: { duration: 5m }
        - setWeight: 100
      
      # 自动分析
      analysis:
        templates:
          - templateName: success-rate
        args:
          - name: service-name
            value: nodejs-app
      
      # 流量路由（Istio）
      trafficRouting:
        istio:
          virtualService:
            name: nodejs-app-vsvc
            routes:
              - primary
          destinationRule:
            name: nodejs-app-destrule
            canarySubsetName: canary
            stableSubsetName: stable

---
# 分析模板
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(
              http_requests_total{service="{{args.service-name}}", status=~"2.*"}[5m]
            )) / sum(rate(
              http_requests_total{service="{{args.service-name}}"}[5m]
            ))
```

---

## 高级模式

### 多租户 GitOps

```yaml
# 租户隔离
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: tenant-a
  namespace: argocd
spec:
  description: Tenant A Applications
  sourceRepos:
    - 'https://github.com/tenant-a/*'
  destinations:
    - namespace: 'tenant-a-*'
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: ''
      kind: '*'
    - group: 'apps'
      kind: '*'
  clusterResourceWhitelist: []  # 禁止集群级资源
  roles:
    - name: admin
      policies:
        - p, proj:tenant-a:admin, applications, *, tenant-a/*, allow
      groups:
        - tenant-a-admins
```

### 秘钥管理（Sealed Secrets）

```bash
# 安装 Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 安装 kubeseal CLI
brew install kubeseal

# 创建加密的 Secret
kubectl create secret generic db-credentials \
  --dry-run=client \
  --from-literal=username=admin \
  --from-literal=password=secret \
  -o yaml | kubeseal -o yaml > sealed-secret.yaml
```

```yaml
# sealed-secret.yaml（可以安全提交到 Git）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO...
    password: AgC5Gm0xwPXz+PiTySYZZA9rO...
```

### External Secrets Operator

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: db-credentials
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: production/nodejs-app/db

---
# cluster-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

### 策略即代码（Policy as Code）

```yaml
# Kyverno 策略
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  validationFailureAction: enforce
  rules:
    - name: require-app-label
      match:
        resources:
          kinds:
            - Deployment
            - StatefulSet
      validate:
        message: "The label 'app' is required."
        pattern:
          metadata:
            labels:
              app: "?*"

---
# 镜像策略
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: enforce
  rules:
    - name: validate-registries
      match:
        resources:
          kinds:
            - Pod
      validate:
        message: "Images must be from approved registries."
        pattern:
          spec:
            containers:
              - image: "ghcr.io/myorg/* | gcr.io/my-project/*"
```

---

## 最佳实践

### 1. 仓库策略

```
# 推荐：每环境一个分支或目录
config-repo/
├── base/                 # 基础配置
├── overlays/
│   ├── development/     # 开发环境
│   ├── staging/         # 预发布环境
│   └── production/      # 生产环境
└── argocd/              # ArgoCD 配置

# 或使用分支
main        -> production
staging     -> staging
development -> development
```

### 2. 变更流程

```yaml
# 开发流程
1. 开发者提交代码到应用仓库
2. CI 构建并推送镜像
3. CI 创建 PR 更新配置仓库
4. 审核 PR（自动/手动）
5. 合并 PR
6. ArgoCD 自动同步

# PR 自动化
name: Update Config

on:
  workflow_dispatch:
    inputs:
      image-tag:
        description: 'Image tag to deploy'
        required: true
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          repository: myorg/nodejs-app-config
          token: ${{ secrets.CONFIG_REPO_TOKEN }}
      
      - name: Update image
        run: |
          cd overlays/${{ inputs.environment }}
          kustomize edit set image \
            ghcr.io/myorg/nodejs-app=ghcr.io/myorg/nodejs-app:${{ inputs.image-tag }}
      
      - name: Create PR
        uses: peter-evans/create-pull-request@v5
        with:
          title: "Deploy ${{ inputs.image-tag }} to ${{ inputs.environment }}"
          branch: "deploy/${{ inputs.environment }}/${{ inputs.image-tag }}"
          commit-message: "Update image to ${{ inputs.image-tag }}"
```

### 3. 灾难恢复

```bash
# 回滚到上一个版本
git revert HEAD
git push

# 或使用 ArgoCD
argocd app rollback nodejs-app <revision>

# 查看历史
argocd app history nodejs-app

# 恢复到特定版本
argocd app sync nodejs-app --revision <commit-sha>
```

### 4. 监控和告警

```yaml
# Prometheus 规则
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: argocd-alerts
  namespace: monitoring
spec:
  groups:
    - name: argocd
      rules:
        - alert: ArgoAppOutOfSync
          expr: argocd_app_info{sync_status="OutOfSync"} == 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Application {{ $labels.name }} is out of sync"
        
        - alert: ArgoAppDegraded
          expr: argocd_app_info{health_status="Degraded"} == 1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Application {{ $labels.name }} is degraded"
```

---

## 常见面试题

### 1. 什么是 GitOps？它的核心原则是什么？

**定义**：
GitOps 是一种使用 Git 作为单一事实源，通过声明式配置来管理基础设施和应用部署的运维模式。

**核心原则**：
1. **声明式**：系统状态以声明式方式描述
2. **版本控制**：所有配置存储在 Git 中
3. **自动拉取**：Agent 自动检测并应用变更
4. **持续协调**：持续确保实际状态与期望状态一致

### 2. GitOps 与传统 CI/CD 的区别？

| 方面 | 传统 CI/CD | GitOps |
|------|-----------|--------|
| 模式 | Push | Pull |
| 权限 | CI 需要集群权限 | Agent 在集群内 |
| 审计 | 需额外配置 | Git 历史即审计 |
| 回滚 | 复杂 | Git revert |
| 漂移检测 | 手动 | 自动 |

### 3. 如何处理敏感数据？

**方案**：
1. **Sealed Secrets**：加密后存储在 Git
2. **External Secrets**：从外部密钥管理服务拉取
3. **SOPS**：加密 YAML 文件
4. **Vault**：HashiCorp Vault 集成

### 4. 如何实现多环境部署？

**方案**：
1. **Kustomize Overlays**：base + overlays/env
2. **Helm Values**：不同环境使用不同 values 文件
3. **ArgoCD ApplicationSet**：自动生成多环境应用
4. **分支策略**：不同分支对应不同环境

---

## 总结

### GitOps 检查清单

- [ ] Git 仓库作为唯一事实源
- [ ] 声明式配置（Kustomize/Helm）
- [ ] 自动同步工具（ArgoCD/Flux）
- [ ] 敏感数据加密存储
- [ ] PR 审核流程
- [ ] 自动化回滚机制
- [ ] 监控和告警

### 关键要点

1. **Git 是唯一事实源**
2. **Pull 模式比 Push 更安全**
3. **声明式优于命令式**
4. **自动化一切可自动化的**
5. **完善的审计和可追溯性**

---

**上一篇**：[Infrastructure as Code](./04-infrastructure-as-code.md)  
**下一篇**：[Cloud Native](./06-cloud-native.md)

