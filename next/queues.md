# Queues

A queue holds messages until a consumer processes them. RedisSMQ supports three queue types with different ordering guarantees.

## Queue Types

### FIFO (First In, First Out)

Messages are processed in the order they arrived. The oldest message is delivered first.

**Best for:** Job processing, task queues, ordered workflows where sequence matters.

### LIFO (Last In, First Out)

The most recently published message is delivered first. Older messages are processed later.

**Best for:** Real-time updates, notifications where the latest data is most relevant.

### Priority

Messages are ordered by priority level (0–7, where 0 is highest). Higher-priority messages are delivered before lower-priority ones, regardless of arrival order.

**Best for:** Alerting systems, VIP processing, where some messages must be handled before others.

## Priority Levels

| Level | Name         |
|-------|--------------|
| 0     | Highest      |
| 1     | Very High    |
| 2     | High         |
| 3     | Above Normal |
| 4     | Normal       |
| 5     | Low          |
| 6     | Very Low     |
| 7     | Lowest       |

Priority queues have more overhead than FIFO/LIFO. Use them only when ordering by priority is required.

## Queue Names

Queue names must follow these rules:

- Start with a letter (a–z)
- Contain only lowercase letters, digits, hyphens, underscores, and dots
- Cannot be empty

```
✅ orders
✅ email-queue
✅ user_events
✅ app.notifications

❌ 3queue (starts with digit)
❌ my queue (contains space)
❌ Queue (contains uppercase)
```

## Namespaces

Queues exist within a **namespace** — a logical grouping that isolates queues from each other. Use namespaces to separate environments (production, staging) or applications.

If no namespace is specified, the configured default namespace is used. The default is `"default"` unless changed in configuration.

```
production/orders
staging/orders
analytics/events
```

## Delivery Models

Each queue uses one of two delivery models:

- **Point-to-Point** — Each message is delivered to exactly one consumer
- **Pub/Sub** — Each message is delivered to all consumer groups

See [Queue Delivery Models](queue-delivery-models.md) for details.

## Queue State

Queues have an operational state that controls whether they accept and process messages:

- **Active** — Normal operation
- **Paused** — Accepts new messages but does not deliver them
- **Stopped** — Neither accepts nor delivers messages
- **Locked** — Reserved for exclusive maintenance operations

See [Queue State Management](queue-state-management.md) for details.

## Performance

FIFO and LIFO queues are the fastest. Priority queues have additional overhead due to the sorting required for each operation. Choose the simplest queue type that meets your needs.

See [Performance](performance.md) for details.
