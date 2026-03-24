# Advanced Components

## Overview

go-zero provides several powerful components in the `core` package for performance optimization and concurrent processing.

| Component | Package | Use Case |
|-----------|---------|----------|
| **Bloom Filter** | `core/bloom` | Probabilistic set membership, prevent cache penetration |
| **Timing Wheel** | `core/collection` | Delayed task scheduling, timeouts |
| **MapReduce** | `core/mr` | Parallel data processing, concurrent API calls |
| **Executors** | `core/executors` | Batch task processing, buffering |
| **SharedCalls** | `core/syncx` | Single flight, prevent duplicate requests |
| **Stream** | `core/fx` | Functional stream processing |

## Bloom Filter

Bloom filter is a space-efficient probabilistic data structure for testing set membership. It can have false positives but no false negatives.

### When to Use

- Prevent cache penetration (check if key exists before querying DB)
- Spam email filtering
- Check if record exists (reduce disk access)
- Web crawler URL deduplication

### ✅ Correct Pattern

```go
package main

import (
    "github.com/zeromicro/go-zero/core/bloom"
    "github.com/zeromicro/go-zero/core/stores/redis"
)

func main() {
    // Initialize Redis-backed bloom filter
    store := redis.New("localhost:6379", func(r *redis.Redis) {
        r.Type = redis.NodeType
    })
    
    // Create bloom filter: key="users", bits=1<<24 (16M bits = 2MB)
    filter := bloom.New(store, "users", 1<<24)
    
    // Add items
    userIDs := []int64{1001, 1002, 1003}
    for _, id := range userIDs {
        if err := filter.Add([]byte(fmt.Sprintf("user:%d", id))); err != nil {
            logx.Errorf("add to bloom filter failed: %v", err)
        }
    }
    
    // Check membership
    exists, err := filter.Exists([]byte("user:1001"))
    if err != nil {
        logx.Errorf("check bloom filter failed: %v", err)
    }
    // exists may be true (possibly) or false (definitely not)
}
```

### ✅ Correct Pattern: Prevent Cache Penetration

```go
// internal/logic/getuserlogic.go
func (l *GetUserLogic) GetUser(req *types.GetUserReq) (*types.User, error) {
    key := fmt.Sprintf("user:%d", req.UserID)
    
    // Check bloom filter first
    exists, err := l.svcCtx.BloomFilter.Exists([]byte(key))
    if err != nil {
        logx.Errorf("bloom filter check failed: %v", err)
        // Fall through to DB check (fail open)
    } else if !exists {
        // Definitely not in DB, return early
        return nil, errors.New("user not found")
    }
    
    // Check cache
    cached, err := l.svcCtx.Redis.Get(key)
    if err == nil && cached != "" {
        var user types.User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }
    
    // Query database
    user, err := l.svcCtx.UserModel.FindOne(l.ctx, req.UserID)
    if err != nil {
        return nil, err
    }
    
    // Cache result
    data, _ := json.Marshal(user)
    l.svcCtx.Redis.Setex(key, string(data), 3600)
    
    return user, nil
}
```

### ❌ Common Mistakes

```go
// DON'T: Use bloom filter for exact membership (false positives possible)
if exists, _ := filter.Exists(key); exists {
    // ❌ This might be a false positive!
    return getItem(key)  // Could return nil
}

// ✅ Use as a first-level filter, then verify
if exists, _ := filter.Exists(key); !exists {
    // Definitely not exists
    return nil, ErrNotFound
}
// Might exist, check cache/DB
return getItem(key)

// DON'T: Use too few bits (high false positive rate)
filter := bloom.New(store, "users", 1024)  // ❌ Too small

// ✅ Calculate appropriate size
// For 1M items with 1% false positive rate: ~9.6M bits
filter := bloom.New(store, "users", 10_000_000)
```

## TimingWheel

Timing wheel is an efficient data structure for managing timed tasks. Better performance than `time.Timer` for large numbers of timers.

### When to Use

- Cache key expiration
- Connection timeouts
- Delayed task scheduling
- Session timeout management

### ✅ Correct Pattern

```go
package main

import (
    "time"
    "github.com/zeromicro/go-zero/core/collection"
)

func main() {
    // Create timing wheel: interval=1s, slots=3600 (1 hour capacity)
    tw, err := collection.NewTimingWheel(time.Second, 3600, func(key, value interface{}) {
        fmt.Printf("expired: key=%v, value=%v\n", key, value)
    })
    if err != nil {
        panic(err)
    }
    
    // Start the timing wheel
    tw.Start()
    defer tw.Stop()
    
    // Set a timer: expire after 10 seconds
    tw.SetTimer("order:123", "order_data", time.Second*10)
    
    // Move timer: extend expiration
    tw.MoveTimer("order:123", time.Second*30)
    
    // Remove timer
    tw.RemoveTimer("order:123")
}
```

### ✅ Correct Pattern: Session Management

```go
// internal/svc/servicecontext.go
type ServiceContext struct {
    Config      config.Config
    sessions    *collection.Cache
    timingWheel *collection.TimingWheel
}

func NewServiceContext(c config.Config) *ServiceContext {
    tw, _ := collection.NewTimingWheel(time.Second, 3600, func(key, value interface{}) {
        // Called when session expires
        sessionID := key.(string)
        logx.Infof("session expired: %s", sessionID)
    })
    tw.Start()
    
    return &ServiceContext{
        Config:      c,
        timingWheel: tw,
    }
}

// internal/logic/sessionlogic.go
func (l *SessionLogic) CreateSession(userID int64) (string, error) {
    sessionID := generateSessionID()
    
    // Set session with 30-minute expiration
    l.svcCtx.timingWheel.SetTimer(sessionID, userID, 30*time.Minute)
    
    return sessionID, nil
}

func (l *SessionLogic) RefreshSession(sessionID string) error {
    // Extend session by 30 minutes
    l.svcCtx.timingWheel.MoveTimer(sessionID, 30*time.Minute)
    return nil
}
```

### ❌ Common Mistakes

```go
// DON'T: Forget to start the timing wheel
tw, _ := collection.NewTimingWheel(time.Second, 3600, handler)
tw.SetTimer("key", "value", time.Second*10)  // ❌ Never fires

// ✅ Always start
tw.Start()
defer tw.Stop()
tw.SetTimer("key", "value", time.Second*10)

// DON'T: Use too few slots for long durations
tw, _ := collection.NewTimingWheel(time.Second, 60, handler)  // ❌ Only 1 minute capacity
tw.SetTimer("key", "value", time.Hour)  // ❌ Won't work properly

// ✅ Use appropriate slots
tw, _ := collection.NewTimingWheel(time.Second, 3600, handler)  // 1 hour capacity
```

## MapReduce

MapReduce is a concurrent processing tool for parallel execution of tasks, inspired by Google's MapReduce.

### When to Use

- Aggregate data from multiple services (product detail = user + inventory + orders)
- Batch processing with error handling
- Parallel API calls with timeout

### ✅ Correct Pattern: Parallel API Calls

```go
// internal/logic/productdetaillogic.go
type ProductDetail struct {
    User     *UserInfo
    Store    *StoreInfo
    Order    *OrderInfo
}

func (l *ProductDetailLogic) GetProductDetail(userID, productID int64) (*ProductDetail, error) {
    var detail ProductDetail
    
    // Run all API calls in parallel
    err := mr.Finish(func() (err error) {
        detail.User, err = l.svcCtx.UserRpc.GetUser(l.ctx, &user.GetReq{UserId: userID})
        return
    }, func() (err error) {
        detail.Store, err = l.svcCtx.StoreRpc.GetStore(l.ctx, &store.GetReq{ProductId: productID})
        return
    }, func() (err error) {
        detail.Order, err = l.svcCtx.OrderRpc.GetOrder(l.ctx, &order.GetReq{ProductId: productID})
        return
    })
    
    if err != nil {
        return nil, err
    }
    
    return &detail, nil
}
```

### ✅ Correct Pattern: Batch Processing with MapReduce

```go
// internal/logic/batchchecklogic.go
func (l *BatchCheckLogic) CheckUsers(userIDs []int64) ([]int64, error) {
    // MapReduce: map = validate each user, reduce = collect valid IDs
    result, err := mr.MapReduce(
        // Generate: send userIDs to channel
        func(source chan<- interface{}) {
            for _, id := range userIDs {
                source <- id
            }
        },
        // Mapper: check each user in parallel
        func(item interface{}, writer mr.Writer, cancel func(error)) {
            userID := item.(int64)
            valid, err := l.checkUser(userID)
            if err != nil {
                cancel(err)  // Cancel all on error
                return
            }
            if valid {
                writer.Write(userID)
            }
        },
        // Reducer: collect results
        func(pipe <-chan interface{}, writer mr.Writer, cancel func(error)) {
            var validIDs []int64
            for id := range pipe {
                validIDs = append(validIDs, id.(int64))
            }
            writer.Write(validIDs)
        },
        mr.WithWorkers(16),  // Max concurrent workers
    )
    
    if err != nil {
        return nil, err
    }
    
    return result.([]int64), nil
}

func (l *BatchCheckLogic) checkUser(userID int64) (bool, error) {
    // Simulate check
    return userID > 0, nil
}
```

### ❌ Common Mistakes

```go
// DON'T: Forget to cancel on error
func(item interface{}, writer mr.Writer, cancel func(error)) {
    if err := process(item); err != nil {
        // ❌ Error ignored, other mappers continue
    }
}

// ✅ Cancel on error
func(item interface{}, writer mr.Writer, cancel func(error)) {
    if err := process(item); err != nil {
        cancel(err)  // Stop all processing
        return
    }
}

// DON'T: Forget to write result in reducer
func(pipe <-chan interface{}, writer mr.Writer, cancel func(error)) {
    var results []string
    for item := range pipe {
        results = append(results, item.(string))
    }
    // ❌ Missing writer.Write, returns ErrReduceNoOutput
}

// ✅ Write result
func(pipe <-chan interface{}, writer mr.Writer, cancel func(error)) {
    var results []string
    for item := range pipe {
        results = append(results, item.(string))
    }
    writer.Write(results)
}
```

## Executors

Executors provide batch task processing with buffering and delayed execution.

### When to Use

- Batch insert to ClickHouse
- Batch SQL operations
- Buffer tasks before commit
- Throttled task execution

### ✅ Correct Pattern: BulkExecutor

```go
// internal/logic/clickhouselogic.go
type ClickHouseInserter struct {
    executor *executors.BulkExecutor
}

func NewClickHouseInserter(db *sql.DB) *ClickHouseInserter {
    inserter := &ClickHouseInserter{}
    
    inserter.executor = executors.NewBulkExecutor(
        // Execute function: batch insert
        func(tasks []interface{}) error {
            if len(tasks) == 0 {
                return nil
            }
            
            // Build batch insert
            var builder strings.Builder
            builder.WriteString("INSERT INTO events (user_id, event_type, created_at) VALUES ")
            
            for i, task := range tasks {
                if i > 0 {
                    builder.WriteString(",")
                }
                event := task.(*Event)
                builder.WriteString(fmt.Sprintf("(%d,'%s','%s')",
                    event.UserID, event.Type, event.CreatedAt))
            }
            
            _, err := db.Exec(builder.String())
            return err
        },
        executors.WithBulkInterval(time.Second*3),  // Flush every 3s
        executors.WithBulkTasks(10240),             // Max 10240 tasks per batch
    )
    
    return inserter
}

func (i *ClickHouseInserter) AddEvent(event *Event) error {
    return i.executor.Add(event)
}

func (i *ClickHouseInserter) Close() error {
    i.executor.Flush()
    i.executor.Wait()
    return nil
}
```

### ✅ Correct Pattern: ChunkExecutor (Size-based)

```go
// For size-based batching (e.g., max 1MB per request)
executor := executors.NewChunkExecutor(
    func(tasks []interface{}) error {
        return sendBatch(tasks)
    },
    executors.WithChunkBytes(1024*1024),  // Max 1MB per batch
)
```

### ❌ Common Mistakes

```go
// DON'T: Forget to flush before exit
executor.Add(task1)
executor.Add(task2)
// ❌ Application exits without flushing, tasks lost

// ✅ Always flush and wait
executor.Add(task1)
executor.Add(task2)
executor.Flush()
executor.Wait()

// DON'T: Ignore Add errors
executor.Add(task)  // ❌ Error ignored

// ✅ Handle errors
if err := executor.Add(task); err != nil {
    logx.Errorf("add task failed: %v", err)
}
```

## SharedCalls (SingleFlight)

SharedCalls prevents duplicate function execution when multiple goroutines call the same function simultaneously.

### When to Use

- Cache stampede prevention
- Prevent duplicate DB queries
- Rate-limited API calls

### ✅ Correct Pattern: Prevent Cache Stampede

```go
// internal/svc/servicecontext.go
type ServiceContext struct {
    Config     config.Config
    UserModel  UserModel
    SharedCalls syncx.SingleFlight
}

// internal/logic/getuserlogic.go
func (l *GetUserLogic) GetUser(req *types.GetUserReq) (*types.User, error) {
    key := fmt.Sprintf("user:%d", req.UserID)
    
    // Use SharedCalls to prevent cache stampede
    val, err := l.svcCtx.SharedCalls.Do(key, func() (interface{}, error) {
        // Check cache first
        cached, err := l.svcCtx.Redis.Get(key)
        if err == nil && cached != "" {
            var user types.User
            json.Unmarshal([]byte(cached), &user)
            return &user, nil
        }
        
        // Query database (only one goroutine executes this)
        user, err := l.svcCtx.UserModel.FindOne(l.ctx, req.UserID)
        if err != nil {
            return nil, err
        }
        
        // Cache the result
        data, _ := json.Marshal(user)
        l.svcCtx.Redis.Setex(key, string(data), 3600)
        
        return user, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    return val.(*types.User), nil
}
```

### ✅ Correct Pattern: ExpiringSharedCalls

```go
// With expiration for long-running operations
calls := syncx.NewSingleFlight()

val, err := calls.DoEx("expensive_key", func() (interface{}, error) {
    return expensiveOperation()
}, time.Minute)  // Cache result for 1 minute
```

### ❌ Common Mistakes

```go
// DON'T: Use different keys for the same resource
key1 := fmt.Sprintf("user:%d", userID)
key2 := fmt.Sprintf("user:%d", userID)  // Same user, same key
// ✅ Use consistent key generation

// DON'T: Forget error handling in Do callback
val, err := calls.Do(key, func() (interface{}, error) {
    // ❌ Panic here will affect all waiting goroutines
    return riskyOperation()
})

// ✅ Handle errors gracefully
val, err := calls.Do(key, func() (interface{}, error) {
    result, err := riskyOperation()
    if err != nil {
        return nil, err  // Return error, don't panic
    }
    return result, nil
})
```

## Quick Reference

```go
// Bloom Filter: Probabilistic membership test
filter := bloom.New(redis, "key", bits)
filter.Add([]byte("item"))
exists, _ := filter.Exists([]byte("item"))

// TimingWheel: Delayed task scheduling
tw, _ := collection.NewTimingWheel(interval, slots, onExpire)
tw.Start()
tw.SetTimer("key", value, delay)

// MapReduce: Parallel processing
mr.Finish(fn1, fn2, fn3)  // Run in parallel
mr.MapReduce(generate, mapper, reducer)  // Full MapReduce

// BulkExecutor: Batch task processing
executor := executors.NewBulkExecutor(execute, opts...)
executor.Add(task)
executor.Flush()
executor.Wait()

// SharedCalls: Prevent duplicate execution
val, err := singleFlight.Do(key, fn)
```

## Best Practices

### ✅ Always Follow

- Choose the right component for the use case
- Initialize components in ServiceContext
- Handle errors in callbacks
- Clean up resources (Stop(), Wait(), Flush())

### ❌ Never Do

- Use bloom filter for exact membership
- Forget to start timing wheel
- Ignore errors in MapReduce mappers
- Skip Flush/Wait before shutdown