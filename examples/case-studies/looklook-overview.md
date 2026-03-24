# go-zero-looklook: Large-Scale Example Project

## Overview

[go-zero-looklook](https://github.com/Mikaelemmmm/go-zero-looklook) is a comprehensive production-style example demonstrating go-zero best practices. It's an excellent reference for building real-world microservices.

## Project Architecture

```
go-zero-looklook/
├── app/                          # Business services
│   ├── identity/                 # User identity service (RPC)
│   ├── usercenter/               # User center service (API + RPC)
│   ├── order/                    # Order service (RPC)
│   ├── payment/                  # Payment service (RPC)
│   ├── travel/                   # Travel service (API + RPC)
│   └── mqueue/                   # Message queue service (job)
├── common/                       # Shared utilities
│   ├── result/                   # Response utilities
│   ├── errorx/                   # Error handling
│   ├── ctxdata/                  # Context utilities
│   └── utils/                    # Common utilities
├── deploy/                       # Deployment configs
│   ├── docker-compose/           # Local development
│   ├── k8s/                      # Kubernetes deployment
│   └── prometheus/               # Monitoring setup
└── service/                      # Gateway services
    └── gateway/                  # API Gateway
```

## Service Breakdown

### Core Services

| Service | Type | Responsibility |
|---------|------|----------------|
| **identity** | RPC | Authentication, JWT token management |
| **usercenter** | API + RPC | User profile, settings |
| **order** | RPC | Order creation, management |
| **payment** | RPC | Payment processing |
| **travel** | API + RPC | Travel booking, homestay |
| **mqueue** | Job | Scheduled tasks, async processing |
| **gateway** | API | Unified entry point, routing |

### Communication Pattern

```
                    ┌─────────────┐
                    │   Gateway   │
                    │   (API)     │
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │usercenter│    │  travel  │    │  order   │
    │ (API/RPC)│    │ (API/RPC)│    │   (RPC)  │
    └────┬─────┘    └────┬─────┘    └────┬─────┘
         │               │               │
         └───────────────┼───────────────┘
                         │
                    ┌────▼────┐
                    │ identity│
                    │  (RPC)  │
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │ MySQL  │ │ Redis  │ │  etcd  │
         └────────┘ └────────┘ └────────┘
```

## Key Patterns Demonstrated

### 1. API Gateway Pattern

```go
// service/gateway/internal/handler/routes.go
func RegisterHandlers(server *rest.Server, serverCtx *svc.ServiceContext) {
    server.AddRoutes(
        []rest.Route{
            {
                Method:  http.MethodGet,
                Path:    "/user/info",
                Handler: user.UserInfoHandler(serverCtx),
            },
            {
                Method:  http.MethodPost,
                Path:    "/order/create",
                Handler: order.CreateOrderHandler(serverCtx),
            },
        },
        rest.WithJwt(serverCtx.Config.Auth.AccessSecret),
        rest.WithTimeout(30*time.Second),
    )
}
```

### 2. Service Discovery (etcd)

```yaml
# app/usercenter/cmd/api/etc/usercenter.yaml
Name: usercenter-api
Host: 0.0.0.0
Port: 8002

UsercenterRpc:
  Etcd:
    Hosts:
      - etcd:2379
    Key: usercenter.rpc

IdentityRpc:
  Etcd:
    Hosts:
      - etcd:2379
    Key: identity.rpc
```

### 3. JWT Authentication

```go
// app/identity/cmd/rpc/internal/logic/generateToken.go
func (l *GenerateTokenLogic) GenerateToken(in *identity.GenerateTokenReq) (*identity.GenerateTokenResp, error) {
    now := time.Now().Unix()
    accessExpire := l.svcCtx.Config.JWT.AccessExpire
    
    accessToken, err := l.genToken(now, accessExpire, in.UserId, "access")
    if err != nil {
        return nil, errors.New("generate access token failed")
    }
    
    refreshToken, err := l.genToken(now, accessExpire*2, in.UserId, "refresh")
    if err != nil {
        return nil, errors.New("generate refresh token failed")
    }
    
    return &identity.GenerateTokenResp{
        AccessToken:  accessToken,
        AccessExpire: now + accessExpire,
        RefreshToken: refreshToken,
    }, nil
}
```

### 4. Error Handling

```go
// common/errorx/errors.go
type CodeError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
}

func NewCodeError(code int, message string) error {
    return &CodeError{
        Code:    code,
        Message: message,
    }
}

// Response format
func (e *CodeError) Data() *Response {
    return &Response{
        Code:    e.Code,
        Message: e.Message,
    }
}
```

```go
// common/result/httpResult.go
func HttpResult(r *http.Request, w http.ResponseWriter, resp interface{}, err error) {
    if err == nil {
        OkJson(w, resp)
    } else {
        // Handle different error types
        switch e := err.(type) {
        case *errorx.CodeError:
            WriteJson(w, e.Code, e.Data())
        default:
            WriteJson(w, http.StatusInternalServerError, err.Error())
        }
    }
}
```

### 5. Message Queue Integration

```go
// app/mqueue/cmd/job/internal/logic/scheduler.go
func (l *SchedulerLogic) Start() {
    // Schedule order timeout check every minute
    l.svcCtx.Scheduler.AddJob("0 */1 * * * *", func() {
        l.checkPendingOrders()
    })
    
    // Schedule homestay cache refresh
    l.svcCtx.Scheduler.AddJob("0 0 3 * * *", func() {
        l.refreshHomestayCache()
    })
}
```

### 6. Context Data Passing

```go
// common/ctxdata/ctxdata.go
func SetUidToCtx(ctx context.Context, uid int64) context.Context {
    return context.WithValue(ctx, CtxKeyUid, uid)
}

func GetUidFromCtx(ctx context.Context) int64 {
    val := ctx.Value(CtxKeyUid)
    if val == nil {
        return 0
    }
    return val.(int64)
}
```

```go
// In middleware
func (m *AuthMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Extract user ID from JWT
        uid := r.Context().Value("uid").(json.Number)
        
        // Add to context for downstream use
        ctx := ctxdata.SetUidToCtx(r.Context(), uid.Int64())
        next(w, r.WithContext(ctx))
    }
}

// In logic layer
func (l *GetUserInfoLogic) GetUserInfo(req *types.UserInfoReq) (*types.UserInfoResp, error) {
    // Get user ID from context
    uid := ctxdata.GetUidFromCtx(l.ctx)
    
    user, err := l.svcCtx.UserModel.FindOne(l.ctx, uid)
    // ...
}
```

## Configuration Management

### Multi-Environment Configuration

```
app/usercenter/cmd/api/etc/
├── usercenter.yaml           # Production
├── usercenter-dev.yaml       # Development
└── usercenter-test.yaml      # Testing
```

```yaml
# usercenter.yaml (Production)
Name: usercenter-api
Host: 0.0.0.0
Port: 8002

Log:
  ServiceName: usercenter-api
  Mode: file
  Level: info

MySQL:
  DataSource: user:pass@tcp(mysql:3306)/looklook_usercenter?charset=utf8mb4

Redis:
  Host: redis:6379
  Type: node

UsercenterRpc:
  Etcd:
    Hosts:
      - etcd:2379
    Key: usercenter.rpc
```

## Deployment Architecture

### Docker Compose (Development)

```yaml
# deploy/docker-compose/docker-compose.yml
version: '3'

services:
  etcd:
    image: bitnami/etcd:latest
    ports:
      - "2379:2379"

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root

  usercenter-api:
    build:
      context: ../../
      dockerfile: app/usercenter/cmd/api/Dockerfile
    ports:
      - "8002:8002"
    depends_on:
      - etcd
      - redis
      - mysql
```

### Kubernetes (Production)

```yaml
# deploy/k8s/usercenter-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: usercenter-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: usercenter-api
  template:
    spec:
      containers:
        - name: usercenter-api
          image: registry/usercenter-api:latest
          ports:
            - containerPort: 8002
          envFrom:
            - configMapRef:
                name: usercenter-config
```

## Observability Setup

### Prometheus Configuration

```yaml
# deploy/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'gozero-services'
    static_configs:
      - targets:
          - 'usercenter-api:9091'
          - 'identity-rpc:9091'
          - 'order-rpc:9091'
```

### Jaeger Tracing

```yaml
# In service config
Telemetry:
  Name: usercenter-api
  Endpoint: http://jaeger:14268/api/traces
  Sampler: 1.0
  Batcher: jaeger
```

## Key Takeaways

### Architecture Lessons

1. **Separate API from RPC** - User-facing APIs vs internal service calls
2. **Gateway as single entry point** - Routing, auth, rate limiting
3. **Shared common package** - DRY principle, consistent error handling
4. **Context for user data** - Pass user identity through service calls

### Best Practices

1. **Consistent error handling** - Use `CodeError` for business errors
2. **Context data passing** - Never pass sensitive data in request body
3. **Configuration per environment** - Separate configs for dev/test/prod
4. **Health checks** - Implement `/healthz` for Kubernetes probes

### Anti-Patterns to Avoid

1. **Direct DB calls from handlers** - Always go through logic layer
2. **Hardcoded configuration** - Use config files
3. **Missing error context** - Wrap errors with meaningful messages
4. **Ignoring context cancellation** - Check `ctx.Done()` in long operations

## Running the Project

```bash
# Clone repository
git clone https://github.com/Mikaelemmmm/go-zero-looklook.git
cd go-zero-looklook

# Start infrastructure
cd deploy/docker-compose
docker-compose up -d

# Run services
go run app/usercenter/cmd/api/usercenter.go -f app/usercenter/cmd/api/etc/usercenter.yaml
```

## References

- [go-zero-looklook GitHub](https://github.com/Mikaelemmmm/go-zero-looklook)
- [go-zero Official Examples](https://github.com/zeromicro/go-zero/tree/master/example)
- [go-zero Documentation](https://go-zero.dev)