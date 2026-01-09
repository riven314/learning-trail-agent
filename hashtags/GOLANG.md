# #GOLANG

- [Go Fundamentals](../2025-12-29/golang-fundamentals.md) - Core language concepts: struct tags, pointers, interfaces, goroutines, channels, context, defer patterns
- [Go Pointer Receivers](../2025-12-30/go-pointer-receivers.md) - Automatic value-to-pointer conversion in method calls vs explicit pointer requirement for function parameters
- [Thread Safety](../2025-12-30/thread-safety.md) - Thread safety concepts with Go's mutex, atomic operations, and channels for concurrent programming
- [Go Concurrency Fundamentals](../2025-12-30/go-concurrency-fundamentals.md) - M:N threading model, goroutine lifecycle, comparison with Python's threading/multiprocessing/asyncio
- [Go Concurrency Fundamentals](../2025-12-31/go-concurrency-fundamentals.md) - Goroutines, race conditions, Mutex/RWMutex, sync.Map, per-instance lock pattern in trading systems
- [Go's make Function](../2025-12-31/go-make-function.md) - Memory allocation and initialization for slices, maps, channels; make vs new; performance optimization via pre-allocation
- [Go Context and Channels](../2026-01-01/go-context-and-channels.md) - Context package for cancellation propagation, channel closure behaviors, graceful shutdown patterns in concurrent programs
- [Go Atomic Operations and PriceMonitor](../2026-01-01/go-atomic-operations-and-pricemonitor.md) - Atomic operations, torn reads, atomic.Value vs atomic.Pointer[T], hybrid push-pull pattern with channels and atomics
- [Go Interface Boxing and Nil Internals](../2026-01-01/go-interface-boxing-nil-internals.md) - Interface iface struct (tab/data pointers), boxing mechanism, typed nil problem root cause, heap allocation, error return best practices
- [Go Error Interface](../2026-01-01/go-error-interface.md) - error as interface type, common implementations, errors.As() for type extraction, typed nil gotcha in error returns
- [Go Type Assertions and Reflection](../2026-01-02/go-type-assertions-and-reflection.md) - Runtime type handling: type assertions for known types, reflection for dynamic field access, architectural patterns for solving circular imports
- [Go Type Assertions, Reflection & Boundary Patterns](../2026-01-03/go-type-assertions-reflection-boundaries.md) - Type system deep dive: type assertions vs reflection, nominal typing, circular import solutions, callback bridge pattern, boundary layer architecture for multi-exchange trading
- [Go Interfaces and Generics](../2026-01-03/go-interfaces-and-generics.md) - Interface semantics (implicit implementation, empty interface internals), generics (type parameters, constraints, GCShape stenciling), when to use each abstraction approach
- [Go Composition and Embedding](../2026-01-03/go-composition-and-embedding.md) - Composition-over-inheritance approach, embedding as automatic delegation, field/method promotion, collision handling, flexibility benefits over traditional inheritance
- [TOCTOU Race Condition in Go](../2026-01-08/toctou-race-condition-golang.md) - Time-of-Check-Time-of-Use race condition pattern, atomic check-and-set fix, critical sections for check-modify sequences
- [Copy-Under-Lock Pattern](../2026-01-10/copy-under-lock-pattern.md) - Concurrency pattern for safe callback invocation: copy data under lock, release lock, invoke callback with snapshot to minimize lock duration and prevent deadlocks
