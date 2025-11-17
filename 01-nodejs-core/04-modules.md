# 模块系统

## CommonJS vs ES Modules

### CommonJS (CJS)

Node.js 的传统模块系统，使用 `require()` 和 `module.exports`。

```javascript
// math.js - 导出
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

// 方式 1: 逐个导出
exports.add = add;
exports.subtract = subtract;

// 方式 2: 整体导出
module.exports = {
  add,
  subtract
};

// 方式 3: 导出单个函数
module.exports = function(a, b) {
  return a + b;
};

// app.js - 导入
const math = require('./math');
console.log(math.add(1, 2));

// 解构导入
const { add, subtract } = require('./math');
console.log(add(1, 2));
```

### ES Modules (ESM)

现代 JavaScript 的标准模块系统。

```javascript
// math.mjs - 导出
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}

// 默认导出
export default function multiply(a, b) {
  return a * b;
}

// app.mjs - 导入
import multiply, { add, subtract } from './math.mjs';
console.log(add(1, 2));
console.log(multiply(2, 3));

// 导入所有
import * as math from './math.mjs';
console.log(math.add(1, 2));

// 重命名导入
import { add as sum } from './math.mjs';
console.log(sum(1, 2));

// 动态导入
const math = await import('./math.mjs');
console.log(math.add(1, 2));
```

### 在 Node.js 中使用 ESM

**方式 1：使用 .mjs 扩展名**

```javascript
// file.mjs
import fs from 'fs';
```

**方式 2：在 package.json 中设置 "type": "module"**

```json
// package.json
{
  "type": "module"
}
```

```javascript
// file.js (现在被视为 ESM)
import fs from 'fs';
```

**方式 3：使用 .cjs 扩展名保持 CommonJS**

```javascript
// file.cjs
const fs = require('fs');
```

### CommonJS vs ESM 对比

| 特性 | CommonJS | ES Modules |
|------|----------|-----------|
| **语法** | `require()` / `module.exports` | `import` / `export` |
| **加载时机** | 运行时（同步） | 编译时（可以静态分析） |
| **动态导入** | 原生支持 | 需要 `import()` |
| **顶层 await** | 不支持 | 支持 |
| **this 值** | `module.exports` | `undefined` |
| **文件扩展名** | .js, .cjs | .mjs, .js (需配置) |
| **浏览器支持** | 不支持 | 原生支持 |
| **Tree Shaking** | 困难 | 容易 |

### 互操作性

```javascript
// ESM 中导入 CommonJS
// cjs-module.js (CommonJS)
module.exports = {
  name: 'CommonJS Module'
};

// esm-module.mjs (ESM)
import cjsModule from './cjs-module.js';
console.log(cjsModule.name);

// CommonJS 中导入 ESM（需要动态导入）
// app.js (CommonJS)
async function loadESM() {
  const esmModule = await import('./esm-module.mjs');
  console.log(esmModule.default);
}

loadESM();
```

## Module Resolution（模块解析）

### 解析规则

```javascript
// 1. 核心模块（优先级最高）
const fs = require('fs');
const http = require('http');

// 2. 文件或目录（相对路径或绝对路径）
const myModule = require('./myModule');      // ./myModule.js
const myModule = require('./myModule.json'); // JSON 文件
const myModule = require('./myModule/');     // ./myModule/index.js

// 3. node_modules 目录
const express = require('express');
// 查找顺序：
// ./node_modules/express
// ../node_modules/express
// ../../node_modules/express
// ...直到根目录
```

### 文件扩展名解析

```javascript
// Node.js 会按以下顺序尝试：
require('./module')
// 1. ./module.js
// 2. ./module.json
// 3. ./module.node (C++ addon)
// 4. ./module/index.js
// 5. ./module/index.json
// 6. ./module/index.node
```

### package.json 的作用

```json
{
  "name": "my-module",
  "version": "1.0.0",
  "main": "./lib/index.js",        // CommonJS 入口
  "module": "./lib/index.mjs",     // ESM 入口（非官方标准）
  "exports": {                      // 新的导出字段（推荐）
    ".": {
      "require": "./lib/index.js",
      "import": "./lib/index.mjs"
    },
    "./utils": {
      "require": "./lib/utils.js",
      "import": "./lib/utils.mjs"
    }
  },
  "type": "module",                 // 设置为 ESM
  "imports": {                      // 内部导入映射
    "#utils": "./src/utils/index.js"
  }
}
```

## 模块缓存

### 缓存机制

```javascript
// counter.js
let count = 0;

module.exports = {
  increment() {
    count++;
  },
  getCount() {
    return count;
  }
};

// app.js
const counter1 = require('./counter');
const counter2 = require('./counter');

counter1.increment();
console.log(counter2.getCount()); // 1

console.log(counter1 === counter2); // true（同一个对象）
```

### 查看和清除缓存

```javascript
// 查看缓存
console.log(require.cache);

// 清除特定模块的缓存
delete require.cache[require.resolve('./module')];

// 重新加载模块
const freshModule = require('./module');

// 清除所有缓存（不推荐）
Object.keys(require.cache).forEach(key => {
  delete require.cache[key];
});
```

### 避免循环依赖

```javascript
// ❌ 不好：循环依赖
// a.js
const b = require('./b');
module.exports = { name: 'A', b };

// b.js
const a = require('./a');
module.exports = { name: 'B', a };

// app.js
const a = require('./a');
console.log(a); // { name: 'A', b: { name: 'B', a: {} } }
// a.b.a 是空对象，因为循环依赖

// ✅ 好：重构避免循环依赖
// a.js
module.exports = { name: 'A' };

// b.js
module.exports = { name: 'B' };

// c.js
const a = require('./a');
const b = require('./b');
module.exports = { a, b };
```

## 内置模块

### 常用核心模块

```javascript
// 1. path - 路径处理
const path = require('path');

path.join('/user', 'local', 'bin');        // '/user/local/bin'
path.resolve('index.js');                  // 绝对路径
path.basename('/user/local/bin/file.txt'); // 'file.txt'
path.dirname('/user/local/bin/file.txt');  // '/user/local/bin'
path.extname('file.txt');                  // '.txt'
path.parse('/user/local/bin/file.txt');    // 解析为对象
path.format({ root: '/', base: 'file.txt' }); // 组合路径

// 2. os - 操作系统信息
const os = require('os');

os.platform();      // 'darwin', 'linux', 'win32'
os.arch();          // 'x64', 'arm64'
os.cpus();          // CPU 信息数组
os.totalmem();      // 总内存（字节）
os.freemem();       // 空闲内存（字节）
os.homedir();       // 用户主目录
os.tmpdir();        // 临时目录
os.hostname();      // 主机名
os.networkInterfaces(); // 网络接口

// 3. util - 实用工具
const util = require('util');

// promisify - 将回调转为 Promise
const fs = require('fs');
const readFile = util.promisify(fs.readFile);
await readFile('file.txt', 'utf8');

// format - 格式化字符串
util.format('%s:%s', 'foo', 'bar'); // 'foo:bar'

// inspect - 对象转字符串
util.inspect({ a: 1, b: 2 }, { colors: true });

// types - 类型检查
util.types.isPromise(Promise.resolve());

// 4. crypto - 加密
const crypto = require('crypto');

// 哈希
const hash = crypto.createHash('sha256');
hash.update('password');
const hashed = hash.digest('hex');

// HMAC
const hmac = crypto.createHmac('sha256', 'secret');
hmac.update('data');
const signature = hmac.digest('hex');

// 随机字节
const randomBytes = crypto.randomBytes(16).toString('hex');

// 5. events - 事件发射器
const EventEmitter = require('events');

class MyEmitter extends EventEmitter {}

const emitter = new MyEmitter();

emitter.on('event', (arg) => {
  console.log('Event triggered:', arg);
});

emitter.emit('event', 'data');
```

### fs 模块详解

```javascript
const fs = require('fs');
const fsPromises = require('fs').promises;

// 同步操作（阻塞）
const data = fs.readFileSync('file.txt', 'utf8');

// 异步回调
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Promise API（推荐）
const data = await fsPromises.readFile('file.txt', 'utf8');

// 写文件
await fsPromises.writeFile('output.txt', 'Hello World');

// 追加文件
await fsPromises.appendFile('log.txt', 'New line\n');

// 删除文件
await fsPromises.unlink('file.txt');

// 重命名/移动文件
await fsPromises.rename('old.txt', 'new.txt');

// 文件信息
const stats = await fsPromises.stat('file.txt');
console.log({
  size: stats.size,
  isFile: stats.isFile(),
  isDirectory: stats.isDirectory(),
  mtime: stats.mtime
});

// 目录操作
await fsPromises.mkdir('newdir', { recursive: true });
const files = await fsPromises.readdir('directory');
await fsPromises.rmdir('directory', { recursive: true });

// 检查文件是否存在
try {
  await fsPromises.access('file.txt');
  console.log('File exists');
} catch {
  console.log('File does not exist');
}

// 复制文件
await fsPromises.copyFile('source.txt', 'dest.txt');

// 监听文件变化
fs.watch('file.txt', (eventType, filename) => {
  console.log(`${filename} changed: ${eventType}`);
});
```

## 创建自己的模块

### 基本模块

```javascript
// logger.js
const chalk = require('chalk'); // 假设已安装

class Logger {
  constructor(prefix = '') {
    this.prefix = prefix;
  }
  
  log(message) {
    console.log(`${this.prefix}${message}`);
  }
  
  error(message) {
    console.error(chalk.red(`${this.prefix}ERROR: ${message}`));
  }
  
  warn(message) {
    console.warn(chalk.yellow(`${this.prefix}WARN: ${message}`));
  }
  
  info(message) {
    console.info(chalk.blue(`${this.prefix}INFO: ${message}`));
  }
}

module.exports = Logger;

// 使用
const Logger = require('./logger');
const logger = new Logger('[MyApp] ');
logger.info('Application started');
```

### 发布 npm 包

```json
// package.json
{
  "name": "@username/my-package",
  "version": "1.0.0",
  "description": "My awesome package",
  "main": "index.js",
  "module": "index.mjs",
  "types": "index.d.ts",
  "exports": {
    ".": {
      "require": "./index.js",
      "import": "./index.mjs",
      "types": "./index.d.ts"
    }
  },
  "files": [
    "index.js",
    "index.mjs",
    "index.d.ts",
    "lib/"
  ],
  "keywords": ["awesome", "package"],
  "author": "Your Name",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/username/my-package"
  },
  "bugs": {
    "url": "https://github.com/username/my-package/issues"
  },
  "engines": {
    "node": ">=14.0.0"
  },
  "dependencies": {},
  "devDependencies": {},
  "peerDependencies": {}
}
```

```bash
# 发布流程
npm login
npm publish --access public  # 对于 scoped packages
```

## 常见面试题

### 1. require 和 import 的区别？

<details>
<summary>点击查看答案</summary>

**语法差异**：

```javascript
// CommonJS (require)
const module = require('module');
const { func } = require('module');

// ES Modules (import)
import module from 'module';
import { func } from 'module';
import * as module from 'module';
```

**关键区别**：

1. **加载时机**：
   - `require`: 运行时同步加载
   - `import`: 编译时加载（静态）

2. **可以动态导入**：
   - `require`: 可以在任何地方使用
   - `import`: 必须在顶层（动态导入需使用 `import()`）

3. **导出方式**：
   - `require`: `module.exports` / `exports`
   - `import`: `export` / `export default`

4. **性能**：
   - `require`: 无法 tree shaking
   - `import`: 支持 tree shaking

5. **循环依赖**：
   - `require`: 返回当前已导出的部分
   - `import`: 返回未初始化的绑定（可能导致错误）

**选择建议**：
- 新项目 → ES Modules
- 需要兼容旧代码 → CommonJS
- 库开发 → 提供两种格式
</details>

### 2. module.exports 和 exports 的区别？

<details>
<summary>点击查看答案</summary>

```javascript
// 初始状态
// exports = module.exports = {}

// ✅ 正确使用 exports
exports.foo = 'bar';
exports.func = function() {};

// 等价于
module.exports.foo = 'bar';
module.exports.func = function() {};

// ❌ 错误：重新赋值 exports
exports = { foo: 'bar' };
// 这不会导出，因为改变了 exports 的引用

// ✅ 正确：重新赋值 module.exports
module.exports = { foo: 'bar' };

// ❌ 混用会导致问题
exports.foo = 'bar';
module.exports = { baz: 'qux' };
// 最终导出：{ baz: 'qux' }，foo 丢失
```

**规则**：
- `module.exports` 是实际导出的对象
- `exports` 只是 `module.exports` 的引用
- 如果重新赋值 `module.exports`，`exports` 失效
- **建议**：只使用 `module.exports`，避免混淆
</details>

### 3. 如何实现一个简单的 require？

<details>
<summary>点击查看答案</summary>

```javascript
const fs = require('fs');
const path = require('path');
const vm = require('vm');

function myRequire(modulePath) {
  // 1. 解析路径
  const absolutePath = path.resolve(__dirname, modulePath);
  
  // 2. 检查缓存
  if (myRequire.cache[absolutePath]) {
    return myRequire.cache[absolutePath].exports;
  }
  
  // 3. 读取文件内容
  const code = fs.readFileSync(absolutePath, 'utf8');
  
  // 4. 创建模块对象
  const module = {
    exports: {},
    id: absolutePath,
    filename: absolutePath,
    loaded: false
  };
  
  // 5. 包装代码
  const wrapper = [
    '(function(exports, require, module, __filename, __dirname) { ',
    '\n});'
  ];
  const wrappedCode = wrapper[0] + code + wrapper[1];
  
  // 6. 执行代码
  const compiledWrapper = vm.runInThisContext(wrappedCode);
  
  compiledWrapper.call(
    module.exports,
    module.exports,
    myRequire,
    module,
    absolutePath,
    path.dirname(absolutePath)
  );
  
  // 7. 标记为已加载
  module.loaded = true;
  
  // 8. 缓存模块
  myRequire.cache[absolutePath] = module;
  
  // 9. 返回导出
  return module.exports;
}

myRequire.cache = {};

// 测试
const myModule = myRequire('./myModule.js');
```
</details>

### 4. 如何解决循环依赖？

<details>
<summary>点击查看答案</summary>

**问题示例**：

```javascript
// a.js
const b = require('./b');
console.log('a.js:', b);
module.exports = { name: 'A' };

// b.js
const a = require('./a');
console.log('b.js:', a);
module.exports = { name: 'B' };

// main.js
require('./a');
// 输出：
// b.js: {}
// a.js: { name: 'B' }
```

**解决方案**：

**1. 重构消除循环依赖**（最佳）

```javascript
// common.js
module.exports = { sharedData: {} };

// a.js
const common = require('./common');
module.exports = { name: 'A', common };

// b.js
const common = require('./common');
module.exports = { name: 'B', common };
```

**2. 延迟加载**

```javascript
// a.js
let b;
function getB() {
  if (!b) b = require('./b');
  return b;
}

module.exports = {
  name: 'A',
  getB
};

// b.js
let a;
function getA() {
  if (!a) a = require('./a');
  return a;
}

module.exports = {
  name: 'B',
  getA
};
```

**3. 使用依赖注入**

```javascript
// a.js
module.exports = function(deps) {
  return {
    name: 'A',
    b: deps.b
  };
};

// b.js
module.exports = function(deps) {
  return {
    name: 'B',
    a: deps.a
  };
};

// main.js
const aFactory = require('./a');
const bFactory = require('./b');

const a = aFactory({});
const b = bFactory({ a });
a.b = b;
```
</details>

### 5. package.json 中的 dependencies, devDependencies, peerDependencies 有什么区别？

<details>
<summary>点击查看答案</summary>

```json
{
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "eslint": "^8.0.0"
  },
  "peerDependencies": {
    "react": "^18.0.0"
  },
  "optionalDependencies": {
    "fsevents": "^2.3.0"
  }
}
```

**1. dependencies**：
- 运行时必需的依赖
- 会被安装到最终产品中
- 用户安装你的包时会自动安装这些依赖

**2. devDependencies**：
- 开发时使用的工具
- 不会被安装到生产环境
- 例如：测试框架、构建工具、代码检查工具

**3. peerDependencies**：
- 要求使用者提供的依赖
- 用于插件开发
- 例如：React 组件库要求用户自己安装 React

**4. optionalDependencies**：
- 可选依赖
- 安装失败不会导致整个安装过程失败
- 代码中需要处理依赖不存在的情况

**安装行为**：

```bash
# 只安装 dependencies
npm install --production

# 安装 dependencies + devDependencies
npm install

# 安装所有类型（包括 optional）
npm install --include=optional
```

**版本范围**：

```json
{
  "dependencies": {
    "package": "1.0.0",      // 精确版本
    "package": "^1.0.0",     // 兼容版本（1.x.x）
    "package": "~1.0.0",     // 补丁版本（1.0.x）
    "package": ">=1.0.0",    // 大于等于
    "package": "*",          // 任意版本（不推荐）
    "package": "latest"      // 最新版本（不推荐）
  }
}
```
</details>

## 最佳实践

### 1. 优先使用 ES Modules

```javascript
// ✅ 推荐（ESM）
import express from 'express';
export function handler() {}

// ❌ 旧方式（CJS）
const express = require('express');
module.exports = function handler() {}
```

### 2. 避免循环依赖

```javascript
// ✅ 好：清晰的依赖关系
// utils.js → models.js → services.js → controllers.js

// ❌ 不好：循环依赖
// a.js ↔ b.js
```

### 3. 正确使用 package.json 字段

```json
{
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./utils": {
      "import": "./dist/utils.mjs",
      "require": "./dist/utils.cjs"
    }
  },
  "engines": {
    "node": ">=14.0.0"
  }
}
```

### 4. 使用绝对路径导入

```javascript
// tsconfig.json 或 jsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/utils/*": ["src/utils/*"],
      "@/models/*": ["src/models/*"]
    }
  }
}

// 使用
import { helper } from '@/utils/helper';
// 而不是
import { helper } from '../../../utils/helper';
```

### 5. 模块应该是纯净的（无副作用）

```javascript
// ❌ 不好：有副作用
// config.js
const db = connectDatabase(); // 导入时就连接数据库
module.exports = { db };

// ✅ 好：延迟初始化
// config.js
let db = null;
function getDB() {
  if (!db) {
    db = connectDatabase();
  }
  return db;
}
module.exports = { getDB };
```

