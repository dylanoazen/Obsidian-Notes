# Concurrency & Ownership

Concurrency is the ability to handle multiple tasks at the same time — or at least give the illusion of doing so.

---

## Hardware Threads vs Software Threads

These are two completely different things that share the same name — a common source of confusion.

### Hardware Threads — Fixed

Hardware threads are physical — determined by your CPU. This is the true limit of what runs truly simultaneously.

```
4 cores × 2 threads per core (hyper-threading) = 8 hardware threads
```

At any given moment, only 8 things can truly run at the same time on this machine. This never changes — it is physical.

### Software Threads — Unlimited

Software threads are created by your program. You can create as many as you want:

```go
for i := 0; i < 10000; i++ {
    go handleRequest()  // created 10,000 goroutines
}
```

A web server can have thousands of software threads "running at the same time" on a machine with only 8 hardware threads.

### The OS Scheduler — The Juggler

The OS constantly switches which software thread runs on which hardware thread:

```
You have 8 "rails"
You have 10,000 "trains"

The OS keeps switching which train runs on which rail —
so fast it looks like all trains are moving at once
```

```
Moment 1:  [req1][req2][req3][req4][req5][req6][req7][req8]   ← 8 running
req3 goes to fetch from DB → sleeps, frees the rail
Moment 2:  [req1][req2][req9][req4][req5][req6][req7][req8]   ← req9 slotted in
req1 finishes → frees the rail
Moment 3:  [req10][req2][req9][req4][req5][req6][req7][req8]  ← req10 slotted in
```

### Why 8 Cores Can Handle 10,000 Connections

Most software threads spend **more time sleeping than running**:

```
req1: received → fetched from DB (slept 10ms) → responded
      |← CPU 0.1ms →|←——— sleeping 10ms ———→|← CPU 0.1ms →|
```

When a thread waits for the database, network, or disk — it is not using CPU. The OS puts another thread on that hardware thread while it waits. This is the key insight behind high-concurrency servers.

### Parallelism vs Concurrency

| | **Concurrency** | **Parallelism** |
|---|---|---|
| Definition | Dealing with many things at once | Doing many things at once |
| Requires | Just a scheduler | Multiple hardware threads |
| Example | 1 chef juggling 10 orders | 10 chefs each cooking 1 order |
| Limited by | Design / scheduler | Hardware |

> Concurrency is about structure. Parallelism is about execution.

---

## Single-thread vs Multi-thread

### Single-thread
The program has only one execution line. Everything runs sequentially — one thing at a time.

```
task 1 → task 2 → task 3 → task 4
```

If task 2 takes 5 seconds, tasks 3 and 4 wait.

---

### Multi-thread
Multiple execution lines run in parallel. Each thread handles its own task independently.

```
thread 1 → task 1 → task 3
thread 2 → task 2 → task 4
```

**Problem:** threads share memory, which creates race conditions — two threads modifying the same data at the same time leads to unpredictable results.

---

## Event Loop

The event loop is the model used by single-threaded runtimes (like JavaScript/Node.js) to handle concurrency without multiple threads.

Instead of blocking and waiting, it registers a callback and moves on. When the result is ready, the callback is placed in the queue and executed.

```
┌─────────────────────────────────────────┐
│              Call Stack                 │
│  (executes one thing at a time)         │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│              Event Loop                 │
│  (checks if stack is empty,             │
│   then picks next from queue)           │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│            Callback Queue               │
│  (tasks ready to be executed)           │
└─────────────────────────────────────────┘
```

**How it works:**
1. Code runs on the call stack
2. When an async operation starts (I/O, timer), it's offloaded
3. The stack continues with the next task
4. When the async operation finishes, its callback goes to the queue
5. The event loop picks it up when the stack is empty

---

## I/O and Blocking

**I/O (Input/Output)** refers to operations that communicate with something outside the CPU — disk, network, database, keyboard.

These operations are **slow** compared to CPU execution. The two ways to handle them:

### Blocking I/O
The thread stops and waits for the operation to complete.

```
thread: ──── read file ░░░░░░░░░░░░░░ done ──── continue
                        ↑ waiting, doing nothing
```

Wastes CPU time. Common in traditional servers — one thread per request.

### Non-blocking I/O
The thread starts the operation and moves on. A notification arrives when it's done.

```
thread: ──── start read ──── other work ──── handle result
                                         ↑ callback/signal
```

Efficient. One thread can handle thousands of connections.

---

## Go's Approach: Goroutines

Go doesn't use an event loop. Instead it uses **goroutines** — lightweight threads managed by the Go runtime, not the OS.

```go
go doSomething() // spawns a goroutine
```

**Key differences from OS threads:**
- A goroutine starts at ~2KB of stack (OS thread ~1MB)
- The Go runtime schedules goroutines across available CPU cores
- You can have millions of goroutines running simultaneously

```
Go Runtime
┌──────────────────────────────────────┐
│  goroutine 1 → goroutine 4 → ...     │  ← OS thread 1
│  goroutine 2 → goroutine 5 → ...     │  ← OS thread 2
│  goroutine 3 → goroutine 6 → ...     │  ← OS thread 3
└──────────────────────────────────────┘
```

When a goroutine blocks on I/O, the runtime moves it off the thread and runs another goroutine in its place — so the thread is never idle.

---

## Ownership & Data Races

When multiple goroutines access the same data, a **race condition** can occur — both read and write at the same time, producing unpredictable results.

```go
// DANGEROUS — race condition
counter := 0
go func() { counter++ }()
go func() { counter++ }()
// result could be 1 or 2
```

### Channels — ownership transfer
Go's idiomatic way to share data between goroutines. Instead of sharing memory, you **send** data through a channel — transferring ownership. Think of it like a pipe: one goroutine puts data in, another takes it out.

```go
ch := make(chan int)

go func() {
    ch <- 42 // send — blocks until someone receives
}()

value := <-ch // receive — blocks until something is sent
```

Both sides block and wait for each other — this is what makes the transfer safe.

A common pattern is a **coordinator goroutine** that collects results from many workers running in parallel (fan-out / fan-in):

```go
ch := make(chan int)

// workers run in parallel
go func() { ch <- processOrder(1) }()
go func() { ch <- processOrder(2) }()
go func() { ch <- processOrder(3) }()

// coordinator collects all results
for i := 0; i < 3; i++ {
    result := <-ch
    fmt.Println("result:", result)
}
```

```
goroutine 1 ──► ch ──┐
goroutine 2 ──► ch ──┼──► coordinator (collects results)
goroutine 3 ──► ch ──┘
```

The coordinator doesn't know which goroutine finishes first — it just waits for the next result to arrive. Whoever finishes first sends first.

**Does this have a performance cost?**
Yes — each send/receive involves synchronization between goroutines. But the cost is minimal in practice. The real bottleneck is almost always the work the goroutines are doing, not the channel itself.

> "Do not communicate by sharing memory; share memory by communicating." — Go proverb

### Channels vs Mutex

| | Channel | Mutex |
|---|---|---|
| Model | Transfers the data | Shares the data |
| Cost | Synchronization on transfer | Lock on access |
| Risk | Deadlock if misused | Race condition if misused |
| Readability | High — explicit flow | Low — hard to trace |
| Best for | Communication between goroutines | Protecting shared state |

Use **channel** when goroutines need to **communicate**.
Use **mutex** when goroutines need to **share state**.

### Mutex — shared access control
When you must share memory, use a mutex to ensure only one goroutine accesses it at a time.

```go
var mu sync.Mutex
counter := 0

go func() {
    mu.Lock()
    counter++
    mu.Unlock()
}()
```

---

## Summary

| Model | Thread | I/O | Use case |
|-------|--------|-----|----------|
| Single-thread + Event Loop | 1 | Non-blocking | Node.js, JS |
| Multi-thread | Many | Blocking | Traditional servers |
| Goroutines (Go) | Many (lightweight) | Non-blocking | Go servers |
