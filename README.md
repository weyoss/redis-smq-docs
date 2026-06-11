<div align="center" style="text-align: center">
  <p>
    <a href="https://github.com/weyoss/redis-smq">
      <img src="logo.png" alt="RedisSMQ" width="500px" />
    </a>
  </p>
  <p><strong>The definitive guide to RedisSMQ</strong><br />language‑agnostic concepts, architecture, and operational patterns.</p>
</div>

---

## 🌐 Language Implementations

| Language             | Repository                                             | Status                   |
|----------------------|--------------------------------------------------------|--------------------------|
| TypeScript / Node.js | [redis-smq](https://github.com/weyoss/redis-smq)       | Reference implementation |
| Go                   | [go-redis-smq](https://github.com/weyoss/go-redis-smq) | Fully compatible         |

> All implementations share the same wire protocol and data schema.

---

## ✨ Key Features

| Feature                       | Description                                                                            |
|-------------------------------|----------------------------------------------------------------------------------------|
| 📬 **Reliable delivery**      | At‑least‑once delivery with retries, dead‑letter queues, and automatic crash recovery. |
| 🚀 **High throughput**        | Atomic Lua scripts and efficient Redis data structures.                                |
| 📊 **Multiple queue types**   | FIFO, LIFO, and Priority.                                                              |
| 🔀 **Flexible routing**       | Direct, Topic, Fanout exchanges plus direct queue publishing.                          |
| 👥 **Delivery models**        | Point‑to‑Point and Pub/Sub with consumer groups.                                       |
| ⏰ **Scheduling**              | Delayed delivery, CRON schedules, and repeating messages.                              |
| 🚦 **Rate limiting**          | Per‑queue throughput control.                                                          |
| ⏱️ **TTL & timeouts**         | Message expiry and consumption timeouts.                                               |
| 📦 **Batch acknowledgements** | Reduced Redis calls for high‑throughput consumers.                                     |
| 🔒 **Queue state management** | Pause, stop, resume, and lock queues with audit.                                       |
| 📡 **Event bus**              | Real‑time system events via Redis Pub/Sub.                                             |
| 🌐 **Cross‑language**         | Go and TypeScript implementations share one protocol.                                  |
| 🔧 **REST API & Web UI**      | Provided by the TypeScript implementation; compatible with all clients.                |

---

## 🎯 Use Cases

- Background job processing (emails, reports, data pipelines)
- Task scheduling with automatic retries
- Microservices communication
- Real‑time event handling (gaming, IoT, analytics)

---

## 📋 Requirements

- **Redis** ≥ 4.0

---

## 📂 Documentation Structure

The documentation is **versioned** to stay in sync with the evolving specification.  
A **specification version** (e.g., `v1.0`) groups a compatible set of language‑specific releases.

> ℹ️ The specification version is independent of any library’s semantic version.  
> Each versioned folder is an immutable snapshot of the concepts at that point in time.

### Folder Layout

| Folder                     | Content                                   | Stability   |
|----------------------------|-------------------------------------------|-------------|
| [`/next/`](next/README.md) | Bleeding‑edge spec (development branches) | ⚠️ Volatile |
| `/v1.0/` (TBD)             | Latest stable specification               | ✅ Immutable |

### Compatibility Matrix

| Docs Folder                | `redis-smq` (TypeScript) | `go-redis-smq` (Go) | Status                |
|----------------------------|--------------------------|---------------------|-----------------------|
| [`/next/`](next/README.md) | `next` branch / `@next`  | `next` branch       | 🚧 Unreleased         |
| **`/v1.0/`** (TBD)         | `≥11.0.0 <11.1.0`        | `≥1.0.0 <1.1.0`     | ✅ **Latest Stable**   |

> ⚠️ Always check this table before reading the docs to ensure you’re viewing the version that matches your runtime.

---

## 🚀 Getting Started

- If you are using a development branch, begin with the [`/next/` specification](next/README.md) and its [Getting Started](next/getting-started.md) guide.
- For stable releases, wait for the versioned folder (e.g., `/v1.0/`) to be published.

For the full document index of the current development specification, see [`next/README.md`](next/README.md).

---

## 🤝 Contributing

Documentation improvements are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## 📄 License

MIT – see [LICENSE](LICENSE).
