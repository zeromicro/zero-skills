# Message Queue Patterns

## Overview

go-zero provides [go-queue](https://github.com/zeromicro/go-queue) for distributed task processing and message queuing. It supports two modes:

| Mode | Backend | Use Case |
|------|---------|----------|
| **dq** | Beanstalkd + Redis | Delayed tasks, scheduled jobs, at-least-once delivery |
| **kq** | Kafka | High-throughput messaging, event streaming |

## dq: Delayed Task Queue

### When to Use dq

- Scheduled tasks (daily reports, cleanup jobs)
- Delayed execution (order timeout, reminder notifications)
- Distributed task coordination
- Tasks requiring persistence and recovery

### ✅ Correct Pattern: Configuration

```yaml
# etc/job.yaml
Name: job-service

Log:
  ServiceName: job-service
  Level: info

DqConf:
  Beanstalks:
    - Endpoint: beanstalkd:7771
      Tube: orders
    - Endpoint: beanstalkd:7772
      Tube: orders
  Redis:
    Host: redis:6379
    Type: node
    Pass: ""  # optional
```

```go
// internal/config/config.go
type Config struct {
    service.ServiceConf
    DqConf dq.DqConf
}
```

### ✅ Correct Pattern: Producer

```go
// internal/logic/orderproducerlogic.go
package logic

import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/zeromicro/go-queue/dq"
    "github.com/zeromicro/go-zero/core/logx"
    "job/internal/svc"
)

type OrderProducer struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
    producer dq.Producer
}

func NewOrderProducerLogic(ctx context.Context, svcCtx *svc.ServiceContext) *OrderProducer {
    return &OrderProducer{
        ctx:      ctx,
        svcCtx:   svcCtx,
        Logger:   logx.WithContext(ctx),
        producer: dq.NewProducer([]dq.Beanstalk{
            {Endpoint: "beanstalkd:7771", Tube: "orders"},
            {Endpoint: "beanstalkd:7772", Tube: "orders"},
        }),
    }
}

func (p *OrderProducer) ScheduleOrderTimeout(orderID int64) error {
    // Create task payload
    task := map[string]interface{}{
        "type":     "order_timeout",
        "order_id": orderID,
        "created":  time.Now().Unix(),
    }
    
    data, err := json.Marshal(task)
    if err != nil {
        return err
    }
    
    // Delay for 30 minutes
    delay := 30 * time.Minute
    _, err = p.producer.Delay(data, delay)
    
    if err != nil {
        logx.WithContext(p.ctx).Errorw("failed to schedule order timeout",
            logx.Field("order_id", orderID),
            logx.Field("error", err.Error()),
        )
        return err
    }
    
    logx.WithContext(p.ctx).Infow("scheduled order timeout",
        logx.Field("order_id", orderID),
        logx.Field("delay", delay),
    )
    
    return nil
}

// Immediate execution
func (p *OrderProducer) SendImmediate(taskType string, data interface{}) error {
    payload, err := json.Marshal(map[string]interface{}{
        "type": taskType,
        "data": data,
    })
    if err != nil {
        return err
    }
    
    return p.producer.At(payload, time.Now())
}
```

### ✅ Correct Pattern: Consumer

```go
// internal/logic/orderconsumerlogic.go
package logic

import (
    "context"
    "encoding/json"
    
    "github.com/zeromicro/go-queue/dq"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/threading"
    "job/internal/svc"
)

type OrderConsumer struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
}

func NewOrderConsumerLogic(ctx context.Context, svcCtx *svc.ServiceContext) *OrderConsumer {
    return &OrderConsumer{
        ctx:    ctx,
        svcCtx: svcCtx,
        Logger: logx.WithContext(ctx),
    }
}

func (c *OrderConsumer) Start() {
    logx.Info("starting order consumer")
    
    threading.GoSafe(func() {
        c.svcCtx.Consumer.Consume(c.handleMessage)
    })
}

func (c *OrderConsumer) Stop() {
    logx.Info("stopping order consumer")
}

func (c *OrderConsumer) handleMessage(body []byte) {
    var task struct {
        Type    string          `json:"type"`
        OrderID int64           `json:"order_id"`
        Data    json.RawMessage `json:"data"`
    }
    
    if err := json.Unmarshal(body, &task); err != nil {
        logx.Errorf("failed to unmarshal task: %v", err)
        return
    }
    
    logx.Infow("processing task",
        logx.Field("type", task.Type),
        logx.Field("order_id", task.OrderID),
    )
    
    switch task.Type {
    case "order_timeout":
        c.handleOrderTimeout(task.OrderID)
    case "order_reminder":
        c.handleOrderReminder(task.OrderID)
    default:
        logx.Errorf("unknown task type: %s", task.Type)
    }
}

func (c *OrderConsumer) handleOrderTimeout(orderID int64) {
    // Check order status
    order, err := c.svcCtx.OrderModel.FindOne(c.ctx, orderID)
    if err != nil {
        logx.Errorf("failed to find order: %v", err)
        return
    }
    
    // Cancel if still pending
    if order.Status == "pending" {
        err = c.svcCtx.OrderModel.UpdateStatus(c.ctx, orderID, "cancelled")
        if err != nil {
            logx.Errorf("failed to cancel order: %v", err)
        } else {
            logx.Infof("order %d cancelled due to timeout", orderID)
        }
    }
}

func (c *OrderConsumer) handleOrderReminder(orderID int64) {
    // Send reminder notification
    // ...
}
```

### ✅ Correct Pattern: Service Registration

```go
// internal/handler/router.go
package handler

import (
    "context"
    "github.com/zeromicro/go-zero/core/service"
    "job/internal/logic"
    "job/internal/svc"
)

func RegisterJobs(svcCtx *svc.ServiceContext, group *service.ServiceGroup) {
    group.Add(logic.NewOrderConsumerLogic(context.Background(), svcCtx))
    group.Add(logic.NewNotificationProducerLogic(context.Background(), svcCtx))
}
```

```go
// internal/svc/servicecontext.go
package svc

import (
    "github.com/zeromicro/go-queue/dq"
    "job/internal/config"
)

type ServiceContext struct {
    Config   config.Config
    Consumer dq.Consumer
}

func NewServiceContext(c config.Config) *ServiceContext {
    return &ServiceContext{
        Config:   c,
        Consumer: dq.NewConsumer(c.DqConf),
    }
}
```

```go
// main.go
package main

import (
    "flag"
    "os"
    "os/signal"
    "syscall"
    
    "github.com/zeromicro/go-zero/core/conf"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/service"
    "job/internal/config"
    "job/internal/handler"
    "job/internal/svc"
)

var configFile = flag.String("f", "etc/job.yaml", "config file")

func main() {
    flag.Parse()
    
    var c config.Config
    conf.MustLoad(*configFile, &c)
    
    svcCtx := svc.NewServiceContext(c)
    
    group := service.NewServiceGroup()
    handler.RegisterJobs(svcCtx, group)
    
    // Handle shutdown signals
    ch := make(chan os.Signal, 1)
    signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
    
    go func() {
        sig := <-ch
        logx.Infof("received signal: %s", sig)
        group.Stop()
    }()
    
    group.Start()
}
```

### ❌ Common Mistakes

```go
// DON'T: Block in consumer handler
func (c *Consumer) handleMessage(body []byte) {
    // ❌ Long-running operation blocks other messages
    time.Sleep(5 * time.Minute)
    c.process(body)
}

// ✅ Use goroutine for long operations
func (c *Consumer) handleMessage(body []byte) {
    go func() {
        c.process(body)
    }()
}

// DON'T: Panic in handler (crashes consumer)
func (c *Consumer) handleMessage(body []byte) {
    // ❌ Panic will crash the consumer
    if err := json.Unmarshal(body, &task); err != nil {
        panic(err)
    }
}

// ✅ Handle errors gracefully
func (c *Consumer) handleMessage(body []byte) {
    if err := json.Unmarshal(body, &task); err != nil {
        logx.Errorf("unmarshal error: %v", err)
        return  // Log and continue
    }
}

// DON'T: Skip Redis configuration (leads to duplicate consumption)
// dq uses Redis for deduplication with multiple Beanstalkd instances
```

## kq: Kafka Message Queue

### When to Use kq

- High-throughput event streaming
- Real-time data pipelines
- Event-driven architectures
- Log aggregation

### ✅ Correct Pattern: Configuration

```yaml
# etc/kafka-consumer.yaml
Name: kafka-consumer

KqConsumerConf:
  Name: order-events
  Brokers:
    - kafka1:9092
    - kafka2:9092
  Topic: orders
  Group: order-processor
  Offset: first   # first, last
  Conns: 1
  Consumers: 8
  Processors: 16
```

```go
// internal/config/config.go
type Config struct {
    service.ServiceConf
    KqConsumerConf kq.KqConf
}
```

### ✅ Correct Pattern: Kafka Consumer

```go
// internal/logic/kafkaconsumerlogic.go
package logic

import (
    "context"
    "encoding/json"
    
    "github.com/zeromicro/go-queue/kq"
    "github.com/zeromicro/go-zero/core/logx"
    "github.com/zeromicro/go-zero/core/service"
    "kafka-consumer/internal/svc"
)

type KafkaConsumer struct {
    ctx    context.Context
    svcCtx *svc.ServiceContext
    logx.Logger
    q      *kq.KafkaQueue
}

func NewKafkaConsumerLogic(ctx context.Context, svcCtx *svc.ServiceContext) *KafkaConsumer {
    return &KafkaConsumer{
        ctx:    ctx,
        svcCtx: svcCtx,
        Logger: logx.WithContext(ctx),
    }
}

func (k *KafkaConsumer) Start() {
    logx.Info("starting kafka consumer")
    
    k.q = kq.MustNewQueue(k.svcCtx.Config.KqConsumerConf, k)
    k.q.Start()
}

func (k *KafkaConsumer) Stop() {
    logx.Info("stopping kafka consumer")
    if k.q != nil {
        k.q.Stop()
    }
}

// Consume implements kq.Consumer interface
func (k *KafkaConsumer) Consume(key, val string) error {
    var event OrderEvent
    if err := json.Unmarshal([]byte(val), &event); err != nil {
        logx.Errorf("failed to unmarshal event: %v", err)
        return nil // Skip invalid messages
    }
    
    logx.Infow("processing order event",
        logx.Field("event_type", event.Type),
        logx.Field("order_id", event.OrderID),
    )
    
    switch event.Type {
    case "order_created":
        return k.handleOrderCreated(&event)
    case "order_cancelled":
        return k.handleOrderCancelled(&event)
    }
    
    return nil
}
```

### ✅ Correct Pattern: Kafka Producer

```go
// internal/logic/kafkaproducerlogic.go
package logic

import (
    "context"
    "encoding/json"
    
    "github.com/zeromicro/go-queue/kq"
    "github.com/zeromicro/go-zero/core/logx"
)

type KafkaProducer struct {
    producer *kq.Producer
    logx.Logger
}

func NewKafkaProducer(brokers []string, topic string) *KafkaProducer {
    return &KafkaProducer{
        producer: kq.NewProducer(kq.KqConf{
            Brokers: brokers,
            Topic:   topic,
        }),
        Logger: logx.NewLogx(),
    }
}

func (p *KafkaProducer) PublishOrderEvent(event *OrderEvent) error {
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    return p.producer.Publish(string(event.OrderID), string(data))
}
```

## Delay Queue Patterns

### ✅ Correct Pattern: Multi-Stage Delay

```go
// Order reminder: 10 min before, at expiration time
func (p *OrderProducer) ScheduleOrderReminders(orderID int64, expireTime time.Time) error {
    // First reminder: 10 minutes before expiration
    firstReminder := expireTime.Add(-10 * time.Minute)
    if firstReminder.After(time.Now()) {
        task := OrderTask{
            Type:    "order_reminder",
            OrderID: orderID,
            Stage:   "first",
        }
        data, _ := json.Marshal(task)
        p.producer.At(data, firstReminder)
    }
    
    // Final notification: at expiration
    task := OrderTask{
        Type:    "order_expire",
        OrderID: orderID,
        Stage:   "final",
    }
    data, _ := json.Marshal(task)
    return p.producer.At(data, expireTime)
}
```

### ✅ Correct Pattern: Retry with Exponential Backoff

```go
func (p *Producer) ScheduleRetry(taskType string, data interface{}, attempt int) error {
    if attempt > 5 {
        return errors.New("max retries exceeded")
    }
    
    // Exponential backoff: 1s, 2s, 4s, 8s, 16s
    delay := time.Second * time.Duration(1<<attempt)
    
    payload := RetryTask{
        Type:    taskType,
        Data:    data,
        Attempt: attempt + 1,
    }
    
    body, _ := json.Marshal(payload)
    return p.producer.Delay(body, delay)
}
```

## Comparison: dq vs kq

| Feature | dq | kq |
|---------|-----|-----|
| **Backend** | Beanstalkd | Kafka |
| **Persistence** | Yes (disk) | Yes (disk) |
| **Delayed Tasks** | ✅ Native | ❌ Requires implementation |
| **Throughput** | Medium | Very High |
| **Ordering** | Per tube | Per partition |
| **Replay** | Limited | ✅ Full replay |
| **Deduplication** | Redis-based | Consumer group-based |
| **Best For** | Scheduled jobs, delays | Event streaming, high throughput |

## Best Practices

### ✅ Always Follow

- Use multiple Beanstalkd instances for HA (dq)
- Configure Redis for deduplication (dq)
- Handle panics in consumer handlers
- Log all task processing for debugging
- Implement idempotent handlers (messages may be delivered more than once)
- Use appropriate consumer/processor counts for kq

### ❌ Never Do

- Block indefinitely in message handlers
- Skip error handling (errors will cause message re-delivery)
- Process without idempotency (duplicates possible)
- Use synchronous operations in async handlers
- Ignore graceful shutdown (may lose in-flight messages)

## Troubleshooting

### Beanstalkd Connection Issues

```bash
# Check beanstalkd status
telnet beanstalkd 7771
# Stats
stats

# List tubes
list-tubes
```

### Kafka Issues

```bash
# Check consumer group lag
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --describe --group order-processor

# Reset consumer offset
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group order-processor --reset-offsets --to-earliest --execute
```

## References

- [go-queue GitHub](https://github.com/zeromicro/go-queue)
- [go-zero go-queue Documentation](https://go-zero.dev/docs/go-queue)
- [Beanstalkd Protocol](https://beanstalkd.github.io/)