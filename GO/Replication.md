# Replication

Replication is the process of keeping data synchronized across multiple nodes. A primary node receives writes and one or more replicas copy the data.

---

## Key Concepts

- **Primary (master):** node that accepts writes.
- **Replica:** node that receives a copy of data from the primary.
- **Replication stream:** ordered sequence of changes sent from primary to replicas.

---

## Offsets

An **offset** is the position in the replication stream.

- Primary keeps a current offset
- Each replica tracks its own offset
- If a replica reconnects, it asks to continue from its last offset

This allows incremental catch-up instead of full resync.

---

## Backlog

A **backlog** is a buffer of recent changes kept by the primary.

- Stores the last N bytes/commands of the replication stream
- Enables replicas to catch up after a short disconnect
- If the replica is too far behind, a full resync is required

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

---

## Quick Notes

- Offsets are just sequence positions, not timestamps
- Backlog size is a trade-off: memory vs recovery speed
- Replication is often async; replicas can lag behind
