# SQL Advanced — CTEs, Indexes Deep Dive, Execution Plans

## What it is
SQL topics beyond the basics: CTEs for complex queries, clustered vs non-clustered indexes, execution plans for diagnosis, deadlocks, and connection pooling. Directly applicable to EF Core performance tuning.

## Why it matters
SQL questions in interviews go deeper than JOINs. CTEs and window functions appear in almost every SQL round. Index type knowledge separates candidates who just use EF Core from those who understand what's happening underneath.

---

## CTEs — Common Table Expressions

A named temporary result set defined with `WITH`. Makes complex queries readable and reusable within the query.

```sql
-- Basic CTE — cleaner than a subquery
WITH RecentOrders AS (
    SELECT UserId, COUNT(*) AS OrderCount, SUM(Total) AS TotalSpent
    FROM Orders
    WHERE CreatedAt >= DATEADD(MONTH, -3, GETDATE())
    GROUP BY UserId
)
SELECT
    u.Name,
    u.Email,
    ro.OrderCount,
    ro.TotalSpent
FROM Users u
INNER JOIN RecentOrders ro ON ro.UserId = u.Id
WHERE ro.TotalSpent > 500
ORDER BY ro.TotalSpent DESC;
```

**Multiple CTEs in one query:**
```sql
WITH
ActiveProducts AS (
    SELECT Id, Name, Price, CategoryId
    FROM Products
    WHERE IsActive = 1
),
CategoryStats AS (
    SELECT CategoryId, AVG(Price) AS AvgPrice, COUNT(*) AS ProductCount
    FROM ActiveProducts
    GROUP BY CategoryId
)
SELECT
    ap.Name,
    ap.Price,
    cs.AvgPrice,
    CASE WHEN ap.Price > cs.AvgPrice THEN 'Above avg' ELSE 'Below avg' END AS PricePosition
FROM ActiveProducts ap
INNER JOIN CategoryStats cs ON cs.CategoryId = ap.CategoryId;
```

**CTE vs Subquery vs Temp Table:**

| | CTE | Subquery | Temp Table (#table) |
|---|---|---|---|
| Readable | ✅ Named, reusable in same query | ❌ Nested, hard to follow | ✅ Named |
| Reusable in same query | ✅ Reference multiple times | ❌ Must repeat | ✅ |
| Persists across queries | ❌ Gone after query | ❌ | ✅ (within session) |
| Indexable | ❌ | ❌ | ✅ Can add indexes |
| Best for | Complex logic, readability | Simple one-use filter | Multi-step ETL, large intermediate sets |

---

## Recursive CTEs

A CTE that references itself. Used for hierarchical or tree-structured data.

```sql
-- Category hierarchy: Electronics → Phones → Smartphones
WITH CategoryTree AS (
    -- Anchor: start from root categories (no parent)
    SELECT Id, Name, ParentId, 0 AS Level, CAST(Name AS VARCHAR(500)) AS Path
    FROM Categories
    WHERE ParentId IS NULL

    UNION ALL

    -- Recursive: join each node to its children
    SELECT c.Id, c.Name, c.ParentId, ct.Level + 1,
           CAST(ct.Path + ' > ' + c.Name AS VARCHAR(500))
    FROM Categories c
    INNER JOIN CategoryTree ct ON ct.Id = c.ParentId
)
SELECT * FROM CategoryTree ORDER BY Path;
-- Result:
-- Level 0: Electronics         | Path: Electronics
-- Level 1: Phones              | Path: Electronics > Phones
-- Level 2: Smartphones         | Path: Electronics > Phones > Smartphones
```

**Real use cases:** Category trees, org charts, bill of materials, folder structures.

```sql
-- Find all reports of a manager (recursive org chart)
WITH Reports AS (
    SELECT Id, Name, ManagerId, 0 AS Depth
    FROM Employees
    WHERE Id = @ManagerId  -- starting point

    UNION ALL

    SELECT e.Id, e.Name, e.ManagerId, r.Depth + 1
    FROM Employees e
    INNER JOIN Reports r ON r.Id = e.ManagerId
)
SELECT * FROM Reports;
```

**Protection against infinite loops:**
```sql
-- Add MAXRECURSION option (default is 100, 0 = unlimited)
OPTION (MAXRECURSION 500)
```

---

## Execution Plans

The database's plan for executing your query. Reading execution plans tells you exactly WHY a query is slow.

### How to Get an Execution Plan

```sql
-- SQL Server Management Studio: press Ctrl+M before running, or use button
-- Or in T-SQL:
SET STATISTICS IO ON;   -- shows logical reads
SET STATISTICS TIME ON; -- shows CPU and elapsed time

SELECT * FROM Orders WHERE UserId = '123e4567-e89b-12d3-a456-426614174000';
-- Output: Table 'Orders'. Scan count 1, logical reads 5432  ← BAD if high
-- With index: logical reads 3  ← GOOD
```

### What to Look For

**Table Scan vs Index Seek:**
```
Table Scan = "I checked every row" → usually bad, means no index used
Index Seek = "I jumped directly to the rows I need" → good
Index Scan = "I scanned the whole index" → better than table scan but may indicate missing covering index
```

**High cost operators:**
- Table Scan on large tables → add an index on the WHERE/JOIN column
- Sort → add an index that provides the ORDER BY order
- Hash Match (for joins) → consider an index on the join column
- Key Lookup → the index found the row but had to go back to the table for extra columns → add a **covering index**

---

## Index Types Deep Dive

### Clustered vs Non-Clustered

```sql
-- Clustered index: the physical order of rows IN the table = the index order
-- One per table. Usually the primary key.
CREATE CLUSTERED INDEX IX_Orders_Id ON Orders(Id);
-- The rows in the data pages ARE sorted by Id. Seeking by Id is very fast.

-- Non-clustered index: a separate B-tree structure with pointers back to rows
CREATE NONCLUSTERED INDEX IX_Orders_UserId ON Orders(UserId);
-- A separate structure sorted by UserId, each entry has a pointer to the actual row.
-- Finding orders by UserId: seek index → get row pointer → fetch row (1 extra hop)
```

| | Clustered | Non-Clustered |
|---|---|---|
| Physical storage | IS the table's row order | Separate structure |
| Count per table | 1 | Up to 999 |
| Lookup cost | Direct — row IS there | Pointer + extra fetch |
| Best for | Primary key, most queried column | Foreign keys, WHERE/JOIN columns |

### Covering Index

Includes all columns the query needs. Eliminates the pointer-chase back to the table (Key Lookup).

```sql
-- Query: find orders for a user, need OrderId + Total + Status
SELECT Id, Total, Status FROM Orders WHERE UserId = @UserId;

-- Non-covering index — finds the rows but then fetches back to table for Total and Status
CREATE INDEX IX_Orders_UserId ON Orders(UserId);

-- Covering index — all needed columns are in the index, no table fetch needed
CREATE INDEX IX_Orders_UserId_Covering
ON Orders(UserId)
INCLUDE (Total, Status); -- INCLUDE adds columns to leaf level without sorting by them
```

### Composite Index Column Order

The **most selective**, most commonly filtered column goes first.

```sql
-- Query: WHERE UserId = @id AND Status = 'Pending'
-- Index on (UserId, Status) ← UserId first: can seek by UserId, then filter Status
-- Index on (Status, UserId) ← only useful if you filter by Status alone

-- Rule: (most selective / most frequently used alone) → (secondary filters)
CREATE INDEX IX_Orders_UserId_Status ON Orders(UserId, Status);
-- Can serve: WHERE UserId = x
-- Can serve: WHERE UserId = x AND Status = y
-- CANNOT efficiently serve: WHERE Status = y alone
```

### Filtered Index

An index on a subset of rows. Very efficient for sparse conditions.

```sql
-- Only 5% of orders are Pending — index just those rows
CREATE INDEX IX_Orders_Pending
ON Orders(CreatedAt)
WHERE Status = 'Pending';
-- Much smaller, faster to update, optimal for "get pending orders by date"
```

---

## Deadlocks

Occur when two transactions each hold a lock the other needs.

```
Transaction A: locks Order row 1, wants to lock OrderItem row 5
Transaction B: locks OrderItem row 5, wants to lock Order row 1
→ DEADLOCK — SQL Server kills one transaction (the "victim") with error 1205
```

### How to Prevent Deadlocks

**1. Consistent lock order** — always access tables in the same order across transactions:
```sql
-- All transactions: always UPDATE Orders before OrderItems
BEGIN TRAN
    UPDATE Orders SET ... WHERE Id = @orderId
    UPDATE OrderItems SET ... WHERE OrderId = @orderId
COMMIT
-- (not sometimes in the other order)
```

**2. Keep transactions short** — don't hold locks longer than needed. Never do user-facing work inside a transaction.

**3. Use NOLOCK sparingly** — reads dirty (uncommitted) data. Only for non-critical reads:
```sql
SELECT COUNT(*) FROM Orders WITH (NOLOCK); -- approximate count, may be slightly off
```

**4. Optimize indexes** — fewer rows scanned = fewer lock conflicts.

**5. Use READ_COMMITTED_SNAPSHOT isolation** (SQL Server) — readers don't block writers:
```sql
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON;
-- Now SELECT doesn't take shared locks → far fewer deadlocks
```

---

## Connection Pooling

Creating a `SqlConnection` or `DbContext` is expensive. .NET reuses connections from a pool.

```
App → request connection → pool has one available → reuse existing connection (fast)
App → request connection → pool empty → create new TCP connection to DB (slow, ~5ms)
App finishes → return connection to pool (not actually closed, ready for next request)
```

### In ASP.NET Core

`DbContext` is **Scoped** by default — one per HTTP request. This is correct. Each request gets one context, which holds one connection from the pool.

```csharp
// Program.cs — correct, one DbContext per request
services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlServer(connectionString));
// Default lifetime = Scoped

// WRONG — Singleton DbContext
services.AddSingleton<AppDbContext>(...); // shares state across requests → bugs
```

**DbContext Pooling** (higher throughput — reuses DbContext instances themselves):
```csharp
services.AddDbContextPool<AppDbContext>(opts =>
    opts.UseSqlServer(connectionString),
    poolSize: 128); // pool of 128 DbContext instances
// Clears change tracker on return to pool
```

**Connection string pool settings:**
```
Server=.;Database=MyDb;Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;
```

---

## Common SQL Patterns (Advanced)

### Soft Delete with Filtered Index

```sql
-- Soft delete: don't actually delete rows, just mark them
ALTER TABLE Products ADD IsDeleted BIT NOT NULL DEFAULT 0, DeletedAt DATETIME NULL;

-- Filtered index on active products only — keeps index small
CREATE UNIQUE INDEX UX_Products_Name_Active
ON Products(Name)
WHERE IsDeleted = 0;

-- EF Core: Global Query Filter to always exclude deleted
modelBuilder.Entity<Product>()
    .HasQueryFilter(p => !p.IsDeleted);
-- Now every query automatically filters: WHERE IsDeleted = 0
// Use IgnoreQueryFilters() when you explicitly need deleted records
```

### Audit Trail

```sql
-- Add audit columns to every important table
ALTER TABLE Orders ADD
    CreatedAt DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    CreatedBy NVARCHAR(100) NOT NULL DEFAULT '',
    UpdatedAt DATETIME2 NULL,
    UpdatedBy NVARCHAR(100) NULL;

-- In EF Core: set automatically via SaveChangesInterceptor
```

---

## Common Interview Questions

1. What is a CTE and how is it different from a subquery?
2. When would you use a recursive CTE?
3. What is the difference between a clustered and non-clustered index?
4. What is a covering index?
5. Why does column order matter in a composite index?
6. What is a deadlock? How do you prevent one?
7. What is connection pooling? How does it relate to DbContext lifetime?
8. What is `READ_COMMITTED_SNAPSHOT` and why might you enable it?

---

## Common Mistakes

- Putting columns in the wrong order in a composite index (most selective first)
- Using `SELECT *` which prevents covering indexes from working
- Not indexing foreign key columns (every FK should have an index)
- Starting a transaction, doing user input/IO inside it, then committing (holds lock too long)
- Using `NOLOCK` everywhere as a performance fix (dirty reads)
- Using a Singleton `DbContext` in ASP.NET Core

---

## How It Connects

- CTEs make complex LINQ-to-SQL queries more readable when written in raw SQL
- EF Core's `AsNoTracking()` reduces connection pressure from change tracking
- `DbContext` pooling pairs with connection pooling for high-throughput APIs
- Soft delete via EF Core global query filters avoids polluting every query with `WHERE IsDeleted = 0`
- Execution plans explain why EF Core queries are sometimes slow despite looking correct
- Deadlock prevention = keeping UnitOfWork transactions short — commit right after the work, not after HTTP response

---

## My Confidence Level
- `[ ]` CTEs — basic usage and readability advantage over subqueries
- `[ ]` Recursive CTEs — hierarchical data traversal
- `[ ]` Execution plans — table scan vs index seek, key lookup
- `[ ]` Clustered vs non-clustered indexes — physical structure difference
- `[ ]` Covering indexes — INCLUDE columns, eliminating key lookups
- `[ ]` Composite index column order — selectivity rule
- `[ ]` Filtered indexes — sparse condition optimization
- `[ ]` Deadlocks — causes, detection, prevention strategies
- `[ ]` Connection pooling — how it works, DbContext lifetime implications
- `[ ]` Soft delete with global query filters

## My Notes
<!-- Personal notes -->
