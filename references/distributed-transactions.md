# Distributed Transaction Patterns

## Overview

In microservices architecture, maintaining data consistency across services requires distributed transactions. go-zero integrates with [DTM](https://github.com/dtm-labs/dtm) to provide seamless distributed transaction support.

### Common Scenarios

| Scenario | Description |
|----------|-------------|
| Order system | Create order + deduct inventory (both succeed or both rollback) |
| Cross-bank transfer | Deduct balance + add balance (atomic across databases) |
| Points redemption | Deduct points + add benefits (consistency required) |
| Travel booking | Book multiple tickets (all succeed or all cancel) |

## DTM Integration

### ✅ Correct Pattern: SAGA

```go
// internal/logic/createorderlogic.go
func (l *CreateOrderLogic) CreateOrder(req *types.CreateOrderReq) (*types.CreateOrderResp, error) {
    orderID := snowflake.New().String()
    
    // Initialize DTM client
    dtmServer := l.svcCtx.Config.DtmServer
    gid := dtm.MustGenGid(dtmServer)
    
    // SAGA transaction
    saga := dtmcli.NewSaga(dtmServer, gid).
        Add(
            // Step 1: Create order
            fmt.Sprintf("%s/order/create", l.svcCtx.Config.OrderRpc),
            fmt.Sprintf("%s/order/create/compensate", l.svcCtx.Config.OrderRpc),
            &OrderReq{OrderID: orderID, UserID: req.UserID, Amount: req.Amount},
        ).
        Add(
            // Step 2: Deduct inventory
            fmt.Sprintf("%s/inventory/deduct", l.svcCtx.Config.InventoryRpc),
            fmt.Sprintf("%s/inventory/deduct/compensate", l.svcCtx.Config.InventoryRpc),
            &InventoryReq{ProductID: req.ProductID, Quantity: req.Quantity},
        )
    
    err := saga.Submit()
    if err != nil {
        logx.Errorf("saga submit failed: %v", err)
        return nil, errors.New("order creation failed")
    }
    
    return &types.CreateOrderResp{OrderID: orderID}, nil
}
```

### ✅ Correct Pattern: TCC (Try-Confirm-Cancel)

```go
// internal/logic/transferlogic.go
func (l *TransferLogic) Transfer(req *types.TransferReq) error {
    gid := dtm.MustGenGid(l.svcCtx.Config.DtmServer)
    
    tcc := dtmcli.NewTcc(l.svcCtx.Config.DtmServer, gid)
    
    err := tcc.TccGlobalTransaction(func(tcc *dtmcli.Tcc) (*resty.Response, error) {
        // Try: Reserve amount in source account
        resp, err := tcc.CallBranch(
            &TransferReq{
                FromAccount: req.FromAccount,
                ToAccount:   req.ToAccount,
                Amount:      req.Amount,
            },
            fmt.Sprintf("%s/account/reserve", l.svcCtx.Config.AccountRpc),
            fmt.Sprintf("%s/account/confirm", l.svcCtx.Config.AccountRpc),
            fmt.Sprintf("%s/account/cancel", l.svcCtx.Config.AccountRpc),
        )
        if err != nil {
            return nil, err
        }
        
        // Try: Reserve in destination account
        resp, err = tcc.CallBranch(
            &TransferReq{
                ToAccount: req.ToAccount,
                Amount:    req.Amount,
            },
            fmt.Sprintf("%s/account/reserveDest", l.svcCtx.Config.AccountRpc),
            fmt.Sprintf("%s/account/confirmDest", l.svcCtx.Config.AccountRpc),
            fmt.Sprintf("%s/account/cancelDest", l.svcCtx.Config.AccountRpc),
        )
        return resp, err
    })
    
    return err
}
```

### ✅ Correct Pattern: 2-Phase Message (Database + Cache Consistency)

```go
// internal/logic/updateuserlogic.go
func (l *UpdateUserLogic) UpdateUser(req *types.UpdateUserReq) error {
    gid := dtm.MustGenGid(l.svcCtx.Config.DtmServer)
    
    msg := dtmcli.NewMsg(l.svcCtx.Config.DtmServer, gid).
        Add(
            fmt.Sprintf("%s/user/update", l.svcCtx.Config.UserRpc),
            &UpdateUserReq{UserID: req.UserID, Name: req.Name},
        ).
        Add(
            fmt.Sprintf("%s/cache/invalidate", l.svcCtx.Config.CacheRpc),
            &CacheInvalidateReq{Key: fmt.Sprintf("user:%d", req.UserID)},
        )
    
    // Query barrier ensures exactly-once execution
    err := msg.DoAndSubmit(
        fmt.Sprintf("%s/user/queryBarrier", l.svcCtx.Config.UserRpc),
        func(bb *dtmcli.BranchBarrier) error {
            // This executes before the message is sent
            // If this succeeds but msg fails, barrier prevents duplicate execution
            return nil
        },
    )
    
    return err
}
```

### ❌ Common Mistakes

```go
// DON'T: Manual distributed transaction without proper compensation
func (l *CreateOrderLogic) CreateOrderBad(req *types.CreateOrderReq) error {
    // ❌ No atomicity guarantee between these operations
    _, err := l.svcCtx.OrderRpc.CreateOrder(ctx, &orderReq)
    if err != nil {
        return err
    }
    
    // ❌ If this fails, order is already created (inconsistent state)
    _, err = l.svcCtx.InventoryRpc.Deduct(ctx, &inventoryReq)
    if err != nil {
        // ❌ Manual rollback is error-prone
        l.svcCtx.OrderRpc.CancelOrder(ctx, &cancelReq)
        return err
    }
    
    return nil
}

// DON'T: Missing compensation endpoint
saga := dtmcli.NewSaga(dtmServer, gid).
    Add(
        orderUrl,
        "",  // ❌ Empty compensate URL - no rollback possible
        &OrderReq{},
    )

// DON'T: Not using barrier for idempotency
func (l *DeductLogic) Deduct(req *DeductReq) error {
    // ❌ No barrier - may execute multiple times on retry
    result, err := l.svcCtx.DB.Exec(
        "UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?",
        req.Quantity, req.ProductID,
    )
    return err
}
```

## Barrier Pattern for Idempotency

### ✅ Correct Pattern

```go
// internal/logic/deductlogic.go
func (l *DeductLogic) Deduct(barrierReq *types.BarrierReq) error {
    // Parse barrier from request
    barrier, err := dtmcli.BarrierFromQuery(barrierReq.TransType, barrierReq.Gid, 
        barrierReq.BranchID, barrierReq.Op)
    if err != nil {
        return err
    }
    
    // Execute with barrier - ensures exactly-once
    return barrier.Call(l.svcCtx.DB, func(tx *sql.Tx) error {
        // This code runs in a transaction with barrier protection
        _, err := tx.Exec(
            "UPDATE inventory SET quantity = quantity - ? WHERE product_id = ? AND quantity >= ?",
            barrierReq.Quantity, barrierReq.ProductID, barrierReq.Quantity,
        )
        return err
    })
}

// HTTP handler for barrier endpoint
func (h *DeductHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    var req types.BarrierReq
    if err := httpx.Parse(r, &req); err != nil {
        httpx.ErrorCtx(r.Context(), w, err)
        return
    }
    
    l := logic.NewDeductLogic(r.Context(), h.svcCtx)
    err := l.Deduct(&req)
    
    // DTM expects specific response format
    if err != nil {
        httpx.ErrorCtx(r.Context(), w, err)
        return
    }
    
    // Success response for DTM
    w.WriteHeader(http.StatusNoContent)
}
```

## Configuration

### ✅ Configuration Pattern

```go
// internal/config/config.go
type Config struct {
    rest.RestConf
    
    DtmServer string `json:",default=http://localhost:36789/dtm"`
    
    OrderRpc     zrpc.RpcClientConf
    InventoryRpc zrpc.RpcClientConf
    AccountRpc   zrpc.RpcClientConf
}
```

```yaml
# etc/service.yaml
Name: order-api
Host: 0.0.0.0
Port: 8888

DtmServer: http://dtm:36789/dtm

OrderRpc:
  Etcd:
    Hosts:
      - etcd:2379
    Key: order.rpc

InventoryRpc:
  Etcd:
    Hosts:
      - etcd:2379
    Key: inventory.rpc
```

## Transaction Patterns Comparison

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| **SAGA** | Long-running transactions | Simple, supports compensation | No isolation during execution |
| **TCC** | Financial operations | Strong isolation | Complex implementation, 3 endpoints per operation |
| **2PC Message** | DB + Cache consistency | Simple, reliable | Requires query barrier |
| **XA** | Database-only transactions | ACID compliant | Poor performance, not recommended for microservices |

## Best Practices

### ✅ Always Follow

- Use barrier for idempotency in all transaction participants
- Design compensation logic carefully - it must be reversible
- Set appropriate timeout values for long-running transactions
- Log transaction IDs for debugging
- Handle network failures with retries

### ❌ Never Do

- Skip compensation endpoints (will cause inconsistent state)
- Use distributed transactions when local transaction suffices
- Ignore barrier errors (may cause duplicate execution)
- Put business logic outside barrier.Call()
- Use XA transactions for high-throughput scenarios

## Error Handling

```go
// Check specific DTM errors
import "github.com/dtm-labs/client/dtmcli/dtmimp"

err := saga.Submit()
if errors.Is(err, dtmimp.ErrFailure) {
    // Transaction failed, compensation should have been triggered
    logx.Error("transaction failed, compensated")
} else if errors.Is(err, dtmimp.ErrOngoing) {
    // Transaction still in progress
    logx.Info("transaction ongoing, check status later")
} else if err != nil {
    // Other errors
    logx.Errorf("transaction error: %v", err)
}
```

## Monitoring

```go
// Query transaction status
status, err := dtmcli.Query(dtmServer, gid)
if err != nil {
    logx.Errorf("query transaction failed: %v", err)
}

// status.Status values:
// - "prepared": Transaction created, not yet submitted
// - "submitted": Transaction submitted, waiting for completion
// - "succeed": All branches succeeded
// - "failed": Transaction failed, compensation triggered
// - "abort": Transaction aborted
```

## References

- [DTM Documentation](https://dtm.pub/)
- [DTM go-zero Integration](https://dtm.pub/ref/gozero.html)
- [go-zero Distributed Transaction Guide](https://go-zero.dev/docs/distributed-transaction)