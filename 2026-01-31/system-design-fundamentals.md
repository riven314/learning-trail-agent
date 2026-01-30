# System Design Fundamentals

Context: consolidated system design interview notes covering core architecture patterns, message queues, database optimization, service discovery, and fault tolerance strategies.

## Architecture Patterns: Direct Write vs Message Queue

### When to use each

- **Direct Write**: simple use cases, low latency requirements, tight coupling acceptable
- **Message Queue**: need durability, replay capability, decoupling between producers and consumers

### Key trade-offs

| Aspect | Direct Write | Via Message Queue |
|--------|--------------|-------------------|
| Latency | Lower | Higher (batching) |
| Durability | Data lost if DB down | Queue retains data |
| Replay | Impossible | Reset offset, reprocess |
| Complexity | Simple | More components |
| Coupling | Tight | Loose |

## Message Queues (Kafka)

### Core concepts

- **Topic**: named stream of messages
- **Partition**: topics split for parallelism; ordering guaranteed within partition
- **Consumer Group**: multiple consumers share work; each partition read by exactly one consumer
- **Offset**: position tracker; enables resume after crash

### Cluster sizing rules

```
2 nodes: 1 dies → 1/2 = no majority → cluster DOWN
3 nodes: 1 dies → 2/3 = majority → cluster CONTINUES
```

**Rule**: use 1 node (dev) or 3+ nodes (prod). Never 2—you pay for redundancy without the benefit.

### Critical configurations

```
acks=all                   # wait for all replicas (safest)
retries=5                  # retry on failure
replication.factor=3       # data copied to 3 brokers
min.insync.replicas=2      # require 2 replicas for write success
```

### Resource requirements explained

- **RAM**: JVM heap + page cache (enables fast reads without disk)
- **Disk IOPS**: sequential writes + replication traffic
- **Network**: replication traffic between brokers
- **CPU**: compression, I/O threads (less critical)

**Minimum**: 2GB RAM per broker. Below this, GC pauses cause broker timeouts.

## Database Write Optimization

### Problem: write amplification

Many small INSERTs → many small parts on disk → expensive background merges.

### Solutions

**1. Async insert (server-side buffering)**
```sql
INSERT INTO table VALUES (...)
SETTINGS async_insert=1, wait_for_async_insert=0
```

**2. Buffer table (client-side batching at DB level)**
```sql
CREATE TABLE buffer AS real_table
ENGINE = Buffer(db, real_table,
    16,        -- parallel buffers
    10, 100,   -- min/max seconds
    10000, 1000000  -- min/max rows
)
```

| Approach | Complexity | Data Visibility |
|----------|------------|-----------------|
| Async Insert | Low (just a setting) | After flush |
| Buffer Table | Medium (schema change) | Immediate |

## Service Discovery & Health Checks

### Heartbeat pattern

```python
# service sends heartbeat with TTL
redis.setex(f"heartbeat:{service_id}", ttl=30, value=timestamp)

# controller detects missing heartbeat
if not redis.exists(f"heartbeat:{service_id}"):
    provision_replacement()
```

### Reconciliation loop pattern

Key principle: declare desired state, let controller converge actual to desired.

```
1. Read desired state (config)
2. Read actual state (heartbeats)
3. Compare: desired vs actual
4. Act: provision/terminate to match
5. Sleep, repeat
```

### Redis vs Consul

| Feature | Redis | Consul |
|---------|-------|--------|
| Health checks | DIY (TTL keys) | Built-in |
| Service discovery | Manual | Native (DNS + API) |
| Complexity | Simple | More features |

## Fault Tolerance Patterns

### Local buffering for outages

```python
def send(message):
    try:
        downstream.send(message)
    except ConnectionError:
        local_buffer.put(message)  # buffer locally, background thread retries later
```

| Buffer Type | Survives Crash | Capacity | Use When |
|-------------|----------------|----------|----------|
| In-memory (Queue) | No | Limited | Short outages |
| Disk (SQLite/File) | Yes | Large | Critical data |

### Retry with exponential backoff

```python
wait_time = min(base * (2 ** attempt), max_wait)
# attempt 1: 1s, attempt 2: 2s, attempt 3: 4s, ...
```

**Why exponential**: prevents thundering herd when service recovers.

#SYSTEM-DESIGN #DISTRIBUTED-SYSTEMS #MESSAGE-QUEUE #KAFKA #DATABASE
