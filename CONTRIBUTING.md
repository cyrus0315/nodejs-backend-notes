# 贡献指南

感谢你对本项目的关注！欢迎任何形式的贡献。

## 🤝 如何贡献

### 报告问题

如果你发现了问题或有建议：

1. 搜索 [Issues](https://github.com/cyrus0315/nodejs-backend-notes/issues) 确认问题是否已存在
2. 如果没有，创建一个新的 Issue
3. 清晰地描述问题或建议
4. 如果是 bug，请提供复现步骤

### 提交代码

1. **Fork 项目**
   ```bash
   # 点击 GitHub 页面上的 Fork 按钮
   ```

2. **Clone 到本地**
   ```bash
   git clone git@github.com:你的用户名/nodejs-backend-notes.git
   cd nodejs-backend-notes
   ```

3. **创建分支**
   ```bash
   git checkout -b feature/your-feature-name
   # 或
   git checkout -b fix/your-bug-fix
   ```

4. **进行修改**
   - 保持代码风格一致
   - 添加必要的注释
   - 确保内容准确、易懂

5. **提交更改**
   ```bash
   git add .
   git commit -m "feat: 添加 XXX 内容"
   # 或
   git commit -m "fix: 修复 XXX 问题"
   ```

6. **推送到 GitHub**
   ```bash
   git push origin feature/your-feature-name
   ```

7. **创建 Pull Request**
   - 在 GitHub 上创建 PR
   - 清晰描述你的修改
   - 等待 review

## 📝 内容贡献规范

### 文档格式

- 使用清晰的 Markdown 格式
- 代码示例要完整可运行
- 添加必要的注释说明
- 保持中文表达准确、通俗易懂

### 代码示例

```javascript
// ✅ 好的示例：有注释、完整、清晰
const express = require('express');
const app = express();

// 基础路由示例
app.get('/api/users', (req, res) => {
  res.json({ users: [] });
});

app.listen(3000, () => {
  console.log('Server started on port 3000');
});
```

```javascript
// ❌ 不好的示例：不完整、没有注释
app.get('/api/users', (req, res) => {
  // ...
});
```

### 内容质量

- ✅ 基于实际经验和最佳实践
- ✅ 提供实用的代码示例
- ✅ 说明使用场景和注意事项
- ✅ 包含常见问题和解决方案
- ❌ 避免过时的内容
- ❌ 避免未经验证的内容
- ❌ 避免纯理论无实践价值的内容

## 🎯 贡献方向

欢迎在以下方面贡献：

- 📚 **补充内容**：添加新的知识点、最佳实践
- 🐛 **修复错误**：纠正文档中的错误或过时内容
- 💡 **优化示例**：改进代码示例，使其更清晰易懂
- 🌐 **翻译**：帮助翻译英文版本
- 📖 **完善文档**：改进排版、添加图表等
- ⚡️ **性能优化**：提供性能优化的实践案例
- 🔒 **安全建议**：分享安全相关的经验

## 💬 讨论交流

- 有想法或建议？在 [Discussions](https://github.com/cyrus0315/nodejs-backend-notes/discussions) 发起讨论
- 遇到问题？在 [Issues](https://github.com/cyrus0315/nodejs-backend-notes/issues) 提问
- 想法不成熟也没关系，欢迎讨论！

## 📋 Commit 规范

建议使用以下格式：

- `feat: 添加 XXX 功能`
- `fix: 修复 XXX 问题`
- `docs: 更新 XXX 文档`
- `style: 格式调整`
- `refactor: 重构 XXX`
- `test: 添加测试`
- `chore: 构建/工具链相关`

## ⚖️ 许可协议

贡献的内容将采用 [MIT License](./LICENSE) 协议开源。

---

再次感谢你的贡献！🎉

