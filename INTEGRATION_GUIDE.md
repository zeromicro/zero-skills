# Integration with zero-skills

## For ai-context Repository

Add this section to `00-instructions.md`:

```markdown
## Detailed Patterns Reference

For complete implementation examples, refer to [zero-skills](https://github.com/zeromicro/zero-skills):

- **REST API** → [rest-api-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/patterns/rest-api-patterns.md)
  - Handler/Logic/Context architecture
  - Middleware patterns
  - Error handling

- **RPC Services** → [rpc-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/patterns/rpc-patterns.md)
  - Service discovery
  - gRPC interceptors
  - Streaming

- **Database** → [database-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/patterns/database-patterns.md)
  - CRUD operations
  - Transactions
  - Caching

- **Resilience** → [resilience-patterns.md](https://github.com/zeromicro/zero-skills/blob/main/patterns/resilience-patterns.md)
  - Circuit breaker
  - Rate limiting
  - Load shedding

- **Troubleshooting** → [common-issues.md](https://github.com/zeromicro/zero-skills/blob/main/troubleshooting/common-issues.md)
```

## For mcp-zero Repository

### Phase 1: Add Resource Support

Add to `main.go`:

```go
// Add zero-skills resources
func listResources() []Resource {
    return []Resource{
        {
            URI:         "zero-skills://patterns/rest-api",
            Name:        "REST API Patterns",
            Description: "Complete REST API implementation patterns with examples",
            MimeType:    "text/markdown",
        },
        {
            URI:         "zero-skills://patterns/rpc",
            Name:        "RPC Patterns",
            Description: "gRPC service patterns with service discovery",
            MimeType:    "text/markdown",
        },
        {
            URI:         "zero-skills://patterns/database",
            Name:        "Database Patterns",
            Description: "SQL, MongoDB, caching, and transaction patterns",
            MimeType:    "text/markdown",
        },
        {
            URI:         "zero-skills://patterns/resilience",
            Name:        "Resilience Patterns",
            Description: "Circuit breaker, rate limiting, load shedding",
            MimeType:    "text/markdown",
        },
        {
            URI:         "zero-skills://troubleshooting",
            Name:        "Troubleshooting Guide",
            Description: "Common issues and solutions",
            MimeType:    "text/markdown",
        },
    }
}

func readResource(uri string) (string, error) {
    // Option 1: Embed files at build time
    // Option 2: Fetch from GitHub at runtime
    // Option 3: Clone repo on first use and cache locally
    
    baseURL := "https://raw.githubusercontent.com/zeromicro/zero-skills/main/"
    
    switch uri {
    case "zero-skills://patterns/rest-api":
        return fetchFromGitHub(baseURL + "patterns/rest-api-patterns.md")
    case "zero-skills://patterns/rpc":
        return fetchFromGitHub(baseURL + "patterns/rpc-patterns.md")
    case "zero-skills://patterns/database":
        return fetchFromGitHub(baseURL + "patterns/database-patterns.md")
    case "zero-skills://patterns/resilience":
        return fetchFromGitHub(baseURL + "patterns/resilience-patterns.md")
    case "zero-skills://troubleshooting":
        return fetchFromGitHub(baseURL + "troubleshooting/common-issues.md")
    default:
        return "", fmt.Errorf("unknown resource: %s", uri)
    }
}
```

### Phase 2: Add Search Tools

Add new tools for querying zero-skills:

```go
{
    Name:        "search_pattern",
    Description: "Search zero-skills for specific go-zero implementation patterns",
    InputSchema: map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "query": {
                "type":        "string",
                "description": "What to search for (e.g., 'REST handler', 'RPC client', 'caching')",
            },
            "category": {
                "type":        "string",
                "enum":        []string{"rest", "rpc", "database", "resilience", "all"},
                "description": "Category to search in",
            },
        },
        "required": []string{"query"},
    },
}

{
    Name:        "get_pattern_example",
    Description: "Get complete working example from zero-skills",
    InputSchema: map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "pattern": {
                "type": "string",
                "enum": []string{
                    "rest_handler",
                    "rest_middleware",
                    "rpc_service",
                    "rpc_client",
                    "database_transaction",
                    "cache_pattern",
                    "circuit_breaker",
                    "rate_limiting",
                },
                "description": "Type of pattern to retrieve",
            },
        },
        "required": []string{"pattern"},
    },
}

{
    Name:        "troubleshoot_issue",
    Description: "Get solution for common go-zero issues from zero-skills",
    InputSchema: map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "problem": {
                "type":        "string",
                "description": "Description of the problem",
            },
            "error_message": {
                "type":        "string",
                "description": "Error message if available",
            },
        },
        "required": []string{"problem"},
    },
}
```

### Phase 3: Update Documentation

Add to `README.md`:

```markdown
## Resources

mcp-zero provides access to comprehensive go-zero patterns through [zero-skills](https://github.com/zeromicro/zero-skills):

- REST API patterns with complete examples
- RPC service patterns with service discovery
- Database operations and caching strategies
- Resilience patterns (circuit breaker, rate limiting)
- Troubleshooting guide for common issues

Access these via MCP resources protocol or search tools.
```

## Testing the Integration

### Test with Claude Desktop

1. Update Claude Desktop config to include mcp-zero
2. Ask: "Show me REST API patterns for go-zero"
3. Claude should query mcp-zero, which fetches from zero-skills
4. Response includes complete patterns with examples

### Example Queries

```
"How do I implement authentication middleware in go-zero?"
→ Searches zero-skills REST patterns for middleware examples

"What's the correct way to handle database transactions?"
→ Fetches database-patterns.md transaction section

"I'm getting a 404 error on my API endpoint"
→ Queries troubleshooting guide for 404 issues

"Show me RPC client configuration with etcd"
→ Returns RPC patterns with service discovery examples
```

## Benefits

1. **Layered Knowledge**
   - ai-context: High-level instructions (what/why)
   - zero-skills: Detailed patterns (how)
   - mcp-zero: Runtime interface (query)

2. **Single Source of Truth**
   - Patterns maintained in zero-skills
   - Both ai-context and mcp-zero reference it
   - Updates propagate automatically

3. **AI-Optimized**
   - Structured for easy parsing
   - Complete working examples
   - Common mistakes highlighted
   - Troubleshooting included

4. **User Experience**
   - Fast pattern lookup
   - Contextual examples
   - Issue resolution
   - Best practices

## Implementation Timeline

- **Week 1**: Add zero-skills reference to ai-context ✅
- **Week 2**: Implement MCP resources in mcp-zero
- **Week 3**: Add search/query tools
- **Week 4**: Test and refine integration
- **Ongoing**: Keep zero-skills synchronized with go-zero releases
