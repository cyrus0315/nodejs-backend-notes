# HTTP/HTTPS 协议详解

HTTP 是 Web 应用的基础协议。本文深入讲解 HTTP 协议的方方面面，包括 HTTPS 加密原理。

## 目录
- [HTTP 基础](#http-基础)
- [HTTP 方法](#http-方法)
- [HTTP 状态码](#http-状态码)
- [HTTP Headers](#http-headers)
- [HTTP 缓存](#http-缓存)
- [HTTP 版本演进](#http-版本演进)
- [HTTPS](#https)
- [Node.js HTTP 编程](#nodejs-http-编程)
- [面试题](#常见面试题)

---

## HTTP 基础

### 什么是 HTTP

```
HTTP（HyperText Transfer Protocol）超文本传输协议

特点：
1. 应用层协议（基于 TCP）
2. 无状态（每次请求独立）
3. 请求-响应模式
4. 文本协议（HTTP/1.x）或二进制协议（HTTP/2+）
```

### HTTP 请求结构

```http
GET /api/users HTTP/1.1          # 请求行（方法 + 路径 + 版本）
Host: api.example.com            # 请求头
Accept: application/json
Authorization: Bearer xxx
User-Agent: Mozilla/5.0

{"name": "John"}                 # 请求体（可选）
```

### HTTP 响应结构

```http
HTTP/1.1 200 OK                  # 状态行（版本 + 状态码 + 描述）
Content-Type: application/json   # 响应头
Content-Length: 45
Cache-Control: max-age=3600

{"id": 1, "name": "John"}        # 响应体
```

---

## HTTP 方法

### 常用方法

| 方法 | 幂等 | 安全 | 请求体 | 说明 |
|------|------|------|--------|------|
| **GET** | ✅ | ✅ | ❌ | 获取资源 |
| **POST** | ❌ | ❌ | ✅ | 创建资源 |
| **PUT** | ✅ | ❌ | ✅ | 替换资源（全量更新） |
| **PATCH** | ❌ | ❌ | ✅ | 修改资源（部分更新） |
| **DELETE** | ✅ | ❌ | ❌ | 删除资源 |
| **HEAD** | ✅ | ✅ | ❌ | 获取响应头（无响应体） |
| **OPTIONS** | ✅ | ✅ | ❌ | 获取支持的方法（预检请求） |

### 幂等性说明

```typescript
// 幂等性：多次执行同一请求，效果相同

// ✅ GET 是幂等的
GET /users/1  // 多次请求，结果相同

// ❌ POST 不是幂等的
POST /users   // 多次请求，创建多个用户
{ "name": "John" }

// ✅ PUT 是幂等的
PUT /users/1  // 多次请求，最终状态相同
{ "name": "John", "age": 30 }

// ❌ PATCH 不一定是幂等的
PATCH /users/1
{ "age": { "$inc": 1 } }  // 每次执行 age 都会增加

// ✅ DELETE 是幂等的
DELETE /users/1  // 多次请求，资源都被删除（即使已删除）
```

### 安全性说明

```typescript
// 安全性：不会修改服务器状态

// ✅ 安全方法：GET, HEAD, OPTIONS
// 不会产生副作用

// ❌ 不安全方法：POST, PUT, PATCH, DELETE
// 会修改服务器状态
```

---

## HTTP 状态码

### 状态码分类

| 范围 | 类别 | 说明 |
|------|------|------|
| **1xx** | 信息 | 请求已接收，继续处理 |
| **2xx** | 成功 | 请求成功处理 |
| **3xx** | 重定向 | 需要进一步操作 |
| **4xx** | 客户端错误 | 请求有误 |
| **5xx** | 服务器错误 | 服务器处理失败 |

### 常用状态码

```typescript
// 2xx 成功
200 OK              // 请求成功
201 Created         // 资源创建成功（POST）
204 No Content      // 成功，无返回内容（DELETE）

// 3xx 重定向
301 Moved Permanently   // 永久重定向（GET 可能变 GET）
302 Found              // 临时重定向（GET 可能变 GET）
303 See Other          // 临时重定向（强制 GET）
304 Not Modified       // 缓存可用
307 Temporary Redirect // 临时重定向（保持方法）
308 Permanent Redirect // 永久重定向（保持方法）

// 4xx 客户端错误
400 Bad Request        // 请求格式错误
401 Unauthorized       // 未认证
403 Forbidden          // 无权限
404 Not Found          // 资源不存在
405 Method Not Allowed // 方法不支持
409 Conflict           // 资源冲突
422 Unprocessable Entity // 验证失败
429 Too Many Requests  // 请求过多（限流）

// 5xx 服务器错误
500 Internal Server Error // 服务器错误
502 Bad Gateway        // 网关错误（上游服务不可用）
503 Service Unavailable // 服务不可用（过载/维护）
504 Gateway Timeout    // 网关超时
```

### 状态码使用建议

```typescript
// Express 示例
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await User.findById(req.params.id);
    
    if (!user) {
      return res.status(404).json({
        error: 'User not found'
      });
    }
    
    res.status(200).json(user);
  } catch (error) {
    if (error.name === 'ValidationError') {
      return res.status(400).json({
        error: error.message
      });
    }
    
    res.status(500).json({
      error: 'Internal server error'
    });
  }
});

app.post('/api/users', async (req, res) => {
  try {
    const user = await User.create(req.body);
    
    // 201 Created，返回新资源
    res.status(201).json(user);
  } catch (error) {
    if (error.code === 11000) {
      // MongoDB 唯一键冲突
      return res.status(409).json({
        error: 'User already exists'
      });
    }
    
    res.status(500).json({
      error: 'Internal server error'
    });
  }
});

app.delete('/api/users/:id', async (req, res) => {
  await User.findByIdAndDelete(req.params.id);
  
  // 204 No Content
  res.status(204).send();
});
```

---

## HTTP Headers

### 请求头

```http
# 通用头
Host: api.example.com           # 目标主机（必需）
Connection: keep-alive          # 连接方式
Accept: application/json        # 期望的响应类型
Accept-Language: en-US,en       # 期望的语言
Accept-Encoding: gzip, deflate  # 支持的压缩格式
User-Agent: Mozilla/5.0         # 客户端信息

# 认证
Authorization: Bearer xxx        # 认证信息
Cookie: session=abc123          # Cookie

# 缓存
Cache-Control: no-cache         # 缓存控制
If-None-Match: "etag123"        # 条件请求（ETag）
If-Modified-Since: Wed, ...     # 条件请求（时间）

# 内容
Content-Type: application/json  # 请求体类型
Content-Length: 123             # 请求体长度

# CORS
Origin: https://example.com     # 请求来源

# 自定义
X-Request-ID: abc123            # 请求追踪
X-Forwarded-For: 1.2.3.4        # 原始 IP（代理）
```

### 响应头

```http
# 内容
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Content-Encoding: gzip          # 压缩方式

# 缓存
Cache-Control: max-age=3600     # 缓存控制
ETag: "abc123"                  # 资源标识
Last-Modified: Wed, ...         # 最后修改时间
Expires: Thu, ...               # 过期时间

# CORS
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400

# 安全
Strict-Transport-Security: max-age=31536000
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'

# 重定向
Location: https://example.com/new-url

# Cookie
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

### Content-Type

```typescript
// 常见 Content-Type
const contentTypes = {
  // 文本
  'text/plain': '纯文本',
  'text/html': 'HTML',
  'text/css': 'CSS',
  'text/javascript': 'JavaScript',

  // 应用
  'application/json': 'JSON',
  'application/xml': 'XML',
  'application/x-www-form-urlencoded': '表单数据',
  'application/octet-stream': '二进制流',

  // 多部分
  'multipart/form-data': '文件上传',

  // 图片
  'image/jpeg': 'JPEG 图片',
  'image/png': 'PNG 图片',
  'image/gif': 'GIF 图片',
  'image/webp': 'WebP 图片'
};

// Express 解析
import express from 'express';

const app = express();

// JSON 解析
app.use(express.json());

// URL 编码解析
app.use(express.urlencoded({ extended: true }));

// 文件上传（multer）
import multer from 'multer';
const upload = multer({ dest: 'uploads/' });

app.post('/upload', upload.single('file'), (req, res) => {
  res.json({ file: req.file });
});
```

---

## HTTP 缓存

### 缓存类型

```
1. 浏览器缓存（本地）
2. 代理缓存（CDN、Nginx）
3. 服务器缓存（Redis、内存）
```

### Cache-Control

```http
# 缓存控制指令
Cache-Control: public            # 任何缓存都可以缓存
Cache-Control: private           # 只有浏览器可以缓存
Cache-Control: no-cache          # 需要验证后使用缓存
Cache-Control: no-store          # 不缓存
Cache-Control: max-age=3600      # 缓存有效期（秒）
Cache-Control: s-maxage=3600     # 共享缓存有效期
Cache-Control: must-revalidate   # 过期后必须验证
Cache-Control: immutable         # 永不变化

# 组合使用
Cache-Control: public, max-age=31536000, immutable  # 静态资源
Cache-Control: private, no-cache                     # API 响应
Cache-Control: no-store                              # 敏感数据
```

### 条件请求

```typescript
// ETag 验证
// 服务端返回 ETag
res.setHeader('ETag', '"abc123"');
res.setHeader('Cache-Control', 'no-cache');

// 客户端发送 If-None-Match
// If-None-Match: "abc123"

// 服务端验证
app.get('/api/data', (req, res) => {
  const currentETag = generateETag(data);
  const clientETag = req.headers['if-none-match'];

  if (clientETag === currentETag) {
    return res.status(304).send(); // Not Modified
  }

  res.setHeader('ETag', currentETag);
  res.json(data);
});

// Last-Modified 验证
// 服务端返回 Last-Modified
res.setHeader('Last-Modified', 'Wed, 21 Oct 2024 07:28:00 GMT');

// 客户端发送 If-Modified-Since
// If-Modified-Since: Wed, 21 Oct 2024 07:28:00 GMT

// 服务端验证
app.get('/api/data', (req, res) => {
  const lastModified = new Date(data.updatedAt);
  const ifModifiedSince = new Date(req.headers['if-modified-since']);

  if (lastModified <= ifModifiedSince) {
    return res.status(304).send();
  }

  res.setHeader('Last-Modified', lastModified.toUTCString());
  res.json(data);
});
```

### 缓存策略

```typescript
// Express 缓存配置
import express from 'express';

const app = express();

// 静态资源：长期缓存
app.use('/static', express.static('public', {
  maxAge: '1y',           // 1 年
  immutable: true,        // 不可变
  etag: false,            // 不需要 ETag
  lastModified: false     // 不需要 Last-Modified
}));

// API 响应：不缓存或短期缓存
app.get('/api/users', (req, res) => {
  res.setHeader('Cache-Control', 'private, no-cache');
  res.json(users);
});

// 公共数据：适当缓存
app.get('/api/config', (req, res) => {
  res.setHeader('Cache-Control', 'public, max-age=300'); // 5 分钟
  res.json(config);
});

// 敏感数据：不缓存
app.get('/api/user/profile', (req, res) => {
  res.setHeader('Cache-Control', 'no-store');
  res.json(profile);
});
```

### CDN 缓存

```typescript
// CDN 缓存策略
// s-maxage 优先于 max-age（针对 CDN）
res.setHeader('Cache-Control', 'public, max-age=60, s-maxage=3600');
// 浏览器缓存 60 秒，CDN 缓存 1 小时

// Vary 头：按 Header 变化缓存
res.setHeader('Vary', 'Accept-Encoding, Accept-Language');
// 不同压缩格式、不同语言分别缓存

// 缓存清除
// CDN 通常支持通过 API 清除缓存
// 或使用版本化 URL：/static/app.v1.0.0.js
```

---

## HTTP 版本演进

### HTTP/1.0 vs HTTP/1.1

```
HTTP/1.0 的问题：
1. 每个请求需要新建 TCP 连接
2. 无 Host 头，不支持虚拟主机
3. 无持久连接

HTTP/1.1 改进：
1. 持久连接（Keep-Alive）默认开启
2. 管道化（Pipelining）
3. 分块传输（Chunked Transfer）
4. Host 头必需
5. 更多缓存控制
```

### HTTP/1.1 vs HTTP/2

```
HTTP/1.1 的问题：
1. 队头阻塞（Head-of-Line Blocking）
2. 头部冗余（每次都发送完整头部）
3. 单向请求（客户端发起）

HTTP/2 改进：
1. 二进制分帧（Binary Framing）
2. 多路复用（Multiplexing）
3. 头部压缩（HPACK）
4. 服务器推送（Server Push）
5. 流量控制和优先级
```

```typescript
// HTTP/2 多路复用示意
// 在单个 TCP 连接上同时发送多个请求/响应

// HTTP/1.1（需要多个连接或串行）
连接1: 请求A → 响应A
连接2: 请求B → 响应B
连接3: 请求C → 响应C

// HTTP/2（单个连接，并行）
连接1: [请求A, 请求B, 请求C] → [响应B, 响应A, 响应C]
```

### HTTP/2 vs HTTP/3

```
HTTP/2 的问题：
1. TCP 队头阻塞（丢包影响所有流）
2. TCP 握手延迟
3. 中间设备可能不支持

HTTP/3 改进：
1. 基于 QUIC（UDP）
2. 无队头阻塞（流独立）
3. 0-RTT 握手
4. 连接迁移（换 IP 不断连接）
```

### Node.js HTTP/2

```typescript
import http2 from 'http2';
import fs from 'fs';

// 创建 HTTP/2 服务器
const server = http2.createSecureServer({
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt')
});

server.on('stream', (stream, headers) => {
  const path = headers[':path'];
  const method = headers[':method'];

  if (path === '/') {
    // 响应 HTML
    stream.respond({
      ':status': 200,
      'content-type': 'text/html'
    });
    stream.end('<html>...</html>');

    // 服务器推送
    stream.pushStream({ ':path': '/style.css' }, (err, pushStream) => {
      if (!err) {
        pushStream.respond({
          ':status': 200,
          'content-type': 'text/css'
        });
        pushStream.end(fs.readFileSync('style.css'));
      }
    });
  }
});

server.listen(443);
```

---

## HTTPS

### HTTPS 工作原理

```
HTTPS = HTTP + TLS/SSL

加密方式：
1. 对称加密：加密数据（AES）
2. 非对称加密：交换密钥（RSA/ECDH）
3. 数字签名：验证身份
4. 证书：验证服务器
```

### TLS 握手过程

```
TLS 1.2 握手（2-RTT）：

1. Client Hello
   - 支持的 TLS 版本
   - 支持的加密套件
   - 随机数 (Client Random)

2. Server Hello
   - 选择的 TLS 版本
   - 选择的加密套件
   - 随机数 (Server Random)
   - 服务器证书
   - Server Key Exchange (DH 参数)
   - Server Hello Done

3. Client Key Exchange
   - Pre-Master Secret (用服务器公钥加密)
   - Change Cipher Spec
   - Finished

4. Server
   - Change Cipher Spec
   - Finished

5. 开始加密通信
   - 使用 Master Secret 对称加密
```

```
TLS 1.3 握手（1-RTT）：

1. Client Hello
   - 支持的加密套件
   - Key Share (DH 公钥)
   - 随机数

2. Server Hello
   - 选择的加密套件
   - Key Share (DH 公钥)
   - 证书
   - Finished

3. Client
   - Finished

4. 开始加密通信

（支持 0-RTT 恢复会话）
```

### 证书验证

```typescript
// 证书链
根证书 (CA)
  └─ 中间证书
      └─ 服务器证书

// 验证过程
1. 验证证书签名（用上级公钥验证）
2. 验证证书有效期
3. 验证证书域名
4. 检查证书吊销状态（CRL/OCSP）
```

### Node.js HTTPS

```typescript
import https from 'https';
import fs from 'fs';
import express from 'express';

const app = express();

// HTTPS 选项
const options = {
  key: fs.readFileSync('server.key'),
  cert: fs.readFileSync('server.crt'),
  ca: fs.readFileSync('ca.crt'), // 可选：CA 证书

  // TLS 配置
  minVersion: 'TLSv1.2',
  ciphers: [
    'ECDHE-ECDSA-AES128-GCM-SHA256',
    'ECDHE-RSA-AES128-GCM-SHA256',
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES256-GCM-SHA384'
  ].join(':'),

  // HSTS
  honorCipherOrder: true
};

// 创建 HTTPS 服务器
const server = https.createServer(options, app);

// HTTP 重定向到 HTTPS
import http from 'http';

http.createServer((req, res) => {
  res.writeHead(301, {
    Location: `https://${req.headers.host}${req.url}`
  });
  res.end();
}).listen(80);

server.listen(443);
```

### Let's Encrypt 证书

```typescript
// 使用 greenlock-express 自动获取证书
import greenlock from 'greenlock-express';

greenlock.init({
  packageRoot: __dirname,
  configDir: './greenlock.d',
  maintainerEmail: 'admin@example.com',
  cluster: false
}).ready((glx) => {
  const httpsServer = glx.httpsServer();
  const app = require('./app');
  
  httpsServer.on('request', app);
});
```

---

## Node.js HTTP 编程

### http 模块

```typescript
import http from 'http';

// 创建服务器
const server = http.createServer((req, res) => {
  // 请求信息
  console.log('Method:', req.method);
  console.log('URL:', req.url);
  console.log('Headers:', req.headers);

  // 读取请求体
  let body = '';
  req.on('data', (chunk) => {
    body += chunk.toString();
  });

  req.on('end', () => {
    // 设置响应头
    res.setHeader('Content-Type', 'application/json');
    res.statusCode = 200;

    // 发送响应
    res.end(JSON.stringify({
      message: 'Hello',
      body: body
    }));
  });
});

server.listen(3000);
```

### 发送 HTTP 请求

```typescript
import http from 'http';
import https from 'https';

// 使用 http 模块
function httpRequest(url: string): Promise<string> {
  return new Promise((resolve, reject) => {
    const client = url.startsWith('https') ? https : http;

    client.get(url, (res) => {
      let data = '';
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => resolve(data));
    }).on('error', reject);
  });
}

// 使用 fetch（Node.js 18+）
const response = await fetch('https://api.example.com/data', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ name: 'John' })
});

const data = await response.json();

// 使用 axios
import axios from 'axios';

const client = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000,
  headers: {
    'Authorization': 'Bearer xxx'
  }
});

// 请求拦截器
client.interceptors.request.use((config) => {
  config.headers['X-Request-ID'] = crypto.randomUUID();
  return config;
});

// 响应拦截器
client.interceptors.response.use(
  (response) => response.data,
  (error) => {
    if (error.response?.status === 401) {
      // 处理认证错误
    }
    return Promise.reject(error);
  }
);

const users = await client.get('/users');
```

### Keep-Alive

```typescript
import http from 'http';
import https from 'https';

// 创建 Agent 复用连接
const httpAgent = new http.Agent({
  keepAlive: true,
  keepAliveMsecs: 1000,
  maxSockets: 50,           // 最大并发连接
  maxFreeSockets: 10,       // 最大空闲连接
  timeout: 60000            // 超时
});

const httpsAgent = new https.Agent({
  keepAlive: true,
  keepAliveMsecs: 1000,
  maxSockets: 50
});

// axios 使用 Agent
import axios from 'axios';

const client = axios.create({
  httpAgent,
  httpsAgent
});

// 获取 Agent 状态
console.log(httpAgent.freeSockets);
console.log(httpAgent.sockets);
```

---

## 常见面试题

### 1. HTTP/1.1 vs HTTP/2 的主要区别？

**答案**：

| 特性 | HTTP/1.1 | HTTP/2 |
|------|---------|--------|
| **协议格式** | 文本 | 二进制 |
| **连接复用** | 串行（队头阻塞） | 多路复用 |
| **头部** | 每次完整发送 | HPACK 压缩 |
| **服务器推送** | ❌ | ✅ |
| **优先级** | ❌ | ✅ |

**关键点**：
- HTTP/2 在单个连接上并行发送多个请求/响应
- 解决了 HTTP/1.1 的队头阻塞问题
- 头部压缩减少了传输数据量

### 2. HTTPS 握手过程？

**简化版答案**：

1. **Client Hello**：客户端发送支持的加密套件
2. **Server Hello**：服务器选择加密套件，发送证书
3. **密钥交换**：客户端验证证书，交换密钥
4. **完成**：双方生成对称密钥，开始加密通信

**关键点**：
- 非对称加密交换密钥
- 对称加密传输数据
- 证书验证服务器身份

### 3. GET vs POST 的区别？

| 特性 | GET | POST |
|------|-----|------|
| **幂等性** | ✅ 幂等 | ❌ 不幂等 |
| **安全性** | ✅ 安全（不修改数据） | ❌ 不安全 |
| **请求体** | ❌ 通常无 | ✅ 有 |
| **缓存** | ✅ 可缓存 | ❌ 默认不缓存 |
| **书签** | ✅ 可收藏 | ❌ 不可收藏 |
| **长度限制** | URL 有长度限制 | 无限制 |
| **数据位置** | URL 参数 | 请求体 |

### 4. HTTP 缓存机制？

**两种缓存策略**：

1. **强缓存**：不发请求，直接用缓存
   - `Cache-Control: max-age=3600`
   - `Expires: Thu, 01 Dec 2024 16:00:00 GMT`

2. **协商缓存**：发请求验证，可能返回 304
   - `ETag` / `If-None-Match`
   - `Last-Modified` / `If-Modified-Since`

**优先级**：Cache-Control > Expires > ETag > Last-Modified

### 5. 常见状态码及使用场景？

```typescript
// 2xx 成功
200 OK           // 请求成功
201 Created      // 创建资源成功
204 No Content   // 删除成功

// 3xx 重定向
301              // 永久重定向（SEO）
302              // 临时重定向
304              // 缓存可用

// 4xx 客户端错误
400              // 请求格式错误
401              // 未登录
403              // 无权限
404              // 资源不存在
429              // 请求过多

// 5xx 服务器错误
500              // 服务器错误
502              // 网关错误
503              // 服务不可用
504              // 网关超时
```

### 6. 什么是 CORS？如何处理？

**CORS（跨域资源共享）**：

当浏览器发起跨域请求时，需要服务器返回正确的 CORS 头。

```typescript
// Express CORS 配置
import cors from 'cors';

app.use(cors({
  origin: ['https://example.com', 'https://app.example.com'],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  maxAge: 86400
}));

// 预检请求（OPTIONS）
// 浏览器在发送「非简单请求」前会发送 OPTIONS 请求
```

---

## 总结

### HTTP 知识图谱

```
HTTP 协议
├── 基础
│   ├── 请求/响应结构
│   ├── 方法（GET, POST, PUT, DELETE...）
│   └── 状态码（2xx, 3xx, 4xx, 5xx）
├── Headers
│   ├── 请求头
│   ├── 响应头
│   └── Content-Type
├── 缓存
│   ├── 强缓存（Cache-Control）
│   └── 协商缓存（ETag, Last-Modified）
├── 版本
│   ├── HTTP/1.0
│   ├── HTTP/1.1
│   ├── HTTP/2
│   └── HTTP/3
└── HTTPS
    ├── TLS 握手
    └── 证书验证
```

### 实践检查清单

- [ ] 理解 HTTP 方法和幂等性
- [ ] 掌握常用状态码
- [ ] 理解 HTTP 缓存机制
- [ ] 了解 HTTP/2 的改进
- [ ] 理解 HTTPS 工作原理
- [ ] 会配置 CORS
- [ ] 掌握 Node.js HTTP 编程

---

**下一篇**：[WebSocket 实时通信](./02-websocket.md)

