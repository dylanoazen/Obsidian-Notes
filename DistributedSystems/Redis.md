# Redis

Redis (Remote Dictionary Server) is an in-memory data structure store used as a database, cache, message broker, and streaming engine.

It defines:
- key-value storage with sub-millisecond latency
- rich data structures beyond simple strings
- persistence options for durability
- built-in replication and clustering

---

## Why Redis?

- Extremely fast — all operations happen in memory
- Versatile data structures (not just key-value)
- Built-in pub/sub for real-time messaging
- Persistence options (RDB snapshots, AOF logs)
- Replication and high availability out of the box
- Widely adopted — used by Twitter, GitHub, Stack Overflow, etc.

## Core Data Structures

- **String**: simplest type, can store text, numbers, or binary (max 512MB)
- **List**: ordered collection, supports push/pop from both ends (queues, stacks)
- **Set**: unordered unique elements, supports unions, intersections, diffs
- **Sorted Set (ZSet)**: like Set but each member has a score, auto-sorted
- **Hash**: field-value pairs inside a single key (like a mini object/document)
- **Stream**: append-only log, used for event sourcing and message queues

## Basic Commands

```
SET key value              # store a string
GET key                    # retrieve a string
DEL key                    # delete a key
EXPIRE key seconds         # set TTL
TTL key                    # check remaining TTL

LPUSH list value           # push to head of list
RPOP list                  # pop from tail

SADD set member            # add to set
SMEMBERS set               # list all members

HSET hash field value      # set field in hash
HGETALL hash               # get all fields

ZADD zset score member     # add to sorted set
ZRANGE zset 0 -1           # get all sorted
```

## Architecture

```
Client ──► Redis Server (single-threaded event loop)
                │
                ├── Memory (primary storage)
                ├── RDB (periodic snapshots to disk)
                ├── AOF (append-only log to disk)
                └── Replication → Replica nodes
```

- Single-threaded for commands — no locks, no race conditions
- I/O multiplexing handles thousands of concurrent connections
- Background threads handle persistence and slow operations

## Topics

- [[DistributedSystems/Persistence|Persistence (RDB & AOF)]]
- [[DistributedSystems/Replication|Replication]]
- [[DistributedSystems/PubSub|Pub/Sub]]
- [[DistributedSystems/TransactionsPipeline|Transactions & Pipeline]]
- [[DistributedSystems/SecurityAuth|Security & Auth]]
- [[DistributedSystems/TLS|TLS]]
- [[DistributedSystems/Performance|Performance]]
- [[DistributedSystems/MemoryManagement|Memory Management]]
- [[DistributedSystems/Concurrency|Concurrency]]
- [[DistributedSystems/DataStructures|Data Structures]]
- [[DistributedSystems/ExpirationTTL|Expiration & TTL]]

## Use Cases

- **Caching**: session storage, API response caching, page caching
- **Rate Limiting**: track request counts with INCR + EXPIRE
- **Leaderboards**: sorted sets with scores
- **Real-time Chat**: pub/sub channels
- **Job Queues**: lists with LPUSH/BRPOP pattern
- **Session Store**: fast read/write for user sessions

## Resources

- https://redis.io/docs
- https://try.redis.io (interactive tutorial)
- https://university.redis.io (free courses)

#### My commentaries
- 
