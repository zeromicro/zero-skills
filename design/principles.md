# go-zero Design Principles

## Core Philosophy

go-zero is a microservices framework designed with **engineering pragmatism** - balancing simplicity, performance, and reliability. The framework follows these guiding principles:

1. **Convention over Configuration** - Sensible defaults, minimal setup
2. **Built-in Best Practices** - Resilience patterns enabled by default
3. **Code Generation** - `goctl` generates boilerplate, developers focus on business logic
4. **Clear Architecture** - Three-layer pattern enforced consistently

## Architecture Design

### Three-Layer Pattern

go-zero enforces a strict three-layer architecture for both REST and RPC services:

```
┌─────────────────────────────────────────────────────────┐
│                    Handler Layer                        │
│  • HTTP routing and request parsing                     │
│  • Response serialization                                │
│  • NO business logic                                     │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                    Logic Layer                          │
│  • Business logic implementation                        │
│  • Validation and orchestration                         │
│  • Calls to external services/models                    │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                    Model Layer                          │
│  • Database operations                                  │
│  • Cache operations                                     │
│  • External service calls                               │
└─────────────────────────────────────────────────────────┘
```

**Why This Matters:**

| Layer | Responsibility | Testability |
|-------|---------------|-------------|
| Handler | HTTP concerns only | Easy to mock |
| Logic | Pure business logic | Unit test friendly |
| Model | Data access | Integration tests |

### ServiceContext Pattern

Dependencies are injected through `ServiceContext`, enabling:

- **Loose coupling** between layers
- **Easy testing** with mock implementations
- **Centralized configuration** of dependencies

```go
type ServiceContext struct {
    Config    config.Config
    UserModel model.UserModel
    Redis     *redis.Redis
    UserRpc   user.UserClient
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:    c,
        UserModel: model.NewUserModel(sqlx.NewMysql(c.DataSource)),
        Redis:     redis.New(c.Redis.Host),
        UserRpc:   user.NewUser(zrpc.MustNewClient(c.UserRpc)),
    }
}
```

## Design Decisions

### 1. Code Generation over Manual Scaffolding

**Rationale:** Manual boilerplate is error-prone and inconsistent. `goctl` ensures:

- Consistent project structure
- Correct naming conventions
- Up-to-date API/RPC definitions
- Type-safe request/response handling

**Impact:** Developers write ~20% less code, focus on business logic

### 2. Built-in Resilience Patterns

go-zero includes production-ready resilience patterns:

| Pattern | Implementation | Default |
|---------|---------------|---------|
| Circuit Breaker | `breaker` package | Enabled |
| Rate Limiting | `periodlimit`, `tokenlimit` | Optional |
| Load Shedding | `loadshedding` package | Optional |
| Timeout Control | Configuration | Enabled |
| Retry | `retry` package | Optional |

**Why Built-in:** Most production failures are caused by missing these patterns, not business logic bugs.

### 3. Configuration-Driven Behavior

Services are configured through YAML files with sensible defaults:

```yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Timeout: 30000  # 30 seconds

# Built-in middleware
Log:
  ServiceName: user-api
  Mode: console
  
# Resilience
Breaker:
  Threshold: 0.5
  Timeout: 60s
```

**Rationale:** Configuration files are:

- Version controllable
- Environment-specific (dev/staging/prod)
- Self-documenting

### 4. Context Propagation

Every function in go-zero takes `context.Context` as the first parameter:

```go
func (l *GetUserLogic) GetUser(req *types.GetUserReq) (*types.User, error) {
    // Context carries:
    // - Trace ID for distributed tracing
    // - Timeout/deadline
    // - User authentication info
    // - Request-scoped values
    
    user, err := l.svcCtx.UserModel.FindOne(l.ctx, req.UserID)
    // ...
}
```

**Why:** Enables:

- Distributed tracing across services
- Request cancellation
- Timeout propagation
- Metadata passing

### 5. Error Handling Strategy

go-zero distinguishes between:

| Error Type | Handling | HTTP Status |
|------------|----------|-------------|
| Business errors | Return to user | 4xx |
| System errors | Log and return | 5xx |
| Validation errors | Return to user | 400 |

```go
// Business error
var ErrUserNotFound = errors.New("user not found")

// System error - wrap with context
return nil, fmt.Errorf("database query failed: %w", err)
```

## Performance Design

### 1. Minimal Allocations

go-zero optimizes for low GC pressure:

- Reusable buffers
- Object pooling
- String builders over concatenation

### 2. Concurrent-Safe by Default

All components are safe for concurrent use:

- `sync.Map` for caching
- Channel-based communication
- Lock-free algorithms where possible

### 3. Efficient Serialization

- Custom JSON encoder for better performance
- Protobuf for RPC communication
- msgpack support available

## Scalability Design

### Horizontal Scaling

Services are designed to be stateless:

1. **No sticky sessions** - Any instance can handle any request
2. **External state storage** - Redis/Database for session data
3. **Service discovery** - Built-in etcd/consul support

### Service Communication

```
┌──────────┐    ┌──────────┐    ┌──────────┐
│  API     │───▶│  RPC     │───▶│  RPC     │
│ Gateway  │    │ Service  │    │ Service  │
└──────────┘    └──────────┘    └──────────┘
     │               │               │
     └───────────────┴───────────────┘
                     │
              ┌──────▼──────┐
              │ Service     │
              │ Discovery   │
              │ (etcd)      │
              └─────────────┘
```

## Anti-Patterns to Avoid

### ❌ Business Logic in Handlers

```go
// WRONG
func UserHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // ❌ Business logic in handler
        user, err := svcCtx.UserModel.FindOne(ctx, id)
        if user.Age < 18 {
            // validation logic
        }
    }
}
```

### ❌ Hardcoded Configuration

```go
// WRONG
db, err := sql.Open("mysql", "user:pass@tcp(localhost:3306)/db")

// CORRECT
db, err := sql.Open("mysql", c.DataSource)  // From config
```

### ❌ Ignoring Context

```go
// WRONG
func Process(data string) error {
    // ❌ No context, no timeout control
    result := callExternalService(data)
}

// CORRECT
func Process(ctx context.Context, data string) error {
    // ✅ Context enables timeout and tracing
    result := callExternalService(ctx, data)
}
```

### ❌ Panic for Recoverable Errors

```go
// WRONG
if user == nil {
    panic("user is nil")  // ❌ Crashes the service
}

// CORRECT
if user == nil {
    return nil, errors.New("user is nil")  // ✅ Return error
}
```

## When to Use go-zero

### ✅ Good Fit

- Microservices architecture
- High-traffic APIs
- Systems requiring resilience patterns
- Teams wanting consistent code structure
- Projects with frequent onboarding

### ⚠️ Consider Alternatives

- Simple monolithic applications (use Gin/Echo)
- Prototyping (too much structure)
- Non-Go projects

## Evolution Philosophy

go-zero follows a **backward-compatible evolution**:

1. New features are additive
2. Deprecated features have migration paths
3. Breaking changes require major version bump
4. `goctl` is updated alongside the framework

## Summary

go-zero's design philosophy prioritizes:

1. **Developer productivity** through code generation
2. **Production readiness** through built-in resilience
3. **Maintainability** through clear architecture
4. **Performance** through careful optimization
5. **Scalability** through stateless design

By following these principles, teams can build reliable, maintainable microservices without reinventing common patterns.