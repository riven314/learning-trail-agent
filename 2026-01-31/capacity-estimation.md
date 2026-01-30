# Capacity Estimation & Interview Framework

Context: quick math templates for capacity planning and structured approach for system design interviews.

## Capacity Estimation Template

### Quick math formula

```
Messages/sec = (entities) × (events per entity per sec)
Throughput   = (messages/sec) × (avg message size)
Storage/day  = (throughput) × 86400
Storage/week = Storage/day × 7
```

### Example calculation

```
100 trading pairs × 1 msg/sec = 100 msg/sec
100 msg/sec × 1KB = 100 KB/sec
100 KB/sec × 86400 = 8.6 GB/day
8.6 GB × 7 days retention = 60 GB storage
```

## Key Trade-offs

| Trade-off | Option A | Option B |
|-----------|----------|----------|
| Consistency vs Availability | Strong consistency (wait for replicas) | Eventual consistency (faster writes) |
| Latency vs Durability | Async writes (fast, lossy) | Sync writes (slow, durable) |
| Cost vs Reliability | Fewer nodes (cheaper) | More nodes (fault tolerant) |
| Simplicity vs Flexibility | Direct writes (simple) | Message queue (decoupled) |

## Interview Framework

### Structure your answer

1. **Clarify requirements**: scale, latency, durability, cost constraints
2. **Start simple**: direct approach first
3. **Identify bottlenecks**: what breaks at scale?
4. **Add complexity only when needed**: queues, caches, sharding
5. **Discuss trade-offs**: why this choice over alternatives?

### Questions to ask

- Expected scale (QPS, data volume)?
- Latency requirements (p50, p99)?
- Durability requirements (can we lose data)?
- Consistency requirements (strong vs eventual)?
- Budget constraints?

## Quick Reference: When to Use What

| Need | Solution |
|------|----------|
| Decouple producers/consumers | Message queue (Kafka, SQS) |
| Fast key-value lookups | Redis, Memcached |
| Time-series analytics | ClickHouse, TimescaleDB |
| Service discovery | Consul, etcd, Redis |
| Auto-scaling VMs | Terraform + Cloud-Init |
| Container orchestration | Kubernetes (if you really need it) |
| Distributed consensus | 3+ nodes with quorum |

#SYSTEM-DESIGN #INTERVIEW-PREP #CAPACITY-PLANNING
