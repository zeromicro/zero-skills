# Collection Patterns

go-zero 的 `core/collection` 包提供了一系列经过生产环境验证的高性能数据结构。

## 概述

| 数据结构 | 用途 | 时间复杂度 |
|---------|------|-----------|
| LRU Cache | 本地缓存热点数据 | O(1) 查找/更新 |
| Ring Buffer | 固定大小的循环缓冲区 | O(1) 添加/读取 |
| TimingWheel | 高效定时器管理 | O(1) 添加/删除/执行 |
| SafeMap | 并发安全的通用 Map | O(1) 查找/更新/删除 |

---

## LRU Cache

LRU（Least Recently Used）缓存淘汰策略：当缓存满时，优先淘汰最久未使用的数据。

### ✅ 基本使用

```go
import "github.com/zeromicro/go-zero/core/collection"

// 创建容量为 100 的缓存
cache, err := collection.NewCache(100)
if err != nil {
    log.Fatal(err)
}

// 设置值
cache.Set("user:1", userData)

// 获取值
if val, ok := cache.Get("user:1"); ok {
    user := val.(*User)
}

// 删除值
cache.Del("user:1")

// 获取缓存大小
size := cache.Size()
```

### ✅ 带淘汰回调

```go
cache, err := collection.NewCache(100, collection.WithEvict(func(key string, value interface{}) {
    // 清理资源
    if closer, ok := value.(io.Closer); ok {
        closer.Close()
    }
    log.Printf("淘汰: %s", key)
}))
```

### ✅ 在 ServiceContext 中使用

```go
type ServiceContext struct {
    Config     config.Config
    UserCache  *collection.Cache
    UsersModel model.UsersModel
}

func NewServiceContext(c config.Config) *ServiceContext {
    cache, _ := collection.NewCache(1000, collection.WithEvict(func(key string, value interface{}) {
        logx.Infof("cache evicted: %s", key)
    }))

    return &ServiceContext{
        Config:    c,
        UserCache: cache,
        UsersModel: model.NewUsersModel(sqlx.NewMysql(c.DataSource), c.Cache),
    }
}
```

### ✅ Logic 层使用缓存

```go
func (l *GetUserLogic) GetUser(req *types.GetUserRequest) (*types.GetUserResponse, error) {
    cacheKey := fmt.Sprintf("user:%d", req.Id)

    // 先查本地缓存
    if val, ok := l.svcCtx.UserCache.Get(cacheKey); ok {
        user := val.(*model.Users)
        return &types.GetUserResponse{
            Id:    user.Id,
            Name:  user.Name,
            Email: user.Email,
        }, nil
    }

    // 本地缓存未命中，查数据库
    user, err := l.svcCtx.UsersModel.FindOne(l.ctx, req.Id)
    if err != nil {
        return nil, err
    }

    // 写入本地缓存
    l.svcCtx.UserCache.Set(cacheKey, user)

    return &types.GetUserResponse{
        Id:    user.Id,
        Name:  user.Name,
        Email: user.Email,
    }, nil
}
```

### ❌ 常见错误

```go
// 错误：缓存容量过大导致内存溢出
cache, _ := collection.NewCache(10000000)  // 太大！

// 错误：缓存持有资源但不清理
cache.Set("conn", dbConn)  // 连接被缓存，无法释放

// 错误：缓存键冲突
cache.Set("user", userA)
cache.Set("user", userB)  // 覆盖了 userA
```

### 最佳实践

- 根据内存和命中率设置合理的缓存大小
- 淘汰回调中的耗时操作应异步执行
- 使用有意义的键名避免冲突
- 缓存的对象应尽量小且不可变

---

## Ring Buffer

环形队列是一种固定大小的队列，新元素会覆盖最旧的元素。

### ✅ 基本使用

```go
import "github.com/zeromicro/go-zero/core/collection"

// 创建保留最近 100 条记录的环形队列
ring := collection.NewRing(100)

// 添加元素
ring.Add(logEntry1)
ring.Add(logEntry2)

// 获取所有元素
items := ring.Take()
for _, item := range items {
    log.Printf("%v", item)
}
```

### ✅ 日志缓冲区模式

```go
type LogBuffer struct {
    ring *collection.Ring
    mu   sync.RWMutex
}

func NewLogBuffer(size int) *LogBuffer {
    return &LogBuffer{
        ring: collection.NewRing(size),
    }
}

func (lb *LogBuffer) AddLog(entry LogEntry) {
    lb.mu.Lock()
    defer lb.mu.Unlock()
    lb.ring.Add(entry)
}

func (lb *LogBuffer) GetRecentLogs() []LogEntry {
    lb.mu.RLock()
    defer lb.mu.RUnlock()

    items := lb.ring.Take()
    logs := make([]LogEntry, 0, len(items))
    for _, item := range items {
        logs = append(logs, item.(LogEntry))
    }
    return logs
}
```

### ✅ 固定窗口统计

```go
type MetricsCollector struct {
    requestTimes *collection.Ring
}

func NewMetricsCollector(windowSize int) *MetricsCollector {
    return &MetricsCollector{
        requestTimes: collection.NewRing(windowSize),
    }
}

func (mc *MetricsCollector) RecordRequest(duration time.Duration) {
    mc.requestTimes.Add(duration)
}

func (mc *MetricsCollector) GetAverageRequestTime() time.Duration {
    items := mc.requestTimes.Take()
    if len(items) == 0 {
        return 0
    }

    var total time.Duration
    for _, item := range items {
        total += item.(time.Duration)
    }
    return total / time.Duration(len(items))
}
```

### ❌ 常见错误

```go
// 错误：大小过大浪费内存
ring := collection.NewRing(1000000)  // 太大！

// 错误：存储大对象占用过多内存
ring.Add(largeFileContent)  // 不应该存储大对象

// 错误：假设元素顺序
items := ring.Take()
// 注意：元素顺序是从最旧到最新，不是反过来的
```

### 最佳实践

- 根据数据量设置合理大小
- 存储轻量级对象或引用
- 配合定时读取使用，避免数据堆积
- 需要顺序保证时使用额外的索引

---

## TimingWheel

时间轮是一种高效的定时器实现，适合管理大量定时任务。

### ✅ 基本使用

```go
import "github.com/zeromicro/go-zero/core/collection"

// 创建时间轮：精度 100ms，3600 个槽位（支持最大 6 分钟延迟）
tw, err := collection.NewTimingWheel(100*time.Millisecond, 3600)
if err != nil {
    log.Fatal(err)
}
defer tw.Stop()

// 设置定时器
tw.SetTimer("task-1", nil, 5*time.Second, func(key string, value interface{}) {
    log.Printf("定时器触发: %s", key)
})

// 取消定时器
tw.RemoveTimer("task-1")
```

### ✅ 参数选择

```go
// 根据最大延迟计算槽位数
maxDelay := 10 * time.Minute   // 最大支持 10 分钟延迟
interval := 100 * time.Millisecond  // 精度 100ms
numSlots := int(maxDelay / interval)  // 槽位数 = 6000

tw, _ := collection.NewTimingWheel(interval, numSlots)
```

**参数说明：**
- `interval`：时间格精度，越小精度越高但 CPU 开销越大
- `numSlots`：槽位数量，影响内存占用和最大延迟
- `最大延迟 = interval * numSlots`

### ✅ 请求超时管理

```go
type RequestManager struct {
    tw         *collection.TimingWheel
    requests   map[string]*Request
    mu         sync.RWMutex
}

func NewRequestManager() *RequestManager {
    tw, _ := collection.NewTimingWheel(100*time.Millisecond, 3600)
    return &RequestManager{
        tw:       tw,
        requests: make(map[string]*Request),
    }
}

func (rm *RequestManager) AddRequest(req *Request) {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    rm.requests[req.ID] = req

    // 设置 30 秒超时
    rm.tw.SetTimer(req.ID, nil, 30*time.Second, func(key string, value interface{}) {
        rm.handleTimeout(key)
    })
}

func (rm *RequestManager) CompleteRequest(id string) {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    delete(rm.requests, id)
    rm.tw.RemoveTimer(id)
}

func (rm *RequestManager) handleTimeout(id string) {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    if req, ok := rm.requests[id]; ok {
        req.Timeout()
        delete(rm.requests, id)
    }
}
```

### ✅ 延迟任务调度

```go
type TaskScheduler struct {
    tw *collection.TimingWheel
}

func NewTaskScheduler() *TaskScheduler {
    tw, _ := collection.NewTimingWheel(time.Second, 3600)
    return &TaskScheduler{tw: tw}
}

func (s *TaskScheduler) ScheduleTask(taskID string, delay time.Duration, task func()) {
    s.tw.SetTimer(taskID, nil, delay, func(key string, value interface{}) {
        // 异步执行耗时任务，避免阻塞时间轮
        go task()
    })
}

func (s *TaskScheduler) CancelTask(taskID string) {
    s.tw.RemoveTimer(taskID)
}
```

### ✅ 心跳检测

```go
type HeartbeatManager struct {
    tw        *collection.TimingWheel
    clients   map[string]time.Time
    timeout   time.Duration
    onTimeout func(clientID string)
}

func NewHeartbeatManager(timeout time.Duration, onTimeout func(string)) *HeartbeatManager {
    tw, _ := collection.NewTimingWheel(time.Second, int(timeout/time.Second)*2)

    hm := &HeartbeatManager{
        tw:        tw,
        clients:   make(map[string]time.Time),
        timeout:   timeout,
        onTimeout: onTimeout,
    }

    // 定期检查
    tw.SetTimer("heartbeat-check", nil, time.Second, hm.checkHeartbeats)
    return hm
}

func (hm *HeartbeatManager) UpdateHeartbeat(clientID string) {
    hm.clients[clientID] = time.Now()
}

func (hm *HeartbeatManager) checkHeartbeats(key string, value interface{}) {
    now := time.Now()
    for clientID, lastBeat := range hm.clients {
        if now.Sub(lastBeat) > hm.timeout {
            go hm.onTimeout(clientID)
            delete(hm.clients, clientID)
        }
    }

    // 继续下次检查
    hm.tw.SetTimer("heartbeat-check", nil, time.Second, hm.checkHeartbeats)
}
```

### ❌ 常见错误

```go
// 错误：槽位数太少，无法支持需要的延迟时间
tw, _ := collection.NewTimingWheel(time.Second, 10)  // 最大只支持 10 秒！

// 错误：回调函数执行耗时操作阻塞时间轮
tw.SetTimer("task", nil, delay, func(key string, value interface{}) {
    time.Sleep(10 * time.Second)  // 阻塞！
    processLargeData()            // 耗时操作！
})

// 错误：程序结束时忘记停止时间轮
// defer tw.Stop()  // 必须调用！
```

### 最佳实践

- 回调函数中执行耗时操作应使用 goroutine
- 程序结束时务必调用 `tw.Stop()`
- 根据业务需求选择合适的精度和槽位数
- 使用有意义的键名便于调试和管理

---

## SafeMap

通用的并发安全 Map 实现，适合读多写少的场景。

### ✅ 基本使用

```go
import "github.com/zeromicro/go-zero/core/collection"

m := collection.NewSafeMap()

// 设置值
m.Set("key", "value")

// 获取值
if val, ok := m.Get("key"); ok {
    fmt.Println(val.(string))
}

// 删除值
m.Del("key")

// 遍历
m.Range(func(key string, value interface{}) bool {
    fmt.Printf("%s: %v\n", key, value)
    return true  // 继续遍历
})
```

### ✅ 服务实例注册表

```go
type ServiceRegistry struct {
    services *collection.SafeMap
}

func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: collection.NewSafeMap(),
    }
}

func (r *ServiceRegistry) Register(serviceID, address string) {
    r.services.Set(serviceID, &ServiceInfo{
        Address:   address,
        LastSeen:  time.Now(),
    })
}

func (r *ServiceRegistry) Deregister(serviceID string) {
    r.services.Del(serviceID)
}

func (r *ServiceRegistry) GetService(serviceID string) (*ServiceInfo, bool) {
    if val, ok := r.services.Get(serviceID); ok {
        return val.(*ServiceInfo), true
    }
    return nil, false
}

func (r *ServiceRegistry) ListServices() []*ServiceInfo {
    var services []*ServiceInfo
    r.services.Range(func(key string, value interface{}) bool {
        services = append(services, value.(*ServiceInfo))
        return true
    })
    return services
}
```

### ✅ 共享配置存储

```go
type ConfigStore struct {
    config *collection.SafeMap
}

func NewConfigStore() *ConfigStore {
    return &ConfigStore{
        config: collection.NewSafeMap(),
    }
}

func (s *ConfigStore) Set(key string, value interface{}) {
    s.config.Set(key, value)
}

func (s *ConfigStore) Get(key string) (interface{}, bool) {
    return s.config.Get(key)
}

func (s *ConfigStore) GetInt(key string, defaultVal int) int {
    if val, ok := s.config.Get(key); ok {
        return val.(int)
    }
    return defaultVal
}

func (s *ConfigStore) GetString(key string, defaultVal string) string {
    if val, ok := s.config.Get(key); ok {
        return val.(string)
    }
    return defaultVal
}
```

### ✅ SafeMap vs sync.Map 选择

| 场景 | 推荐 | 原因 |
|------|------|------|
| 读多写少，需要遍历 | SafeMap | 简单直接，Range 方便 |
| 写一次，读多次 | sync.Map | 针对 this 场景优化 |
| 高频写入 | 都不推荐 | 考虑分片锁方案 |

### ⚠️ 内存管理注意事项

Go 的 map（包括 sync.Map 和 SafeMap）有一个重要特性：**删除元素不会自动释放内存**。

```go
// 问题场景
m := collection.NewSafeMap()

// 大量写入
for i := 0; i < 1000000; i++ {
    m.Set(fmt.Sprintf("key-%d", i), i)
}

// 大量删除
for i := 0; i < 1000000; i++ {
    m.Del(fmt.Sprintf("key-%d", i))
}

// 内存占用仍然很高！底层数组未缩容
```

### ✅ 定期重建回收内存

```go
type ServiceRegistry struct {
    services *collection.SafeMap
    mu       sync.Mutex
}

func (r *ServiceRegistry) Compact() {
    r.mu.Lock()
    defer r.mu.Unlock()

    // 创建新 map 并迁移有效数据
    newMap := collection.NewSafeMap()
    r.services.Range(func(key string, value interface{}) bool {
        newMap.Set(key, value)
        return true
    })
    r.services = newMap
}

// 定期自动压缩
func (r *ServiceRegistry) StartCompactor(interval time.Duration) {
    ticker := time.NewTicker(interval)
    go func() {
        for range ticker.C {
            r.Compact()
        }
    }()
}
```

### ❌ 常见错误

```go
// 错误：在 Range 中修改 map
m.Range(func(key string, value interface{}) bool {
    m.Del(key)  // 可能导致问题！
    return true
})

// 错误：频繁写操作导致锁竞争
for i := 0; i < 10000; i++ {
    go m.Set(key, value)  // 高频写入不适合 SafeMap
}

// 错误：大量写入后又删除导致内存浪费
for i := 0; i < 1000000; i++ {
    m.Set(key, value)
    m.Del(key)  // 内存不会释放！
}
```

### 最佳实践

- 适合读多写少、需要遍历的场景
- 在 Range 中避免修改 map
- 大小波动大的场景定期重建
- 高频写入场景考虑分片锁方案

---

## 完整使用示例

### 多级缓存架构

```go
type CacheService struct {
    localCache  *collection.Cache      // L1: 本地缓存
    redisCache  *redis.Redis           // L2: Redis 缓存
    db          sqlx.SqlConn           // L3: 数据库
    requestLog  *collection.Ring       // 请求日志
    timeoutMgr  *collection.TimingWheel // 超时管理
}

func NewCacheService(cfg config.Config) *CacheService {
    localCache, _ := collection.NewCache(1000, collection.WithEvict(func(key string, value interface{}) {
        logx.Infof("local cache evicted: %s", key)
    }))

    return &CacheService{
        localCache:  localCache,
        redisCache:  redis.MustNewRedis(cfg.Redis),
        db:          sqlx.NewMysql(cfg.DataSource),
        requestLog:  collection.NewRing(100),
        timeoutMgr:  mustNewTimingWheel(),
    }
}

func (s *CacheService) Get(ctx context.Context, key string) (interface{}, error) {
    // L1: 本地缓存
    if val, ok := s.localCache.Get(key); ok {
        s.requestLog.Add("L1 hit: " + key)
        return val, nil
    }

    // L2: Redis 缓存
    val, err := s.redisCache.Get(key)
    if err == nil {
        s.localCache.Set(key, val)
        s.requestLog.Add("L2 hit: " + key)
        return val, nil
    }

    // L3: 数据库
    val, err = s.queryDB(ctx, key)
    if err != nil {
        return nil, err
    }

    // 回填缓存
    s.localCache.Set(key, val)
    s.redisCache.Set(key, val)

    return val, nil
}
```

---

## 最佳实践总结

### ✅ DO

| 数据结构 | 最佳实践 |
|---------|---------|
| LRU Cache | 设置合理大小；淘汰回调清理资源；异步执行耗时操作 |
| Ring Buffer | 固定大小；存储轻量对象；配合定时读取 |
| TimingWheel | 回调中使用 goroutine；程序结束调用 Stop() |
| SafeMap | 读多写少场景；定期重建回收内存 |

### ❌ DON'T

| 数据结构 | 避免做法 |
|---------|---------|
| LRU Cache | 容量过大；缓存大对象；键名冲突 |
| Ring Buffer | 存储大对象；大小过大 |
| TimingWheel | 回调阻塞；槽位数太少；忘记 Stop() |
| SafeMap | Range 中修改；高频写入；忽略内存问题 |

---

## 相关资源

- [REST API Patterns](./rest-api-patterns.md) - API 层缓存策略
- [Database Patterns](./database-patterns.md) - 数据库层 Redis 缓存
- [Resilience Patterns](./resilience-patterns.md) - 弹性模式（熔断、限流）