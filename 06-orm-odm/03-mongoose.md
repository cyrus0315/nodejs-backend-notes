# Mongoose

Mongoose 是 MongoDB 的 ODM（Object Document Mapper），提供了基于 Schema 的模型定义和丰富的查询 API。

## Schema 定义

### 基础 Schema

```typescript
import mongoose, { Schema, Document } from 'mongoose';

interface IUser extends Document {
  email: string;
  name: string;
  age?: number;
  role: 'user' | 'admin';
  isActive: boolean;
  tags: string[];
  metadata?: Map<string, any>;
  createdAt: Date;
  updatedAt: Date;
}

const UserSchema = new Schema<IUser>({
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    lowercase: true,
    trim: true,
    match: [/^\S+@\S+\.\S+$/, 'Invalid email format']
  },
  name: {
    type: String,
    required: true,
    minlength: [2, 'Name too short'],
    maxlength: [50, 'Name too long']
  },
  age: {
    type: Number,
    min: [0, 'Age cannot be negative'],
    max: [150, 'Age cannot exceed 150']
  },
  role: {
    type: String,
    enum: ['user', 'admin'],
    default: 'user'
  },
  isActive: {
    type: Boolean,
    default: true
  },
  tags: [{ type: String }],
  metadata: {
    type: Map,
    of: Schema.Types.Mixed
  }
}, {
  timestamps: true,  // 自动添加 createdAt, updatedAt
  collection: 'users'  // 自定义集合名
});

export const User = mongoose.model<IUser>('User', UserSchema);
```

### 数据类型

```typescript
const ExampleSchema = new Schema({
  // 基本类型
  stringField: String,
  numberField: Number,
  booleanField: Boolean,
  dateField: Date,
  bufferField: Buffer,
  
  // 特殊类型
  objectIdField: Schema.Types.ObjectId,
  mixedField: Schema.Types.Mixed,  // 任意类型
  decimalField: Schema.Types.Decimal128,
  
  // 数组
  arrayField: [String],
  arrayOfObjects: [{
    name: String,
    value: Number
  }],
  
  // 嵌套对象
  nestedObject: {
    field1: String,
    field2: Number
  },
  
  // Map
  mapField: {
    type: Map,
    of: String
  },
  
  // 枚举
  enumField: {
    type: String,
    enum: ['option1', 'option2', 'option3']
  },
  
  // 默认值
  defaultField: {
    type: String,
    default: 'default value'
  },
  defaultFunction: {
    type: Date,
    default: Date.now
  },
  
  // 必填
  requiredField: {
    type: String,
    required: true
  },
  
  // 唯一
  uniqueField: {
    type: String,
    unique: true
  },
  
  // 索引
  indexedField: {
    type: String,
    index: true
  },
  
  // 稀疏索引
  sparseField: {
    type: String,
    sparse: true
  },
  
  // 选择/忽略
  selectField: {
    type: String,
    select: false  // 默认不查询
  }
});
```

## 关系和引用

### 引用（Population）

```typescript
// 用户 Schema
const UserSchema = new Schema({
  name: String,
  email: String
});

// 文章 Schema
const PostSchema = new Schema({
  title: String,
  content: String,
  author: {
    type: Schema.Types.ObjectId,
    ref: 'User',  // 引用 User 模型
    required: true
  },
  comments: [{
    type: Schema.Types.ObjectId,
    ref: 'Comment'
  }]
});

const Post = mongoose.model('Post', PostSchema);

// 查询并填充
const posts = await Post.find()
  .populate('author')  // 填充作者信息
  .populate('comments');  // 填充评论

// 选择性填充
const posts = await Post.find()
  .populate('author', 'name email')  // 只填充 name 和 email
  .populate({
    path: 'comments',
    select: 'content createdAt',
    options: { limit: 5, sort: { createdAt: -1 } }
  });

// 深度填充
const posts = await Post.find()
  .populate({
    path: 'comments',
    populate: {
      path: 'author',
      select: 'name'
    }
  });

// 动态引用
const CommentSchema = new Schema({
  content: String,
  commentableType: {
    type: String,
    enum: ['Post', 'Video']
  },
  commentable: {
    type: Schema.Types.ObjectId,
    refPath: 'commentableType'  // 动态引用
  }
});
```

### 虚拟填充

```typescript
// 用户 Schema
const UserSchema = new Schema({
  name: String,
  email: String
});

// 虚拟属性：用户的文章
UserSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author'
});

const User = mongoose.model('User', UserSchema);

// 使用虚拟填充
const user = await User.findById(userId)
  .populate('posts');

console.log(user.posts);  // 用户的所有文章
```

### 嵌入文档

```typescript
// 地址 Schema
const AddressSchema = new Schema({
  street: String,
  city: String,
  state: String,
  zipCode: String
}, { _id: false });  // 不生成 _id

// 用户 Schema（嵌入地址）
const UserSchema = new Schema({
  name: String,
  email: String,
  address: AddressSchema,  // 嵌入单个地址
  addresses: [AddressSchema]  // 嵌入地址数组
});

// 使用
const user = new User({
  name: 'John',
  email: 'john@example.com',
  address: {
    street: '123 Main St',
    city: 'New York',
    state: 'NY',
    zipCode: '10001'
  }
});

await user.save();
```

## CRUD 操作

### Create

```typescript
// 方式 1：new + save
const user = new User({
  name: 'John Doe',
  email: 'john@example.com'
});
await user.save();

// 方式 2：create
const user = await User.create({
  name: 'John Doe',
  email: 'john@example.com'
});

// 批量创建
const users = await User.insertMany([
  { name: 'User 1', email: 'user1@example.com' },
  { name: 'User 2', email: 'user2@example.com' }
]);
```

### Read

```typescript
// 查找所有
const users = await User.find();

// 条件查询
const users = await User.find({
  isActive: true,
  age: { $gte: 18 }
});

// 查找一个
const user = await User.findOne({ email: 'john@example.com' });
const user = await User.findById('507f1f77bcf86cd799439011');

// 选择字段
const users = await User.find()
  .select('name email')
  .select('-password');  // 排除密码

// 排序
const users = await User.find()
  .sort({ createdAt: -1 });  // 降序
  .sort('name -age');  // name 升序，age 降序

// 限制和跳过
const users = await User.find()
  .limit(10)
  .skip(20);

// 链式调用
const users = await User
  .find({ isActive: true })
  .select('name email')
  .sort({ createdAt: -1 })
  .limit(10)
  .skip(0)
  .populate('posts')
  .exec();

// 计数
const count = await User.countDocuments({ isActive: true });
const total = await User.estimatedDocumentCount();  // 更快，但不精确

// 判断存在
const exists = await User.exists({ email: 'john@example.com' });
```

### Update

```typescript
// 查找并更新
const user = await User.findByIdAndUpdate(
  userId,
  { name: 'Updated Name' },
  { 
    new: true,  // 返回更新后的文档
    runValidators: true  // 运行验证器
  }
);

const user = await User.findOneAndUpdate(
  { email: 'john@example.com' },
  { $set: { isActive: false } },
  { new: true }
);

// 更新多个
const result = await User.updateMany(
  { isActive: false },
  { $set: { deletedAt: new Date() } }
);

// 更新操作符
await User.updateOne(
  { _id: userId },
  {
    $set: { name: 'New Name' },
    $inc: { loginCount: 1 },
    $push: { tags: 'new-tag' },
    $pull: { tags: 'old-tag' },
    $addToSet: { tags: 'unique-tag' },
    $unset: { temporaryField: '' }
  }
);
```

### Delete

```typescript
// 删除一个
await User.findByIdAndDelete(userId);
await User.findOneAndDelete({ email: 'john@example.com' });

// 删除多个
await User.deleteMany({ isActive: false });

// 使用实例方法
const user = await User.findById(userId);
await user.remove();  // 触发中间件
```

## 查询操作符

```typescript
// 比较操作符
const users = await User.find({
  age: { $gt: 18, $lt: 65 },  // 大于18且小于65
  name: { $ne: 'Admin' },  // 不等于
  role: { $in: ['user', 'moderator'] },  // 在列表中
  status: { $nin: ['banned', 'deleted'] }  // 不在列表中
});

// 逻辑操作符
const users = await User.find({
  $and: [
    { isActive: true },
    { age: { $gte: 18 } }
  ]
});

const users = await User.find({
  $or: [
    { role: 'admin' },
    { role: 'moderator' }
  ]
});

const users = await User.find({
  age: { $not: { $lt: 18 } }
});

// 元素操作符
const users = await User.find({
  age: { $exists: true },  // 字段存在
  nickname: { $type: 'string' }  // 字段类型
});

// 数组操作符
const posts = await Post.find({
  tags: { $all: ['mongodb', 'nodejs'] },  // 包含所有
  tags: { $size: 3 },  // 数组长度为3
  'comments.0': { $exists: true }  // 至少有一个评论
});

// 正则表达式
const users = await User.find({
  name: /^John/i,  // 以John开头，不区分大小写
  email: { $regex: '@example\\.com$', $options: 'i' }
});

// 文本搜索
const posts = await Post.find({
  $text: { $search: 'mongodb tutorial' }
});
```

## 聚合管道

```typescript
// 基础聚合
const result = await User.aggregate([
  // 匹配
  { $match: { isActive: true } },
  
  // 分组
  { $group: {
    _id: '$role',
    count: { $sum: 1 },
    avgAge: { $avg: '$age' }
  }},
  
  // 排序
  { $sort: { count: -1 } },
  
  // 限制
  { $limit: 10 }
]);

// 复杂聚合
const result = await Order.aggregate([
  // 1. 匹配条件
  { $match: {
    createdAt: { $gte: new Date('2023-01-01') }
  }},
  
  // 2. 关联查询
  { $lookup: {
    from: 'users',
    localField: 'userId',
    foreignField: '_id',
    as: 'user'
  }},
  
  // 3. 展开数组
  { $unwind: '$user' },
  
  // 4. 投影
  { $project: {
    orderId: '$_id',
    userName: '$user.name',
    total: 1,
    itemCount: { $size: '$items' }
  }},
  
  // 5. 分组统计
  { $group: {
    _id: '$userName',
    orderCount: { $sum: 1 },
    totalAmount: { $sum: '$total' },
    avgAmount: { $avg: '$total' }
  }},
  
  // 6. 排序
  { $sort: { totalAmount: -1 } },
  
  // 7. 分页
  { $skip: 0 },
  { $limit: 20 }
]);

// 分面聚合
const result = await Product.aggregate([
  { $match: { category: 'Electronics' } },
  { $facet: {
    // 分组统计
    byBrand: [
      { $group: { _id: '$brand', count: { $sum: 1 } }},
      { $sort: { count: -1 } }
    ],
    // 价格范围
    priceRanges: [
      { $bucket: {
        groupBy: '$price',
        boundaries: [0, 100, 500, 1000, 5000],
        default: 'other'
      }}
    ],
    // 总体统计
    stats: [
      { $group: {
        _id: null,
        total: { $sum: 1 },
        avgPrice: { $avg: '$price' },
        minPrice: { $min: '$price' },
        maxPrice: { $max: '$price' }
      }}
    ]
  }}
]);
```

## 中间件（Hooks）

```typescript
const UserSchema = new Schema({
  email: String,
  password: String,
  loginCount: Number
});

// pre 中间件（异步）
UserSchema.pre('save', async function(next) {
  // this 指向当前文档
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// post 中间件
UserSchema.post('save', function(doc, next) {
  console.log('User saved:', doc.email);
  next();
});

// pre 查询中间件
UserSchema.pre('find', function() {
  // this 指向查询对象
  this.where({ isActive: true });
});

UserSchema.pre('findOne', function() {
  this.select('-password');  // 自动排除密码
});

// pre 删除中间件
UserSchema.pre('remove', async function(next) {
  // 删除用户前，删除其所有文章
  await Post.deleteMany({ author: this._id });
  next();
});

// pre 更新中间件
UserSchema.pre('findOneAndUpdate', function(next) {
  this.set({ updatedAt: new Date() });
  next();
});

// post 错误处理中间件
UserSchema.post('save', function(error, doc, next) {
  if (error.name === 'MongoServerError' && error.code === 11000) {
    next(new Error('Email already exists'));
  } else {
    next(error);
  }
});
```

## 虚拟属性

```typescript
const UserSchema = new Schema({
  firstName: String,
  lastName: String,
  email: String
});

// 虚拟属性（不存储在数据库）
UserSchema.virtual('fullName')
  .get(function() {
    return `${this.firstName} ${this.lastName}`;
  })
  .set(function(name) {
    const parts = name.split(' ');
    this.firstName = parts[0];
    this.lastName = parts[1];
  });

// 使用
const user = new User({
  firstName: 'John',
  lastName: 'Doe'
});

console.log(user.fullName);  // "John Doe"

user.fullName = 'Jane Smith';
console.log(user.firstName);  // "Jane"
console.log(user.lastName);  // "Smith"

// 虚拟属性也可以用于查询
UserSchema.virtual('posts', {
  ref: 'Post',
  localField: '_id',
  foreignField: 'author'
});

// 查询时填充虚拟属性
const user = await User.findById(userId)
  .populate('posts');
```

## 实例方法和静态方法

```typescript
const UserSchema = new Schema({
  email: String,
  password: String
});

// 实例方法
UserSchema.methods.comparePassword = async function(candidatePassword: string): Promise<boolean> {
  return bcrypt.compare(candidatePassword, this.password);
};

UserSchema.methods.generateToken = function(): string {
  return jwt.sign({ id: this._id }, process.env.JWT_SECRET);
};

// 静态方法
UserSchema.statics.findByEmail = function(email: string) {
  return this.findOne({ email });
};

UserSchema.statics.findActive = function() {
  return this.find({ isActive: true });
};

// 查询助手
UserSchema.query.byRole = function(role: string) {
  return this.where({ role });
};

const User = mongoose.model('User', UserSchema);

// 使用实例方法
const user = await User.findById(userId);
const isValid = await user.comparePassword('password123');
const token = user.generateToken();

// 使用静态方法
const user = await User.findByEmail('john@example.com');
const activeUsers = await User.findActive();

// 使用查询助手
const admins = await User.find().byRole('admin');
```

## 索引

```typescript
// Schema 级别索引
const UserSchema = new Schema({
  email: { type: String, unique: true, index: true },
  name: { type: String, index: true },
  location: {
    type: { type: String, default: 'Point' },
    coordinates: [Number]
  }
});

// 复合索引
UserSchema.index({ firstName: 1, lastName: 1 });

// 文本索引
UserSchema.index({ bio: 'text', interests: 'text' });

// 地理空间索引
UserSchema.index({ location: '2dsphere' });

// TTL 索引（自动过期）
UserSchema.index({ createdAt: 1 }, { expireAfterSeconds: 86400 });

// 稀疏索引
UserSchema.index({ referralCode: 1 }, { sparse: true });

// 唯一索引
UserSchema.index({ email: 1 }, { unique: true });

// 部分索引
UserSchema.index(
  { email: 1 },
  { 
    partialFilterExpression: { 
      isActive: true 
    }
  }
);

// 创建索引
await User.createIndexes();

// 查看索引
await User.collection.getIndexes();

// 删除索引
await User.collection.dropIndex('email_1');
```

## 事务

```typescript
// 方式 1：使用 session
const session = await mongoose.startSession();

try {
  await session.withTransaction(async () => {
    const user = await User.create([{
      name: 'John',
      email: 'john@example.com'
    }], { session });
    
    await Post.create([{
      title: 'First Post',
      author: user[0]._id
    }], { session });
  });
} finally {
  await session.endSession();
}

// 方式 2：手动控制
const session = await mongoose.startSession();
session.startTransaction();

try {
  const user = await User.create([{
    name: 'John',
    email: 'john@example.com'
  }], { session });
  
  await Post.create([{
    title: 'First Post',
    author: user[0]._id
  }], { session });
  
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  await session.endSession();
}
```

## 与 NestJS 集成

```typescript
// app.module.ts
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost/mydb', {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      autoIndex: true
    }),
    MongooseModule.forFeature([
      { name: User.name, schema: UserSchema }
    ])
  ]
})
export class AppModule {}

// user.schema.ts
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type UserDocument = User & Document;

@Schema({ timestamps: true })
export class User {
  @Prop({ required: true, unique: true })
  email: string;
  
  @Prop({ required: true })
  name: string;
  
  @Prop({ default: true })
  isActive: boolean;
}

export const UserSchema = SchemaFactory.createForClass(User);

// user.service.ts
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';

@Injectable()
export class UserService {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>
  ) {}
  
  async findAll(): Promise<User[]> {
    return this.userModel.find().exec();
  }
  
  async create(data: Partial<User>): Promise<User> {
    const user = new this.userModel(data);
    return user.save();
  }
}
```

## 常见面试题

### 1. Mongoose 的 populate 原理？

<details>
<summary>点击查看答案</summary>

**原理**：
1. 第一次查询：获取主文档
2. 提取引用字段的 ObjectId
3. 第二次查询：根据 ObjectId 查询关联文档
4. 将关联文档填充到主文档

**性能考虑**：
- 会产生额外查询（类似 JOIN）
- 可能导致 N+1 问题
- 大量数据时性能下降

**优化方案**：
```typescript
// ❌ N+1 问题
const posts = await Post.find();
for (const post of posts) {
  await post.populate('author');
}

// ✅ 一次性填充
const posts = await Post.find().populate('author');

// ✅ 选择性填充
const posts = await Post.find()
  .populate('author', 'name email');

// ✅ 使用聚合管道（更高效）
const posts = await Post.aggregate([
  {
    $lookup: {
      from: 'users',
      localField: 'author',
      foreignField: '_id',
      as: 'author'
    }
  },
  { $unwind: '$author' }
]);
```
</details>

### 2. 嵌入文档 vs 引用文档如何选择？

<details>
<summary>点击查看答案</summary>

**嵌入文档**：
```typescript
const UserSchema = new Schema({
  name: String,
  address: {
    street: String,
    city: String
  }
});
```

优点：
- 一次查询获取所有数据
- 更好的性能
- 原子性更新

缺点：
- 文档大小限制（16MB）
- 数据重复
- 难以独立查询

**引用文档**：
```typescript
const UserSchema = new Schema({
  name: String,
  addressId: { type: ObjectId, ref: 'Address' }
});
```

优点：
- 避免数据重复
- 灵活的查询
- 无大小限制

缺点：
- 需要多次查询
- 性能较差

**选择建议**：
- 一对少量（1-10）且不频繁变化 → 嵌入
- 一对大量或频繁变化 → 引用
- 数据需要独立访问 → 引用
- 数据总是一起访问 → 嵌入
</details>

### 3. Mongoose 中间件的执行顺序？

<details>
<summary>点击查看答案</summary>

```typescript
UserSchema.pre('save', function(next) {
  console.log('Pre save 1');
  next();
});

UserSchema.pre('save', function(next) {
  console.log('Pre save 2');
  next();
});

UserSchema.post('save', function(doc, next) {
  console.log('Post save 1');
  next();
});

UserSchema.post('save', function(doc, next) {
  console.log('Post save 2');
  next();
});

// 执行顺序：
// Pre save 1
// Pre save 2
// （实际保存操作）
// Post save 1
// Post save 2
```

**类型**：
- **pre**: 操作前执行
- **post**: 操作后执行

**支持的钩子**：
- `save`, `remove`, `update`, `updateOne`, `deleteOne`
- `find`, `findOne`, `findOneAndUpdate`, `findOneAndRemove`
- `validate`, `init`
</details>

## 最佳实践

1. **使用 Schema 验证确保数据完整性**
2. **为频繁查询字段添加索引**
3. **合理使用嵌入 vs 引用**
4. **避免过深的 populate**
5. **使用聚合管道处理复杂查询**
6. **启用连接池优化性能**
7. **使用 lean() 提高只读查询性能**
8. **敏感数据使用 select: false**
9. **使用中间件实现通用逻辑**
10. **定期监控和优化慢查询**

