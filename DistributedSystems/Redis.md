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

---

## Session Management — How It Works in Practice

Redis does not have tables or SQL. The "link" to the user is built into the key name itself:

```
key:   "session:token:abc32323"   ← identifies whose session it is
value: "123"                       ← the user ID
TTL:   3600 seconds                ← expires automatically
```

### Full Flow

**Login:**
```
User logs in with correct password
→ server generates a random token: "abc32323"
→ SET session:token:abc32323 "123" EX 3600
→ returns token to the browser (cookie/header)
```

**Every request:**
```
Browser sends token "abc32323"
→ server asks Redis: GET session:token:abc32323
→ Redis returns: "123" (the user ID)
→ authenticated ✓

If Redis returns nil → session expired or never existed → 401
```

**Logout:**
```
DEL session:token:abc32323
→ next request: GET session:token:abc32323 → nil → not authenticated
```

### Middleware Pattern (PHP)

The session check does not live inside each action — it lives in a single file included everywhere. One check runs before anything else on every protected page.

```php
// config.php — included in every protected page

$redis = new Redis();
$redis->connect('127.0.0.1', 6379);

$token = $_COOKIE['token'] ?? null;

if (!$token) {
    header('Location: /login.php');
    exit;
}

$user_id = $redis->get("session:token:$token");

if (!$user_id) {
    header('Location: /login.php');
    exit;
}

// if we got here, $user_id is available for the entire page
```

Every protected page:
```php
<?php
require_once 'config.php';  // ← check done, $user_id available

// rest of the page normally
echo "Welcome, user $user_id";
?>
```

The login page is the only one that does **not** include this file — or includes it without the redirect, since the user does not have a session yet.

> Redis does not notify you when a session expires — it just returns nil. Your application decides what to do: redirect to login, return 401, etc.

## Resources

- https://redis.io/docs
- https://try.redis.io (interactive tutorial)
- https://university.redis.io (free courses)

#### My commentaries
- 
