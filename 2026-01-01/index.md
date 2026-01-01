# Learning Log: 2026-01-01

- **[Go Context and Channels](go-context-and-channels.md)**: Deep dive into context.Context for cancellation propagation, channel closure behaviors, and graceful shutdown patterns in concurrent Go programs
- **[Go Atomic Operations and PriceMonitor](go-atomic-operations-and-pricemonitor.md)**: Atomic operations (atomic.Value vs atomic.Pointer[T]), torn reads, hybrid push-pull pattern, channels with select/default for backpressure handling
- **[Go Interface Boxing and Nil Internals](go-interface-boxing-nil-internals.md)**: Interface internal structure (iface struct with tab/data), boxing mechanism, typed nil vs true nil problem, heap allocation implications, and error return patterns
- **[Go Error Interface](go-error-interface.md)**: error as interface type with Error() string method, common implementations (errors.New, fmt.Errorf, custom types), errors.As() for type extraction, typed nil gotcha
