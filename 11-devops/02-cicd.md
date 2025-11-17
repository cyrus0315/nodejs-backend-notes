# CI/CD 持续集成与部署

CI/CD 是现代软件开发的核心实践，本文专注于 Node.js 项目的 CI/CD 实现。

## CI/CD 概念

### 持续集成（CI）

**定义**：频繁地将代码集成到主分支，自动执行构建和测试。

**核心实践**：
- 频繁提交代码
- 自动构建
- 自动测试
- 快速反馈

### 持续交付（CD）

**定义**：确保代码随时可以部署到生产环境。

**核心实践**：
- 自动部署到测试环境
- 手动批准部署到生产
- 自动回滚

### 持续部署（CD）

**定义**：自动部署到生产环境，无需人工干预。

## GitHub Actions

### 基础工作流

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Build
        run: npm run build
```

### 完整的 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: 18.x
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 代码质量检查
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check formatting
        run: npm run format:check

  # 测试
  test:
    runs-on: ubuntu-latest
    needs: lint
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
      
      - name: Generate coverage
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  # 构建 Docker 镜像
  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
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
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

  # 部署到 AWS ECS
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: app
          image: ${{ steps.login-ecr.outputs.registry }}/my-app:${{ github.sha }}
      
      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: my-app-service
          cluster: my-cluster
          wait-for-service-stability: true
```

### 多环境部署

```yaml
# .github/workflows/deploy-multi-env.yml
name: Multi-Environment Deployment

on:
  push:
    branches:
      - develop
      - staging
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set environment
        id: set-env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "ENVIRONMENT=production" >> $GITHUB_OUTPUT
            echo "AWS_ROLE=${{ secrets.AWS_ROLE_PRODUCTION }}" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/staging" ]]; then
            echo "ENVIRONMENT=staging" >> $GITHUB_OUTPUT
            echo "AWS_ROLE=${{ secrets.AWS_ROLE_STAGING }}" >> $GITHUB_OUTPUT
          else
            echo "ENVIRONMENT=development" >> $GITHUB_OUTPUT
            echo "AWS_ROLE=${{ secrets.AWS_ROLE_DEVELOPMENT }}" >> $GITHUB_OUTPUT
          fi
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ steps.set-env.outputs.AWS_ROLE }}
          aws-region: us-east-1
      
      - name: Deploy to ${{ steps.set-env.outputs.ENVIRONMENT }}
        run: |
          echo "Deploying to ${{ steps.set-env.outputs.ENVIRONMENT }}"
          # 部署逻辑
```

### 缓存优化

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'  # 自动缓存 npm
      
      # 或手动缓存
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      - name: Cache build output
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
      
      - run: npm ci
      - run: npm run build
```

## 部署策略

### 1. 蓝绿部署（Blue-Green Deployment）

```yaml
# 蓝绿部署策略
name: Blue-Green Deployment

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to Green environment
        run: |
          # 部署新版本到 Green 环境
          aws ecs update-service \
            --cluster my-cluster \
            --service my-app-green \
            --task-definition my-app:${{ github.sha }}
      
      - name: Wait for Green to be stable
        run: |
          aws ecs wait services-stable \
            --cluster my-cluster \
            --services my-app-green
      
      - name: Run smoke tests
        run: |
          # 对 Green 环境运行测试
          npm run test:smoke -- --url https://green.example.com
      
      - name: Switch traffic to Green
        run: |
          # 切换流量到 Green
          aws elbv2 modify-rule \
            --rule-arn ${{ secrets.ALB_RULE_ARN }} \
            --actions Type=forward,TargetGroupArn=${{ secrets.GREEN_TG_ARN }}
      
      - name: Update Blue as backup
        run: |
          # Blue 环境保留作为回滚备份
          echo "Blue environment kept for rollback"
```

### 2. 金丝雀部署（Canary Deployment）

```yaml
# 金丝雀部署
name: Canary Deployment

jobs:
  canary:
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy canary (10% traffic)
        run: |
          # 部署到 10% 实例
          aws ecs update-service \
            --cluster my-cluster \
            --service my-app-canary \
            --desired-count 1 \
            --task-definition my-app:${{ github.sha }}
      
      - name: Monitor canary for 10 minutes
        run: |
          sleep 600
          # 监控指标
          ERROR_RATE=$(aws cloudwatch get-metric-statistics ...)
          if [ $ERROR_RATE -gt 1 ]; then
            echo "Canary failed, rolling back"
            exit 1
          fi
      
      - name: Increase to 50% traffic
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-app-canary \
            --desired-count 5
      
      - name: Monitor for another 10 minutes
        run: |
          sleep 600
          # 再次检查指标
      
      - name: Full rollout
        run: |
          # 部署到所有实例
          aws ecs update-service \
            --cluster my-cluster \
            --service my-app \
            --task-definition my-app:${{ github.sha }}
```

### 3. 滚动更新（Rolling Update）

```yaml
# 滚动更新
name: Rolling Update

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy with rolling update
        run: |
          # ECS 自动滚动更新
          aws ecs update-service \
            --cluster my-cluster \
            --service my-app \
            --task-definition my-app:${{ github.sha }} \
            --deployment-configuration \
              "maximumPercent=200,minimumHealthyPercent=100"
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster my-cluster \
            --services my-app
```

## Secrets 管理

### GitHub Secrets

```yaml
# 使用 GitHub Secrets
jobs:
  deploy:
    steps:
      - name: Use secrets
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          # 使用环境变量
          echo "Deploying with secrets"
```

### AWS Secrets Manager 集成

```yaml
jobs:
  deploy:
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Get secrets from AWS
        id: secrets
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id production/my-app \
            --query SecretString \
            --output text)
          echo "::add-mask::$SECRET"
          echo "DB_PASSWORD=$(echo $SECRET | jq -r .db_password)" >> $GITHUB_OUTPUT
      
      - name: Deploy with secrets
        env:
          DB_PASSWORD: ${{ steps.secrets.outputs.DB_PASSWORD }}
        run: |
          # 使用密钥部署
```

## 自动化测试

### E2E 测试集成

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Start services
        run: docker-compose up -d
      
      - name: Wait for services
        run: |
          timeout 60 sh -c 'until curl -f http://localhost:3000/health; do sleep 1; done'
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-results
          path: test-results/
      
      - name: Cleanup
        if: always()
        run: docker-compose down -v
```

## 性能测试

```yaml
jobs:
  performance:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to staging
        run: |
          # 部署到 staging
      
      - name: Run k6 load test
        uses: grafana/k6-action@v0.3.0
        with:
          filename: tests/load-test.js
          cloud: true
          token: ${{ secrets.K6_CLOUD_TOKEN }}
      
      - name: Check performance thresholds
        run: |
          # 检查性能指标是否满足要求
          if [ $P95_LATENCY -gt 500 ]; then
            echo "Performance degradation detected"
            exit 1
          fi
```

## 回滚策略

```yaml
# 手动触发回滚
name: Rollback

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to'
        required: true
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  rollback:
    runs-on: ubuntu-latest
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Rollback to version ${{ inputs.version }}
        run: |
          aws ecs update-service \
            --cluster ${{ inputs.environment }}-cluster \
            --service my-app \
            --task-definition my-app:${{ inputs.version }} \
            --force-new-deployment
      
      - name: Wait for rollback
        run: |
          aws ecs wait services-stable \
            --cluster ${{ inputs.environment }}-cluster \
            --services my-app
      
      - name: Verify rollback
        run: |
          # 验证回滚成功
          npm run test:smoke -- --env ${{ inputs.environment }}
```

## 最佳实践

### 1. Pipeline 设计

```yaml
# 完整 pipeline 示例
name: Production Pipeline

on:
  push:
    branches: [ main ]

jobs:
  # 1. 代码检查
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run format:check
  
  # 2. 测试
  test:
    needs: quality
    strategy:
      matrix:
        test-type: [unit, integration, e2e]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test:${{ matrix.test-type }}
  
  # 3. 安全扫描
  security:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm audit
      - uses: snyk/actions/node@master
        with:
          args: --severity-threshold=high
  
  # 4. 构建
  build:
    needs: [test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v3
        with:
          name: build
          path: dist/
  
  # 5. 部署
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: build
      - name: Deploy
        run: |
          # 部署逻辑
```

### 2. 环境变量管理

```yaml
# 不同环境使用不同变量
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}
    
    steps:
      - name: Deploy
        env:
          # 自动从 environment 获取 secrets
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          # 部署
```

### 3. 并行执行

```yaml
# 并行运行多个 job
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps: [...]
  
  test-integration:
    runs-on: ubuntu-latest
    steps: [...]
  
  test-e2e:
    runs-on: ubuntu-latest
    steps: [...]
  
  # 等待所有测试完成
  deploy:
    needs: [test-unit, test-integration, test-e2e]
    runs-on: ubuntu-latest
    steps: [...]
```

## 常见面试题

### 1. CI/CD 的好处是什么？

<details>
<summary>点击查看答案</summary>

**主要好处**：

1. **更快的交付速度**：
   - 自动化构建和测试
   - 减少手动操作时间

2. **更高的代码质量**：
   - 自动化测试
   - 早期发现问题

3. **更低的风险**：
   - 小批量频繁发布
   - 快速回滚能力

4. **更好的协作**：
   - 统一的流程
   - 可视化的状态

5. **成本降低**：
   - 减少人工成本
   - 提高效率
</details>

### 2. 蓝绿部署 vs 金丝雀部署？

<details>
<summary>点击查看答案</summary>

**蓝绿部署**：
- ✅ 快速切换和回滚
- ✅ 零停机时间
- ❌ 需要双倍资源
- ❌ 数据库迁移复杂

**金丝雀部署**：
- ✅ 渐进式发布，风险小
- ✅ 节省资源
- ❌ 部署时间长
- ❌ 监控要求高

**选择**：
- 关键系统 → 蓝绿
- 资源有限 → 金丝雀
- 大规模变更 → 金丝雀
- 快速验证 → 蓝绿
</details>

### 3. 如何确保 CI/CD 的安全性？

<details>
<summary>点击查看答案</summary>

**安全措施**：

1. **密钥管理**：
   - 使用 Secrets Manager
   - 不在代码中硬编码
   - 定期轮换

2. **权限控制**：
   - 最小权限原则
   - 使用 IAM 角色
   - 分离环境权限

3. **代码扫描**：
   - 依赖扫描（npm audit）
   - 安全漏洞扫描（Snyk）
   - SAST/DAST

4. **审计日志**：
   - 记录所有部署
   - 可追溯性

5. **环境隔离**：
   - 生产环境需要审批
   - 使用不同的 AWS 账户
</details>

## 总结

CI/CD 关键点：
1. 自动化一切
2. 快速反馈
3. 小批量频繁发布
4. 可靠的回滚机制
5. 完善的监控
6. 安全优先

