# Caching

## What it is
Storing the result of an expensive computation or data fetch in fast storage (memory) so subsequent requests are served quickly without repeating the work.

## Why it matters
Caching is often the single highest-leverage performance improvement. The difference between a 200ms DB query and a 1ms Redis cache hit at 10,000 RPS is enormous.

---

## Cache-Aside (Lazy Loading)

Most common pattern. The application manages the cache itself.

```
1. Look in cache
2. If cache hit → return cached value
3. If cache miss → fetch from DB → store in cache → return value
```

```csharp
public async Task<Product?> GetProductAsync(Guid id, CancellationToken ct)
{
    var cacheKey = $"product:{id}";

    // 1. Check cache
    var cached = await _cache.GetStringAsync(cacheKey, ct);
    if (cached is not null)
        return JsonSerializer.Deserialize<Product>(cached);

    // 2. Cache miss — fetch from DB
    var product = await _repository.GetByIdAsync(id, ct);
    if (product is null) return null;

    // 3. Store in cache
    var options = new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
    };
    await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product), options, ct);

    return product;
}
```

**Pros:** Only caches what's actually used. DB is always the source of truth.
**Cons:** Cold start — first request always goes to DB. Cache can become stale.

---

## Write-Through

Write to cache and DB at the same time on every update.

```
Write → DB + Cache simultaneously
Read → always from cache
```

**Pros:** Cache is always up to date. No stale data.
**Cons:** Higher write latency. Cache fills with data that may never be read.

---

## Write-Behind (Write-Back)

Write to cache immediately. Write to DB asynchronously.

```
Write → Cache (immediately) → DB (async, batched)
```

**Pros:** Very fast writes.
**Cons:** Risk of data loss if cache goes down before DB write. Complex consistency.

---

## Cache Eviction Policies

When cache is full, something must be evicted:

| Policy | Evicts | Use when |
|---|---|---|
| **LRU** (Least Recently Used) | Oldest accessed item | General purpose — default choice |
| **LFU** (Least Frequently Used) | Least accessed item | Access frequency matters |
| **FIFO** | Oldest inserted item | Order of insertion matters |
| **TTL** | Items past their expiry | Time-sensitive data |

Redis default: LRU (configurable with `maxmemory-policy`)

---

## Redis in .NET

### Setup
```bash
dotnet add package StackExchange.Redis
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Program.cs
services.AddStackExchangeRedisCache(opts =>
    opts.Configuration = config["Redis:ConnectionString"]);
```

### Using IDistributedCache
```csharp
// Inject IDistributedCache
private readonly IDistributedCache _cache;

// Get
var json = await _cache.GetStringAsync(key, ct);
var value = json is not null ? JsonSerializer.Deserialize<T>(json) : null;

// Set
await _cache.SetStringAsync(key, JsonSerializer.Serialize(value),
    new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) },
    ct);

// Remove (on update/delete)
await _cache.RemoveAsync(key, ct);
```

### Using IMemoryCache (in-process, single server)
```csharp
services.AddMemoryCache();

// Get or create
var value = await _memoryCache.GetOrCreateAsync(key, async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await _repository.GetByIdAsync(id, ct);
});
```

**IMemoryCache vs IDistributedCache:**
- `IMemoryCache`: In-process, single server. Lost on restart. Fast.
- `IDistributedCache`: External (Redis), shared across all servers. Survives restarts. Use in production.

---

## Cache Invalidation

"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Time-based (TTL)
Simple. Data can be stale for up to TTL. Acceptable for product lists, catalog data.

### Event-based
Invalidate on write. Requires tracking which cache keys relate to which data.

```csharp
// After updating a product
await _productRepository.UpdateAsync(product, ct);
await _uow.CommitAsync(ct);

// Invalidate cache
await _cache.RemoveAsync($"product:{product.Id}", ct);
await _cache.RemoveAsync("products:all", ct);  // invalidate list cache too
```

### Cache Key Design
Include enough specificity to avoid collisions and enable targeted invalidation:
```
product:{id}               → single product
products:page:{n}:size:{k} → paginated list
user:{id}:cart             → user's cart
category:{id}:products     → products in category
```

---

## Cache Stampede (Thundering Herd)

**Problem:** Many requests arrive simultaneously for an expired key. All find a miss and all hit the DB at once.

**Solutions:**
- **Lock/mutex:** Only one request fetches from DB; others wait
- **Probabilistic early expiration:** Randomly expire slightly before TTL
- **Background refresh:** Refresh cache before it expires

```csharp
// Simple lock pattern
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task<Product?> GetWithLockAsync(Guid id, CancellationToken ct)
{
    var cached = await GetFromCacheAsync(id, ct);
    if (cached is not null) return cached;

    await _lock.WaitAsync(ct);
    try
    {
        // Double-check after acquiring lock
        cached = await GetFromCacheAsync(id, ct);
        if (cached is not null) return cached;

        var product = await _repository.GetByIdAsync(id, ct);
        await SetInCacheAsync(id, product, ct);
        return product;
    }
    finally
    {
        _lock.Release();
    }
}
```

---

## Distributed Cache Consistency

In a multi-server setup, each server queries the shared Redis cache. If one server updates the cache, all servers see the update immediately.

**Challenge:** What if Redis is unavailable?
- **Circuit breaker:** Fall back to DB when Redis is down
- **Never fail because of cache** — cache is a performance optimization, not a source of truth

---

## What to Cache

| Cache | Example | TTL |
|---|---|---|
| Product catalog | `product:{id}` | 10-30 min |
| Category list | `categories:all` | 1 hour |
| User session data | `session:{token}` | Session lifetime |
| Search results | `search:{hash}` | 5 min |
| Rate limit counters | `rate:{ip}:{window}` | Window duration |
| Auth token blacklist | `blacklist:{jti}` | Token remaining lifetime |

### Don't Cache:
- Frequently changing data (stock levels in real-time)
- User-specific data unless keyed per user
- Sensitive data (passwords, payment info)
- Data that's cheaper to compute than to cache

---

## Redis Data Structures

| Structure | Use case |
|---|---|
| String | Simple key-value, counters |
| Hash | Object fields (`HSET user:1 name "Ivan" email "ivan@..."`) |
| List | Message queue, recent activity feed |
| Set | Unique values, social graph (`SADD user:1:followers user:2`) |
| Sorted Set | Leaderboards, rate limiting, feed ranked by score |
| Pub/Sub | Real-time event broadcasting |

---

## Common Interview Questions

1. What is the difference between cache-aside and write-through?
2. What is a cache stampede and how do you prevent it?
3. When would you choose IMemoryCache vs IDistributedCache?
4. How do you invalidate cache when data changes?
5. What are Redis eviction policies?

---

## Common Mistakes

- Caching mutable data without invalidation strategy
- Using the same cache key for different users' data
- Not handling cache failure gracefully (falling back to DB)
- Over-caching: caching data that's fast to compute anyway
- Cache key collisions between different environments (add env prefix in production)

---

## How It Connects

- Cache-aside is the Decorator pattern applied to repositories
- Redis pub/sub can notify other services when cache is invalidated
- Rate limiter uses Redis sorted sets or counters
- RTK Query is client-side caching for frontend — same concepts
- Cache TTL design depends on your data freshness requirements (business decision)

---

## My Confidence Level
- `[~]` Cache-aside pattern
- `[~]` Write-through / write-behind
- `[ ]` Redis data structures
- `[ ]` Cache eviction policies
- `[ ]` Distributed cache consistency

## My Notes
<!-- Personal notes -->
