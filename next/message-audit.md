# Message Audit

Message audit tracks processed messages for monitoring, debugging, and compliance. By default, audit is **disabled** to minimise overhead.

When enabled, the system keeps separate records for successfully processed messages, permanently failed messages, and a per‑message history of unacknowledgment events.

## What Audit Tracks

### 1. Acknowledged Messages

Every message that a consumer successfully acknowledges is appended to a queue‑specific acknowledged list (`ns:{ns}:q:{name}:ack`). This allows you to:

- Verify that a message was processed.
- Replay or reprocess historical messages if needed.
- Monitor processing rates and completion.

### 2. Dead‑Lettered Messages

Messages that could not be processed after all retries (or that expired, or are periodic messages) are appended to the dead‑letter list (`ns:{ns}:q:{name}:dl`). You can:

- Inspect why a message failed (check the unacknowledgment history).
- Manually requeue the message after fixing the underlying issue.
- Detect systemic failures by monitoring the dead‑letter list.

### 3. Unacknowledgment History

Each message can carry a history of its failure events. Every time the message is unacknowledged, a record is pushed to a list attached to the message (`main:msg:{id}:uh`). The record includes:

- The unacknowledgment cause (e.g., timeout, handler error, TTL expiry).
- The action taken (requeue, delay, dead‑letter).
- The attempt number and timestamp.

This history helps you diagnose flaky handlers or transient errors without enabling global acknowledged/dead‑letter audit.

## Configuration

Audit is configured globally and applies to all queues. The following settings are available:

```json
{
  "messageAudit": {
    "acknowledgedMessages": {
      "enabled": true,
      "queueSize": 10000,
      "expire": 86400
    },
    "deadLetteredMessages": {
      "enabled": true,
      "queueSize": 5000,
      "expire": 604800
    },
    "unacknowledgementHistory": {
      "enabled": true,
      "maxSize": 100
    }
  }
}
```

| Setting                            | Purpose                                                                    | Default |
|------------------------------------|----------------------------------------------------------------------------|---------|
| `acknowledgedMessages.enabled`     | Enable acknowledged message list                                           | `false` |
| `acknowledgedMessages.queueSize`   | Maximum entries in the list per queue (0 = unlimited)                      | `0`     |
| `acknowledgedMessages.expire`      | Retention time in seconds before the whole list key is deleted (0 = never) | `0`     |
| `deadLetteredMessages.enabled`     | Enable dead‑lettered message list                                          | `false` |
| `deadLetteredMessages.queueSize`   | Maximum entries in the list per queue (0 = unlimited)                      | `0`     |
| `deadLetteredMessages.expire`      | Retention time in seconds for the list key (0 = never)                     | `0`     |
| `unacknowledgementHistory.enabled` | Enable per‑message failure history                                         | `false` |
| `unacknowledgementHistory.maxSize` | Maximum number of history records per message (0 = unlimited)              | `100`   |

### Storage Limits

- **queueSize** – The list is trimmed to the most recent `queueSize` entries. Older entries are discarded. `0` means no trimming.
- **expire** – The entire list key (e.g., the acknowledged list for a queue) is given a Redis expire time. After the specified seconds, the key and its contents are automatically removed.
- **maxSize** – The unacknowledgment history list for each message is trimmed to the most recent `maxSize` records. Older records are removed. `0` disables trimming.

When limits are reached, the oldest entries are evicted first.

## Where Audit Data Lives

| Audit Category            | Redis Key                     | Type | Trimmed / Expired?        |
|---------------------------|-------------------------------|------|---------------------------|
| Acknowledged messages     | `ns:{namespace}:q:{name}:ack` | List | By `queueSize` / `expire` |
| Dead‑lettered messages    | `ns:{namespace}:q:{name}:dl`  | List | By `queueSize` / `expire` |
| Unacknowledgment history  | `main:msg:{messageId}:uh`     | List | By `maxSize`              |

The acknowledged and dead‑lettered lists only contain message IDs; the full message data is stored separately in the message hash (`main:msg:{id}`) and is never automatically deleted.

## Using Audit Data

Once enabled, you can:

- **Browse** acknowledged and dead‑lettered messages using the message browsing API. If audit is disabled, browsing these categories returns an error.
- **Purge** acknowledged or dead‑lettered messages using the purge API (requires audit enabled).
- **Retrieve unacknowledgment history** for any message to see its failure timeline, even if the message itself is still in‑flight.

## Performance Impact

- **Disabled (default):** No overhead.
- **Enabled with limits:** Minimal Redis memory and CPU overhead.
- **Unlimited storage (`queueSize = 0`, `expire = 0`, `maxSize = 0`):** Can consume significant memory over time, especially for high‑throughput queues.

Enable only what you need. Set appropriate limits for your retention requirements.

## Use Cases

- **Debugging** – Inspect why a message failed and what action was taken.
- **Compliance** – Maintain processing records for auditing purposes.
- **Monitoring** – Track success / failure rates per queue.
- **Recovery** – Requeue dead‑lettered messages after fixing the underlying issue.
