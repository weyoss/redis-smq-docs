# Interoperability

RedisSMQ implementations in different languages share the same Redis data structures, Lua scripts, and serialisation formats. Messages published from one language can be consumed from another without any translation layer.

## How interoperability works

All implementations use a common set of shared components, which guarantees that operations performed in one language are visible and consistent in every other language.

### Shared Redis key schema

Every key in Redis follows the same hierarchical pattern:

```
redis-smq:{version}:{domain}:{...path components}
```

The `{version}` component (currently `10`) is incremented when the schema changes in a backward‑incompatible way. The key schema is documented in detail in [Redis Key Schema](redis-key-schema.md). All implementations use the identical key structure.

### Shared Lua scripts

All multi‑key operations are performed by Lua scripts that are loaded into Redis at startup. The Lua scripts are identical across implementations – the TypeScript repository is the source of truth, and the Go implementation syncs the same scripts via `make sync`.

The scripts are registered by `SCRIPT LOAD` and executed by SHA, which means they are byte‑identical regardless of which language invoked them. This guarantees atomic, consistent behaviour for operations like publishing, acknowledging, and state transitions.

### Shared serialisation formats

All structured data stored in Redis uses JSON with the same field names, types, and semantics:

- **Message payload** – A JSON object with fields like `id`, `status`, `message`, `priority`, scheduling properties, timestamps, and counters.
- **Queue properties** – A JSON hash with fields like `type`, `deliveryModel`, `rateLimit`, and counters.
- **Exchange properties** – A JSON hash with fields `type` and `policy`.
- **State transitions** – A JSON object with fields `from`, `to`, `reason`, `timestamp`, etc.
- **Events** – All event payloads published via the event bus use consistent JSON shapes.

Every field name is documented in the language‑agnostic specification. Implementations validate these shapes at build time through their type systems.

## Compatible implementations

| Language             | Package                                            | Status |
|----------------------|----------------------------------------------------|--------|
| TypeScript / Node.js | [redis-smq](https://github.com/weyoss/redis-smq)   | ✓ Main implementation |
| Go                   | [redis-smq-go](https://github.com/weyoss/go-redis-smq) | ✓ Fully compatible |

The TypeScript implementation is the reference implementation. The Go implementation passes the same test suite against the same Redis instance, ensuring full wire‑level compatibility.

## Version compatibility matrix

Always use library versions that target the same specification version. The specification is versioned independently of the libraries, and each version defines a compatible set of library releases.

| Specification version | TypeScript (`redis-smq`) | Go (`go-redis-smq`) | Notes |
|-----------------------|--------------------------|---------------------|-------|
| `/next/` (unstable)   | `next` branch / `@next` tag | `next` branch   | Development only |
| **v1.0** (stable)     | `≥11.0.0 <11.1.0`       | `≥1.0.0 <1.1.0`    | Current stable |

> ⚠️ Mixing library versions that target different specification versions can lead to data corruption or runtime errors. Always check the compatibility matrix in the project README.

## Cross‑Language scenarios

All operations work across language boundaries:

### Go Producer → TypeScript Consumer

A Go service publishes messages to a FIFO queue. Node.js workers consume them and process them.

### TypeScript Producer → Go Consumer

A Node.js API publishes tasks. Go workers (for CPU‑bound processing) consume and execute them.

### Mixed management

You can create queues and exchanges from Go, bind them from TypeScript, publish from either language, and inspect the state from both – all in real time, sharing the same Redis instance.

### Shared configuration

Configuration stored in Redis (`redis-smq:{v}:main:cfg`) is read and written by all implementations. Changing the default namespace from Go immediately affects new queues created from TypeScript.

### Event bus

All implementations publish and subscribe to the same event channels (`redis-smq:events:*`). A TypeScript monitor can observe producer lifecycle events from Go producers, and vice versa.

## Requirements for compatibility

- **Redis version** ≥ 4.0 (for `EVALSHA` and `ZPOPLPUSH` support).
- **Same RedisSMQ schema version** – the key version prefix must match.
- **Identical Lua scripts** – scripts must be in sync; use `make sync` in the Go implementation to pull the latest from the TypeScript source.
- **Consistent namespace configuration** – the default namespace should be aligned if you want queues to be visible without explicit namespace qualification.

## Limitations

- **Message bodies must be JSON‑serialisable.** All implementations serialise message payloads to JSON. Binary data must be base64‑encoded.
- **Consumer groups are shared.** A consumer group created by Go behaves identically to one created by TypeScript. The ephemeral group naming convention (`cid‑{consumerId}`) is the same.
- **Lua script changes** must be propagated to all implementations. The TypeScript repository is the source of truth; the Go implementation syncs scripts automatically.
- **No cross‑Redis communication.** All implementations must connect to the same Redis instance (or a Redis Cluster/ Sentinel group) – there is no cross‑instance routing.
