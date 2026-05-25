# Data Structures

A data structure is a way of organizing and storing data so it can be accessed and modified efficiently. The choice of structure directly impacts performance — the wrong one can turn an O(1) operation into O(n).

> Context: these structures are the foundation of how Redis works internally, and how a mini-cache can be implemented in Go.

---

## Study Order

If you're building a cache (like Redis), follow this order:

1. **String** — GET / SET / DEL
2. **TTL** — expiration
3. **Hash** — multiple fields per key
4. **List** — ordered sequences
5. **Set** — unique items
6. **Sorted Set** — ranking with scores

---

## String

The simplest structure — a key maps directly to a value.

**Redis:** `GET`, `SET`, `DEL`

**Go implementation:**
```go
map[string]string
```

**Example:**
```go
store := map[string]string{}
store["name"] = "Dylan"   // SET name Dylan
fmt.Println(store["name"]) // GET name → "Dylan"
```

**Cost:** O(1) expected for get and set.

**Use when:** storing simple values — user sessions, config flags, counters.

---

## Hash

A map inside a map. Each key holds multiple fields instead of a single value.

**Redis:** `HSET`, `HGET`, `HGETALL`

```bash
HSET user:1 name Dylan age 25
HGET user:1 name  # → "Dylan"
```

**Go implementation:**
```go
map[string]map[string]string
```

**Example:**
```go
store := map[string]map[string]string{}
store["user:1"] = map[string]string{
    "name": "Dylan",
    "age":  "25",
}
fmt.Println(store["user:1"]["name"]) // → "Dylan"
```

**Cost:** O(1) expected.

**Use when:** storing objects with multiple fields — user profiles, product data, session info.

---

## List

An ordered sequence where insertion order is preserved. Supports efficient operations at both ends.

**Redis:** `LPUSH`, `RPUSH`, `LPOP`, `RPOP`, `LRANGE`

```bash
LPUSH tasks "task3"   # insert at left
RPUSH tasks "task4"   # insert at right
LPOP tasks            # remove from left
```

**Go implementation:**
```go
map[string][]string
```

**Example:**
```go
store := map[string][]string{}
store["tasks"] = append([]string{"task3"}, store["tasks"]...) // LPUSH
store["tasks"] = append(store["tasks"], "task4")              // RPUSH
popped := store["tasks"][0]                                    // LPOP
store["tasks"] = store["tasks"][1:]
```

**Cost:** O(1) at the ends, O(n) in the middle.

**Use when:** queues, stacks, history logs, activity feeds.

---

## Set

A collection of unique items — no duplicates allowed.

**Redis:** `SADD`, `SREM`, `SISMEMBER`, `SMEMBERS`

```bash
SADD tags "go"
SADD tags "go"      # ignored — already exists
SISMEMBER tags "go" # → 1 (true)
```

**Go implementation:**
```go
map[string]map[string]struct{}
```

> `struct{}` takes zero bytes — it's the idiomatic way to implement a set in Go.

**Example:**
```go
store := map[string]map[string]struct{}{}
store["tags"] = map[string]struct{}{}
store["tags"]["go"] = struct{}{}      // SADD
store["tags"]["docker"] = struct{}{}

_, exists := store["tags"]["go"]      // SISMEMBER → true
delete(store["tags"], "go")           // SREM
```

**Cost:** O(1) expected.

**Use when:** unique visitors, tags, permissions, friend lists.

---

## Sorted Set

Like a Set, but each item has a **score** that determines its position. Items are always kept in order by score.

**Redis:** `ZADD`, `ZRANK`, `ZRANGE`, `ZSCORE`

```bash
ZADD leaderboard 100 "Dylan"
ZADD leaderboard 250 "Alice"
ZRANGE leaderboard 0 -1 WITHSCORES  # sorted by score
```

**Go implementation:**
Two structures working together:
```go
// member → score (for O(1) score lookup)
scores map[string]float64

// sorted slice for ordered access
members []string  // kept sorted by score
```

> Redis uses a **skip list** internally for O(log n) inserts and lookups. A heap can also be used.

**Example:**
```go
type SortedSet struct {
    scores  map[string]float64
    members []string
}
```

**Cost:** O(log n) for insert and lookup.

**Use when:** leaderboards, rankings, rate limiting, time-series data.

---

## TTL / Expiration

Keys with a time limit — automatically expire after a set duration.

**Redis:** `EXPIRE`, `TTL`, `PERSIST`

```bash
SET session:abc "token123"
EXPIRE session:abc 3600    # expires in 1 hour
TTL session:abc            # → 3598
```

**Go implementation:**
Two maps working together:
```go
data       map[string]string      // main store
expiration map[string]time.Time   // key → expiry time
```

**Example:**
```go
data := map[string]string{}
expiration := map[string]time.Time{}

// SET with TTL
data["session:abc"] = "token123"
expiration["session:abc"] = time.Now().Add(1 * time.Hour)

// GET with expiry check
func get(key string) (string, bool) {
    if exp, ok := expiration[key]; ok {
        if time.Now().After(exp) {
            delete(data, key)
            delete(expiration, key)
            return "", false  // expired
        }
    }
    val, ok := data[key]
    return val, ok
}
```

**Cost:** O(1) per access + periodic cleanup for expired keys.

**Use when:** sessions, caches, rate limiters, temporary tokens.

---

## Complexity Summary

| Structure | Get | Insert | Delete | Use case |
|-----------|-----|--------|--------|----------|
| String | O(1) | O(1) | O(1) | Simple values |
| Hash | O(1) | O(1) | O(1) | Objects with fields |
| List | O(n) | O(1) ends | O(1) ends | Queues, stacks |
| Set | O(1) | O(1) | O(1) | Unique items |
| Sorted Set | O(log n) | O(log n) | O(log n) | Rankings |
| TTL | O(1) | O(1) | O(1) | Expiring keys |
