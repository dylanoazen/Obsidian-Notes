---
tags: [docplanner-prep, redis, infra]
status: draft
---
# Redis Patterns

## Cache-Aside
```php
$slots = $cache->get("doctor:{$id}:slots");
if ($slots === null) {
    $slots = $repo->findAvailableSlots($id);
    $cache->set("doctor:{$id}:slots", $slots, 300);
}
```

## Write-Through
```php
$repo->save($appointment);
$cache->set("appointment:{$id}", $appointment, 600); // sempre fresh
```

## Rate Limiting
```php
$key = "rate:{$userId}:" . date('i');
$count = $redis->incr($key);
$redis->expire($key, 60);
if ($count > 100) throw new RateLimitException();
```

## Cache Stampede
Muitos requests no cache miss simultâneo. Solução: lock (só 1 reconstrói).

## Links
- [[DistributedSystems/Redis]] · [[Docplanner/SymfonyServiceContainer]]
