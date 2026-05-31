# Performance

Performance is about understanding **where time and memory go** — and reducing both.

Three pillars: **latency**, **memory allocation**, and **benchmarking**.

---

## Latency

Latency is the time between a request being sent and a response being received.

```
Client sends request  →  [time passes]  →  Client receives response
                              ↑
                          latency
```

**Analogy:** you order food at a restaurant. Latency is the time between ordering and the food arriving at your table.

### Latency vs Throughput

These are related but different:

|          | **Latency**                       | **Throughput**                |
| -------- | --------------------------------- | ----------------------------- |
| Question | How fast is one request?          | How many requests per second? |
| Unit     | milliseconds (ms)                 | requests/sec (RPS)            |
| Analogy  | How fast one car crosses a bridge | How many cars cross per hour  |

You can have high throughput with high latency — many requests processed, but each one slowly. The goal is usually both: fast and many.

### Sources of Latency

```
Total latency = network + queue + processing + response

Network      → physical distance, hops between servers
Queue        → waiting behind other requests
Processing   → actual computation time
Response     → sending the result back
```

### Latency Numbers Worth Knowing

| Operation                 | Approximate Latency |
| ------------------------- | ------------------- |
| L1 cache access           | ~1 ns               |
| RAM access                | ~100 ns             |
| SSD read                  | ~100 µs             |
| Network (same datacenter) | ~1 ms               |
| Network (cross-continent) | ~100 ms             |
| HDD read                  | ~10 ms              |

This is why Redis is fast — it reads from RAM (~100 ns), not disk.

### P50, P95, P99 — Percentile Latency

Averages lie. A system with average 10ms latency might have 1% of requests taking 2 seconds.

Percentiles tell the real story:

```
P50  = 50% of requests finish in under X ms  (median)
P95  = 95% of requests finish in under X ms
P99  = 99% of requests finish in under X ms
P999 = 99.9% of requests finish in under X ms
```

**Analogy:** average salary in a company with one billionaire CEO looks high. The median (P50) tells you what most employees actually earn.

In production, you optimize P99 — the worst 1% still affects real users.

---

## Memory Allocator

A memory allocator manages how your program requests and releases memory from the OS.

Your program never talks directly to hardware. When you need memory, you talk to the allocator, and it talks to the OS:

```
Your code  →  malloc(64)  →  Allocator  →  OS (only when necessary)
```

The OS works in pages (usually 4KB). Allocating 4KB every time you need 64 bytes would be wasteful. The allocator requests large pages from the OS and subdivides them internally.

### How Memory Allocation Works

When your program needs memory:

```
Program  →  "I need 64 bytes"  →  Allocator
Allocator → finds free block → returns pointer
Program uses memory
Program  →  "I'm done with this"  →  Allocator
Allocator → marks block as free → available for reuse
```

### The Problem: Fragmentation

Over time, memory ends up looking like swiss cheese — lots of small free holes that are hard to reuse:

```
Before:
[████████████████████████████████] 100MB contiguous

After many allocs/frees:
[used][    ][used][used][    ][used][    ]
       16MB        8MB        4MB
```

You have 28MB free total, but in separate chunks. If you need 20MB contiguous — it is not available.

### How ptmalloc (glibc) Works — and Why It Is Slow

ptmalloc was designed in 1987 for single-threaded workloads and had threading bolted on later. Its model is too simple for modern servers:

```
Thread 1 wants to alloc  →  locks the heap  →  allocs  →  unlocks
Thread 2 wants to alloc  →  waiting in queue...
Thread 3 wants to alloc  →  waiting in queue...
```

In systems with many threads, threads sleep waiting for the lock. This becomes latency.

### How jemalloc Solves This — Arenas

jemalloc divides memory into independent **arenas**. Each thread is assigned to an arena, so threads rarely compete for the same lock:

```
Thread 1  →  Arena 1  (own lock)
Thread 2  →  Arena 2  (own lock)
Thread 3  →  Arena 1  (shares with Thread 1, but rarely conflicts)
```

Inside each arena, jemalloc divides allocations into **size classes** — predefined fixed sizes:

```
Small  (≤ 14KB)   →  thread-local cache (tcache) — zero locks
Large  (14KB–4MB) →  directly from arena
Huge   (> 4MB)    →  mmap'd directly from OS
```

For small allocations (the majority), jemalloc uses a **per-thread cache** (tcache) — zero contention, zero lock.

jemalloc also solves fragmentation with size classes — instead of allocating exactly what was requested, it rounds up to the nearest fixed size:

```
You request 57 bytes  →  jemalloc allocates 64 bytes (next size class)
You request 100 bytes →  jemalloc allocates 128 bytes
```

Result: blocks of the same size stay together, easy to reuse:

```
[64][64][64][64][64]  ← all equal, any one works for any ≤64 byte request
```

### Common Allocators

| Allocator | Used by | Notes |
|---|---|---|
| **glibc (ptmalloc)** | Default on Linux | General purpose, slow under contention |
| **jemalloc** | Redis, Firefox, Rust | Best fragmentation handling for long-running servers |
| **tcmalloc** | Google, Go | Per-thread private cache, best throughput for high-concurrency |
| **mimalloc** | Microsoft | Modern, very fast across workloads |

### jemalloc vs tcmalloc

| | **jemalloc** | **tcmalloc** |
|---|---|---|
| Strong at | Fragmentation — long-running servers | Throughput — many threads, many small allocs |
| Thread model | Shared arenas | Fully private per-thread cache |
| Ideal for | Redis, databases | HTTP servers, Google-style applications |
| Weak at | High-frequency small allocs | Long-running with varied allocation patterns |

### How This Appears in Redis

Redis reports two memory metrics:

```bash
INFO memory

used_memory:     1000000000   # 1GB — what Redis thinks it is using
used_memory_rss: 1500000000   # 1.5GB — what the OS sees

mem_fragmentation_ratio: 1.5  # used_by_OS / used_by_Redis
```

**Fragmentation ratio above 1.5 = problem.** jemalloc is holding memory in its internal bins that the OS counts as used, but Redis is not actively using.

**How to fix:**

```bash
MEMORY PURGE  # Redis 4.0+ — forces jemalloc to return idle pages to the OS
```

Redis 4.0+ also has **active defragmentation** — automatically reallocates fragmented keys in the background.

### Why Redis Chose jemalloc

Redis is a process that runs for weeks or months, making millions of small allocations (keys, values, internal structures). This is exactly the scenario where jemalloc excels — controlled fragmentation over time.

With ptmalloc, after weeks of running the process would consume 3-4x more RAM than the actual data occupies.

### Why It Matters

A bad allocator on a high-traffic system can:
- Fragment memory → process uses 2GB RAM but only 1GB is actual data
- Slow down allocation → extra latency on every operation
- Cause unexpected OOM (out of memory) kills

**References:**
- [Memory Allocators: malloc vs. tcmalloc vs. jemalloc](https://howtech.substack.com/p/memory-allocators-malloc-vs-tcmalloc)
- [Exploring Different Memory Allocators](https://dev.to/frosnerd/libmalloc-jemalloc-tcmalloc-mimalloc-exploring-different-memory-allocators-4lp3)
- [How Redis jemalloc Memory Allocator Works](https://oneuptime.com/blog/post/2026-03-31-redis-jemalloc-memory-allocator/view)
- [How to Troubleshoot Redis Memory Fragmentation](https://oneuptime.com/blog/post/2026-03-31-redis-how-to-troubleshoot-redis-memory-fragmentation/view)
- [Testing Memory Allocators: ptmalloc2 vs tcmalloc vs hoard vs jemalloc](http://ithare.com/testing-memory-allocators-ptmalloc2-tcmalloc-hoard-jemalloc-while-trying-to-simulate-real-world-loads/)

---

## Benchmarking

Benchmarking is the process of **measuring performance under controlled conditions** to understand where your system stands and where it can improve.

### Why Benchmark

- Verify your system meets performance requirements
- Compare two implementations (which is faster?)
- Catch regressions — did the new code make things slower?
- Find bottlenecks — where is time actually being spent?

### Types of Benchmarks

**Microbenchmark** — measures a single small operation in isolation

```go
// how fast is a single SET command?
for i := 0; i < 1000000; i++ {
    redis.Set("key", "value")
}
```

**Load test** — simulates real traffic at scale

```
Send 10,000 requests/sec for 60 seconds
Measure: latency P50/P95/P99, error rate, throughput
```

**Stress test** — pushes beyond expected limits to find breaking points

```
Keep increasing load until the system fails
Observe: when does it degrade? when does it break?
```

### Common Benchmarking Tools

| Tool | Use case |
|---|---|
| **redis-benchmark** | Built-in Redis benchmark |
| **wrk / wrk2** | HTTP load testing |
| **k6** | Scriptable load testing |
| **pprof** (Go) | CPU and memory profiling |
| **perf** (Linux) | System-level performance analysis |

### Redis Benchmark Example

```bash
# run 100,000 SET commands with 50 concurrent clients
redis-benchmark -t set -n 100000 -c 50

# output:
# 100000 requests completed in 1.23 seconds
# 50 parallel clients
# Latency: avg 0.5ms, P99 1.2ms
```

### Benchmarking Pitfalls

- **Benchmark in production-like conditions** — your laptop is not your server
- **Warm up first** — cold start skews results (JIT, caches, connections)
- **Measure P99, not just average** — averages hide outliers
- **Isolate variables** — change one thing at a time
- **Run multiple times** — variance is real, one run is not enough

---

## How They Connect

```
Latency       →  what you measure (how slow is it?)
Allocator     →  one source of latency (memory operations)
Benchmarking  →  how you find where latency comes from
```

The loop:

```
Benchmark → find bottleneck → fix it (tune allocator, reduce copies, cache) → benchmark again
```

---

## In Redis Context

- Redis is fast because it operates entirely in **RAM** — latency in microseconds
- Uses **jemalloc** to minimize fragmentation over long uptimes
- **redis-benchmark** is built-in for quick performance checks
- Latency spikes in Redis often come from: large keys, blocking commands, persistence flushes (AOF fsync)

---

## Design Notes

- Premature optimization is the root of all evil — benchmark first, optimize what the data shows
- Latency budget: know how much latency each layer is allowed to add
- Memory fragmentation ratio above 1.5 in Redis is a warning sign
- Always measure before and after an optimization to confirm it helped

---

## Related Notes

- [[Replication]]
- [[Persistence]]
- [[TLS]]
