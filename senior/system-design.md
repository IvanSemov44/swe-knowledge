# System Design

## What it is
The process of defining the architecture, components, and data flow of a large-scale system to meet functional and non-functional requirements (scale, availability, latency, consistency).

## Why it matters
System design interviews are standard for mid/senior roles. More importantly, understanding these concepts prevents you from building systems that can't scale or recover from failures.

---

## Framework for System Design Interviews

1. **Clarify requirements** (5 min)
   - Functional: what does the system do?
   - Non-functional: scale (users/requests), latency SLAs, availability (99.9% = 8h downtime/year), consistency requirements

2. **Back-of-envelope estimation** (5 min)
   - DAU (daily active users), QPS (queries per second), storage requirements
   - 1M users × 10 requests/day = 10M requests/day = ~115 QPS

3. **High-level design** (10 min)
   - Draw the main components: clients, load balancer, API servers, databases, caches

4. **Deep dive into components** (20 min)
   - Database schema, sharding strategy, cache design, API design

5. **Discuss trade-offs** (10 min)
   - CAP theorem, consistency vs availability, SQL vs NoSQL

---

## Key Estimations

| Thing | Size |
|---|---|
| 1 character | 1 byte |
| UUID | 16 bytes |
| Average tweet | 300 bytes |
| Average image | 1 MB |
| Average video (1 min) | 50 MB |
| 1M requests/day | ~12 RPS |
| 1B requests/day | ~12,000 RPS |
| QPS for 1M users (10 req/user/day) | ~115 QPS |

---

## Common System Design Problems

### URL Shortener (e.g., bit.ly)

**Requirements:**
- Given a long URL, generate a short URL
- Redirect short URL to original
- ~100M URLs, 10:1 read:write ratio

**Core design:**
```
POST /shorten → generate 6-char code, store in DB
GET /{code} → look up in DB, return 301 redirect
```

**Unique code generation:**
- Base62 (a-z, A-Z, 0-9) with 6 chars = 62^6 = 56 billion combinations
- Options: random + check, MD5(url)+truncate, auto-increment ID → base62

**Caching:** Popular URLs served from Redis (cache-aside). LRU eviction.

**Database:** Simple K-V store (Redis) or SQL. If Redis: hash of short_code → long_url.

**Scale:** Single DB + read replicas for high read:write ratio.

---

### Rate Limiter

**Requirements:** Limit to N requests per user per time window.

**Algorithms:**
- **Token Bucket:** Bucket fills at rate R. Each request consumes 1 token. Allows bursts.
- **Fixed Window Counter:** Count requests in a fixed time window. Simple but boundary burst problem.
- **Sliding Window Log:** Store timestamp of each request. Accurate but memory-intensive.
- **Sliding Window Counter:** Hybrid. 95% accuracy with low memory.

**In Redis:**
```
Token bucket with Lua script (atomic increment + expire)
INCR user:{id}:requests:{window}
EXPIRE user:{id}:requests:{window} 60
```

**In ASP.NET Core:**
```csharp
services.AddRateLimiter(opts =>
    opts.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
    }));
```

---

### News Feed (e.g., Twitter/Instagram)

**Requirements:** User sees posts from people they follow, in reverse chronological order.

**Approach 1 — Fan-out on Write (Push):**
- When user posts, write to all followers' feeds immediately
- Read feed from pre-computed cache: fast reads, slow writes
- Problem: celebrity with 10M followers → 10M writes per post

**Approach 2 — Fan-out on Read (Pull):**
- When user opens feed, query all followees' recent posts and merge
- No write amplification, but slower reads

**Approach 3 — Hybrid:**
- Regular users: push to followers' feeds
- Celebrities: pull on read, merge at feed generation time

**Storage:** Posts in DB + Redis sorted set per user (feed:{userId} → sorted by timestamp)

---

### Notification System

**Requirements:** Send email, SMS, push notifications. Handle millions/day.

**Components:**
- **API Service:** Receives notification requests from internal services
- **Message Queue:** Decouples notification creation from delivery (Kafka/RabbitMQ)
- **Workers:** Email worker, SMS worker, push notification worker
- **Retry with dead-letter queue:** Failed deliveries retry, then go to DLQ for investigation

---

## CAP Theorem

In a distributed system, you can only guarantee 2 of 3:
- **C**onsistency: all nodes see the same data at the same time
- **A**vailability: every request gets a response
- **P**artition tolerance: system works despite network partition

Since network partitions are unavoidable in distributed systems, you choose **CP** or **AP**.

| System | Choice | Example |
|---|---|---|
| SQL databases (single node) | CA | PostgreSQL, SQL Server |
| Distributed SQL | CP | Google Spanner, CockroachDB |
| Eventual consistency | AP | DynamoDB, Cassandra |

---

## SQL vs NoSQL

| | SQL | NoSQL |
|---|---|---|
| Schema | Fixed, enforced | Flexible |
| Relationships | Built-in (joins) | Manual |
| ACID | Yes | Often eventual consistency |
| Scale | Vertical + read replicas | Horizontal sharding |
| Query flexibility | High (any query) | Limited (design for access patterns) |
| Use when | Relational data, ACID needed | Massive scale, flexible schema, key-value |

**For your E-commerce app:** SQL Server is correct. Relational data, ACID transactions for orders.

---

## Caching in System Design

See `caching.md` for detail. Key point: cache at the right layer.

```
Client → CDN (static assets) → Load Balancer → App Server → Redis (hot data) → DB
```

---

## Common Interview Questions

1. Design a URL shortener
2. Design a rate limiter
3. Design a news feed
4. What is the CAP theorem?
5. When would you use NoSQL over SQL?
6. How does horizontal scaling differ from vertical scaling?
7. What is database sharding?

---

## Common Mistakes

- Designing for a scale you'll never reach (premature optimization)
- Not asking clarifying questions (scale, read:write ratio, consistency needs)
- Ignoring failure scenarios (what happens if the cache is down?)
- Over-indexing on one aspect (only talking about the DB, forgetting caching, queues)

---

## How It Connects

- Rate limiter → Redis + ASP.NET Core `AddRateLimiter()`
- Notification system → RabbitMQ + worker services (see `message-queues.md`)
- Caching → Redis (see `caching.md`)
- CAP theorem → why your E-commerce app uses SQL (consistency > availability for orders)

---

## My Confidence Level
- `[ ]` URL shortener design
- `[ ]` Rate limiter design
- `[ ]` News feed design
- `[ ]` Notification system design
- `[ ]` CAP theorem

## My Notes
<!-- Personal notes -->
