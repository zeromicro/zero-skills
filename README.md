# go-zero Skills for AI Agents

English | [简体中文](README_CN.md)

This is an [Agent Skill](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) containing structured knowledge and patterns for AI coding assistants to help developers work effectively with the [go-zero](https://github.com/zeromicro/go-zero) framework.

## What is a Skill?

Skills are folders of instructions, scripts, and resources that AI agents discover and load dynamically to perform better at specific tasks. This skill teaches AI agents how to generate production-ready go-zero microservices code.

## Purpose

This skill enables AI agents (Claude, GitHub Copilot, Cursor, etc.) to:
- Generate accurate go-zero code following framework conventions
- Understand the three-layer architecture (Handler → Logic → Model)
- Apply best practices for microservices development
- Troubleshoot common issues efficiently
- Build production-ready applications

## Agent Skill Structure

Following the [Agent Skills Spec](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md):

```
zero-skills/
├── SKILL.md                    # Entry point with YAML frontmatter
├── getting-started/            # Quick start guides
├── references/                 # Detailed pattern documentation
│   ├── rest-api-patterns.md    # REST API development patterns
│   ├── rpc-patterns.md         # gRPC service patterns
│   ├── database-patterns.md    # Database operations
│   └── resilience-patterns.md  # Resilience and fault tolerance
├── best-practices/             # Production recommendations
├── troubleshooting/            # Common issues and solutions
├── articles/                   # In-depth guides
└── examples/                   # Demo projects and verification scripts
```

## Using This Skill

### With Claude Desktop/Code

This skill works automatically when loaded into Claude. See [SKILL.md](SKILL.md) for the complete guide.

### With Other AI Assistants

Reference the skill in your AI context:
1. For **GitHub Copilot**: Use [ai-context](https://github.com/zeromicro/ai-context) which links to this skill
2. For **Cursor/Windsurf**: Add as project rules (see [AI Ecosystem Guide](articles/ai-ecosystem-guide.md))
3. For **API usage**: Include relevant pattern files from `references/` in your prompts

## Integration with go-zero AI Ecosystem

zero-skills is part of the go-zero AI tools ecosystem:

- **[ai-context](https://github.com/zeromicro/ai-context)** - Concise instructions for GitHub Copilot
- **zero-skills** (this repo) - Detailed knowledge base for all AI assistants
- **[mcp-zero](https://github.com/zeromicro/mcp-zero)** - Runtime tools for Claude Desktop

See [AI Ecosystem Guide](articles/ai-ecosystem-guide.md) for details.

## Contributing

Contributions are welcome! Please ensure:
- Examples are complete and tested
- Patterns follow official go-zero conventions
- Content is structured for AI consumption
- Include both correct and incorrect examples

## License

MIT License - Same as go-zero framework
