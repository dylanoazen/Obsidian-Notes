# Data Structures

A data structure is a way of organizing and storing data so it can be accessed and modified efficiently. The choice of structure directly impacts performance — the wrong one can turn an O(1) operation into O(n).

---

## Study Order

1. **String** — simple key/value
2. **TTL** — expiration
3. **Hash** — multiple fields per key
4. **List** — ordered sequences
5. **Set** — unique items
6. **Sorted Set** — ranking with scores

---

## String

The simplest structure — a key maps directly to a value.

**Go implementation:**
```go
map[string]string
```

**Example:**
```go
store := map[string]string{}
store["name"] = "Dylan"
fmt.Println(store["name"]) // → "Dylan"
```

**Cost:** O(1) for get and set.

**Use when:** storing simple values — user sessions, config flags, counters.

---

## Hash

A map inside a map. Each key holds multiple fields instead of a single value.

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

**Cost:** O(1) for get and set.

**Use when:** storing objects with multiple fields — user profiles, product data, session info.

---

## List

An ordered sequence where insertion order is preserved. Supports efficient operations at both ends.

**Go implementation:**
```go
[]string
```

**Example:**
```go
list := []string{}

// insert at end
list = append(list, "task1")

// insert at start
list = append([]string{"task0"}, list...)

// remove from start
first := list[0]
list = list[1:]

// remove from end
last := list[len(list)-1]
list = list[:len(list)-1]
```

**Cost:** O(1) at the ends, O(n) in the middle.

**Use when:** queues, stacks, history logs, activity feeds.

---

## Set

A collection of unique items — duplicates are ignored.

**Go implementation:**
```go
map[string]struct{}
```

> `struct{}` takes zero bytes — it's the idiomatic way to implement a set in Go.

**Example:**
```go
set := map[string]struct{}{}

// add
set["go"] = struct{}{}
set["go"] = struct{}{} // ignored — already exists

// check existence
_, exists := set["go"] // → true

// remove
delete(set, "go")
```

**Cost:** O(1) for add, remove and lookup.

**Use when:** unique visitors, tags, permissions, friend lists.

---

## Sorted Set

Like a Set, but each item has a **score** that determines its position. Items are always kept in order by score.

**Go implementation:**
Two structures working together:
```go
type SortedSet struct {
    scores  map[string]float64 // member → score (O(1) lookup)
    members []string           // kept sorted by score
}
```

> A **skip list** or **heap** can be used internally for O(log n) inserts and lookups.

**Example:**
```go
ss := SortedSet{
    scores:  map[string]float64{},
    members: []string{},
}

// add member with score
ss.scores["Dylan"] = 100
ss.scores["Alice"] = 250
// members slice must be kept sorted after each insert
```

**Cost:** O(log n) for insert and lookup.

**Use when:** leaderboards, rankings, rate limiting, time-series data.

---

## TTL / Expiration

A way to attach a time limit to any stored value — it automatically becomes invalid after the duration.

**Go implementation:**
Two maps working together:
```go
data       map[string]string    // main store
expiration map[string]time.Time // key → expiry time
```

**Example:**
```go
data := map[string]string{}
expiration := map[string]time.Time{}

// store with TTL
data["session:abc"] = "token123"
expiration["session:abc"] = time.Now().Add(1 * time.Hour)

// get with expiry check
func get(key string) (string, bool) {
    if exp, ok := expiration[key]; ok {
        if time.Now().After(exp) {
            delete(data, key)
            delete(expiration, key)
            return "", false // expired
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
