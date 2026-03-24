# Observability Patterns

## Overview

go-zero provides built-in support for observability through Prometheus metrics, distributed tracing, and structured logging. This guide covers monitoring, tracing, and alerting patterns.

## Three Pillars of Observability

| Pillar | Tool | Purpose |
|--------|------|---------|
| **Metrics** | Prometheus | Quantitative monitoring (QPS, latency, error rate) |
| **Tracing** | OpenTelemetry/Jaeger | Request flow across services |
| **Logging** | logx | Structured event recording |

## Prometheus Metrics

### ✅ Correct Pattern: Enable Built-in Metrics

```yaml
# etc/service.yaml
Name: user-api
Host: 0.0.0.0
Port: 8888

# Enable Prometheus metrics
Prometheus:
  Host: 0.0.0.0    # Metrics endpoint host
  Port: 9091       # Metrics endpoint port
  Path: /metrics   # Metrics path (default: /metrics)
```

### ✅ Correct Pattern: Custom Metrics

```go
// internal/metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Counter: Total requests
    RequestTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total number of HTTP requests",
    }, []string{"method", "path", "status"})

    // Histogram: Request duration
    RequestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration in seconds",
        Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
    }, []string{"method", "path"})

    // Gauge: Active connections
    ActiveConnections = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "active_connections",
        Help: "Number of active connections",
    })

    // Counter: Business metrics
    OrdersCreated = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "orders_created_total",
        Help: "Total number of orders created",
    }, []string{"status", "payment_method"})
)
```

### ✅ Correct Pattern: Record Metrics in Logic Layer

```go
// internal/logic/createorderlogic.go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    start := time.Now()
    
    // Business logic
    resp, err := l.createOrder(req)
    
    // Record metrics
    duration := time.Since(start).Seconds()
    status := "success"
    if err != nil {
        status = "error"
    }
    
    metrics.RequestTotal.WithLabelValues("POST", "/api/order", status).Inc()
    metrics.RequestDuration.WithLabelValues("POST", "/api/order").Observe(duration)
    
    if err == nil {
        metrics.OrdersCreated.WithLabelValues("success", req.PaymentMethod).Inc()
    }
    
    return resp, err
}
```

### ❌ Common Mistakes

```go
// DON'T: Record metrics in handler (violates layer separation)
func CreateUserHandler(svcCtx *svc.ServiceContext) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // ❌ Metrics should be in logic layer
        metrics.RequestTotal.Inc()
        
        var req types.CreateUserReq
        httpx.Parse(r, &req)
        // ...
    }
}

// DON'T: Use high-cardinality labels
metrics.RequestTotal.WithLabelValues(
    "POST",
    "/api/order",
    userID,  // ❌ High cardinality - can explode memory
).Inc()

// DON'T: Forget to register custom metrics
var myCounter = prometheus.NewCounter(...)  // ❌ Not registered
// Use promauto or prometheus.MustRegister()
```

## Distributed Tracing

### ✅ Correct Pattern: Enable Tracing

```yaml
# etc/service.yaml
Name: user-api
Host: 0.0.0.0
Port: 8888

Telemetry:
  Name: user-api
  Endpoint: http://jaeger:14268/api/traces
  Sampler: 1.0  # Sample rate (0.0 - 1.0)
  Batcher: jaeger  # jaeger, zipkin, otlp
```

### ✅ Correct Pattern: Context Propagation

```go
// internal/logic/getuserlogic.go
func (l *GetUserLogic) GetUser(req *types.GetUserReq) (*types.GetUserResp, error) {
    // Context already contains trace info from HTTP middleware
    ctx := l.ctx
    
    // Add custom span attributes
    span := trace.SpanFromContext(ctx)
    span.SetAttributes(
        attribute.String("user.id", strconv.FormatInt(req.UserID, 10)),
        attribute.String("user.action", "get"),
    )
    
    // Add event for debugging
    span.AddEvent("cache_lookup_start")
    
    // Check cache
    user, err := l.svcCtx.UserModel.FindOne(ctx, req.UserID)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }
    
    span.AddEvent("cache_lookup_end")
    
    return &types.GetUserResp{
        ID:    user.Id,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### ✅ Correct Pattern: Trace RPC Calls

```go
// internal/logic/createorderlogic.go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) error {
    ctx := l.ctx
    
    // RPC call with tracing (automatic in go-zero)
    userResp, err := l.svcCtx.UserRpc.GetUser(ctx, &user.GetUserReq{
        UserId: req.UserID,
    })
    if err != nil {
        return err
    }
    
    // Another RPC call
    inventoryResp, err := l.svcCtx.InventoryRpc.CheckStock(ctx, &inventory.CheckStockReq{
        ProductId: req.ProductID,
        Quantity:  req.Quantity,
    })
    
    return err
}
```

### ❌ Common Mistakes

```go
// DON'T: Lose context in goroutines
func (l *ProcessLogic) Process(req *types.ProcessReq) error {
    // ❌ Context lost in goroutine
    go func() {
        l.svcCtx.ExternalService.Call(context.Background(), data)
    }()
    
    // ✅ Pass context
    go func(ctx context.Context) {
        l.svcCtx.ExternalService.Call(ctx, data)
    }(l.ctx)
    
    return nil
}

// DON'T: Ignore tracing errors
span.RecordError(err)  // ❌ Missing span.SetStatus
span.SetStatus(codes.Error, err.Error())  // ✅ Set status
```

## Structured Logging

### ✅ Correct Pattern: Use logx

```go
// internal/logic/createorderlogic.go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) error {
    // Context-aware logging
    logx.WithContext(l.ctx).Infow("creating order",
        logx.Field("user_id", req.UserID),
        logx.Field("product_id", req.ProductID),
        logx.Field("quantity", req.Quantity),
    )
    
    orderID, err := l.createOrder(req)
    if err != nil {
        logx.WithContext(l.ctx).Errorw("failed to create order",
            logx.Field("error", err.Error()),
            logx.Field("user_id", req.UserID),
        )
        return err
    }
    
    logx.WithContext(l.ctx).Infow("order created successfully",
        logx.Field("order_id", orderID),
    )
    
    return nil
}
```

### ✅ Correct Pattern: Log Configuration

```yaml
# etc/service.yaml
Name: user-api

Log:
  ServiceName: user-api
  Mode: console    # console, file, volume
  Encoding: json   # json, plain
  Level: info      # debug, info, error, severe
  Path: logs       # Log file path (for file mode)
  KeepDays: 7      # Days to keep log files
  Compress: true   # Compress old logs
```

### ❌ Common Mistakes

```go
// DON'T: Use fmt.Println or log.Println
fmt.Println("creating order")  // ❌ No structure, no context
log.Println("creating order")  // ❌ Not integrated with logx

// DON'T: Log sensitive data
logx.Infow("user login",
    logx.Field("password", req.Password),  // ❌ Security risk
)

// DON'T: Use string formatting in logs
logx.Infof("creating order for user %d", req.UserID)  // ❌ Less efficient
logx.Infow("creating order", logx.Field("user_id", req.UserID))  // ✅ Better
```

## Grafana Dashboard Configuration

### ✅ Recommended Dashboard Panels

```json
{
  "panels": [
    {
      "title": "Request Rate (QPS)",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[1m])) by (service)"
        }
      ]
    },
    {
      "title": "P99 Latency",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[1m])) by (le, service))"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=\"error\"}[1m])) by (service)"
        }
      ]
    },
    {
      "title": "Active Connections",
      "type": "gauge",
      "targets": [
        {
          "expr": "active_connections"
        }
      ]
    }
  ]
}
```

## Alerting Rules

### ✅ Prometheus Alert Rules

```yaml
# prometheus_rules.yml
groups:
  - name: gozero-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status="error"}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: High error rate detected
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: High P99 latency
          description: "P99 latency is {{ $value | humanizeDuration }}"

      - alert: ServiceDown
        expr: up{job="gozero-services"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: Service is down
          description: "{{ $labels.instance }} is down"
```

## ELK Integration

### ✅ Correct Pattern: Log Format for ELK

```yaml
# etc/service.yaml
Log:
  ServiceName: user-api
  Encoding: json  # Required for ELK
  Level: info
```

```go
// Logs will be in JSON format, ready for Logstash
// Example output:
// {"level":"info","ts":"2024-01-15T10:30:00Z","caller":"logic.go:42","msg":"creating order","service":"user-api","user_id":123,"trace_id":"abc123"}
```

### ✅ Filebeat Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/gozero/*.log
    json.keys_under_root: true
    json.add_error_key: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "gozero-%{+yyyy.MM.dd}"
```

## ServiceMonitor (Kubernetes)

### ✅ Correct Pattern: ServiceMonitor for Prometheus Operator

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: user-api
  labels:
    app: user-api
spec:
  selector:
    matchLabels:
      app: user-api
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
---
apiVersion: v1
kind: Service
metadata:
  name: user-api
  labels:
    app: user-api
spec:
  ports:
    - name: http
      port: 8888
      targetPort: 8888
    - name: metrics
      port: 9091
      targetPort: 9091
```

## Health Checks

### ✅ Correct Pattern: Health Endpoint

```go
// go-zero provides built-in health check
// Access: GET /healthz

// Custom health check
func (h *HealthHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Check database
    if err := h.svcCtx.DB.Ping(); err != nil {
        httpx.ErrorCtx(r.Context(), w, errors.New("database unhealthy"))
        return
    }
    
    // Check Redis
    if err := h.svcCtx.Redis.Ping(); err != nil {
        httpx.ErrorCtx(r.Context(), w, errors.New("redis unhealthy"))
        return
    }
    
    httpx.OkJsonCtx(r.Context(), w, map[string]string{"status": "healthy"})
}
```

## Best Practices

### ✅ Always Follow

- Enable metrics for all services in production
- Use consistent label names across services
- Set appropriate sample rates (1.0 for dev, 0.1 for high-traffic production)
- Include trace ID in all logs for correlation
- Create alerts for critical metrics (error rate, latency, availability)
- Use structured logging (JSON) for ELK integration

### ❌ Never Do

- Use high-cardinality labels in Prometheus (user_id, request_id)
- Sample at 100% in high-traffic production (use 0.01 - 0.1)
- Ignore tracing context propagation
- Log sensitive information (passwords, tokens, PII)
- Create too many custom metrics (cardinality explosion)
- Skip health checks for critical dependencies

## Quick Reference

```bash
# Check Prometheus targets
curl http://prometheus:9090/api/v1/targets

# Check metrics endpoint
curl http://localhost:9091/metrics

# Query metrics
curl 'http://prometheus:9090/api/v1/query?query=http_requests_total'

# Check trace in Jaeger
# Open http://jaeger:16686
```