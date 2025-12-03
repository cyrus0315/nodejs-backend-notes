# 网络基础

网络基础是理解 Web 开发的根基。本文深入讲解网络模型、DNS、IP 和完整的请求过程。

## 目录
- [网络模型](#网络模型)
- [DNS](#dns)
- [IP 协议](#ip-协议)
- [完整请求过程](#完整请求过程)
- [网络调试](#网络调试)
- [面试题](#常见面试题)

---

## 网络模型

### OSI 七层模型

```
┌─────────────────────────────────────────────────────────┐
│ 7. 应用层 (Application)                                  │
│    HTTP, FTP, SMTP, DNS, WebSocket                      │
├─────────────────────────────────────────────────────────┤
│ 6. 表示层 (Presentation)                                 │
│    SSL/TLS, 数据加密、压缩、格式转换                       │
├─────────────────────────────────────────────────────────┤
│ 5. 会话层 (Session)                                      │
│    建立、管理、终止会话                                    │
├─────────────────────────────────────────────────────────┤
│ 4. 传输层 (Transport)                                    │
│    TCP, UDP - 端到端传输                                 │
├─────────────────────────────────────────────────────────┤
│ 3. 网络层 (Network)                                      │
│    IP, ICMP, ARP - 路由选择                              │
├─────────────────────────────────────────────────────────┤
│ 2. 数据链路层 (Data Link)                                │
│    Ethernet, WiFi - 帧传输                               │
├─────────────────────────────────────────────────────────┤
│ 1. 物理层 (Physical)                                     │
│    网线、光纤 - 比特传输                                  │
└─────────────────────────────────────────────────────────┘
```

### TCP/IP 四层模型

```
┌─────────────────────────────────────────────────────────┐
│ 4. 应用层 (Application)                                  │
│    HTTP, FTP, SMTP, DNS                                 │
│    对应 OSI: 应用层 + 表示层 + 会话层                     │
├─────────────────────────────────────────────────────────┤
│ 3. 传输层 (Transport)                                    │
│    TCP, UDP                                             │
├─────────────────────────────────────────────────────────┤
│ 2. 网络层 (Internet)                                     │
│    IP, ICMP, ARP                                        │
├─────────────────────────────────────────────────────────┤
│ 1. 网络接口层 (Network Interface)                        │
│    Ethernet, WiFi                                       │
│    对应 OSI: 数据链路层 + 物理层                          │
└─────────────────────────────────────────────────────────┘
```

### 数据封装

```
应用层数据
    ↓ + HTTP 头
传输层数据（TCP/UDP 段）
    ↓ + TCP/UDP 头
网络层数据（IP 数据包）
    ↓ + IP 头
数据链路层数据（帧）
    ↓ + 帧头 + 帧尾
物理层（比特流）

发送方：自上而下封装
接收方：自下而上解封
```

### 各层协议

```typescript
// 应用层协议
const applicationProtocols = {
  HTTP: { port: 80, description: 'Web' },
  HTTPS: { port: 443, description: 'Secure Web' },
  FTP: { port: 21, description: 'File Transfer' },
  SSH: { port: 22, description: 'Secure Shell' },
  SMTP: { port: 25, description: 'Email Sending' },
  DNS: { port: 53, description: 'Domain Name Resolution' },
  DHCP: { port: 67, description: 'IP Address Assignment' },
  Redis: { port: 6379, description: 'Cache' },
  MySQL: { port: 3306, description: 'Database' },
  PostgreSQL: { port: 5432, description: 'Database' },
  MongoDB: { port: 27017, description: 'NoSQL Database' }
};

// 传输层协议
const transportProtocols = {
  TCP: 'Transmission Control Protocol - 可靠传输',
  UDP: 'User Datagram Protocol - 快速传输'
};

// 网络层协议
const networkProtocols = {
  IP: 'Internet Protocol - 路由',
  ICMP: 'Internet Control Message Protocol - ping',
  ARP: 'Address Resolution Protocol - IP→MAC',
  RARP: 'Reverse ARP - MAC→IP'
};
```

---

## DNS

### DNS 解析过程

```
浏览器输入 www.example.com

1. 浏览器缓存
   - 检查浏览器 DNS 缓存
   - Chrome: chrome://net-internals/#dns

2. 操作系统缓存
   - 检查 hosts 文件
   - 检查系统 DNS 缓存

3. 本地 DNS 服务器（递归查询）
   - 通常是 ISP 提供的 DNS 服务器
   - 或配置的公共 DNS（如 8.8.8.8）

4. 根域名服务器
   - 返回 .com 顶级域名服务器地址

5. 顶级域名服务器（TLD）
   - 返回 example.com 权威域名服务器地址

6. 权威域名服务器
   - 返回 www.example.com 的 IP 地址

7. 本地 DNS 服务器缓存结果
   - 根据 TTL 缓存

8. 返回 IP 地址给浏览器
```

```
查询过程图示：

浏览器 → 本地 DNS
           ↓ (递归查询)
         根 DNS → 返回 .com DNS 地址
           ↓ (迭代查询)
        .com DNS → 返回 example.com DNS 地址
           ↓ (迭代查询)
     example.com DNS → 返回 IP 地址
           ↓
         本地 DNS (缓存)
           ↓
         浏览器 (获得 IP)
```

### DNS 记录类型

```typescript
// DNS 记录类型
const dnsRecordTypes = {
  A: {
    description: 'IPv4 地址',
    example: 'example.com → 93.184.216.34'
  },
  AAAA: {
    description: 'IPv6 地址',
    example: 'example.com → 2606:2800:220:1:248:1893:25c8:1946'
  },
  CNAME: {
    description: '别名记录',
    example: 'www.example.com → example.com'
  },
  MX: {
    description: '邮件服务器',
    example: 'example.com → mail.example.com (priority 10)'
  },
  TXT: {
    description: '文本记录（SPF、DKIM 等）',
    example: 'example.com → "v=spf1 include:_spf.google.com ~all"'
  },
  NS: {
    description: '域名服务器',
    example: 'example.com → ns1.example.com'
  },
  SOA: {
    description: '权威记录',
    example: '域名区域的主要信息'
  },
  PTR: {
    description: '反向解析（IP→域名）',
    example: '34.216.184.93 → example.com'
  },
  SRV: {
    description: '服务记录',
    example: '_http._tcp.example.com → target port priority weight'
  }
};
```

### Node.js DNS

```typescript
import dns from 'dns';
import { promisify } from 'util';

// 回调方式
dns.lookup('google.com', (err, address, family) => {
  console.log('IP:', address, 'Family:', family);
});

// Promise 方式
const lookup = promisify(dns.lookup);
const resolve = promisify(dns.resolve);
const resolve4 = promisify(dns.resolve4);
const resolveMx = promisify(dns.resolveMx);

async function dnsQuery() {
  try {
    // A 记录
    const ipv4 = await resolve4('google.com');
    console.log('IPv4:', ipv4);

    // MX 记录
    const mx = await resolveMx('google.com');
    console.log('MX:', mx);

    // 所有记录
    const all = await dns.promises.resolveAny('google.com');
    console.log('All:', all);

    // 反向解析
    const hostnames = await dns.promises.reverse('8.8.8.8');
    console.log('Reverse:', hostnames);

  } catch (error) {
    console.error('DNS error:', error);
  }
}

dnsQuery();
```

### DNS 缓存

```typescript
// Node.js DNS 缓存（使用 lru-cache）
import dns from 'dns';
import { LRUCache } from 'lru-cache';

const dnsCache = new LRUCache<string, string[]>({
  max: 1000,
  ttl: 1000 * 60 * 5 // 5 分钟
});

async function cachedDnsLookup(hostname: string): Promise<string[]> {
  const cached = dnsCache.get(hostname);
  if (cached) {
    return cached;
  }

  return new Promise((resolve, reject) => {
    dns.resolve4(hostname, (err, addresses) => {
      if (err) {
        reject(err);
        return;
      }
      dnsCache.set(hostname, addresses);
      resolve(addresses);
    });
  });
}

// 使用自定义 DNS 服务器
dns.setServers([
  '8.8.8.8',      // Google DNS
  '8.8.4.4',
  '1.1.1.1',      // Cloudflare DNS
  '1.0.0.1'
]);

console.log('DNS Servers:', dns.getServers());
```

---

## IP 协议

### IPv4 vs IPv6

```typescript
// IPv4
// 32 位，约 43 亿地址
// 格式：192.168.1.1

// IPv6
// 128 位，地址空间巨大
// 格式：2001:0db8:85a3:0000:0000:8a2e:0370:7334
// 简写：2001:db8:85a3::8a2e:370:7334

// 特殊地址
const specialAddresses = {
  // IPv4
  localhost: '127.0.0.1',
  any: '0.0.0.0',
  broadcast: '255.255.255.255',
  
  // 私有地址段
  classA: '10.0.0.0/8',      // 10.0.0.0 - 10.255.255.255
  classB: '172.16.0.0/12',   // 172.16.0.0 - 172.31.255.255
  classC: '192.168.0.0/16',  // 192.168.0.0 - 192.168.255.255
  
  // IPv6
  localhost6: '::1',
  any6: '::',
};
```

### 子网掩码

```
IP 地址：192.168.1.100
子网掩码：255.255.255.0（或 /24）

网络地址 = IP & 子网掩码
192.168.1.100 AND 255.255.255.0 = 192.168.1.0

广播地址 = 网络地址 | ~子网掩码
192.168.1.0 OR 0.0.0.255 = 192.168.1.255

可用主机：192.168.1.1 - 192.168.1.254（254 个）

常见子网掩码：
/8  = 255.0.0.0       (16,777,214 hosts)
/16 = 255.255.0.0     (65,534 hosts)
/24 = 255.255.255.0   (254 hosts)
/32 = 255.255.255.255 (1 host)
```

### NAT（网络地址转换）

```
NAT 解决了 IPv4 地址不足的问题

内网设备           路由器（NAT）        外网服务器
192.168.1.100:5000  →  公网IP:60000  →  Server
192.168.1.101:5001  →  公网IP:60001  →  Server
192.168.1.102:5002  →  公网IP:60002  →  Server

NAT 类型：
1. 完全圆锥型（Full Cone）
2. 受限圆锥型（Restricted Cone）
3. 端口受限圆锥型（Port Restricted Cone）
4. 对称型（Symmetric）- 最严格
```

---

## 完整请求过程

### 浏览器输入 URL 到页面展示

```
1. URL 解析
   - 解析协议（http/https）
   - 解析域名（www.example.com）
   - 解析端口（默认 80/443）
   - 解析路径（/path/to/resource）

2. DNS 解析
   - 浏览器缓存 → 系统缓存 → hosts 文件
   - 本地 DNS → 根 DNS → 顶级 DNS → 权威 DNS
   - 获得 IP 地址

3. TCP 连接
   - 三次握手建立连接
   - 如果是 HTTPS，还需要 TLS 握手

4. 发送 HTTP 请求
   - 构建请求行（GET /path HTTP/1.1）
   - 添加请求头（Host, Cookie, User-Agent...）
   - 发送请求体（POST 数据）

5. 服务器处理
   - 负载均衡分发
   - Web 服务器接收请求
   - 后端应用处理业务逻辑
   - 数据库查询
   - 构建响应

6. 返回响应
   - 状态行（HTTP/1.1 200 OK）
   - 响应头（Content-Type, Set-Cookie...）
   - 响应体（HTML/JSON）

7. 浏览器渲染
   - 解析 HTML，构建 DOM 树
   - 解析 CSS，构建 CSSOM 树
   - 合并为渲染树
   - 布局（Layout）
   - 绘制（Paint）
   - 合成（Composite）

8. 连接关闭
   - HTTP/1.0：立即关闭
   - HTTP/1.1：Keep-Alive 复用
   - HTTP/2：多路复用
```

### 时间线分析

```typescript
// 请求时间分解
interface RequestTiming {
  // DNS 查询
  dnsLookup: number;
  
  // TCP 连接
  tcpConnection: number;
  
  // TLS 握手
  tlsHandshake: number;
  
  // 请求发送
  requestSent: number;
  
  // 等待响应（TTFB）
  waitingTTFB: number;
  
  // 内容下载
  contentDownload: number;
  
  // 总时间
  total: number;
}

// 优化方向
const optimizations = {
  dns: ['DNS 预解析', '减少域名数量', 'DNS 缓存'],
  tcp: ['TCP 连接复用', 'HTTP/2 多路复用', '减少重定向'],
  tls: ['TLS 1.3', '会话复用', 'OCSP Stapling'],
  ttfb: ['CDN', '服务端缓存', '数据库优化'],
  content: ['Gzip/Brotli 压缩', '资源压缩', '按需加载']
};
```

---

## 网络调试

### curl 命令

```bash
# 基本请求
curl https://api.example.com/users

# 显示响应头
curl -i https://api.example.com/users

# 只显示响应头
curl -I https://api.example.com/users

# POST 请求
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name": "John", "email": "john@example.com"}'

# 带认证
curl -H "Authorization: Bearer xxx" https://api.example.com/users

# 显示详细信息（包括 TLS 握手）
curl -v https://api.example.com/users

# 显示时间统计
curl -w "\
  DNS: %{time_namelookup}s\n\
  TCP: %{time_connect}s\n\
  TLS: %{time_appconnect}s\n\
  TTFB: %{time_starttransfer}s\n\
  Total: %{time_total}s\n" \
  -o /dev/null -s https://api.example.com

# 下载文件
curl -O https://example.com/file.zip

# 跟随重定向
curl -L https://example.com/redirect

# 使用代理
curl -x http://proxy:8080 https://api.example.com

# 忽略证书验证
curl -k https://self-signed.example.com
```

### tcpdump

```bash
# 捕获所有流量
sudo tcpdump -i any

# 捕获特定端口
sudo tcpdump -i any port 80

# 捕获特定主机
sudo tcpdump -i any host 192.168.1.1

# 捕获 HTTP 流量
sudo tcpdump -i any port 80 -A

# 保存到文件
sudo tcpdump -i any -w capture.pcap

# 读取文件
tcpdump -r capture.pcap

# 常用过滤
sudo tcpdump -i any 'tcp port 443 and host example.com'
```

### netstat / ss

```bash
# 查看监听端口
netstat -tlnp
ss -tlnp

# 查看所有连接
netstat -anp
ss -anp

# 查看特定端口
netstat -anp | grep :3000
ss -anp | grep :3000

# 统计连接状态
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

### dig

```bash
# DNS 查询
dig example.com

# 查询特定记录类型
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com TXT
dig example.com NS

# 使用特定 DNS 服务器
dig @8.8.8.8 example.com

# 追踪解析过程
dig +trace example.com

# 简洁输出
dig +short example.com

# 反向解析
dig -x 8.8.8.8
```

### Node.js 网络调试

```typescript
import http from 'http';
import https from 'https';

// 请求计时
async function timedFetch(url: string): Promise<void> {
  const startTime = Date.now();
  const marks: Record<string, number> = {};

  const client = url.startsWith('https') ? https : http;

  return new Promise((resolve, reject) => {
    const req = client.request(url, (res) => {
      marks['ttfb'] = Date.now() - startTime;

      let data = '';
      res.on('data', (chunk) => {
        data += chunk;
      });

      res.on('end', () => {
        marks['total'] = Date.now() - startTime;
        console.log('Timing:', marks);
        console.log('Status:', res.statusCode);
        console.log('Headers:', res.headers);
        resolve();
      });
    });

    req.on('socket', (socket) => {
      socket.on('lookup', () => {
        marks['dns'] = Date.now() - startTime;
      });
      socket.on('connect', () => {
        marks['tcp'] = Date.now() - startTime;
      });
      socket.on('secureConnect', () => {
        marks['tls'] = Date.now() - startTime;
      });
    });

    req.on('error', reject);
    req.end();
  });
}

await timedFetch('https://api.example.com');
```

---

## 常见面试题

### 1. OSI 七层模型和 TCP/IP 四层模型？

| OSI 七层 | TCP/IP 四层 | 协议示例 |
|---------|------------|---------|
| 应用层 | 应用层 | HTTP, FTP, DNS |
| 表示层 | ↑ | SSL/TLS |
| 会话层 | ↑ | - |
| 传输层 | 传输层 | TCP, UDP |
| 网络层 | 网络层 | IP, ICMP |
| 数据链路层 | 网络接口层 | Ethernet |
| 物理层 | ↑ | - |

### 2. DNS 解析过程？

1. 浏览器缓存
2. 操作系统缓存
3. hosts 文件
4. 本地 DNS 服务器（递归查询）
5. 根 DNS → 顶级 DNS → 权威 DNS（迭代查询）
6. 返回 IP 并缓存

### 3. 浏览器输入 URL 后发生了什么？

1. **URL 解析**
2. **DNS 解析** → 获取 IP
3. **TCP 连接** → 三次握手
4. **TLS 握手**（HTTPS）
5. **发送 HTTP 请求**
6. **服务器处理**
7. **返回响应**
8. **浏览器渲染**
9. **连接关闭/复用**

### 4. 什么是 CDN？

**CDN（Content Delivery Network）内容分发网络**

工作原理：
1. 用户请求资源
2. DNS 解析到最近的 CDN 节点
3. CDN 节点返回缓存内容
4. 若无缓存，回源获取并缓存

优势：
- 加速访问（就近访问）
- 减少源站压力
- 提高可用性
- DDoS 防护

### 5. 什么是 CORS？

**CORS（Cross-Origin Resource Sharing）跨域资源共享**

同源策略限制：
- 协议、域名、端口必须相同
- JavaScript 不能跨域请求

CORS 解决方案：
```http
# 服务器响应头
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
```

预检请求（OPTIONS）：
- 非简单请求会先发送 OPTIONS
- 检查服务器是否允许实际请求

### 6. 什么是 DNS 劫持？如何防护？

**DNS 劫持**：攻击者篡改 DNS 解析结果，将用户导向恶意网站

**防护措施**：
1. 使用可信 DNS（8.8.8.8, 1.1.1.1）
2. DNSSEC（DNS 安全扩展）
3. DNS over HTTPS (DoH)
4. DNS over TLS (DoT)
5. HTTPS 证书验证

---

## 总结

### 网络知识图谱

```
网络基础
├── 模型
│   ├── OSI 七层
│   └── TCP/IP 四层
├── DNS
│   ├── 解析过程
│   ├── 记录类型
│   └── 缓存机制
├── IP
│   ├── IPv4 vs IPv6
│   ├── 子网掩码
│   └── NAT
└── 调试
    ├── curl
    ├── tcpdump
    ├── dig
    └── netstat/ss
```

### 实践检查清单

- [ ] 理解 OSI 和 TCP/IP 模型
- [ ] 掌握 DNS 解析过程
- [ ] 了解 IP 地址和子网
- [ ] 能描述完整请求过程
- [ ] 会使用 curl 调试
- [ ] 会使用 dig 查询 DNS
- [ ] 了解 CDN 工作原理
- [ ] 理解 CORS

---

**上一篇**：[TCP/UDP](./03-tcp-udp.md)  
**下一篇**：[代理与负载均衡](./05-proxy-load-balancing.md)

