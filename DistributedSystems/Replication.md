# Replication

Replication is the process of keeping data synchronized across multiple nodes. A primary node receives writes and one or more replicas copy the data.

---

## Core Terms

- **Primary (master):** node that accepts writes.
- **Replica:** node that receives a copy of data from the primary.
- **Replication stream:** ordered sequence of changes sent from primary to replicas.
- **Offset:** current position in the replication stream.
- **Backlog:** buffer of recent changes kept by the primary.

---

## Offsets

An **offset** is the position in the replication stream — a single integer that grows monotonically as commands are processed.

- Primary keeps a current offset (advances with every write)
- Each replica tracks its own offset (advances as it applies commands)
- If a replica reconnects, it sends its last known offset to the primary

The primary then compares offsets:
- Gap is small and still in backlog → **partial resync** (send only the missing commands)
- Gap is too large or offset not found → **full resync** (send a full RDB snapshot)

This allows incremental catch-up instead of always copying everything from scratch.

### How the offset works step by step

```
Primary offset: 100

  [write: SET name "João"]  → primary offset becomes 101
  [write: SET age 25]       → primary offset becomes 102
  [write: SET name "Maria"] → primary offset becomes 103

Replica was at offset 101 and reconnects:
  → asks: "give me everything after 101"
  → primary sends: commands at 102 and 103
  → replica applies them, now also at offset 103 ✓
```

### Why offset and not a timestamp?

Timestamps have problems in distributed systems — clocks drift between machines. An offset is just a counter: deterministic, unambiguous, and always increasing. There is no disagreement about what "103" means.

---

## Backlog

A **backlog** is a buffer of recent changes kept by the primary.

- Stores the last N bytes/commands of the replication stream
- Enables replicas to catch up after a short disconnect
- If the replica is too far behind, a full resync is required

Backlog size is a trade-off: memory vs recovery speed.

---

## Full Resync vs Partial Resync

- **Partial resync:** replica requests missing data starting at its offset
- **Full resync:** replica discards state and copies a full snapshot

Partial resync is faster and cheaper, but only works if the backlog still contains the missing data.

---

## Minimal Flow (Concept)

1) Replica connects to primary
2) Replica sends its last known offset
3) Primary decides:
   - If backlog covers the gap -> send missing data (partial resync)
   - Else -> send full snapshot (full resync)
4) Replica applies changes and updates its offset

---

## Failure Scenarios

- **Network drop:** replica reconnects using its last offset
- **Long outage:** backlog too small -> full resync
- **Lag:** replica behind -> reads may be stale

## Failover

Failover is when the primary dies and a replica automatically takes its place.

```
Before:
Primary (alive)  →  Replica

Primary dies ↓

After:
Primary (dead)      Replica → promoted to new Primary
```

### How it works

1. Primary stops responding
2. The system detects it is down
3. The replica with the least lag is **promoted** to primary
4. The application starts writing to it
5. When the old primary comes back, it becomes a replica of the new one

### Without failover vs with failover

| | **Without failover** | **With failover** |
|---|---|---|
| Primary dies | App stops until someone fixes it manually | App recovers in seconds, automatically |
| Human needed? | Yes | No |

### Who manages failover in Redis

- **Redis Sentinel** — monitors the primary, triggers failover, notifies the app of the new primary address
- **Redis Cluster** — built-in failover as part of the cluster protocol

> Replication is the mechanism. Failover is what replication makes possible.

---

## When to Use Replication

Replication makes sense when:

- **Data loss has real cost** — shopping carts, queues, application state that cannot be trivially rebuilt
- **Read throughput needs to scale** — distribute reads across replicas, keep writes on the primary
- **Availability matters** — if the primary goes down, a replica takes over and the system keeps running

A single Redis instance is fine when the data is purely a disposable cache — if it dies, you rebuild from the source of truth elsewhere.

> Set up replication when the cost of losing the data exceeds the cost of running two instances.

## Replication vs Horizontal Scaling

These are often confused but solve different problems:

| | **Replication** | **Horizontal Scaling** |
|---|---|---|
| What is copied | Data (state) | The application (logic) |
| Example | Redis primary + replicas | 10 containers of your backend |
| Why | Survival + read scaling | Throughput + availability |
| Needs sync? | Yes — replicas must stay in sync | No — each instance is independent |

Your **backend** (the application) you scale by adding more instances — no replication needed, because it is **stateless**: it does not hold state, it just processes requests. Any instance can handle any request.

Your **Redis** you replicate because it is **stateful**: it *is* the state. If it dies without a replica, the data is gone.

> Stateless things you scale. Stateful things you replicate.

### Horizontal Scaling in Practice

When your backend is under heavy load, you scale it by launching more instances — more containers, more processes, more machines. This works because the backend is **stateless**: every request carries everything it needs (token, body, params), so any instance can handle any request.

```
Before:
[Container 1 - Backend]  ← overloaded

After:
[Container 1 - Backend]
[Container 2 - Backend]  ← same app, more instances
[Container 3 - Backend]
         ↑
    Load Balancer distributes requests across them
```

State lives **outside** the containers:

```
[Container 1]  ↘
[Container 2]  →  Redis (sessions, cache)  +  PostgreSQL (data)
[Container 3]  ↗
```

All containers read and write to the same Redis and the same database. None of them store anything locally.

This is exactly why Redis replication matters at scale — if 3 containers depend on a single Redis and it dies, all 3 stop. Replication ensures Redis survives.

> You scale the backend horizontally because it is stateless.
> You replicate Redis because it carries the state of all of them.

## Design Notes

- Offsets are sequence positions, not timestamps
- Replication is often async; replicas can lag behind
- If backlog is smaller than the write rate over the disconnect window, full resync is inevitable
- These concepts (offset, backlog, partial/full resync) only matter when something breaks — on the happy path, the primary just streams commands to replicas continuously and offsets stay in sync

---

## Minimal Protocol Sketch

- Replica -> Primary: `PSYNC <replica_id> <offset>`
- Primary -> Replica: `+CONTINUE <offset>` or `+FULLRESYNC <offset>`
- Primary -> Replica: stream of writes (each write advances offset)
---

## Related Notes

- [[Persistence]] — RDB, AOF, e como se conectam com replicação
- [[Concurrency]]
- [[TCP]]
