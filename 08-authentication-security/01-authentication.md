# 认证机制

认证（Authentication）是验证用户身份的过程。本文深入讲解各种认证机制的原理、实现和最佳实践。

## 目录
- [认证基础](#认证基础)
- [Session 认证](#session-认证)
- [Token 认证](#token-认证)
- [JWT 认证](#jwt-认证)
- [OAuth 2.0](#oauth-20)
- [单点登录（SSO）](#单点登录)
- [多因素认证（MFA）](#多因素认证)
- [最佳实践](#最佳实践)
- [面试题](#常见面试题)

---

## 认证基础

### 认证 vs 授权

```typescript
// 认证（Authentication）：你是谁？
// 示例：验证用户名和密码
if (username === 'john' && password === 'secret') {
  authenticated = true; // 认证通过
}

// 授权（Authorization）：你能做什么？
// 示例：检查用户权限
if (user.role === 'admin') {
  authorized = true; // 有权限
}
```

### 常见认证方式

| 方式 | 特点 | 适用场景 |
|------|------|---------|
| **Basic Auth** | 简单，每次请求带用户名密码 | 内部 API |
| **Session** | 服务器存储状态 | 传统 Web 应用 |
| **Token（JWT）** | 无状态，客户端存储 | SPA、移动应用 |
| **OAuth 2.0** | 第三方授权 | 社交登录 |
| **SSO** | 单点登录 | 企业应用 |

---

## Session 认证

### 工作原理

```
1. 用户登录
   ↓
2. 服务器创建 Session，存储到内存/数据库/Redis
   ↓
3. 返回 Session ID（通过 Cookie）
   ↓
4. 后续请求携带 Session ID
   ↓
5. 服务器验证 Session
```

### 实现

```typescript
import express from 'express';
import session from 'express-session';
import RedisStore from 'connect-redis';
import Redis from 'ioredis';
import bcrypt from 'bcryptjs';
import { PrismaClient } from '@prisma/client';

const app = express();
const redis = new Redis();
const prisma = new PrismaClient();

// Session 配置
app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production', // HTTPS only
    httpOnly: true, // 防止 XSS
    maxAge: 24 * 60 * 60 * 1000, // 24 小时
    sameSite: 'strict' // 防止 CSRF
  },
  name: 'sessionId' // 自定义 Cookie 名称
}));

// 扩展 Session 类型
declare module 'express-session' {
  interface SessionData {
    userId: number;
    role: string;
  }
}

// 登录
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;

  try {
    // 1. 查找用户
    const user = await prisma.user.findUnique({
      where: { email }
    });

    if (!user) {
      return res.status(401).json({
        error: 'Invalid credentials'
      });
    }

    // 2. 验证密码
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      return res.status(401).json({
        error: 'Invalid credentials'
      });
    }

    // 3. 创建 Session
    req.session.userId = user.id;
    req.session.role = user.role;

    // 4. 保存 Session（确保写入存储）
    req.session.save((err) => {
      if (err) {
        return res.status(500).json({ error: 'Session save failed' });
      }

      res.json({
        message: 'Login successful',
        user: {
          id: user.id,
          name: user.name,
          email: user.email,
          role: user.role
        }
      });
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 登出
app.post('/api/logout', (req, res) => {
  req.session.destroy((err) => {
    if (err) {
      return res.status(500).json({ error: 'Logout failed' });
    }

    res.clearCookie('sessionId');
    res.json({ message: 'Logout successful' });
  });
});

// 获取当前用户
app.get('/api/me', async (req, res) => {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }

  const user = await prisma.user.findUnique({
    where: { id: req.session.userId },
    select: {
      id: true,
      name: true,
      email: true,
      role: true
    }
  });

  res.json({ user });
});

// 认证中间件
function requireAuth(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'Not authenticated' });
  }
  next();
}

// 受保护的路由
app.get('/api/protected', requireAuth, (req, res) => {
  res.json({ message: 'Protected resource', userId: req.session.userId });
});
```

### Session 存储

```typescript
// 1. 内存存储（仅开发）
import session from 'express-session';

app.use(session({
  secret: 'secret',
  resave: false,
  saveUninitialized: false
}));

// 2. Redis 存储（推荐生产环境）
import RedisStore from 'connect-redis';
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: Number(process.env.REDIS_PORT),
  password: process.env.REDIS_PASSWORD
});

app.use(session({
  store: new RedisStore({
    client: redis,
    prefix: 'sess:' // Session 键前缀
  }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    maxAge: 86400000 // 24 小时
  }
}));

// 3. 数据库存储
import connectPg from 'connect-pg-simple';
import pg from 'pg';

const PostgresStore = connectPg(session);
const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL
});

app.use(session({
  store: new PostgresStore({ pool }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false
}));
```

### Session 优缺点

**优点**：
- ✅ 服务器完全控制
- ✅ 可随时撤销
- ✅ 安全性高（信息存服务器）

**缺点**：
- ❌ 服务器存储压力
- ❌ 水平扩展困难（需要共享存储）
- ❌ 跨域困难

---

## Token 认证

### 工作原理

```
1. 用户登录
   ↓
2. 服务器生成 Token
   ↓
3. 返回 Token 给客户端
   ↓
4. 客户端存储 Token（LocalStorage/Cookie）
   ↓
5. 后续请求携带 Token（Authorization Header）
   ↓
6. 服务器验证 Token
```

### 简单 Token 实现

```typescript
import crypto from 'crypto';

class TokenManager {
  private redis: Redis;
  
  constructor(redis: Redis) {
    this.redis = redis;
  }

  // 生成 Token
  generateToken(): string {
    return crypto.randomBytes(32).toString('hex');
  }

  // 保存 Token
  async saveToken(userId: number, token: string): Promise<void> {
    const key = `token:${token}`;
    const value = JSON.stringify({ userId, createdAt: Date.now() });
    
    // 设置过期时间：24 小时
    await this.redis.setex(key, 86400, value);
    
    // 记录用户的所有 Token（用于登出所有设备）
    await this.redis.sadd(`user:${userId}:tokens`, token);
  }

  // 验证 Token
  async verifyToken(token: string): Promise<{ userId: number } | null> {
    const key = `token:${token}`;
    const value = await this.redis.get(key);
    
    if (!value) {
      return null;
    }
    
    const data = JSON.parse(value);
    return { userId: data.userId };
  }

  // 删除 Token（登出）
  async deleteToken(token: string): Promise<void> {
    const key = `token:${token}`;
    const value = await this.redis.get(key);
    
    if (value) {
      const { userId } = JSON.parse(value);
      await this.redis.del(key);
      await this.redis.srem(`user:${userId}:tokens`, token);
    }
  }

  // 删除用户所有 Token（登出所有设备）
  async deleteAllUserTokens(userId: number): Promise<void> {
    const tokens = await this.redis.smembers(`user:${userId}:tokens`);
    
    const pipeline = this.redis.pipeline();
    for (const token of tokens) {
      pipeline.del(`token:${token}`);
    }
    pipeline.del(`user:${userId}:tokens`);
    
    await pipeline.exec();
  }
}

const tokenManager = new TokenManager(redis);

// 登录
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;

  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 生成 Token
  const token = tokenManager.generateToken();
  await tokenManager.saveToken(user.id, token);

  res.json({
    message: 'Login successful',
    token,
    user: {
      id: user.id,
      name: user.name,
      email: user.email
    }
  });
});

// 登出
app.post('/api/logout', async (req, res) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (token) {
    await tokenManager.deleteToken(token);
  }
  
  res.json({ message: 'Logout successful' });
});

// 登出所有设备
app.post('/api/logout-all', requireAuth, async (req, res) => {
  await tokenManager.deleteAllUserTokens(req.user.id);
  res.json({ message: 'Logged out from all devices' });
});

// 认证中间件
async function requireAuth(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  const data = await tokenManager.verifyToken(token);
  if (!data) {
    return res.status(401).json({ error: 'Invalid token' });
  }

  req.user = await prisma.user.findUnique({
    where: { id: data.userId }
  });
  
  next();
}
```

---

## JWT 认证

### JWT 结构

```
Header.Payload.Signature

// Header（算法和类型）
{
  "alg": "HS256",
  "typ": "JWT"
}

// Payload（数据）
{
  "userId": 123,
  "email": "john@example.com",
  "role": "user",
  "iat": 1640000000,
  "exp": 1640086400
}

// Signature（签名）
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### JWT 实现

```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';

interface JWTPayload {
  userId: number;
  email: string;
  role: string;
}

class JWTManager {
  private readonly secret: string;
  private readonly expiresIn: string;

  constructor(secret: string, expiresIn: string = '24h') {
    this.secret = secret;
    this.expiresIn = expiresIn;
  }

  // 生成 Access Token
  generateAccessToken(payload: JWTPayload): string {
    return jwt.sign(payload, this.secret, {
      expiresIn: this.expiresIn
    });
  }

  // 生成 Refresh Token
  generateRefreshToken(userId: number): string {
    return jwt.sign(
      { userId, type: 'refresh' },
      this.secret,
      { expiresIn: '7d' }
    );
  }

  // 验证 Token
  verifyToken(token: string): JWTPayload {
    try {
      return jwt.verify(token, this.secret) as JWTPayload;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('Token expired');
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error('Invalid token');
      }
      throw error;
    }
  }

  // 解码 Token（不验证）
  decodeToken(token: string): JWTPayload | null {
    return jwt.decode(token) as JWTPayload | null;
  }
}

const jwtManager = new JWTManager(process.env.JWT_SECRET!);

// 登录
app.post('/api/login', async (req, res) => {
  const { email, password } = req.body;

  try {
    // 1. 验证用户
    const user = await prisma.user.findUnique({ where: { email } });
    if (!user || !await bcrypt.compare(password, user.password)) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // 2. 生成 Tokens
    const accessToken = jwtManager.generateAccessToken({
      userId: user.id,
      email: user.email,
      role: user.role
    });

    const refreshToken = jwtManager.generateRefreshToken(user.id);

    // 3. 保存 Refresh Token（用于撤销）
    await prisma.refreshToken.create({
      data: {
        token: refreshToken,
        userId: user.id,
        expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
      }
    });

    res.json({
      message: 'Login successful',
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role
      }
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// 刷新 Token
app.post('/api/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  if (!refreshToken) {
    return res.status(400).json({ error: 'Refresh token required' });
  }

  try {
    // 1. 验证 Refresh Token
    const payload = jwtManager.verifyToken(refreshToken);

    // 2. 检查 Refresh Token 是否在数据库中
    const storedToken = await prisma.refreshToken.findFirst({
      where: {
        token: refreshToken,
        userId: payload.userId,
        expiresAt: { gt: new Date() }
      }
    });

    if (!storedToken) {
      return res.status(401).json({ error: 'Invalid refresh token' });
    }

    // 3. 生成新的 Access Token
    const user = await prisma.user.findUnique({
      where: { id: payload.userId }
    });

    const accessToken = jwtManager.generateAccessToken({
      userId: user!.id,
      email: user!.email,
      role: user!.role
    });

    res.json({ accessToken });

  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

// 登出
app.post('/api/logout', async (req, res) => {
  const { refreshToken } = req.body;

  if (refreshToken) {
    // 删除 Refresh Token
    await prisma.refreshToken.deleteMany({
      where: { token: refreshToken }
    });
  }

  res.json({ message: 'Logout successful' });
});

// JWT 认证中间件
function requireJWT(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const payload = jwtManager.verifyToken(token);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({ error: error.message });
  }
}

// 受保护的路由
app.get('/api/me', requireJWT, async (req, res) => {
  const user = await prisma.user.findUnique({
    where: { id: req.user.userId },
    select: {
      id: true,
      name: true,
      email: true,
      role: true
    }
  });

  res.json({ user });
});
```

### Refresh Token 机制

```typescript
// 完整的 Token 刷新流程
class AuthService {
  async login(email: string, password: string) {
    // 验证用户...
    
    // 生成 Tokens
    const accessToken = this.generateAccessToken(user); // 15 分钟
    const refreshToken = this.generateRefreshToken(user); // 7 天
    
    // 保存 Refresh Token
    await this.saveRefreshToken(user.id, refreshToken);
    
    return { accessToken, refreshToken };
  }

  async refreshAccessToken(refreshToken: string) {
    // 验证 Refresh Token
    const payload = this.verifyRefreshToken(refreshToken);
    
    // 检查是否被撤销
    const isValid = await this.checkRefreshToken(refreshToken);
    if (!isValid) {
      throw new Error('Refresh token revoked');
    }
    
    // 生成新的 Access Token
    const accessToken = this.generateAccessToken(payload);
    
    return { accessToken };
  }

  async logout(refreshToken: string) {
    // 撤销 Refresh Token
    await this.revokeRefreshToken(refreshToken);
  }

  async logoutAllDevices(userId: number) {
    // 撤销用户所有 Refresh Token
    await prisma.refreshToken.deleteMany({
      where: { userId }
    });
  }
}
```

### JWT 优缺点

**优点**：
- ✅ 无状态（服务器不存储）
- ✅ 易于水平扩展
- ✅ 跨域友好
- ✅ 包含用户信息（减少数据库查询）

**缺点**：
- ❌ 无法主动撤销（需要额外机制）
- ❌ Token 较大（每次请求都携带）
- ❌ 敏感信息可被解码（不要存敏感数据）

---

## OAuth 2.0

### OAuth 2.0 流程

```
1. 用户点击"使用 Google 登录"
   ↓
2. 重定向到 Google 授权页面
   ↓
3. 用户授权
   ↓
4. Google 重定向回应用（带 Authorization Code）
   ↓
5. 应用用 Code 换取 Access Token
   ↓
6. 使用 Access Token 获取用户信息
   ↓
7. 创建本地用户会话
```

### OAuth 2.0 实现

```typescript
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';

// 配置 Google OAuth
passport.use(new GoogleStrategy({
    clientID: process.env.GOOGLE_CLIENT_ID!,
    clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    callbackURL: '/api/auth/google/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    try {
      // 查找或创建用户
      let user = await prisma.user.findUnique({
        where: { googleId: profile.id }
      });

      if (!user) {
        user = await prisma.user.create({
          data: {
            googleId: profile.id,
            email: profile.emails![0].value,
            name: profile.displayName,
            avatar: profile.photos![0].value
          }
        });
      }

      done(null, user);
    } catch (error) {
      done(error, undefined);
    }
  }
));

// 初始化 Passport
app.use(passport.initialize());

// Google 登录路由
app.get('/api/auth/google',
  passport.authenticate('google', {
    scope: ['profile', 'email'],
    session: false
  })
);

// Google 回调路由
app.get('/api/auth/google/callback',
  passport.authenticate('google', {
    session: false,
    failureRedirect: '/login'
  }),
  async (req, res) => {
    // 生成 JWT
    const token = jwtManager.generateAccessToken({
      userId: req.user.id,
      email: req.user.email,
      role: req.user.role
    });

    // 重定向到前端，带上 Token
    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
  }
);
```

### 多个 OAuth 提供商

```typescript
import { Strategy as GitHubStrategy } from 'passport-github2';
import { Strategy as FacebookStrategy } from 'passport-facebook';

// GitHub OAuth
passport.use(new GitHubStrategy({
    clientID: process.env.GITHUB_CLIENT_ID!,
    clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    callbackURL: '/api/auth/github/callback'
  },
  async (accessToken, refreshToken, profile, done) => {
    let user = await prisma.user.findUnique({
      where: { githubId: profile.id }
    });

    if (!user) {
      user = await prisma.user.create({
        data: {
          githubId: profile.id,
          email: profile.emails![0].value,
          name: profile.displayName,
          avatar: profile.photos![0].value
        }
      });
    }

    done(null, user);
  }
));

// Facebook OAuth
passport.use(new FacebookStrategy({
    clientID: process.env.FACEBOOK_APP_ID!,
    clientSecret: process.env.FACEBOOK_APP_SECRET!,
    callbackURL: '/api/auth/facebook/callback',
    profileFields: ['id', 'emails', 'name', 'picture']
  },
  async (accessToken, refreshToken, profile, done) => {
    let user = await prisma.user.findUnique({
      where: { facebookId: profile.id }
    });

    if (!user) {
      user = await prisma.user.create({
        data: {
          facebookId: profile.id,
          email: profile.emails![0].value,
          name: `${profile.name!.givenName} ${profile.name!.familyName}`,
          avatar: profile.photos![0].value
        }
      });
    }

    done(null, user);
  }
));

// GitHub 登录路由
app.get('/api/auth/github',
  passport.authenticate('github', { scope: ['user:email'] })
);

app.get('/api/auth/github/callback',
  passport.authenticate('github', { session: false }),
  handleOAuthCallback
);

// Facebook 登录路由
app.get('/api/auth/facebook',
  passport.authenticate('facebook', { scope: ['email'] })
);

app.get('/api/auth/facebook/callback',
  passport.authenticate('facebook', { session: false }),
  handleOAuthCallback
);

function handleOAuthCallback(req, res) {
  const token = jwtManager.generateAccessToken({
    userId: req.user.id,
    email: req.user.email,
    role: req.user.role
  });

  res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${token}`);
}
```

---

## 单点登录

### SSO 原理

```
应用 A     应用 B     应用 C
   ↓          ↓          ↓
   └──────────┴──────────┘
              ↓
          SSO 服务器
```

### 简单 SSO 实现

```typescript
// SSO 服务器
import express from 'express';
import jwt from 'jsonwebtoken';

const ssoApp = express();

// SSO 登录
ssoApp.post('/sso/login', async (req, res) => {
  const { email, password } = req.body;

  // 验证用户
  const user = await verifyUser(email, password);
  if (!user) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 生成 SSO Token
  const ssoToken = jwt.sign(
    { userId: user.id, email: user.email },
    process.env.SSO_SECRET!,
    { expiresIn: '1h' }
  );

  res.json({ ssoToken });
});

// SSO 验证
ssoApp.get('/sso/verify', (req, res) => {
  const { token } = req.query;

  try {
    const payload = jwt.verify(token as string, process.env.SSO_SECRET!);
    res.json({ valid: true, user: payload });
  } catch {
    res.json({ valid: false });
  }
});

// 应用 A（使用 SSO）
const appA = express();

appA.get('/login', (req, res) => {
  // 重定向到 SSO 登录页
  const returnUrl = encodeURIComponent(req.query.returnUrl || '/');
  res.redirect(`${process.env.SSO_URL}/login?returnUrl=${returnUrl}`);
});

appA.get('/auth/callback', async (req, res) => {
  const { ssoToken } = req.query;

  // 验证 SSO Token
  const response = await fetch(`${process.env.SSO_URL}/sso/verify?token=${ssoToken}`);
  const { valid, user } = await response.json();

  if (!valid) {
    return res.redirect('/login');
  }

  // 创建本地会话
  req.session.userId = user.userId;
  req.session.email = user.email;

  res.redirect('/dashboard');
});
```

---

## 多因素认证

### TOTP（Time-based OTP）

```typescript
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

// 启用 2FA
app.post('/api/2fa/enable', requireAuth, async (req, res) => {
  const user = req.user;

  // 生成密钥
  const secret = speakeasy.generateSecret({
    name: `MyApp (${user.email})`,
    issuer: 'MyApp'
  });

  // 保存密钥（临时，待验证后正式启用）
  await prisma.user.update({
    where: { id: user.id },
    data: { tempTwoFactorSecret: secret.base32 }
  });

  // 生成二维码
  const qrCode = await QRCode.toDataURL(secret.otpauth_url!);

  res.json({
    secret: secret.base32,
    qrCode
  });
});

// 验证并启用 2FA
app.post('/api/2fa/verify', requireAuth, async (req, res) => {
  const { token } = req.body;
  const user = req.user;

  // 获取临时密钥
  const userData = await prisma.user.findUnique({
    where: { id: user.id }
  });

  if (!userData?.tempTwoFactorSecret) {
    return res.status(400).json({ error: '2FA not initiated' });
  }

  // 验证 Token
  const verified = speakeasy.totp.verify({
    secret: userData.tempTwoFactorSecret,
    encoding: 'base32',
    token,
    window: 2 // 允许前后 2 个时间窗口
  });

  if (!verified) {
    return res.status(400).json({ error: 'Invalid token' });
  }

  // 启用 2FA
  await prisma.user.update({
    where: { id: user.id },
    data: {
      twoFactorSecret: userData.tempTwoFactorSecret,
      twoFactorEnabled: true,
      tempTwoFactorSecret: null
    }
  });

  res.json({ message: '2FA enabled successfully' });
});

// 登录（带 2FA）
app.post('/api/login', async (req, res) => {
  const { email, password, twoFactorToken } = req.body;

  // 1. 验证用户名密码
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  // 2. 检查是否启用 2FA
  if (user.twoFactorEnabled) {
    if (!twoFactorToken) {
      return res.status(200).json({
        requires2FA: true,
        message: 'Please provide 2FA token'
      });
    }

    // 验证 2FA Token
    const verified = speakeasy.totp.verify({
      secret: user.twoFactorSecret!,
      encoding: 'base32',
      token: twoFactorToken,
      window: 2
    });

    if (!verified) {
      return res.status(401).json({ error: 'Invalid 2FA token' });
    }
  }

  // 3. 生成 JWT
  const token = jwtManager.generateAccessToken({
    userId: user.id,
    email: user.email,
    role: user.role
  });

  res.json({
    message: 'Login successful',
    token,
    user: {
      id: user.id,
      name: user.name,
      email: user.email
    }
  });
});
```

### SMS 验证码

```typescript
import twilio from 'twilio';

const twilioClient = twilio(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

// 发送验证码
app.post('/api/send-code', async (req, res) => {
  const { phone } = req.body;

  // 生成 6 位验证码
  const code = Math.floor(100000 + Math.random() * 900000).toString();

  // 保存到 Redis（5 分钟过期）
  await redis.setex(`sms:${phone}`, 300, code);

  // 发送短信
  await twilioClient.messages.create({
    body: `Your verification code is: ${code}`,
    from: process.env.TWILIO_PHONE_NUMBER,
    to: phone
  });

  res.json({ message: 'Code sent successfully' });
});

// 验证验证码
app.post('/api/verify-code', async (req, res) => {
  const { phone, code } = req.body;

  // 从 Redis 获取验证码
  const savedCode = await redis.get(`sms:${phone}`);

  if (!savedCode || savedCode !== code) {
    return res.status(400).json({ error: 'Invalid code' });
  }

  // 删除验证码
  await redis.del(`sms:${phone}`);

  res.json({ message: 'Verification successful' });
});
```

---

## 最佳实践

### 1. 密码安全

```typescript
import bcrypt from 'bcryptjs';

// ✅ 使用 bcrypt（推荐）
const saltRounds = 10;
const hashedPassword = await bcrypt.hash(password, saltRounds);

// ✅ 使用 argon2（更安全）
import argon2 from 'argon2';
const hashedPassword = await argon2.hash(password);

// ❌ 不要使用 MD5、SHA1
const hash = crypto.createHash('md5').update(password).digest('hex'); // 不安全！
```

### 2. Token 安全

```typescript
// ✅ 使用 httpOnly Cookie 存储 Refresh Token
res.cookie('refreshToken', refreshToken, {
  httpOnly: true,  // 防止 XSS
  secure: true,    // HTTPS only
  sameSite: 'strict', // 防止 CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000
});

// ❌ 不要在 LocalStorage 存储敏感 Token
localStorage.setItem('refreshToken', token); // 容易被 XSS 攻击
```

### 3. 限流保护

```typescript
import rateLimit from 'express-rate-limit';

// 登录限流
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 5, // 最多 5 次尝试
  message: 'Too many login attempts, please try again later'
});

app.post('/api/login', loginLimiter, loginHandler);
```

### 4. 账户锁定

```typescript
async function handleFailedLogin(userId: number) {
  const key = `login_attempts:${userId}`;
  
  // 增加失败次数
  const attempts = await redis.incr(key);
  
  // 第一次失败，设置过期时间
  if (attempts === 1) {
    await redis.expire(key, 900); // 15 分钟
  }

  // 超过 5 次，锁定账户
  if (attempts >= 5) {
    await prisma.user.update({
      where: { id: userId },
      data: { lockedUntil: new Date(Date.now() + 30 * 60 * 1000) } // 锁定 30 分钟
    });
  }

  return attempts;
}
```

---

## 常见面试题

### 1. Session vs JWT，如何选择？

| 场景 | 推荐 | 理由 |
|------|------|------|
| 传统 Web 应用 | Session | 服务器完全控制 |
| SPA/移动应用 | JWT | 无状态，易扩展 |
| 需要立即撤销 | Session | 可直接删除 |
| 分布式系统 | JWT | 无需共享存储 |
| 高安全要求 | Session + 2FA | 更安全 |

### 2. JWT 如何撤销？

**方案**：

1. **黑名单**：将撤销的 Token 加入 Redis
```typescript
await redis.setex(`blacklist:${token}`, ttl, '1');
```

2. **Token 版本号**：用户密码修改时，递增版本号
```typescript
// JWT Payload
{ userId: 123, version: 5 }

// 验证时检查版本
if (payload.version !== user.currentVersion) {
  throw new Error('Token revoked');
}
```

3. **Refresh Token**：Access Token 短期，通过 Refresh Token 控制

### 3. OAuth 2.0 的优势？

**优势**：
- ✅ 用户无需创建新密码
- ✅ 应用不存储用户密码
- ✅ 用户体验好（一键登录）
- ✅ 利用第三方的安全措施

**缺点**：
- ❌ 依赖第三方服务
- ❌ 需要处理多个账号关联
- ❌ 隐私问题

### 4. 如何防止暴力破解？

**方法**：

1. **限流**：限制登录尝试次数
2. **验证码**：多次失败后要求验证码
3. **账户锁定**：临时锁定账户
4. **慢哈希**：使用 bcrypt/argon2（计算慢）
5. **监控告警**：异常登录行为告警

### 5. 2FA 的原理？

**TOTP 原理**：

```
密钥（服务器和客户端共享）
    ↓
当前时间 / 30秒
    ↓
HMAC-SHA1 算法
    ↓
6 位数字
```

- 每 30 秒生成一个新的 6 位数字
- 服务器和客户端使用相同算法
- 只要时间同步，生成的数字就一致

---

## 总结

### 认证方式选择

| 方式 | 适用场景 | 优先级 |
|------|---------|--------|
| **Session** | 传统 Web 应用 | ⭐⭐⭐ |
| **JWT** | SPA、移动应用 | ⭐⭐⭐⭐⭐ |
| **OAuth 2.0** | 社交登录 | ⭐⭐⭐⭐ |
| **SSO** | 企业多应用 | ⭐⭐⭐ |
| **2FA** | 高安全要求 | ⭐⭐⭐⭐ |

### 实践检查清单

- [ ] 是否使用安全的密码哈希？
- [ ] 是否实现了限流保护？
- [ ] Token 是否安全存储？
- [ ] 是否实现了 Refresh Token？
- [ ] 是否有账户锁定机制？
- [ ] 是否支持 OAuth 登录？
- [ ] 是否支持 2FA？
- [ ] 是否记录登录日志？
- [ ] 是否有异常登录检测？
- [ ] 是否定期更新密钥？

---

**下一篇**：[授权机制](./02-authorization.md)

