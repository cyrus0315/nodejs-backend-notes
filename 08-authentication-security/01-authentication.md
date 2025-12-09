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

## WebAuthn/FIDO2（无密码认证）

### WebAuthn 简介

WebAuthn 是 W3C 标准，允许使用生物识别、安全密钥等方式进行无密码认证。

```
优势：
- 🔐 抗钓鱼（基于域名绑定）
- 🚀 用户体验好（指纹、Face ID）
- 💪 高安全性（私钥永不离开设备）
- 🔑 无需记忆密码
```

### WebAuthn 流程

```
注册流程：
1. 服务器生成 challenge
   ↓
2. 浏览器调用 navigator.credentials.create()
   ↓
3. 认证器（如 YubiKey、Touch ID）生成密钥对
   ↓
4. 返回公钥给服务器
   ↓
5. 服务器存储公钥

认证流程：
1. 服务器生成 challenge
   ↓
2. 浏览器调用 navigator.credentials.get()
   ↓
3. 认证器使用私钥签名 challenge
   ↓
4. 服务器验证签名
```

### WebAuthn 实现

```typescript
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
  generateAuthenticationOptions,
  verifyAuthenticationResponse
} from '@simplewebauthn/server';
import { isoUint8Array } from '@simplewebauthn/server/helpers';

// 配置
const rpName = 'My App';
const rpID = 'example.com';
const origin = `https://${rpID}`;

// ===== 注册流程 =====

// 1. 生成注册选项
app.post('/api/webauthn/register/options', requireAuth, async (req, res) => {
  const user = req.user;

  // 获取用户已有的认证器
  const userAuthenticators = await prisma.authenticator.findMany({
    where: { userId: user.id }
  });

  const options = await generateRegistrationOptions({
    rpName,
    rpID,
    userID: isoUint8Array.fromUTF8String(user.id.toString()),
    userName: user.email,
    userDisplayName: user.name,
    // 排除已注册的认证器
    excludeCredentials: userAuthenticators.map(auth => ({
      id: auth.credentialID,
      type: 'public-key',
      transports: auth.transports
    })),
    authenticatorSelection: {
      // 跨平台认证器（如 YubiKey）或平台认证器（如 Touch ID）
      authenticatorAttachment: 'platform',
      residentKey: 'preferred',
      userVerification: 'preferred'
    }
  });

  // 保存 challenge 用于验证
  await redis.setex(`webauthn:challenge:${user.id}`, 300, options.challenge);

  res.json(options);
});

// 2. 验证注册响应
app.post('/api/webauthn/register/verify', requireAuth, async (req, res) => {
  const user = req.user;
  const { body } = req;

  // 获取保存的 challenge
  const expectedChallenge = await redis.get(`webauthn:challenge:${user.id}`);
  if (!expectedChallenge) {
    return res.status(400).json({ error: 'Challenge expired' });
  }

  try {
    const verification = await verifyRegistrationResponse({
      response: body,
      expectedChallenge,
      expectedOrigin: origin,
      expectedRPID: rpID
    });

    if (verification.verified && verification.registrationInfo) {
      const { credentialPublicKey, credentialID, counter } = verification.registrationInfo;

      // 保存认证器信息
      await prisma.authenticator.create({
        data: {
          userId: user.id,
          credentialID: Buffer.from(credentialID),
          credentialPublicKey: Buffer.from(credentialPublicKey),
          counter,
          transports: body.response.transports || []
        }
      });

      // 清除 challenge
      await redis.del(`webauthn:challenge:${user.id}`);

      res.json({ verified: true });
    } else {
      res.status(400).json({ error: 'Verification failed' });
    }
  } catch (error) {
    console.error('WebAuthn registration error:', error);
    res.status(400).json({ error: error.message });
  }
});

// ===== 认证流程 =====

// 3. 生成认证选项
app.post('/api/webauthn/authenticate/options', async (req, res) => {
  const { email } = req.body;

  const user = await prisma.user.findUnique({
    where: { email },
    include: { authenticators: true }
  });

  if (!user || user.authenticators.length === 0) {
    return res.status(400).json({ error: 'No authenticators found' });
  }

  const options = await generateAuthenticationOptions({
    rpID,
    allowCredentials: user.authenticators.map(auth => ({
      id: auth.credentialID,
      type: 'public-key',
      transports: auth.transports
    })),
    userVerification: 'preferred'
  });

  // 保存 challenge
  await redis.setex(`webauthn:auth:challenge:${email}`, 300, options.challenge);

  res.json(options);
});

// 4. 验证认证响应
app.post('/api/webauthn/authenticate/verify', async (req, res) => {
  const { email, ...body } = req.body;

  const user = await prisma.user.findUnique({
    where: { email },
    include: { authenticators: true }
  });

  if (!user) {
    return res.status(400).json({ error: 'User not found' });
  }

  const expectedChallenge = await redis.get(`webauthn:auth:challenge:${email}`);
  if (!expectedChallenge) {
    return res.status(400).json({ error: 'Challenge expired' });
  }

  // 找到使用的认证器
  const authenticator = user.authenticators.find(
    auth => Buffer.from(auth.credentialID).equals(Buffer.from(body.id, 'base64url'))
  );

  if (!authenticator) {
    return res.status(400).json({ error: 'Authenticator not found' });
  }

  try {
    const verification = await verifyAuthenticationResponse({
      response: body,
      expectedChallenge,
      expectedOrigin: origin,
      expectedRPID: rpID,
      authenticator: {
        credentialID: authenticator.credentialID,
        credentialPublicKey: authenticator.credentialPublicKey,
        counter: authenticator.counter
      }
    });

    if (verification.verified) {
      // 更新计数器（防止重放攻击）
      await prisma.authenticator.update({
        where: { id: authenticator.id },
        data: { counter: verification.authenticationInfo.newCounter }
      });

      // 清除 challenge
      await redis.del(`webauthn:auth:challenge:${email}`);

      // 生成 JWT
      const token = jwtManager.generateAccessToken({
        userId: user.id,
        email: user.email,
        role: user.role
      });

      res.json({ verified: true, token });
    } else {
      res.status(400).json({ error: 'Verification failed' });
    }
  } catch (error) {
    console.error('WebAuthn authentication error:', error);
    res.status(400).json({ error: error.message });
  }
});
```

### 前端实现

```typescript
// 注册
async function registerWebAuthn() {
  // 1. 获取注册选项
  const optionsRes = await fetch('/api/webauthn/register/options', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${token}` }
  });
  const options = await optionsRes.json();

  // 2. 调用浏览器 API
  const credential = await navigator.credentials.create({
    publicKey: {
      ...options,
      challenge: base64urlToBuffer(options.challenge),
      user: {
        ...options.user,
        id: base64urlToBuffer(options.user.id)
      },
      excludeCredentials: options.excludeCredentials?.map(cred => ({
        ...cred,
        id: base64urlToBuffer(cred.id)
      }))
    }
  });

  // 3. 发送给服务器验证
  const verifyRes = await fetch('/api/webauthn/register/verify', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      id: credential.id,
      rawId: bufferToBase64url(credential.rawId),
      response: {
        clientDataJSON: bufferToBase64url(credential.response.clientDataJSON),
        attestationObject: bufferToBase64url(credential.response.attestationObject),
        transports: credential.response.getTransports?.() || []
      },
      type: credential.type
    })
  });

  return verifyRes.json();
}

// 认证
async function authenticateWebAuthn(email: string) {
  // 1. 获取认证选项
  const optionsRes = await fetch('/api/webauthn/authenticate/options', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email })
  });
  const options = await optionsRes.json();

  // 2. 调用浏览器 API
  const credential = await navigator.credentials.get({
    publicKey: {
      ...options,
      challenge: base64urlToBuffer(options.challenge),
      allowCredentials: options.allowCredentials?.map(cred => ({
        ...cred,
        id: base64urlToBuffer(cred.id)
      }))
    }
  });

  // 3. 发送给服务器验证
  const verifyRes = await fetch('/api/webauthn/authenticate/verify', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email,
      id: credential.id,
      rawId: bufferToBase64url(credential.rawId),
      response: {
        clientDataJSON: bufferToBase64url(credential.response.clientDataJSON),
        authenticatorData: bufferToBase64url(credential.response.authenticatorData),
        signature: bufferToBase64url(credential.response.signature),
        userHandle: credential.response.userHandle
          ? bufferToBase64url(credential.response.userHandle)
          : null
      },
      type: credential.type
    })
  });

  return verifyRes.json();
}

// 工具函数
function base64urlToBuffer(base64url: string): ArrayBuffer {
  const base64 = base64url.replace(/-/g, '+').replace(/_/g, '/');
  const padding = '='.repeat((4 - base64.length % 4) % 4);
  const binary = atob(base64 + padding);
  return Uint8Array.from(binary, c => c.charCodeAt(0)).buffer;
}

function bufferToBase64url(buffer: ArrayBuffer): string {
  const bytes = new Uint8Array(buffer);
  let binary = '';
  bytes.forEach(b => binary += String.fromCharCode(b));
  return btoa(binary).replace(/\+/g, '-').replace(/\//g, '_').replace(/=/g, '');
}
```

---

## Magic Link（魔术链接）

### Magic Link 原理

通过邮件发送一次性登录链接，用户点击即可登录，无需密码。

```
流程：
1. 用户输入邮箱
   ↓
2. 服务器生成一次性 Token
   ↓
3. 发送带 Token 的链接到邮箱
   ↓
4. 用户点击链接
   ↓
5. 服务器验证 Token
   ↓
6. 登录成功，生成 Session/JWT
```

### Magic Link 实现

```typescript
import crypto from 'crypto';
import nodemailer from 'nodemailer';

// 邮件配置
const transporter = nodemailer.createTransport({
  host: process.env.SMTP_HOST,
  port: Number(process.env.SMTP_PORT),
  secure: true,
  auth: {
    user: process.env.SMTP_USER,
    pass: process.env.SMTP_PASS
  }
});

// 1. 请求 Magic Link
app.post('/api/auth/magic-link', async (req, res) => {
  const { email } = req.body;

  // 验证邮箱格式
  if (!email || !validator.isEmail(email)) {
    return res.status(400).json({ error: 'Invalid email' });
  }

  // 检查用户是否存在（或自动创建）
  let user = await prisma.user.findUnique({ where: { email } });
  if (!user) {
    // 可选：自动注册
    user = await prisma.user.create({
      data: { email, name: email.split('@')[0] }
    });
  }

  // 生成安全 Token
  const token = crypto.randomBytes(32).toString('hex');
  const hashedToken = crypto.createHash('sha256').update(token).digest('hex');

  // 保存 Token（15 分钟过期）
  await prisma.magicLinkToken.create({
    data: {
      token: hashedToken,
      userId: user.id,
      expiresAt: new Date(Date.now() + 15 * 60 * 1000)
    }
  });

  // 构建 Magic Link
  const magicLink = `${process.env.FRONTEND_URL}/auth/magic?token=${token}&email=${encodeURIComponent(email)}`;

  // 发送邮件
  await transporter.sendMail({
    from: `"My App" <${process.env.SMTP_FROM}>`,
    to: email,
    subject: '登录链接 - My App',
    html: `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h2>登录 My App</h2>
        <p>点击下面的按钮登录你的账户：</p>
        <a href="${magicLink}" 
           style="display: inline-block; background: #4F46E5; color: white; 
                  padding: 12px 24px; text-decoration: none; border-radius: 6px;">
          登录
        </a>
        <p style="margin-top: 20px; color: #666;">
          此链接将在 15 分钟后过期。如果你没有请求此链接，请忽略此邮件。
        </p>
        <p style="color: #999; font-size: 12px;">
          或者复制此链接到浏览器：<br>
          ${magicLink}
        </p>
      </div>
    `
  });

  res.json({ message: 'Magic link sent to your email' });
});

// 2. 验证 Magic Link
app.post('/api/auth/magic-link/verify', async (req, res) => {
  const { token, email } = req.body;

  if (!token || !email) {
    return res.status(400).json({ error: 'Missing token or email' });
  }

  // 哈希 Token 进行比对
  const hashedToken = crypto.createHash('sha256').update(token).digest('hex');

  // 查找并验证 Token
  const magicLinkToken = await prisma.magicLinkToken.findFirst({
    where: {
      token: hashedToken,
      user: { email },
      expiresAt: { gt: new Date() },
      usedAt: null
    },
    include: { user: true }
  });

  if (!magicLinkToken) {
    return res.status(400).json({ error: 'Invalid or expired token' });
  }

  // 标记 Token 已使用
  await prisma.magicLinkToken.update({
    where: { id: magicLinkToken.id },
    data: { usedAt: new Date() }
  });

  // 生成 JWT
  const accessToken = jwtManager.generateAccessToken({
    userId: magicLinkToken.user.id,
    email: magicLinkToken.user.email,
    role: magicLinkToken.user.role
  });

  const refreshToken = jwtManager.generateRefreshToken(magicLinkToken.user.id);

  // 保存 Refresh Token
  await prisma.refreshToken.create({
    data: {
      token: refreshToken,
      userId: magicLinkToken.user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    }
  });

  res.json({
    message: 'Login successful',
    accessToken,
    refreshToken,
    user: {
      id: magicLinkToken.user.id,
      name: magicLinkToken.user.name,
      email: magicLinkToken.user.email
    }
  });
});

// 3. 清理过期 Token（定时任务）
async function cleanupExpiredMagicLinks() {
  await prisma.magicLinkToken.deleteMany({
    where: {
      OR: [
        { expiresAt: { lt: new Date() } },
        { usedAt: { not: null } }
      ]
    }
  });
}

// 每小时清理一次
setInterval(cleanupExpiredMagicLinks, 60 * 60 * 1000);
```

### Magic Link 安全考虑

```typescript
// 1. 限流（防止滥用）
const magicLinkLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 小时
  max: 3, // 每小时最多 3 次
  keyGenerator: (req) => req.body.email,
  message: 'Too many magic link requests'
});

app.post('/api/auth/magic-link', magicLinkLimiter, requestMagicLink);

// 2. Token 一次性使用
// 验证后立即标记为已使用

// 3. 短过期时间
// 15-30 分钟为宜

// 4. 哈希存储
// 数据库存储哈希后的 Token

// 5. 安全的 Token 生成
const token = crypto.randomBytes(32).toString('hex'); // 256 bits
```

---

## OpenID Connect (OIDC)

### OIDC 简介

OIDC 是建立在 OAuth 2.0 之上的身份认证协议，提供了标准化的身份验证流程。

```
OAuth 2.0 vs OIDC：
- OAuth 2.0：授权（Access Token）
- OIDC：认证（ID Token）+ 授权

OIDC 增加的内容：
- ID Token（JWT 格式）
- UserInfo 端点
- 标准化的用户属性（claims）
- Discovery（自动发现配置）
```

### OIDC 实现（作为客户端）

```typescript
import { Issuer, generators } from 'openid-client';

// 1. 配置 OIDC Provider
async function setupOIDC() {
  // 自动发现配置
  const googleIssuer = await Issuer.discover('https://accounts.google.com');
  
  // 创建客户端
  const client = new googleIssuer.Client({
    client_id: process.env.GOOGLE_CLIENT_ID!,
    client_secret: process.env.GOOGLE_CLIENT_SECRET!,
    redirect_uris: ['https://example.com/callback'],
    response_types: ['code']
  });

  return client;
}

const oidcClient = await setupOIDC();

// 2. 发起认证
app.get('/api/auth/oidc/login', async (req, res) => {
  // 生成安全参数
  const codeVerifier = generators.codeVerifier();
  const codeChallenge = generators.codeChallenge(codeVerifier);
  const state = generators.state();
  const nonce = generators.nonce();

  // 保存到 Session
  req.session.oidc = { codeVerifier, state, nonce };

  // 生成授权 URL
  const authorizationUrl = oidcClient.authorizationUrl({
    scope: 'openid email profile',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state,
    nonce
  });

  res.redirect(authorizationUrl);
});

// 3. 处理回调
app.get('/callback', async (req, res) => {
  const { codeVerifier, state, nonce } = req.session.oidc;

  // 验证 state
  if (req.query.state !== state) {
    return res.status(400).json({ error: 'State mismatch' });
  }

  try {
    // 交换 Token
    const tokenSet = await oidcClient.callback(
      'https://example.com/callback',
      req.query,
      { code_verifier: codeVerifier, state, nonce }
    );

    // 验证 ID Token
    const claims = tokenSet.claims();
    console.log('ID Token claims:', claims);
    // {
    //   sub: '1234567890',
    //   email: 'user@example.com',
    //   email_verified: true,
    //   name: 'John Doe',
    //   picture: 'https://...',
    //   iat: 1234567890,
    //   exp: 1234567890,
    //   nonce: '...'
    // }

    // 获取用户信息（可选）
    const userinfo = await oidcClient.userinfo(tokenSet.access_token!);
    console.log('UserInfo:', userinfo);

    // 创建或更新用户
    let user = await prisma.user.findUnique({
      where: { email: claims.email }
    });

    if (!user) {
      user = await prisma.user.create({
        data: {
          email: claims.email,
          name: claims.name,
          avatar: claims.picture,
          emailVerified: claims.email_verified,
          googleId: claims.sub
        }
      });
    }

    // 生成应用的 JWT
    const accessToken = jwtManager.generateAccessToken({
      userId: user.id,
      email: user.email,
      role: user.role
    });

    // 清除 OIDC Session 数据
    delete req.session.oidc;

    res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${accessToken}`);

  } catch (error) {
    console.error('OIDC callback error:', error);
    res.redirect('/login?error=auth_failed');
  }
});

// 4. 登出
app.get('/api/auth/oidc/logout', async (req, res) => {
  const endSessionUrl = oidcClient.endSessionUrl({
    post_logout_redirect_uri: 'https://example.com'
  });

  res.redirect(endSessionUrl);
});
```

### OIDC 作为 Provider（自建身份认证服务）

```typescript
import Provider from 'oidc-provider';

// 配置 OIDC Provider
const oidc = new Provider('https://auth.example.com', {
  clients: [{
    client_id: 'client1',
    client_secret: 'secret1',
    redirect_uris: ['https://app.example.com/callback'],
    grant_types: ['authorization_code', 'refresh_token'],
    response_types: ['code']
  }],
  
  // 用户查找
  async findAccount(ctx, id) {
    const user = await prisma.user.findUnique({ where: { id: parseInt(id) } });
    if (!user) return undefined;
    
    return {
      accountId: id,
      async claims(use, scope) {
        return {
          sub: id,
          email: user.email,
          email_verified: user.emailVerified,
          name: user.name,
          picture: user.avatar
        };
      }
    };
  },

  // Token 存储
  adapter: new PrismaAdapter(), // 自定义适配器

  // 功能配置
  features: {
    devInteractions: { enabled: false },
    deviceFlow: { enabled: true },
    introspection: { enabled: true },
    revocation: { enabled: true }
  },

  // Cookie 配置
  cookies: {
    keys: [process.env.COOKIE_SECRET!]
  },

  // JWT 配置
  jwks: {
    keys: [/* RSA 或 EC 密钥 */]
  },

  // 声明配置
  claims: {
    openid: ['sub'],
    email: ['email', 'email_verified'],
    profile: ['name', 'picture']
  },

  // TTL 配置
  ttl: {
    AccessToken: 3600,
    AuthorizationCode: 600,
    IdToken: 3600,
    RefreshToken: 1209600
  }
});

// 集成到 Express
app.use('/oidc', oidc.callback());
```

---

## OAuth 2.0 PKCE

### PKCE 简介

PKCE（Proof Key for Code Exchange）用于保护 Authorization Code 流程，特别适用于移动应用和 SPA。

```
问题：Authorization Code 可能被截获
解决：使用 Code Verifier 和 Code Challenge 绑定请求

流程：
1. 客户端生成 code_verifier（随机字符串）
   ↓
2. 计算 code_challenge = SHA256(code_verifier)
   ↓
3. 发送授权请求（带 code_challenge）
   ↓
4. 获取 authorization_code
   ↓
5. 用 code + code_verifier 换取 token
   ↓
6. 服务器验证 SHA256(code_verifier) === code_challenge
```

### PKCE 实现

```typescript
import crypto from 'crypto';

class PKCEService {
  // 生成 Code Verifier（43-128 字符）
  generateCodeVerifier(): string {
    return crypto.randomBytes(32)
      .toString('base64url')
      .slice(0, 43);
  }

  // 生成 Code Challenge
  generateCodeChallenge(verifier: string): string {
    return crypto
      .createHash('sha256')
      .update(verifier)
      .digest('base64url');
  }

  // 验证 Code Verifier
  verifyCodeChallenge(verifier: string, challenge: string): boolean {
    const computedChallenge = this.generateCodeChallenge(verifier);
    return crypto.timingSafeEqual(
      Buffer.from(computedChallenge),
      Buffer.from(challenge)
    );
  }
}

const pkceService = new PKCEService();

// OAuth 授权端点
app.get('/oauth/authorize', async (req, res) => {
  const {
    client_id,
    redirect_uri,
    response_type,
    scope,
    state,
    code_challenge,
    code_challenge_method
  } = req.query;

  // 验证客户端
  const client = await prisma.oauthClient.findUnique({
    where: { clientId: client_id as string }
  });

  if (!client || !client.redirectUris.includes(redirect_uri as string)) {
    return res.status(400).json({ error: 'invalid_client' });
  }

  // 验证 PKCE（对于公开客户端是必须的）
  if (client.tokenEndpointAuthMethod === 'none') {
    if (!code_challenge || code_challenge_method !== 'S256') {
      return res.status(400).json({
        error: 'invalid_request',
        error_description: 'PKCE required for public clients'
      });
    }
  }

  // 渲染授权页面或自动授权
  res.render('authorize', {
    client,
    scope,
    state,
    code_challenge,
    code_challenge_method
  });
});

// 用户同意后生成授权码
app.post('/oauth/authorize', requireAuth, async (req, res) => {
  const {
    client_id,
    redirect_uri,
    scope,
    state,
    code_challenge,
    code_challenge_method
  } = req.body;

  // 生成授权码
  const authorizationCode = crypto.randomBytes(32).toString('hex');

  // 保存授权码（10 分钟过期）
  await prisma.authorizationCode.create({
    data: {
      code: authorizationCode,
      clientId: client_id,
      userId: req.user.id,
      redirectUri: redirect_uri,
      scope,
      codeChallenge: code_challenge,
      codeChallengeMethod: code_challenge_method,
      expiresAt: new Date(Date.now() + 10 * 60 * 1000)
    }
  });

  // 重定向回客户端
  const params = new URLSearchParams({
    code: authorizationCode,
    state
  });

  res.redirect(`${redirect_uri}?${params}`);
});

// Token 端点
app.post('/oauth/token', async (req, res) => {
  const {
    grant_type,
    code,
    redirect_uri,
    client_id,
    client_secret,
    code_verifier
  } = req.body;

  if (grant_type !== 'authorization_code') {
    return res.status(400).json({ error: 'unsupported_grant_type' });
  }

  // 查找授权码
  const authCode = await prisma.authorizationCode.findUnique({
    where: { code },
    include: { user: true }
  });

  if (!authCode || authCode.expiresAt < new Date()) {
    return res.status(400).json({ error: 'invalid_grant' });
  }

  // 验证客户端
  const client = await prisma.oauthClient.findUnique({
    where: { clientId: client_id }
  });

  if (!client) {
    return res.status(400).json({ error: 'invalid_client' });
  }

  // 验证客户端密钥（机密客户端）
  if (client.tokenEndpointAuthMethod === 'client_secret_post') {
    if (client.clientSecret !== client_secret) {
      return res.status(400).json({ error: 'invalid_client' });
    }
  }

  // 验证 PKCE
  if (authCode.codeChallenge) {
    if (!code_verifier) {
      return res.status(400).json({
        error: 'invalid_request',
        error_description: 'code_verifier required'
      });
    }

    const valid = pkceService.verifyCodeChallenge(
      code_verifier,
      authCode.codeChallenge
    );

    if (!valid) {
      return res.status(400).json({ error: 'invalid_grant' });
    }
  }

  // 验证 redirect_uri
  if (authCode.redirectUri !== redirect_uri) {
    return res.status(400).json({ error: 'invalid_grant' });
  }

  // 删除授权码（一次性使用）
  await prisma.authorizationCode.delete({ where: { code } });

  // 生成 Token
  const accessToken = jwtManager.generateAccessToken({
    userId: authCode.user.id,
    email: authCode.user.email,
    clientId: client_id,
    scope: authCode.scope
  });

  const refreshToken = jwtManager.generateRefreshToken(authCode.user.id);

  res.json({
    access_token: accessToken,
    token_type: 'Bearer',
    expires_in: 3600,
    refresh_token: refreshToken,
    scope: authCode.scope
  });
});
```

### 前端 PKCE 流程

```typescript
class OAuthClient {
  private clientId: string;
  private redirectUri: string;
  private authorizationEndpoint: string;
  private tokenEndpoint: string;

  constructor(config: OAuthConfig) {
    this.clientId = config.clientId;
    this.redirectUri = config.redirectUri;
    this.authorizationEndpoint = config.authorizationEndpoint;
    this.tokenEndpoint = config.tokenEndpoint;
  }

  // 生成 PKCE 参数
  private async generatePKCE() {
    const codeVerifier = this.generateRandomString(43);
    const codeChallenge = await this.sha256(codeVerifier);
    
    return { codeVerifier, codeChallenge };
  }

  private generateRandomString(length: number): string {
    const array = new Uint8Array(length);
    crypto.getRandomValues(array);
    return btoa(String.fromCharCode(...array))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '')
      .slice(0, length);
  }

  private async sha256(plain: string): Promise<string> {
    const encoder = new TextEncoder();
    const data = encoder.encode(plain);
    const hash = await crypto.subtle.digest('SHA-256', data);
    return btoa(String.fromCharCode(...new Uint8Array(hash)))
      .replace(/\+/g, '-')
      .replace(/\//g, '_')
      .replace(/=+$/, '');
  }

  // 发起授权
  async authorize() {
    const { codeVerifier, codeChallenge } = await this.generatePKCE();
    const state = this.generateRandomString(32);

    // 保存到 sessionStorage
    sessionStorage.setItem('oauth_code_verifier', codeVerifier);
    sessionStorage.setItem('oauth_state', state);

    const params = new URLSearchParams({
      response_type: 'code',
      client_id: this.clientId,
      redirect_uri: this.redirectUri,
      scope: 'openid email profile',
      state,
      code_challenge: codeChallenge,
      code_challenge_method: 'S256'
    });

    window.location.href = `${this.authorizationEndpoint}?${params}`;
  }

  // 处理回调
  async handleCallback(): Promise<TokenResponse> {
    const params = new URLSearchParams(window.location.search);
    const code = params.get('code');
    const state = params.get('state');

    // 验证 state
    const savedState = sessionStorage.getItem('oauth_state');
    if (state !== savedState) {
      throw new Error('State mismatch');
    }

    // 获取 code_verifier
    const codeVerifier = sessionStorage.getItem('oauth_code_verifier');
    if (!codeVerifier) {
      throw new Error('Code verifier not found');
    }

    // 清除存储
    sessionStorage.removeItem('oauth_state');
    sessionStorage.removeItem('oauth_code_verifier');

    // 交换 Token
    const response = await fetch(this.tokenEndpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: new URLSearchParams({
        grant_type: 'authorization_code',
        code: code!,
        redirect_uri: this.redirectUri,
        client_id: this.clientId,
        code_verifier: codeVerifier
      })
    });

    if (!response.ok) {
      throw new Error('Token exchange failed');
    }

    return response.json();
  }
}

// 使用
const oauthClient = new OAuthClient({
  clientId: 'my-app',
  redirectUri: 'https://app.example.com/callback',
  authorizationEndpoint: 'https://auth.example.com/oauth/authorize',
  tokenEndpoint: 'https://auth.example.com/oauth/token'
});

// 登录按钮
document.getElementById('login').onclick = () => oauthClient.authorize();

// 回调页面
if (window.location.pathname === '/callback') {
  oauthClient.handleCallback()
    .then(tokens => {
      localStorage.setItem('access_token', tokens.access_token);
      window.location.href = '/dashboard';
    })
    .catch(error => {
      console.error('OAuth error:', error);
      window.location.href = '/login?error=auth_failed';
    });
}
```

---

## 总结

### 认证方式选择

| 方式 | 适用场景 | 优先级 |
|------|---------|--------|
| **Session** | 传统 Web 应用 | ⭐⭐⭐ |
| **JWT** | SPA、移动应用 | ⭐⭐⭐⭐⭐ |
| **OAuth 2.0** | 社交登录 | ⭐⭐⭐⭐ |
| **OIDC** | 企业 SSO、标准化身份认证 | ⭐⭐⭐⭐⭐ |
| **SSO** | 企业多应用 | ⭐⭐⭐ |
| **2FA** | 高安全要求 | ⭐⭐⭐⭐ |
| **WebAuthn** | 无密码、高安全 | ⭐⭐⭐⭐⭐ |
| **Magic Link** | 简单登录、无密码 | ⭐⭐⭐ |

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
- [ ] 是否考虑 WebAuthn 无密码认证？
- [ ] 是否使用 PKCE 保护 OAuth 流程？
- [ ] 是否支持 OIDC 标准？

### 高级认证面试题

#### 6. WebAuthn 的优势是什么？

**优势**：
- 🔐 **抗钓鱼**：基于域名绑定，无法在假冒网站使用
- 🚀 **无密码**：使用生物识别或硬件密钥
- 💪 **高安全**：私钥永不离开设备
- ⚡ **便捷**：指纹/Face ID 一触即登

**挑战**：
- 浏览器兼容性
- 用户设备支持
- 恢复流程设计

#### 7. PKCE 解决什么问题？

**问题**：Authorization Code 可能被截获（特别是移动/SPA 应用）

**解决方案**：
1. 客户端生成 `code_verifier`（随机字符串）
2. 计算 `code_challenge = SHA256(code_verifier)`
3. 授权请求携带 `code_challenge`
4. Token 请求携带 `code_verifier`
5. 服务器验证两者匹配

**适用场景**：所有公开客户端（移动应用、SPA）

#### 8. OIDC vs OAuth 2.0？

| 特性 | OAuth 2.0 | OIDC |
|------|-----------|------|
| **目的** | 授权 | 认证 + 授权 |
| **Token** | Access Token | Access Token + ID Token |
| **用户信息** | 自定义 | 标准化 UserInfo |
| **发现** | 无 | Discovery 端点 |

---

**下一篇**：[授权机制](./02-authorization.md)

