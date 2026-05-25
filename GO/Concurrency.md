# Concurrency & Ownership

Concurrency is the ability to handle multiple tasks at the same time — or at least give the illusion of doing so.

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
