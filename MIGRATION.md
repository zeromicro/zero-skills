# Migration Guide: Agent Skills Refactor

## Summary

zero-skills has been refactored to follow the [Agent Skills Specification](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md) from Anthropic. This makes it a standardized skill that works seamlessly with Claude Desktop, Claude Code, and other AI agents.

## What Changed

### Directory Structure

**Before:**
```
zero-skills/
├── patterns/
│   ├── rest-api-patterns.md
│   ├── rpc-patterns.md
│   ├── database-patterns.md
│   └── resilience-patterns.md
├── getting-started/
├── best-practices/
└── troubleshooting/
```

**After:**
```
zero-skills/
├── SKILL.md                # NEW: Entry point with YAML frontmatter
├── references/             # RENAMED: patterns/ → references/
│   ├── rest-api-patterns.md
│   ├── rpc-patterns.md
│   ├── database-patterns.md
│   └── resilience-patterns.md
├── getting-started/
├── best-practices/
└── troubleshooting/
```

### New Files

- **SKILL.md**: Entry point with YAML frontmatter (name, description, license)
- Follows Agent Skills Spec v1.0 for standardization

### Breaking Changes

⚠️ **The `patterns/` directory has been renamed to `references/`**

If your projects reference the old path, update them:

```diff
- https://github.com/zeromicro/zero-skills/blob/main/patterns/rest-api-patterns.md
+ https://github.com/zeromicro/zero-skills/blob/main/references/rest-api-patterns.md
```

## Migration Steps

### For ai-context References

If you reference zero-skills in ai-context or custom instructions:

**Update links:**
```markdown
<!-- Old -->
- REST API → [rest-api-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/patterns/rest-api-patterns.md)

<!-- New -->
- REST API → [rest-api-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/references/rest-api-patterns.md)
```

### For mcp-zero Integration

No changes needed! mcp-zero will be updated to reference the new paths automatically.

### For Custom Prompts/Instructions

Update any hardcoded references from `patterns/` to `references/`:

```bash
# Find all references to patterns/
grep -r "patterns/" .cursorrules/ .github/

# Replace with references/
sed -i '' 's|patterns/|references/|g' .cursorrules/*.md
```

### For Direct GitHub URLs

Update URLs in documentation or scripts:

```bash
# Before
curl https://raw.githubusercontent.com/zeromicro/zero-skills/main/patterns/rest-api-patterns.md

# After
curl https://raw.githubusercontent.com/zeromicro/zero-skills/main/references/rest-api-patterns.md
```

## Benefits of This Change

### 1. **Standardization**
- Follows Anthropic's Agent Skills Spec v1.0
- Works automatically with Claude Desktop/Code
- Discoverable by AI agents

### 2. **Better Organization**
- Clear entry point (SKILL.md) with metadata
- Progressive disclosure pattern
- References loaded only when needed

### 3. **Improved AI Agent Support**
- Standardized YAML frontmatter for discoverability
- Optimized context usage
- Better integration with skill-aware AI systems

### 4. **Future-Proof**
- Compatible with emerging AI agent standards
- Ready for skill marketplaces
- Extensible structure

## Rollback (If Needed)

If you need the old structure:

```bash
# Checkout the commit before the refactor
git checkout <commit-before-refactor>

# Or create a branch from the old structure
git checkout -b legacy-structure <commit-before-refactor>
```

The last commit before the refactor: `4373c88`

## Support

If you encounter issues during migration:

1. Check [SKILL.md](SKILL.md) for the new structure
2. See [README.md](README.md) for updated usage instructions
3. Open an issue at [zeromicro/zero-skills/issues](https://github.com/zeromicro/zero-skills/issues)

## Timeline

- **Released**: December 9, 2025
- **Deprecation**: `patterns/` directory removed
- **Support**: Old URLs will 404 - update references immediately

## FAQ

### Q: Why was this change necessary?

A: Following the Agent Skills Spec provides:
- Better discoverability by AI agents
- Standard format for emerging AI ecosystems
- Improved context efficiency
- Future compatibility

### Q: Do I need to update my code?

A: Only if you have hardcoded references to `patterns/` directory. Most integrations through ai-context or mcp-zero will be updated automatically.

### Q: Will old bookmarks/links work?

A: No. GitHub will return 404 for `patterns/` references. Update your bookmarks to `references/`.

### Q: Is content the same?

A: Yes! All pattern files are identical. Only the directory structure changed.

## Related Links

- [Agent Skills Spec](https://github.com/anthropics/skills/blob/main/spec/agent-skills-spec.md)
- [SKILL.md](SKILL.md) - New entry point
- [README.md](README.md) - Updated documentation
- [AI Ecosystem Guide](articles/ai-ecosystem-guide.md) - Integration guide
