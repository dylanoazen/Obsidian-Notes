# Transactions & Pipeline

Two mechanisms that control how multiple commands are sent and executed — but they solve different problems.

---

## The Problem They Solve

By default, every Redis command is independent:

```
SET balance 100   ← executes now
SET balance 90    ← executes now, separately
```

What if you need multiple commands to:
- Execute together as a single unit? → **Transaction**
- Be sent all at once without waiting? → **Pipeline**

### The Restaurant Analogy

**Pipeline** — you write down 10 orders on a piece of paper and hand it all at once to the waiter, instead of calling him 10 separate times. You saved 9 trips.

**Transaction** — the waiter only brings the food when the drink is also ready. No drink, no food. Either everything arrives together or nothing arrives.

**Both together** — you hand the waiter a list of orders *(pipeline)* and tell him "only bring it all when everything is ready" *(transaction)*.

> Pipeline solves the **delivery**. Transaction solves the **rule of execution**.

---

## Transactions

A transaction groups multiple commands so they execute **atomically** — all or nothing, with no other client interrupting in between.

```
MULTI         ← start transaction
SET balance 90
SET log "deducted 10"
EXEC          ← execute all at once
```

### Atomicity

Atomicity means: **either all commands run, or none do.**

If the server crashes mid-transaction, none of the commands take effect. No partial state.

**Analogy:** a bank transfer — debit one account and credit another. If only the debit runs and the credit fails, the money disappears. Both must succeed or neither should happen.

### What Redis transactions guarantee

- Commands between `MULTI` and `EXEC` are queued, not executed immediately
- At `EXEC`, all queued commands run sequentially with no interruption from other clients
- No other client can sneak a command between them

### What Redis transactions do NOT guarantee

- If one command inside the transaction fails (e.g. wrong type), the others still run — Redis does not roll back
- This is different from SQL transactions which roll back everything on failure

---

## Pipeline

A pipeline is about **network efficiency** — sending multiple commands to Redis in a single round trip instead of one by one.

```
Without pipeline:
[Client] → SET a 1 → [Redis] → OK → [Client] → SET b 2 → [Redis] → OK

With pipeline:
[Client] → SET a 1 / SET b 2 / SET c 3 → [Redis] → OK / OK / OK
```

### The round-trip problem

Every command sent individually pays a round-trip cost:
- Client sends command
- Waits for Redis to respond
- Then sends the next command

With 100 commands, you pay 100 round trips. With a pipeline, you pay 1.

```
Without pipeline:
GET user:1  → wait for response
GET user:2  → wait for response
GET user:3  → wait for response
(3 round trips)

With pipeline:
GET user:1
GET user:2  → send all at once, receive all at once
GET user:3
(1 round trip)
```

Redis is key-value — you already know exactly which key you want before asking. Pipeline just lets you ask for many keys at once without waiting between each one.

| | **SQL** | **Redis + Pipeline** |
|---|---|---|
| You say | "give me everyone with age > 18" | "give me keys user:1, user:2, user:3" |
| You need to know | Just the condition | The exact keys upfront |
| Pipeline helps | Does not apply | Send multiple fetches in one round trip |

### Batching

Batching is the act of grouping multiple commands to send together. Pipeline is how you implement batching in Redis.

### What pipeline does NOT guarantee

- Commands in a pipeline are **not atomic** — other clients can execute commands between them
- If the connection drops mid-pipeline, some commands may have run and others not

---

## Transactions vs Pipeline

| | **Transaction** | **Pipeline** |
|---|---|---|
| Purpose | Atomicity | Network efficiency |
| Atomic? | Yes | No |
| Reduces round trips? | No | Yes |
| Commands queued? | On server | On client |
| Other clients blocked? | Yes (during EXEC) | No |

They are complementary — you can use both at the same time: send a transaction through a pipeline.

---

## Using Both Together

```
[Client pipelines these commands in one round trip]

MULTI
SET balance 90
SET log "deducted 10"
EXEC
```

- Pipeline handles the **network** — one round trip
- Transaction handles the **atomicity** — no interruptions

---

## Practical Examples

**Transaction:** deducting balance and logging it — both must happen or neither should
**Pipeline:** loading a user profile that requires 5 GET commands — send all 5 at once, receive all 5 responses at once

---

## Design Notes

- Redis is single-threaded for command execution — transactions work because nothing else runs between MULTI and EXEC
- Pipeline is a client-side optimization — the client buffers commands and flushes them together
- For true rollback behavior (like SQL), Redis is not the right tool
- WATCH command can be used before MULTI to abort a transaction if a key was modified by another client (optimistic locking)

---

## Related Notes

- [[Replication]]
- [[PubSub]]
- [[Persistence]]
