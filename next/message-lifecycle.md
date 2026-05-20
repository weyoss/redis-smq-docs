# Message Lifecycle

A message in RedisSMQ goes through several states from creation to final resolution.

## Status Enum

| State             | Description                                                                                     |
|-------------------|-------------------------------------------------------------------------------------------------|
| `NEW`             | Message has been created but not yet published to the queue.                                    |
| `PENDING`         | Message is waiting to be consumed.                                                              |
| `PROCESSING`      | Message is currently being processed by a consumer.                                             |
| `SCHEDULED`       | Message is scheduled to be delivered at a later time (delay, CRON, or repeat).                  |
| `ACKNOWLEDGED`    | Message has been successfully consumed and acknowledged.                                        |
| `UNACK_REQUEUING` | Message has been unacknowledged and is waiting in the requeue list to be moved back to pending. |
| `UNACK_DELAYING`  | Message has been unacknowledged and is waiting in the delayed queue for a scheduled retry.      |
| `DEAD_LETTERED`   | Message has failed processing and has been moved to the dead‑letter queue.                      |

> ℹ️ `UNACK_REQUEUING` and `UNACK_DELAYING` are **internal intermediate states** that exist while the message is being moved between queues by background workers.

## State Diagram

```mermaid
stateDiagram-v2
    [*] --> NEW : Message created
    NEW --> PENDING : Published immediately
    NEW --> SCHEDULED : Scheduled for later

    SCHEDULED --> PENDING : Due time reached (scheduler)

    PENDING --> PROCESSING : Consumer dequeues

    PROCESSING --> ACKNOWLEDGED : Handler returns nil
    PROCESSING --> UNACK_REQUEUING : Failure, retry immediate
    PROCESSING --> UNACK_DELAYING : Failure, retry with delay
    PROCESSING --> DEAD_LETTERED : Retries exhausted / TTL expired

    UNACK_REQUEUING --> PENDING : Requeued immediately by worker
    UNACK_DELAYING --> PENDING : Delay elapsed, moved by worker

    ACKNOWLEDGED --> [*]
    DEAD_LETTERED --> [*]
```

## Complete Lifecycle Walkthrough

The following sequence diagram illustrates every stage of a message's journey, from production to final resolution. Reliability mechanisms are active at each step.

```mermaid
sequenceDiagram
    participant P as Producer
    participant Redis as Redis
    participant Scheduler as Scheduler (worker)
    participant C1 as Consumer 1
    participant C2 as Consumer 2
    participant DLQ as Dead‑Letter Queue

    P->>Redis: publish‑message.lua (store & add to pending/scheduled)
    alt Scheduled
        Redis-->>Redis: message in SCHEDULED
        Scheduler->>Redis: publish‑scheduled.lua
        Redis-->>Redis: move to PENDING
    end

    C1->>Redis: dequeue (RPopLPush / ZPOPLPUSH)
    Redis-->>C1: message moved to processing queue
    C1->>Redis: checkout‑message.lua (claim & update status)
    Redis-->>C1: message envelope

    C1->>C1: invoke handler
    alt success
        C1->>Redis: acknowledge‑message.lua
        Redis-->>Redis: status → ACKNOWLEDGED (audit if enabled)
        C1->>C1: consumer.messageAcknowledged event
    else failure
        C1->>Redis: unacknowledge‑message.lua
        alt retry immediate
            Redis-->>Redis: status → UNACK_REQUEUING, move to requeue list
            Note right of Redis: requeue‑immediate.lua (background worker) moves to PENDING
        else retry with delay
            Redis-->>Redis: status → UNACK_DELAYING, add to delayed set
            Note right of Redis: requeue‑delayed.lua (background worker) moves to PENDING when delay expires
        else dead‑letter
            Redis-->>Redis: status → DEAD_LETTERED, move to DLQ
        end
    end

    opt Consumer crash
        C1--xC1: crash
        Note right of C2: Reaper detects dead consumer, recovers messages
        C2->>Redis: unacknowledge‑message.lua (recover processing queue)
        Redis-->>Redis: messages returned to PENDING
    end

    Note over P,DLQ: At‑least‑once delivery with idempotency guidance
```

## Key Concepts

### Expiry (TTL)
If a message has a time‑to‑live, it will be dead‑lettered (or discarded) when the TTL expires, regardless of its current state.

### Retry Policy
- **Retry Threshold** – maximum number of consumption attempts before the message is dead‑lettered.
- **Retry Delay** – time to wait between retry attempts.

### Consumption Timeout
If a consumer does not acknowledge a message within the configured timeout, the message is assumed failed and returned to the queue for another consumer.

### Periodic Messages
Messages with CRON expressions or repeat counts are never retried on failure—they are immediately dead‑lettered. The next scheduled instance will be delivered as usual.

### Crash Recovery
If a consumer crashes without acknowledging, the **Reaper** (a background worker running on another consumer) detects the dead consumer, recovers the in‑flight message via the same unacknowledge script, and makes it available for another consumer.

## Related Pages

- [Messages](messages.md) – message properties and configuration
- [Scheduling Messages](scheduling-messages.md) – delays, CRON, and repeating delivery
- [Message Reliability](message-reliability.md) – delivery guarantees and recovery
- [Message Audit](message-audit.md) – tracking processed messages
