# 代理与负载均衡

代理和负载均衡是构建高可用、高性能系统的关键技术。本文深入讲解它们的原理和实践。

## 目录
- [代理](#代理)
- [Nginx 配置](#nginx-配置)
- [负载均衡](#负载均衡)
- [CDN](#cdn)
- [API Gateway](#api-gateway)
- [Node.js 代理实现](#nodejs-代理实现)
- [面试题](#常见面试题)

---

## 代理

### 正向代理 vs 反向代理

```
正向代理（Forward Proxy）：
- 代理客户端
- 客户端知道代理存在
- 服务器不知道真实客户端

客户端 → [正向代理] → 互联网 → 服务器

用途：
- 翻墙、科学上网
- 访问内网资源
- 缓存加速
- 匿名访问


反向代理（Reverse Proxy）：
- 代理服务器
- 服务器知道代理存在
- 客户端不知道真实服务器

客户端 → 互联网 → [反向代理] → 服务器集群

用途：
- 负载均衡
- SSL 终止
- 缓存
- 安全防护
- 压缩
```

### 代理协议

```typescript
// HTTP 代理
// 使用 CONNECT 方法建立隧道（HTTPS）
CONNECT example.com:443 HTTP/1.1
Host: example.com:443

// HTTP 代理请求
GET http://example.com/path HTTP/1.1
Host: example.com

// SOCKS 代理
// 支持 TCP/UDP，更底层
// SOCKS4: 只支持 TCP
// SOCKS5: 支持 TCP/UDP，支持认证

// WebSocket 代理
// 需要处理 Upgrade 头
```

---

## Nginx 配置

### 基础反向代理

```nginx
# /etc/nginx/nginx.conf

http {
    # 上游服务器组
    upstream backend {
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
        server 127.0.0.1:3003;
    }

    server {
        listen 80;
        server_name example.com;

        # 静态文件
        location /static/ {
            alias /var/www/static/;
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # API 反向代理
        location /api/ {
            proxy_pass http://backend;
            
            # 代理头设置
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # 超时设置
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            # 缓冲设置
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }

        # WebSocket 代理
        location /ws/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_read_timeout 3600s;
        }
    }
}
```

### HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL 证书
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # SSL 配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;

    location / {
        proxy_pass http://backend;
    }
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### 缓存配置

```nginx
http {
    # 缓存路径配置
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;

    server {
        location /api/ {
            proxy_pass http://backend;
            
            # 启用缓存
            proxy_cache my_cache;
            
            # 缓存键
            proxy_cache_key "$scheme$request_method$host$request_uri";
            
            # 缓存有效期
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            
            # 缓存状态头
            add_header X-Cache-Status $upstream_cache_status;
            
            # 绕过缓存条件
            proxy_cache_bypass $http_authorization;
            
            # 锁定防止缓存击穿
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;
        }
    }
}
```

### 压缩配置

```nginx
http {
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/xml+rss
        image/svg+xml;

    # Brotli 压缩（需要模块）
    brotli on;
    brotli_comp_level 6;
    brotli_types
        text/plain
        text/css
        application/json
        application/javascript;
}
```

---

## 负载均衡

### 负载均衡算法

```nginx
# 1. 轮询（Round Robin）- 默认
upstream backend {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

# 2. 加权轮询（Weighted Round Robin）
upstream backend {
    server 127.0.0.1:3001 weight=5;  # 5/8 流量
    server 127.0.0.1:3002 weight=2;  # 2/8 流量
    server 127.0.0.1:3003 weight=1;  # 1/8 流量
}

# 3. IP Hash（会话保持）
upstream backend {
    ip_hash;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

# 4. 最少连接（Least Connections）
upstream backend {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

# 5. 一致性哈希
upstream backend {
    hash $request_uri consistent;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

# 6. 随机（Random）
upstream backend {
    random two least_conn;  # 随机选两个，再选连接少的
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}
```

### 健康检查

```nginx
upstream backend {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003 backup;  # 备用服务器

    # 被动健康检查（免费版）
    # max_fails: 失败次数阈值
    # fail_timeout: 失败后暂停时间
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;

    # 主动健康检查（Nginx Plus）
    # health_check interval=5s;
}

# 配置失败判定
server {
    location / {
        proxy_pass http://backend;
        
        # 超时算作失败
        proxy_connect_timeout 5s;
        proxy_read_timeout 5s;
        
        # 重试
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
        proxy_next_upstream_tries 3;
        proxy_next_upstream_timeout 10s;
    }
}
```

### L4 vs L7 负载均衡

```
L4 负载均衡（传输层）：
- 基于 IP 和端口
- 性能高，延迟低
- 不解析 HTTP 内容
- 无法基于 URL 路由

L7 负载均衡（应用层）：
- 基于 HTTP 内容
- 可以根据 URL、Header 路由
- 支持会话保持
- 性能稍低
```

```nginx
# L7 负载均衡示例（基于 URL）
upstream api_servers {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

upstream web_servers {
    server 127.0.0.1:4001;
    server 127.0.0.1:4002;
}

server {
    location /api/ {
        proxy_pass http://api_servers;
    }

    location / {
        proxy_pass http://web_servers;
    }
}

# L4 负载均衡（Stream 模块）
stream {
    upstream backend {
        server 127.0.0.1:3001;
        server 127.0.0.1:3002;
    }

    server {
        listen 3000;
        proxy_pass backend;
    }
}
```

### AWS 负载均衡器

```typescript
// AWS 负载均衡器类型
const awsLoadBalancers = {
  ALB: {
    name: 'Application Load Balancer',
    layer: 'L7',
    features: [
      '基于路径/主机路由',
      'WebSocket 支持',
      'HTTP/2 支持',
      '容器和微服务友好',
      '目标组健康检查'
    ],
    useCase: 'HTTP/HTTPS 应用'
  },

  NLB: {
    name: 'Network Load Balancer',
    layer: 'L4',
    features: [
      '超低延迟',
      '百万级并发',
      '静态 IP',
      'TCP/UDP 支持',
      'TLS 终止'
    ],
    useCase: '高性能、低延迟应用'
  },

  CLB: {
    name: 'Classic Load Balancer',
    layer: 'L4/L7',
    features: [
      'EC2-Classic 支持',
      '基本负载均衡'
    ],
    useCase: '遗留应用（不推荐新项目）'
  },

  GWLB: {
    name: 'Gateway Load Balancer',
    layer: 'L3',
    features: [
      '透明网络网关',
      '第三方虚拟设备'
    ],
    useCase: '防火墙、入侵检测'
  }
};
```

---

## CDN

### CDN 工作原理

```
1. 用户请求 www.example.com/image.jpg

2. DNS 解析
   - CNAME: www.example.com → cdn.example.com
   - CDN 智能 DNS 返回最近的边缘节点 IP

3. 用户访问边缘节点
   - 命中缓存：直接返回
   - 未命中：回源获取

4. 回源
   - 边缘节点 → 区域节点 → 中心节点 → 源站
   - 获取后缓存在各级节点

架构：
用户 → 边缘节点 → 区域节点 → 中心节点 → 源站
      (城市)     (省份)     (全国)    (原始服务器)
```

### CDN 缓存策略

```nginx
# 源站缓存头设置
location /static/ {
    # 长期缓存（版本化资源）
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
}

location /api/ {
    # 不缓存 API
    add_header Cache-Control "no-store, no-cache, must-revalidate";
}

location / {
    # 短期缓存（HTML）
    expires 5m;
    add_header Cache-Control "public, max-age=300";
}
```

```typescript
// CDN 缓存键设计
const cacheKeyFactors = {
  url: '请求 URL',
  queryString: '查询参数（可忽略或包含特定参数）',
  headers: ['Accept-Encoding', 'Accept-Language'],
  cookies: '特定 Cookie（谨慎使用）',
  device: '设备类型（PC/Mobile）'
};

// 缓存规则示例
const cacheRules = [
  { pattern: '*.jpg', ttl: '30d', cacheKey: 'url' },
  { pattern: '*.js', ttl: '1y', cacheKey: 'url' },
  { pattern: '/api/*', ttl: '0', bypass: true },
  { pattern: '*.html', ttl: '5m', cacheKey: 'url' }
];
```

### CDN 刷新和预热

```typescript
// CloudFront 缓存失效
import { CloudFrontClient, CreateInvalidationCommand } from '@aws-sdk/client-cloudfront';

const cloudfront = new CloudFrontClient({ region: 'us-east-1' });

async function invalidateCache(distributionId: string, paths: string[]) {
  await cloudfront.send(new CreateInvalidationCommand({
    DistributionId: distributionId,
    InvalidationBatch: {
      CallerReference: Date.now().toString(),
      Paths: {
        Quantity: paths.length,
        Items: paths // ['/images/*', '/api/users']
      }
    }
  }));
}

// 预热（主动推送到边缘节点）
// 大多数 CDN 提供预热 API
async function warmCache(urls: string[]) {
  // 并发请求各个边缘节点
  const regions = ['us-east-1', 'eu-west-1', 'ap-northeast-1'];
  
  for (const url of urls) {
    await Promise.all(
      regions.map(region => 
        fetch(url, { headers: { 'X-CDN-Region': region } })
      )
    );
  }
}
```

---

## API Gateway

### 功能概述

```typescript
// API Gateway 核心功能
const apiGatewayFeatures = {
  routing: {
    description: '路由管理',
    features: ['路径路由', '版本路由', '灰度发布']
  },

  authentication: {
    description: '认证授权',
    features: ['API Key', 'JWT', 'OAuth2', 'IAM']
  },

  rateLimit: {
    description: '限流熔断',
    features: ['速率限制', '并发限制', '熔断器']
  },

  transformation: {
    description: '请求转换',
    features: ['请求/响应转换', '协议转换']
  },

  monitoring: {
    description: '监控日志',
    features: ['请求日志', '指标统计', '告警']
  },

  security: {
    description: '安全防护',
    features: ['WAF', 'DDoS 防护', 'IP 黑白名单']
  }
};
```

### Kong 示例

```yaml
# kong.yml
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service:3000
    routes:
      - name: user-route
        paths:
          - /api/users
        strip_path: false

  - name: order-service
    url: http://order-service:3000
    routes:
      - name: order-route
        paths:
          - /api/orders

plugins:
  # 全局限流
  - name: rate-limiting
    config:
      second: 100
      policy: local

  # JWT 认证
  - name: jwt
    service: user-service
    config:
      claims_to_verify:
        - exp

  # 日志
  - name: http-log
    config:
      http_endpoint: http://log-service:8080/logs
      method: POST
      content_type: application/json

  # 请求转换
  - name: request-transformer
    service: user-service
    config:
      add:
        headers:
          - X-Request-ID:$(uuid)
```

### AWS API Gateway

```typescript
// AWS API Gateway + Lambda
import { APIGatewayProxyHandler, APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

export const handler: APIGatewayProxyHandler = async (event: APIGatewayProxyEvent): Promise<APIGatewayProxyResult> => {
  const { httpMethod, path, queryStringParameters, body, headers } = event;

  // 认证信息
  const userId = event.requestContext.authorizer?.claims?.sub;

  try {
    // 路由处理
    if (httpMethod === 'GET' && path === '/users') {
      const users = await getUsers();
      return {
        statusCode: 200,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify(users)
      };
    }

    return {
      statusCode: 404,
      body: JSON.stringify({ error: 'Not Found' })
    };

  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal Server Error' })
    };
  }
};

// Serverless Framework 配置
// serverless.yml
/*
service: my-api

provider:
  name: aws
  runtime: nodejs18.x

functions:
  api:
    handler: handler.handler
    events:
      - http:
          path: /{proxy+}
          method: any
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref ApiGatewayAuthorizer
*/
```

---

## Node.js 代理实现

### HTTP 代理

```typescript
import http from 'http';
import https from 'https';
import { URL } from 'url';

// 简单 HTTP 代理
const proxyServer = http.createServer((clientReq, clientRes) => {
  const targetUrl = new URL(clientReq.url!);
  const client = targetUrl.protocol === 'https:' ? https : http;

  const proxyReq = client.request(
    {
      hostname: targetUrl.hostname,
      port: targetUrl.port,
      path: targetUrl.pathname + targetUrl.search,
      method: clientReq.method,
      headers: {
        ...clientReq.headers,
        host: targetUrl.host
      }
    },
    (proxyRes) => {
      clientRes.writeHead(proxyRes.statusCode!, proxyRes.headers);
      proxyRes.pipe(clientRes);
    }
  );

  proxyReq.on('error', (err) => {
    console.error('Proxy error:', err);
    clientRes.writeHead(502);
    clientRes.end('Bad Gateway');
  });

  clientReq.pipe(proxyReq);
});

proxyServer.listen(8080);
```

### 使用 http-proxy

```bash
npm install http-proxy
```

```typescript
import http from 'http';
import httpProxy from 'http-proxy';

const proxy = httpProxy.createProxyServer({});

// 负载均衡
const servers = [
  'http://127.0.0.1:3001',
  'http://127.0.0.1:3002',
  'http://127.0.0.1:3003'
];

let currentIndex = 0;

const server = http.createServer((req, res) => {
  // 轮询选择服务器
  const target = servers[currentIndex];
  currentIndex = (currentIndex + 1) % servers.length;

  proxy.web(req, res, { target }, (err) => {
    console.error('Proxy error:', err);
    res.writeHead(502);
    res.end('Bad Gateway');
  });
});

// WebSocket 代理
server.on('upgrade', (req, socket, head) => {
  const target = servers[currentIndex];
  currentIndex = (currentIndex + 1) % servers.length;

  proxy.ws(req, socket, head, { target });
});

// 事件监听
proxy.on('proxyReq', (proxyReq, req, res) => {
  proxyReq.setHeader('X-Forwarded-For', req.socket.remoteAddress);
  proxyReq.setHeader('X-Request-ID', crypto.randomUUID());
});

proxy.on('proxyRes', (proxyRes, req, res) => {
  console.log(`${req.method} ${req.url} -> ${proxyRes.statusCode}`);
});

server.listen(8080);
```

### Express 代理中间件

```typescript
import express from 'express';
import { createProxyMiddleware, Options } from 'http-proxy-middleware';

const app = express();

// API 代理
app.use('/api', createProxyMiddleware({
  target: 'http://api-server:3000',
  changeOrigin: true,
  pathRewrite: {
    '^/api': '' // 去掉 /api 前缀
  },
  onProxyReq(proxyReq, req, res) {
    proxyReq.setHeader('X-Forwarded-For', req.ip);
  },
  onProxyRes(proxyRes, req, res) {
    console.log(`Proxied: ${req.method} ${req.url} -> ${proxyRes.statusCode}`);
  },
  onError(err, req, res) {
    res.status(502).json({ error: 'Bad Gateway' });
  }
}));

// WebSocket 代理
app.use('/ws', createProxyMiddleware({
  target: 'ws://ws-server:3000',
  ws: true,
  changeOrigin: true
}));

// 静态文件代理
app.use('/static', createProxyMiddleware({
  target: 'http://cdn-origin:3000',
  changeOrigin: true,
  selfHandleResponse: true, // 自定义响应处理
  onProxyRes(proxyRes, req, res) {
    // 添加缓存头
    res.setHeader('Cache-Control', 'public, max-age=31536000');
    proxyRes.pipe(res);
  }
}));

app.listen(8080);
```

---

## 常见面试题

### 1. 正向代理和反向代理的区别？

| 特性 | 正向代理 | 反向代理 |
|------|---------|---------|
| **代理对象** | 客户端 | 服务器 |
| **客户端知道** | 知道代理存在 | 不知道真实服务器 |
| **服务器知道** | 不知道真实客户端 | 知道代理存在 |
| **用途** | 翻墙、匿名、缓存 | 负载均衡、安全、缓存 |

### 2. 常见的负载均衡算法？

1. **轮询（Round Robin）**：依次分配
2. **加权轮询**：按权重分配
3. **IP Hash**：同一 IP 访问同一服务器
4. **最少连接**：选择连接数最少的服务器
5. **一致性哈希**：减少服务器变化时的影响
6. **随机**：随机选择

### 3. CDN 的工作原理？

1. 用户请求资源
2. DNS 解析到 CDN
3. CDN 智能 DNS 返回最近边缘节点
4. 边缘节点：
   - 缓存命中：直接返回
   - 缓存未命中：回源获取并缓存
5. 返回资源给用户

### 4. L4 和 L7 负载均衡的区别？

| 特性 | L4 | L7 |
|------|-----|-----|
| **层级** | 传输层 | 应用层 |
| **基于** | IP + 端口 | HTTP 内容 |
| **性能** | 更高 | 稍低 |
| **功能** | 简单转发 | URL 路由、Header 处理 |
| **会话保持** | IP Hash | Cookie |

### 5. 如何实现会话保持？

1. **IP Hash**：同一 IP 访问同一服务器
2. **Cookie**：在 Cookie 中标记服务器
3. **Session 共享**：Redis 存储 Session
4. **JWT**：无状态 Token

### 6. 什么是服务发现？

服务发现用于在分布式系统中自动发现服务实例：

1. **客户端发现**：客户端查询注册中心，自己选择实例
2. **服务端发现**：通过负载均衡器转发，客户端不感知

常见工具：
- Consul
- Eureka
- etcd
- Kubernetes DNS

---

## 总结

### 技术选型

| 场景 | 推荐方案 |
|------|---------|
| **反向代理** | Nginx |
| **API Gateway** | Kong、AWS API Gateway |
| **L7 负载均衡** | Nginx、ALB |
| **L4 负载均衡** | HAProxy、NLB |
| **CDN** | CloudFront、Cloudflare |

### 实践检查清单

- [ ] 理解正向/反向代理
- [ ] 掌握 Nginx 配置
- [ ] 了解负载均衡算法
- [ ] 理解 L4 vs L7
- [ ] 了解 CDN 原理
- [ ] 掌握 API Gateway
- [ ] 会用 Node.js 实现代理
- [ ] 了解服务发现

---

**上一篇**：[网络基础](./04-network-fundamentals.md)  
**返回目录**：[网络与协议](./README.md)

