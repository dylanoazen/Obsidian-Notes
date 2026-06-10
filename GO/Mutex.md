# Mutex in Go

A **Mutex** (mutual exclusion) is a lock that ensures only one goroutine accesses a piece of shared data at a time.

---

## The Problem — Race Condition

When multiple goroutines read and write the same variable simultaneously, the result is unpredictable:

```go
counter := 0

go func() { counter++ }()
go func() { counter++ }()

// expected: 2
// actual: could be 1 or 2 — race condition
```

Why? `counter++` is not one operation — it is three:

```
1. read counter  (gets 0)
2. add 1         (gets 1)
3. write counter (writes 1)
```

If two goroutines do this simultaneously, both read `0`, both add `1`, both write `1` — and the final result is `1` instead of `2`.

---

## Mutex — The Fix

A Mutex is a lock. Only one goroutine can hold the lock at a time. Others wait.

```go
var mu sync.Mutex
counter := 0

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

Now:
```
goroutine 1: Lock() → reads 0 → adds 1 → writes 1 → Unlock()
goroutine 2: waits... Lock() → reads 1 → adds 1 → writes 2 → Unlock()
result: 2 ✓
```

---

## sync.Mutex vs sync.RWMutex

Go has two types of mutex:

### sync.Mutex — exclusive lock

Only one goroutine at a time, for both reads and writes.

```go
var mu sync.Mutex

mu.Lock()    // acquire exclusive lock
// critical section — only one goroutine here
mu.Unlock()  // release
```

### sync.RWMutex — read/write lock

Multiple goroutines can read simultaneously, but writes are exclusive.

```go
var mu sync.RWMutex

// reading — multiple goroutines allowed at the same time
mu.RLock()
value := store[key]
mu.RUnlock()

// writing — exclusive, blocks all readers and writers
mu.Lock()
store[key] = value
mu.Unlock()
```

**When to use RWMutex:** when reads are much more frequent than writes. A cache is the classic example — thousands of reads, occasional writes.

| | **sync.Mutex** | **sync.RWMutex** |
|---|---|---|
| Reads | One at a time | Many simultaneously |
| Writes | One at a time | One at a time |
| Use when | Read/write equally frequent | Reads >> writes |

---

## defer mu.Unlock() — Always Unlock

If your function returns early or panics without unlocking, the program deadlocks. Use `defer` to guarantee unlock:

```go
func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()  // always runs, even if function panics or returns early

    val, ok := s.data[key]
    return val, ok
}
```

---

## Real Example — Thread-Safe Cache

```go
type Store struct {
    mu   sync.RWMutex
    data map[string]string
}

func NewStore() *Store {
    return &Store{
        data: make(map[string]string),
    }
}

func (s *Store) Set(key, value string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = value
}

func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    val, ok := s.data[key]
    return val, ok
}

func (s *Store) Delete(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.data, key)
}
```

This is exactly the pattern for your goCache store — multiple goroutines handling connections, all accessing the same map safely.

---

## Deadlock — The Main Danger

A deadlock happens when two goroutines are each waiting for the other to release a lock — forever.

```go
var mu1, mu2 sync.Mutex

// goroutine 1
go func() {
    mu1.Lock()
    mu2.Lock()   // waiting for goroutine 2 to release mu2
    // ...
}()

// goroutine 2
go func() {
    mu2.Lock()
    mu1.Lock()   // waiting for goroutine 1 to release mu1
    // ...
}()

// both wait forever — deadlock
```

**How to avoid:**
- Always acquire locks in the same order
- Keep the critical section (code between Lock and Unlock) as short as possible
- Use `defer mu.Unlock()` so you never forget to unlock

---

## Detecting Race Conditions

Go has a built-in race detector:

```bash
go run -race ./cmd/server
go test -race ./...
```

It instruments the code at runtime and reports any concurrent access to shared data without proper synchronization:

```
WARNING: DATA RACE
Write at 0x00c0000b4010 by goroutine 7:
  main.main.func1()
      /main.go:12

Previous read at 0x00c0000b4010 by goroutine 6:
  main.main.func2()
      /main.go:17
```

Run with `-race` during development. It has a performance cost (~5x slower) so do not use in production.

---

## Mutex vs Channel — When to Use Each

```
Use Mutex when:   protecting shared state (a map, a counter, a struct)
Use Channel when: communicating between goroutines (sending data, signaling)
```

```go
// Mutex — protecting a shared map
var mu sync.RWMutex
var cache map[string]string

// Channel — signaling between goroutines
results := make(chan string)
go func() { results <- process() }()
```

> "Do not communicate by sharing memory; share memory by communicating." — Go proverb
>
> In practice: if you need a shared data structure → Mutex. If you need coordination between goroutines → Channel.

---

## In goCache Context

Your store is accessed by multiple goroutines simultaneously — one per client connection. Without a Mutex, two clients doing `SET` at the same time would corrupt the map.

```go
// store.go — the pattern to use
type Store struct {
    mu   sync.RWMutex
    data map[string]string
    ttl  map[string]time.Time
}

// SET — exclusive write lock
func (s *Store) Set(key, value string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.data[key] = value
}

// GET — shared read lock (multiple clients can read simultaneously)
func (s *Store) Get(key string) (string, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    // check TTL, return value...
}
```

---

## Related Notes

- [[GO/MemoryManagement]]
- [[DistributedSystems/Concurrency]]
- [[DistributedSystems/Performance]]
