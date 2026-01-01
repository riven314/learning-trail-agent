# Go Concurrency Fundamentals

**Context**: Deep dive into Go's concurrency model while working on the OpenSQT Market Maker system. This session explored fundamental differences between goroutines and traditional threading models, lifecycle management, and comparisons with Python's concurrency approaches.

**Session directory**: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

## Goroutines vs Traditional Threading

### M:N Threading Model
- Goroutines use **M:N threading**: M goroutines multiplexed onto N OS threads
- Closer to multithreading (shared memory, single process) than multiprocessing
- Scheduled by Go runtime, not OS kernel
- **Extremely lightweight**: ~2KB initial stack vs ~1-8MB for OS threads
- Can spawn millions of goroutines without overwhelming the system

### Key Characteristics
```go
// goroutines share memory space within same process
go func() {
    counter++ // can access and modify shared variables
}()
```

- Shared memory: All goroutines access same address space
- Preemptive scheduling: Go runtime decides when to switch
- Work stealing: Idle threads steal work from busy ones
- Fast context switching: ~200ns vs ~1-10μs for kernel threads

## Goroutine Lifecycle Management

### Critical Insight: Goroutines Don't Auto-Terminate
```go
func caller() {
    go worker() // spawns goroutine
    return      // caller exits, but goroutine keeps running!
}
```

- Goroutines live until:
  1. They complete execution naturally, OR
  2. The main goroutine exits (kills all others)
- Fork-join pattern is **opt-in**, not automatic

### Coordination Patterns

**Option 1: WaitGroup (fork-join)**
```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // work here
}()
wg.Wait() // caller waits for completion
```

**Option 2: Context (cancellation)**
```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go func() {
    for {
        select {
        case <-ctx.Done():
            return // coordinated shutdown
        default:
            // work here
        }
    }
}()
```

### Main Goroutine is Special
```go
func main() { // this IS a goroutine (the "main goroutine")
    go worker() // spawns additional goroutine
    // if main returns here, worker dies immediately
}
```

- Main goroutine created implicitly by Go runtime
- When main exits, entire program terminates (all goroutines killed)
- `go` keyword spawns additional goroutines, but main already IS one

## Go vs Python Concurrency Models

### Python Threading (1:1 OS Threads)
- One Python thread = One OS thread
- **GIL limitation**: Only one thread executes Python bytecode at a time
- Good for I/O-bound work, poor for CPU-bound work
- No true parallelism for CPU-intensive tasks

### Python Multiprocessing (Isolated Processes)
- True parallel execution (each process has own GIL)
- **Isolated memory**: Must use IPC (pipes, queues) for communication
- Heavy overhead: Each process ~20-50MB minimum
- Good for CPU-bound work requiring true parallelism

### Python Asyncio (Single-threaded Concurrency)
- Cooperative multitasking on single thread
- Requires async/await syntax and compatible libraries
- No parallelism whatsoever (single thread)
- Excellent for I/O-bound with high concurrency needs

### Go Goroutines (M:N Hybrid)
```
Python Threading:     1:1 (+ GIL bottleneck)
Python Multiprocessing: N:N (isolated memory)
Python Asyncio:       1:0 (no parallelism)
Go Goroutines:        M:N (best of both worlds)
```

## Advantages of Go's Design

1. **No GIL**: True parallel execution for CPU-bound work
2. **Massive concurrency**: Millions of goroutines possible
3. **Unified model**: Same approach for I/O-bound and CPU-bound
4. **Shared memory**: Easy communication between goroutines
5. **Cheap context switching**: Runtime-level scheduling overhead minimal
6. **Channels**: Built-in safe communication primitives
7. **Simple syntax**: Just `go func()` - no complex APIs

## Disadvantages of Go's Design

1. **Less isolation**: Shared memory requires careful synchronization
   - Data races possible without proper sync primitives
   - One goroutine's corruption affects all others

2. **Less control**: No direct thread/core affinity management
   - Runtime decides scheduling, not programmer
   - Harder to optimize for NUMA architectures

3. **GC pauses**: Affect all goroutines simultaneously
   - Stop-the-world phases impact latency-sensitive code
   - Can't isolate critical paths from GC

4. **Hidden complexity**: Scheduler behavior not always obvious
   - Debugging race conditions can be challenging
   - GOMAXPROCS tuning sometimes needed

## Practical Context Switch Comparison

```
Context Switch Times:
- Go goroutine:     ~200 nanoseconds
- OS thread:        ~1-10 microseconds (5-50x slower)
- Process:          ~10-100 microseconds (50-500x slower)
```

This overhead difference is why Go can handle millions of concurrent operations while traditional threading models struggle beyond thousands.

## Real-World Application (OpenSQT Market Maker)

The market maker system leverages goroutines for:
- WebSocket price stream processing (continuous)
- WebSocket order update handling (event-driven)
- Periodic reconciliation tasks
- Risk monitoring on K-line data
- Concurrent order placement across price levels

All coordinated via channels and context cancellation for clean shutdown.

---

#GOLANG #GO-CONCURRENCY

