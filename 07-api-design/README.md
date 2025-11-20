# API 设计

API 设计是后端开发的核心技能。本模块深入讲解 RESTful API、GraphQL、API 文档和版本控制的设计原则与最佳实践。

## 📚 学习文档

### ✅ 已完成

1. **[RESTful API 设计](./01-restful-api.md)** (40KB)
   - REST 六大约束
   - HTTP 方法详解（GET、POST、PUT、PATCH、DELETE）
   - 状态码使用规范（2xx、4xx、5xx）
   - URL 设计规范
   - 请求与响应格式
   - 分页、排序、过滤
   - 错误处理
   - 最佳实践（HATEOAS、幂等性、限流、缓存、CORS）
   - Node.js/TypeScript 完整实现
   - 5+ 道高频面试题

2. **[GraphQL](./02-graphql.md)** (45KB)
   - GraphQL 核心概念
   - Schema 定义（Type、Query、Mutation、Subscription）
   - Resolver 实现（Query、Mutation、Field Resolver）
   - 查询与变更（片段、别名、指令）
   - 分页实现（Cursor 分页）
   - N+1 问题及解决方案（DataLoader）
   - 认证与授权（JWT、装饰器、字段级权限）
   - 错误处理（自定义错误、全局处理）
   - 性能优化（复杂度限制、深度限制、缓存）
   - GraphQL vs REST 对比
   - 5+ 道高频面试题

3. **[API 文档与版本控制](./03-api-docs-versioning.md)** (35KB)
   - OpenAPI/Swagger 规范
   - 自动生成文档：
     - swagger-jsdoc + swagger-ui-express
     - @nestjs/swagger
     - tsoa
   - API 版本控制方案：
     - URL 版本（推荐）
     - Header 版本
     - 内容协商
   - API 弃用流程
   - 变更日志
   - 迁移指南
   - 版本兼容性测试
   - 5+ 道高频面试题

## 📝 知识点清单

### RESTful API
- [x] REST 设计原则
- [x] HTTP 方法（GET, POST, PUT, PATCH, DELETE）
- [x] 状态码使用规范
- [x] URL 设计规范
- [x] 请求与响应格式
- [x] 分页、排序、过滤
- [x] 错误处理
- [x] HATEOAS
- [x] 幂等性设计
- [x] 缓存策略

### GraphQL
- [x] GraphQL 基础概念
- [x] Schema 定义
- [x] Resolver 实现
- [x] Query、Mutation、Subscription
- [x] DataLoader（解决 N+1 问题）
- [x] 认证与授权
- [x] 错误处理
- [x] 性能优化
- [x] GraphQL vs REST

### API 文档
- [x] Swagger/OpenAPI 规范
- [x] 自动生成文档
- [x] API 文档最佳实践

### API 版本控制
- [x] URL 版本控制
- [x] Header 版本控制
- [x] 内容协商
- [x] 版本管理策略
- [x] API 弃用流程

## 🎯 学习建议

### 基础阶段（必修）
1. 理解 REST 六大约束
2. 掌握 HTTP 方法和状态码
3. 学习 URL 设计规范
4. 理解幂等性和安全性

### 进阶阶段（深入）
1. 深入理解 GraphQL
2. 掌握 DataLoader 解决 N+1
3. 学习 API 版本控制策略
4. 实践 API 文档自动生成

### 实战阶段（应用）
1. 设计完整的 RESTful API
2. 实现 GraphQL API
3. 编写 API 文档
4. 实现版本控制

## 💡 重点难点

### 必须掌握 ⭐⭐⭐
- REST 设计原则
- HTTP 方法正确使用
- 状态码规范
- URL 设计规范
- 分页实现（Cursor 分页）
- 错误处理
- API 版本控制（URL 版本）

### 高级主题 ⭐⭐
- GraphQL Schema 设计
- DataLoader 实现
- GraphQL 认证授权
- API 性能优化
- 查询复杂度限制
- API 弃用流程

### 实战技能 ⭐⭐⭐
- RESTful API 设计
- GraphQL API 设计
- API 文档生成
- 版本控制实践

## 🔥 常见面试题

### RESTful API

1. **PUT 和 PATCH 的区别？**
   - PUT：完整替换资源，幂等
   - PATCH：部分更新资源，不严格幂等

2. **如何设计幂等性接口？**
   - GET、PUT、DELETE 天然幂等
   - POST 使用幂等性键（Idempotency-Key）

3. **RESTful API 的优缺点？**
   - 优点：简单、无状态、可缓存
   - 缺点：过度获取、多次请求、版本管理困难

4. **如何设计 API 分页？**
   - Offset 分页：简单但性能差
   - Cursor 分页：性能好（推荐）

### GraphQL

1. **GraphQL 的优缺点？**
   - 优点：按需查询、单次请求、强类型
   - 缺点：学习曲线陡、缓存复杂、查询复杂度

2. **如何解决 N+1 问题？**
   - 使用 DataLoader 批量查询和缓存

3. **GraphQL 如何做认证授权？**
   - 认证：Context 中解析 token
   - 授权：@Authorized 装饰器、Field Resolver

4. **GraphQL vs REST，如何选择？**
   - 复杂关系、移动应用 → GraphQL
   - 简单 CRUD、文件上传 → REST

### API 文档与版本

1. **API 版本控制有哪些方案？**
   - URL 版本（推荐）：`/api/v1/users`
   - Header 版本：`API-Version: 1`
   - 内容协商：`Accept: app/vnd.api.v1+json`

2. **如何优雅地弃用 API？**
   - 提前公告（6 个月）
   - 添加响应头（Deprecation、Sunset）
   - 提供迁移指南
   - 通知用户

3. **如何生成 API 文档？**
   - Express: swagger-jsdoc
   - NestJS: @nestjs/swagger
   - TypeScript: tsoa

## 📊 内容统计

- **3 个核心主题**
- **120KB+ 详细内容**
- **50+ 个代码示例**
- **15+ 道高频面试题**
- **完整的实战案例**

## 🎓 技术选型建议

### REST vs GraphQL

| 场景 | 推荐 | 理由 |
|------|------|------|
| **简单 CRUD** | REST | 实现简单 |
| **复杂数据关系** | GraphQL | 减少请求 |
| **移动应用** | GraphQL | 节省流量 |
| **公共 API** | REST | 易于使用 |
| **内部 API** | GraphQL | 灵活性高 |
| **文件上传** | REST | 原生支持 |
| **实时数据** | GraphQL | Subscription |
| **微服务** | REST | 简单可靠 |

### 文档生成工具

| 工具 | 适用框架 | 特点 |
|------|---------|------|
| **swagger-jsdoc** | Express | JSDoc 注释 |
| **@nestjs/swagger** | NestJS | 装饰器，自动生成 |
| **tsoa** | Express/Koa | TypeScript 优先 |
| **GraphQL Playground** | GraphQL | 内置文档 |

### 版本控制方案

| 方案 | 适用场景 | 推荐度 |
|------|---------|--------|
| **URL 版本** | 大多数场景 | ⭐⭐⭐⭐⭐ |
| **Header 版本** | RESTful 纯粹主义 | ⭐⭐⭐ |
| **查询参数** | 简单场景 | ⭐⭐ |
| **内容协商** | 学术场景 | ⭐⭐ |

## ✅ 检查清单

完成以下内容后，你应该能够：

### REST API
- [ ] 理解 REST 设计原则
- [ ] 正确使用 HTTP 方法
- [ ] 正确使用状态码
- [ ] 设计规范的 URL
- [ ] 实现分页、排序、过滤
- [ ] 实现完善的错误处理
- [ ] 理解幂等性
- [ ] 实现 API 缓存

### GraphQL
- [ ] 定义 GraphQL Schema
- [ ] 实现 Query 和 Mutation
- [ ] 实现 Field Resolver
- [ ] 使用 DataLoader 解决 N+1
- [ ] 实现认证和授权
- [ ] 处理错误
- [ ] 优化查询性能
- [ ] 对比 REST 和 GraphQL

### API 文档与版本
- [ ] 编写 OpenAPI 规范
- [ ] 自动生成 API 文档
- [ ] 实现 API 版本控制
- [ ] 编写变更日志
- [ ] 编写迁移指南
- [ ] 实现 API 弃用流程
- [ ] 测试版本兼容性

## 📚 推荐资源

### 书籍
- 《RESTful Web APIs》- Leonard Richardson
- 《GraphQL in Action》- Samer Buna
- 《API Design Patterns》- JJ Geewax

### 文档
- [REST API Tutorial](https://restfulapi.net/)
- [GraphQL Official](https://graphql.org/)
- [OpenAPI Specification](https://swagger.io/specification/)
- [Swagger Editor](https://editor.swagger.io/)

### 工具
- [Postman](https://www.postman.com/) - API 测试
- [Insomnia](https://insomnia.rest/) - API 客户端
- [GraphQL Playground](https://github.com/graphql/graphql-playground) - GraphQL IDE
- [Swagger UI](https://swagger.io/tools/swagger-ui/) - API 文档

## 🚀 下一步

完成本模块后，建议：
1. 设计一个完整的 RESTful API
2. 实现一个 GraphQL API
3. 为 API 生成文档
4. 实践 API 版本控制
5. 阅读优秀开源项目的 API 设计
6. 学习 API 网关（Kong、Tyk）

## 🎯 面试评分标准

### 初级（0-2 年）
- 理解 REST 基本概念
- 会使用 HTTP 方法和状态码
- 能设计简单的 API
- 了解 API 文档

### 中级（2-4 年）
- 深入理解 REST 设计原则
- 掌握 GraphQL 基础
- 能设计复杂的 API
- 实现 API 版本控制
- 掌握性能优化技巧

### 高级（4+ 年）
- 精通 REST 和 GraphQL
- 能够权衡技术方案
- 有大规模 API 设计经验
- 能够指导团队 API 设计
- 深入理解 API 网关和微服务

---

**上一模块**：[06-orm-odm（ORM/ODM）](../06-orm-odm/README.md)  
**下一模块**：[08-authentication-security（认证与安全）](../08-authentication-security/README.md)
