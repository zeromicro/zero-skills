---
name: zero-skills
description: |
  Comprehensive knowledge base for go-zero microservices framework.

  **Use this skill when:**
  - Building REST APIs with go-zero (Handler → Logic → Model architecture)
  - Creating RPC services with service discovery and load balancing
  - Implementing database operations with sqlx, MongoDB, or Redis caching
  - Adding resilience patterns (circuit breaker, rate limiting, load shedding)
  - Implementing distributed transactions with DTM (SAGA, TCC patterns)
  - Setting up observability (Prometheus, Jaeger, ELK)
  - Processing async tasks with message queues (go-queue, Kafka)
  - Using advanced components (Bloom filter, MapReduce, TimingWheel)
  - Troubleshooting go-zero issues or understanding framework conventions
  - Generating production-ready microservices code

  **Features:**
  - Complete pattern guides with ✅ correct and ❌ incorrect examples
  - Three-layer architecture enforcement
  - Production best practices
  - Distributed transaction patterns
  - Observability and monitoring setup
  - Common pitfall solutions
license: MIT
allowed-tools:
  - Read
  - Grep
  - Glob
---

# go-zero Skills for AI Agents

This skill provides comprehensive go-zero microservices framework knowledge, optimized for AI agents helping developers build production-ready services. It covers REST APIs, RPC services, database operations, resilience patterns, and troubleshooting.

## 🎯 When to Use This Skill

Invoke this skill when working with go-zero:
- **Creating services**: REST APIs, gRPC services, or microservices architectures
- **Database integration**: SQL, MongoDB, Redis, or connection pooling
- **Production hardening**: Circuit breakers, rate limiting, or error handling
- **Debugging**: Understanding errors, fixing configuration, or resolving issues
- **Learning**: Understanding go-zero patterns and best practices

## 📚 Knowledge Structure

This skill organizes go-zero knowledge into focused modules. **Load specific guides as needed** rather than reading everything at once:

### Quick Start Guide
**Link**: [Official go-zero Documentation](https://go-zero.dev/docs/quick-start)
**Contains**: Installation, first API service, basic commands, hello-world examples (refer to official docs)

### Pattern Guides (Detailed Reference)

#### 1. REST API Patterns
**File**: [references/rest-api-patterns.md](references/rest-api-patterns.md)
**When to load**: Creating HTTP endpoints, implementing CRUD operations, adding middleware
**Contains**:
- Handler → Logic → Context three-layer architecture
- Request/response handling with proper types
- Middleware (auth, logging, metrics, CORS)
- Error handling with `httpx.Error()` and `httpx.OkJson()`
- Complete CRUD examples with ✅ correct vs ❌ incorrect patterns

#### 2. RPC Service Patterns
**File**: [references/rpc-patterns.md](references/rpc-patterns.md)
**When to load**: Building gRPC services, service-to-service communication
**Contains**:
- Protocol Buffers definition and code generation
- Service discovery with etcd/consul/kubernetes
- Load balancing strategies
- Client configuration and interceptors
- Error handling in RPC contexts

#### 3. Database Patterns
**File**: [references/database-patterns.md](references/database-patterns.md)
**When to load**: Implementing data persistence, caching, or complex queries
**Contains**:
- SQL operations with sqlx (CRUD, transactions, batch inserts)
- MongoDB integration patterns
- Redis caching strategies and cache-aside pattern
- Model generation with `goctl model`
- Connection pooling and performance tuning

#### 4. Resilience Patterns
**File**: [references/resilience-patterns.md](references/resilience-patterns.md)
**When to load**: Production hardening, handling failures, managing system load
**Contains**:
- Circuit breaker configuration (Breaker)
- Rate limiting and API throttling
- Load shedding under pressure
- Timeout and retry strategies
- Graceful shutdown and degradation

#### 5. goctl Command Reference
**File**: [references/goctl-commands.md](references/goctl-commands.md)
**When to load**: Generating code with goctl, setting up new services, post-generation steps
**Contains**:
- goctl installation and detection
- API/RPC/Model generation commands with exact flags
- Post-generation pipeline (mod tidy, import fixing, build verification)
- Config templates (API, RPC, production)
- Deployment templates (Dockerfile, Kubernetes, Docker Compose)
- Middleware and error handler templates
- API spec patterns (CRUD, JWT, mixed auth)

#### 6. Distributed Transaction Patterns
**File**: [references/distributed-transactions.md](references/distributed-transactions.md)
**When to load**: Cross-service data consistency, DTM integration, SAGA/TCC patterns
**Contains**:
- DTM integration with go-zero
- SAGA pattern for long-running transactions
- TCC pattern for financial operations
- 2-Phase message for DB + cache consistency
- Barrier pattern for idempotency
- Configuration and error handling

#### 7. Observability Patterns
**File**: [references/observability.md](references/observability.md)
**When to load**: Production monitoring, tracing, alerting setup
**Contains**:
- Prometheus metrics configuration
- Custom metrics implementation
- Distributed tracing with OpenTelemetry/Jaeger
- Structured logging with logx
- Grafana dashboards and alerting rules
- ELK integration for log aggregation

#### 8. Message Queue Patterns
**File**: [references/message-queue.md](references/message-queue.md)
**When to load**: Async processing, delayed tasks, event streaming
**Contains**:
- go-queue dq (Beanstalkd) for delayed tasks
- go-queue kq (Kafka) for high-throughput messaging
- Producer and consumer patterns
- Delay queue and retry patterns
- Configuration and error handling

#### 9. Advanced Components
**File**: [references/advanced-components.md](references/advanced-components.md)
**When to load**: Performance optimization, concurrent processing, caching
**Contains**:
- Bloom filter for cache penetration prevention
- TimingWheel for delayed task scheduling
- MapReduce for parallel processing
- Executors for batch task buffering
- SharedCalls (SingleFlight) for duplicate prevention

### Supporting Resources

#### Best Practices
**File**: [best-practices/overview.md](best-practices/overview.md)
**When to load**: Production deployment, code review, optimization
**Contains**: Configuration management, logging, monitoring, security, performance

#### Troubleshooting
**File**: [troubleshooting/common-issues.md](troubleshooting/common-issues.md)
**When to load**: Debugging errors, configuration issues, runtime problems
**Contains**: Common error messages, solutions, configuration pitfalls, debugging tips

#### Claude Code Integration
**File**: [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md)
**When to load**: Setting up Claude Code for zero-skills usage
**Contains**: Installation, invocation methods, advanced features (subagents, dynamic context)

#### Design Principles
**File**: [design/principles.md](design/principles.md)
**When to load**: Understanding framework philosophy, architecture decisions
**Contains**: Three-layer architecture rationale, design decisions, performance considerations, anti-patterns

#### Case Study: go-zero-looklook
**File**: [examples/case-studies/looklook-overview.md](examples/case-studies/looklook-overview.md)
**When to load**: Learning from production-scale example, real-world architecture
**Contains**: Large-scale microservice architecture, service breakdown, key patterns, deployment setup

## 🚀 Common Workflows

These workflows guide you through typical go-zero development tasks:

### Creating a New REST API Service

**Steps:**
1. Define API specification in `.api` file with types and routes
2. Generate code: `goctl api go -api user.api -dir .`
3. Implement business logic in `internal/logic/` layer
4. Add validation and error handling with `httpx`
5. Test endpoints with proper request/response handling

**Detailed guide**: [references/rest-api-patterns.md](references/rest-api-patterns.md#complete-rest-api-workflow)

### Implementing Database Operations

**Steps:**
1. Design database schema and create tables
2. Generate model: `goctl model mysql datasource -url="..." -table="users" -dir="./model"`
3. Inject model into ServiceContext in `internal/svc/service_context.go`
4. Use sqlx methods in logic layer (`Insert`, `FindOne`, `Update`, `Delete`)
5. Handle transactions and errors properly with `ctx` propagation

**Detailed guide**: [references/database-patterns.md](references/database-patterns.md#crud-operations)

### Adding Middleware

**Steps:**
1. Create middleware function in `internal/middleware/` directory
2. Define middleware in `.api` file or register programmatically
3. Implement authentication/authorization logic
4. Pass validated data through `r.Context()`
5. Handle errors with appropriate HTTP status codes

**Detailed guide**: [references/rest-api-patterns.md](references/rest-api-patterns.md#middleware-patterns)

### Building an RPC Service

**Steps:**
1. Define service in `.proto` file with messages and RPCs
2. Generate code: `goctl rpc protoc user.proto --go_out=. --go-grpc_out=. --zrpc_out=.`
3. Implement service logic in `internal/logic/`
4. Configure service discovery (etcd/consul/kubernetes)
5. Test with RPC client and handle errors

**Detailed guide**: [references/rpc-patterns.md](references/rpc-patterns.md#complete-rpc-workflow)

## ⚡ Key Principles

When generating or reviewing go-zero code, always apply these principles:

### ✅ Always Follow

- **Three-layer separation**: Keep Handler (routing) → Logic (business) → Model (data) distinct
- **Structured errors**: Use `httpx.Error(w, err)` for HTTP errors, not `fmt.Errorf`
- **Configuration**: Load with `conf.MustLoad(&c, *configFile)` and inject via ServiceContext
- **Context propagation**: Pass `ctx context.Context` through all layers for tracing and cancellation
- **Type safety**: Define request/response types in `.api` files, generate with goctl
- **goctl generation**: Always use `goctl` to generate boilerplate, never hand-write handlers/routes

### ❌ Never Do

- Put business logic directly in handlers (violates three-layer architecture)
- Return raw errors with `w.Write()` or `fmt.Fprintf()` instead of using httpx helpers
- Hard-code configuration values (ports, hosts, database credentials)
- Skip validation of user inputs or forget to check `err != nil`
- Modify generated code (customize via `logic` layer instead)
- Bypass ServiceContext injection (leads to tight coupling and testing issues)

## 📖 Progressive Learning Path

Follow this path based on your needs:

### 🟢 New to go-zero?

1. **Start here**: [Official go-zero Quick Start](https://go-zero.dev/docs/quick-start)
   Install go-zero, create your first API, understand basic concepts

   Connect to MySQL/PostgreSQL, generate models, implement CRUD

### 🟡 Building production services?

1. **Review best practices**: [best-practices/overview.md](best-practices/overview.md)
   Configuration, logging, monitoring, security checklist

2. **Add resilience**: [references/resilience-patterns.md](references/resilience-patterns.md)
   Circuit breakers, rate limiting, graceful degradation

3. **Check common pitfalls**: [troubleshooting/common-issues.md](troubleshooting/common-issues.md)
   Avoid typical mistakes and know how to debug issues

4. **Set up observability**: [references/observability.md](references/observability.md)
   Prometheus metrics, distributed tracing, structured logging

### 🔴 Advanced scenarios?

1. **Distributed transactions**: [references/distributed-transactions.md](references/distributed-transactions.md)
   DTM integration, SAGA/TCC patterns, data consistency

2. **Message queues**: [references/message-queue.md](references/message-queue.md)
   go-queue for async processing, delayed tasks, Kafka integration

3. **Performance optimization**: [references/advanced-components.md](references/advanced-components.md)
   Bloom filter, MapReduce, TimingWheel, SharedCalls

4. **Understand design**: [design/principles.md](design/principles.md)
   Framework philosophy, architecture decisions, anti-patterns

5. **Learn from examples**: [examples/case-studies/looklook-overview.md](examples/case-studies/looklook-overview.md)
   Production-scale architecture, real-world patterns

### 🔵 Extending capabilities?

1. **Use with Claude Code**: [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md)
   Learn advanced features like subagents, dynamic context, and argument passing
   Run demo projects to validate your environment

3. **Verify knowledge**: [examples/verify-tutorial.sh](examples/verify-tutorial.sh)
   Script to check if examples work correctly

## 🔗 Integration with go-zero AI Ecosystem

This skill is part of a two-layer ecosystem for AI-assisted go-zero development:

| Tool | Purpose | Best For |
|------|---------|----------|
| **[ai-context](https://github.com/zeromicro/ai-context)** | Concise workflow instructions (~5KB) | GitHub Copilot, Cursor, Windsurf |
| **zero-skills** (this repo) | Comprehensive knowledge base + goctl reference (~45KB) | All AI tools, deep learning, reference |

The AI runs `goctl` directly in the terminal for code generation — no separate MCP server needed. See [references/goctl-commands.md](references/goctl-commands.md) for the complete command reference.

**Usage in Claude Code:**
- This skill loads automatically when working with go-zero projects
- Use `/zero-skills` to invoke manually for go-zero guidance
- AI runs goctl commands directly in the terminal for code generation
- Reference specific pattern files when needed (Claude loads them on demand)

See [getting-started/claude-code-guide.md](getting-started/claude-code-guide.md) for detailed usage instructions.

## 🌐 Additional Resources

- **Official docs**: [go-zero.dev](https://go-zero.dev) - Latest API reference and guides
- **GitHub**: [zeromicro/go-zero](https://github.com/zeromicro/go-zero) - Source code and examples
- **Community**: Discussions, issues, and contributions welcome in the main repository

## 📝 Version Compatibility

- **Target version**: go-zero 1.5+
- **Go version**: Go 1.19 or later recommended
- **Updates**: Patterns updated regularly to reflect framework evolution
- **Breaking changes**: Check official docs for API changes between versions

---

**Quick invocation**: Use `/zero-skills` or ask "How do I [task] with go-zero?"
**Need help?** Reference the specific pattern guide for detailed examples and explanations.
