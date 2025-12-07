# go-zero AI 工具生态：让大模型成为你的 go-zero 专家

## 引言

在 AI 辅助编程时代，如何让大模型真正理解你使用的框架，生成符合框架规范的代码？go-zero 团队构建了一套完整的 AI 工具生态，包括 **ai-context**、**zero-skills** 和 **mcp-zero** 三个项目，让 Claude、GitHub Copilot、Cursor 等 AI 编程助手成为你的 go-zero 专家。

## 三大核心项目

### 1. ai-context：AI 的"工作手册"

**仓库**：https://github.com/zeromicro/ai-context

**定位**：简洁的指令层，告诉 AI "怎么做"

**内容**：
- 📋 工作流程（Workflows）：什么时候做什么
- 🔧 工具使用（Tools）：如何使用 mcp-zero 工具
- 📝 代码模式（Patterns）：快速参考的代码片段

**特点**：
- 文件小（约 5KB），加载快速
- 专注于工作流程和决策树
- 配合 GitHub Copilot 使用，放在 `.github/copilot-instructions.md`

**示例**：
```markdown
## Decision Tree

User Request →
├─ New API? → create_api_service → generate_api_from_spec
├─ New RPC? → create_rpc_service
├─ Database? → generate_model
└─ Modify? → Edit .api → generate_api_from_spec
```

### 2. zero-skills：AI 的"技术百科"

**仓库**：https://github.com/zeromicro/zero-skills

**定位**：详细的知识库，告诉 AI "为什么这样做"和"完整的实现"

**内容**：
- 🎯 详细模式（Patterns）：REST API、RPC、数据库、弹性保护
- 📚 最佳实践（Best Practices）：生产级代码规范
- 🔍 故障排查（Troubleshooting）：常见问题解决方案
- 🚀 快速开始（Getting Started）：从零到一的完整示例

**特点**：
- 内容详尽（约 40KB+），包含完整示例
- ✅ 正确做法 + ❌ 错误做法对比
- 适合深度学习和参考
- 为 AI 优化的结构化内容

**示例**：
```markdown
### ✅ Correct Pattern
func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    // 完整的实现代码
}

### ❌ Common Mistakes
// 不要把业务逻辑放在 handler 中
// 不要使用 fmt.Errorf 处理 API 错误
```

### 3. mcp-zero：AI 的"实时助手"

**仓库**：https://github.com/zeromicro/mcp-zero

**定位**：运行时接口，让 AI 能"调用工具"生成代码

**功能**：
- 🛠️ 代码生成工具：创建 API、RPC 服务，生成 Model
- 📊 项目分析：理解现有 go-zero 项目结构
- ✅ 输入验证：检查 API spec、protobuf 定义
- 📖 文档查询：访问 go-zero 文档和迁移指南

**特点**：
- 基于 Model Context Protocol (MCP)
- 与 Claude Desktop 深度集成
- 提供 10+ 个实用工具
- 安全的凭证处理

**示例工具**：
```json
{
  "name": "create_api_service",
  "description": "Creates a new go-zero API service",
  "parameters": {
    "service_name": "user-service",
    "port": 8080
  }
}
```

## 协同工作原理

### 架构图

```
┌─────────────────────────────────────────────────────────┐
│           AI 编程助手 (Claude/Copilot/Cursor)            │
└────────────┬────────────────────────────────────────────┘
             │
    ┌────────┴─────────┐
    │                  │
    ▼                  ▼
┌──────────┐      ┌──────────┐
│ai-context│      │ mcp-zero │
│          │      │          │
│ 工作指令  │      │ 工具调用  │
│ 快速参考  │      │ 代码生成  │
└────┬─────┘      └────┬─────┘
     │                 │
     │    ┌────────────┘
     │    │
     ▼    ▼
┌──────────────┐
│ zero-skills  │
│              │
│ 详细模式      │
│ 完整示例      │
│ 故障排查      │
└──────────────┘
```

### 工作流程

**场景 1：创建新的 REST API**

```
开发者：创建一个用户服务的 REST API

AI 助手：
1. 读取 ai-context → 了解应该用 create_api_service 工具
2. 调用 mcp-zero → 执行 create_api_service 生成项目结构
3. 参考 zero-skills → 获取 Handler/Logic/Context 三层架构的完整模式
4. 生成代码 → 符合 go-zero 规范的完整实现
```

**场景 2：遇到数据库问题**

```
开发者：我的数据库连接失败了，报错 "dial tcp: no such host"

AI 助手：
1. 读取 ai-context → 知道要查询文档或故障排查
2. 查询 zero-skills/troubleshooting → 找到数据库连接失败的章节
3. 返回解决方案 → 检查连接字符串格式、特殊字符转义等
```

**场景 3：实现中间件**

```
开发者：如何实现 JWT 认证中间件？

AI 助手：
1. 读取 ai-context/patterns.md → 获取中间件基本结构
2. 查询 zero-skills/patterns/rest-api-patterns.md → 获取完整的中间件模式
3. 生成代码 → 包含验证、错误处理、上下文传递的完整实现
```

## 实际使用指南

### 配置 GitHub Copilot

1. 在项目根目录创建 `.github/copilot-instructions.md`
2. 复制 ai-context 内容
3. 开始编码，Copilot 会遵循 go-zero 规范

```bash
# 克隆 ai-context
git clone https://github.com/zeromicro/ai-context.git

# 复制到项目
mkdir -p .github
cp ai-context/*.md .github/
```

### 配置 Claude Desktop + mcp-zero

1. 安装 mcp-zero：

```bash
git clone https://github.com/zeromicro/mcp-zero.git
cd mcp-zero
go build -o mcp-zero main.go
```

2. 配置 Claude Desktop（macOS）：

编辑 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "mcp-zero": {
      "command": "/path/to/mcp-zero",
      "env": {
        "GOCTL_PATH": "/Users/yourname/go/bin/goctl"
      }
    }
  }
}
```

3. 重启 Claude Desktop，开始使用：

```
你：创建一个名为 user-api 的 REST API 服务，端口 8080

Claude：好的，我将使用 mcp-zero 工具创建服务...
[调用 create_api_service 工具]
服务创建成功！接下来我将实现用户相关的 handler...
[参考 zero-skills 中的模式生成代码]
```

### 配置 Cursor

1. 在 Cursor 设置中添加项目规则
2. 引用 ai-context 和 zero-skills
3. 编码时 Cursor 会自动遵循规范

## 核心价值

### 1. **分层设计，各司其职**

- **ai-context**（5KB）：快速加载，立即响应
- **zero-skills**（40KB+）：深度学习，按需查询
- **mcp-zero**：工具执行，实时生成

### 2. **单一数据源**

- 模式和最佳实践统一维护在 zero-skills
- ai-context 和 mcp-zero 都引用它
- 更新一处，全局生效

### 3. **AI 优化的内容组织**

- 结构化的 Markdown，易于 AI 解析
- 明确的 ✅ 正确做法 vs ❌ 错误做法
- 完整的代码示例，可直接使用
- 丰富的上下文信息

### 4. **覆盖完整开发周期**

```
项目创建 → 代码生成 → 问题排查 → 性能优化
   ↓           ↓          ↓          ↓
mcp-zero  zero-skills  zero-skills  zero-skills
         ai-context   ai-context   ai-context
```

## 实际效果对比

### 没有 go-zero AI 工具生态

```
开发者：创建一个用户 API

AI：好的，这是基本的 HTTP handler...
[生成通用的 Go HTTP 代码，不符合 go-zero 规范]

开发者：这不是 go-zero 的写法，handler 应该调用 logic 层

AI：抱歉，这是修改后的代码...
[多轮对话才能得到正确的代码]
```

### 使用 go-zero AI 工具生态

```
开发者：创建一个用户 API

AI：好的，我将按照 go-zero 的三层架构创建...
[直接生成符合规范的 Handler → Logic → Model 代码]
[包含正确的错误处理、上下文传递、数据验证]

开发者：完美！✅
```

## 社区贡献

### 贡献到 zero-skills

zero-skills 欢迎社区贡献新的模式和示例：

1. Fork 仓库
2. 添加新的模式或改进现有模式
3. 确保示例完整且可运行
4. 提交 Pull Request

**贡献指南**：
- 包含 ✅ 正确做法和 ❌ 错误做法
- 提供完整的代码示例
- 添加使用场景说明
- 遵循现有的文档结构

### 改进 mcp-zero 工具

mcp-zero 工具持续演进，欢迎：

- 新增代码生成工具
- 改进现有工具的功能
- 添加更多验证规则
- 优化错误提示

## 未来计划

### 短期（1-3 个月）

- [ ] mcp-zero 集成 zero-skills 作为 MCP 资源
- [ ] 添加语义搜索功能
- [ ] 支持中英文双语
- [ ] 扩展更多实用工具

### 中期（3-6 个月）

- [ ] 支持项目模板市场
- [ ] 集成 AI 代码审查
- [ ] 添加性能分析工具
- [ ] 支持多版本 go-zero

### 长期愿景

- 让 AI 成为每个 go-zero 开发者的专属助手
- 降低 go-zero 学习曲线
- 提升团队开发效率
- 推动 AI 辅助编程在微服务领域的最佳实践

## 总结

go-zero AI 工具生态通过三个项目的协同配合，为开发者提供了完整的 AI 辅助开发体验：

- **ai-context** 让 AI 知道"做什么"
- **zero-skills** 让 AI 知道"怎么做好"
- **mcp-zero** 让 AI 能"真正动手做"

这不仅仅是文档或工具，而是一套专为 AI 时代设计的开发范式。无论你使用 Claude、GitHub Copilot 还是 Cursor，都能获得一致的、符合 go-zero 规范的代码生成体验。

**立即开始**：

1. ⭐ Star 三个项目
2. 🔧 配置你的 AI 编程助手
3. 🚀 体验 AI 驱动的 go-zero 开发
4. 💡 向社区分享你的经验

---

**相关链接**：

- go-zero 框架：https://github.com/zeromicro/go-zero
- ai-context：https://github.com/zeromicro/ai-context
- zero-skills：https://github.com/zeromicro/zero-skills
- mcp-zero：https://github.com/zeromicro/mcp-zero
- go-zero 文档：https://go-zero.dev

**加入社区**：

- 微信群：go-zero 仓库中文 readme 扫码加入 go-zero 开发者社区
