# Graceful Shutdown

RedisSMQ is designed to handle shutdowns without losing messages. Core operations are atomic, and proper shutdown ensures all in‑flight messages are recovered.

## Shutdown Sequence

When the system shuts down, components stop in this order:

```
1. Process-scoped background workers stop
   ├─→ Coordinator stops its per-queue workers,
   │   releases WorkersLock leases
   └─→ Purge worker drains its in-flight job,
       unlocks the queue it was purging

2. Consumers stop
   ├─→ Stop accepting new messages
   ├─→ Complete or return in-flight messages to the queue
   ├─→ Disconnect from queues
   └─→ Clean up any temporary resources (e.g., ephemeral groups)

3. Producers stop
   ├─→ Stop accepting new publish requests
   └─→ Complete in-flight publishes

4. System services stop
   ├─→ User event bus shuts down
   ├─→ System event bus shuts down
   ├─→ Configuration manager closes
   └─→ Logger shuts down

5. Internal Redis references cleared
   (The Redis client itself remains open; the caller closes it.)
```

**Why this order matters:**

- **Background workers first** — the Coordinator releases its WorkersLock leases before consumers stop, so another process can elect workers immediately rather than waiting for the lease TTL to expire. The Purge Worker finishes its in-flight job and unlocks the queue it was purging, so the queue is in a clean state for the next process.
- **Consumers before producers** — consumers must finish their in-flight acks/unacks while Redis is still reachable, and stop publishing "down" events after their final state transitions. Producers then flush their in-flight publishes.
- **System services last** — the event buses stay up until after consumers and producers have published their final events (`consumer.goingDown`, `consumer.down`, `producer.goingDown`, `producer.down`). Then they shut down, and finally Redis references are cleared.

The order among steps 1–3 is implementation-specific — any of (background workers, consumers, producers) can be sequenced first, as long as all three complete before step 4. The invariant is: **all in-flight Redis operations from RedisSMQ code must complete before internal Redis references are cleared.**

## In‑Flight Messages

When a consumer shuts down gracefully:

- Messages currently being processed are returned to the pending queue.
- Other consumers can pick them up immediately.
- No messages are lost.

If a consumer crashes without a clean shutdown:

- Heartbeats stop.
- The reaper (a background worker elected per queue by the Coordinator) detects the dead consumer within one or two tick intervals.
- In‑flight messages are recovered and returned to the pending queue.

## Crash Recovery

RedisSMQ is resilient to unexpected failures. Two recovery mechanisms cover the two failure modes:

### Consumer crash

The reaper is a background worker elected per queue across the cluster — one process per queue holds the queue's WorkersLock lease and runs the reaper. When a consumer's heartbeat expires, the reaper:

1. Unacknowledges all messages in the dead consumer's processing queue.
2. Returns them to the pending queue for other consumers.
3. Cleans up the dead consumer's subscriptions and ephemeral resources.

### Process holding a queue's lease crash

When the process that holds a queue's WorkersLock lease crashes:

1. Its lease expires after the TTL (default 30 seconds).
2. The Coordinator's next reconcile tick on any live process re-elects and re-spawns the worker set for that queue.

No messages are lost in either scenario.

## Best Practices

- **Use a single system shutdown** at application exit.
- **Handle OS signals** (SIGINT, SIGTERM) to trigger shutdown.
- **Wait for shutdown to complete** before exiting the process.
- **Don’t force exit** — let the cleanup sequence finish.
- **Set appropriate heartbeat TTLs** for your deployment environment. Shorter TTLs mean faster detection of dead consumers but more frequent heartbeat updates.
- **Shut down RedisSMQ before closing the Redis client.** The Redis client is caller-owned; RedisSMQ releases its internal references but does not close the client.

## Related Pages

- [Architecture Overview](architecture.md) — system design and component interaction
- [Message Reliability](message-reliability.md) — delivery guarantees and recovery mechanisms
