<div align="center" style="text-align: center">
  <p>
    <a href="https://github.com/weyoss/redis-smq">
      <img src="logo.png" alt="RedisSMQ" width="500px" />
    </a>
  </p>
  <p><strong>The definitive guide to RedisSMQ</strong><br />language‑agnostic concepts, architecture, and operational patterns.</p>
</div>

---

## 🌐 Language implementations

| Language             | Repository                                             |
|----------------------|--------------------------------------------------------|
| TypeScript / Node.js | [redis-smq](https://github.com/weyoss/redis-smq)       |
| Go                   | [go-redis-smq](https://github.com/weyoss/go-redis-smq) |

> All implementations share the same wire protocol and data schema.

## ✨ Features

- 📬 Reliable delivery with retries and dead‑letter queues
- 🚀 High-throughput design with atomic Lua scripts
- 📊 Multiple queue types: FIFO, LIFO, Priority
- 🔀 Flexible routing: Direct, Topic, Fanout exchanges + direct queue publishing
- 👥 Pub/Sub and Point‑to‑Point delivery models, consumer groups
- ⏰ Scheduling: delays, CRON, repeating messages
- 🚦 Queue‑level rate limiting
- ⏱️ Message TTL and consumption timeouts
- 📦 Batch acknowledgements (99% fewer Redis calls)
- 🔒 Queue state management (pause/stop/resume) with audit
- 📡 Internal event bus  for real-time internal events
- 🌐 REST API and Web UI (provided by the TypeScript implementation, compatible with all clients)

## 🎯 Use Cases

- Background job processing (emails, reports, data pipelines)
- Task scheduling with automatic retries
- Microservices communication
- Real‑time event handling (gaming, IoT, analytics)

## 📋 Requirements

- **Redis** ≥ 4

## 📂 Documentation Structure

The documentation is **versioned** to stay in sync with the evolving specification.  
A **specification version** (e.g., `v1.0`) groups a compatible set of language‑specific releases.

> ℹ️ The specification version is independent of any library’s semantic version.  
> Each folder is an immutable snapshot of the concepts at that point in time.

### Folder Layout

| Folder                     | Content                                   | Stability   |
|----------------------------|-------------------------------------------|-------------|
| [`/next/`](next/README.md) | Bleeding‑edge spec (development branches) | ⚠️ Volatile |
| `/v1.0/` (TBD)             | Latest stable specification               | ✅ Immutable |

### Compatibility Matrix

| Docs Folder                | `redis-smq` (TypeScript) | `go-redis-smq` (Go) | Status               |
|----------------------------|--------------------------|---------------------|----------------------|
| [`/next/`](next/README.md) | `next` branch / `@next`  | `next` branch       | 🚧 Unreleased        |
| **`/v1.0/`** (TBD)         | `≥11.0.0 <11.1.0`        | `≥1.0.0 <1.1.0`     | ✅ **Latest Stable**  |

> ⚠️ Always check this table before reading the docs to ensure you’re viewing the version that matches your runtime.

## 📖 How to Read

1. Start with the **Features** and **Use Cases** above.  
2. Pick the specification folder that matches your library version (see matrix).  
3. Inside that folder you’ll find language‑agnostic descriptions of:
   - Queues, delivery models, exchanges
   - Message lifecycle, scheduling, retries
   - Rate limiting, namespaces, and more
4. For API details, refer to the language‑specific library documentation.

## 🤝 Contributing

Documentation improvements are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## 📄 License

MIT – see [LICENSE](LICENSE).
