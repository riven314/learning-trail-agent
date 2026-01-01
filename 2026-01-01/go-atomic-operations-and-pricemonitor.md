# Go Atomic Operations and PriceMonitor Architecture

**Context**: Deep dive into the PriceMonitor component from the OpenSQT Market Maker codebase, exploring Go's concurrency primitives and atomic operations. This session focused on understanding why certain patterns exist and their hardware-level implications.

**Session Directory**: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

---

## PriceMonitor: Dual Data Propagation Pattern

PriceMonitor acts as the single source of truth for market prices, implementing two complementary data access patterns:

### Channel Subscription (Push Model)
- Components subscribe via channels to receive price updates
- Enables reactive event-driven architecture
- Backpressure handling with "latest wins" pattern - drops stale updates when subscriber is slow

### Atomic Read (Pull Model)
- Direct atomic reads via `atomic.Pointer[T]` for immediate access
- Zero-latency access to current price without waiting for channel
- No blocking, always returns latest value

### Why Both Patterns?
The hybrid approach provides:
- **Channels**: Wake-up signals for event-driven components that need to react to changes
- **Atomic reads**: Instant access to latest state for components that need current value on-demand
- Best of both worlds: reactive updates + low-latency reads

---

## Atomic Operations: Preventing Torn Reads

### The Problem: Torn Reads
When reading multi-byte values without atomics, CPU might read partially written data:

```
Thread A writes: 0x1234567812345678
Thread B reads:  0x1234567800000000  ← torn read (old + new bytes mixed)
```

This happens at hardware level on multi-core systems when reads/writes cross cache line boundaries.

### atomic.Value vs atomic.Pointer[T]

**atomic.Value** (older, Go 1.4+):
```go
var price atomic.Value
price.Store(&PriceChange{...})

val := price.Load()           // returns interface{}
pc := val.(*PriceChange)      // requires type assertion
```

- Generic but requires runtime type assertions
- Type consistency enforced at runtime (panics if types mismatch)

**atomic.Pointer[T]** (newer, Go 1.19+):
```go
var price atomic.Pointer[PriceChange]
price.Store(&PriceChange{...})

pc := price.Load()  // returns *PriceChange directly
```

- Compile-time type safety
- No type assertions needed
- Cleaner, more ergonomic API

### Why Pointers, Not Values?

`atomic.Pointer[T]` stores `*T` (pointer), not `T` directly because:
- CPU atomic operations work on machine word size (64 bits on 64-bit systems)
- A pointer is 64 bits, guaranteeing atomic read/write in single instruction
- Arbitrary structs can be larger than 64 bits, cannot be atomically swapped

### Specialized Atomics for Primitives

For basic types, use specialized atomics (no pointers needed):
```go
atomic.Int64    // for int64
atomic.Float64  // for float64
atomic.Bool     // for bool
```

These store values directly, not pointers, because primitives fit in machine word.

---

## Channels and Select: Non-Blocking Patterns

### Buffered Channels as Bounded Pipes
```go
priceChan := make(chan *PriceChange, 100)  // buffer size 100
```

- Buffer acts as queue with max capacity
- Send blocks when full, receive blocks when empty
- Size determines how much "slack" subscribers have

### Select with Default: Non-Blocking Send

```go
select {
case subscriber <- price:
    // successfully sent
default:
    // channel full, drop update (latest wins pattern)
}
```

Key insight: **select both checks AND performs operation atomically**. The case doesn't just check if channel is ready - it performs the send/receive as part of the selection.

This enables "latest wins" backpressure handling - slow consumers get latest data rather than stale queued updates.

---

## Ticker: Rate Limiting Pattern

```go
ticker := time.NewTicker(500 * time.Millisecond)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        // execute every 500ms
    }
}
```

- `ticker.C` is a channel receiving time.Time on each interval
- Used to throttle updates, preventing excessive operations
- Always `defer ticker.Stop()` to prevent goroutine leaks

---

## Go Syntax Deep Dive

### Type Assertion Syntax
```go
val := price.Load()           // interface{}
pc := val.(*PriceChange)      // assert to *PriceChange
```

Pattern: `value.(ConcreteType)` - asserts interface to concrete type.

### Typed Nil
```go
price.Store((*PriceChange)(nil))  // typed nil pointer
```

Why needed: `atomic.Value` enforces type consistency. Cannot store untyped `nil` then later store `*PriceChange`. Must establish type with typed nil first.

### *T vs &T
- `*PriceChange` in type assertion = pointer type (type name)
- `&priceChange` in code = address-of operator (taking address of variable)
- Different contexts, different meanings

---

## Design Patterns Summary

### Hybrid Push-Pull Pattern
- Store state in atomic variable (pull)
- Notify via channels (push)
- Combines reactive updates with instant reads

### Latest Wins Backpressure
```go
select {
case ch <- data:
    // sent
default:
    // dropped, consumer too slow
}
```

Prevents slow consumers from blocking system - they get latest data, not stale queue.

### Single Source of Truth
PriceMonitor demonstrates architectural principle:
- One component owns the state
- Multiple consumers access via well-defined interfaces
- No state duplication or synchronization issues

---

#GOLANG #ALGO-TRADING #QUANT
