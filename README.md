<div align="center">

# 🚀 Node.js 后端开发学习笔记

[![GitHub stars](https://img.shields.io/github/stars/cyrus0315/nodejs-backend-notes?style=social)](https://github.com/cyrus0315/nodejs-backend-notes/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/cyrus0315/nodejs-backend-notes?style=social)](https://github.com/cyrus0315/nodejs-backend-notes/network/members)
[![GitHub issues](https://img.shields.io/github/issues/cyrus0315/nodejs-backend-notes)](https://github.com/cyrus0315/nodejs-backend-notes/issues)
[![GitHub license](https://img.shields.io/github/license/cyrus0315/nodejs-backend-notes)](https://github.com/cyrus0315/nodejs-backend-notes/blob/main/LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/cyrus0315/nodejs-backend-notes/pulls)

**一份系统、实用的 Node.js 后端技术栈学习笔记** 📚

涵盖 Node.js 核心、TypeScript、数据库、消息队列、框架、ORM、API 设计、安全、测试、AWS、DevOps、系统设计等

[English](./README_EN.md) | 简体中文

</div>

---

## ✨ 项目简介

工作几年下来，发现很多知识点都是零散地学习的，没有系统地整理过。这个仓库是我重新梳理 Node.js 后端技术栈的一个过程，把平时用到的、看过的、踩过坑的东西都归纳一下，方便以后查阅。

### 🎯 适合人群

- 🌱 想系统学习 Node.js 后端开发的新手
- 💼 需要快速查阅技术点的在职开发者
- 📖 准备面试需要复习知识点的求职者
- 🚀 想提升后端技术栈的全栈工程师

### 🌟 项目特点

- ✅ **系统全面**：覆盖 Node.js 后端开发的方方面面
- ✅ **实战导向**：基于真实项目经验，包含大量实用示例
- ✅ **持续更新**：根据工作实践不断补充新内容
- ✅ **中文友好**：适合中文开发者阅读学习
- ✅ **开箱即用**：代码示例可直接运行测试

## 📑 目录

- [✨ 项目简介](#-项目简介)
- [📚 知识整理](#-知识整理)
  - [语言和核心](#语言和核心)
  - [数据存储](#数据存储)
  - [框架和工具](#框架和工具)
  - [API 开发](#api-开发)
  - [云服务](#云服务)
  - [基础巩固](#基础巩固)
  - [架构设计](#架构设计)
- [📝 使用说明](#-使用说明)
- [💭 学习心得](#-学习心得)
- [🤝 贡献指南](#-贡献指南)
- [📄 许可证](#-许可证)
- [⭐ Star History](#-star-history)

## 📚 知识整理

### 语言和核心

- **[01-nodejs-core](./01-nodejs-core)** - Node.js 核心机制
  - Event Loop、Stream、Buffer、进程线程、性能优化等
  - 这部分内容经常被问到，也是最容易被忽略的基础
  
- **[02-typescript](./02-typescript)** - TypeScript 深入
  - 类型系统、泛型、装饰器、工具类型
  - 现在基本都在用 TS 了，把这些整理一下

### 数据存储

- **[03-database](./03-database)** - 数据库
  - PostgreSQL, MySQL, MongoDB, Redis, DynamoDB
  - 项目里用过的数据库，踩过的坑都记录下来

- **[04-message-queue](./04-message-queue)** - 消息队列
  - Kafka, RabbitMQ, AWS SQS/SNS
  - 异步处理和解耦必备

### 框架和工具

- **[05-frameworks](./05-frameworks)** - 框架对比
  - Express, Koa, NestJS, Fastify, Monorepo
  - 各个框架的优缺点，什么场景用什么

- **[06-orm-odm](./06-orm-odm)** - ORM/ODM
  - Prisma, TypeORM, Sequelize, Mongoose
  - 数据库操作工具，用哪个效率更高

### API 开发

- **[07-api-design](./07-api-design)** - API 设计
  - RESTful, GraphQL, 接口规范
  - 好的 API 设计真的能省很多事

- **[08-authentication-security](./08-authentication-security)** - 安全
  - JWT, OAuth, 常见安全问题
  - 安全问题不能忽视

- **[09-testing](./09-testing)** - 测试
  - Jest, 单元测试, 集成测试
  - 写测试真的能减少 bug

### 云服务

- **[10-aws](./10-aws)** - AWS 服务
  - Lambda, API Gateway, SQS, S3, DynamoDB
  - 用 AWS 比较多，把常用的服务整理一下

- **[11-devops](./11-devops)** - DevOps
  - Docker, CI/CD, GitHub Actions
  - 部署和运维的一些实践

- **[12-monitoring-logging](./12-monitoring-logging)** - 监控日志
  - 日志管理、性能监控、错误追踪
  - 线上问题排查必备

### 基础巩固

- **[13-network-protocols](./13-network-protocols)** - 网络协议
  - HTTP/HTTPS, WebSocket, TCP/UDP
  - 基础但很重要

- **[14-algorithms-data-structures](./14-algorithms-data-structures)** - 算法
  - 常见算法、数据结构
  - 虽然不常写，但要理解

### 架构设计

- **[15-system-design](./15-system-design)** - 系统设计
  - 高可用、高并发、分布式
  - 随着项目变大，这些越来越重要

- **[16-architecture](./16-architecture)** - 架构模式
  - 微服务、DDD、事件驱动
  - 不同架构的取舍

### 其他

- **[17-tools-skills](./17-tools-skills)** - 工具技能
  - Git, Linux, 代码规范
  - 日常工作必备

## 📝 使用说明

### 这个仓库怎么用

每个目录下都有详细的笔记和示例代码，按照自己的节奏学习就好。我一般是：

1. 先看 README 了解整体结构
2. 挑自己不熟悉的重点看
3. 把代码示例跑一遍
4. 遇到不懂的就深入研究
5. 有新理解就补充进来

### Checkbox 的用途

每个模块里有很多 `[ ]` checkbox，可以用来标记自己的掌握程度：
- `[ ]` 还没看过或者不太熟
- `[x]` 已经理解掌握了

不用强迫自己全部打勾，根据实际需求来就行。

## 💭 学习心得

### 关于学习方法

1. **不要贪多**：一次专注一个主题，搞透了再换下一个
2. **多动手**：光看不练假把式，代码一定要自己写
3. **做笔记**：理解了的东西写下来，以后翻起来更快
4. **造轮子**：有时候自己实现一遍，理解会更深刻
5. **看源码**：遇到疑惑就去看源码，别猜

### 推荐资源

**书籍**（我看过觉得不错的）：
- 《深入浅出 Node.js》- 朴灵写的，经典
- 《Node.js 设计模式》- 讲了很多实用模式
- 《设计数据密集型应用》- 偏理论但很有用

**网站**：
- Node.js 官方文档 - 最权威的资料
- MDN - Web 相关的都能查到
- GitHub - 看优秀项目的代码

**实践**：
- 写点自己的小项目
- 参与开源项目
- 重构之前的老代码

## 🔄 持续更新

这个仓库会根据工作中遇到的新东西不断更新：
- 新学到的技术点
- 踩过的坑和解决方案
- 性能优化的实践
- 架构设计的思考

如果你也在学 Node.js 后端，欢迎一起交流。

## 📂 目录结构

```
.
├── 01-nodejs-core/          # Node.js 核心
├── 02-typescript/           # TypeScript
├── 03-database/             # 数据库
├── 04-message-queue/        # 消息队列
├── 05-frameworks/           # 框架
├── 06-orm-odm/              # ORM/ODM
├── 07-api-design/           # API 设计
├── 08-authentication-security/  # 安全
├── 09-testing/              # 测试
├── 10-aws/                  # AWS
├── 11-devops/               # DevOps
├── 12-monitoring-logging/   # 监控日志
├── 13-network-protocols/    # 网络协议
├── 14-algorithms-data-structures/  # 算法
├── 15-system-design/        # 系统设计
├── 16-architecture/         # 架构
└── 17-tools-skills/         # 工具技能
```

---

<div align="center">

**持续学习，慢慢积累。** 💪

Made with ❤️ by [cyrus0315](https://github.com/cyrus0315)

如果这个项目对你有帮助，欢迎 ⭐️ Star 支持一下！!!

</div>
