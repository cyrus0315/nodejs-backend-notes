# Web 安全

Web 安全是后端开发的重要组成部分。本文深入讲解 OWASP Top 10 和常见安全漏洞的防护。

## 目录
- [OWASP Top 10](#owasp-top-10)
- [注入攻击](#注入攻击)
- [XSS 跨站脚本](#xss-跨站脚本)
- [CSRF 跨站请求伪造](#csrf-跨站请求伪造)
- [认证与会话安全](#认证与会话安全)
- [加密与数据保护](#加密与数据保护)
- [安全响应头](#安全响应头)
- [API 安全](#api-安全)
- [面试题](#常见面试题)

---

## OWASP Top 10

### 2021 版 Top 10

| 排名 | 漏洞类型 | 描述 |
|------|---------|------|
| A01 | **访问控制失效** | 未正确实施权限检查 |
| A02 | **加密失败** | 敏感数据未加密或加密不当 |
| A03 | **注入** | SQL、NoSQL、命令注入等 |
| A04 | **不安全设计** | 设计层面的安全缺陷 |
| A05 | **安全配置错误** | 默认配置、错误信息泄露 |
| A06 | **易受攻击的组件** | 使用有漏洞的依赖 |
| A07 | **身份认证失败** | 弱密码、会话管理缺陷 |
| A08 | **软件和数据完整性失败** | 不安全的 CI/CD、反序列化 |
| A09 | **安全日志和监控失败** | 无法检测攻击 |
| A10 | **服务端请求伪造（SSRF）** | 服务器发起恶意请求 |

---

## 注入攻击

### SQL 注入

#### 漏洞示例

```typescript
// ❌ 危险：SQL 注入漏洞
app.get('/users', async (req, res) => {
  const { username } = req.query;
  
  // 用户输入：admin' OR '1'='1
  const query = `SELECT * FROM users WHERE username = '${username}'`;
  const users = await db.query(query);
  
  res.json(users);
});

// 实际执行：SELECT * FROM users WHERE username = 'admin' OR '1'='1'
// 结果：返回所有用户！
```

#### 防护方法

```typescript
// ✅ 方法 1：参数化查询（推荐）
app.get('/users', async (req, res) => {
  const { username } = req.query;
  
  const users = await db.query(
    'SELECT * FROM users WHERE username = $1',
    [username] // 参数化
  );
  
  res.json(users);
});

// ✅ 方法 2：使用 ORM
app.get('/users', async (req, res) => {
  const { username } = req.query;
  
  const users = await prisma.user.findMany({
    where: { username } // Prisma 自动防注入
  });
  
  res.json(users);
});

// ✅ 方法 3：输入验证
import { z } from 'zod';

const UserQuerySchema = z.object({
  username: z.string().regex(/^[a-zA-Z0-9_]+$/).max(50)
});

app.get('/users', async (req, res) => {
  const { username } = UserQuerySchema.parse(req.query);
  
  const users = await prisma.user.findMany({
    where: { username }
  });
  
  res.json(users);
});
```

### NoSQL 注入

#### 漏洞示例

```typescript
// ❌ 危险：NoSQL 注入
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  // 用户输入：{ "username": { "$ne": null }, "password": { "$ne": null } }
  const user = await User.findOne({
    username,
    password
  });
  
  // 绕过认证！
});
```

#### 防护方法

```typescript
// ✅ 方法 1：输入验证
import { z } from 'zod';

const LoginSchema = z.object({
  username: z.string().min(1).max(100),
  password: z.string().min(1).max(100)
});

app.post('/login', async (req, res) => {
  // 验证输入类型
  const { username, password } = LoginSchema.parse(req.body);
  
  const user = await User.findOne({
    username,
    password: await bcrypt.hash(password, 10)
  });
  
  // ...
});

// ✅ 方法 2：禁止对象查询
app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  
  // 确保是字符串
  if (typeof username !== 'string' || typeof password !== 'string') {
    return res.status(400).json({ error: 'Invalid input' });
  }
  
  const user = await User.findOne({
    username: String(username),
    password: await bcrypt.hash(String(password), 10)
  });
  
  // ...
});

// ✅ 方法 3：使用 mongoose-sanitize
import mongoSanitize from 'express-mongo-sanitize';

app.use(mongoSanitize()); // 自动清理 $、. 等特殊字符
```

### 命令注入

#### 漏洞示例

```typescript
// ❌ 危险：命令注入
import { exec } from 'child_process';

app.get('/ping', (req, res) => {
  const { host } = req.query;
  
  // 用户输入：google.com; rm -rf /
  exec(`ping -c 4 ${host}`, (error, stdout) => {
    res.send(stdout);
  });
});
```

#### 防护方法

```typescript
// ✅ 方法 1：使用安全的 API
import { execFile } from 'child_process';

app.get('/ping', (req, res) => {
  const { host } = req.query;
  
  // 白名单验证
  if (!/^[a-zA-Z0-9.-]+$/.test(host)) {
    return res.status(400).json({ error: 'Invalid host' });
  }
  
  // 使用 execFile（参数分离）
  execFile('ping', ['-c', '4', host], (error, stdout) => {
    res.send(stdout);
  });
});

// ✅ 方法 2：完全避免执行系统命令
// 使用 Node.js 库代替系统命令
import ping from 'ping';

app.get('/ping', async (req, res) => {
  const { host } = req.query;
  
  if (!/^[a-zA-Z0-9.-]+$/.test(host)) {
    return res.status(400).json({ error: 'Invalid host' });
  }
  
  const result = await ping.promise.probe(host);
  res.json(result);
});
```

---

## XSS 跨站脚本

### 反射型 XSS

#### 漏洞示例

```typescript
// ❌ 危险：反射型 XSS
app.get('/search', (req, res) => {
  const { query } = req.query;
  
  // 用户输入：<script>alert('XSS')</script>
  res.send(`Search results for: ${query}`);
});
```

#### 防护方法

```typescript
// ✅ 方法 1：转义输出
import escapeHtml from 'escape-html';

app.get('/search', (req, res) => {
  const { query } = req.query;
  
  res.send(`Search results for: ${escapeHtml(query)}`);
  // 输出：Search results for: &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;
});

// ✅ 方法 2：使用模板引擎（自动转义）
app.get('/search', (req, res) => {
  const { query } = req.query;
  
  res.render('search', { query }); // EJS、Pug 等自动转义
});

// ✅ 方法 3：Content Security Policy
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self'"
  );
  next();
});
```

### 存储型 XSS

#### 漏洞示例

```typescript
// ❌ 危险：存储型 XSS
app.post('/comments', async (req, res) => {
  const { content } = req.body;
  
  // 用户输入：<img src=x onerror="alert('XSS')">
  await prisma.comment.create({
    data: { content }
  });
  
  res.json({ message: 'Comment created' });
});

app.get('/comments', async (req, res) => {
  const comments = await prisma.comment.findMany();
  
  // 直接输出，触发 XSS
  res.send(comments.map(c => `<div>${c.content}</div>`).join(''));
});
```

#### 防护方法

```typescript
// ✅ 方法 1：输入过滤 + 输出转义
import DOMPurify from 'isomorphic-dompurify';
import escapeHtml from 'escape-html';

app.post('/comments', async (req, res) => {
  let { content } = req.body;
  
  // 清理 HTML（允许部分标签）
  content = DOMPurify.sanitize(content, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong'],
    ALLOWED_ATTR: []
  });
  
  await prisma.comment.create({
    data: { content }
  });
  
  res.json({ message: 'Comment created' });
});

app.get('/comments', async (req, res) => {
  const comments = await prisma.comment.findMany();
  
  // 转义输出
  const html = comments.map(c => 
    `<div>${escapeHtml(c.content)}</div>`
  ).join('');
  
  res.send(html);
});

// ✅ 方法 2：使用 Markdown（更安全）
import MarkdownIt from 'markdown-it';

const md = new MarkdownIt({
  html: false, // 禁止 HTML
  linkify: true,
  typographer: true
});

app.post('/comments', async (req, res) => {
  const { content } = req.body;
  
  // 存储 Markdown
  await prisma.comment.create({
    data: { content }
  });
  
  res.json({ message: 'Comment created' });
});

app.get('/comments', async (req, res) => {
  const comments = await prisma.comment.findMany();
  
  // 渲染为 HTML
  const html = comments.map(c => 
    `<div>${md.render(c.content)}</div>`
  ).join('');
  
  res.send(html);
});
```

### DOM-Based XSS

#### 漏洞示例

```html
<!-- ❌ 危险：DOM-Based XSS -->
<script>
  // URL: http://example.com/#<img src=x onerror="alert('XSS')">
  const hash = location.hash.substring(1);
  document.getElementById('output').innerHTML = hash;
</script>
```

#### 防护方法

```html
<!-- ✅ 使用 textContent -->
<script>
  const hash = location.hash.substring(1);
  document.getElementById('output').textContent = hash; // 安全
</script>

<!-- ✅ 或使用框架（自动转义） -->
<script>
  // React
  function App() {
    const hash = location.hash.substring(1);
    return <div>{hash}</div>; // React 自动转义
  }
</script>
```

---

## CSRF 跨站请求伪造

### 漏洞原理

```
1. 用户登录 bank.com，获得 Cookie
2. 用户访问恶意网站 evil.com
3. evil.com 页面包含：
   <img src="http://bank.com/transfer?to=attacker&amount=1000">
4. 浏览器自动携带 bank.com 的 Cookie
5. 转账请求成功！
```

### 防护方法

#### 1. CSRF Token

```typescript
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

app.use(cookieParser());
app.use(csrf({ cookie: true }));

// 渲染表单时，包含 CSRF Token
app.get('/transfer', (req, res) => {
  res.render('transfer', { csrfToken: req.csrfToken() });
});

// 表单
// <form method="POST" action="/transfer">
//   <input type="hidden" name="_csrf" value="{{ csrfToken }}">
//   ...
// </form>

// 验证 Token
app.post('/transfer', (req, res) => {
  // csurf 中间件自动验证
  // 如果 Token 无效，返回 403
  
  const { to, amount } = req.body;
  // 执行转账...
});

// 对于 AJAX 请求
app.get('/api/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// 前端
fetch('/api/csrf-token')
  .then(res => res.json())
  .then(data => {
    fetch('/api/transfer', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': data.csrfToken
      },
      body: JSON.stringify({ to, amount })
    });
  });
```

#### 2. SameSite Cookie

```typescript
// ✅ 设置 SameSite 属性
app.use(session({
  secret: process.env.SESSION_SECRET!,
  cookie: {
    sameSite: 'strict', // 或 'lax'
    secure: true, // HTTPS only
    httpOnly: true
  }
}));

// SameSite 值：
// - strict：完全禁止跨站请求携带 Cookie
// - lax：GET 请求可以（如链接、预加载）
// - none：允许跨站（需要 secure: true）
```

#### 3. Double Submit Cookie

```typescript
// 生成随机 Token
function generateToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

// 登录时设置 Token Cookie
app.post('/login', async (req, res) => {
  // 验证用户...
  
  const csrfToken = generateToken();
  
  // 设置 Cookie
  res.cookie('csrf-token', csrfToken, {
    httpOnly: false, // 允许 JS 读取
    secure: true,
    sameSite: 'strict'
  });
  
  res.json({ message: 'Login successful' });
});

// 验证 Token
function verifyCsrfToken(req, res, next) {
  const tokenFromCookie = req.cookies['csrf-token'];
  const tokenFromHeader = req.headers['x-csrf-token'];
  
  if (!tokenFromCookie || tokenFromCookie !== tokenFromHeader) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }
  
  next();
}

// 应用到需要保护的路由
app.post('/transfer', verifyCsrfToken, transferHandler);

// 前端发送请求
function getCookie(name) {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop().split(';').shift();
}

fetch('/api/transfer', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': getCookie('csrf-token')
  },
  body: JSON.stringify({ to, amount })
});
```

#### 4. 验证 Referer/Origin

```typescript
// 验证请求来源
function verifyOrigin(req, res, next) {
  const origin = req.headers.origin || req.headers.referer;
  const allowedOrigins = [
    'https://example.com',
    'https://www.example.com'
  ];
  
  if (!origin || !allowedOrigins.some(allowed => origin.startsWith(allowed))) {
    return res.status(403).json({ error: 'Invalid origin' });
  }
  
  next();
}

app.post('/api/transfer', verifyOrigin, transferHandler);
```

---

## 认证与会话安全

### 密码安全

```typescript
import bcrypt from 'bcryptjs';
import argon2 from 'argon2';

// ✅ 使用 bcrypt
async function hashPassword(password: string): Promise<string> {
  const saltRounds = 10; // 成本因子
  return await bcrypt.hash(password, saltRounds);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return await bcrypt.compare(password, hash);
}

// ✅ 使用 argon2（更安全，推荐）
async function hashPasswordArgon2(password: string): Promise<string> {
  return await argon2.hash(password, {
    type: argon2.argon2id,
    memoryCost: 65536, // 64 MB
    timeCost: 3,
    parallelism: 4
  });
}

async function verifyPasswordArgon2(password: string, hash: string): Promise<boolean> {
  return await argon2.verify(hash, password);
}

// ❌ 不要使用弱哈希
const hash = crypto.createHash('md5').update(password).digest('hex'); // 不安全！
const hash = crypto.createHash('sha1').update(password).digest('hex'); // 不安全！
```

### 密码强度验证

```typescript
import passwordValidator from 'password-validator';

const schema = new passwordValidator();

schema
  .is().min(8)                                    // 最少 8 个字符
  .is().max(100)                                  // 最多 100 个字符
  .has().uppercase()                              // 必须包含大写字母
  .has().lowercase()                              // 必须包含小写字母
  .has().digits(1)                                // 必须包含数字
  .has().symbols()                                // 必须包含特殊字符
  .has().not().spaces()                           // 不能包含空格
  .is().not().oneOf(['Password123', 'Admin123']); // 禁止常见密码

app.post('/register', (req, res) => {
  const { password } = req.body;
  
  const errors = schema.validate(password, { details: true });
  
  if (errors.length > 0) {
    return res.status(400).json({
      error: 'Password does not meet requirements',
      details: errors
    });
  }
  
  // 继续注册...
});
```

### 会话固定攻击防护

```typescript
// ✅ 登录后重新生成 Session ID
app.post('/login', async (req, res) => {
  // 验证用户...
  
  // 重新生成 Session ID
  req.session.regenerate((err) => {
    if (err) {
      return res.status(500).json({ error: 'Session regeneration failed' });
    }
    
    req.session.userId = user.id;
    res.json({ message: 'Login successful' });
  });
});
```

### 会话超时

```typescript
// Session 超时配置
app.use(session({
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 30 * 60 * 1000, // 30 分钟
    rolling: true // 每次请求刷新过期时间
  }
}));

// 手动检查最后活动时间
app.use((req, res, next) => {
  if (req.session.userId) {
    const now = Date.now();
    const lastActivity = req.session.lastActivity || now;
    const timeout = 30 * 60 * 1000; // 30 分钟
    
    if (now - lastActivity > timeout) {
      req.session.destroy(() => {
        res.status(401).json({ error: 'Session expired' });
      });
      return;
    }
    
    req.session.lastActivity = now;
  }
  
  next();
});
```

---

## 加密与数据保护

### 敏感数据加密

```typescript
import crypto from 'crypto';

class EncryptionService {
  private algorithm = 'aes-256-gcm';
  private key: Buffer;

  constructor() {
    // 密钥应该从环境变量读取
    this.key = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex');
  }

  // 加密
  encrypt(plaintext: string): { encrypted: string; iv: string; tag: string } {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');

    const tag = cipher.getAuthTag();

    return {
      encrypted,
      iv: iv.toString('hex'),
      tag: tag.toString('hex')
    };
  }

  // 解密
  decrypt(encrypted: string, iv: string, tag: string): string {
    const decipher = crypto.createDecipheriv(
      this.algorithm,
      this.key,
      Buffer.from(iv, 'hex')
    );

    decipher.setAuthTag(Buffer.from(tag, 'hex'));

    let decrypted = decipher.update(encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');

    return decrypted;
  }
}

const encryption = new EncryptionService();

// 存储敏感数据
app.post('/api/credit-card', requireAuth, async (req, res) => {
  const { cardNumber } = req.body;

  // 加密信用卡号
  const { encrypted, iv, tag } = encryption.encrypt(cardNumber);

  await prisma.paymentMethod.create({
    data: {
      userId: req.user.id,
      encryptedCardNumber: encrypted,
      iv,
      tag
    }
  });

  res.json({ message: 'Card saved' });
});

// 读取敏感数据
app.get('/api/credit-card', requireAuth, async (req, res) => {
  const paymentMethod = await prisma.paymentMethod.findFirst({
    where: { userId: req.user.id }
  });

  if (!paymentMethod) {
    return res.status(404).json({ error: 'Not found' });
  }

  // 解密
  const cardNumber = encryption.decrypt(
    paymentMethod.encryptedCardNumber,
    paymentMethod.iv,
    paymentMethod.tag
  );

  // 只返回部分信息
  res.json({
    maskedCardNumber: `****-****-****-${cardNumber.slice(-4)}`
  });
});
```

### HTTPS 配置

```typescript
import https from 'https';
import fs from 'fs';

const options = {
  key: fs.readFileSync('private-key.pem'),
  cert: fs.readFileSync('certificate.pem'),
  
  // 推荐的 TLS 配置
  minVersion: 'TLSv1.2',
  ciphers: [
    'ECDHE-ECDSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES256-GCM-SHA384'
  ].join(':'),
  honorCipherOrder: true
};

https.createServer(options, app).listen(443, () => {
  console.log('HTTPS server running on port 443');
});

// 强制 HTTPS
app.use((req, res, next) => {
  if (!req.secure && process.env.NODE_ENV === 'production') {
    return res.redirect(`https://${req.headers.host}${req.url}`);
  }
  next();
});
```

---

## 安全响应头

### Helmet 配置

```typescript
import helmet from 'helmet';

app.use(helmet());

// 或自定义配置
app.use(helmet({
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", 'cdn.example.com'],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'api.example.com'],
      fontSrc: ["'self'", 'fonts.googleapis.com'],
      objectSrc: ["'none'"],
      mediaSrc: ["'self'"],
      frameSrc: ["'none'"]
    }
  },
  
  // Strict-Transport-Security
  hsts: {
    maxAge: 31536000, // 1 年
    includeSubDomains: true,
    preload: true
  },
  
  // X-Frame-Options
  frameguard: {
    action: 'deny' // 防止点击劫持
  },
  
  // X-Content-Type-Options
  noSniff: true,
  
  // X-XSS-Protection
  xssFilter: true,
  
  // Referrer-Policy
  referrerPolicy: {
    policy: 'strict-origin-when-cross-origin'
  }
}));
```

### 手动设置响应头

```typescript
app.use((req, res, next) => {
  // 防止点击劫持
  res.setHeader('X-Frame-Options', 'DENY');
  
  // 防止 MIME 类型嗅探
  res.setHeader('X-Content-Type-Options', 'nosniff');
  
  // XSS 保护
  res.setHeader('X-XSS-Protection', '1; mode=block');
  
  // HSTS
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  
  // CSP
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline'"
  );
  
  // Referrer Policy
  res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
  
  // Permissions Policy
  res.setHeader(
    'Permissions-Policy',
    'geolocation=(), microphone=(), camera=()'
  );
  
  next();
});
```

---

## API 安全

### 速率限制

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redis = new Redis();

// 全局限流
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 最多 100 个请求
  message: 'Too many requests',
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    client: redis,
    prefix: 'rl:'
  })
});

app.use('/api/', globalLimiter);

// 登录限流（更严格）
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts',
  skipSuccessfulRequests: true // 成功的请求不计数
});

app.post('/api/login', loginLimiter, loginHandler);

// IP + 用户双重限流
const createPostLimiter = async (req, res, next) => {
  const userId = req.user?.id;
  const ip = req.ip;
  
  // IP 限流：每小时 20 个
  const ipKey = `rl:post:ip:${ip}`;
  const ipCount = await redis.incr(ipKey);
  if (ipCount === 1) {
    await redis.expire(ipKey, 3600);
  }
  if (ipCount > 20) {
    return res.status(429).json({ error: 'Too many requests' });
  }
  
  // 用户限流：每小时 10 个
  if (userId) {
    const userKey = `rl:post:user:${userId}`;
    const userCount = await redis.incr(userKey);
    if (userCount === 1) {
      await redis.expire(userKey, 3600);
    }
    if (userCount > 10) {
      return res.status(429).json({ error: 'Too many posts' });
    }
  }
  
  next();
};

app.post('/api/posts', requireAuth, createPostLimiter, createPost);
```

### 输入验证

```typescript
import { z } from 'zod';
import validator from 'validator';

// 使用 Zod
const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100),
  age: z.number().int().min(18).max(120),
  website: z.string().url().optional(),
  bio: z.string().max(500).optional()
});

app.post('/api/users', async (req, res) => {
  try {
    const data = CreateUserSchema.parse(req.body);
    
    // 额外的安全检查
    if (!validator.isEmail(data.email)) {
      return res.status(400).json({ error: 'Invalid email' });
    }
    
    // 创建用户...
  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      });
    }
    throw error;
  }
});

// 自定义验证中间件
function validate(schema: z.ZodSchema) {
  return (req, res, next) => {
    try {
      req.body = schema.parse(req.body);
      next();
    } catch (error) {
      if (error instanceof z.ZodError) {
        return res.status(400).json({
          error: 'Validation failed',
          details: error.errors
        });
      }
      next(error);
    }
  };
}

// 使用
app.post('/api/users', validate(CreateUserSchema), createUser);
```

### API Key 认证

```typescript
class APIKeyService {
  // 生成 API Key
  static generate(): string {
    return `sk_${crypto.randomBytes(32).toString('hex')}`;
  }

  // 验证 API Key
  static async verify(apiKey: string): Promise<{ userId: number } | null> {
    // 从数据库查询
    const key = await prisma.apiKey.findUnique({
      where: { key: apiKey, active: true }
    });

    if (!key) return null;

    // 更新最后使用时间
    await prisma.apiKey.update({
      where: { id: key.id },
      data: {
        lastUsedAt: new Date(),
        usageCount: { increment: 1 }
      }
    });

    return { userId: key.userId };
  }
}

// API Key 认证中间件
async function requireAPIKey(req, res, next) {
  const apiKey = req.headers['x-api-key'] as string;

  if (!apiKey) {
    return res.status(401).json({ error: 'API key required' });
  }

  const user = await APIKeyService.verify(apiKey);

  if (!user) {
    return res.status(401).json({ error: 'Invalid API key' });
  }

  req.user = user;
  next();
}

// 使用
app.get('/api/data', requireAPIKey, getData);

// 创建 API Key
app.post('/api/keys', requireAuth, async (req, res) => {
  const { name } = req.body;

  const apiKey = APIKeyService.generate();

  await prisma.apiKey.create({
    data: {
      key: apiKey,
      name,
      userId: req.user.id
    }
  });

  res.json({ apiKey });
});
```

---

## 常见面试题

### 1. 如何防止 SQL 注入？

**方法**：

1. **参数化查询**（最重要）
2. **使用 ORM**（Prisma、TypeORM）
3. **输入验证**（白名单验证）
4. **最小权限原则**（数据库用户权限）
5. **WAF**（Web Application Firewall）

### 2. XSS 和 CSRF 的区别？

| 特性 | XSS | CSRF |
|------|-----|------|
| **攻击目标** | 注入恶意脚本 | 伪造用户请求 |
| **攻击方式** | 在页面执行 JS | 诱导用户点击/访问 |
| **防护** | 输入过滤、输出转义、CSP | CSRF Token、SameSite Cookie |
| **危害** | 窃取 Cookie、会话劫持 | 执行非授权操作 |

### 3. 如何安全存储密码？

**步骤**：

1. **使用强哈希算法**：bcrypt、argon2（不要用 MD5、SHA1）
2. **加盐**：每个密码使用唯一的 salt
3. **慢哈希**：增加计算成本，防止暴力破解
4. **密码强度验证**：要求复杂密码
5. **密码泄露检测**：检查密码是否在已泄露数据库中

### 4. HTTPS 为什么安全？

**原理**：

1. **加密**：TLS 加密传输内容
2. **身份验证**：证书验证服务器身份
3. **完整性**：防止数据篡改

**注意**：

- HTTPS 只保护传输过程
- 不保护服务器端数据
- 不防护 XSS、CSRF 等

### 5. 如何防止暴力破解？

**方法**：

1. **速率限制**：限制尝试次数
2. **验证码**：多次失败后要求验证码
3. **账户锁定**：临时锁定账户
4. **慢哈希**：增加每次尝试的时间
5. **监控告警**：检测异常登录
6. **2FA**：多因素认证

---

## SSRF（服务端请求伪造）

### SSRF 原理

SSRF 攻击利用服务器发起请求，访问内部资源或执行恶意操作。

```
攻击流程：
1. 攻击者发送恶意 URL 给服务器
   ↓
2. 服务器根据 URL 发起请求
   ↓
3. 攻击者访问到内部资源
   
危害：
- 访问内部服务（如 http://localhost:9200）
- 读取云平台元数据（如 AWS 169.254.169.254）
- 端口扫描内部网络
- 绕过防火墙
```

### SSRF 漏洞示例

```typescript
// ❌ 危险：SSRF 漏洞
app.post('/api/fetch-url', async (req, res) => {
  const { url } = req.body;
  
  // 用户输入：http://169.254.169.254/latest/meta-data/
  // 或：http://localhost:6379/
  const response = await fetch(url);
  const data = await response.text();
  
  res.send(data);
});

// ❌ 危险：通过图片 URL
app.post('/api/upload-from-url', async (req, res) => {
  const { imageUrl } = req.body;
  
  // 用户输入：file:///etc/passwd
  const response = await fetch(imageUrl);
  const buffer = await response.buffer();
  
  // 保存图片...
});
```

### SSRF 防护

```typescript
import { URL } from 'url';
import dns from 'dns';
import { promisify } from 'util';

const dnsLookup = promisify(dns.lookup);

class SSRFProtection {
  // 禁止的 IP 范围
  private blockedRanges = [
    // 私有 IP
    /^10\./,
    /^172\.(1[6-9]|2[0-9]|3[01])\./,
    /^192\.168\./,
    // 本地回环
    /^127\./,
    /^localhost$/i,
    /^0\./,
    // 链路本地
    /^169\.254\./,
    // IPv6 私有
    /^::1$/,
    /^fc00:/i,
    /^fe80:/i,
    // 云平台元数据
    /^169\.254\.169\.254$/,
    /^fd00:/i
  ];

  // 允许的协议
  private allowedProtocols = ['http:', 'https:'];

  // 允许的域名（白名单）
  private allowedDomains: string[] = [];

  constructor(allowedDomains: string[] = []) {
    this.allowedDomains = allowedDomains;
  }

  async validateUrl(urlString: string): Promise<{ valid: boolean; error?: string }> {
    try {
      // 1. 解析 URL
      const url = new URL(urlString);

      // 2. 检查协议
      if (!this.allowedProtocols.includes(url.protocol)) {
        return { valid: false, error: `Protocol not allowed: ${url.protocol}` };
      }

      // 3. 检查端口（只允许 80 和 443）
      const port = url.port || (url.protocol === 'https:' ? '443' : '80');
      if (!['80', '443'].includes(port)) {
        return { valid: false, error: `Port not allowed: ${port}` };
      }

      // 4. 白名单检查
      if (this.allowedDomains.length > 0) {
        const isAllowed = this.allowedDomains.some(
          domain => url.hostname === domain || url.hostname.endsWith(`.${domain}`)
        );
        if (!isAllowed) {
          return { valid: false, error: 'Domain not in whitelist' };
        }
      }

      // 5. DNS 解析检查
      const hostname = url.hostname;
      
      // 检查是否直接是 IP
      if (this.isBlockedIP(hostname)) {
        return { valid: false, error: 'IP address blocked' };
      }

      // DNS 解析获取真实 IP
      try {
        const { address } = await dnsLookup(hostname);
        if (this.isBlockedIP(address)) {
          return { valid: false, error: 'Resolved IP address blocked' };
        }
      } catch (e) {
        return { valid: false, error: 'DNS resolution failed' };
      }

      return { valid: true };
    } catch (e) {
      return { valid: false, error: 'Invalid URL' };
    }
  }

  private isBlockedIP(ip: string): boolean {
    return this.blockedRanges.some(range => range.test(ip));
  }
}

const ssrfProtection = new SSRFProtection([
  'api.example.com',
  'cdn.example.com'
]);

// ✅ 安全的实现
app.post('/api/fetch-url', async (req, res) => {
  const { url } = req.body;

  // 验证 URL
  const validation = await ssrfProtection.validateUrl(url);
  if (!validation.valid) {
    return res.status(400).json({ error: validation.error });
  }

  try {
    // 设置超时和重定向限制
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 10000);

    const response = await fetch(url, {
      signal: controller.signal,
      redirect: 'manual', // 不跟随重定向
      headers: {
        'User-Agent': 'MyApp/1.0'
      }
    });

    clearTimeout(timeout);

    // 检查重定向目标
    if (response.status >= 300 && response.status < 400) {
      const redirectUrl = response.headers.get('location');
      if (redirectUrl) {
        const redirectValidation = await ssrfProtection.validateUrl(redirectUrl);
        if (!redirectValidation.valid) {
          return res.status(400).json({ error: 'Redirect URL blocked' });
        }
      }
    }

    const data = await response.text();
    res.send(data);

  } catch (error) {
    res.status(500).json({ error: 'Failed to fetch URL' });
  }
});

// ✅ 更安全：使用代理服务
app.post('/api/fetch-url', async (req, res) => {
  const { url } = req.body;

  // 通过专用的出口代理访问外部资源
  const response = await fetch(process.env.EGRESS_PROXY_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ url })
  });

  // 代理服务负责 SSRF 防护
  const data = await response.json();
  res.json(data);
});
```

---

## 原型链污染（Prototype Pollution）

### 原型链污染原理

JavaScript 原型链污染攻击通过修改对象原型来影响所有对象实例。

```typescript
// JavaScript 原型链
const obj = {};
obj.__proto__ === Object.prototype; // true

// 污染示例
obj.__proto__.polluted = true;

// 所有对象都受影响
const newObj = {};
console.log(newObj.polluted); // true
```

### 漏洞示例

```typescript
// ❌ 危险：递归合并对象
function merge(target: any, source: any): any {
  for (const key in source) {
    if (typeof source[key] === 'object' && source[key] !== null) {
      if (!target[key]) target[key] = {};
      merge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
  return target;
}

// 攻击载荷
const maliciousPayload = JSON.parse(
  '{"__proto__": {"isAdmin": true}}'
);

const user = {};
merge(user, maliciousPayload);

// 所有对象都被污染
const newUser = {};
console.log(newUser.isAdmin); // true !

// ❌ 危险：路径赋值
function setPath(obj: any, path: string, value: any) {
  const keys = path.split('.');
  let current = obj;
  
  for (let i = 0; i < keys.length - 1; i++) {
    if (!current[keys[i]]) current[keys[i]] = {};
    current = current[keys[i]];
  }
  
  current[keys[keys.length - 1]] = value;
}

// 攻击
setPath({}, '__proto__.polluted', true);
```

### 原型链污染防护

```typescript
// ✅ 方法 1：使用 Object.create(null)
function safeMerge(target: any, source: any): any {
  const result = Object.create(null);
  
  for (const key of Object.keys(target)) {
    result[key] = target[key];
  }
  
  for (const key of Object.keys(source)) {
    // 跳过危险属性
    if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
      continue;
    }
    
    if (
      typeof source[key] === 'object' &&
      source[key] !== null &&
      !Array.isArray(source[key])
    ) {
      result[key] = safeMerge(result[key] || {}, source[key]);
    } else {
      result[key] = source[key];
    }
  }
  
  return result;
}

// ✅ 方法 2：使用 Object.hasOwn() 或 hasOwnProperty
function safeSetPath(obj: any, path: string, value: any) {
  const dangerousKeys = ['__proto__', 'constructor', 'prototype'];
  const keys = path.split('.');
  
  // 检查危险键
  if (keys.some(key => dangerousKeys.includes(key))) {
    throw new Error('Dangerous path detected');
  }
  
  let current = obj;
  
  for (let i = 0; i < keys.length - 1; i++) {
    const key = keys[i];
    if (!Object.hasOwn(current, key)) {
      current[key] = {};
    }
    current = current[key];
  }
  
  current[keys[keys.length - 1]] = value;
}

// ✅ 方法 3：使用安全的库
import merge from 'lodash.merge'; // lodash 已修复此问题
import cloneDeep from 'lodash.clonedeep';

// ✅ 方法 4：冻结原型
Object.freeze(Object.prototype);
Object.freeze(Array.prototype);

// ✅ 方法 5：输入验证
import { z } from 'zod';

const SafeObjectSchema = z.object({}).passthrough().refine(
  (obj) => {
    const checkKeys = (o: any): boolean => {
      for (const key of Object.keys(o)) {
        if (key === '__proto__' || key === 'constructor' || key === 'prototype') {
          return false;
        }
        if (typeof o[key] === 'object' && o[key] !== null) {
          if (!checkKeys(o[key])) return false;
        }
      }
      return true;
    };
    return checkKeys(obj);
  },
  { message: 'Object contains dangerous keys' }
);

app.post('/api/data', (req, res) => {
  try {
    const safeData = SafeObjectSchema.parse(req.body);
    // 处理数据...
  } catch (error) {
    res.status(400).json({ error: 'Invalid data' });
  }
});

// ✅ 方法 6：使用 Map 代替普通对象
const safeStorage = new Map<string, any>();
safeStorage.set('user', { name: 'John' });
// Map 不受原型链污染影响
```

### 检测原型链污染

```typescript
// 检测原型是否被污染
function checkPrototypePollution(): boolean {
  const testObj = {};
  
  // 检查常见的污染属性
  const suspiciousKeys = [
    'isAdmin',
    'role',
    'admin',
    'authenticated',
    'polluted'
  ];
  
  for (const key of suspiciousKeys) {
    if ((testObj as any)[key] !== undefined) {
      console.error(`Prototype pollution detected: ${key}`);
      return true;
    }
  }
  
  return false;
}

// 定期检测
setInterval(() => {
  if (checkPrototypePollution()) {
    // 记录并告警
    logger.error('Prototype pollution detected!');
    Sentry.captureMessage('Prototype pollution detected');
  }
}, 60000);
```

---

## 路径遍历（Path Traversal）

### 路径遍历原理

攻击者通过操纵文件路径访问服务器上的任意文件。

```
攻击示例：
- ../../../etc/passwd
- ..%2F..%2F..%2Fetc/passwd（URL 编码）
- ....//....//etc/passwd（双写绕过）
- /var/www/app/../../../etc/passwd
```

### 漏洞示例

```typescript
import path from 'path';
import fs from 'fs';

// ❌ 危险：直接拼接路径
app.get('/api/files/:filename', (req, res) => {
  const { filename } = req.params;
  
  // 用户输入：../../../etc/passwd
  const filepath = `./uploads/${filename}`;
  
  // 读取任意文件！
  const content = fs.readFileSync(filepath, 'utf8');
  res.send(content);
});

// ❌ 危险：包含用户输入的路径
app.get('/api/download', (req, res) => {
  const { file } = req.query;
  
  // 用户输入：file=../../../../etc/passwd
  res.download(`./public/${file}`);
});
```

### 路径遍历防护

```typescript
import path from 'path';
import fs from 'fs';

class PathSecurity {
  private basePath: string;

  constructor(basePath: string) {
    // 规范化并解析为绝对路径
    this.basePath = path.resolve(basePath);
  }

  // 验证路径是否在允许的目录内
  isPathSafe(userPath: string): boolean {
    // 规范化路径
    const normalizedPath = path.normalize(userPath);
    
    // 解析为绝对路径
    const resolvedPath = path.resolve(this.basePath, normalizedPath);
    
    // 检查是否在基础目录内
    return resolvedPath.startsWith(this.basePath + path.sep) ||
           resolvedPath === this.basePath;
  }

  // 获取安全的绝对路径
  getSafePath(userPath: string): string | null {
    if (!this.isPathSafe(userPath)) {
      return null;
    }
    
    return path.resolve(this.basePath, path.normalize(userPath));
  }

  // 验证文件名
  isValidFilename(filename: string): boolean {
    // 禁止路径分隔符
    if (filename.includes('/') || filename.includes('\\')) {
      return false;
    }
    
    // 禁止空字节
    if (filename.includes('\0')) {
      return false;
    }
    
    // 禁止 .. 
    if (filename.includes('..')) {
      return false;
    }
    
    // 只允许安全字符
    const safePattern = /^[a-zA-Z0-9_\-\.]+$/;
    return safePattern.test(filename);
  }
}

const uploadsPath = new PathSecurity('./uploads');

// ✅ 安全的文件访问
app.get('/api/files/:filename', (req, res) => {
  const { filename } = req.params;

  // 验证文件名
  if (!uploadsPath.isValidFilename(filename)) {
    return res.status(400).json({ error: 'Invalid filename' });
  }

  // 获取安全路径
  const safePath = uploadsPath.getSafePath(filename);
  if (!safePath) {
    return res.status(403).json({ error: 'Access denied' });
  }

  // 检查文件是否存在
  if (!fs.existsSync(safePath)) {
    return res.status(404).json({ error: 'File not found' });
  }

  // 检查是否是文件（而不是目录）
  const stat = fs.statSync(safePath);
  if (!stat.isFile()) {
    return res.status(400).json({ error: 'Not a file' });
  }

  res.sendFile(safePath);
});

// ✅ 使用白名单
const allowedFiles = new Set(['readme.txt', 'license.txt', 'changelog.md']);

app.get('/api/docs/:filename', (req, res) => {
  const { filename } = req.params;

  if (!allowedFiles.has(filename)) {
    return res.status(404).json({ error: 'File not found' });
  }

  const safePath = path.join(__dirname, 'docs', filename);
  res.sendFile(safePath);
});

// ✅ 使用数据库存储文件信息
app.get('/api/files/:id', async (req, res) => {
  const { id } = req.params;

  // 从数据库获取文件信息
  const file = await prisma.file.findUnique({
    where: { id: parseInt(id) }
  });

  if (!file) {
    return res.status(404).json({ error: 'File not found' });
  }

  // 检查权限
  if (file.userId !== req.user.id && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied' });
  }

  // 使用数据库中存储的安全路径
  const safePath = path.join(
    process.env.UPLOAD_DIR!,
    file.storedFilename // 服务器生成的文件名
  );

  res.download(safePath, file.originalFilename);
});

// ✅ 使用随机文件名存储
import { v4 as uuidv4 } from 'uuid';

app.post('/api/upload', upload.single('file'), async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' });
  }

  // 生成安全的存储文件名
  const ext = path.extname(req.file.originalname);
  const storedFilename = `${uuidv4()}${ext}`;

  // 移动文件到存储目录
  const storedPath = path.join(process.env.UPLOAD_DIR!, storedFilename);
  await fs.promises.rename(req.file.path, storedPath);

  // 保存文件信息到数据库
  const file = await prisma.file.create({
    data: {
      originalFilename: req.file.originalname,
      storedFilename,
      mimeType: req.file.mimetype,
      size: req.file.size,
      userId: req.user.id
    }
  });

  res.json({ fileId: file.id });
});
```

### ZIP 文件路径遍历（Zip Slip）

```typescript
import unzipper from 'unzipper';

// ❌ 危险：直接解压
app.post('/api/extract', upload.single('archive'), async (req, res) => {
  const extractPath = './extracted';
  
  // ZIP 中的文件名可能包含 ../
  await fs.createReadStream(req.file.path)
    .pipe(unzipper.Extract({ path: extractPath }));
});

// ✅ 安全：验证每个文件路径
app.post('/api/extract', upload.single('archive'), async (req, res) => {
  const extractPath = path.resolve('./extracted');
  const pathSecurity = new PathSecurity(extractPath);

  const directory = await unzipper.Open.file(req.file.path);

  for (const entry of directory.files) {
    // 验证路径
    const targetPath = pathSecurity.getSafePath(entry.path);
    
    if (!targetPath) {
      console.warn(`Skipping unsafe path: ${entry.path}`);
      continue;
    }

    if (entry.type === 'Directory') {
      await fs.promises.mkdir(targetPath, { recursive: true });
    } else {
      // 确保目录存在
      await fs.promises.mkdir(path.dirname(targetPath), { recursive: true });
      
      // 解压文件
      const content = await entry.buffer();
      await fs.promises.writeFile(targetPath, content);
    }
  }

  res.json({ message: 'Extracted successfully' });
});
```

---

## 总结

### 安全检查清单

#### 认证与授权
- [ ] 使用强密码哈希（bcrypt/argon2）
- [ ] 实施密码强度策略
- [ ] 实现速率限制
- [ ] 支持 2FA
- [ ] 正确实施 RBAC/ABAC

#### 输入验证
- [ ] 验证所有输入
- [ ] 使用参数化查询
- [ ] 转义输出
- [ ] 限制文件上传

#### 会话管理
- [ ] 使用安全的 Session 配置
- [ ] 登录后重新生成 Session ID
- [ ] 设置合理的超时时间
- [ ] 使用 SameSite Cookie

#### CSRF 防护
- [ ] 实现 CSRF Token
- [ ] 使用 SameSite Cookie
- [ ] 验证 Origin/Referer

#### 加密
- [ ] 使用 HTTPS
- [ ] 加密敏感数据
- [ ] 安全存储密钥
- [ ] 使用最新的 TLS 版本

#### 响应头
- [ ] 配置 CSP
- [ ] 设置 HSTS
- [ ] 防止点击劫持
- [ ] 防止 MIME 嗅探

#### 高级防护
- [ ] 防护 SSRF 攻击
- [ ] 防护原型链污染
- [ ] 防护路径遍历
- [ ] ZIP 文件安全解压

#### 依赖管理
- [ ] 定期更新依赖
- [ ] 使用 npm audit
- [ ] 扫描漏洞

#### 监控与日志
- [ ] 记录安全事件
- [ ] 监控异常行为
- [ ] 实施告警机制

### 高级安全面试题

#### 6. SSRF 攻击如何防护？

**防护措施**：

1. **URL 白名单**：只允许访问特定域名
2. **IP 黑名单**：禁止私有 IP、本地回环、云元数据
3. **DNS 解析验证**：检查解析后的真实 IP
4. **禁用重定向**：或验证重定向目标
5. **协议限制**：只允许 HTTP/HTTPS
6. **端口限制**：只允许 80/443
7. **使用出口代理**：集中管理出口流量

#### 7. 原型链污染有哪些危害？

**危害**：

- 🔓 **权限绕过**：`{}.isAdmin = true`
- 💉 **注入攻击**：污染模板引擎
- 🛠️ **DoS 攻击**：污染核心方法
- 🔐 **认证绕过**：修改校验逻辑

**防护**：

- 使用 `Object.create(null)`
- 过滤危险键（`__proto__`、`constructor`）
- 使用 Map 代替普通对象
- 冻结原型

#### 8. 路径遍历如何防护？

**核心原则**：永远不信任用户输入的路径

**防护措施**：

1. **路径规范化**：`path.normalize()`
2. **路径解析验证**：检查解析后的路径是否在允许目录内
3. **文件名白名单**：只允许字母、数字、下划线
4. **数据库存储映射**：用 ID 查询，不用文件名
5. **随机文件名**：存储时使用 UUID

---

**上一篇**：[授权机制](./02-authorization.md)  
**下一篇**：[安全实践](./04-security-practices.md)

