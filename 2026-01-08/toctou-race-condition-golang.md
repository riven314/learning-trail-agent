# TOCTOU Race Condition in Go Concurrent Programming

**Context**: Encountered and fixed a TOCTOU (Time-of-Check-Time-of-Use) race condition while implementing spread-based cooldown mechanism in standx-maker-bot trading system. Two goroutines (order placement loop and WebSocket order fill handler) were competing to update shared cooldown state.

**Session**: `/Users/alexlwh/Desktop/crypto/dex-farming/standx-maker-bot`

## The TOCTOU Pattern

TOCTOU occurs when you separate the "check" and "use" operations with different locks, creating a vulnerable gap where state can change between the two operations.

### Broken Pattern
```go
// Step 1: Check with read lock
b.mu.RLock()
alreadyInCooldown := b.inCooldown
b.mu.RUnlock()

// ⚠️ VULNERABLE GAP - state can change here!

if alreadyInCooldown {
    return true
}

// Step 2: Use with write lock
b.mu.Lock()
b.inCooldown = true
b.cooldownUntil = time.Now().Add(30 * time.Second)
b.mu.Unlock()
```

### What Goes Wrong
1. Goroutine A: reads `inCooldown = false`, releases read lock
2. Goroutine B: acquires write lock, sets `cooldownUntil = now + 600s`
3. Goroutine A: acquires write lock, overwrites with `cooldownUntil = now + 30s`

Result: 600-second cooldown silently shortened to 30 seconds.

## The Fix: Atomic Check-and-Set

```go
// Single write lock for both check AND set
b.mu.Lock()
if b.inCooldown {
    b.mu.Unlock()
    return true
}
b.inCooldown = true
b.cooldownUntil = time.Now().Add(30 * time.Second)
b.mu.Unlock()
```

## Key Insights

**Critical Rule**: When you need to check state and then modify it based on that check, both operations must happen under the same lock.

**Common Mistakes**:
- Using read lock for check, then upgrading to write lock (lock upgrade is not atomic)
- Releasing lock between check and set operations
- Assuming read-only checks don't need synchronization with subsequent writes

**Related Patterns**:
- Double-checked locking: similar check-then-act pattern
- Compare-and-swap (CAS): atomic alternative for simple cases
- Critical sections: the entire check-modify sequence must be atomic

**Trade-offs**:
- Using write lock for check reduces concurrency slightly
- But prevents silent data corruption from race conditions
- In high-contention scenarios, consider sync/atomic package for CAS operations

**Real-world Impact**: In this trading bot case, the bug could cause risk limits to be violated by allowing more aggressive trading than intended when cooldown periods are unexpectedly shortened.

#GOLANG #ALGO-TRADING
