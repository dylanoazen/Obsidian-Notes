# Expiration / TTL

Expiration (TTL) is the ability to set a time limit on a key. After the TTL, the key is removed automatically.

---

## Key Concepts

- **TTL (Time To Live):** how long a key should exist.
- **Expire time:** absolute timestamp when the key becomes invalid.
- **Expired key:** a key whose TTL has passed.

---

## Example (10s TTL, read at 12s)

- Key is set with TTL = 10 seconds.
- If the client reads the key after 12 seconds, it is already expired.
- The server deletes it and returns "missing".

---

## Lazy Expiration

**Lazy expire** means keys are removed only when they are accessed.

**How it works:**
- Client requests a key
- Server checks if TTL has expired
- If expired, delete and return missing

**Pros:**
- Very cheap and simple
- No background work

**Cons:**
- Expired keys can stay in memory if never accessed

---

## Active Expiration

**Active expire** means the server periodically scans keys and removes expired ones.

**How it works:**
- Timer runs every N milliseconds
- Random sample of keys is checked
- Expired keys are deleted

**Pros:**
- Frees memory even if keys are never accessed

**Cons:**
- Background cost (CPU)
- Needs tuning to avoid latency spikes

---

## Hybrid Approach

A common strategy is to use **both**:
- Lazy expire on access
- Active expire in background

This balances correctness and performance.

---

## Eviction Policies

Eviction happens **when memory is full**, not just when keys expire.

Common policies:

- **noeviction**
  - Reject new writes when memory is full

- **allkeys-lru**
  - Remove least recently used key (any key)

- **allkeys-lfu**
  - Remove least frequently used key (any key)

- **allkeys-random**
  - Remove random key

- **volatile-lru**
  - Remove least recently used key **with TTL**

- **volatile-lfu**
  - Remove least frequently used key **with TTL**

- **volatile-random**
  - Remove random key **with TTL**

- **volatile-ttl**
  - Remove key with the **shortest remaining TTL**

---

## Implementation Model (Concept)

You can store TTLs in a separate map:

```
store:   map[string]string
expires: map[string]time.Time
```

Check on read:
- If key has TTL and it expired -> delete and return missing

Background cleanup:
- On a timer, scan a sample of keys and remove expired

---

## Quick Notes

- TTL is not a promise of exact time, only an upper bound
- Lazy expire ensures correctness on access
- Active expire ensures memory does not fill with dead keys

---

## Terms to Remember

- TTL
- Expire time
- Lazy expiration
- Active expiration
- Eviction policy
- LRU / LFU
