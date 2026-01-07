# #ALGO-TRADING

- [Go Fundamentals](../2025-12-29/golang-fundamentals.md) - Go language concepts learned through market maker codebase context
- [Go Concurrency Fundamentals](../2025-12-31/go-concurrency-fundamentals.md) - Thread-safe order management using sync.Map and per-slot RWMutex in high-frequency trading
- [Go Atomic Operations and PriceMonitor](../2026-01-01/go-atomic-operations-and-pricemonitor.md) - PriceMonitor architecture with dual data propagation (push/pull), backpressure handling, single source of truth pattern
- [Go Type Assertions, Reflection & Boundary Patterns](../2026-01-03/go-type-assertions-reflection-boundaries.md) - Multi-exchange adapter architecture: callback bridge pattern, type conversion at boundaries, avoiding reflection in hot paths for HFT performance
- [TOCTOU Race Condition in Go](../2026-01-08/toctou-race-condition-golang.md) - Fixed race condition in trading bot cooldown mechanism preventing silent shortening of risk management timeouts
