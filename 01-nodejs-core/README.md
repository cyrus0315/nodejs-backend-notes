# Node.js 核心知识

本模块已创建详细的学习文档，涵盖 Node.js 核心知识的各个方面。

## 📚 学习文档

### ✅ 已完成

1. **[Event Loop & 异步编程](./01-event-loop.md)**
   - Event Loop 六个阶段详解
   - 微任务 vs 宏任务
   - process.nextTick vs setImmediate
   - Callback, Promise, Async/Await
   - 异步错误处理
   - 经典面试题及解答

2. **[Stream & Buffer](./02-stream-buffer.md)**
   - Buffer 创建和操作
   - 四种 Stream 类型
   - Pipe 和背压处理
   - 自定义 Stream
   - 实战示例和最佳实践

3. **[进程与线程](./03-process-threads.md)**
   - Node.js 单线程模型
   - Process 对象详解
   - Child Process（spawn, exec, fork）
   - Cluster 集群实现
   - Worker Threads 工作线程
   - 进程间通信（IPC）

4. **[模块系统](./04-modules.md)**
   - CommonJS vs ES Modules
   - Module Resolution 机制
   - 模块缓存和循环依赖
   - 内置核心模块
   - 创建和发布 npm 包

5. **[性能优化](./05-performance.md)**
   - 内存管理和垃圾回收
   - 内存泄漏检测与修复
   - CPU 优化技巧
   - 数据库性能优化
   - HTTP 性能优化
   - 性能监控工具

6. **[错误处理](./06-error-handling.md)**
   - 错误类型（操作错误 vs 程序错误）
   - 同步和异步错误处理
   - 全局错误处理
   - Express 错误中间件
   - 错误日志和监控
   - 优雅关闭

7. **[现代特性](./07-modern-features.md)** ⭐ 新增
   - Fetch API（Node.js 18+）
   - Test Runner（内置测试框架）
   - Watch Mode（自动重启）
   - Web Streams API
   - Web Crypto API
   - Single Executable Applications
   - 权限模型（Node.js 20+）
   - .env 支持、性能改进等

## 📝 知识点清单

### 基础知识
- [x] Node.js 运行原理
- [x] CommonJS vs ES Modules
- [x] npm/yarn/pnpm 包管理
- [x] package.json 详解
- [x] 全局对象与内置模块

### Event Loop & 异步编程
- [x] Event Loop 工作原理
- [x] Callback, Promise, Async/Await
- [x] 微任务 vs 宏任务
- [x] Process.nextTick vs setImmediate
- [x] 异步错误处理

### Stream & Buffer
- [x] Buffer 原理与使用
- [x] Stream 类型（Readable, Writable, Duplex, Transform）
- [x] 管道（Pipe）与链式操作
- [x] 背压（Backpressure）处理
- [x] Stream 实战场景

### 进程与线程
- [x] 单线程模型
- [x] Child Process 模块
- [x] Cluster 集群模式
- [x] Worker Threads
- [x] 进程间通信（IPC）

### 文件系统
- [x] fs 模块（同步 vs 异步）
- [x] 文件读写操作
- [x] 目录操作
- [x] 文件流操作
- [x] 路径处理（path 模块）

### 网络编程
- [x] HTTP 模块
- [x] HTTPS 模块
- [x] Net 模块（TCP）
- [x] DNS 模块
- [x] URL 处理

### 性能优化
- [x] 内存管理与垃圾回收
- [x] 内存泄漏检测
- [x] CPU Profiling
- [x] V8 引擎优化
- [x] 性能监控工具

### 错误处理
- [x] 错误类型与处理
- [x] Uncaught Exception
- [x] Unhandled Rejection
- [x] 错误堆栈追踪
- [x] 优雅退出

### 现代特性（Node.js 16+）
- [x] Fetch API（内置 HTTP 客户端）
- [x] Test Runner（内置测试框架）
- [x] Watch Mode（开发模式）
- [x] Web Streams API
- [x] Web Crypto API
- [x] .env 文件支持
- [x] 权限模型
- [x] JSON 模块导入
- [x] Single Executable Applications
- [x] 性能改进（V8、HTTP）

## 🎯 学习建议

1. **按顺序学习**：建议按照文档编号顺序学习，建立完整的知识体系
2. **动手实践**：每个文档都包含代码示例，务必自己运行和修改
3. **理解原理**：不仅要知道怎么用，更要理解为什么这样设计
4. **做练习题**：每个文档末尾都有练习题，帮助巩固知识
5. **模拟面试**：准备好常见面试题的答案，能够清晰表达

## 💡 重点难点

### 必须掌握
- ⭐⭐⭐ Event Loop 执行顺序
- ⭐⭐⭐ 异步编程（Promise/Async-Await）
- ⭐⭐⭐ Stream 和背压处理
- ⭐⭐⭐ 模块系统和循环依赖
- ⭐⭐⭐ 错误处理最佳实践
- ⭐⭐⭐ Node.js 现代特性（Fetch、Test Runner）

### 高级主题
- ⭐⭐ Worker Threads vs Cluster
- ⭐⭐ 内存泄漏检测
- ⭐⭐ 性能优化技巧
- ⭐⭐ 自定义 Stream
- ⭐⭐ Web Streams API
- ⭐⭐ 权限模型和安全

## 🔥 常见面试题

每个文档都包含了该主题的常见面试题和详细解答，建议重点复习：

1. Event Loop 执行顺序分析题
2. require 和 import 的区别
3. Stream vs Buffer 的使用场景
4. 如何充分利用多核 CPU
5. 内存泄漏的原因和解决方案
6. 错误处理最佳实践
7. Node.js 18/20 的重要新特性
8. fetch vs axios 如何选择
9. Web Streams vs Node Streams

## 📚 推荐资源

- [Node.js 官方文档](https://nodejs.org/docs/)
- [Node.js 设计模式](https://github.com/PacktPublishing/Node.js-Design-Patterns-Third-Edition)
- [深入浅出 Node.js](https://book.douban.com/subject/25768396/)
- [Node.js 调试指南](https://nodejs.org/en/docs/guides/debugging-getting-started/)

