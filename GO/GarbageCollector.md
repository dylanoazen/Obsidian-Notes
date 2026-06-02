# Garbage Collector

The Garbage Collector (GC) is the part of the Go runtime that automatically frees heap memory that is no longer reachable by the program.

Related: [[GO/MemoryManagement|Memory Management in Go]]

---

## Why GC Matters

- Without GC, you must manually free memory (like C/C++) — error-prone
- GC trades some CPU time for safety: no dangling pointers, no double frees
- Understanding GC helps you reduce allocations and avoid latency spikes

## Go's GC Algorithm

Go uses a **concurrent, tri-color, mark-and-sweep** collector.

### Mark Phase

- Starts from **root objects** (globals, stack variables, goroutine stacks)
- Traverses all reachable objects, marking them as "alive"
- Runs **concurrently** with the application (not stop-the-world)

### Sweep Phase

- Walks through all heap memory
- Frees any object that was **not marked** as alive
- Also runs concurrently

```
Roots (stacks, globals)
    │
    ▼
  Mark: walk all references, mark reachable objects
    │
    ▼
  Sweep: free unmarked objects
    │
    ▼
  Done — memory reclaimed
```

## Tri-Color Marking

The GC classifies objects into three colors:

- **White**: not yet visited — candidates for collection
- **Grey**: visited, but references not fully scanned
- **Black**: visited, all references scanned — definitely alive

```
Start:   all objects are White
Process: pick Grey → scan its references → move to Black
         referenced Whites → move to Grey
End:     remaining Whites are garbage → free them
```

The key invariant: **a Black object never points directly to a White object** (maintained by write barriers).

## Write Barrier

Since the GC runs concurrently with the application, the program might modify pointers while the GC is scanning.

- The **write barrier** is a small hook that runs on every pointer write during GC
- It ensures new references are tracked so the GC doesn't miss live objects
- Small performance cost, but enables concurrent collection

## GC Triggers

The GC runs when:

- Heap size grows to **2x** the live heap after last collection (default)
- Controlled by `GOGC` environment variable (default = 100, meaning 100% growth)
- `GOGC=50` → GC runs more often (less memory, more CPU)
- `GOGC=200` → GC runs less often (more memory, less CPU)
- `GOGC=off` → disables GC entirely

```go
// Force a GC cycle
runtime.GC()

// Read GC stats
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Println("HeapAlloc:", stats.HeapAlloc)
fmt.Println("NumGC:", stats.NumGC)
fmt.Println("PauseTotalNs:", stats.PauseTotalNs)
```

## GOMEMLIMIT (Go 1.19+)

- Sets a **soft memory limit** for the entire Go process
- GC becomes more aggressive as memory approaches the limit
- Better than GOGC alone for memory-constrained environments (containers)

```bash
GOMEMLIMIT=512MiB ./myapp
```

## STW Pauses (Stop-the-World)

Go's GC is mostly concurrent, but has **two brief STW pauses**:

1. **Mark start**: enable write barrier, scan stacks (~microseconds)
2. **Mark termination**: finalize marking, disable write barrier (~microseconds)

Typical STW pauses: **< 1ms**, often **< 100μs** in modern Go versions.

## Reducing GC Pressure

Practical techniques to minimize allocations:

- **sync.Pool**: reuse temporary objects instead of allocating new ones
- **Pre-allocate slices**: `make([]T, 0, expectedSize)` avoids repeated growing
- **Avoid pointer-heavy structures**: fewer pointers = less work for GC to scan
- **Stack allocation**: keep values local, avoid escaping to heap
- **String builders**: use `strings.Builder` instead of `+` concatenation
- **Escape analysis**: run `go build -gcflags="-m"` to see what escapes to heap

```go
// sync.Pool example
var bufPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

func process(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    buf.Reset()
    defer bufPool.Put(buf)
    // use buf...
}
```

## Profiling GC

```bash
# GC trace — see every GC cycle
GODEBUG=gctrace=1 ./myapp

# Output example:
# gc 1 @0.012s 2%: 0.019+0.45+0.003 ms clock, ...
#    ^ GC#  ^time ^CPU%  ^STW1 ^concurrent ^STW2

# pprof — heap profile
go tool pprof http://localhost:6060/debug/pprof/heap
```

## Comparison with Other Languages

| Language | GC Type | STW Pauses |
|----------|---------|------------|
| Go | Concurrent mark-sweep | < 1ms |
| Java (G1) | Generational, concurrent | Configurable, can be low |
| Python | Reference counting + cycle detector | GIL pauses |
| Rust | No GC (ownership system) | None |
| C/C++ | Manual | None |

## Resources

- https://tip.golang.org/doc/gc-guide
- https://go.dev/blog/ismmkeynote (Go GC design talk)

#### My commentaries
- 
