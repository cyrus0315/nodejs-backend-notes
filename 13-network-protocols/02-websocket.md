# WebSocket 实时通信

WebSocket 是实现实时双向通信的核心技术。本文深入讲解 WebSocket 协议原理和 Node.js 实践。

## 目录
- [WebSocket 基础](#websocket-基础)
- [ws 库](#ws-库)
- [Socket.io](#socketio)
- [心跳与重连](#心跳与重连)
- [消息协议设计](#消息协议设计)
- [生产环境实践](#生产环境实践)
- [面试题](#常见面试题)

---

## WebSocket 基础

### WebSocket vs HTTP

```
HTTP（短连接/轮询）：
客户端 → 服务器（请求）
客户端 ← 服务器（响应）
连接关闭

WebSocket（长连接）：
客户端 ←→ 服务器（双向通信）
连接保持
```

| 特性 | HTTP | WebSocket |
|------|------|-----------|
| **连接** | 短连接（请求-响应） | 长连接（保持） |
| **通信** | 单向（客户端发起） | 双向（任一方发起） |
| **头部** | 每次完整头部 | 只有握手时有头部 |
| **实时性** | 需要轮询 | 即时推送 |
| **开销** | 高（重复连接） | 低（保持连接） |

### WebSocket 握手

```http
# 客户端请求
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat

# 服务器响应
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

```typescript
// Sec-WebSocket-Accept 计算
import crypto from 'crypto';

const MAGIC_STRING = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11';

function generateAccept(key: string): string {
  return crypto
    .createHash('sha1')
    .update(key + MAGIC_STRING)
    .digest('base64');
}

// 验证
const clientKey = 'dGhlIHNhbXBsZSBub25jZQ==';
const accept = generateAccept(clientKey);
// s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### WebSocket 帧结构

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+

Opcode:
  0x0: 继续帧
  0x1: 文本帧
  0x2: 二进制帧
  0x8: 关闭连接
  0x9: Ping
  0xA: Pong
```

---

## ws 库

### 基础使用

```bash
npm install ws
npm install @types/ws -D
```

```typescript
// server.ts
import { WebSocketServer, WebSocket } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws: WebSocket, req) => {
  const ip = req.socket.remoteAddress;
  console.log(`Client connected from ${ip}`);

  // 接收消息
  ws.on('message', (data: Buffer, isBinary: boolean) => {
    const message = isBinary ? data : data.toString();
    console.log('Received:', message);

    // 回复消息
    ws.send(`Echo: ${message}`);

    // 广播给所有客户端
    wss.clients.forEach((client) => {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(message);
      }
    });
  });

  // 连接关闭
  ws.on('close', (code: number, reason: Buffer) => {
    console.log(`Client disconnected: ${code} - ${reason.toString()}`);
  });

  // 错误处理
  ws.on('error', (error) => {
    console.error('WebSocket error:', error);
  });

  // 发送欢迎消息
  ws.send('Welcome to the server!');
});

console.log('WebSocket server running on port 8080');
```

```typescript
// client.ts
import WebSocket from 'ws';

const ws = new WebSocket('ws://localhost:8080');

ws.on('open', () => {
  console.log('Connected to server');
  ws.send('Hello, server!');
});

ws.on('message', (data) => {
  console.log('Received:', data.toString());
});

ws.on('close', () => {
  console.log('Disconnected from server');
});

ws.on('error', (error) => {
  console.error('Error:', error);
});
```

### 与 Express 集成

```typescript
import express from 'express';
import http from 'http';
import { WebSocketServer, WebSocket } from 'ws';

const app = express();
const server = http.createServer(app);
const wss = new WebSocketServer({ server });

// Express 路由
app.get('/health', (req, res) => {
  res.json({ status: 'ok', connections: wss.clients.size });
});

// WebSocket 处理
wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    console.log('Received:', message.toString());
  });
});

server.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### 用户管理

```typescript
interface Client {
  id: string;
  ws: WebSocket;
  userId?: string;
  rooms: Set<string>;
}

class WebSocketManager {
  private clients: Map<string, Client> = new Map();
  private userSockets: Map<string, Set<string>> = new Map(); // userId -> clientIds
  private rooms: Map<string, Set<string>> = new Map(); // roomId -> clientIds

  addClient(ws: WebSocket): string {
    const id = crypto.randomUUID();
    const client: Client = {
      id,
      ws,
      rooms: new Set()
    };
    this.clients.set(id, client);
    return id;
  }

  removeClient(clientId: string): void {
    const client = this.clients.get(clientId);
    if (!client) return;

    // 从所有房间移除
    client.rooms.forEach(roomId => {
      this.leaveRoom(clientId, roomId);
    });

    // 从用户映射移除
    if (client.userId) {
      const userClients = this.userSockets.get(client.userId);
      userClients?.delete(clientId);
      if (userClients?.size === 0) {
        this.userSockets.delete(client.userId);
      }
    }

    this.clients.delete(clientId);
  }

  authenticateClient(clientId: string, userId: string): void {
    const client = this.clients.get(clientId);
    if (!client) return;

    client.userId = userId;

    if (!this.userSockets.has(userId)) {
      this.userSockets.set(userId, new Set());
    }
    this.userSockets.get(userId)!.add(clientId);
  }

  joinRoom(clientId: string, roomId: string): void {
    const client = this.clients.get(clientId);
    if (!client) return;

    client.rooms.add(roomId);

    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, new Set());
    }
    this.rooms.get(roomId)!.add(clientId);
  }

  leaveRoom(clientId: string, roomId: string): void {
    const client = this.clients.get(clientId);
    if (!client) return;

    client.rooms.delete(roomId);

    const room = this.rooms.get(roomId);
    room?.delete(clientId);
    if (room?.size === 0) {
      this.rooms.delete(roomId);
    }
  }

  // 发送给特定客户端
  sendToClient(clientId: string, message: any): void {
    const client = this.clients.get(clientId);
    if (client?.ws.readyState === WebSocket.OPEN) {
      client.ws.send(JSON.stringify(message));
    }
  }

  // 发送给特定用户（可能有多个客户端）
  sendToUser(userId: string, message: any): void {
    const clientIds = this.userSockets.get(userId);
    clientIds?.forEach(clientId => {
      this.sendToClient(clientId, message);
    });
  }

  // 广播到房间
  broadcastToRoom(roomId: string, message: any, excludeClientId?: string): void {
    const room = this.rooms.get(roomId);
    room?.forEach(clientId => {
      if (clientId !== excludeClientId) {
        this.sendToClient(clientId, message);
      }
    });
  }

  // 广播给所有人
  broadcast(message: any, excludeClientId?: string): void {
    this.clients.forEach((client, clientId) => {
      if (clientId !== excludeClientId && client.ws.readyState === WebSocket.OPEN) {
        client.ws.send(JSON.stringify(message));
      }
    });
  }
}

// 使用
const wsManager = new WebSocketManager();

wss.on('connection', (ws) => {
  const clientId = wsManager.addClient(ws);

  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());

    switch (message.type) {
      case 'auth':
        wsManager.authenticateClient(clientId, message.userId);
        break;

      case 'join':
        wsManager.joinRoom(clientId, message.roomId);
        wsManager.broadcastToRoom(message.roomId, {
          type: 'user_joined',
          userId: message.userId
        }, clientId);
        break;

      case 'message':
        wsManager.broadcastToRoom(message.roomId, {
          type: 'message',
          content: message.content,
          userId: message.userId
        });
        break;
    }
  });

  ws.on('close', () => {
    wsManager.removeClient(clientId);
  });
});
```

---

## Socket.io

### 基础使用

```bash
npm install socket.io
npm install socket.io-client  # 客户端
```

```typescript
// server.ts
import { Server } from 'socket.io';
import http from 'http';
import express from 'express';

const app = express();
const server = http.createServer(app);

const io = new Server(server, {
  cors: {
    origin: ['http://localhost:3000'],
    methods: ['GET', 'POST']
  },
  pingTimeout: 60000,
  pingInterval: 25000
});

// 中间件：认证
io.use((socket, next) => {
  const token = socket.handshake.auth.token;
  
  try {
    const user = verifyToken(token);
    socket.data.user = user;
    next();
  } catch (err) {
    next(new Error('Authentication error'));
  }
});

// 连接处理
io.on('connection', (socket) => {
  console.log(`User connected: ${socket.data.user.id}`);

  // 加入用户专属房间
  socket.join(`user:${socket.data.user.id}`);

  // 监听事件
  socket.on('message', (data, callback) => {
    console.log('Received message:', data);

    // 广播给房间
    socket.to(data.roomId).emit('message', {
      content: data.content,
      userId: socket.data.user.id,
      timestamp: new Date()
    });

    // 确认收到
    callback({ success: true });
  });

  // 加入房间
  socket.on('join_room', async (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user_joined', {
      userId: socket.data.user.id
    });
  });

  // 离开房间
  socket.on('leave_room', (roomId) => {
    socket.leave(roomId);
    socket.to(roomId).emit('user_left', {
      userId: socket.data.user.id
    });
  });

  // 断开连接
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${socket.data.user.id}, reason: ${reason}`);
  });
});

// 从外部发送消息（如 API 调用）
app.post('/api/notify', (req, res) => {
  const { userId, message } = req.body;
  
  io.to(`user:${userId}`).emit('notification', message);
  
  res.json({ success: true });
});

server.listen(3000);
```

```typescript
// client.ts
import { io, Socket } from 'socket.io-client';

const socket: Socket = io('http://localhost:3000', {
  auth: {
    token: 'your-jwt-token'
  },
  autoConnect: true,
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
  timeout: 20000
});

// 连接成功
socket.on('connect', () => {
  console.log('Connected:', socket.id);
});

// 连接失败
socket.on('connect_error', (error) => {
  console.error('Connection error:', error.message);
});

// 接收消息
socket.on('message', (data) => {
  console.log('Received message:', data);
});

// 发送消息（带回调）
socket.emit('message', {
  roomId: 'room-1',
  content: 'Hello!'
}, (response) => {
  if (response.success) {
    console.log('Message sent successfully');
  }
});

// 加入房间
socket.emit('join_room', 'room-1');
```

### 命名空间（Namespace）

```typescript
// 创建命名空间
const adminNamespace = io.of('/admin');
const chatNamespace = io.of('/chat');

// 管理员命名空间
adminNamespace.use((socket, next) => {
  // 验证管理员权限
  if (socket.data.user?.role === 'admin') {
    next();
  } else {
    next(new Error('Admin access required'));
  }
});

adminNamespace.on('connection', (socket) => {
  console.log('Admin connected');

  socket.on('broadcast', (message) => {
    // 广播给所有用户
    chatNamespace.emit('announcement', message);
  });
});

// 聊天命名空间
chatNamespace.on('connection', (socket) => {
  console.log('User connected to chat');
});
```

### 房间管理

```typescript
// 加入房间
socket.join('room1');
socket.join(['room1', 'room2', 'room3']);

// 离开房间
socket.leave('room1');

// 发送到房间
io.to('room1').emit('event', data);              // 包括发送者
socket.to('room1').emit('event', data);          // 不包括发送者
io.to('room1').to('room2').emit('event', data);  // 多个房间

// 获取房间信息
const sockets = await io.in('room1').fetchSockets();
console.log(`Room has ${sockets.length} members`);

// 获取用户所在的房间
console.log(socket.rooms); // Set { 'socket-id', 'room1', 'room2' }
```

### 适配器（多服务器）

```typescript
// Redis 适配器
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: 'redis://localhost:6379' });
const subClient = pubClient.duplicate();

await Promise.all([pubClient.connect(), subClient.connect()]);

io.adapter(createAdapter(pubClient, subClient));

// 现在可以在多个服务器实例之间共享消息
```

---

## 心跳与重连

### 心跳机制

```typescript
// ws 库心跳实现
class HeartbeatWebSocket {
  private ws: WebSocket | null = null;
  private heartbeatInterval: NodeJS.Timer | null = null;
  private missedPongs = 0;
  private readonly maxMissedPongs = 3;
  private readonly heartbeatIntervalMs = 30000;

  connect(url: string): void {
    this.ws = new WebSocket(url);

    this.ws.on('open', () => {
      console.log('Connected');
      this.startHeartbeat();
    });

    this.ws.on('pong', () => {
      this.missedPongs = 0;
    });

    this.ws.on('close', () => {
      console.log('Disconnected');
      this.stopHeartbeat();
    });

    this.ws.on('error', (error) => {
      console.error('Error:', error);
    });
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = setInterval(() => {
      if (this.missedPongs >= this.maxMissedPongs) {
        console.log('Connection dead, closing...');
        this.ws?.terminate();
        return;
      }

      this.missedPongs++;
      this.ws?.ping();
    }, this.heartbeatIntervalMs);
  }

  private stopHeartbeat(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }
}

// 服务端心跳
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws) => {
  (ws as any).isAlive = true;

  ws.on('pong', () => {
    (ws as any).isAlive = true;
  });
});

// 定期检查
const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if ((ws as any).isAlive === false) {
      return ws.terminate();
    }

    (ws as any).isAlive = false;
    ws.ping();
  });
}, 30000);

wss.on('close', () => {
  clearInterval(interval);
});
```

### 断线重连

```typescript
class ReconnectingWebSocket {
  private ws: WebSocket | null = null;
  private url: string;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnectInterval = 1000;
  private maxReconnectInterval = 30000;
  private isManualClose = false;

  constructor(url: string) {
    this.url = url;
  }

  connect(): void {
    this.isManualClose = false;
    this.ws = new WebSocket(this.url);

    this.ws.on('open', () => {
      console.log('Connected');
      this.reconnectAttempts = 0;
    });

    this.ws.on('close', () => {
      if (!this.isManualClose) {
        this.reconnect();
      }
    });

    this.ws.on('error', (error) => {
      console.error('Error:', error);
    });
  }

  private reconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.log('Max reconnect attempts reached');
      return;
    }

    this.reconnectAttempts++;

    // 指数退避
    const delay = Math.min(
      this.reconnectInterval * Math.pow(2, this.reconnectAttempts - 1),
      this.maxReconnectInterval
    );

    console.log(`Reconnecting in ${delay}ms (attempt ${this.reconnectAttempts})`);

    setTimeout(() => {
      this.connect();
    }, delay);
  }

  send(data: string): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(data);
    } else {
      console.warn('WebSocket not connected');
    }
  }

  close(): void {
    this.isManualClose = true;
    this.ws?.close();
  }
}
```

### Socket.io 自动重连

```typescript
// Socket.io 内置重连机制
const socket = io('http://localhost:3000', {
  reconnection: true,
  reconnectionAttempts: 10,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  randomizationFactor: 0.5
});

socket.on('reconnect', (attemptNumber) => {
  console.log(`Reconnected after ${attemptNumber} attempts`);
});

socket.on('reconnect_attempt', (attemptNumber) => {
  console.log(`Reconnect attempt ${attemptNumber}`);
});

socket.on('reconnect_error', (error) => {
  console.error('Reconnect error:', error);
});

socket.on('reconnect_failed', () => {
  console.log('Reconnect failed');
});
```

---

## 消息协议设计

### JSON 协议

```typescript
// 消息类型定义
interface Message {
  type: string;
  payload: any;
  timestamp: number;
  id: string;
}

interface ChatMessage extends Message {
  type: 'chat';
  payload: {
    roomId: string;
    content: string;
    userId: string;
  };
}

interface TypingMessage extends Message {
  type: 'typing';
  payload: {
    roomId: string;
    userId: string;
    isTyping: boolean;
  };
}

interface PresenceMessage extends Message {
  type: 'presence';
  payload: {
    userId: string;
    status: 'online' | 'offline' | 'away';
  };
}

// 消息处理器
type MessageHandler<T extends Message> = (message: T) => void;

class MessageRouter {
  private handlers: Map<string, MessageHandler<any>[]> = new Map();

  on<T extends Message>(type: string, handler: MessageHandler<T>): void {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, []);
    }
    this.handlers.get(type)!.push(handler);
  }

  handle(raw: string): void {
    try {
      const message = JSON.parse(raw) as Message;
      const handlers = this.handlers.get(message.type) || [];
      handlers.forEach(handler => handler(message));
    } catch (error) {
      console.error('Failed to parse message:', error);
    }
  }
}

// 使用
const router = new MessageRouter();

router.on<ChatMessage>('chat', (message) => {
  console.log(`Chat from ${message.payload.userId}: ${message.payload.content}`);
});

router.on<TypingMessage>('typing', (message) => {
  console.log(`${message.payload.userId} is ${message.payload.isTyping ? 'typing' : 'not typing'}`);
});

ws.on('message', (data) => {
  router.handle(data.toString());
});
```

### Binary 协议（高性能）

```typescript
// 使用 Protocol Buffers 或 MessagePack

// MessagePack 示例
import { encode, decode } from '@msgpack/msgpack';

// 编码（比 JSON 小 30-50%）
const message = {
  type: 'chat',
  payload: {
    roomId: 'room1',
    content: 'Hello!'
  }
};

const encoded = encode(message);
ws.send(encoded);

// 解码
ws.on('message', (data: Buffer) => {
  const message = decode(data);
  console.log(message);
});
```

### 消息确认（ACK）

```typescript
interface AckableMessage {
  id: string;
  type: string;
  payload: any;
  requiresAck: boolean;
}

class AckManager {
  private pending: Map<string, {
    message: AckableMessage;
    resolve: () => void;
    reject: (error: Error) => void;
    timer: NodeJS.Timer;
  }> = new Map();

  private timeout = 5000;

  send(ws: WebSocket, message: AckableMessage): Promise<void> {
    return new Promise((resolve, reject) => {
      ws.send(JSON.stringify(message));

      if (message.requiresAck) {
        const timer = setTimeout(() => {
          this.pending.delete(message.id);
          reject(new Error('Ack timeout'));
        }, this.timeout);

        this.pending.set(message.id, {
          message,
          resolve,
          reject,
          timer
        });
      } else {
        resolve();
      }
    });
  }

  ack(messageId: string): void {
    const pending = this.pending.get(messageId);
    if (pending) {
      clearTimeout(pending.timer);
      pending.resolve();
      this.pending.delete(messageId);
    }
  }
}

// 使用
const ackManager = new AckManager();

// 发送需要确认的消息
await ackManager.send(ws, {
  id: crypto.randomUUID(),
  type: 'important',
  payload: { data: 'critical' },
  requiresAck: true
});

// 接收 ACK
ws.on('message', (data) => {
  const message = JSON.parse(data.toString());
  if (message.type === 'ack') {
    ackManager.ack(message.id);
  }
});
```

---

## 生产环境实践

### 鉴权与安全

```typescript
import { WebSocketServer, WebSocket } from 'ws';
import jwt from 'jsonwebtoken';

const wss = new WebSocketServer({ noServer: true });

// HTTP 服务器升级连接时验证
server.on('upgrade', (request, socket, head) => {
  // 从查询参数或 Cookie 获取 token
  const url = new URL(request.url!, `http://${request.headers.host}`);
  const token = url.searchParams.get('token');

  if (!token) {
    socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n');
    socket.destroy();
    return;
  }

  try {
    const user = jwt.verify(token, process.env.JWT_SECRET!);
    
    wss.handleUpgrade(request, socket, head, (ws) => {
      (ws as any).user = user;
      wss.emit('connection', ws, request);
    });
  } catch (error) {
    socket.write('HTTP/1.1 401 Unauthorized\r\n\r\n');
    socket.destroy();
  }
});

// 连接后的权限检查
wss.on('connection', (ws) => {
  ws.on('message', (data) => {
    const message = JSON.parse(data.toString());

    // 检查权限
    if (message.type === 'admin_action' && (ws as any).user.role !== 'admin') {
      ws.send(JSON.stringify({ error: 'Permission denied' }));
      return;
    }

    // 处理消息...
  });
});
```

### 限流与保护

```typescript
class RateLimiter {
  private counts: Map<string, { count: number; resetTime: number }> = new Map();
  private limit: number;
  private windowMs: number;

  constructor(limit: number = 100, windowMs: number = 60000) {
    this.limit = limit;
    this.windowMs = windowMs;
  }

  check(key: string): boolean {
    const now = Date.now();
    const record = this.counts.get(key);

    if (!record || now > record.resetTime) {
      this.counts.set(key, { count: 1, resetTime: now + this.windowMs });
      return true;
    }

    if (record.count >= this.limit) {
      return false;
    }

    record.count++;
    return true;
  }
}

const rateLimiter = new RateLimiter(100, 60000); // 每分钟 100 条消息

wss.on('connection', (ws) => {
  const userId = (ws as any).user.id;

  ws.on('message', (data) => {
    if (!rateLimiter.check(userId)) {
      ws.send(JSON.stringify({ error: 'Rate limit exceeded' }));
      return;
    }

    // 处理消息...
  });
});
```

### 负载均衡（Sticky Session）

```nginx
# Nginx 配置 WebSocket 负载均衡
upstream websocket {
    ip_hash;  # Sticky session
    server backend1:8080;
    server backend2:8080;
    server backend3:8080;
}

server {
    listen 80;
    server_name example.com;

    location /ws {
        proxy_pass http://websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### 监控与日志

```typescript
import { WebSocketServer, WebSocket } from 'ws';

// 统计
const stats = {
  connections: 0,
  messagesReceived: 0,
  messagesSent: 0,
  errors: 0
};

wss.on('connection', (ws) => {
  stats.connections++;
  console.log(`[+] Connection (total: ${stats.connections})`);

  ws.on('message', () => {
    stats.messagesReceived++;
  });

  const originalSend = ws.send.bind(ws);
  ws.send = (...args: any[]) => {
    stats.messagesSent++;
    return originalSend(...args);
  };

  ws.on('close', () => {
    stats.connections--;
    console.log(`[-] Disconnection (total: ${stats.connections})`);
  });

  ws.on('error', () => {
    stats.errors++;
  });
});

// 暴露统计端点
app.get('/ws/stats', (req, res) => {
  res.json(stats);
});

// Prometheus 指标
import client from 'prom-client';

const wsConnections = new client.Gauge({
  name: 'websocket_connections',
  help: 'Number of active WebSocket connections'
});

const wsMessages = new client.Counter({
  name: 'websocket_messages_total',
  help: 'Total WebSocket messages',
  labelNames: ['direction'] // 'sent' | 'received'
});
```

---

## 常见面试题

### 1. WebSocket 和 HTTP 的区别？

| 特性 | HTTP | WebSocket |
|------|------|-----------|
| **连接** | 短连接 | 长连接 |
| **通信** | 单向（请求-响应） | 双向 |
| **协议** | HTTP | ws:// 或 wss:// |
| **头部** | 每次完整 | 握手时一次 |
| **实时性** | 需轮询 | 即时推送 |

### 2. WebSocket 握手过程？

1. 客户端发送 HTTP 请求，包含 `Upgrade: websocket` 头
2. 服务器返回 `101 Switching Protocols`
3. 连接升级为 WebSocket
4. 双方可以双向通信

### 3. 如何处理 WebSocket 断线重连？

**策略**：
- 监听 `close` 事件
- 指数退避重连
- 最大重连次数限制
- 重连成功后恢复状态

### 4. WebSocket vs SSE（Server-Sent Events）？

| 特性 | WebSocket | SSE |
|------|-----------|-----|
| **通信** | 双向 | 单向（服务器 → 客户端） |
| **协议** | ws:// | HTTP |
| **浏览器支持** | 广泛 | 需要 polyfill |
| **自动重连** | 需实现 | 内置 |
| **适用场景** | 聊天、游戏 | 推送通知 |

### 5. 如何实现 WebSocket 集群？

**方案**：
- Sticky Session（同一用户连接同一服务器）
- Redis Pub/Sub（消息广播）
- Socket.io Redis Adapter
- 消息队列（Kafka、RabbitMQ）

---

## 总结

### 技术选型

| 场景 | 推荐方案 |
|------|---------|
| **简单双向通信** | ws 库 |
| **需要房间、命名空间** | Socket.io |
| **需要跨服务器** | Socket.io + Redis |
| **高性能** | ws + 自定义协议 |

### 实践检查清单

- [ ] 理解 WebSocket 握手
- [ ] 实现心跳机制
- [ ] 实现断线重连
- [ ] 设计消息协议
- [ ] 实现用户/房间管理
- [ ] 配置认证鉴权
- [ ] 实现限流保护
- [ ] 配置负载均衡
- [ ] 监控连接状态

---

**上一篇**：[HTTP/HTTPS](./01-http-https.md)  
**下一篇**：[TCP/UDP](./03-tcp-udp.md)

