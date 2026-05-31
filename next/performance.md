# Performance

RedisSMQ is designed for high throughput. Absolute throughput depends on message size, network latency, Redis configuration, and handler logic, but the queue type, routing path, and optional features have the biggest built‑in impact.

## What affects performance

### Queue type

| Type     | Underlying Redis structure | Relative speed | Notes                                                               |
|----------|----------------------------|----------------|---------------------------------------------------------------------|
| FIFO     | List                       | Fastest        | LPUSH / RPOPLPUSH – O(1)                                            |
| LIFO     | List                       | Fastest        | Same operations as FIFO                                             |
| Priority | Sorted Set                 | Slower         | ZADD / ZPOPLPUSH – O(log N) per operation; uses a custom Lua script |

FIFO and LIFO queues rely on plain list operations, which are the fastest Redis primitives. Priority queues use sorted sets, which have a higher per‑operation cost due to the need to maintain ordering by score.

### Routing path

| Method          | Redis lookups                    | Relative speed  | Notes                                      |
|-----------------|----------------------------------|-----------------|--------------------------------------------|
| Direct to queue | None                             | Fastest         | No exchange resolution                     |
| Direct exchange | 1 set lookup per routing key     | Fast            | Exact string match                         |
| Topic exchange  | Pattern matching + set lookups   | Slower          | Regex compilation and matching per publish |
| Fanout exchange | Copies message to N bound queues | Slower          | Work multiplies by number of bound queues  |

Publishing directly to a queue avoids all exchange overhead. For exchange‑based routing, direct exchanges are the most efficient because they only require a set membership lookup. Topic exchanges add pattern matching (regex‑based), which is more expensive. Fanout exchanges copy the message to every bound queue, so throughput is inversely proportional to the number of bound queues.

### Delivery model

| Model          | Copies per message   | Overhead |
|----------------|----------------------|----------|
| Point‑to‑Point | 1                    | Lower    |
| Pub/Sub        | 1 per consumer group | Higher   |

Each consumer group in Pub/Sub requires a separate copy of the message, so publish throughput decreases as the number of groups increases. Within a group, load‑balancing uses the same list/set operations as point‑to‑point, so consumption throughput is comparable.

### Optional features

Each feature adds some overhead. Here is the approximate impact:

| Feature        | When active                        | Overhead   |
|----------------|------------------------------------|------------|
| Logging        | Console output                     | Medium     |
| Message audit  | Acknowledged / dead‑letter storage | Medium     |
| Event bus      | Redis Pub/Sub publish              | Low‑Medium |
| Rate limiting  | Per‑dequeue Lua script             | Low        |

When these features are disabled (the default for audit and logging), there is zero overhead. Enable only what you need in production.

## Tuning checklist

- **Use FIFO/LIFO queues** unless priority ordering is strictly required.
- **Publish directly to queues** instead of through exchanges when possible.
- **Prefer direct exchanges** over topic exchanges when pattern matching is not needed.
- **Keep message bodies small.** Payload size directly impacts serialisation/deserialisation and network transfer.
- **Disable logging** in production.
- **Disable message audit** unless you need it for debugging or compliance.
- **Set appropriate storage limits** (`queueSize`, `expire`) if audit is enabled.
- **Use batch acknowledgments** for high‑throughput consumers to reduce the number of Redis calls per message.

## Measuring performance

Throughput is typically measured in **messages per second**. Benchmark in three scenarios:

1. **Producer‑only** – How fast can messages be published to a queue?
2. **Consumer‑only** – How fast can messages be consumed from a pre‑filled queue?
3. **Combined** – End‑to‑end throughput with producers and consumers running concurrently.

The implementation includes ready‑to‑run benchmarks (e.g., `BenchmarkProducer_10K`, `BenchmarkConsumer_100K`, `BenchmarkCombined_10K`) that can be used as a starting point. Results depend heavily on hardware, network latency, message size, and handler complexity. Always benchmark in an environment representative of production.

## Redis considerations

Redis is single‑threaded, so it can become a bottleneck under very high load. To scale beyond a single Redis instance:

- Run Redis on the **same host** or on a low‑latency network.
- **Shard queues** across multiple Redis instances (each RedisSMQ instance can connect to only one Redis at a time).
- Tune Redis persistence settings (RDB/AOF) based on your consistency requirements – disable them for maximum throughput if data loss is acceptable during Redis restarts.

For most applications, a single well‑configured Redis instance can handle hundreds of thousands of messages per second.

## Related Pages

- [Queues](queues.md) – queue types and their characteristics
- [Message Exchanges](message-exchanges.md) – exchange types and routing performance
- [Queue Delivery Models](queue-delivery-models.md) – point‑to‑point vs pub/sub
- [Message Audit](message-audit.md) – audit overhead and configuration
