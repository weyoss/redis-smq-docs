# Event Bus

The event bus is the internal publish/subscribe mechanism of RedisSMQ. It uses Redis Pub/Sub to deliver real‑time notifications about system activity. Other components inside the same instance, other connected instances, and external monitors can all subscribe to these events.

## How It Works

Events are published to Redis channels. Each channel corresponds to a specific event type, for example:

```
queue.queueCreated
consumer.messageAcknowledged
events:producer.up
```

All channels are namespaced under `redis-smq:events:`. Subscribers listen on one or more channels and receive event payloads as JSON messages.

## Characteristics

- **Fire‑and‑forget** – Events are not persisted. If no subscriber is listening at the moment an event is published, that event is lost.
- **No delivery guarantees** – There is no acknowledgment mechanism for events. A subscriber that disconnects and reconnects will miss all events that occurred while it was away.
- **Real‑time only** – Subscribers only receive events published **after** they subscribe. There is no history or replay.
- **Optional** – The event bus can be disabled. When disabled, events are not published at all, reducing Redis load but also preventing cross‑instance state synchronisation.
- **Internal synchronisation** – Within the system, the event bus is the primary mechanism for keeping multiple instances in sync. For example:
  - Producers update their local cache of Pub/Sub consumer groups when a consumer group is created or deleted.
  - Consumers react to queue state changes (e.g., stop processing when a queue is paused) without polling Redis.

## Event Reference

Events are grouped by the domain that emits them.

### Configuration Events

#### `configuration.updated`

Fires when the system configuration is saved or updated.

| Field     | Type     | Description                                                       |
|-----------|----------|-------------------------------------------------------------------|
| `config`  | object   | Full configuration object (see [Configuration](configuration.md)) |
| `version` | number   | Configuration version                                             |

### Queue Events

#### `queue.queueCreated`

Fires when a new queue is created.

| Field        | Type   | Description                                                   |
|--------------|--------|---------------------------------------------------------------|
| `queue`      | object | `{ ns: string, name: string }`                                |
| `properties` | object | Queue properties (type, delivery model, message counts, etc.) |

#### `queue.queueDeleted`

Fires when a queue is deleted.

| Field   | Type   | Description                    |
|---------|--------|--------------------------------|
| `queue` | object | `{ ns: string, name: string }` |

#### `queue.stateChanged`

Fires when the operational state of a queue changes.

| Field        | Type   | Description                                                                                                |
|--------------|--------|------------------------------------------------------------------------------------------------------------|
| `queue`      | object | `{ ns: string, name: string }`                                                                             |
| `transition` | object | State transition object (`from`, `to`, `reason`, `timestamp`, optional `description`/`lockId`/`lockOwner`) |

#### `queue.consumerGroupCreated`

Fires when a consumer group is created for a Pub/Sub queue.

| Field     | Type   | Description                    |
|-----------|--------|--------------------------------|
| `queue`   | object | `{ ns: string, name: string }` |
| `groupId` | string | Consumer group identifier      |

#### `queue.consumerGroupDeleted`

Fires when a consumer group is deleted from a Pub/Sub queue.

| Field     | Type   | Description                    |
|-----------|--------|--------------------------------|
| `queue`   | object | `{ ns: string, name: string }` |
| `groupId` | string | Consumer group identifier      |

### Producer Events

#### `producer.up`

Fires when a producer has fully started and is ready to publish messages.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `producerId` | string | Producer ID |

#### `producer.goingUp`

Fires just before a producer starts its initialisation.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `producerId` | string | Producer ID |

#### `producer.down`

Fires after a producer has completely shut down.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `producerId` | string | Producer ID |

#### `producer.goingDown`

Fires just before a producer begins its shutdown sequence.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `producerId` | string | Producer ID |

#### `producer.messagePublished`

Fires when a message is successfully published to a queue or via an exchange.

| Field        | Type   | Description                                                   |
|--------------|--------|---------------------------------------------------------------|
| `messageId`  | string | Published message ID                                          |
| `queue`      | object | `{ ns: string, name: string }` (destination queue)            |
| `producerId` | string | Producer ID                                                   |
| `groupId`    | string | Consumer group ID (only for Pub/Sub queues, empty otherwise)  |

### Consumer Events

#### `consumer.up`

Fires when a consumer has fully started.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `consumerId` | string | Consumer ID |

#### `consumer.goingUp`

Fires just before a consumer starts its initialisation.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `consumerId` | string | Consumer ID |

#### `consumer.down`

Fires after a consumer has completely shut down.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `consumerId` | string | Consumer ID |

#### `consumer.goingDown`

Fires just before a consumer begins its shutdown sequence.

| Field        | Type   | Description |
|--------------|--------|-------------|
| `consumerId` | string | Consumer ID |

#### `consumer.messageReceived`

Fires when a message has been dequeued and is about to be passed to the handler.

| Field        | Type   | Description                    |
|--------------|--------|--------------------------------|
| `messageId`  | string | Message ID                     |
| `queue`      | object | `{ ns: string, name: string }` |
| `consumerId` | string | Consumer ID                    |

#### `consumer.messageAcknowledged`

Fires when a message was successfully processed by the handler.

| Field              | Type   | Description                              |
|--------------------|--------|------------------------------------------|
| `messageId`        | string | Message ID                               |
| `queue`            | object | `{ ns: string, name: string }`           |
| `messageHandlerId` | string | Message handler ID (same as consumer ID) |
| `consumerId`       | string | Consumer ID                              |

#### `consumer.messageUnacknowledged`

Fires when message processing failed. The handler returned an error, or the message expired.

| Field              | Type   | Description                                                 |
|--------------------|--------|-------------------------------------------------------------|
| `messageId`        | string | Message ID                                                  |
| `queue`            | object | `{ ns: string, name: string }`                              |
| `messageHandlerId` | string | Message handler ID                                          |
| `consumerId`       | string | Consumer ID                                                 |
| `cause`            | number | Unacknowledgment cause (see implementation for enum values) |

#### `consumer.messageDeadLettered`

Fires when a message was moved to the dead‑letter queue after exhausting retries or expiring.

| Field              | Type   | Description                                            |
|--------------------|--------|--------------------------------------------------------|
| `messageId`        | string | Message ID                                             |
| `queue`            | object | `{ ns: string, name: string }`                         |
| `messageHandlerId` | string | Message handler ID                                     |
| `consumerId`       | string | Consumer ID                                            |
| `cause`            | number | Dead‑letter cause (see implementation for enum values) |

#### `consumer.messageRequeued`

Fires when a message was requeued for immediate retry.

| Field              | Type   | Description                    |
|--------------------|--------|--------------------------------|
| `messageId`        | string | Message ID                     |
| `queue`            | object | `{ ns: string, name: string }` |
| `messageHandlerId` | string | Message handler ID             |
| `consumerId`       | string | Consumer ID                    |

#### `consumer.messageDelayed`

Fires when a message was delayed and will be retried later.

| Field              | Type   | Description                    |
|--------------------|--------|--------------------------------|
| `messageId`        | string | Message ID                     |
| `queue`            | object | `{ ns: string, name: string }` |
| `messageHandlerId` | string | Message handler ID             |
| `consumerId`       | string | Consumer ID                    |

## Subscribing to Events

Subscribe to one or more event channels by name. Handlers receive a JSON payload with the fields described above. Multiple handlers can subscribe to the same event, and they will all be invoked when the event is published.

The subscription API and exact handler signature depend on the language implementation, but they all follow the same pattern: register a callback for a given event name, and call `unsubscribe()` to stop receiving events.

## Use Cases

- **Monitoring** – Track message flow and system health in real time.
- **Alerting** – Detect failures, dead‑lettered messages, or queue state changes.
- **Metrics** – Count messages processed, failed, requeued, and delayed.
- **Audit** – Log all system activity alongside the internal message audit.
- **Integration** – Trigger external actions on specific events (e.g., send an HTTP webhook when a message is dead‑lettered).

## Performance

The event bus uses Redis Pub/Sub, which is efficient but fire‑and‑forget. Publishing an event adds a single Redis `PUBLISH` command. Subscribers receive messages asynchronously.

- Events are not persisted; they exist only in transit.
- Publishing is fast and non‑blocking.
- High‑frequency events (e.g., every message acknowledgment) can generate significant Pub/Sub traffic. Disable the event bus entirely if it is not needed.

For critical, persistent audit trails, use the built‑in message audit feature instead of relying solely on events.

## Best Practices

- Keep event handlers fast. They run synchronously in the subscriber’s event loop; long‑running handlers can delay other events.
- Do not rely on events for guaranteed data delivery. They are best for reactive, non‑critical logic.
- Unsubscribe when no longer needed to free resources and avoid memory leaks.
- Treat the event bus as optional. If cross‑instance synchronisation or real‑time monitoring is not required, disable it to reduce Redis load.
