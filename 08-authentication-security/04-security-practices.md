# Node.js 安全实践

本文介绍 Node.js 应用的安全实践，包括依赖管理、环境配置、错误处理、日志监控等。

## 目录
- [依赖安全](#依赖安全)
- [环境配置](#环境配置)
- [错误处理](#错误处理)
- [日志与监控](#日志与监控)
- [文件上传安全](#文件上传安全)
- [API 安全](#api-安全)
- [Docker 安全](#docker-安全)
- [安全测试](#安全测试)
- [面试题](#常见面试题)

---

## 依赖安全

### 依赖扫描

```bash
# npm audit（检查漏洞）
npm audit

# 自动修复
npm audit fix

# 强制修复（可能破坏兼容性）
npm audit fix --force

# 查看详情
npm audit --json

# 设置审计级别
npm audit --audit-level=moderate
```

### package-lock.json

```json
// ✅ 锁定依赖版本
{
  "name": "my-app",
  "version": "1.0.0",
  "lockfileVersion": 2,
  "requires": true,
  "packages": {
    "node_modules/express": {
      "version": "4.18.2",  // 精确版本
      "resolved": "https://registry.npmjs.org/express/-/express-4.18.2.tgz",
      "integrity": "sha512-..."
    }
  }
}

// ❌ 不要忽略 package-lock.json
// .gitignore
# package-lock.json  // 不要这样做！
```

### 依赖版本管理

```json
// package.json
{
  "dependencies": {
    // ✅ 推荐：锁定主版本
    "express": "^4.18.2",  // 4.x.x
    
    // ✅ 更严格：锁定次版本
    "mongoose": "~7.0.3",  // 7.0.x
    
    // ⚠️ 谨慎：锁定精确版本
    "lodash": "4.17.21",   // 只用这个版本
    
    // ❌ 不推荐：使用 latest
    "axios": "*"  // 危险！
  }
}
```

### Snyk 集成

```bash
# 安装 Snyk
npm install -g snyk

# 认证
snyk auth

# 测试项目
snyk test

# 监控项目
snyk monitor

# 修复漏洞
snyk wizard
```

```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: test
```

### Dependabot 配置

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    
    # 自动合并补丁更新
    automerged_updates:
      - match:
          dependency_type: "all"
          update_type: "security:patch"
```

---

## 环境配置

### .env 文件

```bash
# .env
NODE_ENV=production
PORT=3000

# 数据库
DATABASE_URL=postgresql://user:password@localhost:5432/mydb

# 密钥
JWT_SECRET=your-secret-key-here-use-long-random-string
ENCRYPTION_KEY=32-byte-hex-key

# 第三方服务
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Redis
REDIS_URL=redis://localhost:6379
```

```typescript
// 使用 dotenv
import dotenv from 'dotenv';
import { z } from 'zod';

// 加载 .env
dotenv.config();

// 环境变量验证
const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  PORT: z.string().transform(Number),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  ENCRYPTION_KEY: z.string().length(64),
  AWS_ACCESS_KEY_ID: z.string(),
  AWS_SECRET_ACCESS_KEY: z.string(),
  REDIS_URL: z.string().url()
});

// 验证
try {
  const env = EnvSchema.parse(process.env);
  export default env;
} catch (error) {
  console.error('Environment validation failed:', error);
  process.exit(1);
}
```

### 敏感信息保护

```bash
# ✅ .gitignore
.env
.env.local
.env.*.local
*.pem
*.key
secrets/

# ❌ 不要提交敏感信息到 Git
```

```typescript
// ✅ 使用环境变量
const secret = process.env.JWT_SECRET;

// ❌ 不要硬编码
const secret = 'my-secret-key'; // 危险！
```

### 密钥管理

```typescript
// 使用 AWS Secrets Manager
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManagerClient({ region: 'us-east-1' });

async function getSecret(secretName: string): Promise<string> {
  const command = new GetSecretValueCommand({ SecretId: secretName });
  const response = await client.send(command);
  return response.SecretString!;
}

// 使用
const dbPassword = await getSecret('prod/db/password');

// 使用 Vault
import vault from 'node-vault';

const client = vault({
  endpoint: 'http://localhost:8200',
  token: process.env.VAULT_TOKEN
});

const secrets = await client.read('secret/data/myapp');
const dbPassword = secrets.data.data.password;
```

---

## 错误处理

### 不暴露错误详情

```typescript
// ❌ 危险：暴露敏感信息
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,
    stack: err.stack,  // 暴露代码结构！
    query: req.query   // 可能包含敏感数据！
  });
});

// ✅ 安全：隐藏详情
app.use((err, req, res, next) => {
  // 记录完整错误
  console.error('Error:', {
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    user: req.user?.id
  });

  // 返回通用错误
  const isDevelopment = process.env.NODE_ENV === 'development';
  
  res.status(err.status || 500).json({
    error: isDevelopment ? err.message : 'Internal server error',
    ...(isDevelopment && { stack: err.stack })
  });
});
```

### 自定义错误类

```typescript
// 错误基类
export class AppError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public isOperational = true
  ) {
    super(message);
    Object.setPrototypeOf(this, AppError.prototype);
  }
}

// 具体错误类
export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(404, `${resource} not found`);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(401, message);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(403, message);
  }
}

export class ValidationError extends AppError {
  constructor(public details: any) {
    super(400, 'Validation failed');
  }
}

// 使用
app.get('/posts/:id', async (req, res, next) => {
  try {
    const post = await prisma.post.findUnique({
      where: { id: parseInt(req.params.id) }
    });

    if (!post) {
      throw new NotFoundError('Post');
    }

    res.json(post);
  } catch (error) {
    next(error);
  }
});

// 错误处理中间件
app.use((err, req, res, next) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.message,
      ...(err instanceof ValidationError && { details: err.details })
    });
  }

  // 未知错误
  console.error('Unexpected error:', err);
  res.status(500).json({
    error: 'Internal server error'
  });
});
```

### 未捕获异常处理

```typescript
// 未捕获的 Promise rejection
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  
  // 记录到监控系统
  // Sentry.captureException(reason);
  
  // 优雅关闭
  process.exit(1);
});

// 未捕获的异常
process.on('uncaughtException', (error) => {
  console.error('Uncaught Exception:', error);
  
  // 记录到监控系统
  // Sentry.captureException(error);
  
  // 立即退出（不要继续运行）
  process.exit(1);
});

// 优雅关闭
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down gracefully');
  
  server.close(() => {
    console.log('Server closed');
    
    // 关闭数据库连接
    prisma.$disconnect();
    
    // 关闭 Redis 连接
    redis.quit();
    
    process.exit(0);
  });
  
  // 强制退出（30 秒超时）
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 30000);
});
```

---

## 日志与监控

### Winston 日志

```typescript
import winston from 'winston';

// 创建 logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'my-app',
    environment: process.env.NODE_ENV
  },
  transports: [
    // 错误日志
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 5
    }),
    
    // 所有日志
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 10
    })
  ]
});

// 开发环境：输出到控制台
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}

export default logger;

// 使用
logger.info('User logged in', { userId: 123 });
logger.warn('Rate limit exceeded', { ip: req.ip });
logger.error('Database error', { error: err.message, stack: err.stack });
```

### 请求日志

```typescript
import morgan from 'morgan';
import logger from './logger';

// 自定义 token
morgan.token('user-id', (req) => req.user?.id || 'anonymous');

// 开发环境
if (process.env.NODE_ENV === 'development') {
  app.use(morgan('dev'));
}

// 生产环境
if (process.env.NODE_ENV === 'production') {
  app.use(morgan(
    ':method :url :status :response-time ms - :user-id',
    {
      stream: {
        write: (message) => logger.info(message.trim())
      }
    }
  ));
}
```

### 安全审计日志

```typescript
class AuditLogger {
  // 登录事件
  static async logLogin(userId: number, success: boolean, req: Request) {
    await prisma.auditLog.create({
      data: {
        userId,
        action: 'LOGIN',
        success,
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        metadata: {
          method: success ? 'password' : 'failed_attempt'
        }
      }
    });
  }

  // 权限检查
  static async logPermissionCheck(
    userId: number,
    resource: string,
    action: string,
    allowed: boolean,
    req: Request
  ) {
    await prisma.auditLog.create({
      data: {
        userId,
        action: 'PERMISSION_CHECK',
        success: allowed,
        ip: req.ip,
        metadata: {
          resource,
          action
        }
      }
    });
  }

  // 数据访问
  static async logDataAccess(
    userId: number,
    resourceType: string,
    resourceId: number,
    action: 'read' | 'create' | 'update' | 'delete',
    req: Request
  ) {
    await prisma.auditLog.create({
      data: {
        userId,
        action: `DATA_${action.toUpperCase()}`,
        success: true,
        ip: req.ip,
        metadata: {
          resourceType,
          resourceId
        }
      }
    });
  }

  // 敏感操作
  static async logSensitiveOperation(
    userId: number,
    operation: string,
    details: any,
    req: Request
  ) {
    await prisma.auditLog.create({
      data: {
        userId,
        action: operation,
        success: true,
        ip: req.ip,
        userAgent: req.headers['user-agent'],
        metadata: details
      }
    });
    
    // 同时发送告警
    if (this.isCriticalOperation(operation)) {
      await this.sendAlert(userId, operation, details);
    }
  }

  private static isCriticalOperation(operation: string): boolean {
    return [
      'DELETE_USER',
      'CHANGE_ROLE',
      'EXPORT_DATA',
      'BULK_DELETE'
    ].includes(operation);
  }

  private static async sendAlert(userId: number, operation: string, details: any) {
    // 发送到 Slack、PagerDuty 等
    logger.warn('Critical operation', { userId, operation, details });
  }
}

// 使用
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;
  
  const user = await verifyUser(email, password);
  
  if (!user) {
    await AuditLogger.logLogin(0, false, req);
    return res.status(401).json({ error: 'Invalid credentials' });
  }
  
  await AuditLogger.logLogin(user.id, true, req);
  
  res.json({ token: generateToken(user) });
});

app.delete('/api/users/:id', requireAuth, requireRole('admin'), async (req, res) => {
  const userId = parseInt(req.params.id);
  
  await prisma.user.delete({ where: { id: userId } });
  
  await AuditLogger.logSensitiveOperation(
    req.user.id,
    'DELETE_USER',
    { deletedUserId: userId },
    req
  );
  
  res.json({ message: 'User deleted' });
});
```

### Sentry 集成

```typescript
import * as Sentry from '@sentry/node';
import * as Tracing from '@sentry/tracing';

// 初始化 Sentry
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1, // 10% 的请求追踪
  
  // 过滤敏感数据
  beforeSend(event) {
    // 删除敏感 headers
    if (event.request?.headers) {
      delete event.request.headers['authorization'];
      delete event.request.headers['cookie'];
    }
    
    // 删除敏感 query 参数
    if (event.request?.query_string) {
      // 过滤 password、token 等
      event.request.query_string = event.request.query_string
        .replace(/password=[^&]*/g, 'password=[FILTERED]')
        .replace(/token=[^&]*/g, 'token=[FILTERED]');
    }
    
    return event;
  }
});

// 请求处理器（必须在所有中间件之前）
app.use(Sentry.Handlers.requestHandler());

// 追踪中间件
app.use(Sentry.Handlers.tracingHandler());

// 你的路由...

// 错误处理器（必须在所有中间件之后）
app.use(Sentry.Handlers.errorHandler());

// 手动捕获异常
try {
  await riskyOperation();
} catch (error) {
  Sentry.captureException(error, {
    tags: {
      section: 'payment'
    },
    user: {
      id: req.user.id,
      email: req.user.email
    }
  });
  
  throw error;
}
```

---

## 文件上传安全

### 文件类型验证

```typescript
import multer from 'multer';
import path from 'path';
import crypto from 'crypto';
import fileType from 'file-type';

// 允许的文件类型
const ALLOWED_TYPES = {
  'image/jpeg': ['.jpg', '.jpeg'],
  'image/png': ['.png'],
  'image/gif': ['.gif'],
  'image/webp': ['.webp'],
  'application/pdf': ['.pdf']
};

// 文件大小限制
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

// Multer 配置
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/');
  },
  filename: (req, file, cb) => {
    // 使用随机文件名
    const randomName = crypto.randomBytes(16).toString('hex');
    const ext = path.extname(file.originalname);
    cb(null, `${randomName}${ext}`);
  }
});

const upload = multer({
  storage,
  limits: {
    fileSize: MAX_SIZE,
    files: 1
  },
  fileFilter: (req, file, cb) => {
    // 检查 MIME 类型
    if (!ALLOWED_TYPES[file.mimetype]) {
      return cb(new Error('Invalid file type'));
    }
    
    // 检查文件扩展名
    const ext = path.extname(file.originalname).toLowerCase();
    const allowedExts = ALLOWED_TYPES[file.mimetype];
    
    if (!allowedExts.includes(ext)) {
      return cb(new Error('Invalid file extension'));
    }
    
    cb(null, true);
  }
});

// 上传路由
app.post('/api/upload', requireAuth, upload.single('file'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    // 验证文件内容（Magic Number）
    const buffer = await fs.promises.readFile(req.file.path);
    const type = await fileType.fromBuffer(buffer);

    if (!type || !ALLOWED_TYPES[type.mime]) {
      // 删除文件
      await fs.promises.unlink(req.file.path);
      return res.status(400).json({ error: 'Invalid file content' });
    }

    // 保存文件记录
    const file = await prisma.file.create({
      data: {
        filename: req.file.filename,
        originalName: req.file.originalname,
        mimeType: type.mime,
        size: req.file.size,
        userId: req.user.id
      }
    });

    res.json({
      message: 'File uploaded',
      file: {
        id: file.id,
        filename: file.filename,
        url: `/uploads/${file.filename}`
      }
    });

  } catch (error) {
    // 清理文件
    if (req.file) {
      await fs.promises.unlink(req.file.path).catch(() => {});
    }
    throw error;
  }
});
```

### 图片处理

```typescript
import sharp from 'sharp';

app.post('/api/upload/image', requireAuth, upload.single('image'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No image uploaded' });
    }

    const filename = req.file.filename;
    const filepath = req.file.path;

    // 处理图片（去除 EXIF 数据、压缩）
    await sharp(filepath)
      .resize(1920, 1080, {
        fit: 'inside',
        withoutEnlargement: true
      })
      .jpeg({
        quality: 80,
        progressive: true
      })
      .withMetadata({}) // 删除所有元数据
      .toFile(`uploads/processed/${filename}`);

    // 删除原文件
    await fs.promises.unlink(filepath);

    res.json({
      message: 'Image uploaded',
      url: `/uploads/processed/${filename}`
    });

  } catch (error) {
    if (req.file) {
      await fs.promises.unlink(req.file.path).catch(() => {});
    }
    throw error;
  }
});
```

### 病毒扫描

```typescript
import NodeClam from 'clamscan';

const clamscan = new NodeClam().init({
  clamdscan: {
    host: 'localhost',
    port: 3310
  }
});

app.post('/api/upload', requireAuth, upload.single('file'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: 'No file uploaded' });
    }

    // 病毒扫描
    const { isInfected, viruses } = await clamscan.isInfected(req.file.path);

    if (isInfected) {
      // 删除文件
      await fs.promises.unlink(req.file.path);
      
      logger.warn('Infected file uploaded', {
        userId: req.user.id,
        filename: req.file.originalname,
        viruses
      });

      return res.status(400).json({ error: 'File is infected' });
    }

    // 继续处理...

  } catch (error) {
    if (req.file) {
      await fs.promises.unlink(req.file.path).catch(() => {});
    }
    throw error;
  }
});
```

---

## API 安全

### CORS 配置

```typescript
import cors from 'cors';

// ✅ 严格的 CORS 配置
app.use(cors({
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://example.com',
      'https://www.example.com',
      'https://app.example.com'
    ];

    // 开发环境允许 localhost
    if (process.env.NODE_ENV === 'development') {
      allowedOrigins.push('http://localhost:3000');
    }

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true, // 允许 Cookie
  maxAge: 86400, // 预检请求缓存 24 小时
  optionsSuccessStatus: 200
}));

// ❌ 不安全的配置
app.use(cors({
  origin: '*', // 允许所有来源！
  credentials: true // 配合通配符，浏览器会拒绝
}));
```

### API 版本控制

```typescript
// URL 版本
app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);

// Header 版本
app.use((req, res, next) => {
  const version = req.headers['api-version'] || '1';
  
  if (version === '1') {
    v1Router(req, res, next);
  } else if (version === '2') {
    v2Router(req, res, next);
  } else {
    res.status(400).json({ error: 'Invalid API version' });
  }
});
```

### API 文档保护

```typescript
// Swagger UI 保护
app.use('/api-docs',
  requireAuth,
  requireRole('admin'),
  swaggerUi.serve,
  swaggerUi.setup(swaggerDocument)
);

// 或使用 Basic Auth
import basicAuth from 'express-basic-auth';

app.use('/api-docs',
  basicAuth({
    users: { 'admin': process.env.SWAGGER_PASSWORD! },
    challenge: true
  }),
  swaggerUi.serve,
  swaggerUi.setup(swaggerDocument)
);
```

---

## Docker 安全

### Dockerfile 最佳实践

```dockerfile
# ✅ 使用官方镜像
FROM node:18-alpine

# ✅ 创建非 root 用户
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# ✅ 设置工作目录
WORKDIR /app

# ✅ 复制依赖文件
COPY package*.json ./

# ✅ 安装依赖（生产环境）
RUN npm ci --only=production && \
    npm cache clean --force

# ✅ 复制应用代码
COPY --chown=nodejs:nodejs . .

# ✅ 切换到非 root 用户
USER nodejs

# ✅ 暴露端口
EXPOSE 3000

# ✅ 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node healthcheck.js

# ✅ 启动应用
CMD ["node", "dist/main.js"]
```

### 扫描镜像漏洞

```bash
# 使用 Trivy
trivy image my-app:latest

# 使用 Snyk
snyk container test my-app:latest

# 使用 Docker Scan
docker scan my-app:latest
```

### docker-compose 安全

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    # ✅ 只读文件系统
    read_only: true
    # ✅ 临时目录
    tmpfs:
      - /tmp
    # ✅ 资源限制
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    # ✅ 安全选项
    security_opt:
      - no-new-privileges:true
    # ✅ 禁用特权模式
    privileged: false
    
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    # ✅ 使用 secrets
    secrets:
      - db_password
    # ✅ 数据持久化
    volumes:
      - postgres_data:/var/lib/postgresql/data

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  postgres_data:
```

---

## 安全测试

### 单元测试

```typescript
import { describe, it, expect } from '@jest/globals';
import { hashPassword, verifyPassword } from '../src/auth';

describe('Password Security', () => {
  it('should hash password securely', async () => {
    const password = 'MyPassword123!';
    const hash = await hashPassword(password);

    // 哈希应该不同于原密码
    expect(hash).not.toBe(password);
    
    // 哈希长度应该正确
    expect(hash.length).toBeGreaterThan(50);
  });

  it('should verify correct password', async () => {
    const password = 'MyPassword123!';
    const hash = await hashPassword(password);

    const isValid = await verifyPassword(password, hash);
    expect(isValid).toBe(true);
  });

  it('should reject incorrect password', async () => {
    const password = 'MyPassword123!';
    const hash = await hashPassword(password);

    const isValid = await verifyPassword('WrongPassword', hash);
    expect(isValid).toBe(false);
  });
});
```

### 集成测试

```typescript
import request from 'supertest';
import app from '../src/app';

describe('Authentication Security', () => {
  it('should rate limit login attempts', async () => {
    const credentials = {
      email: 'test@example.com',
      password: 'wrong'
    };

    // 尝试 6 次登录
    for (let i = 0; i < 6; i++) {
      await request(app)
        .post('/api/login')
        .send(credentials);
    }

    // 第 6 次应该被限流
    const response = await request(app)
      .post('/api/login')
      .send(credentials);

    expect(response.status).toBe(429);
  });

  it('should prevent SQL injection', async () => {
    const response = await request(app)
      .get('/api/users')
      .query({ username: "admin' OR '1'='1" });

    // 不应该返回所有用户
    expect(response.status).not.toBe(200);
  });

  it('should set secure headers', async () => {
    const response = await request(app).get('/');

    expect(response.headers['x-frame-options']).toBe('DENY');
    expect(response.headers['x-content-type-options']).toBe('nosniff');
    expect(response.headers['strict-transport-security']).toBeDefined();
  });
});
```

### 渗透测试工具

```bash
# OWASP ZAP
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://example.com \
  -r report.html

# Nikto（Web 服务器扫描）
nikto -h https://example.com

# SQLMap（SQL 注入）
sqlmap -u "https://example.com/api/users?id=1" --batch

# Burp Suite（手动测试）
# 需要 GUI，用于拦截和修改请求
```

---

## 常见面试题

### 1. 如何保护 API 免受攻击？

**方法**：

1. **认证与授权**：JWT、OAuth 2.0
2. **速率限制**：防止暴力破解和 DDoS
3. **输入验证**：防止注入攻击
4. **HTTPS**：加密传输
5. **CORS**：限制跨域访问
6. **安全响应头**：Helmet
7. **日志监控**：检测异常行为
8. **API Key**：限制访问

### 2. 生产环境的安全检查清单？

**清单**：

- [ ] 使用 HTTPS
- [ ] 配置安全响应头
- [ ] 实施速率限制
- [ ] 验证所有输入
- [ ] 使用参数化查询
- [ ] 加密敏感数据
- [ ] 定期更新依赖
- [ ] 扫描漏洞
- [ ] 配置 CORS
- [ ] 实施日志监控
- [ ] 错误不暴露详情
- [ ] 使用环境变量
- [ ] 定期备份数据
- [ ] 实施灾难恢复计划

### 3. 如何检测和响应安全事件？

**步骤**：

1. **日志记录**：记录所有安全相关事件
2. **实时监控**：使用 Sentry、DataDog 等
3. **异常检测**：识别异常登录、API 调用
4. **告警机制**：Slack、PagerDuty
5. **事件响应**：
   - 确认事件
   - 隔离受影响系统
   - 调查根本原因
   - 修复漏洞
   - 恢复服务
   - 总结和改进

### 4. Docker 容器的安全最佳实践？

**实践**：

1. **使用官方镜像**
2. **定期更新镜像**
3. **扫描漏洞**
4. **非 root 用户**
5. **只读文件系统**
6. **资源限制**
7. **最小权限**
8. **secrets 管理**
9. **网络隔离**
10. **日志审计**

### 5. 如何安全地存储敏感数据？

**方法**：

1. **加密存储**：AES-256-GCM
2. **密钥管理**：AWS KMS、Vault
3. **数据脱敏**：仅显示部分数据
4. **访问控制**：严格的权限
5. **审计日志**：记录所有访问
6. **定期轮换密钥**
7. **备份加密**
8. **传输加密**：TLS

---

## 总结

### 安全实践优先级

| 优先级 | 实践 | 难度 | 影响 |
|--------|------|------|------|
| **P0（必须）** | HTTPS、输入验证、参数化查询 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **P1（重要）** | 速率限制、CORS、安全响应头 | ⭐⭐ | ⭐⭐⭐⭐ |
| **P2（推荐）** | 日志监控、依赖扫描、审计 | ⭐⭐⭐ | ⭐⭐⭐ |
| **P3（加分）** | 渗透测试、WAF、DDoS 防护 | ⭐⭐⭐⭐ | ⭐⭐⭐ |

### 实践路线图

**第 1 周**：基础安全
- [ ] 配置 HTTPS
- [ ] 使用 Helmet
- [ ] 实施输入验证
- [ ] 参数化查询

**第 2 周**：认证与授权
- [ ] 实现 JWT 认证
- [ ] 配置 RBAC
- [ ] 实施速率限制
- [ ] 账户锁定机制

**第 3 周**：监控与日志
- [ ] 配置 Winston
- [ ] 集成 Sentry
- [ ] 实施审计日志
- [ ] 告警机制

**第 4 周**：高级安全
- [ ] 依赖扫描
- [ ] 容器安全
- [ ] 渗透测试
- [ ] 灾难恢复计划

---

**上一篇**：[Web 安全](./03-web-security.md)  
**返回目录**：[认证与安全](./README.md)

