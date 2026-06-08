# Glossary

## General Concepts

| Term          | Definition                                                                                                                                                                   |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Message**   | A unit of data with delivery instructions. It flows from a producer through optional exchanges and queues to a consumer.                                                     |
| **Producer**  | A component that creates and sends messages.                                                                                                                                 |
| **Consumer**  | A component that receives and processes messages from a queue.                                                                                                               |
| **Queue**     | A named storage location where messages wait for consumers.                                                                                                                  |
| **Exchange**  | A routing component that delivers a message to one or more queues based on routing rules.                                                                                    |
| **Namespace** | A logical grouping that isolates queues and exchanges from each other (e.g., per environment or application).                                                                |
| **Event Bus** | The internal publish/subscribe mechanism that delivers real‑time notifications about system activity (queue state changes, message lifecycle, etc.). Based on Redis Pub/Sub. |
| **RedisSMQ**  | The overall system consisting of language‑specific libraries that share a common Redis schema and Lua scripts.                                                               |

## Queue Concepts

| Term                | Definition                                                                                                               |
|---------------------|--------------------------------------------------------------------------------------------------------------------------|
| **FIFO**            | First‑In, First‑Out. Messages are processed in the order they arrived.                                                   |
| **LIFO**            | Last‑In, First‑Out. The newest message is processed first.                                                               |
| **Priority Queue**  | Messages are ordered by a numeric priority level (0–7). Lower numbers have higher priority.                              |
| **Delivery Model**  | How a queue distributes messages: **Point‑to‑Point** (one consumer per message) or **Pub/Sub** (all groups per message). |
| **Point‑to‑Point**  | A delivery model where each message goes to exactly one consumer.                                                        |
| **Pub/Sub**         | A delivery model where each message is broadcast to all consumer groups.                                                 |
| **Consumer Group**  | A named set of consumers that collectively process a copy of every message in a Pub/Sub queue.                           |
| **Ephemeral Group** | A consumer group created automatically for a single consumer, deleted when the consumer shuts down.                      |
| **Queue State**     | The operational mode of a queue: **Active**, **Paused**, **Stopped**, or **Locked**.                                     |
| **Rate Limiting**   | Controlling the maximum number of messages delivered from a queue within a time window.                                  |
| **Worker Lock**     | A distributed lock that ensures only one consumer per queue runs background workers (scheduler, reaper, etc.).           |

## Message Lifecycle

| Term                        | Definition                                                                                                              |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **Message Status**          | The current lifecycle state of a message (e.g., `PENDING`, `PROCESSING`, `ACKNOWLEDGED`, `DEAD_LETTERED`).              |
| **Acknowledge**             | Confirming that a message was successfully processed. The message is removed from the active queue.                     |
| **Unacknowledge**           | Indicating that a message failed to process. It will be retried or dead‑lettered.                                       |
| **Requeue**                 | Returning a failed message to the pending queue for immediate retry.                                                    |
| **Retry Threshold**         | The maximum number of consumption attempts before a message is dead‑lettered.                                           |
| **Retry Delay**             | The time to wait between retry attempts.                                                                                |
| **Dead‑Letter Queue (DLQ)** | A storage area where messages go after exhausting retries, expiring, or being periodic messages that cannot be retried. |
| **TTL (Time‑to‑Live)**      | The maximum time a message can exist before it is considered expired and dead‑lettered.                                 |
| **Consumption Timeout**     | The maximum time a consumer has to process and acknowledge a message before it is considered failed.                    |
| **Scheduled Message**       | A message set for future delivery (via delay, CRON, or repeat).                                                         |
| **CRON**                    | A string expression defining a calendar‑based schedule for message delivery.                                            |
| **Delay**                   | A fixed offset from the publish time before a message is first delivered.                                               |
| **Repeat**                  | Delivering a message multiple times at a fixed interval.                                                                |
| **Periodic Message**        | A message with a CRON or repeat schedule.                                                                               |

## Reliability & Background Processes

| Term                       | Definition                                                                                                                        |
|----------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| **At‑Least‑Once Delivery** | The guarantee that a message will never be lost in transit. It may be delivered multiple times, so handlers should be idempotent. |
| **Heartbeat**              | A periodic signal from a consumer (stored in Redis with a TTL) proving it is still alive.                                         |
| **Reaper**                 | A background worker that detects dead consumers (by missing heartbeats) and recovers their in‑flight messages.                    |
| **Idempotency**            | The property that processing the same message multiple times produces the same side‑effects.                                      |
| **Batch Acknowledgment**   | Grouping multiple acknowledgments into a single Redis operation for higher throughput.                                            |

## Management & Audit

| Term                         | Definition                                                                                                                              |
|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| **Message Audit**            | Tracking processed messages for later inspection (acknowledged messages, dead‑lettered messages, unacknowledgment history).             |
| **Unacknowledgment History** | A per‑message list of failure records. Each record includes the cause, the action taken (requeue, delay, dead‑letter), and a timestamp. |
| **Purge Job**                | A background job that deletes all messages of a certain category (e.g., pending, scheduled, acknowledged) from a queue.                 |
| **Configuration**            | System‑wide settings (namespace, logger, audit) stored in Redis and shared across all connected instances.                              |

## Exchange Concepts

| Term                | Definition                                                                                   |
|---------------------|----------------------------------------------------------------------------------------------|
| **Direct Exchange** | Routes messages to queues by exact routing key match.                                        |
| **Topic Exchange**  | Routes messages to queues by pattern matching with wildcards (`*` and `#`).                  |
| **Fanout Exchange** | Broadcasts every message to all bound queues.                                                |
| **Routing Key**     | A string used by exchanges to determine which queues receive a message.                      |
| **Binding**         | A connection between a queue and an exchange, optionally with a routing key or pattern.      |
| **Exchange Policy** | A restriction that limits which queue types (FIFO/LIFO or Priority) can bind to an exchange. |

## Implementation Details

| Term               | Definition                                                                                                   |
|--------------------|--------------------------------------------------------------------------------------------------------------|
| **Lua Script**     | An atomic multi‑key operation stored and executed inside Redis. All implementations share identical scripts. |
| **Schema Version** | A numeric prefix embedded in every Redis key (e.g., `10`) that is incremented when the key layout changes.   |
| **Codec**          | A component that serialises/deserialises data between Go/TypeScript and Redis storage formats.               |
| **Event**          | A JSON message published via the Event Bus carrying information about a system occurrence.                   |
