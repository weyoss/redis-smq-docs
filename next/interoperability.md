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

Because all keys are derived from the same schema, a queue created by a Go process is immediately visible to a TypeScript process, and vice versa — there is no per-language isolation layer.

### Shared Lua scripts

All multi‑key operations are performed by Lua scripts that are loaded into Redis at startup. The scripts are identical across implementations.

The TypeScript repository is the source of truth for the scripts:

```
packages/redis-smq/src/common/scripts                 → core scripts
packages/redis-smq/src/common/background-jobs/scripts → job scripts
```

The Go implementation syncs the same scripts via `make sync` (which invokes `sync_redis_scripts.sh` — it copies the two source directories above into `internal/redis/scripts/scripts/core` and `internal/redis/scripts/scripts/jobs`, overwriting them wholesale).

The scripts are registered by `SCRIPT LOAD` and executed by SHA, which means they are byte‑identical regardless of which language invoked them. This guarantees atomic, consistent behaviour for operations like publishing, acknowledging, and state transitions.

> If you fork or patch the Lua scripts, all implementations must run the same bytes. A version skew in the scripts is the single most likely source of cross-language bugs.

### Shared serialisation formats

All structured data stored in Redis uses JSON with the same field names, types, and semantics:

- **Message payload** – A JSON object with fields like `id`, `status`, `message`, `priority`, scheduling properties, timestamps, and counters. See [Redis Key Schema → Message Storage](redis-key-schema.md#message-storage) for the field‑by‑field table.
- **Queue properties** – A JSON hash with fields like `type`, `deliveryModel`, `rateLimit`, and counters.
- **Exchange properties** – A JSON hash with fields `type` and `policy`.
- **State transitions** – A JSON object with fields `from`, `to`, `reason`, `timestamp`, etc.
- **Events** – All event payloads published via the event bus use consistent JSON shapes. See [Event Bus](event-bus.md) for the full reference.

Every field name is documented in the language‑agnostic specification. Implementations validate these shapes at build time through their type systems.

### Shared coordination primitives

Correct cross-language operation depends on every implementation using the **same coordination primitives** for background work:

- **Per-queue WorkersLock** (`ns:{ns}:q:{name}:wlock`) — a Redis key with TTL, acquired via `SETNX`, refreshed via `extend-lock.lua`, released via `release-lock.lua`. Every implementation must elect per-queue workers through this same lock. If two implementations used different lock keys for the same queue, both would elect and both would run the scheduler/requeuer/reaper concurrently — producing duplicate scheduled publishes, duplicate retries, and duplicate recovery events.

  The lock primitive is **shared protocol**, not implementation detail. The Lua scripts enforce it byte‑for‑byte; both implementations must honor it.

- **Per-consumer and per-worker heartbeats** (`main:cons:{id}:hb`, `main:wrk:{id}:hb`) — Redis keys with TTL, refreshed periodically. The reaper detects liveness by key existence. Any implementation that stops refreshing these keys within the TTL window will be recovered by any other implementation's reaper, regardless of which language started the consumer.

## Compatible implementations

| Language             | Package                                              | Status |
|----------------------|------------------------------------------------------|--------|
| TypeScript / Node.js | [redis-smq](https://github.com/weyoss/redis-smq)      | ✓ Reference implementation |
| Go                   | [go-redis-smq](https://github.com/weyoss/go-redis-smq) | ✓ Fully compatible |

The TypeScript implementation is the reference implementation. The Go implementation passes the same test suite against the same Redis instance, ensuring full wire‑level compatibility.

## Version compatibility matrix

Always use library versions that target the same specification version. The specification is versioned independently of the libraries, and each version defines a compatible set of library releases.

| Specification version | TypeScript (`redis-smq`) | Go (`go-redis-smq`) | Notes |
|-----------------------|--------------------------|---------------------|-------|
| `/next/` (unstable)   | `next` branch / `@next` tag | `next` branch    | Development only |
| **v1.0** (stable)     | `≥11.0.0 <11.1.0`       | `≥1.0.0 <1.1.0`     | Current stable |

> ⚠️ Mixing library versions that target different specification versions can lead to data corruption or runtime errors. Always check the compatibility matrix in the project README and in the [redis-smq-docs repository](https://github.com/weyoss/redis-smq-docs#compatibility-matrix).

The key-schema version prefix (`redis-smq:{version}:`) is the strongest compatibility signal. Two implementations with the same key version prefix are wire-compatible; different prefixes cannot coexist in the same Redis database without key collisions.

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

**Hybrid `configuration.updated` wire format.** Go publishers emit a 3‑arg event `(config, version, epoch)`. TypeScript publishers emit a 2‑arg event `(config, version)` — TS has no epoch concept. Subscribers must accept both forms:

- Go subscribers dispatch on epoch presence — an empty epoch means "no epoch to compare, fall back to version-only staleness detection."
- TypeScript subscribers dispatch on argument count — JS handler dispatch ignores extra args, so a TS subscriber sees a well-formed 2‑arg call even when a Go publisher emitted 3.

See [Event Bus → `configuration.updated`](event-bus.md#configurationupdated--internal-only) for the full contract.

### Event bus

Each implementation maintains **two** independent Redis Pub/Sub buses, both using the shared channel namespace:

| Bus | Channel prefix | Activation | Purpose |
|-----|----------------|------------|---------|
| **System bus** | `redis-smq:events:system:*` | Always active (required) | Cross‑instance synchronization — queue discovery, producer cache invalidation, consumer state tracking, Coordinator queue subscription |
| **User bus** | `redis-smq:events:user:*` | Optional (started on demand) | External monitoring and integration |

Both buses are **cross-language compatible**: a Go subscriber on the system bus receives events from TypeScript publishers and vice versa, using the identical channel names and payload formats. The user bus is likewise shared — a TypeScript monitor can observe producer lifecycle events from Go producers, and vice versa.

**Why the system bus must be active:** The Coordinator subscribes to `queue.queueCreated` on the system bus to discover newly created queues. If a TypeScript process creates a queue and the Go process's system bus were disabled, the Go Coordinator would only discover it on the next 30‑second `AllQueues()` sweep — new‑queue latency would regress from ~1ms to 30s. The system bus is not a performance optimization; it is a correctness requirement for interactive queue creation.

**Why the user bus is optional:** External monitoring is opt‑in by design. An application that doesn't need observability shouldn't pay for a second Pub/Sub connection. The user bus can be started (`InitUserEventBus` in Go) and stopped independently of the runtime.

### Background workers

The Coordinator's five per‑queue background workers (scheduled publisher, immediate requeuer, delayed requeuer, reaper, orphaned lock recoverer) are elected cluster‑wide via the shared `WorkersLock` key. This means:

- A queue's workers run on **one process cluster‑wide**, regardless of whether the processes are Go, TypeScript, or a mix.
- If a Go process holds a queue's lease, TypeScript processes on the same Redis will not run workers for that queue (they observe the lease as held).
- If the Go process holding a lease crashes, a TypeScript process will acquire the lease on its next reconcile tick.

The per-queue worker patterns are language‑agnostic; both implementations share the same leasing protocol.

## Requirements for compatibility

- **Redis version** ≥ 4.0 (for `EVALSHA` and the Lua features the shared scripts rely on — `SET` with `NX`/`PX`, `ZRANGEBYSCORE`, `BRPOPLPUSH`, etc.). See [BUILD.md](../BUILD.md) for exact minimums.
- **Same RedisSMQ key schema version** — the `{version}` prefix embedded in every Redis key must match. Two implementations with different key prefixes cannot coexist on the same Redis database.
- **Identical Lua scripts** — scripts must be in sync. Use `make sync` in the Go implementation to pull the latest from the TypeScript source (the source path is documented in [Shared Lua scripts](#shared-lua-scripts) above).
- **Identical coordination primitives** — both implementations must use the same `WorkersLock` key, the same `extend-lock.lua`/`release-lock.lua` scripts, and the same heartbeat TTLs. Divergence in any of these means election doesn't work across the mixed cluster.
- **Consistent namespace configuration** — the default namespace should be aligned if you want queues to be visible without explicit namespace qualification.
- **Consistent event-routing policy** — both implementations must route events to the same bus (system vs user vs both). A divergence means one implementation's events reach a bus the other doesn't listen on.

## Limitations

- **Message bodies must be JSON‑serialisable.** All implementations serialise message payloads to JSON. Binary data must be base64‑encoded.
- **Consumer groups are shared.** A consumer group created by Go behaves identically to one created by TypeScript. The ephemeral group naming convention (`cid-{consumerId}`) is the same.
- **Lua script changes** must be propagated to all implementations. The TypeScript repository is the source of truth; the Go implementation syncs scripts automatically via `make sync`. **A version skew in the scripts breaks atomicity guarantees cluster‑wide.**
- **No cross‑Redis communication.** All implementations must connect to the same Redis instance — there is no cross‑instance routing or federation.
