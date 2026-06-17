# RedisSMQ Documentation

This folder contains the **bleeding‑edge language‑agnostic specification** for RedisSMQ. It describes concepts, architecture, data structures, and operational patterns that apply to every implementation.

> ⚠️ This documentation tracks the `next` development branch and may change without notice.  
> For stable specifications, refer to the versioned folders (e.g., `/v1.0/`) once released.

---

## 📚 Document Index

### Start Here

| Document                                 | Description                                                       |
|------------------------------------------|-------------------------------------------------------------------|
| [Getting Started](getting-started.md)    | What RedisSMQ is, core concepts, and how the pieces fit together. |
| [Architecture Overview](architecture.md) | System design, components, data structures, and message flow.     |

### Core Concepts

| Document                                          | Description                                                                  |
|---------------------------------------------------|------------------------------------------------------------------------------|
| [Message Lifecycle](message-lifecycle.md)         | Complete state diagram and walkthrough from publication to final resolution. |
| [Messages](messages.md)                           | Message properties, TTL, priority, retry policy, and scheduling.             |
| [Queues](queues.md)                               | FIFO, LIFO, and Priority queue types plus namespaces.                        |
| [Queue Delivery Models](queue-delivery-models.md) | Point‑to‑Point vs Pub/Sub and consumer groups.                               |
| [Message Exchanges](message-exchanges.md)         | Direct, Topic, and Fanout routing patterns.                                  |
| [Scheduling Messages](scheduling-messages.md)     | Delays, CRON schedules, and repeating delivery.                              |
| [Consumer Groups](consumer-groups.md)             | Broadcasting to multiple services with horizontal scaling.                   |

### Operations

| Document                                            | Description                                                         |
|-----------------------------------------------------|---------------------------------------------------------------------|
| [Queue Rate Limiting](queue-rate-limiting.md)       | Controlling message throughput per queue.                           |
| [Queue State Management](queue-state-management.md) | Pausing, stopping, resuming, and locking queues.                    |
| [Message Audit](message-audit.md)                   | Tracking acknowledged, dead‑lettered, and unacknowledgment history. |
| [Configuration](configuration.md)                   | System‑wide settings and defaults.                                  |

### Reliability & Performance

| Document                                      | Description                                              |
|-----------------------------------------------|----------------------------------------------------------|
| [Message Reliability](message-reliability.md) | At‑least‑once delivery, crash recovery, and idempotency. |
| [Performance](performance.md)                 | Throughput characteristics and tuning guidance.          |
| [Graceful Shutdown](graceful-shutdown.md)     | Clean shutdown procedures and crash recovery.            |

### Advanced

| Document                                | Description                                       |
|-----------------------------------------|---------------------------------------------------|
| [Event Bus](event-bus.md)               | Real‑time system events via Redis Pub/Sub.        |
| [Interoperability](interoperability.md) | Cross‑language message passing and shared schema. |
| [Redis Key Schema](redis-key-schema.md) | Complete reference of Redis keys and hash fields. |
| [Glossary](glossary.md)                 | Terminology used throughout the documentation.    |

---

## 🔄 Corresponding Implementations

This specification corresponds to the **development branch** of each implementation.  
For exact version mapping, see the [compatibility matrix](../README.md#compatibility-matrix) in the root README.

---

## 📄 License

MIT – see [LICENSE](../LICENSE).
