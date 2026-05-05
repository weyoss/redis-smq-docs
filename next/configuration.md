# Configuration

RedisSMQ configuration controls system‑wide settings. Configuration is stored in Redis and shared across all connected instances.

## What Configuration Controls

- **Namespace** — Default namespace for queue and exchange operations
- **Logger** — Console logging settings
- **Message Audit** — Tracking of processed messages

## Where Configuration Lives

Configuration is stored as a Redis hash, with each field representing a setting. A version counter ensures safe 
concurrent updates. All instances share the same configuration, and changes are propagated automatically via the event 
bus, eliminating the need for restarts or manual reloads.

## Configuration Object

The configuration object is a JSON document with the following structure:

### `namespace` (string)

Default namespace used when no explicit namespace is provided during queue operations. Default: `"default"`.

### `logger`

Controls console output.

| Field                      | Type    | Description                       | Default |
|----------------------------|---------|-----------------------------------|---------|
| `enabled`                  | bool    | Enable or disable console logging | `false` |
| `options`                  | object  | Logger options                    |         |
| `options.includeTimestamp` | bool    | Include timestamps in log output  | `true`  |
| `options.colorize`         | bool    | Enable ANSI color coding          | `true`  |
| `options.logLevel`         | integer | 0=DEBUG, 1=INFO, 2=WARN, 3=ERROR  | `1`     |

### `messageAudit`

Controls message audit trails.

| Field                              | Type   | Description                                                   | Default |
|------------------------------------|--------|---------------------------------------------------------------|---------|
| `acknowledgedMessages`             | object | Audit settings for successfully processed messages            |         |
| `acknowledgedMessages.enabled`     | bool   | Enable audit for acknowledged messages                        | `false` |
| `acknowledgedMessages.queueSize`   | int    | Maximum number of messages to keep per queue (0 = unlimited)  | `0`     |
| `acknowledgedMessages.expire`      | int    | Retention time in seconds (0 = never)                         | `0`     |
| `deadLetteredMessages`             | object | Audit settings for dead‑lettered messages                     |         |
| `deadLetteredMessages.enabled`     | bool   | Enable audit for dead‑lettered messages                       | `false` |
| `deadLetteredMessages.queueSize`   | int    | Maximum number of messages to keep per queue (0 = unlimited)  | `0`     |
| `deadLetteredMessages.expire`      | int    | Retention time in seconds (0 = never)                         | `0`     |
| `unacknowledgementHistory`         | object | Audit settings for unacknowledgment history                   |         |
| `unacknowledgementHistory.enabled` | bool   | Enable unacknowledgment history per message                   | `false` |
| `unacknowledgementHistory.maxSize` | int    | Maximum number of history records per message (0 = unlimited) | `100`   |

## Full Example

```json
{
  "namespace": "production",
  "logger": {
    "enabled": true,
    "options": {
      "includeTimestamp": true,
      "colorize": true,
      "logLevel": 1
    }
  },
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
      "maxSize": 50
    }
  }
}
```

## Default Configuration

If no configuration exists when the system initializes, defaults are saved to Redis:

- `namespace`: `"default"`
- `logger.enabled`: `false`
- All message audit settings: `false` / `0` (disabled/unlimited)

## Updating Configuration

Configuration can be read and updated at runtime. Changes are saved to Redis and published as an event. All connected instances receive the update automatically.

Version checking prevents conflicting updates from multiple instances — if two instances try to save simultaneously, one will receive a version mismatch error and should retry.

## Best Practices

- Set the namespace early — before creating queues
- Disable logging and audit in production unless needed
- Use storage limits for audit to control Redis memory
- Let configuration sync automatically — don’t manually reload
