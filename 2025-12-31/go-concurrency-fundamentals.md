# Go Concurrency Fundamentals

**Context**: Learning from OpenSQT Market Maker codebase - a high-frequency trading system using goroutines, sync.Map, and per-slot RWMutex for thread-safe order management. Session directory: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

## Goroutines

- Lightweight threads spawned with `go functionA()`
- Multiple goroutines run concurrently on available CPU cores
- Cheap to create - can spawn thousands without significant overhead

## Race Conditions

**Problem**: Two goroutines accessing shared variable simultaneously causes lost updates

```go
// BAD: race condition
counter = counter + 1  // from goroutine A
counter = counter + 1  // from goroutine B
// Result: may only increment once instead of twice
```

**Real impact**: In trading systems, this corrupts position state and causes order duplication

## Mutex (sync.Mutex)

**Concept**: Mutual exclusion lock - only one goroutine holds it at a time

```go
var mu sync.Mutex

mu.Lock()
counter = counter + 1  // protected section
mu.Unlock()
```

**Mental model**: Like a bathroom lock - one person at a time, others wait

## RWMutex (sync.RWMutex)

**Optimization**: Separate read and write locks for read-heavy workloads

```go
var mu sync.RWMutex

// Multiple readers allowed simultaneously
mu.RLock()
value := data.field
mu.RUnlock()

// Writer gets exclusive access
mu.Lock()
data.field = newValue
mu.Unlock()
```

**Key behavior**:
- Multiple `RLock()` holders can coexist
- `Lock()` blocks until all readers and writers release
- Use when reads vastly outnumber writes

**Critical insight**: Blocking is based on **mutex instance state**, not data access patterns. If 10 goroutines hold `RLock()` on the same `mu`, calling `mu.Lock()` blocks regardless of what code runs inside the critical section (even just `time.Sleep`). The mutex tracks lock acquisitions/releases - it has no awareness of what data is being protected.

## sync.Map

**Problem**: Regular Go maps crash with concurrent access (`fatal error: concurrent map writes`)

**Solution**: Built-in thread-safe map with these methods:

```go
var slots sync.Map

slots.Store(key, value)      // instead of slots[key] = value
val, ok := slots.Load(key)   // instead of val := slots[key]
slots.Delete(key)            // instead of delete(slots, key)
slots.Range(func(k, v interface{}) bool { ... })  // iteration
```

**No external lock needed** - sync.Map handles concurrency internally

## Per-Instance Lock Pattern

**Pattern**: Embed mutex inside struct for fine-grained locking

```go
type InventorySlot struct {
    mu sync.RWMutex  // each slot has its own lock
    Price float64
    Status string
}

func (s *InventorySlot) UpdateStatus(status string) {
    s.mu.Lock()         // lock this specific slot
    s.Status = status
    s.mu.Unlock()
}
```

**Important**: Lock is convention-based - programmer must manually lock/unlock around field access. The lock doesn't "know" what it protects.

## Global Lock vs Per-Instance Lock Trade-off

**Global lock**:
- Simple - one lock protects entire collection
- Bottleneck - only one goroutine operates at a time
- Example: single mutex protecting map of 1000 price slots

**Per-instance lock**:
- Complex - each resource has its own lock
- Parallel access - goroutines work on independent resources simultaneously
- Example: 1000 price slots, each with embedded mutex, allowing concurrent updates to different slots

**When to use per-instance locks**: High-throughput systems with many independent resources (like market maker managing hundreds of price levels)

## Real-World Application Pattern

OpenSQT Market Maker combines these concepts:

```go
// Thread-safe slot storage
var inventory sync.Map

type InventorySlot struct {
    mu sync.RWMutex  // per-slot lock
    Status string    // FREE/PENDING/LOCKED state machine
    // ... other fields
}

// Concurrent order updates from WebSocket
func OnOrderUpdate(price float64) {
    slotInterface, _ := inventory.Load(price)
    slot := slotInterface.(*InventorySlot)

    slot.mu.Lock()  // lock only this price level
    slot.Status = "LOCKED"
    slot.mu.Unlock()
}
```

This design allows:
- Thousands of goroutines processing different price levels in parallel
- No global bottleneck
- Safe concurrent state transitions

#GOLANG #ALGO-TRADING
