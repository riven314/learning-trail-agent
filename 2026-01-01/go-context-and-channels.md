# Go Context and Channels Deep Dive

**Session**: /Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker
**Context**: Working on OpenSQT Market Maker codebase, a WebSocket-driven high-frequency trading system. Explored Go's context package and channel behaviors for graceful shutdown and cancellation patterns.

## Context Package Fundamentals

### What is Context?

`context.Context` is Go's standard mechanism for cancellation signals, deadlines, and request-scoped values across API boundaries and goroutines.

**Key insight**: Context is NOT a channel itself, but CONTAINS a channel internally. The `ctx.Done()` method returns `<-chan struct{}` that gets closed when cancellation occurs.

### Root Context Creation

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

- `context.Background()` creates the root context
- `context.WithCancel()` wraps it with cancellation capability
- Returns both context and cancel function
- Always defer cancel() to prevent context leaks

### Context Hierarchy

Contexts form parent-child relationships. Cancelling parent propagates to all children:

```go
parentCtx, parentCancel := context.WithCancel(context.Background())
childCtx, childCancel := context.WithCancel(parentCtx)

parentCancel() // this cancels BOTH parent and child
```

### Component-Level Context Pattern

Store context in struct for component lifecycle management:

```go
type PriceMonitor struct {
    ctx    context.Context
    cancel context.CancelFunc
    // other fields
}

func NewPriceMonitor(parentCtx context.Context) *PriceMonitor {
    ctx, cancel := context.WithCancel(parentCtx)
    return &PriceMonitor{
        ctx:    ctx,
        cancel: cancel,
    }
}

func (pm *PriceMonitor) Start() {
    go func() {
        select {
        case <-pm.ctx.Done():
            return
        // other cases
        }
    }()
}

func (pm *PriceMonitor) Stop() {
    pm.cancel()
}
```

### Context with Timeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

// operation with deadline
select {
case <-operationDone:
    // success
case <-ctx.Done():
    // timeout or cancellation
}
```

## Channel Behaviors - Critical Insights

### Closed Channel Behavior (Most Important)

**Receiving from a closed channel returns the zero value immediately and does NOT block or error**:

```go
ch := make(chan int)
close(ch)

v := <-ch  // v = 0 (int's zero value), returns immediately
```

This is why graceful shutdown works:

```go
select {
case <-ctx.Done():  // fires immediately when cancel() closes internal channel
    return
}
```

### Two-Value Receive Form

Detect channel closure explicitly:

```go
v, ok := <-ch

if !ok {
    // channel is closed
}
```

Single-value form `v := <-ch` cannot detect closure - you get zero value whether channel is closed or sender sent zero value.

### Range Over Channel

`for v := range ch` uses two-value form internally and stops when channel closes:

```go
for msg := range ch {
    // loop exits automatically when ch is closed
}
```

### Select with Default (Non-Blocking)

```go
select {
case msg := <-ch:
    // received message
default:
    // executes immediately if nothing available
}
```

Without `default`, `select` blocks until at least one case can proceed.

### Channel Closing Semantics

**Closing broadcasts to ALL receivers** (vs sending which reaches only one):

```go
ch := make(chan struct{})
// 100 goroutines all waiting on <-ch

close(ch)  // ALL 100 goroutines wake up immediately
```

This is why `cancel()` can stop all goroutines - it closes the internal context channel.

### Why chan struct{} for Done Channel

```go
<-chan struct{}  // receive-only channel of empty structs
```

- `struct{}` has zero size in memory
- Never written to, only closed
- Optimized for signaling without data transfer

## Two-Level Cancellation Pattern

Real-world example from WebSocket handler:

```go
func (ws *WebSocket) startOrderStream(ctx context.Context) {
    innerCtx, innerCancel := context.WithCancel(ctx)
    defer innerCancel()

    go func() {
        for {
            select {
            case <-innerCtx.Done():
                return
            case msg := <-msgChan:
                // process
            }
        }
    }()

    // blocking websocket read loop
    for {
        select {
        case <-ctx.Done():
            innerCancel()  // stop message processor first
            return
        default:
            // non-blocking check, then do work
            err := conn.ReadJSON(&msg)
            msgChan <- msg
        }
    }
}
```

**Why two contexts?**
- Outer `ctx` controls the WebSocket reader
- Inner `innerCtx` controls the message processor goroutine
- Allows graceful shutdown: stop processor first, then reader

## Python Equivalents

### Context Cancellation
**Go**:
```go
ctx, cancel := context.WithCancel(context.Background())
cancel()  // signal cancellation
```

**Python** (manual):
```python
import asyncio

shutdown_event = asyncio.Event()

async def worker():
    while not shutdown_event.is_set():
        await asyncio.sleep(0.1)

shutdown_event.set()  # signal cancellation
```

### Context Timeout
**Go**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
```

**Python**:
```python
await asyncio.wait_for(some_coroutine(), timeout=5.0)
```

## Practical Patterns from OpenSQT Codebase

### Graceful Shutdown in main.go

```go
ctx, cancel := context.WithCancel(context.Background())

// start components with context
go priceMonitor.Start(ctx)

// wait for interrupt signal
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)
<-sigChan

// graceful shutdown
cancel()  // stops all components

// wait for cleanup with timeout
cleanupCtx, cleanupCancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cleanupCancel()

exchange.CancelAllOrders(cleanupCtx)
```

### Select Pattern for Responsive Shutdown

```go
for {
    select {
    case <-ctx.Done():
        return  // immediate exit on cancellation
    case event := <-eventChan:
        handleEvent(event)
    case <-time.After(1 * time.Second):
        doPeriodicWork()
    }
}
```

**Without select**: goroutine would block on channel receive and miss cancellation signal.

## Key Takeaways

1. **Context is not a channel** - it wraps a channel accessed via `Done()`
2. **Closed channels don't block** - receiving from closed channel returns zero value immediately
3. **Use two-value receive to detect closure** - `v, ok := <-ch`
4. **Closing broadcasts to all receivers** - perfect for shutdown signals
5. **Select without default blocks** - add default for non-blocking checks
6. **Context hierarchy enables cascading cancellation** - cancel parent stops all children
7. **Store context in component structs** - enables clean lifecycle management
8. **Two-level contexts for complex shutdown** - inner context for nested goroutines

#GOLANG

