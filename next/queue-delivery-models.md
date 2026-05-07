# Queue Delivery Models

RedisSMQ supports two delivery models: **Point-to-Point** and **Pub/Sub**. The delivery model is set when a queue is created and cannot be changed afterwards.

## Point-to-Point

```
Producer → Queue → Consumer A (one consumer processes each message)
```

Each message is delivered to exactly one consumer. If multiple consumers are subscribed to the same queue, messages are distributed among them — each message goes to one consumer.

**Best for:**

- Task queues and job processing
- Workload distribution
- Ordered processing where each message must be handled once

**Behavior:**

- Messages are delivered in queue order (FIFO, LIFO, or Priority)
- Multiple consumers provide load balancing — messages are distributed across them
- If a consumer fails, its in-flight messages are recovered and delivered to another consumer
- Consumer groups are not used

**Example scenario:** An order processing system where each order must be processed exactly once. Multiple worker instances consume from the same queue, each picking up the next available order.

## Pub/Sub

```
Producer → Queue → Group "email" → Consumer A (one per group)
                 → Group "sms"   → Consumer B (one per group)
                 → Group "push"  → Consumer C (one per group)
```

Each message is delivered to **all consumer groups**. Within a group, the message is delivered to exactly one consumer.

**Best for:**

- Event broadcasting to multiple services
- Fan-out patterns where multiple systems need the same message
- Multi-channel notifications

**Behavior:**

- Each consumer group receives a copy of every message
- Within a group, messages are load-balanced across consumers
- Groups are independent — failures in one group don't affect others
- Groups must be created before messages are published

**Example scenario:** A notification system where a single event triggers email, SMS, and push notifications. Each notification channel is a consumer group. Within the email group, multiple workers share the load.

## Comparison

|                      | Point-to-Point                 | Pub/Sub                                   |
|----------------------|--------------------------------|-------------------------------------------|
| **Message delivery** | One consumer total             | All groups, one consumer per group        |
| **Load balancing**   | Across all consumers           | Within each group                         |
| **Consumer groups**  | Not used                       | Required                                  |
| **Use case**         | Work queues, task distribution | Event broadcasting, multi-service fan-out |
| **Message copies**   | One                            | One per consumer group                    |
| **Overhead**         | Lower                          | Higher (message duplicated per group)     |

## Consumer Groups in Pub/Sub

A consumer group is a named set of consumers that together process messages. Each group:

- Has a unique name within the queue (e.g., "email-service", "audit-logger")
- Receives every message published to the queue
- Delivers each message to exactly one consumer within the group
- Operates independently — groups don't know about each other

### Creating Groups

Groups can be created explicitly before consumption or automatically when a consumer subscribes with a group ID. A queue must be Pub/Sub for groups to be used — Point-to-Point queues reject group operations.

### Ephemeral Groups

If a consumer subscribes to a Pub/Sub queue without specifying a group, an ephemeral group is created. The group is named after the consumer ID and is automatically deleted when the consumer shuts down. Ephemeral groups are useful for temporary or single-instance consumers.

### Group Lifecycle

- **Created** — Explicitly or on first subscription
- **Active** — Consumers are subscribed and processing
- **Empty** — No active consumers, but group still exists
- **Deleted** — Explicitly removed; must be empty with no pending messages

## Choosing a Delivery Model

**Use Point-to-Point when:**

- Each message represents a unit of work to be done once
- You need load balancing across worker instances
- Message ordering matters within a single consumer

**Use Pub/Sub when:**

- Multiple independent services need the same message
- You want to add consumers without changing producers
- Each consumer represents a different function or channel

## Delivery Model and Exchanges

The delivery model is a property of the **queue**, not the exchange. An exchange can route messages to queues with different delivery models:

```
Exchange "events"
├── Queue "worker-tasks" (Point-to-Point)
└── Queue "notifications" (Pub/Sub)
```

A single message published to the exchange goes to both queues. The worker-tasks queue delivers it to one worker. The notifications queue broadcasts it to all consumer groups.
