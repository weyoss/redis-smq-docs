# Redis Key Schema

All Redis keys used by RedisSMQ follow a consistent hierarchical pattern:

```
redis-smq:{version}:{domain}:{...path components}
```

- **`redis-smq`** — constant prefix for the project.
- **`{version}`** — a numeric version (e.g., `10`) that is incremented when the key schema changes. This allows multiple schema versions to coexist in the same Redis database.
- **`{domain}`** — high‑level category (`main`, `ns`, etc.).
- **`{...path components}`** — additional segments joined by the separator (`:`).

The separator is always `:`.

## System Keys (`main` domain)

| Description | Pattern |
|-------------|---------|
| Configuration | `redis-smq:{version}:main:cfg` |
| Registry of all queues | `redis-smq:{version}:main:q` |
| Registry of all exchanges | `redis-smq:{version}:main:exs` |
| Registry of all namespaces | `redis-smq:{version}:main:ns` |
| Consumer heartbeat | `redis-smq:{version}:main:cons:{consumerId}:hb` |
| Queues a consumer is subscribed to | `redis-smq:{version}:main:cons:{consumerId}:q` |
| Message data | `redis-smq:{version}:main:msg:{messageId}` |
| Message unacknowledgment history | `redis-smq:{version}:main:msg:{messageId}:uh` |
| Purge jobs hash | `redis-smq:{version}:main:pg-jobs` |
| Pending purge jobs list | `redis-smq:{version}:main:pg-jobs:pend` |
| Active (processing) purge jobs list | `redis-smq:{version}:main:pg-jobs:proc` |
| Worker heartbeat | `redis-smq:{version}:main:wrk:{workerId}:hb` |
| Job‑to‑worker mapping | `redis-smq:{version}:main:jobs:{jobId}:wrk` |

## Namespace Keys

| Description | Pattern |
|-------------|---------|
| All queues in a namespace | `redis-smq:{version}:ns:{namespace}:q` |
| All exchanges in a namespace | `redis-smq:{version}:ns:{namespace}:exs` |

## Queue Keys

`{queueName}` and `{namespace}` identify the queue.

### Queue Properties and Counters (Hash)

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:prop` | Queue metadata: type, delivery model, message counts, operational state, rate limit, lock ID, etc. |

### Message Categories

| Category | Key (Redis type) |
|----------|------------------|
| Published (all messages in the queue) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:pub` (List) |
| Pending (waiting for consumer) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:pend` (List) — for FIFO/LIFO queues.<br>`redis-smq:{version}:ns:{namespace}:q:{queueName}:prio` (Sorted Set) — for Priority queues. |
| Scheduled (future delivery) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:sched` (Sorted Set) |
| Processing (currently being handled) | Per‑consumer: `redis-smq:{version}:ns:{namespace}:q:{queueName}:cons:{consumerId}:proc` (List) |
| Acknowledged (successfully processed) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:ack` (List) — only if audit enabled. |
| Dead‑Letter (failed permanently) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:dl` (List) — only if audit enabled. |
| Delayed (waiting for retry delay) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:dly` (Sorted Set) |
| Requeued (waiting to be re‑inserted into pending) | `redis-smq:{version}:ns:{namespace}:q:{queueName}:req` (List) |

### Consumer and Consumer Group Management

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cons` | Hash of consumer IDs → consumer info. |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cgp` | Set of consumer group IDs. |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cgp:{groupId}:pend` | Pending list for a consumer group (Pub/Sub). |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cgp:{groupId}:prio` | Priority sorted set for a consumer group (Pub/Sub). |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cgp:{groupId}:cons` | Set of consumers belonging to a consumer group. |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:cons:{consumerId}:proc` | Processing list for a specific consumer. |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:proc-q` | Hash mapping processing queue keys to consumer IDs. |

### Other Queue Keys

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:sh` | State transition history (List). |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:bind` | Set of exchange names bound to this queue. |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:wlock` | Workers distributed lock (String). |
| `redis-smq:{version}:ns:{namespace}:q:{queueName}:rate` | Rate limit counter (String). |

## Exchange Keys

`{exchangeName}` and `{namespace}` identify the exchange.

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}` | Exchange identifier key. |
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:prop` | Exchange properties (type, policy, timestamps). |

### Direct Exchange

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:rk` | Set of all routing keys. |
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:rk:{routingKey}:q` | Set of queues bound to the routing key. |

### Topic Exchange

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:pat` | Set of all binding patterns. |
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:pat:{pattern}:q` | Set of queues bound to the pattern. |

### Fanout Exchange

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:ns:{namespace}:exs:{exchangeName}:q` | Set of all bound queues. |

## Message Keys

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:main:msg:{messageId}` | Message hash with all properties (status, payload, timestamps, counters). |
| `redis-smq:{version}:main:msg:{messageId}:uh` | Unacknowledgment history list (if audit enabled). |

## Purge / Background Job Keys

| Key | Description |
|-----|-------------|
| `redis-smq:{version}:main:pg-jobs` | Hash of all purge jobs. |
| `redis-smq:{version}:main:pg-jobs:pend` | Pending jobs list (used as a queue for workers). |
| `redis-smq:{version}:main:pg-jobs:proc` | Processing jobs list (jobs currently being executed). |
| `redis-smq:{version}:main:jobs:{jobId}:wrk` | String mapping job ID → worker ID. |
| `redis-smq:{version}:main:wrk:{workerId}:hb` | Worker heartbeat (String with TTL). |

## Notes

- The `{version}` component is a hardcoded constant in each implementation (currently `10`). It is **not** user‑configurable and must be incremented when the schema changes in a backward‑incompatible way.
- All queue‑specific message lists (pending, processing, scheduled, etc.) use the **same** queue namespace and name as the queue properties key, ensuring data locality.
- For Pub/Sub queues, each consumer group has its own pending and priority keys, allowing independent consumption per group.
- Consumer‑specific processing keys are derived from the consumer ID, ensuring each consumer has its own processing queue.
- All keys are documented here for reference; implementations may use additional transient keys for locks, rate limiting counters, etc., but they follow the same pattern.
