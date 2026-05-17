# Queue State Management

Queues have an operational state that controls whether they accept new messages and deliver existing ones. State transitions are atomic, validated by Lua scripts, and recorded in a state history log.

## States

| State       | Accepts Messages  | Delivers Messages | Description                         |
|-------------|-------------------|-------------------|-------------------------------------|
| **Active**  | Yes               | Yes               | Normal operation                    |
| **Paused**  | Yes               | No                | Buffers messages without processing |
| **Stopped** | No                | No                | Fully halted                        |
| **Locked**  | No                | No                | Exclusive maintenance access        |

## State Transitions

```
Active  → Paused, Stopped, Locked
Paused  → Active, Stopped, Locked
Stopped → Active
Locked  → Active, Stopped
```

Transitions are validated — not all state changes are allowed. For example, a stopped queue cannot be paused directly; it must be resumed first.

## When to Use Each State

### Active

The normal operating state. Queues are created in this state.

### Paused

Temporarily stop message processing while allowing new messages to accumulate. Useful for:

- **Rolling deployments** — Pause processing during updates
- **Downstream maintenance** — Hold messages while a service is being restarted
- **Debugging** — Inspect queue state without messages being consumed

When resumed, buffered messages are processed immediately.

### Stopped

Completely halt the queue. No messages are accepted or processed. Useful for:

- **Major maintenance** — Extended downtime
- **Emergency intervention** — Stop all activity immediately
- **Queue deprecation** — Prevent new messages while draining existing ones

When resumed, the queue returns to Active state and begins processing normally.

### Locked

Exclusive access for administrative operations. Only the lock holder can operate on the queue. Used internally for operations like queue purging. External operations on a locked queue are rejected.

## State History

Every state transition is recorded as a JSON object in a Redis list. The object has the following structure:

```json
{
  "from": 1,
  "to": 0,
  "reason": "MANUAL",
  "timestamp": 1700000000000,
  "description": "Scheduled database maintenance",
  "lockId": "job-abc123",
  "lockOwner": 0,
  "metadata": {}
}
```

| Field         | Type    | Description                                                                                |
|---------------|---------|--------------------------------------------------------------------------------------------|
| `from`        | integer | Previous state, or `null` for the initial transition. Matches `QueueState` enum values.    |
| `to`          | integer | New state. Matches `QueueState` enum values.                                               |
| `reason`      | string  | Why the transition occurred. Must be one of the **[valid reason values](#reason-values)**. |
| `timestamp`   | integer | Unix timestamp in milliseconds when the transition took place.                             |
| `description` | string  | Optional human‑readable description of the transition.                                     |
| `lockId`      | string  | Lock identifier (present only for LOCKED ↔ ACTIVE transitions).                            |
| `lockOwner`   | integer | Lock owner enum (present only for LOCKED ↔ ACTIVE transitions). `0` = Purge Job.           |
| `metadata`    | object  | Optional key‑value pairs with additional context (e.g., `queueType`, `deliveryModel`).     |

State history is stored as a list, ordered from most recent to oldest. It is never automatically deleted.

### Reason Values

The `reason` field is a string whose value must come from one of the two following lists. The same values are used in all language implementations, ensuring cross‑language compatibility.

**User‑facing reasons** (can be specified via the public API):

| Value           | Description                                |
|-----------------|--------------------------------------------|
| `MANUAL`        | A human manually changed the state         |
| `SCHEDULED`     | An automated schedule triggered the change |
| `EMERGENCY`     | An emergency stop                          |
| `PERFORMANCE`   | Performance‑related adjustment             |
| `ERROR`         | An error occurred                          |
| `CONFIG_CHANGE` | Configuration was modified                 |
| `TESTING`       | For testing purposes                       |
| `OTHER`         | An uncategorised reason                    |

**System‑only reasons** (generated automatically, not exposed in the public API):

| Value                  | Description                              |
|------------------------|------------------------------------------|
| `SYSTEM_INIT`          | Initial state when a queue is created    |
| `RECOVERY`             | State recovered after a failure          |
| `PURGE_QUEUE_START`    | A purge operation started                |
| `PURGE_QUEUE_CANCEL`   | A purge operation was cancelled          |
| `PURGE_QUEUE_FAIL`     | A purge operation failed                 |
| `PURGE_QUEUE_COMPLETE` | A purge operation completed successfully |

## Monitoring State Changes

State changes are published as events via the event bus. Other system components and external monitors can subscribe to react to state transitions — for example, stopping a consumer when its queue is paused.

## Best Practices

- Provide meaningful reasons and descriptions for state changes
- Monitor state change events for operational awareness
- Don't leave queues locked longer than necessary
- Use pause for short-term maintenance, stop for extended downtime
