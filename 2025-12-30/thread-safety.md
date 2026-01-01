# Thread Safety

**Context**: Captured fundamental understanding of thread safety concepts, with specific focus on how Python's GIL provides practical safety for simple operations, and when explicit locking is actually needed. This clarifies common misconceptions about race conditions in Python vs Go.

**Session**: `/Users/alexlwh/Desktop/crypto/archives/opensqt_market_maker`

---

## Core Concept

**Thread safety** = code works correctly when multiple threads access it simultaneously, without data corruption or race conditions.

## The Classic Problem: Race Condition

```
Thread 1: Read(0) → Add(1) → Write(1)
Thread 2: Read(0) → Add(1) → Write(1)
Result: 1 (WRONG! Should be 2)
```

Both threads read the same value before either writes back = **lost update**.

---

## Python's GIL (Global Interpreter Lock)

**Key Facts:**
- Only **one thread** executes Python bytecode at a time
- GIL releases during: `time.sleep()`, I/O operations, C extensions
- Python threading = **concurrent** (interleaved) NOT **parallel** (simultaneous)

### Why Simple Operations Are Practically Safe

```python
counter += 1  # Executes so fast, GIL rarely switches mid-operation
              # Practically thread-safe (no lock needed)
```

### When Race Conditions Actually Appear

Race conditions occur when there's a **window for thread switching** between read and write:

```python
# ❌ RACE CONDITION - sleep creates window for thread switch
temp = counter      # Read
time.sleep(0.001)   # GIL releases! Other threads run!
counter = temp + 1  # Write with stale value

# ✅ SAFE - no window between read and write
counter += 1        # Fast atomic-like operation
```

### When Locks Are Needed

Locks protect code when operations span multiple steps with potential thread switching:

```python
import threading
import time

counter = 0
lock = threading.Lock()

def increment_with_delay_safe():
    global counter
    for _ in range(10000):
        with lock:  # Lock BEFORE reading
            temp = counter
            time.sleep(0.0000001)  # Even with sleep, safe now!
            counter = temp + 1
        # Lock automatically released here

# Thread-safe! Always produces 100000
threads = [threading.Thread(target=increment_with_delay_safe) for _ in range(10)]
for t in threads:
    t.start()
for t in threads:
    t.join()
print(f"Safe counter: {counter}")  # Always 100000 ✓
```

The lock is meaningful here because thread switching can occur between read and update operations.

---

## Thread Safety Mechanisms

### Python

```python
import threading

# 1. Lock (Mutex) - for multi-step operations
lock = threading.Lock()
with lock:
    temp = shared_data
    # ... operations that might yield GIL ...
    shared_data = temp + 1

# 2. Thread-safe data structures
from queue import Queue
q = Queue()  # No manual locking needed
q.put(item)
q.get()

# 3. For CPU-bound parallelism: multiprocessing (bypasses GIL)
from multiprocessing import Process, Value, Lock
```

### Asyncio (Single-threaded Concurrency)

Race conditions in asyncio occur at **`await` points** where the event loop can switch tasks:

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def increment_with_lock():
    global counter
    for _ in range(10000):
        async with lock:  # Lock around entire operation
            temp = counter
            await asyncio.sleep(0.00001)  # Safe now!
            counter = temp + 1

async def main():
    tasks = [increment_with_lock() for _ in range(5)]
    await asyncio.gather(*tasks)

asyncio.run(main())
print(counter)  # Always 50000 ✓
```

**Key difference from threading:**
- Threading: GIL releases during I/O, sleep, C extensions
- Asyncio: Task switches only at explicit `await` points

Same principle applies - lock protects the read-modify-write sequence from task switching.

### Go

```go
import "sync"

// 1. Mutex
var mu sync.Mutex
mu.Lock()
counter++
mu.Unlock()

// 2. Atomic operations
import "sync/atomic"
atomic.AddInt64(&counter, 1)

// 3. Channels (idiomatic Go)
ch := make(chan int)
ch <- value  // Thread-safe by design
```

---

## Key Differences: Python vs Go

| Aspect | Python | Go |
|--------|--------|-----|
| **Parallelism** | No (GIL) | Yes |
| **Race Conditions** | Only when GIL releases mid-operation | Easy to trigger anytime |
| **Best For** | I/O-bound tasks | CPU + I/O-bound |
| **Cores Used** | 1 for CPU work | All cores |

---

## Critical Terms

- **Critical Section**: Code accessing shared resources
- **Mutex**: Lock ensuring only one thread enters critical section
- **Atomic Operation**: Completes in one step (indivisible)
- **Deadlock**: Threads waiting for each other infinitely

---

## Mental Model

```
WITHOUT LOCK (multi-step operation):
Thread 1: ----R----W----
Thread 2: --R----W------  ← Overlap causes race!

WITH LOCK:
Thread 1: [--R--W--]-----
Thread 2: ----------[--R--W--]  ✓ Serialized
```

---

## When to Use Locks

**Need locks when:**
- Operations span multiple steps with potential GIL release
- I/O or sleep happens between read and write
- Complex state updates that must be atomic

**Don't need locks for:**
- Simple operations like `counter += 1` (fast, GIL rarely switches)
- Read-only access
- Thread-local data
- Immutable data structures

---

## Best Practices

**Python:**
- Use `threading.Lock` when operations have windows for thread switching
- Use `Queue` for producer-consumer patterns
- Use `multiprocessing` for true CPU parallelism

**Go:**
- Prefer **channels** for communication
- Use **mutex** when shared state unavoidable
- Use **atomic** for simple counters
- Mantra: "Don't communicate by sharing memory; share memory by communicating"

---

## Key Takeaway

**GIL provides practical safety for fast, simple operations** - but any operation with a window for thread switching (sleep, I/O, multiple steps) needs explicit locking.

---

#PYTHON #GOLANG #CONCURRENCY
