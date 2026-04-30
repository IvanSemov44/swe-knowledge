# Database Internals
<!-- level: Senior | B-tree, WAL, MVCC, partitioning, replication, sharding, Redis deep, Elasticsearch -->

## B-tree Internals

A B-tree is the default index structure in Postgres, SQL Server, and MySQL.

**Page structure (8 KB default):**
- Each node is one page. Internal nodes store keys + child pointers. Leaf nodes store keys + row pointers (heap TID in Postgres; actual row data in SQL Server clustered indexes).
- Height for 1 billion rows ≈ 4–5 levels. One root I/O + 3–4 branch I/Os + 1 leaf I/O = ~5 random reads per lookup — still O(log N).

**Fill factor:** Leaves are not packed 100%. Default fill factor = 90% (Postgres) / 80% (SQL Server). Leftover space absorbs inserts without an immediate split.

**Page splits:** When a full leaf receives an insert, it splits into two half-full pages. The middle key is promoted to the parent. If the parent is also full, the split cascades upward. Splits are expensive (I/O + WAL writes) but rare on sequential data.

**UUID primary keys (random insert problem):**
Random UUIDs scatter inserts across the entire index. Every insert lands on a different (likely cached-out) page, causing constant page splits and poor cache utilisation.

```sql
-- Bad: random UUID causes page splits everywhere
CREATE TABLE orders (id UUID DEFAULT gen_random_uuid() PRIMARY KEY, ...);

-- Good: sequential UUID, monotonically increasing
-- SQL Server
CREATE TABLE orders (id UNIQUEIDENTIFIER DEFAULT NEWSEQUENTIALID() PRIMARY KEY, ...);
```

```csharp
// .NET 9+: Guid.CreateVersion7() generates time-ordered UUIDs
var id = Guid.CreateVersion7(); // safe to use as PK
```

Integer PKs (`IDENTITY` / `SERIAL`) are still the simplest choice for single-node databases.

---

## LSM Tree (Log-Structured Merge-Tree)

Used by RocksDB (backing MySQL MyRocks, CockroachDB), Cassandra, InfluxDB, LevelDB.

**Write path:**
1. Write lands in in-memory **memtable** (sorted skip list or red-black tree).
2. When memtable is full, it is flushed as an immutable **SSTable** (Sorted String Table) on disk.
3. All disk writes are sequential — no random I/O. Writes are very fast.

**Compaction:** Background threads merge SSTables, eliminate tombstones (deleted keys), and enforce level size limits. Without compaction, read amplification grows (must scan N SSTables per key).

**Bloom filters:** Each SSTable carries a Bloom filter. Before reading an SSTable, check the filter — if it says "definitely not here", skip the file. Reduces read I/O dramatically.

**B-tree vs LSM trade-off:**

| | B-tree | LSM |
|---|---|---|
| Write | Moderate (random I/O) | Very fast (sequential) |
| Read | Fast (single path) | Moderate (may check multiple SSTables) |
| Space amp | Low | Higher (compaction lag) |
| Best for | OLTP reads | Write-heavy, time-series, analytics |

---

## Write-Ahead Log (WAL)

**Core guarantee:** A change is only considered durable when its WAL record is on disk — the data page itself can stay in memory (buffer pool) until later.

**Commit sequence:**
1. Write WAL record to WAL buffer.
2. `fsync` WAL buffer to disk.
3. Return success to client.
4. Data page written to disk lazily by background writer.

If the server crashes after step 2 but before step 4, the WAL lets Postgres replay the missing change on restart.

**Log Sequence Numbers (LSNs):** Every WAL record has a monotonically increasing LSN. Recovery replays from the last checkpoint LSN forward. LSNs are also used by streaming replication.

**Checkpointing:** Postgres periodically flushes all dirty buffer pages to disk and writes a checkpoint record. This bounds recovery time (only WAL since last checkpoint is replayed).

**`synchronous_commit = off`:** WAL is not fsynced before client ack. Gain: lower latency. Risk: on crash, up to `wal_writer_delay` (default 200ms) of committed transactions can be lost. Data never corrupts — just reverts to a slightly earlier consistent state.

**SQL Server equivalent:** Transaction log (`.ldf`). `CHECKPOINT` flushes dirty pages. Log shipping is analogous to WAL archiving.

---

## MVCC (Multi-Version Concurrency Control)

**Core idea:** Never block a reader with a writer. Keep old row versions alive so concurrent readers see a consistent snapshot without locks.

**Postgres row headers:**

| Field | Meaning |
|---|---|
| `xmin` | Transaction ID that created this row version |
| `xmax` | Transaction ID that deleted/updated this row (0 = still live) |

A SELECT sees a row if `xmin` committed before the snapshot and `xmax` is either 0 or did not commit before the snapshot.

**Snapshot isolation:** Each transaction gets a snapshot of which XIDs were committed at start. It ignores rows created by XIDs after its snapshot (even if committed before the SELECT runs). Writers never block readers; readers never block writers.

**Write skew:** Snapshot isolation does NOT prevent write skew (two transactions read overlapping data and each writes based on a stale view). Serializable isolation (`SERIALIZABLE` level) uses SSI (Serializable Snapshot Isolation) predicate locks.

**SQL Server row versioning:** Under `READ_COMMITTED_SNAPSHOT` or `SNAPSHOT` isolation, SQL Server stores old row versions in the **version store** in `tempdb`. Functionally equivalent to Postgres MVCC but costs tempdb I/O.

---

## VACUUM and Bloat (Postgres)

**Dead tuples:** When a row is UPDATEd or DELETEd, the old version is kept (MVCC requires it for active snapshots). Once no active transaction can see it, the tuple is "dead" — wasted space.

**Table bloat:** A table that runs 1 000 UPDATEs/s continuously will bloat if AUTOVACUUM can't keep up.

**AUTOVACUUM triggers:**
```
dead tuples > autovacuum_vacuum_threshold (50) + autovacuum_vacuum_scale_factor (0.2) × table rows
```
For a 10M-row table: triggers at 2 000 050 dead tuples. On very large tables, lower the scale factor:

```sql
ALTER TABLE orders
  SET (autovacuum_vacuum_scale_factor = 0.01,  -- trigger at 1% dead
       autovacuum_vacuum_cost_delay  = 2);      -- more aggressive I/O
```

**`VACUUM FULL`:** Rewrites the table to a new file, reclaims disk space. Takes `ACCESS EXCLUSIVE` lock — table is offline. Use only for emergency bloat recovery; prefer `pg_repack` for online repack.

**`pg_stat_user_tables`:** Check `n_dead_tup` and `last_autovacuum` to detect bloat problems.

---

## Index Deep Dive

| Index Type | Use Case | Notes |
|---|---|---|
| B-tree | Equality, range, `ORDER BY` | Default; handles `<`, `>`, `BETWEEN`, `LIKE 'foo%'` |
| Hash | Equality only (`=`) | Faster than B-tree for pure equality; no range/sort |
| GIN | Arrays, JSONB, full-text | Inverted index; slow to build, fast multi-value query |
| GiST | Geometric types, ranges, full-text | Generalized search tree; extensible via operator classes |
| BRIN | Large sorted tables (time-series, append-only logs) | Stores min/max per block range; tiny, approximate |
| Covering (`INCLUDE`) | Avoid heap fetch for frequent queries | Extra columns in leaf nodes only, not search keys |

**Composite index column order:**
- Equality predicates first, range predicates last.
- `(status, created_at)` supports `WHERE status = 'active' AND created_at > '2024-01-01'` with full index use.
- Reversed order `(created_at, status)` forces a range scan on the leading column first — many leaf pages visited.

**Index-only scan:** If every column in the SELECT and WHERE is covered by the index, Postgres skips the heap fetch. Check visibility map — heap pages must be all-visible (VACUUM must have run).

```sql
-- Covering index: id and status in key; email in INCLUDE (leaf only)
CREATE INDEX ix_users_status ON users (status) INCLUDE (email);
-- SELECT email FROM users WHERE status = 'active' → index-only scan
```

---

## Partitioning

**Range partitioning** — most common, by date:
```sql
CREATE TABLE events (
    id BIGINT, created_at TIMESTAMPTZ, payload JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE events_2025 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```
Query planner prunes irrelevant partitions automatically. Old partitions can be detached and archived with zero downtime.

**Hash partitioning** — even distribution by ID:
```sql
CREATE TABLE orders PARTITION BY HASH (user_id);
CREATE TABLE orders_0 PARTITION OF orders FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- ... orders_1, orders_2, orders_3
```

**List partitioning** — by enumerable value:
```sql
CREATE TABLE tenants PARTITION BY LIST (region);
CREATE TABLE tenants_eu PARTITION OF tenants FOR VALUES IN ('de', 'fr', 'nl');
```
Useful for multi-tenant isolation — each tenant's data in its own partition (and potentially its own tablespace).

**Local vs global indexes:** A local index covers one partition; created/dropped with the partition. A global (non-partitioned) index covers all partitions — harder to maintain, but supports cross-partition uniqueness.

**SQL Server:** Uses `CREATE PARTITION FUNCTION` (define ranges) + `CREATE PARTITION SCHEME` (map ranges to filegroups) + `CREATE TABLE ... ON scheme(column)`.

---

## Replication

**Physical replication (streaming / WAL shipping):**
- Primary sends raw WAL bytes to standby. Standby replays them byte-for-byte.
- Replica is an exact copy — same Postgres major version required.
- Very fast; replica lag measured in milliseconds.
- Standby can serve read-only queries (`hot_standby = on`).

**Logical replication:**
- Decodes WAL into row-level change events (INSERT/UPDATE/DELETE on specific tables).
- Works across major Postgres versions (upgrade path).
- Allows subscribing to a subset of tables.
- Foundation for CDC tools (Debezium, pgoutput protocol).

**Monitoring replica lag:**
```sql
-- On primary
SELECT client_addr, state, replay_lag
FROM pg_stat_replication;
```

**Synchronous replication:** `synchronous_standby_names = 'replica1'`. Primary waits for replica WAL flush before committing. Zero data loss, but adds latency of one round-trip.

**Read replicas in practice:** Route heavy reporting queries to replicas. Application must tolerate eventual consistency — a replica might be 50–200ms behind. Use primary for anything requiring read-your-writes.

---

## Sharding

Sharding splits data across multiple independent database nodes (unlike partitioning, which stays in one server).

**Hash sharding:**
Consistent hashing maps `hash(key) mod N → shard`. Even distribution regardless of key distribution. Adding a shard with consistent hashing moves only `1/N` of keys.

**Range sharding:**
Ordered key ranges assigned to shards. Enables range scans on shard key without scatter-gather. Risk: hotspot if recent data is always inserted into the last shard (time-series).

**Cross-shard queries:**
A `JOIN` across shards requires scatter-gather: query all shards, merge results in application or a coordinator node. Expensive — design schemas to avoid cross-shard joins.

**Resharding pain:**
Moving rows between shards requires atomic double-write or a live migration strategy. Consistent hashing minimises movement (only `K/N` keys move when adding one shard to an N-shard cluster).

**Managed layers:**
- **Vitess** (MySQL): transparent sharding with a proxy layer; used by YouTube, Slack.
- **Citus** (Postgres): extension-based; shard key defined on `CREATE TABLE`; coordinator routes queries.

**Application concern:** Shard key must be chosen carefully — it dictates which queries are shard-local and which require scatter-gather. A bad shard key (e.g. low-cardinality enum) creates hotspots.

---

## Connection Pooling at Scale

Postgres process model: one OS process per connection. Memory cost ≈ 5–10 MB per connection. At 500 connections, that's 2.5–5 GB just for process overhead before any query runs.

**PgBouncer (transaction mode):**
- N app connections → M Postgres connections where M << N.
- A Postgres connection is leased only for the duration of a transaction.
- Limitation: prepared statements need `server_reset_query` handling; `SET` session variables don't persist across transactions.

```ini
# pgbouncer.ini
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25   # real Postgres connections per db/user pair
```

**SQL Server / ADO.NET:**
- Connection pool is per connection string (per process). Default `Max Pool Size = 100`.
- Monitor: `sys.dm_exec_connections`, `sys.dm_os_wait_stats` for `ASYNC_NETWORK_IO`.

**EF Core DbContext pooling:**
```csharp
// Reuses DbContext instances (not connections) — reduces GC pressure
builder.Services.AddDbContextPool<AppDbContext>(
    o => o.UseSqlServer(connStr),
    poolSize: 128);
```
Pooling DbContext means `OnConfiguring` runs once per pool slot, not per request. Avoid storing per-request state on a pooled DbContext.

---

## Redis Deep Dive

**Data structures and when to use them:**

| Structure | Command | Use case |
|---|---|---|
| String | `SET`/`GET` | Cache, counters (`INCR`), session tokens |
| Hash | `HSET`/`HGET` | Object fields (user profile, avoids serialize overhead) |
| List | `LPUSH`/`BRPOP` | Message queue, activity feed (ordered) |
| Set | `SADD`/`SMEMBERS` | Unique tags, online users |
| Sorted Set | `ZADD`/`ZRANGE` | Leaderboards, rate limiting windows |
| Stream | `XADD`/`XREAD` | Persistent log, consumer groups, event sourcing |
| HyperLogLog | `PFADD`/`PFCOUNT` | Approximate unique count (1% error, 12 KB max) |

**Persistence:**
- **RDB (snapshot):** Fork + write entire dataset to disk at intervals (`save 900 1`). Fast restart; may lose up to interval of data.
- **AOF (append-only file):** Every write command appended. `appendfsync everysec` = at most 1s data loss. Slower restart (must replay log). `AOF rewrite` compacts it.
- **Hybrid:** AOF + RDB; on restart use RDB as base then replay recent AOF delta.

**High availability:**
- **Sentinel:** 3+ Sentinel processes monitor primary; automatic failover promotes a replica. Single shard only.
- **Redis Cluster:** 16 384 hash slots split across N primaries; each primary has replicas. Built-in sharding + HA. Cross-slot operations require `{}` hash tags.

**Distributed lock:**
```
SET lockKey <uuid> EX 30 NX
```
- `NX` = only set if not exists. `EX 30` = expire in 30s (prevents deadlock if holder crashes).
- Release: check value == uuid (your lock), then DEL. Use Lua for atomicity:

```lua
-- Atomic check-and-delete
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Redlock:** Acquire lock on 5 independent Redis nodes; require quorum (3/5). Release on all 5. Controversial: Martin Kleppmann argues clock drift makes it unsafe for fencing tokens; use ZooKeeper/etcd for strong guarantees.

**Pub/Sub vs Streams:**
- Pub/Sub: fire-and-forget. Message lost if no subscriber connected. No persistence, no consumer groups.
- Streams: persistent log. `XADD` appends; `XREAD`/`XREADGROUP` consume with acknowledgement. Survives restarts. Use Streams for anything that cannot afford message loss.

---

## Elasticsearch Basics

**Inverted index:** For each analyzed term, Elasticsearch stores a posting list — the list of document IDs containing that term. Lookup is O(1) for a term regardless of corpus size.

```
"quick brown fox" → analyzed to: [quick, brown, fox]
term "quick" → [doc1, doc5, doc17, ...]
```

**Shards and replicas:**
- An index is split into N **primary shards** (set at creation, immutable).
- Each primary shard has R **replica shards** (can change at runtime).
- Reads served by any primary or replica; writes go to primary then replicated.

**Query vs filter:**
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "database" } }
      ],
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "published_at": { "gte": "2024-01-01" } } }
      ]
    }
  }
}
```
`filter` clauses are cached by Elasticsearch; they don't contribute to relevance score. Always put binary yes/no conditions in `filter`, text relevance in `must`/`should`.

**Mapping types:**
- `text`: tokenized and analyzed. Used for full-text search. Not aggregatable.
- `keyword`: stored as-is. Used for exact match, filtering, sorting, aggregations.
- Common pattern: dual-map a field as both `text` (search) and `keyword` (aggregate) via `fields`.

**Aggregations:**
- **Bucket:** group documents (`terms`, `date_histogram`, `range`).
- **Metric:** compute a value per bucket (`avg`, `min`, `max`, `cardinality`).
- **Pipeline:** operate on other aggregations (`moving_avg`, `cumulative_sum`).

---

## Key Interview Questions

1. **Why are random UUID primary keys bad for B-tree indexes?**
   They scatter inserts across all leaf pages, causing constant page splits and poor buffer cache utilisation — use sequential UUIDs or integer PKs.

2. **How does MVCC avoid read-write contention without locks?**
   Each transaction sees a snapshot of committed data at its start time; writers create new row versions rather than overwriting, so readers always find a consistent version without waiting.

3. **What is WAL and why must it be fsynced before a transaction commits?**
   WAL is a sequential log of every change; fsyncing it before ack guarantees the change survives a crash — the data page itself can stay in memory until later.

4. **Why does Postgres need VACUUM and what happens if it falls behind?**
   MVCC keeps old row versions for active snapshots; VACUUM removes dead tuples once no transaction can see them — if it falls behind, tables bloat, query plans degrade, and eventually transaction ID wraparound occurs.

5. **When would you choose Redis Streams over Pub/Sub?**
   Whenever message loss is unacceptable — Streams persist messages, support consumer groups with acknowledgement, and survive Redis restarts; Pub/Sub drops messages if no subscriber is connected at publish time.
