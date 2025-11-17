# TypeScript

本模块已创建详细的学习文档，涵盖 TypeScript 从基础到高级的所有核心知识。

## 📚 学习文档

### ✅ 已完成

1. **[基础类型](./01-basic-types.md)** (25KB)
   - 原始类型和特殊类型
   - any vs unknown 详解
   - 数组、元组、对象类型
   - 接口 (Interface) 完整指南
   - 类型别名 (Type Alias)
   - 字面量类型和枚举
   - 类型断言和类型守卫
   - 5+ 道面试题及解答

2. **[高级类型](./02-advanced-types.md)** (27KB)
   - 联合类型和交叉类型
   - 条件类型 (Conditional Types)
   - 映射类型 (Mapped Types)
   - 索引类型和 keyof
   - 模板字面量类型
   - infer 关键字深度解析
   - 工具类型 (Utility Types) 详解
   - 类型推断和类型断言
   - 5+ 道面试题及解答

3. **[泛型](./03-generics.md)** (25KB)
   - 泛型基础和使用场景
   - 泛型函数、接口、类
   - 泛型约束
   - 泛型默认值
   - 协变和逆变
   - 实战：事件发射器、状态管理、API 客户端
   - 泛型构建器模式
   - 5+ 道面试题及解答

4. **[装饰器与配置](./04-decorators-config.md)** (24KB)
   - 类装饰器、方法装饰器
   - 访问器装饰器、属性装饰器
   - 参数装饰器
   - 装饰器执行顺序
   - NestJS 风格装饰器实现
   - tsconfig.json 完整配置详解
   - 不同场景的配置方案
   - 路径映射和项目引用
   - 5+ 道面试题及解答

## 📝 知识点清单

### 基础类型系统
- [x] 基本类型（string, number, boolean, etc.）
- [x] 数组与元组
- [x] 枚举（Enum）
- [x] Any, Unknown, Never, Void
- [x] 类型断言与类型守卫
- [x] 字面量类型

### 高级类型
- [x] 联合类型（Union）
- [x] 交叉类型（Intersection）
- [x] 类型别名（Type Alias）
- [x] 接口（Interface）
- [x] 泛型（Generics）
- [x] 条件类型
- [x] 映射类型
- [x] 索引类型
- [x] 工具类型（Utility Types）

### 泛型系统
- [x] 泛型函数、接口、类
- [x] 泛型约束
- [x] 泛型默认值
- [x] 协变和逆变
- [x] 高级泛型模式

### 装饰器
- [x] 类装饰器
- [x] 方法装饰器
- [x] 访问器装饰器
- [x] 属性装饰器
- [x] 参数装饰器
- [x] 装饰器组合

### 配置与工程化
- [x] tsconfig.json 配置详解
- [x] 编译选项
- [x] 路径映射
- [x] 严格模式
- [x] 项目引用

## 🎯 学习建议

1. **按顺序学习**：建议按照文档编号顺序（01 → 04）学习
2. **动手实践**：每个文档都包含大量代码示例，务必自己运行
3. **理解原理**：不仅要会用，更要理解类型系统的设计原理
4. **做练习题**：通过面试题巩固知识
5. **实战应用**：将学到的知识应用到实际项目中

## 💡 重点难点

### 必须掌握 ⭐⭐⭐
- any vs unknown 的区别和使用
- Interface vs Type 的选择
- 泛型的基本使用和约束
- 常用工具类型（Partial, Required, Pick, Omit）
- 类型守卫和类型收窄
- tsconfig.json 核心配置

### 高级主题 ⭐⭐
- 条件类型和 infer
- 映射类型
- 模板字面量类型
- 装饰器的实现和使用
- 协变和逆变
- 复杂泛型模式

### 实战技能 ⭐⭐⭐
- 类型安全的 API 设计
- 类型安全的事件系统
- 类型安全的状态管理
- 装饰器在框架中的应用

## 🔥 常见面试题

每个文档都包含了该主题的常见面试题和详细解答：

1. **基础类型**
   - any 和 unknown 的区别？
   - interface 和 type 的区别？
   - never 类型的使用场景？
   - 什么时候使用类型断言？
   - 如何定义一个只读对象？

2. **高级类型**
   - 什么是类型收窄？
   - 联合类型和交叉类型的区别？
   - 如何实现深度 Readonly？
   - Partial 和 Required 如何实现？
   - 如何提取函数的参数和返回类型？

3. **泛型**
   - 什么时候需要使用泛型？
   - 泛型约束有什么作用？
   - 如何实现类型安全的深度克隆？
   - 如何实现类型安全的 Omit？
   - 泛型的协变和逆变是什么？

4. **装饰器与配置**
   - 装饰器的执行顺序是什么？
   - tsconfig.json 中 strict 包含哪些选项？
   - 如何实现一个日志装饰器？
   - baseUrl 和 paths 的区别？
   - 如何在运行时使用装饰器元数据？

## 📊 内容统计

- **4 个主题文档**
- **100+ 个代码示例**
- **20+ 道面试题**
- **10+ 个实战案例**
- **总计 100KB+ 高质量内容**

## 🎓 实战应用

文档中包含多个完整的实战案例：

1. **类型安全的事件发射器**
2. **类型安全的状态管理**
3. **类型安全的 API 客户端**
4. **泛型构建器模式**
5. **NestJS 风格的装饰器系统**
6. **日志和性能监控装饰器**

## 📚 推荐资源

- [TypeScript 官方文档](https://www.typescriptlang.org/docs/)
- [TypeScript Deep Dive](https://basarat.gitbook.io/typescript/)
- [Type Challenges](https://github.com/type-challenges/type-challenges) - 类型体操练习
- [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) - 类型定义库

## 🚀 下一步

完成本模块后，建议：
1. 在实际项目中应用 TypeScript
2. 尝试为现有 JavaScript 项目添加类型
3. 练习 Type Challenges 的题目
4. 学习框架（NestJS）中的 TypeScript 应用
5. 阅读优秀开源项目的类型定义

