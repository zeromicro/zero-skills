# goctl Command Reference

> This file is a skill for AI assistants. It teaches the AI to use goctl directly in the terminal
> to generate go-zero code.

## 1. goctl Installation & Detection

### Check if goctl is installed

```bash
which goctl && goctl --version
```

### Install goctl if not found

```bash
go install github.com/zeromicro/go-zero/tools/goctl@latest
```

### Verify installation

```bash
goctl --version
```

If `go install` fails, check that `$GOPATH/bin` or `$HOME/go/bin` is in `$PATH`.

## 2. API Service

### 2.1 Create new API service

```bash
# Basic
goctl api new <service-name> --style go_zero

# With output directory
mkdir -p <output-dir> && cd <output-dir>
goctl api new <service-name> --style go_zero
```

### 2.2 Generate code from .api spec

```bash
goctl api go -api <file>.api -dir . --style go_zero
```

Safe to re-run — goctl will NOT overwrite files with custom logic (handler/logic files are only created if they don't exist).

### 2.3 Create .api spec file

Write the `.api` spec manually following the go-zero API syntax. See [API Spec Patterns](#api-spec-patterns) below.

### 2.4 Validate .api spec

```bash
goctl api validate -api <file>.api
```

## 3. RPC Service

### 3.1 Create new RPC service from .proto

```bash
goctl rpc protoc <file>.proto --go_out=. --go-grpc_out=. --zrpc_out=. --style go_zero
```

### 3.2 Create simple RPC service

```bash
goctl rpc new <service-name> --style go_zero
```

## 4. Database Model

### 4.1 From MySQL

```bash
# From live database
goctl model mysql datasource \
  -url "user:pass@tcp(host:3306)/dbname" \
  -table "<table-name>" \
  -dir ./model \
  --style go_zero

# With cache
goctl model mysql datasource \
  -url "user:pass@tcp(host:3306)/dbname" \
  -table "<table-name>" \
  -dir ./model \
  -cache \
  --style go_zero
```

### 4.2 From DDL file

```bash
goctl model mysql ddl -src <file>.sql -dir ./model --style go_zero

# With cache
goctl model mysql ddl -src <file>.sql -dir ./model -cache --style go_zero
```

### 4.3 From PostgreSQL

```bash
goctl model pg datasource \
  -url "postgres://user:pass@host:5432/dbname?sslmode=disable" \
  -table "<table-name>" \
  -dir ./model \
  --style go_zero
```

### 4.4 From MongoDB

```bash
goctl model mongo -type <TypeName> -dir ./model --style go_zero
```

## 5. Post-Generation Pipeline

**CRITICAL**: After every goctl generation command, always run these steps:

### Step 1: Initialize Go module (if new project)

```bash
# Only if go.mod doesn't exist
[ ! -f go.mod ] && go mod init <module-name>
```

### Step 2: Tidy dependencies

```bash
go mod tidy
```

### Step 3: Verify imports

Check that generated files have correct import paths matching the `module` in `go.mod`.
If imports reference the wrong module path, fix them:

```bash
# Find files with wrong import path
grep -r "old/module/path" --include="*.go" -l

# Fix imports (replace old path with correct one)
find . -name "*.go" -exec sed -i '' "s|old/module/path|correct/module/path|g" {} +
```

### Step 4: Verify build

```bash
go build ./...
```

If build fails, check:
- Missing imports → run `go mod tidy` again
- Import path mismatch → fix module paths (Step 3)
- Style conflicts → check `--style` flag matches existing code

### Step 5: Check naming style consistency

If the project already has generated files, check their naming convention:

```bash
# Check existing file names
ls internal/handler/ internal/logic/ internal/types/ 2>/dev/null
```

- Files like `get_user_handler.go` → use `--style go_zero`
- Files like `getuserhandler.go` → use `--style gozero` (or omit `--style`)
- Files like `getUserHandler.go` → use `--style goZero`

**Always match the existing style** to avoid conflicts.

## 6. Config Templates

### API service config (etc/<service>.yaml)

```yaml
Name: <service-name>
Host: 0.0.0.0
Port: <port>

# Database (if needed)
MySQL:
  DataSource: user:pass@tcp(localhost:3306)/dbname

# Auth (if needed)
Auth:
  AccessSecret: "your-secret-key-change-in-production"
  AccessExpire: 86400

# Cache (if needed)
Cache:
  - Host: localhost:6379
```

### RPC service config (etc/<service>.yaml)

```yaml
Name: <service-name>.rpc
ListenOn: 0.0.0.0:<port>

# Etcd for service discovery (if needed)
Etcd:
  Hosts:
    - localhost:2379
  Key: <service-name>.rpc

# Database (if needed)
MySQL:
  DataSource: user:pass@tcp(localhost:3306)/dbname
```

### Production config additions

```yaml
# Logging
Log:
  Mode: file
  Path: logs
  Level: error
  Compress: true
  KeepDays: 7

# Telemetry
Telemetry:
  Name: <service-name>
  Endpoint: http://localhost:14268/api/traces
  Sampler: 1.0

# Prometheus
Prometheus:
  Host: 0.0.0.0
  Port: 9091
  Path: /metrics
```

## 7. Deployment Templates

### Dockerfile

```dockerfile
FROM golang:1.23-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates tzdata
WORKDIR /app
COPY --from=builder /app/server .
COPY etc/ etc/

EXPOSE <port>
CMD ["./server", "-f", "etc/<service>.yaml"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <service-name>
  labels:
    app: <service-name>
spec:
  replicas: 3
  selector:
    matchLabels:
      app: <service-name>
  template:
    metadata:
      labels:
        app: <service-name>
    spec:
      containers:
        - name: <service-name>
          image: <registry>/<service-name>:latest
          ports:
            - containerPort: <port>
          volumeMounts:
            - name: config
              mountPath: /app/etc
          livenessProbe:
            httpGet:
              path: /health
              port: <port>
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
      volumes:
        - name: config
          configMap:
            name: <service-name>-config
---
apiVersion: v1
kind: Service
metadata:
  name: <service-name>
spec:
  selector:
    app: <service-name>
  ports:
    - port: <port>
      targetPort: <port>
  type: ClusterIP
```

### Docker Compose (development)

```yaml
version: '3.8'
services:
  <service-name>:
    build: .
    ports:
      - "<port>:<port>"
    volumes:
      - ./etc:/app/etc
    depends_on:
      - mysql
      - redis
    restart: unless-stopped

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: <dbname>
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  mysql-data:
```

## 8. Middleware Template

```go
package middleware

import "net/http"

type <Name>Middleware struct{}

func New<Name>Middleware() *<Name>Middleware {
	return &<Name>Middleware{}
}

func (m *<Name>Middleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// TODO: implement middleware logic
		next(w, r)
	}
}
```

## 9. Error Handler Template

```go
package errorx

import (
	"net/http"

	"github.com/zeromicro/go-zero/rest/httpx"
)

type CodeError struct {
	Code int    `json:"code"`
	Msg  string `json:"msg"`
}

func NewCodeError(code int, msg string) *CodeError {
	return &CodeError{Code: code, Msg: msg}
}

func (e *CodeError) Error() string {
	return e.Msg
}

// ErrorHandler is a custom error handler for go-zero REST server.
// Set it with: httpx.SetErrorHandler(errorx.ErrorHandler)
func ErrorHandler(err error) (int, any) {
	switch e := err.(type) {
	case *CodeError:
		return http.StatusOK, e
	default:
		return http.StatusInternalServerError, CodeError{
			Code: http.StatusInternalServerError,
			Msg:  err.Error(),
		}
	}
}
```

## 10. Common goctl Flags Reference

| Flag | Description | Example |
|------|-------------|---------|
| `--style` | Naming convention for generated files | `go_zero`, `gozero`, `goZero` |
| `-dir` | Output directory | `-dir ./` |
| `-api` | Path to .api spec file | `-api user.api` |
| `-cache` | Generate with cache support (model only) | `-cache` |
| `-url` | Database connection URL | `-url "user:pass@tcp(host)/db"` |
| `-table` | Table name for model generation | `-table users` |
| `-src` | Source DDL file path | `-src schema.sql` |
| `--home` | Custom template directory | `--home ~/.goctl/templates` |
| `--remote` | Remote template repo | `--remote https://github.com/...` |

## API Spec Patterns

### Basic CRUD

```api
syntax = "v1"

type (
    CreateRequest {
        Name  string `json:"name" validate:"required"`
        Email string `json:"email" validate:"required,email"`
    }
    CreateResponse {
        Id int64 `json:"id"`
    }
    GetRequest {
        Id int64 `path:"id"`
    }
    GetResponse {
        Id    int64  `json:"id"`
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    UpdateRequest {
        Id    int64  `path:"id"`
        Name  string `json:"name,optional"`
        Email string `json:"email,optional"`
    }
    DeleteRequest {
        Id int64 `path:"id"`
    }
    ListRequest {
        Page     int64 `form:"page,default=1"`
        PageSize int64 `form:"pageSize,default=20"`
    }
    ListResponse {
        Total int64        `json:"total"`
        Items []GetResponse `json:"items"`
    }
)

@server (
    group:  <resource>
    prefix: /api/v1
)
service <service-name> {
    @handler Create<Resource>
    post /<resources> (CreateRequest) returns (CreateResponse)

    @handler Get<Resource>
    get /<resources>/:id (GetRequest) returns (GetResponse)

    @handler Update<Resource>
    put /<resources>/:id (UpdateRequest) returns (GetResponse)

    @handler Delete<Resource>
    delete /<resources>/:id (DeleteRequest)

    @handler List<Resource>
    get /<resources> (ListRequest) returns (ListResponse)
}
```

### JWT Protected Routes

```api
@server (
    jwt:    Auth
    group:  <resource>
    prefix: /api/v1
)
service <service-name> {
    @handler GetProfile
    get /profile returns (ProfileResponse)
}
```

### Mixed (public + protected)

```api
// Public routes
@server (
    group:  auth
    prefix: /api/v1
)
service <service-name> {
    @handler Login
    post /login (LoginRequest) returns (LoginResponse)

    @handler Register
    post /register (RegisterRequest) returns (RegisterResponse)
}

// Protected routes
@server (
    jwt:    Auth
    group:  user
    prefix: /api/v1
)
service <service-name> {
    @handler GetProfile
    get /users/profile returns (ProfileResponse)
}
```

## Quick Reference: Full Workflow

```
1. Create .api spec file
2. goctl api go -api <file>.api -dir . --style go_zero
3. [ ! -f go.mod ] && go mod init <module>
4. go mod tidy
5. Verify imports match go.mod module path
6. go build ./...
7. Implement business logic in internal/logic/
8. Update config in etc/<service>.yaml
9. go run <service>.go -f etc/<service>.yaml
```
