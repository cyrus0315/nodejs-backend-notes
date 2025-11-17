# Event Loop & 异步编程

## Event Loop 工作原理

### 什么是 Event Loop？

Event Loop（事件循环）是 Node.js 实现异步非阻塞 I/O 的核心机制。它允许 Node.js 在单线程环境下执行非阻塞操作。

### Event Loop 的六个阶段

```
   ┌───────────────────────────┐
┌─>│           timers          │  执行 setTimeout/setInterval 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  执行延迟到下一个循环的 I/O 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  内部使用
│  └─────────────┬─────────────┘      ┌───────────────┐
│  ┌─────────────┴─────────────┐      │   incoming:   │
│  │           poll            │<─────┤  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  执行 setImmediate 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  执行 close 事件回调
   └───────────────────────────┘
```

### 1. Timers 阶段
执行 `setTimeout()` 和 `setInterval()` 的回调函数。

```javascript
setTimeout(() => {
  console.log('timeout');
}, 0);
```

### 2. Pending Callbacks 阶段
执行某些系统操作的回调，如 TCP 错误。

### 3. Idle, Prepare 阶段
仅供 Node.js 内部使用。

### 4. Poll 阶段
**最重要的阶段**，检索新的 I/O 事件，执行 I/O 回调。

- 如果 poll 队列不为空，会遍历执行回调队列
- 如果 poll 队列为空：
  - 如果有 `setImmediate()` 回调，会结束 poll 阶段进入 check 阶段
  - 如果没有 `setImmediate()`，会等待新的回调加入并立即执行

### 5. Check 阶段
执行 `setImmediate()` 的回调。

```javascript
setImmediate(() => {
  console.log('immediate');
});
```

### 6. Close Callbacks 阶段
执行关闭事件的回调，如 `socket.on('close', ...)`。

## 微任务 vs 宏任务

### 微任务（Microtasks）
- `process.nextTick()`
- `Promise.then/catch/finally`
- `queueMicrotask()`

### 宏任务（Macrotasks）
- `setTimeout`
- `setInterval`
- `setImmediate`
- I/O 操作

### 执行顺序

**关键规则**：
1. 同步代码最先执行
2. 每个宏任务执行完后，会清空所有微任务
3. `process.nextTick()` 优先级高于 Promise 微任务

```javascript
console.log('1: 同步代码');

setTimeout(() => {
  console.log('2: setTimeout');
}, 0);

setImmediate(() => {
  console.log('3: setImmediate');
});

Promise.resolve().then(() => {
  console.log('4: Promise');
});

process.nextTick(() => {
  console.log('5: nextTick');
});

console.log('6: 同步代码');

// 输出顺序：
// 1: 同步代码
// 6: 同步代码
// 5: nextTick
// 4: Promise
// 2: setTimeout
// 3: setImmediate
```

## process.nextTick vs setImmediate

### process.nextTick()
- 在**当前操作完成后**立即执行，在 Event Loop 继续之前
- 优先级最高，会阻塞 Event Loop
- 不属于 Event Loop 的任何阶段

```javascript
process.nextTick(() => {
  console.log('nextTick');
});
```

### setImmediate()
- 在 Event Loop 的 **check 阶段**执行
- 不会阻塞 Event Loop

```javascript
setImmediate(() => {
  console.log('immediate');
});
```

### 实际对比

```javascript
// 在 I/O 循环内
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  
  setImmediate(() => {
    console.log('immediate');
  });
});

// 输出：
// immediate
// timeout
// 原因：在 I/O 循环内，setImmediate 总是先于 setTimeout 执行
```

```javascript
// 不在 I/O 循环内
setTimeout(() => {
  console.log('timeout');
}, 0);

setImmediate(() => {
  console.log('immediate');
});

// 输出顺序不确定，取决于进程性能
// 可能是 timeout -> immediate
// 也可能是 immediate -> timeout
```

## 异步编程模式

### 1. Callback（回调）

```javascript
const fs = require('fs');

fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(data);
});
```

**缺点**：回调地狱（Callback Hell）

```javascript
fs.readFile('file1.txt', (err, data1) => {
  if (err) throw err;
  fs.readFile('file2.txt', (err, data2) => {
    if (err) throw err;
    fs.readFile('file3.txt', (err, data3) => {
      if (err) throw err;
      console.log(data1, data2, data3);
    });
  });
});
```

### 2. Promise

```javascript
const fs = require('fs').promises;

fs.readFile('file.txt', 'utf8')
  .then(data => {
    console.log(data);
  })
  .catch(err => {
    console.error(err);
  });
```

**链式调用**：

```javascript
fs.readFile('file1.txt', 'utf8')
  .then(data1 => {
    console.log(data1);
    return fs.readFile('file2.txt', 'utf8');
  })
  .then(data2 => {
    console.log(data2);
    return fs.readFile('file3.txt', 'utf8');
  })
  .then(data3 => {
    console.log(data3);
  })
  .catch(err => {
    console.error(err);
  });
```

### 3. Async/Await

```javascript
const fs = require('fs').promises;

async function readFiles() {
  try {
    const data1 = await fs.readFile('file1.txt', 'utf8');
    console.log(data1);
    
    const data2 = await fs.readFile('file2.txt', 'utf8');
    console.log(data2);
    
    const data3 = await fs.readFile('file3.txt', 'utf8');
    console.log(data3);
  } catch (err) {
    console.error(err);
  }
}

readFiles();
```

**并行执行**：

```javascript
async function readFilesParallel() {
  try {
    const [data1, data2, data3] = await Promise.all([
      fs.readFile('file1.txt', 'utf8'),
      fs.readFile('file2.txt', 'utf8'),
      fs.readFile('file3.txt', 'utf8')
    ]);
    
    console.log(data1, data2, data3);
  } catch (err) {
    console.error(err);
  }
}
```

## 异步错误处理

### Promise 错误处理

```javascript
// 方式 1: .catch()
promise
  .then(result => {
    // 处理结果
  })
  .catch(error => {
    // 处理错误
  });

// 方式 2: .then() 的第二个参数
promise.then(
  result => {
    // 处理结果
  },
  error => {
    // 处理错误
  }
);
```

### Async/Await 错误处理

```javascript
async function example() {
  try {
    const result = await someAsyncOperation();
    return result;
  } catch (error) {
    console.error('Error:', error);
    throw error; // 重新抛出或处理
  }
}
```

### 未捕获的 Promise Rejection

```javascript
// 全局捕获未处理的 Promise rejection
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // 应用程序特定的日志记录、错误处理等
});

// 示例：忘记捕获的 Promise
Promise.reject(new Error('Oops!')); // 会触发 unhandledRejection
```

## 常见面试题

### 1. 说明以下代码的输出顺序

```javascript
console.log('1');

setTimeout(() => {
  console.log('2');
  process.nextTick(() => {
    console.log('3');
  });
  new Promise(resolve => {
    console.log('4');
    resolve();
  }).then(() => {
    console.log('5');
  });
});

process.nextTick(() => {
  console.log('6');
});

new Promise(resolve => {
  console.log('7');
  resolve();
}).then(() => {
  console.log('8');
});

setTimeout(() => {
  console.log('9');
  process.nextTick(() => {
    console.log('10');
  });
  new Promise(resolve => {
    console.log('11');
    resolve();
  }).then(() => {
    console.log('12');
  });
});

console.log('13');
```

<details>
<summary>点击查看答案</summary>

**输出顺序**：1, 7, 13, 6, 8, 2, 4, 9, 11, 3, 5, 10, 12

**解析**：
1. 同步代码：1, 7, 13
2. 微任务（第一轮）：6（nextTick）, 8（Promise）
3. 第一个 setTimeout 宏任务：2, 4（Promise 构造函数同步）
4. 第二个 setTimeout 宏任务：9, 11（Promise 构造函数同步）
5. 第一个 setTimeout 的微任务：3（nextTick）, 5（Promise）
6. 第二个 setTimeout 的微任务：10（nextTick）, 12（Promise）
</details>

### 2. process.nextTick 和 setImmediate 的区别？

<details>
<summary>点击查看答案</summary>

**区别**：

1. **执行时机**：
   - `process.nextTick()`: 在当前操作完成后立即执行，在 Event Loop 继续之前
   - `setImmediate()`: 在 Event Loop 的 check 阶段执行

2. **优先级**：
   - `process.nextTick()` 优先级更高
   - `process.nextTick()` 可能会阻塞 Event Loop

3. **使用场景**：
   - `process.nextTick()`: 适合在当前操作完成后立即执行的任务
   - `setImmediate()`: 适合在 I/O 事件后执行的任务

**命名争议**：实际上这两个 API 的命名是反直觉的，`nextTick` 听起来应该更晚执行，但实际上它最快。
</details>

### 3. 如何避免 process.nextTick 阻塞 Event Loop？

<details>
<summary>点击查看答案</summary>

**问题**：过度使用 `process.nextTick()` 会导致 Event Loop 饥饿，I/O 永远得不到处理。

```javascript
// ❌ 错误示例：会阻塞 Event Loop
let count = 0;
function recursiveNextTick() {
  if (count < 1000000) {
    count++;
    process.nextTick(recursiveNextTick);
  }
}
recursiveNextTick();
```

**解决方案**：

1. 使用 `setImmediate()` 替代
```javascript
// ✅ 正确示例
let count = 0;
function recursiveImmediate() {
  if (count < 1000000) {
    count++;
    setImmediate(recursiveImmediate);
  }
}
recursiveImmediate();
```

2. 限制 `process.nextTick()` 的使用次数
3. 使用 `Promise` 或 `async/await`
</details>

### 4. Promise.all vs Promise.race vs Promise.allSettled

<details>
<summary>点击查看答案</summary>

```javascript
// Promise.all - 所有 Promise 都成功才成功
Promise.all([promise1, promise2, promise3])
  .then(results => {
    // results 是所有 Promise 结果的数组
  })
  .catch(error => {
    // 任何一个 Promise 失败就会进入这里
  });

// Promise.race - 第一个完成的 Promise 决定结果
Promise.race([promise1, promise2, promise3])
  .then(result => {
    // result 是第一个完成的 Promise 的结果
  });

// Promise.allSettled - 等待所有 Promise 完成（无论成功还是失败）
Promise.allSettled([promise1, promise2, promise3])
  .then(results => {
    // results 是所有 Promise 的结果
    // [{ status: 'fulfilled', value: ... }, { status: 'rejected', reason: ... }]
  });

// Promise.any - 第一个成功的 Promise 决定结果
Promise.any([promise1, promise2, promise3])
  .then(result => {
    // result 是第一个成功的 Promise 的结果
  })
  .catch(error => {
    // 所有 Promise 都失败才会进入这里
  });
```
</details>

### 5. 如何实现 Promise.all？

<details>
<summary>点击查看答案</summary>

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('promises must be an array'));
    }
    
    const results = [];
    let completedCount = 0;
    
    if (promises.length === 0) {
      return resolve(results);
    }
    
    promises.forEach((promise, index) => {
      // 确保每个元素都是 Promise
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completedCount++;
          
          if (completedCount === promises.length) {
            resolve(results);
          }
        })
        .catch(error => {
          reject(error);
        });
    });
  });
}

// 测试
promiseAll([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3)
]).then(console.log); // [1, 2, 3]
```
</details>

## 最佳实践

### 1. 优先使用 Async/Await
```javascript
// ✅ 推荐
async function getData() {
  try {
    const result = await fetchData();
    return result;
  } catch (error) {
    handleError(error);
  }
}

// ❌ 不推荐（除非需要并行执行）
function getData() {
  return fetchData()
    .then(result => result)
    .catch(error => handleError(error));
}
```

### 2. 使用 Promise.all 实现并行
```javascript
// ✅ 并行执行，速度快
const [user, posts, comments] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchComments()
]);

// ❌ 串行执行，速度慢
const user = await fetchUser();
const posts = await fetchPosts();
const comments = await fetchComments();
```

### 3. 避免在循环中使用 await
```javascript
// ❌ 不好：串行执行
for (const id of ids) {
  const result = await processItem(id);
  results.push(result);
}

// ✅ 好：并行执行
const results = await Promise.all(
  ids.map(id => processItem(id))
);
```

### 4. 始终处理错误
```javascript
// ✅ 推荐
async function safeOperation() {
  try {
    return await riskyOperation();
  } catch (error) {
    logger.error('Operation failed:', error);
    throw error; // 或者返回默认值
  }
}

// 全局错误处理
process.on('uncaughtException', (error) => {
  logger.error('Uncaught Exception:', error);
  process.exit(1);
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection:', reason);
});
```

### 5. 使用 util.promisify 转换回调函数
```javascript
const util = require('util');
const fs = require('fs');

// 转换为 Promise
const readFile = util.promisify(fs.readFile);

async function readMyFile() {
  const data = await readFile('file.txt', 'utf8');
  return data;
}
```

## 练习题

1. 实现一个 `sleep` 函数
2. 实现一个带超时的 Promise
3. 实现 Promise 的串行执行
4. 实现 Promise 的并发控制（最多同时执行 N 个）
5. 实现 Promise.retry（失败后重试）

<details>
<summary>点击查看答案</summary>

```javascript
// 1. sleep 函数
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// 使用
await sleep(1000);
console.log('1 秒后执行');

// 2. 带超时的 Promise
function promiseWithTimeout(promise, timeout) {
  return Promise.race([
    promise,
    new Promise((_, reject) => 
      setTimeout(() => reject(new Error('Timeout')), timeout)
    )
  ]);
}

// 3. Promise 串行执行
async function serial(promises) {
  const results = [];
  for (const promiseFunc of promises) {
    results.push(await promiseFunc());
  }
  return results;
}

// 4. Promise 并发控制
async function promiseLimit(promises, limit) {
  const results = [];
  const executing = [];
  
  for (const promise of promises) {
    const p = Promise.resolve().then(() => promise());
    results.push(p);
    
    if (limit <= promises.length) {
      const e = p.then(() => executing.splice(executing.indexOf(e), 1));
      executing.push(e);
      
      if (executing.length >= limit) {
        await Promise.race(executing);
      }
    }
  }
  
  return Promise.all(results);
}

// 5. Promise.retry
async function retry(fn, times = 3, delay = 1000) {
  try {
    return await fn();
  } catch (error) {
    if (times === 1) throw error;
    await sleep(delay);
    return retry(fn, times - 1, delay);
  }
}
```
</details>

