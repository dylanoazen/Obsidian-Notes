# Memory Management

Memory management is about understanding **where data lives, how it gets there, and how it gets cleaned up**.

Every program needs memory to store data while it runs. How that memory is requested, used, and released determines the performance and reliability of the system.

---

## Stack vs Heap

Every program has two main places to store data:

```
┌─────────────────────────────────┐
│             STACK               │  ← fast, small, automatic
├─────────────────────────────────┤
│              HEAP               │  ← slower, large, managed
└─────────────────────────────────┘
```

---

## Stack

The stack is fast, automatic memory tied to function calls.

- When a function is called → a stack frame is created
- When a function returns → the stack frame is destroyed
- No allocator involved — just a pointer moving up and down
- Each thread has its own stack

```
main() calls handleRequest()
  handleRequest() calls parseCommand()

Stack:
┌──────────────────┐
│  parseCommand()  │  ← top (current function)
├──────────────────┤
│ handleRequest()  │
├──────────────────┤
│     main()       │
└──────────────────┘

parseCommand() returns → frame destroyed
handleRequest() returns → frame destroyed
```

### What lives on the stack

- Local variables with known, fixed size
- Function arguments and return values
- Variables that do not outlive the function

**Analogy:** a whiteboard you erase after each meeting. Instant to write, instant to erase, but limited space.

---

## Heap

The heap is a large pool of memory for data that needs to **outlive a function** or has a **size unknown at compile time**.

- Slower to allocate — goes through a memory allocator (jemalloc, tcmalloc, etc.)
- Lives until explicitly freed or garbage collected
- Shared across threads

If a variable needs to survive after the function that created it returns, it must live on the heap.

**Analogy:** a storage unit you rent. More space, but someone has to manage it — and the rent has to be paid until you leave.

---

## Manual vs Automatic Memory Management

Different languages handle heap cleanup differently:

| Approach | Language | How |
|---|---|---|
| **Manual** | C, C++ | Programmer calls `free()` — fast but error-prone |
| **Garbage Collected** | Go, Java, Python | Runtime cleans up automatically |
| **Ownership system** | Rust | Compiler enforces rules at compile time — no GC needed |

### Manual (C / C++)

```c
char *buf = malloc(1024);  // allocate
// use buf...
free(buf);                 // you must free it
// if you forget → memory leak
// if you free twice → crash
// if you use after free → undefined behavior
```

Fast, but dangerous. Redis is written in C — every allocation is manual.

### Garbage Collector

The runtime periodically scans the heap, finds data that nothing points to anymore, and frees it automatically.

```
heap:
[User A ✓][string ✓][int ✗][User B ✓][[]byte ✗]
                      ↑ nothing points here    ↑ nothing points here

GC runs → frees the ✗ blocks
```

Safer and easier, but the GC has a cost — it runs on CPU and can cause latency pauses.

### Ownership (Rust)

The compiler tracks who owns each piece of memory. When the owner goes out of scope, memory is freed automatically — at compile time, with no runtime cost.

```rust
{
    let s = String::from("hello");  // s owns the memory
    // use s...
}  // s goes out of scope → memory freed automatically, no GC needed
```

---

## Garbage Collector — How It Works

The most common GC algorithm is **Mark and Sweep**:

**Phase 1 — Mark:**
Starting from roots (stack variables, globals), follow all pointers and mark everything reachable:

```
roots → User{} ✓ → name string ✓
      → cache map ✓ → entries ✓
      → (old buffer nobody points to) ✗ — never reached
```

**Phase 2 — Sweep:**
Free everything unmarked:

```
[User ✓][string ✓][buffer ✗ freed][map ✓][[]byte ✗ freed]
```

### Stop-the-World vs Concurrent GC

Older GC implementations paused the entire program while collecting:

```
Stop-the-world:
────────────────|░░░░░░░GC░░░░░░|────────────────
program runs     everything pauses   program resumes
```

This caused latency spikes. Modern GCs (Go, JVM G1) run **concurrently** — most work happens while the program keeps running, with only brief pauses.

---

## Memory Leaks

A memory leak happens when memory is allocated but never freed — the program slowly consumes more and more RAM until it crashes or is killed.

Common causes:
- **Manual:** forgetting to call `free()`
- **GC languages:** keeping a reference to data you no longer need (GC can't collect it if something still points to it)
- **Caches without eviction:** storing data forever without a TTL

```
// leak in a GC language — cache grows forever
var cache = map[string][]byte{}

func store(key string, data []byte) {
    cache[key] = data  // never removed → GC never collects it
}
```

---

## Stack vs Heap — Summary

| | **Stack** | **Heap** |
|---|---|---|
| Speed | Instant | Slower (allocator + GC) |
| Size | Small, per-thread | Large, shared |
| Lifetime | Function scope | Until freed or GC'd |
| Managed by | Automatically | Allocator + GC or programmer |
| Fragmentation | None | Yes |
| GC involvement | None | Yes |

---

## Design Notes

- Stack allocations are essentially free — prefer them when possible
- Heap allocations have cost: allocator time + future cleanup work
- GC is not free — it runs on CPU and competes with your program
- Memory leaks in long-running servers are fatal — they compound over time
- Measure before optimizing — profilers show exactly where allocations happen

---

## Language-Specific Notes

- **Go** → see [[GO/MemoryManagement]]
- **Redis (C)** → manual memory management via jemalloc, see [[Performance]]

---

## Related Notes

- [[Performance]]
- [[Concurrency]]
- [[Persistence]]
