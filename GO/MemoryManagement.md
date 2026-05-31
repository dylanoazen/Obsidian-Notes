# Memory Management in Go

Go manages memory automatically — you allocate, the runtime cleans up. But understanding how it works internally lets you write faster, more predictable code.

Three concepts: **Stack**, **Heap**, and **Garbage Collector**.

---

## Stack

The stack is fast, automatic memory tied to function calls.

- When a function is called → stack frame is created
- When a function returns → stack frame is destroyed
- Each goroutine has its own stack (starts at ~2KB, grows as needed)

```go
func add(a, b int) int {
    result := a + b   // lives on the stack
    return result     // stack frame destroyed on return
}
```

**Key property:** stack memory is free — no allocator, no GC. Just a pointer moving up and down.

**Analogy:** a whiteboard you erase after each meeting. Instant to write, instant to erase.

---

## Heap

The heap is memory for data that needs to **outlive a function** or has a **size unknown at compile time**.

```go
func newUser(name string) *User {
    u := &User{Name: name}  // escapes to heap
    return u                // function returns, but User survives
}
```

If `User` lived on the stack, it would be destroyed when `newUser` returns. Since we return a pointer to it, Go puts it on the heap.

Heap allocations are slower — they go through the memory allocator (Go uses a custom allocator inspired by tcmalloc) and eventually need to be cleaned up by the GC.

**Analogy:** a storage unit you rent. More space, but someone has to manage it.

---

## Escape Analysis

Go decides at **compile time** whether a variable goes to the stack or heap.

> If the variable is used only inside the function → **stack**
> If the variable "escapes" the function → **heap**

```go
// stack — used only locally
func localOnly() {
    x := 42
    fmt.Println(x)
}

// heap — pointer returned, x outlives the function
func escapes() *int {
    x := 42
    return &x
}

// heap — sent to a channel, outlives the function
func toChannel(ch chan *User) {
    u := &User{}
    ch <- u
}
```

### See escape analysis decisions

```bash
go build -gcflags="-m" ./...

# output:
# ./main.go:8:2: x escapes to heap
# ./main.go:14:6: u does not escape
```

This is useful when optimizing hot paths — every heap escape has a GC cost.

---

## Garbage Collector

The GC automatically frees heap memory that nothing points to anymore.

### How it works — Tri-color Mark and Sweep

**Mark phase:** starting from roots (stack variables, globals), follow all pointers and mark everything reachable:

```
roots → User{} ✓ → name string ✓ → ...
      → cache map ✓ → entries ✓ → ...
      → (old buffer nobody points to) ✗ — never reached
```

**Sweep phase:** free everything unmarked:

```
[User ✓][string ✓][buffer ✗ freed][map ✓][[]byte ✗ freed]
```

### Concurrent GC — Go's approach

Go's GC runs **concurrently** with your program. Most work happens while goroutines keep running:

```
your program:  ────────────────────────────────────────
GC:                  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                              |short STW|   ← stop-the-world pause (< 1ms)
```

STW (stop-the-world) pauses in modern Go are typically under **1ms**. But under heavy allocation, GC runs more frequently — and the pauses add up in P99 latency.

### GOGC — controlling GC frequency

```bash
GOGC=100   # default — GC triggers when heap doubles in size
GOGC=200   # GC runs less often — uses more memory, less CPU
GOGC=50    # GC runs more often — less memory, more CPU
GOGC=off   # disable GC entirely (only for special cases)
```

Higher GOGC = less GC work = lower CPU cost = higher memory usage.
Lower GOGC = more GC work = more CPU cost = lower memory usage.

---

## Reducing GC Pressure

Every heap allocation is future GC work. Reducing allocations = reducing GC pressure = lower P99 latency.

### 1. sync.Pool — reuse objects

Instead of allocating a new buffer on every request, reuse them:

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func handleRequest() {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)
    // use buf to parse command, write response...
}
```

The buffer is reused across requests — allocator and GC never see it after the first allocation.

### 2. Avoid unnecessary pointers

Every pointer the GC has to follow is more work. Flat structs with value types are GC-friendly:

```go
// GC-heavy — pointer to every field
type Heavy struct {
    Name    *string
    Age     *int
    Active  *bool
}

// GC-friendly — value types, no pointers
type Light struct {
    Name    string
    Age     int
    Active  bool
}
```

### 3. Preallocate slices

```go
// bad — slice grows, reallocates multiple times
results := []string{}
for _, item := range items {
    results = append(results, process(item))
}

// good — one allocation upfront
results := make([]string, 0, len(items))
for _, item := range items {
    results = append(results, process(item))
}
```

### 4. Avoid string + concatenation in loops

```go
// bad — allocates a new string on every iteration
s := ""
for _, part := range parts {
    s += part  // new allocation each time
}

// good — one allocation at the end
var sb strings.Builder
for _, part := range parts {
    sb.WriteString(part)
}
s := sb.String()
```

---

## Profiling — Find Where Allocations Happen

Do not guess. Measure.

```bash
# run with profiling
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap

# shows which functions allocate the most
```

```go
// expose pprof endpoint in your server
import _ "net/http/pprof"

go http.ListenAndServe(":6060", nil)
```

### Runtime memory stats

```go
var stats runtime.MemStats
runtime.ReadMemStats(&stats)

fmt.Println("heap in use:", stats.HeapInuse)
fmt.Println("GC cycles:", stats.NumGC)
fmt.Println("GC pause total:", stats.PauseTotalNs)
```

---

## Stack vs Heap — Summary

| | **Stack** | **Heap** |
|---|---|---|
| Speed | Instant | Slower (allocator + GC) |
| Size | Small, per-goroutine | Large, shared |
| Lifetime | Function scope | Until GC frees it |
| GC involvement | None | Yes |
| How to use | Local variables | Pointers, interfaces, closures |

---

## In MiniRedisGo Context

Every `SET key value` command your server handles involves allocations:

```
parse command  → []byte buffer  → heap
store key      → string         → heap
store value    → []byte         → heap
write response → []byte buffer  → heap
```

Under heavy load, this becomes significant GC pressure. Optimizations:

- Use `sync.Pool` for command parsing buffers
- Reuse response buffers
- Profile with pprof under load to find the biggest allocators
- Watch `mem_fragmentation_ratio` if you add a persistence layer

---

## Related Notes

- [[DistributedSystems/MemoryManagement]]
- [[DistributedSystems/Performance]]
- [[Concurrency]]
