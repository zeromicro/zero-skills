# go-zero Skills - AI 助手的知识库

[English](README.md) | 简体中文

这是一个 [Agent Skill（智能体技能）](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)，包含为 AI 编程助手优化的 go-zero 框架知识和模式，帮助开发者更高效地构建微服务应用。

## 什么是 Skill？

Skills 是包含指令、脚本和资源的文件夹，AI 智能体可以动态发现和加载，以更好地完成特定任务。这个 skill 教会 AI 智能体如何生成生产级的 go-zero 微服务代码。

## 目标

本 skill 使 AI 助手（Claude、GitHub Copilot、Cursor 等）能够：
- 生成符合 go-zero 规范的准确代码
- 理解三层架构（Handler → Logic → Model）
- 应用微服务开发最佳实践
- 高效排查常见问题
- 构建生产就绪的应用

## Agent Skill 结构

遵循 [Agent Skills 规范](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md)：

```
zero-skills/
├── SKILL.md                    # 入口文件，包含 YAML 元数据
├── getting-started/            # 快速开始指南
├── references/                 # 详细模式文档
│   ├── rest-api-patterns.md    # REST API 开发模式
│   ├── rpc-patterns.md         # gRPC 服务模式
│   ├── database-patterns.md    # 数据库操作
│   └── resilience-patterns.md  # 弹性和容错
├── best-practices/             # 生产级建议
├── troubleshooting/            # 常见问题和解决方案
├── articles/                   # 深度指南
└── examples/                   # 演示项目和验证脚本
```

## 使用这个 Skill

### 在 Claude Desktop/Code 中使用

加载到 Claude 后，此 skill 会自动工作。完整指南请参阅 [SKILL.md](SKILL.md)。

### 在其他 AI 助手中使用

在 AI 上下文中引用这个 skill：
1. **GitHub Copilot**：使用 [ai-context](https://github.com/zeromicro/ai-context)，它会链接到这个 skill
2. **Cursor/Windsurf**：作为项目规则添加（参见 [AI 生态指南](articles/ai-ecosystem-guide.md)）
3. **API 使用**：在提示词中包含 `references/` 中的相关模式文件

## 特色功能

### ✅ 正确做法 vs ❌ 错误做法

每个模式都包含对比示例：

```go
// ✅ 正确：Handler 只处理 HTTP 相关逻辑
func (h *Handler) Handle(w http.ResponseWriter, r *http.Request) {
    var req types.Request
    if err := httpx.Parse(r, &req); err != nil {
        httpx.ErrorCtx(r.Context(), w, err)
        return
    }

    l := logic.NewLogic(r.Context(), h.svcCtx)
    resp, err := l.Process(&req)
    // ...
}

// ❌ 错误：不要在 Handler 中写业务逻辑
func (h *Handler) Handle(w http.ResponseWriter, r *http.Request) {
    // 直接查询数据库、处理业务逻辑等
    user, _ := h.svcCtx.UserModel.FindOne(ctx, id)
    // ...
}
```

### 📚 完整的代码示例

不是代码片段，而是可以直接运行的完整示例，包括：
- 完整的类型定义
- 错误处理
- 配置示例
- 测试代码

### 🔍 详细的故障排查

常见问题和解决方案，包括：
- 症状描述
- 根本原因
- 完整的解决步骤
- 预防措施

## 与 go-zero AI 生态的关系

zero-skills 是 go-zero AI 工具生态的一部分：

```
ai-context        → 简明的工作指令（给 GitHub Copilot）
zero-skills       → 详细的知识库（给所有 AI 助手）
mcp-zero          → 运行时工具调用（给 Claude Desktop）
```

详细说明参见：[go-zero AI 工具生态指南](articles/ai-ecosystem-guide.md)

## 内容示例

### REST API 模式

```go
// Handler 层 - 只处理 HTTP
func CreateUserHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req types.CreateUserRequest
        if err := httpx.Parse(r, &req); err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
            return
        }

        l := logic.NewCreateUserLogic(r.Context(), svcCtx)
        resp, err := l.CreateUser(&req)
        if err != nil {
            httpx.ErrorCtx(r.Context(), w, err)
        } else {
            httpx.OkJsonCtx(r.Context(), w, resp)
        }
    }
}

// Logic 层 - 业务逻辑实现
func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    // 验证
    if err := l.validateUser(req); err != nil {
        return nil, err
    }

    // 业务逻辑
    user := &model.User{
        Name:  req.Name,
        Email: req.Email,
    }

    // 数据库操作
    result, err := l.svcCtx.UserModel.Insert(l.ctx, user)
    if err != nil {
        return nil, err
    }

    userId, _ := result.LastInsertId()
    return &types.CreateUserResponse{Id: userId}, nil
}
```

## 贡献指南

欢迎贡献！请确保：
- 示例完整且经过测试
- 模式遵循官方 go-zero 约定
- 内容结构化，便于 AI 理解
- 包含正确和错误的示例对比

### 贡献内容

1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/new-pattern`
3. 提交更改：`git commit -am 'Add new pattern for XXX'`
4. 推送分支：`git push origin feature/new-pattern`
5. 提交 Pull Request

### 内容要求

- 使用清晰的标题和章节
- 提供完整的代码示例
- 说明使用场景
- 包含配置示例
- 添加故障排查提示

## 版本说明

当前版本：**v1.0.0**

兼容 go-zero 版本：
- v1.6.x ✅
- v1.5.x ✅

## 许可证

MIT License - 与 go-zero 框架相同

## 相关链接

- [go-zero 框架](https://github.com/zeromicro/go-zero)
- [go-zero 文档](https://go-zero.dev)
- [ai-context](https://github.com/zeromicro/ai-context) - GitHub Copilot 指令
- [mcp-zero](https://github.com/zeromicro/mcp-zero) - MCP 工具服务器

## 社区

- Discord: https://discord.gg/4JQvC5A4Fe
- 微信群：加入 go-zero 开发者社区
- GitHub Discussions: https://github.com/zeromicro/go-zero/discussions
