# Docker 容器化

Docker 是容器化应用的标准工具，本文专注于 Node.js 应用的 Docker 最佳实践。

## Docker 基础

### 核心概念

```bash
# 镜像（Image）：只读模板
# 容器（Container）：镜像的运行实例
# Dockerfile：构建镜像的配置文件
# Docker Hub：镜像仓库
# Volume：持久化数据
```

### Docker vs 虚拟机

| 特性 | Docker | 虚拟机 |
|------|--------|--------|
| 启动速度 | 秒级 | 分钟级 |
| 资源占用 | 小（MB） | 大（GB） |
| 隔离性 | 进程级 | 系统级 |
| 性能 | 接近原生 | 有损耗 |
| 移植性 | 强 | 一般 |

## Dockerfile 编写

### 基础 Dockerfile

```dockerfile
# Node.js 应用基础 Dockerfile
FROM node:18-alpine

# 设置工作目录
WORKDIR /app

# 复制 package 文件
COPY package*.json ./

# 安装依赖
RUN npm ci --only=production

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 3000

# 启动应用
CMD ["node", "dist/index.js"]
```

### 多阶段构建（最佳实践）

```dockerfile
# 构建阶段
FROM node:18-alpine AS builder

WORKDIR /app

# 复制依赖文件
COPY package*.json ./
COPY tsconfig.json ./

# 安装所有依赖（包括 devDependencies）
RUN npm ci

# 复制源代码
COPY src ./src

# 构建应用
RUN npm run build

# 生产阶段
FROM node:18-alpine AS production

WORKDIR /app

# 只安装生产依赖
COPY package*.json ./
RUN npm ci --only=production

# 从构建阶段复制编译后的文件
COPY --from=builder /app/dist ./dist

# 创建非 root 用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 切换到非 root 用户
USER nodejs

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### NestJS 应用 Dockerfile

```dockerfile
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
COPY tsconfig*.json ./

RUN npm ci

COPY src ./src

RUN npm run build

# 生产阶段
FROM node:18-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs

EXPOSE 3000

CMD ["node", "dist/main.js"]
```

### 优化的 Dockerfile

```dockerfile
# 使用特定版本，避免使用 latest
FROM node:18.17.0-alpine3.18 AS base

# 安装必要的系统依赖
RUN apk add --no-cache dumb-init

WORKDIR /app

# 利用 Docker 缓存层
COPY package*.json ./

# 开发阶段
FROM base AS development
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]

# 构建阶段
FROM base AS builder
RUN npm ci
COPY . .
RUN npm run build && npm prune --production

# 生产阶段
FROM base AS production

# 复制生产依赖和构建产物
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package*.json ./

# 安全：使用非 root 用户
USER node

# 使用 dumb-init 处理信号
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

## 镜像优化

### 1. 减小镜像大小

```dockerfile
# ❌ 大镜像（~1GB）
FROM node:18

# ✅ 小镜像（~170MB）
FROM node:18-alpine

# ✅ 更小（~100MB）
FROM node:18-alpine AS builder
# ... build
FROM node:18-alpine AS production
# 只复制必需文件
```

### 2. .dockerignore

```bash
# .dockerignore
node_modules
npm-debug.log
dist
build
.git
.gitignore
README.md
.env
.env.*
coverage
.vscode
.idea
*.md
.DS_Store
```

### 3. 层缓存优化

```dockerfile
# ❌ 代码修改会重新安装依赖
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN npm ci

# ✅ 先复制 package.json，利用缓存
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
```

### 4. 合并 RUN 指令

```dockerfile
# ❌ 多层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ 单层
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

## Docker Compose

### 基础配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 开发环境配置

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app
      - /app/node_modules  # 避免覆盖
    command: npm run dev
    environment:
      - NODE_ENV=development
    ports:
      - "3000:3000"
      - "9229:9229"  # 调试端口

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb_dev
    ports:
      - "5432:5432"
    volumes:
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### 完整示例（带健康检查）

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        - NODE_ENV=production
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

## 容器管理

### 常用命令

```bash
# 构建镜像
docker build -t my-app:latest .

# 多阶段构建特定阶段
docker build --target development -t my-app:dev .

# 运行容器
docker run -d -p 3000:3000 --name my-app my-app:latest

# 查看容器日志
docker logs my-app
docker logs -f my-app  # 实时

# 进入容器
docker exec -it my-app sh

# 停止和删除
docker stop my-app
docker rm my-app

# 清理
docker system prune -a  # 清理所有未使用的资源
```

### Docker Compose 命令

```bash
# 启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f app

# 重启服务
docker-compose restart app

# 停止服务
docker-compose stop

# 停止并删除
docker-compose down

# 删除包括卷
docker-compose down -v

# 查看状态
docker-compose ps

# 执行命令
docker-compose exec app sh

# 构建并启动
docker-compose up --build -d
```

## 数据持久化

### Volume 使用

```yaml
services:
  db:
    image: postgres:15-alpine
    volumes:
      # 命名卷
      - postgres_data:/var/lib/postgresql/data
      # 绑定挂载
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      # 匿名卷
      - /var/lib/postgresql/data

volumes:
  postgres_data:
    driver: local
```

### 备份和恢复

```bash
# 备份 PostgreSQL
docker-compose exec db pg_dump -U postgres mydb > backup.sql

# 恢复
docker-compose exec -T db psql -U postgres mydb < backup.sql

# 备份 Volume
docker run --rm -v postgres_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/postgres_backup.tar.gz -C /data .

# 恢复 Volume
docker run --rm -v postgres_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/postgres_backup.tar.gz -C /data
```

## 网络配置

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    image: my-frontend
    networks:
      - frontend-network

  backend:
    image: my-backend
    networks:
      - frontend-network
      - backend-network

  db:
    image: postgres
    networks:
      - backend-network

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
    internal: true  # 隔离网络，无法访问外网
```

## 安全最佳实践

### 1. 使用非 root 用户

```dockerfile
# 创建用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# 修改文件所有权
RUN chown -R nodejs:nodejs /app

# 切换用户
USER nodejs
```

### 2. 最小化基础镜像

```dockerfile
# ✅ 使用 Alpine（更小、更安全）
FROM node:18-alpine

# ✅ 使用 distroless（最小化）
FROM gcr.io/distroless/nodejs18-debian11
```

### 3. 扫描漏洞

```bash
# 使用 Trivy 扫描
trivy image my-app:latest

# 使用 Snyk
snyk container test my-app:latest
```

### 4. 不在镜像中存储敏感信息

```dockerfile
# ❌ 不要这样做
ENV API_KEY=secret123

# ✅ 使用运行时注入
# docker run -e API_KEY=secret123 my-app
```

### 5. 使用 .dockerignore

确保不将敏感文件打包进镜像：

```bash
.env
.env.*
*.pem
*.key
secrets/
```

## 调试和监控

### 调试容器

```bash
# 查看容器日志
docker logs -f my-app

# 进入容器
docker exec -it my-app sh

# 查看容器资源使用
docker stats my-app

# 查看容器详情
docker inspect my-app
```

### 健康检查

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node healthcheck.js
```

```javascript
// healthcheck.js
const http = require('http');

const options = {
  host: 'localhost',
  port: 3000,
  path: '/health',
  timeout: 2000
};

const request = http.request(options, (res) => {
  if (res.statusCode === 200) {
    process.exit(0);
  } else {
    process.exit(1);
  }
});

request.on('error', () => {
  process.exit(1);
});

request.end();
```

## 常见面试题

### 1. Docker 容器与虚拟机的区别？

<details>
<summary>点击查看答案</summary>

**架构差异**：
- **虚拟机**：硬件虚拟化，每个 VM 有完整 OS
- **容器**：操作系统级虚拟化，共享宿主机内核

**性能对比**：

| 方面 | 容器 | 虚拟机 |
|------|------|--------|
| 启动 | 秒级 | 分钟级 |
| 大小 | MB | GB |
| 性能 | 接近原生 | 有开销 |
| 隔离 | 进程级 | 系统级 |
| 移植性 | 强 | 弱 |

**使用场景**：
- 微服务架构 → 容器
- 需要强隔离 → 虚拟机
- 快速部署 → 容器
- 不同操作系统 → 虚拟机
</details>

### 2. 如何优化 Docker 镜像大小？

<details>
<summary>点击查看答案</summary>

**优化策略**：

1. **使用小基础镜像**：
   - `node:18-alpine`（~170MB）
   - `node:18-slim`（~250MB）
   - distroless（最小）

2. **多阶段构建**：
```dockerfile
FROM node:18-alpine AS builder
RUN npm ci && npm run build

FROM node:18-alpine AS production
COPY --from=builder /app/dist ./dist
```

3. **清理无用文件**：
```dockerfile
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/*
```

4. **合并 RUN 指令**

5. **使用 .dockerignore**

**效果对比**：
- 未优化：~1.2GB
- 基础优化：~400MB
- 多阶段 + Alpine：~150MB
</details>

### 3. Docker 层缓存如何工作？

<details>
<summary>点击查看答案</summary>

**工作原理**：
- Docker 逐行执行 Dockerfile
- 每条指令创建新层
- 如果指令和内容未变，复用缓存

**优化示例**：
```dockerfile
# ❌ 代码改变会重装依赖
COPY . .
RUN npm ci

# ✅ 先复制 package.json，依赖不变时用缓存
COPY package*.json ./
RUN npm ci
COPY . .
```

**关键点**：
- 把不常变的指令放前面
- 把常变的指令放后面
- 合理组织 COPY 顺序
</details>

### 4. Docker Compose vs Kubernetes？

<details>
<summary>点击查看答案</summary>

| 特性 | Docker Compose | Kubernetes |
|------|----------------|------------|
| 复杂度 | 简单 | 复杂 |
| 适用场景 | 单机 | 集群 |
| 扩展性 | 有限 | 强 |
| 自动恢复 | 无 | 有 |
| 负载均衡 | 基础 | 高级 |
| 学习曲线 | 平缓 | 陡峭 |

**选择**：
- 开发环境 → Docker Compose
- 小项目 → Docker Compose
- 生产环境 → Kubernetes
- 微服务集群 → Kubernetes
</details>

### 5. 如何处理容器中的数据持久化？

<details>
<summary>点击查看答案</summary>

**三种方式**：

1. **Volume**（推荐）：
```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

2. **Bind Mount**：
```yaml
volumes:
  - ./data:/app/data
```

3. **tmpfs**（临时）：
```yaml
tmpfs:
  - /tmp
```

**对比**：

| 类型 | 优点 | 缺点 |
|------|------|------|
| Volume | Docker 管理，跨平台 | 不易直接访问 |
| Bind Mount | 直接访问 | 平台相关 |
| tmpfs | 快速 | 不持久 |

**最佳实践**：
- 数据库数据 → Volume
- 配置文件 → Bind Mount
- 临时文件 → tmpfs
</details>

## 总结

Docker 关键点：
1. 多阶段构建优化镜像
2. 使用 Alpine 基础镜像
3. 合理利用层缓存
4. 安全：非 root 用户
5. Docker Compose 管理多容器
6. 正确处理数据持久化

