# Graceful Shutdown

RedisSMQ is designed to handle shutdowns without losing messages. Core operations are atomic, and proper shutdown ensures all in‑flight messages are recovered.

## Shutdown Sequence

When the system shuts down, components stop in this order:

```
1. Consumers stop
   ├─→ Stop accepting new messages
   ├─→ Complete or return in‑flight messages to the queue
   ├─→ Disconnect from queues
   └─→ Clean up any temporary resources (e.g., ephemeral groups)

2. Producers stop
   ├─→ Stop accepting new publish requests
   └─→ Complete in‑flight publishes

3. System services stop
   ├─→ Configuration service closes
   ├─→ Event bus stops listening
   └─→ Background workers (scheduler, reaper, etc.) stop

4. Redis connection pool closes
```

The order is important: consumers must stop first so that in‑flight messages are returned before the event bus or Redis connections are torn down. 

Managers are stateless and require no special shutdown handling.

## In‑Flight Messages

When a consumer shuts down gracefully:

- Messages currently being processed are returned to the pending queue.
- Other consumers can pick them up immediately.
- No messages are lost.

If a consumer crashes without a clean shutdown:

- Heartbeats stop.
- A background reaper detects the dead consumer.
- In‑flight messages are recovered and returned to the pending queue.

## Crash Recovery

RedisSMQ is resilient to unexpected failures. The reaper (a background worker that runs periodically in every active consumer) monitors heartbeats. When a consumer’s heartbeat expires, the reaper:

1. Unacknowledges all messages in the dead consumer’s processing queue.
2. Returns them to the pending queue for other consumers.
3. Cleans up the dead consumer’s subscriptions and ephemeral resources.

This ensures that messages are never lost, even when a consumer process dies without any cleanup.

## Best Practices

- **Use a single system shutdown** at application exit.
- **Handle OS signals** (SIGINT, SIGTERM) to trigger shutdown.
- **Wait for shutdown to complete** before exiting the process.
- **Don’t force exit** — let the cleanup sequence finish.
- **Set appropriate heartbeat TTLs** for your deployment environment. Shorter TTLs mean faster detection of dead consumers but more frequent heartbeat updates.

## Related Pages

- [Architecture Overview](architecture.md) — system design and component interaction
- [Message Reliability](message-reliability.md) — delivery guarantees and recovery mechanisms
