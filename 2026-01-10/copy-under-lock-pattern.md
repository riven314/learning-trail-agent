# Copy-Under-Lock Pattern

## Context
Building a market maker trading bot for StandX (Go-based). This pattern emerged from handling order fill callbacks while avoiding lock contention and deadlocks. Session: `/Users/alexlwh/Desktop/crypto/dex-farming/standx-maker-bot`

## Core Pattern

When holding a mutex and needing to call external callbacks:
1. Decide what action to take while holding lock
2. Copy necessary data while holding lock
3. Release lock
4. Call callback with copied data (outside lock)

## Why This Matters

**Lock scope minimization**: Lock held only for ~100ns memory operations, not ~100ms I/O operations

**Deadlock prevention**: Callback can safely call back into the locked object without risk of recursive lock acquisition

**Snapshot consistency**: Callback receives point-in-time snapshot, immune to concurrent mutations

## Inverse of TOCTOU

This is the correct inverse of the TOCTOU anti-pattern:
- TOCTOU (bad): separate check and use across lock boundaries
- Copy-under-lock (good): combine check AND capture under one lock, then safely use captured data outside

## Critical Nuance: Shallow vs Deep Copy

### Shallow Copy Behavior

```go
trackedCopy := &TrackedOrder{
    Order:   tracked.Order,    // pointer copied - points to same object
    ClOrdId: tracked.ClOrdId,  // value copied - independent copy
}
```

**Pointer reassignment is safe**:
```go
tracked.Order = &standx.Order{Status: "new"}  // new pointer
// trackedCopy.Order still points to old object - unaffected
```

**Object mutation is NOT safe**:
```go
tracked.Order.Status = "cancelled"  // mutating shared object
// trackedCopy.Order.Status ALSO shows "cancelled" - same memory location
```

### When Deep Copy is Required

- **Value fields** (string, int, float64): automatically safe - Go copies by value
- **Pointer fields**: shallow copy shares underlying object
- **Need deep copy if**: other goroutines could mutate the pointed-to object after lock release

## Implementation Example

```go
func (m *OrderManager) HandleOrderUpdate(update *standx.OrderUpdate) {
    m.mu.Lock()

    var shouldCallFill bool
    var trackedCopy *TrackedOrder

    if update.Status == "filled" {
        shouldCallFill = true
        trackedCopy = &TrackedOrder{
            Order:   tracked.Order,    // shallow copy
            ClOrdId: tracked.ClOrdId,
        }
    }

    m.mu.Unlock()  // release BEFORE callback

    if shouldCallFill {
        positions, _ := m.restClient.GetPositions(...)  // slow I/O outside lock
        m.onFill(trackedCopy, position)
    }
}
```

## Key Insight

The pattern prioritizes **latency** (minimize lock duration) and **safety** (prevent deadlocks) over **memory efficiency** (temporary copies). Critical for high-frequency trading systems where lock contention directly impacts performance.

#GOLANG #CONCURRENCY #MUTEX #DESIGN-PATTERN
