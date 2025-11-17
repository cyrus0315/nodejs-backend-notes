# Monorepo 单仓多包管理

Monorepo 是将多个项目存储在单一代码仓库中的开发策略，在大型 Node.js 项目中越来越流行。

## Monorepo 概念

### 什么是 Monorepo

**定义**：
- 在一个 Git 仓库中管理多个相关项目
- 共享代码、工具和配置
- 统一的版本管理和构建流程

### Monorepo vs Multirepo

| 特性 | Monorepo | Multirepo |
|------|----------|-----------|
| 代码共享 | 简单 | 复杂（需要发包） |
| 依赖管理 | 统一 | 分散 |
| 版本控制 | 统一 | 独立 |
| 构建复杂度 | 高 | 低 |
| CI/CD | 复杂 | 简单 |
| 工具链 | 统一 | 独立 |
| 协作 | 容易 | 困难 |

### 适用场景

**适合 Monorepo**：
- 多个相互依赖的项目
- 共享大量代码
- 需要原子性提交（跨项目）
- 团队协作紧密

**适合 Multirepo**：
- 完全独立的项目
- 不同的技术栈
- 团队完全分离
- 权限严格隔离

## 主流 Monorepo 工具

### 1. Nx

**特点**：
- 强大的构建缓存
- 智能任务调度
- 依赖图可视化
- 支持多框架

#### 创建 Nx Monorepo

```bash
# 创建 Nx 工作区
npx create-nx-workspace@latest my-workspace

# 选择类型
? What to create in the new workspace
  - apps              (empty workspace)
  - nest              (NestJS)
  - express           (Express)
  - react             (React)
  - angular           (Angular)
```

#### 目录结构

```
my-workspace/
├── apps/                   # 应用程序
│   ├── api/               # 后端 API (NestJS)
│   ├── web/               # 前端 Web (React)
│   └── mobile/            # 移动端 (React Native)
├── libs/                   # 共享库
│   ├── shared/            # 共享工具
│   │   ├── data-access/   # 数据访问层
│   │   ├── ui/            # UI 组件
│   │   └── utils/         # 工具函数
│   ├── api/               # API 相关
│   │   ├── auth/          # 认证模块
│   │   └── users/         # 用户模块
│   └── domain/            # 领域模型
├── tools/                  # 工具脚本
├── nx.json                 # Nx 配置
├── workspace.json          # 工作区配置
└── package.json
```

#### Nx 配置

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "parallel": 3
      }
    }
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    }
  },
  "affected": {
    "defaultBase": "main"
  }
}
```

#### 创建库和应用

```bash
# 创建 NestJS 应用
nx g @nrwl/nest:app api

# 创建共享库
nx g @nrwl/node:lib shared-utils

# 创建 NestJS 库
nx g @nrwl/nest:lib api-users --directory=api

# 目录结构
libs/
└── api/
    └── users/
        ├── src/
        │   ├── lib/
        │   │   ├── api-users.module.ts
        │   │   ├── api-users.service.ts
        │   │   └── api-users.controller.ts
        │   └── index.ts
        └── tsconfig.json
```

#### 使用共享库

```typescript
// apps/api/src/app/app.module.ts
import { Module } from '@nestjs/common';
import { ApiUsersModule } from '@my-workspace/api-users';
import { SharedUtilsModule } from '@my-workspace/shared-utils';

@Module({
  imports: [ApiUsersModule, SharedUtilsModule],
})
export class AppModule {}
```

#### Nx 任务执行

```bash
# 运行特定应用
nx serve api
nx serve web

# 构建
nx build api

# 测试
nx test api

# 只构建受影响的项目
nx affected:build

# 只测试受影响的项目
nx affected:test

# 查看依赖图
nx graph
```

#### Nx 缓存

```bash
# 首次构建
nx build api
# 构建时间: 30s

# 再次构建（使用缓存）
nx build api
# 构建时间: 0.1s (从缓存读取)

# 清除缓存
nx reset
```

### 2. Turborepo

**特点**：
- 极速构建
- 远程缓存
- 增量构建
- 简单配置

#### 创建 Turborepo

```bash
# 创建 Turborepo
npx create-turbo@latest

# 目录结构
my-turborepo/
├── apps/
│   ├── api/               # NestJS API
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/               # Next.js Web
│       ├── pages/
│       ├── package.json
│       └── tsconfig.json
├── packages/
│   ├── eslint-config/     # 共享 ESLint 配置
│   ├── tsconfig/          # 共享 TypeScript 配置
│   └── ui/                # 共享 UI 组件
├── turbo.json             # Turborepo 配置
└── package.json
```

#### Turborepo 配置

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

#### 根 package.json

```json
{
  "name": "my-turborepo",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "format": "prettier --write \"**/*.{ts,tsx,md}\""
  },
  "devDependencies": {
    "turbo": "latest",
    "prettier": "latest"
  }
}
```

#### 创建共享包

```typescript
// packages/shared-types/src/index.ts
export interface User {
  id: string;
  email: string;
  name: string;
}

export interface ApiResponse<T> {
  data: T;
  message: string;
  success: boolean;
}
```

```json
// packages/shared-types/package.json
{
  "name": "@my-app/shared-types",
  "version": "1.0.0",
  "main": "./src/index.ts",
  "types": "./src/index.ts"
}
```

#### 在应用中使用

```typescript
// apps/api/src/users/users.service.ts
import { User } from '@my-app/shared-types';

export class UsersService {
  async findOne(id: string): Promise<User> {
    // ...
  }
}
```

```json
// apps/api/package.json
{
  "name": "api",
  "dependencies": {
    "@my-app/shared-types": "*"
  }
}
```

#### Turborepo 执行

```bash
# 运行所有应用的 dev
turbo run dev

# 只运行 api 的 dev
turbo run dev --filter=api

# 构建所有应用
turbo run build

# 并行运行
turbo run build --parallel

# 远程缓存（需要 Vercel 账户）
turbo login
turbo link
```

### 3. pnpm Workspaces

**特点**：
- 磁盘空间节省
- 安装速度快
- 严格的依赖管理
- 原生 monorepo 支持

#### 创建 pnpm Workspace

```yaml
# pnpm-workspace.yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

#### 目录结构

```
my-monorepo/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   └── package.json
│   └── web/
│       ├── src/
│       └── package.json
├── packages/
│   ├── shared-utils/
│   │   ├── src/
│   │   └── package.json
│   └── database/
│       ├── src/
│       └── package.json
├── pnpm-workspace.yaml
├── package.json
└── pnpm-lock.yaml
```

#### 根 package.json

```json
{
  "name": "my-monorepo",
  "private": true,
  "scripts": {
    "dev": "pnpm -r --parallel run dev",
    "build": "pnpm -r run build",
    "test": "pnpm -r run test",
    "lint": "pnpm -r run lint"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "prettier": "^3.0.0"
  }
}
```

#### 包之间的依赖

```json
// apps/api/package.json
{
  "name": "api",
  "dependencies": {
    "@my-app/shared-utils": "workspace:*",
    "@my-app/database": "workspace:*"
  }
}
```

#### pnpm 命令

```bash
# 安装所有依赖
pnpm install

# 为特定包添加依赖
pnpm add express --filter api

# 运行所有包的脚本
pnpm -r run build

# 并行运行
pnpm -r --parallel run dev

# 只运行特定包
pnpm --filter api run dev

# 运行依赖该包的所有包
pnpm --filter ...api run build
```

### 4. Lerna（传统方案）

```bash
# 创建 Lerna 项目
npx lerna init

# 创建包
lerna create @my-app/utils
lerna create @my-app/api

# 安装依赖
lerna bootstrap

# 运行命令
lerna run build
lerna run test

# 发布
lerna publish
```

## 实战：NestJS Monorepo

### 完整项目结构

```
my-monorepo/
├── apps/
│   ├── api-gateway/           # API 网关
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   └── ...
│   │   ├── test/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── user-service/          # 用户微服务
│   └── order-service/         # 订单微服务
├── libs/
│   ├── shared/
│   │   ├── dto/               # 共享 DTO
│   │   ├── entities/          # 共享实体
│   │   ├── enums/             # 枚举
│   │   ├── interfaces/        # 接口
│   │   └── utils/             # 工具函数
│   ├── database/              # 数据库配置
│   ├── auth/                  # 认证模块
│   ├── logging/               # 日志模块
│   └── config/                # 配置模块
├── tools/                      # 构建工具
├── docs/                       # 文档
├── nx.json
├── tsconfig.base.json
└── package.json
```

### 共享库示例

```typescript
// libs/shared/dto/src/user.dto.ts
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(3)
  name: string;

  @IsString()
  @MinLength(6)
  password: string;
}

export class UserResponseDto {
  id: string;
  email: string;
  name: string;
  createdAt: Date;
}
```

```typescript
// libs/shared/entities/src/user.entity.ts
export class User {
  id: string;
  email: string;
  name: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}
```

```typescript
// libs/shared/utils/src/hash.util.ts
import * as bcrypt from 'bcrypt';

export class HashUtil {
  static async hash(password: string): Promise<string> {
    return bcrypt.hash(password, 10);
  }

  static async compare(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }
}
```

### tsconfig.base.json 配置

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@my-app/shared/dto": ["libs/shared/dto/src/index.ts"],
      "@my-app/shared/entities": ["libs/shared/entities/src/index.ts"],
      "@my-app/shared/utils": ["libs/shared/utils/src/index.ts"],
      "@my-app/database": ["libs/database/src/index.ts"],
      "@my-app/auth": ["libs/auth/src/index.ts"],
      "@my-app/config": ["libs/config/src/index.ts"]
    }
  }
}
```

### 在应用中使用共享库

```typescript
// apps/api-gateway/src/users/users.controller.ts
import { Controller, Post, Body } from '@nestjs/common';
import { CreateUserDto } from '@my-app/shared/dto';
import { UsersService } from './users.service';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  async create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
}
```

```typescript
// apps/api-gateway/src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { CreateUserDto, UserResponseDto } from '@my-app/shared/dto';
import { User } from '@my-app/shared/entities';
import { HashUtil } from '@my-app/shared/utils';

@Injectable()
export class UsersService {
  async create(dto: CreateUserDto): Promise<UserResponseDto> {
    const hashedPassword = await HashUtil.hash(dto.password);
    
    const user: User = {
      id: generateId(),
      email: dto.email,
      name: dto.name,
      password: hashedPassword,
      createdAt: new Date(),
      updatedAt: new Date()
    };
    
    // 保存到数据库...
    
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      createdAt: user.createdAt
    };
  }
}
```

## CI/CD 配置

### GitHub Actions for Nx

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0  # 需要完整历史来计算 affected
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Derive appropriate SHAs for base and head
        uses: nrwl/nx-set-shas@v3
      
      - name: Lint affected
        run: npx nx affected --target=lint --parallel=3
      
      - name: Test affected
        run: npx nx affected --target=test --parallel=3 --ci --code-coverage
      
      - name: Build affected
        run: npx nx affected --target=build --parallel=3
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

### Turborepo CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Build
        run: pnpm turbo run build
      
      - name: Test
        run: pnpm turbo run test
      
      - name: Lint
        run: pnpm turbo run lint
```

## 最佳实践

### 1. 代码组织

```typescript
// ✅ 好的实践：按功能分离
libs/
├── shared/
│   ├── dto/              # 只包含 DTO
│   ├── entities/         # 只包含实体
│   └── utils/            # 只包含工具
├── auth/                 # 完整的认证模块
└── logging/              # 完整的日志模块

// ❌ 不好的实践：混合所有东西
libs/
└── shared/
    ├── user.dto.ts
    ├── user.entity.ts
    ├── auth.guard.ts
    ├── logger.service.ts
    └── utils.ts
```

### 2. 依赖管理

```json
// ✅ 在根 package.json 管理共享依赖
{
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "prettier": "^3.0.0"
  }
}

// ✅ 在具体包中管理特定依赖
// apps/api/package.json
{
  "dependencies": {
    "@nestjs/core": "^10.0.0",
    "@nestjs/platform-express": "^10.0.0"
  }
}
```

### 3. 版本控制

```bash
# 使用 workspace:* 引用本地包
{
  "dependencies": {
    "@my-app/shared": "workspace:*"
  }
}

# pnpm 会自动链接到本地版本
```

### 4. 构建顺序

```json
// turbo.json - 定义依赖关系
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // 先构建依赖
      "outputs": ["dist/**"]
    }
  }
}
```

### 5. 测试策略

```bash
# 只测试受影响的包
nx affected:test

# 测试所有包
nx run-many --target=test --all

# 并行测试
nx run-many --target=test --all --parallel=3
```

## 常见面试题

### 1. 为什么使用 Monorepo？

<details>
<summary>点击查看答案</summary>

**优势**：

1. **代码共享简单**：
   - 直接 import，无需发包
   - 原子性提交（跨项目）

2. **统一工具链**：
   - 统一的 TypeScript 配置
   - 统一的 ESLint、Prettier
   - 统一的构建工具

3. **简化依赖管理**：
   - 统一的版本管理
   - 避免版本冲突

4. **更好的协作**：
   - 一次 PR 可以改多个项目
   - 更容易重构

5. **高效 CI/CD**：
   - 只构建/测试受影响的项目
   - 构建缓存

**劣势**：

1. **仓库体积大**：
   - Git 操作慢
   - CI/CD 时间长

2. **权限控制难**：
   - 所有人看到所有代码

3. **工具链复杂**：
   - 需要 Nx/Turborepo
   - 学习成本高
</details>

### 2. Nx vs Turborepo vs pnpm 如何选择？

<details>
<summary>点击查看答案</summary>

| 工具 | 优势 | 适用场景 |
|------|------|----------|
| **Nx** | 功能最强大、依赖图、缓存 | 大型企业项目 |
| **Turborepo** | 简单、快速、远程缓存 | 中小型项目 |
| **pnpm** | 原生、灵活、省空间 | 简单 monorepo |

**选择建议**：
- 复杂企业级 → Nx
- 快速上手 → Turborepo
- 简单需求 → pnpm workspaces
</details>

### 3. Monorepo 如何优化 CI/CD？

<details>
<summary>点击查看答案</summary>

**优化策略**：

1. **只构建受影响的项目**：
```bash
# Nx
nx affected:build

# Turborepo
turbo run build --filter=[HEAD^1]
```

2. **并行执行**：
```bash
nx run-many --target=test --all --parallel=3
```

3. **使用缓存**：
   - 本地缓存
   - 远程缓存（Nx Cloud、Vercel）

4. **增量构建**：
   - 只重新构建改变的部分

**效果**：
- 未优化：构建所有项目（20分钟）
- 优化后：只构建受影响（3分钟）
</details>

### 4. 如何在 Monorepo 中共享 TypeScript 配置？

<details>
<summary>点击查看答案</summary>

```json
// tsconfig.base.json（根目录）
{
  "compilerOptions": {
    "target": "ES2021",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@my-app/*": ["libs/*/src"]
    }
  }
}

// apps/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist"
  },
  "include": ["src/**/*"]
}

// libs/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist"
  }
}
```
</details>

### 5. Monorepo 如何处理版本发布？

<details>
<summary>点击查看答案</summary>

**策略**：

1. **统一版本**（推荐）：
   - 所有包使用相同版本
   - 简单易管理
   ```bash
   lerna publish --conventional-commits
   ```

2. **独立版本**：
   - 每个包独立版本
   - 更灵活，但复杂
   ```bash
   lerna publish --independent
   ```

3. **语义化版本**：
   - 使用 conventional commits
   - 自动生成 CHANGELOG
   ```bash
   # feat: 新功能 -> minor
   # fix: 修复 -> patch
   # BREAKING CHANGE -> major
   ```
</details>

## 总结

Monorepo 关键点：
1. 选择合适的工具（Nx/Turborepo/pnpm）
2. 合理组织代码结构
3. 优化构建和测试
4. 使用缓存提升效率
5. 统一工具链和配置

