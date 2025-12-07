# Examples

This directory contains example scripts and demo code for zero-skills tutorials.

## demo-project/

完整的 GitHub Copilot + go-zero 演示项目。自动创建配置了 ai-context 的 go-zero 项目，用于验证 AI 辅助开发效果。

**快速开始：**

```bash
cd demo-project
./setup-demo.sh
```

**详细文档：** [demo-project/README.md](demo-project/README.md)

**包含功能：**
- ✅ 自动配置 GitHub Copilot（ai-context submodule）
- ✅ 创建完整的 go-zero REST API 项目
- ✅ 提供多个测试场景验证 Copilot 效果
- ✅ 包含验证脚本确认配置正确

## verify-tutorial.sh

验证 AI 工具生态配置教程的完整性和正确性。

**功能：**
- ✅ 测试 GitHub Copilot 配置（submodule + 符号链接）
- ✅ 测试 Cursor 配置（.cursorrules）
- ✅ 测试 Windsurf 配置（.windsurfrules）
- ✅ 测试 submodule 更新功能
- ✅ 验证 ai-context 内容结构
- ✅ 验证 zero-skills 模式引用

**使用方法：**

```bash
# 运行验证脚本
./examples/verify-tutorial.sh

# 或者使用绝对路径
bash /path/to/zero-skills/examples/verify-tutorial.sh
```

**测试内容：**

1. **GitHub Copilot 配置测试**
   - 添加 ai-context 为 submodule 到 `.github/ai-context`
   - 创建符号链接到 `.github/copilot-instructions.md`
   - 验证文件内容包含 go-zero 相关内容

2. **Cursor 配置测试**
   - 添加 ai-context 为 submodule 到 `.cursorrules`
   - 验证目录结构和 markdown 文件
   - 确认内容正确加载

3. **Windsurf 配置测试**
   - 添加 ai-context 为 submodule 到 `.windsurfrules`
   - 验证目录结构和 markdown 文件
   - 确认内容正确加载

4. **Submodule 更新测试**
   - 执行 `git submodule update --remote`
   - 验证更新功能正常工作

5. **内容结构验证**
   - 检查 ai-context 包含必需的章节
   - 确认文档结构完整

6. **zero-skills 引用验证**
   - 验证所有模式文档链接存在
   - 确认引用正确指向 zero-skills 仓库

**输出示例：**

```
================================================
零技能 AI 工具生态配置验证脚本
Zero-Skills AI Ecosystem Tutorial Verification
================================================

✓ 创建测试目录: /tmp/zero-skills-demo-12345

=== 测试 1: GitHub Copilot 配置 ===
✓ PASS: GitHub Copilot 配置
  Submodule 添加成功，符号链接创建成功，内容验证通过

[更多测试输出...]

================================================
测试总结 / Test Summary
================================================
总测试数 / Total Tests: 6
通过 / Passed: 6
失败 / Failed: 0

✓ 所有测试通过！教程验证成功！
```

**清理测试目录：**

脚本会在 `/tmp` 下创建临时测试目录，测试完成后可以删除：

```bash
# 脚本会输出清理命令，例如：
rm -rf /tmp/zero-skills-demo-12345
```

**注意事项：**

- 脚本需要 Git 环境
- 需要网络连接以克隆 ai-context 仓库
- macOS/Linux 系统可直接运行
- Windows 用户建议在 Git Bash 或 WSL 中运行

## 贡献

欢迎添加更多示例脚本！请确保：

1. 脚本包含清晰的注释
2. 提供使用说明
3. 包含错误处理
4. 验证功能正确性
