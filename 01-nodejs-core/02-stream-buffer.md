# Stream & Buffer

## Buffer

### 什么是 Buffer？

Buffer 是 Node.js 中用于处理二进制数据的类，类似于数组，但专门用于存储字节数据。

```javascript
// 创建 Buffer 的几种方式

// 1. 从字符串创建
const buf1 = Buffer.from('Hello World');
console.log(buf1); // <Buffer 48 65 6c 6c 6f 20 57 6f 72 6c 64>

// 2. 创建指定大小的 Buffer（已弃用 new Buffer）
const buf2 = Buffer.alloc(10); // 创建 10 字节的 Buffer，默认填充 0
const buf3 = Buffer.allocUnsafe(10); // 更快，但内容未初始化

// 3. 从数组创建
const buf4 = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
console.log(buf4.toString()); // 'buffer'
```

### Buffer 的基本操作

```javascript
const buf = Buffer.from('Hello World');

// 1. 获取长度
console.log(buf.length); // 11

// 2. 读取字节
console.log(buf[0]); // 72 (H 的 ASCII 码)

// 3. 修改字节
buf[0] = 0x48;

// 4. 转换为字符串
console.log(buf.toString()); // 'Hello World'
console.log(buf.toString('hex')); // '48656c6c6f20576f726c64'
console.log(buf.toString('base64')); // 'SGVsbG8gV29ybGQ='

// 5. 切片
const slice = buf.slice(0, 5);
console.log(slice.toString()); // 'Hello'

// 6. 拷贝
const target = Buffer.alloc(11);
buf.copy(target);
console.log(target.toString()); // 'Hello World'

// 7. 拼接
const buf1 = Buffer.from('Hello ');
const buf2 = Buffer.from('World');
const result = Buffer.concat([buf1, buf2]);
console.log(result.toString()); // 'Hello World'
```

### Buffer 的字符编码

```javascript
// 支持的编码格式
const encodings = [
  'utf8',      // 默认，多字节编码的 Unicode 字符
  'utf16le',   // 2 或 4 字节，小端编码的 Unicode 字符
  'latin1',    // 单字节编码的字符串
  'base64',    // Base64 编码
  'hex',       // 十六进制编码
  'ascii',     // 7 位 ASCII 数据
  'binary'     // latin1 的别名
];

const buf = Buffer.from('你好世界');
console.log(buf.length); // 12 (UTF-8 中文字符占 3 字节)
console.log(buf.toString('utf8')); // '你好世界'
console.log(buf.toString('hex')); // 'e4bda0e5a5bde4b896e7958c'
```

### Buffer 性能优化

```javascript
// ❌ 不好：频繁创建 Buffer
function badExample() {
  let result = Buffer.alloc(0);
  for (let i = 0; i < 1000; i++) {
    result = Buffer.concat([result, Buffer.from('data')]);
  }
  return result;
}

// ✅ 好：预分配空间或使用数组
function goodExample() {
  const buffers = [];
  for (let i = 0; i < 1000; i++) {
    buffers.push(Buffer.from('data'));
  }
  return Buffer.concat(buffers);
}

// ✅ 更好：如果知道总大小，预分配
function betterExample() {
  const totalSize = 1000 * 4;
  const result = Buffer.alloc(totalSize);
  let offset = 0;
  
  for (let i = 0; i < 1000; i++) {
    const data = Buffer.from('data');
    data.copy(result, offset);
    offset += data.length;
  }
  
  return result;
}
```

## Stream

### 什么是 Stream？

Stream 是用于处理流式数据的抽象接口。它允许你以块（chunk）的方式处理数据，而不是一次性加载到内存中。

### Stream 的四种类型

```javascript
// 1. Readable Stream - 可读流
const fs = require('fs');
const readableStream = fs.createReadStream('input.txt');

// 2. Writable Stream - 可写流
const writableStream = fs.createWriteStream('output.txt');

// 3. Duplex Stream - 双工流（既可读又可写）
const net = require('net');
const socket = net.connect(8080); // TCP socket 是 duplex stream

// 4. Transform Stream - 转换流（读写过程中可以修改数据）
const { Transform } = require('stream');
const upperCaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});
```

### Readable Stream

```javascript
const fs = require('fs');

// 方式 1: 使用事件
const readStream = fs.createReadStream('bigfile.txt', {
  encoding: 'utf8',
  highWaterMark: 64 * 1024 // 64KB 的缓冲区
});

readStream.on('data', (chunk) => {
  console.log('Received chunk:', chunk.length);
});

readStream.on('end', () => {
  console.log('No more data');
});

readStream.on('error', (err) => {
  console.error('Error:', err);
});

// 方式 2: 暂停和恢复
readStream.pause(); // 暂停读取
setTimeout(() => {
  readStream.resume(); // 恢复读取
}, 1000);

// 方式 3: 使用异步迭代器 (Node.js 10+)
async function readFile() {
  const stream = fs.createReadStream('bigfile.txt', { encoding: 'utf8' });
  
  for await (const chunk of stream) {
    console.log('Chunk:', chunk.length);
  }
}
```

### Writable Stream

```javascript
const fs = require('fs');

const writeStream = fs.createWriteStream('output.txt');

// 写入数据
writeStream.write('First line\n');
writeStream.write('Second line\n');

// 结束写入
writeStream.end('Last line\n');

// 监听事件
writeStream.on('finish', () => {
  console.log('All data has been written');
});

writeStream.on('error', (err) => {
  console.error('Error:', err);
});

// 处理背压（Backpressure）
function writeMillionTimes(writer, data, encoding, callback) {
  let i = 1000000;
  
  write();
  
  function write() {
    let ok = true;
    
    while (i-- > 0 && ok) {
      // 如果返回 false，说明缓冲区已满
      ok = writer.write(data, encoding);
    }
    
    if (i > 0) {
      // 等待 drain 事件，表示缓冲区已清空
      writer.once('drain', write);
    } else {
      callback();
    }
  }
}
```

### Pipe（管道）

Pipe 是处理流的最简单方式，自动处理背压。

```javascript
const fs = require('fs');

// 简单的文件复制
fs.createReadStream('source.txt')
  .pipe(fs.createWriteStream('destination.txt'));

// 链式管道
const zlib = require('zlib');

fs.createReadStream('input.txt')
  .pipe(zlib.createGzip()) // 压缩
  .pipe(fs.createWriteStream('input.txt.gz'));

// 解压缩
fs.createReadStream('input.txt.gz')
  .pipe(zlib.createGunzip()) // 解压
  .pipe(fs.createWriteStream('output.txt'));

// 错误处理
const { pipeline } = require('stream');

pipeline(
  fs.createReadStream('source.txt'),
  zlib.createGzip(),
  fs.createWriteStream('source.txt.gz'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);
```

### Transform Stream

```javascript
const { Transform } = require('stream');

// 创建一个转换流
class UpperCaseTransform extends Transform {
  _transform(chunk, encoding, callback) {
    // 转换数据
    this.push(chunk.toString().toUpperCase());
    callback();
  }
}

// 使用
fs.createReadStream('input.txt')
  .pipe(new UpperCaseTransform())
  .pipe(fs.createWriteStream('output.txt'));

// 简化版本
const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

// CSV 解析示例
class CSVParser extends Transform {
  constructor(options) {
    super(options);
    this.buffer = '';
  }
  
  _transform(chunk, encoding, callback) {
    // 将新数据添加到缓冲区
    this.buffer += chunk.toString();
    
    // 按行分割
    const lines = this.buffer.split('\n');
    
    // 保留最后一行（可能不完整）
    this.buffer = lines.pop();
    
    // 处理完整的行
    lines.forEach(line => {
      if (line) {
        const row = line.split(',');
        this.push(JSON.stringify(row) + '\n');
      }
    });
    
    callback();
  }
  
  _flush(callback) {
    // 处理剩余的数据
    if (this.buffer) {
      const row = this.buffer.split(',');
      this.push(JSON.stringify(row) + '\n');
    }
    callback();
  }
}
```

### 自定义 Readable Stream

```javascript
const { Readable } = require('stream');

// 方式 1: 继承 Readable
class CounterStream extends Readable {
  constructor(max, options) {
    super(options);
    this.max = max;
    this.current = 0;
  }
  
  _read() {
    if (this.current < this.max) {
      this.push(`${this.current++}\n`);
    } else {
      this.push(null); // 表示流结束
    }
  }
}

const counter = new CounterStream(10);
counter.pipe(process.stdout);

// 方式 2: 使用简化的构造函数
const counter2 = new Readable({
  read() {
    if (this.currentCharCode > 90) {
      this.push(null);
      return;
    }
    
    this.push(String.fromCharCode(this.currentCharCode++));
  }
});
counter2.currentCharCode = 65; // A

// 方式 3: Readable.from (Node.js 12.3+)
const readableFromArray = Readable.from(['a', 'b', 'c']);
readableFromArray.pipe(process.stdout);

// 异步生成器
async function* generate() {
  yield 'Hello ';
  yield 'World';
}
const readableFromGenerator = Readable.from(generate());
```

### 自定义 Writable Stream

```javascript
const { Writable } = require('stream');

class ConsoleWriter extends Writable {
  _write(chunk, encoding, callback) {
    console.log(`Writing: ${chunk.toString()}`);
    callback();
  }
}

const writer = new ConsoleWriter();
writer.write('Hello ');
writer.write('World');
writer.end();

// 批量写入示例
class BatchWriter extends Writable {
  constructor(options) {
    super(options);
    this.batch = [];
    this.batchSize = 10;
  }
  
  _write(chunk, encoding, callback) {
    this.batch.push(chunk);
    
    if (this.batch.length >= this.batchSize) {
      this._flush(callback);
    } else {
      callback();
    }
  }
  
  _final(callback) {
    // 写入剩余的数据
    this._flush(callback);
  }
  
  _flush(callback) {
    console.log('Flushing batch:', this.batch.length);
    // 这里可以批量写入数据库等
    this.batch = [];
    callback();
  }
}
```

### Stream 的背压（Backpressure）

背压是流控制机制，防止数据生产速度超过消费速度。

```javascript
const fs = require('fs');

const readStream = fs.createReadStream('large-file.txt');
const writeStream = fs.createWriteStream('output.txt');

// ❌ 不好：没有处理背压
readStream.on('data', (chunk) => {
  writeStream.write(chunk);
});

// ✅ 好：使用 pipe 自动处理背压
readStream.pipe(writeStream);

// ✅ 手动处理背压
readStream.on('data', (chunk) => {
  const canContinue = writeStream.write(chunk);
  
  if (!canContinue) {
    // 缓冲区已满，暂停读取
    readStream.pause();
  }
});

writeStream.on('drain', () => {
  // 缓冲区已清空，恢复读取
  readStream.resume();
});
```

### Stream vs Buffer：何时使用？

```javascript
const fs = require('fs');

// ❌ 不好：对大文件使用 readFile（一次性加载到内存）
fs.readFile('large-video.mp4', (err, data) => {
  // 可能会耗尽内存
});

// ✅ 好：使用 Stream 处理大文件
const stream = fs.createReadStream('large-video.mp4');
stream.pipe(response); // 发送给 HTTP 响应

// 实际示例：HTTP 文件上传
const http = require('http');

http.createServer((req, res) => {
  if (req.method === 'POST') {
    // ✅ 使用 Stream 处理上传
    const writeStream = fs.createWriteStream('uploaded-file.dat');
    
    req.pipe(writeStream);
    
    req.on('end', () => {
      res.end('Upload complete');
    });
  }
}).listen(3000);
```

## 常见面试题

### 1. Buffer 和 Array 的区别？

<details>
<summary>点击查看答案</summary>

**主要区别**：

1. **内存分配**：
   - Buffer：在 V8 堆外分配内存
   - Array：在 V8 堆内分配内存

2. **存储内容**：
   - Buffer：只能存储字节（0-255）
   - Array：可以存储任何 JavaScript 值

3. **性能**：
   - Buffer：处理二进制数据更高效
   - Array：处理 JavaScript 对象更灵活

4. **大小**：
   - Buffer：创建后大小固定
   - Array：动态大小

```javascript
// Buffer
const buf = Buffer.alloc(10); // 固定 10 字节
buf[0] = 256; // 会被截断为 0（256 % 256）

// Array
const arr = []; // 动态大小
arr[0] = 256; // 可以存储任何值
```
</details>

### 2. Stream 相比一次性读取的优势是什么？

<details>
<summary>点击查看答案</summary>

**优势**：

1. **内存效率**：不需要一次性将整个文件加载到内存
2. **时间效率**：可以边读边处理，不需要等待全部读取完成
3. **可组合性**：通过 pipe 可以轻松组合多个操作

**示例对比**：

```javascript
// ❌ 一次性读取 1GB 文件
fs.readFile('1gb-file.txt', (err, data) => {
  // 需要 1GB 内存
  process(data);
});

// ✅ Stream 读取 1GB 文件
fs.createReadStream('1gb-file.txt')
  .pipe(processStream)
  .pipe(writeStream);
  // 只需要很小的缓冲区内存（默认 64KB）
```

**适用场景**：
- 处理大文件
- 网络传输
- 数据转换管道
- 实时数据处理
</details>

### 3. 如何处理 Stream 的错误？

<details>
<summary>点击查看答案</summary>

```javascript
const fs = require('fs');
const { pipeline } = require('stream');

// ❌ 不好：只监听一个流的错误
const read = fs.createReadStream('input.txt');
const write = fs.createWriteStream('output.txt');

read.on('error', handleError);
read.pipe(write);
// write 的错误没有处理！

// ✅ 好：监听所有流的错误
read.on('error', handleError);
write.on('error', handleError);
read.pipe(write);

// ✅ 更好：使用 pipeline
pipeline(
  fs.createReadStream('input.txt'),
  transformStream,
  fs.createWriteStream('output.txt'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Pipeline succeeded');
    }
  }
);

// ✅ 使用 finished 监听流完成
const { finished } = require('stream');

const stream = fs.createReadStream('file.txt');
finished(stream, (err) => {
  if (err) {
    console.error('Stream failed:', err);
  } else {
    console.log('Stream finished');
  }
});
```
</details>

### 4. 实现一个简单的 Stream 处理 CSV 文件

<details>
<summary>点击查看答案</summary>

```javascript
const { Transform } = require('stream');
const fs = require('fs');
const { pipeline } = require('stream');

// CSV 解析 Transform Stream
class CSVParser extends Transform {
  constructor(options) {
    super({ ...options, objectMode: true });
    this.header = null;
    this.buffer = '';
  }
  
  _transform(chunk, encoding, callback) {
    this.buffer += chunk.toString();
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop(); // 保留不完整的行
    
    lines.forEach(line => {
      if (!line.trim()) return;
      
      const values = line.split(',');
      
      if (!this.header) {
        // 第一行作为表头
        this.header = values;
      } else {
        // 将数据转换为对象
        const obj = {};
        this.header.forEach((key, index) => {
          obj[key.trim()] = values[index]?.trim();
        });
        this.push(obj);
      }
    });
    
    callback();
  }
  
  _flush(callback) {
    if (this.buffer && this.header) {
      const values = this.buffer.split(',');
      const obj = {};
      this.header.forEach((key, index) => {
        obj[key.trim()] = values[index]?.trim();
      });
      this.push(obj);
    }
    callback();
  }
}

// 过滤 Transform Stream
class Filter extends Transform {
  constructor(filterFn, options) {
    super({ ...options, objectMode: true });
    this.filterFn = filterFn;
  }
  
  _transform(obj, encoding, callback) {
    if (this.filterFn(obj)) {
      this.push(obj);
    }
    callback();
  }
}

// 使用
pipeline(
  fs.createReadStream('data.csv'),
  new CSVParser(),
  new Filter(row => parseInt(row.age) >= 18),
  new Transform({
    objectMode: true,
    transform(obj, encoding, callback) {
      this.push(JSON.stringify(obj) + '\n');
      callback();
    }
  }),
  fs.createWriteStream('filtered.json'),
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    } else {
      console.log('Processing complete');
    }
  }
);
```
</details>

### 5. 如何限制 Stream 的速度（流量控制）？

<details>
<summary>点击查看答案</summary>

```javascript
const { Transform } = require('stream');

// 限速 Transform Stream
class ThrottleTransform extends Transform {
  constructor(bytesPerSecond, options) {
    super(options);
    this.bytesPerSecond = bytesPerSecond;
    this.bytesWritten = 0;
    this.startTime = Date.now();
  }
  
  _transform(chunk, encoding, callback) {
    this.bytesWritten += chunk.length;
    
    const elapsedTime = (Date.now() - this.startTime) / 1000;
    const expectedTime = this.bytesWritten / this.bytesPerSecond;
    const delay = Math.max(0, (expectedTime - elapsedTime) * 1000);
    
    setTimeout(() => {
      this.push(chunk);
      callback();
    }, delay);
  }
}

// 使用：限制为 1MB/s
const fs = require('fs');

fs.createReadStream('large-file.txt')
  .pipe(new ThrottleTransform(1024 * 1024)) // 1MB/s
  .pipe(fs.createWriteStream('output.txt'));
```
</details>

## 最佳实践

### 1. 始终使用 pipeline 而不是 pipe

```javascript
const { pipeline } = require('stream');

// ✅ 推荐：自动处理错误和清理
pipeline(
  source,
  transform,
  destination,
  (err) => {
    if (err) {
      console.error('Pipeline failed:', err);
    }
  }
);

// ❌ 不推荐：需要手动处理每个流的错误
source
  .on('error', handleError)
  .pipe(transform)
  .on('error', handleError)
  .pipe(destination)
  .on('error', handleError);
```

### 2. 处理大文件时使用 Stream

```javascript
// ✅ 处理大文件
app.get('/download', (req, res) => {
  const stream = fs.createReadStream('large-file.zip');
  stream.pipe(res);
});

// ❌ 避免一次性读取
app.get('/download', (req, res) => {
  fs.readFile('large-file.zip', (err, data) => {
    res.send(data); // 可能内存溢出
  });
});
```

### 3. 使用 Buffer.from 而不是 new Buffer

```javascript
// ✅ 推荐
const buf = Buffer.from('hello');
const buf2 = Buffer.alloc(10);

// ❌ 已弃用（安全问题）
const buf = new Buffer('hello');
const buf2 = new Buffer(10);
```

### 4. 注意 Buffer 的切片不会复制数据

```javascript
const buf = Buffer.from('Hello World');
const slice = buf.slice(0, 5);

// 修改 slice 会影响原始 buffer
slice[0] = 0x48;
console.log(buf.toString()); // 'Hello World' (被修改)

// 如果需要独立的副本，使用 Buffer.from
const copy = Buffer.from(buf.slice(0, 5));
```

## 练习题

1. 实现一个 Transform Stream，将所有输入转换为大写
2. 实现一个限制并发的文件复制函数
3. 实现一个按行读取文件的 Stream
4. 实现一个计算文件 MD5 的 Stream
5. 实现一个批量处理的 Writable Stream

<details>
<summary>点击查看部分答案</summary>

```javascript
// 4. 计算文件 MD5 的 Stream
const crypto = require('crypto');
const fs = require('fs');

function calculateMD5(filePath) {
  return new Promise((resolve, reject) => {
    const hash = crypto.createHash('md5');
    const stream = fs.createReadStream(filePath);
    
    stream.on('data', (chunk) => {
      hash.update(chunk);
    });
    
    stream.on('end', () => {
      resolve(hash.digest('hex'));
    });
    
    stream.on('error', reject);
  });
}

// 使用
calculateMD5('file.txt').then(console.log);
```
</details>

