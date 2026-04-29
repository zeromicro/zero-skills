# Best Practices

## Code Organization

### ✅ Project Structure

```
service-name/
├── etc/
│   └── config.yaml           # Configuration files
├── internal/
│   ├── config/
│   │   └── config.go         # Config struct
│   ├── handler/
│   │   └── *handler.go       # HTTP handlers (thin layer)
│   ├── logic/
│   │   └── *logic.go         # Business logic (thick layer)
│   ├── middleware/
│   │   └── *middleware.go    # Custom middlewares
│   ├── svc/
│   │   └── servicecontext.go # Dependency injection
│   ├── types/
│   │   └── types.go          # Request/Response types
│   └── model/
│       └── *model.go         # Database models
├── service.go                # Entry point
└── service.api               # API definition
```

**Key Principles:**
- Keep `handler` thin - only HTTP concerns
- Put business logic in `logic` layer
- Centralize dependencies in `svc`
- Generated code in `internal/`, custom code alongside it

### ✅ File Naming

```go
// Handlers: <resource><action>handler.go
createuserhandler.go
getuserhandler.go
updateuserhandler.go

// Logic: <resource><action>logic.go
createuserlogic.go
getuserlogic.go
updateuserlogic.go

// Models: <table>model.go
usermodel.go
ordermodel.go
productmodel.go

// Middleware: <purpose>middleware.go
authmiddleware.go
loggingmiddleware.go
ratelimitmiddleware.go
```

## Configuration Management

### ✅ Configuration Pattern

```go
// internal/config/config.go
type Config struct {
    rest.RestConf  // or zrpc.RpcServerConf for RPC

    // Group related settings
    Auth struct {
        AccessSecret string
        AccessExpire int64
    }

    Database struct {
        DataSource string
        Cache      cache.CacheConf
    }

    Redis struct {
        Host string
        Type string
        Pass string `json:",optional"`
    }

    // Use tags effectively
    MaxUploadSize int64  `json:",default=10485760"`  // 10MB
    EnableFeature bool   `json:",default=true"`
    Environment   string `json:",default=prod,options=[dev|test|prod]"`
}
```

### ✅ Configuration File

```yaml
# etc/service.yaml
Name: user-api
Host: 0.0.0.0
Port: 8888
Timeout: 30000

Auth:
  AccessSecret: your-secret-key
  AccessExpire: 3600

Database:
  DataSource: "user:pass@tcp(localhost:3306)/db?parseTime=true"
  Cache:
    - Host: localhost:6379
      Type: node

Redis:
  Host: localhost:6379
  Type: node

MaxUploadSize: 52428800  # 50MB
EnableFeature: true
Environment: prod
```

### ✅ Environment Variables

```go
// Support environment variable overrides
// Set in shell: export USER_API_PORT=9999
// Will override Port in config file

// Or use .env file with godotenv
import "github.com/joho/godotenv"

func main() {
    _ = godotenv.Load()  // Load .env file

    var c config.Config
    conf.MustLoad(*configFile, &c)
    // ...
}
```

## Error Handling

### ✅ Error Definition

```go
// Define errors at package level
var (
    ErrUserNotFound     = errors.New("user not found")
    ErrInvalidInput     = errors.New("invalid input")
    ErrUnauthorized     = errors.New("unauthorized")
    ErrDuplicateEmail   = errors.New("email already exists")
    ErrInsufficientFunds = errors.New("insufficient funds")
)

// Or use custom error types
type BusinessError struct {
    Code    int
    Message string
}

func (e *BusinessError) Error() string {
    return e.Message
}
```

### ✅ Error Wrapping

```go
func (l *Logic) Operation() error {
    user, err := l.svcCtx.UserModel.FindOne(l.ctx, id)
    if err != nil {
        // Wrap errors with context
        return fmt.Errorf("failed to find user %d: %w", id, err)
    }

    // Use errors.Is for checking
    if errors.Is(err, sqlc.ErrNotFound) {
        return ErrUserNotFound
    }

    return nil
}
```

### ✅ Error Response

```go
// Custom error handler
httpx.SetErrorHandler(func(err error) (int, any) {
    switch {
    case errors.Is(err, ErrUserNotFound):
        return http.StatusNotFound, map[string]string{
            "code":    "USER_NOT_FOUND",
            "message": err.Error(),
        }
    case errors.Is(err, ErrInvalidInput):
        return http.StatusBadRequest, map[string]string{
            "code":    "INVALID_INPUT",
            "message": err.Error(),
        }
    case errors.Is(err, ErrUnauthorized):
        return http.StatusUnauthorized, map[string]string{
            "code":    "UNAUTHORIZED",
            "message": err.Error(),
        }
    default:
        return http.StatusInternalServerError, map[string]string{
            "code":    "INTERNAL_ERROR",
            "message": "internal server error",
        }
    }
})
```

## Logging

### ✅ Structured Logging

```go
// Use logx for structured logging
import "github.com/zeromicro/go-zero/core/logx"

// In logic
func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    // Info level
    l.Logger.Infof("creating user: %s", req.Email)

    // With fields
    l.Logger.WithFields(logx.Field("email", req.Email), logx.Field("age", req.Age)).
        Info("user validation passed")

    user, err := l.createUser(req)
    if err != nil {
        // Error level with context
        l.Logger.Errorf("failed to create user %s: %v", req.Email, err)
        return nil, err
    }

    // Success with result
    l.Logger.Infow("user created successfully",
        logx.Field("user_id", user.Id),
        logx.Field("email", user.Email),
    )

    return &types.CreateUserResponse{Id: user.Id}, nil
}
```

### ✅ Log Configuration

```yaml
Log:
  Mode: console  # console or file
  Level: info    # debug, info, error, severe
  Encoding: json # json or plain
  Path: logs     # for file mode
  MaxSize: 100   # MB
  MaxBackups: 30
  MaxAge: 7      # days
  Compress: true
```

### ❌ Logging Anti-Patterns

```go
// DON'T: Log sensitive information
l.Logger.Infof("user password: %s", password)  // ❌
l.Logger.Infof("credit card: %s", ccNumber)    // ❌
l.Logger.Infof("auth token: %s", token)        // ❌

// DON'T: Log in loops without throttling
for _, item := range items {
    l.Logger.Infof("processing %v", item)  // ❌ Too verbose
}

// DON'T: Use print statements
fmt.Println("debug info")  // ❌ Use l.Logger instead
log.Println("error")       // ❌ Use l.Logger instead

// DO: Log summary
l.Logger.Infof("processing %d items", len(items))  // ✅
```

## Testing

### ✅ Unit Test Pattern

```go
// logic_test.go
package logic

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/zeromicro/go-zero/core/logx"
)

func TestCreateUserLogic_CreateUser(t *testing.T) {
    tests := []struct {
        name    string
        req     *types.CreateUserRequest
        wantErr bool
        errMsg  string
    }{
        {
            name: "valid user",
            req: &types.CreateUserRequest{
                Name:  "John Doe",
                Email: "john@example.com",
                Age:   25,
            },
            wantErr: false,
        },
        {
            name: "invalid age",
            req: &types.CreateUserRequest{
                Name:  "Jane Doe",
                Email: "jane@example.com",
                Age:   15,
            },
            wantErr: true,
            errMsg:  "age must be at least 18",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            ctx := context.Background()
            svcCtx := &svc.ServiceContext{
                // Mock dependencies
            }

            logic := NewCreateUserLogic(ctx, svcCtx)

            // Execute
            resp, err := logic.CreateUser(tt.req)

            // Assert
            if tt.wantErr {
                assert.Error(t, err)
                if tt.errMsg != "" {
                    assert.Contains(t, err.Error(), tt.errMsg)
                }
                assert.Nil(t, resp)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, resp)
                assert.Greater(t, resp.Id, int64(0))
            }
        })
    }
}
```

### ✅ Integration Test with Database

```go
func TestUserModel_Integration(t *testing.T) {
    // Setup test database
    conn := sqlx.NewMysql("test:test@tcp(localhost:3306)/testdb")
    model := NewUsersModel(conn, cache.CacheConf{})

    ctx := context.Background()

    // Cleanup after test
    defer func() {
        conn.Exec("DELETE FROM users WHERE email = ?", "test@example.com")
    }()

    // Test insert
    user := &Users{
        Name:  "Test User",
        Email: "test@example.com",
        Age:   25,
    }
    result, err := model.Insert(ctx, user)
    assert.NoError(t, err)

    userId, _ := result.LastInsertId()
    assert.Greater(t, userId, int64(0))

    // Test find
    found, err := model.FindOne(ctx, userId)
    assert.NoError(t, err)
    assert.Equal(t, user.Name, found.Name)
    assert.Equal(t, user.Email, found.Email)
}
```

### ✅ Mock Dependencies

```go
import "go.uber.org/mock/gomock"

func TestWithMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Create mock
    mockModel := mock.NewMockUsersModel(ctrl)

    // Set expectations
    mockModel.EXPECT().
        FindOne(gomock.Any(), int64(1)).
        Return(&model.Users{
            Id:    1,
            Name:  "John",
            Email: "john@example.com",
        }, nil)

    // Use mock in test
    svcCtx := &svc.ServiceContext{
        UsersModel: mockModel,
    }

    logic := NewGetUserLogic(context.Background(), svcCtx)
    resp, err := logic.GetUser(&types.GetUserRequest{Id: 1})

    assert.NoError(t, err)
    assert.Equal(t, "John", resp.Name)
}
```

## Performance

### ✅ Connection Pooling

```go
// Reuse connections - initialized once in service context
func NewServiceContext(c config.Config) *ServiceContext {
    // Single connection pool for entire service
    conn := sqlx.NewMysql(c.DataSource)

    // Single Redis client
    rds := redis.MustNewRedis(c.Redis)

    return &ServiceContext{
        Config: c,
        DB:     conn,
        Redis:  rds,
        UsersModel: model.NewUsersModel(conn, c.Cache),
    }
}

// DON'T create connections in handlers or logic
```

### ✅ Caching Strategy

```go
// Cache read-heavy data
func (l *GetUserLogic) GetUser(req *types.GetUserRequest) (*types.GetUserResponse, error) {
    // Automatic cache with model
    user, err := l.svcCtx.UsersModel.FindOne(l.ctx, req.Id)
    if err != nil {
        return nil, err
    }

    return &types.GetUserResponse{
        Id:    user.Id,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}

// Manual cache for complex queries
func (l *GetUserStatsLogic) GetUserStats(userId int64) (*Stats, error) {
    cacheKey := fmt.Sprintf("user:stats:%d", userId)

    var stats Stats
    err := l.svcCtx.Redis.GetCtx(l.ctx, cacheKey, &stats)
    if err == nil {
        return &stats, nil
    }

    // Cache miss - fetch from DB
    stats, err = l.fetchStatsFromDB(userId)
    if err != nil {
        return nil, err
    }

    // Cache for 1 hour
    l.svcCtx.Redis.SetexCtx(l.ctx, cacheKey, stats, 3600)
    return &stats, nil
}
```

### ✅ Batch Operations

```go
// Use MapReduce for parallel processing
import "github.com/zeromicro/go-zero/core/mr"

func (l *BatchLogic) ProcessUsers(userIds []int64) error {
    _, err := mr.MapReduce(
        func(source chan<- interface{}) {
            for _, id := range userIds {
                source <- id
            }
        },
        func(item interface{}, writer mr.Writer, cancel func(error)) {
            id := item.(int64)
            if err := l.processUser(id); err != nil {
                l.Logger.Errorf("failed to process user %d: %v", id, err)
            }
        },
        func(pipe <-chan interface{}, writer mr.Writer, cancel func(error)) {
            // Aggregate if needed
        },
        mr.WithWorkers(10),
    )
    return err
}
```

### ❌ Performance Anti-Patterns

```go
// DON'T: Query in loops (N+1 problem)
for _, orderId := range orderIds {
    order, _ := l.svcCtx.OrderModel.FindOne(l.ctx, orderId)  // ❌
    orders = append(orders, order)
}

// DO: Batch query
orders, err := l.svcCtx.OrderModel.FindMany(l.ctx, orderIds)  // ✅

// DON'T: Create goroutines without limit
for _, item := range items {
    go l.process(item)  // ❌ Unbounded goroutines
}

// DO: Use worker pool
workers := threading.NewTaskRunner(10)  // ✅ Limited to 10
for _, item := range items {
    item := item
    workers.Schedule(func() {
        l.process(item)
    })
}
workers.Wait()
```

## Security

### ✅ Input Validation

```go
// Use validation tags
type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=2,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Age      int    `json:"age" validate:"required,gte=18,lte=120"`
    Password string `json:"password" validate:"required,min=8"`
}

// Validate in logic
import "github.com/go-playground/validator/v10"

func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    validate := validator.New()
    if err := validate.Struct(req); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    // Continue processing...
}
```

### ✅ Password Handling

```go
import "golang.org/x/crypto/bcrypt"

// Hash password before storing
func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

// Verify password
func checkPassword(hashedPassword, password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password))
    return err == nil
}

// In logic
func (l *CreateUserLogic) CreateUser(req *types.CreateUserRequest) (*types.CreateUserResponse, error) {
    // Hash password
    hashedPassword, err := hashPassword(req.Password)
    if err != nil {
        return nil, err
    }

    user := &model.Users{
        Name:     req.Name,
        Email:    req.Email,
        Password: hashedPassword,  // Store hashed password
    }

    // Never log the password
    l.Logger.Infof("creating user: %s", req.Email)

    // ...
}
```

### ✅ JWT Authentication

```go
import "github.com/golang-jwt/jwt/v5"

func generateToken(userId int64, secret string, expire int64) (string, error) {
    now := time.Now().Unix()
    claims := make(jwt.MapClaims)
    claims["userId"] = userId
    claims["iat"] = now
    claims["exp"] = now + expire

    token := jwt.New(jwt.SigningMethodHS256)
    token.Claims = claims

    return token.SignedString([]byte(secret))
}

func validateToken(tokenString, secret string) (int64, error) {
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return []byte(secret), nil
    }, jwt.WithValidMethods([]string{"HS256"}))

    if err != nil || !token.Valid {
        return 0, errors.New("invalid token")
    }

    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return 0, errors.New("invalid claims")
    }

    userId := int64(claims["userId"].(float64))
    return userId, nil
}
```

### ❌ Security Anti-Patterns

```go
// DON'T: Store plain text passwords
user.Password = req.Password  // ❌

// DON'T: Log sensitive data
l.Logger.Infof("password: %s", password)  // ❌
l.Logger.Infof("token: %s", token)        // ❌

// DON'T: Use weak secrets
secret := "123456"  // ❌

// DON'T: Expose internal errors to clients
return nil, fmt.Errorf("database error: %v", err)  // ❌
// DO: Return generic error
return nil, errors.New("internal server error")     // ✅

// DON'T: Trust user input without validation
filePath := req.FilePath  // ❌ Path traversal risk
// DO: Validate and sanitize
filePath := filepath.Clean(req.FilePath)  // ✅
```

## Deployment

### ✅ Docker

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o service .

FROM alpine:latest
RUN apk --no-cache add ca-certificates tzdata

WORKDIR /app
COPY --from=builder /app/service .
COPY --from=builder /app/etc ./etc

EXPOSE 8888
CMD ["./service", "-f", "etc/config.yaml"]
```

### ✅ Kubernetes

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-api
  template:
    metadata:
      labels:
        app: user-api
    spec:
      containers:
      - name: user-api
        image: user-api:latest
        ports:
        - containerPort: 8888
        env:
        - name: USER_API_MODE
          value: "pro"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8888
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8888
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: user-api
spec:
  selector:
    app: user-api
  ports:
  - port: 8888
    targetPort: 8888
  type: ClusterIP
```

## Summary

### Always Do:
1. Keep handlers thin, logic thick
2. Use structured logging with context
3. Handle all errors explicitly
4. Validate input thoroughly
5. Use connection pooling
6. Enable caching for read-heavy data
7. Write unit tests
8. Use transactions for atomic operations
9. Implement proper security measures
10. Monitor production metrics

### Never Do:
1. Put business logic in handlers
2. Log sensitive information
3. Ignore errors
4. Create connections in handlers
5. Query in loops
6. Disable resilience features in production
7. Use global variables
8. Block without timeouts
9. Create unbounded goroutines
10. Trust user input without validation
