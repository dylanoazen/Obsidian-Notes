# Persistence

Persistence is the ability to keep data alive after the process stops. Without it, everything stored in memory is lost when the server restarts.

There are two main strategies:

---

## RDB — Snapshot

RDB (Redis Database) takes a **full snapshot** of all data in memory and saves it to disk at a given point in time.

```
memory ──► [snapshot] ──► dump.rdb (file on disk)
```

**How it works:**
- At a defined interval (or manually), Go forks the process
- The child process writes a full copy of the data to a `.rdb` file
- The main process keeps running without interruption

**Configuration example:**
```
save 900 1     # snapshot if at least 1 key changed in 900s
save 300 10    # snapshot if at least 10 keys changed in 300s
save 60 10000  # snapshot if at least 10000 keys changed in 60s
```

**Pros:**
- Compact single file — easy to backup and transfer
- Fast restarts — loading one file is quick
- No performance impact during normal operation

**Cons:**
- Data between snapshots is lost on crash
- If the server crashes at 4:59 and snapshot runs every 5 minutes, you lose ~5 minutes of data

**Use when:** you can tolerate some data loss and want fast restarts.

---

## AOF — Append-Only File

AOF records **every write operation** as it happens, appending it to a log file. On restart, it replays the log to rebuild the state.

```
SET name Dylan  ──► appended to file
SET age 25      ──► appended to file
DEL name        ──► appended to file

restart ──► replay all entries ──► state restored
```

**How it works:**
- Every write command is appended to `appendonly.aof`
- On restart, the file is read line by line and each command is re-executed
- Over time the file grows — a rewrite (compaction) can shrink it

**Fsync policies:**
```
appendfsync always    # write to disk on every command (safest, slowest)
appendfsync everysec  # write to disk every second (good balance)
appendfsync no        # let the OS decide (fastest, least safe)
```

**Pros:**
- At most 1 second of data loss (with `everysec`)
- Human-readable log — you can inspect or edit it
- More durable than RDB

**Cons:**
- Larger file size than RDB
- Slower restarts — must replay every command
- Rewrite process needed to keep file size manageable

**Use when:** data durability is critical and you cannot afford to lose more than 1 second of data.

---

## RDB vs AOF

| | RDB | AOF |
|---|---|---|
| What it saves | Full snapshot | Every write command |
| Data loss risk | Minutes (between snapshots) | At most 1 second |
| File size | Small | Large (grows over time) |
| Restart speed | Fast | Slow (replays all commands) |
| Best for | Backups, fast restarts | Durability, audit log |

---

## Using Both Together

RDB and AOF can be used simultaneously — RDB for fast restarts and AOF for durability. On restart, AOF takes priority since it has more recent data.

```
RDB  ──► fast restore of base snapshot
AOF  ──► replay only the commands since last snapshot
```

---

## No Persistence

If neither is enabled, the data lives only in memory — everything is lost on restart. Useful for pure caching where data can be rebuilt from the source.

```
restart ──► empty state
```

---

## Implementation Concept in Go

```go
// RDB — save snapshot
func saveRDB(store map[string]string) error {
    data, _ := json.Marshal(store)
    return os.WriteFile("dump.rdb", data, 0644)
}

// RDB — load snapshot
func loadRDB() map[string]string {
    data, err := os.ReadFile("dump.rdb")
    if err != nil {
        return map[string]string{}
    }
    store := map[string]string{}
    json.Unmarshal(data, &store)
    return store
}

// AOF — append command
func appendAOF(command string) error {
    f, err := os.OpenFile("appendonly.aof", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()
    _, err = f.WriteString(command + "\n")
    return err
}
```
