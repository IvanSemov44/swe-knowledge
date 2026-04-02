# SQL Basics

## What it is
Structured Query Language — the standard language for managing and querying relational databases. Data is stored in tables (relations) with rows and columns.

## Why it matters
Almost every backend application persists data in a relational database. Slow queries are among the most common production performance problems. Understanding SQL deeply makes you a better EF Core developer too.

---

## Core SELECT

```sql
SELECT column1, column2
FROM table_name
WHERE condition
GROUP BY column
HAVING group_condition
ORDER BY column ASC|DESC
LIMIT 10 OFFSET 20;
```

**Execution order (not write order):**
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

---

## JOINs

### INNER JOIN
Returns rows that have matches **in both tables**.
```sql
SELECT o.Id, u.Name
FROM Orders o
INNER JOIN Users u ON o.UserId = u.Id;
```

### LEFT JOIN (LEFT OUTER JOIN)
All rows from left table + matching rows from right. Non-matches → NULL.
```sql
SELECT u.Name, o.Id
FROM Users u
LEFT JOIN Orders o ON o.UserId = u.Id;
-- Returns all users, including those with no orders
```

### RIGHT JOIN
All rows from right table + matching rows from left. Rarely used — swap table order and use LEFT JOIN instead.

### FULL OUTER JOIN
All rows from both tables. NULL where no match.

### CROSS JOIN
Cartesian product — every row in A with every row in B. Rarely intentional.

### SELF JOIN
Join a table to itself (e.g., employee → manager).
```sql
SELECT e.Name, m.Name AS Manager
FROM Employees e
LEFT JOIN Employees m ON e.ManagerId = m.Id;
```

---

## Aggregates

```sql
SELECT
    CategoryId,
    COUNT(*) AS TotalProducts,
    AVG(Price) AS AvgPrice,
    MAX(Price) AS MaxPrice,
    MIN(Price) AS MinPrice,
    SUM(Price) AS TotalValue
FROM Products
GROUP BY CategoryId
HAVING COUNT(*) > 5;   -- filter AFTER grouping (WHERE filters BEFORE)
ORDER BY TotalProducts DESC;
```

---

## Subqueries

```sql
-- Scalar subquery
SELECT Name, (SELECT COUNT(*) FROM Orders WHERE UserId = u.Id) AS OrderCount
FROM Users u;

-- IN subquery
SELECT * FROM Products
WHERE CategoryId IN (SELECT Id FROM Categories WHERE Name = 'Electronics');

-- EXISTS (often faster than IN for large datasets)
SELECT * FROM Users u
WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.UserId = u.Id);
```

---

## Window Functions

Perform calculations across a set of rows related to the current row, without collapsing them into groups.

```sql
SELECT
    Name,
    Department,
    Salary,
    RANK() OVER (PARTITION BY Department ORDER BY Salary DESC) AS SalaryRank,
    ROW_NUMBER() OVER (ORDER BY Salary DESC) AS GlobalRank,
    AVG(Salary) OVER (PARTITION BY Department) AS DeptAvgSalary,
    LAG(Salary) OVER (ORDER BY HireDate) AS PreviousSalary
FROM Employees;
```

**Key window functions:** `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `SUM()`, `AVG()`, `FIRST_VALUE()`, `LAST_VALUE()`

---

## Indexes

**What:** A data structure (usually B-tree) that enables the database to find rows without scanning the entire table.

**Without index:** Full table scan O(n)
**With B-tree index:** O(log n)

```sql
-- Create index
CREATE INDEX IX_Orders_UserId ON Orders(UserId);
CREATE UNIQUE INDEX UX_Users_Email ON Users(Email);

-- Composite index
CREATE INDEX IX_Orders_UserId_CreatedAt ON Orders(UserId, CreatedAt);
```

### When indexes help:
- Columns in WHERE clauses
- Columns in JOIN conditions
- Columns in ORDER BY (avoids sort)
- Foreign keys (always index them)

### When indexes hurt:
- High write volume (every INSERT/UPDATE/DELETE must update indexes)
- Low cardinality columns (e.g., `IsDeleted` boolean — not useful)
- Very small tables (full scan is faster)

### Index column order matters (composite index):
```sql
-- Index on (UserId, CreatedAt)
-- Fast: WHERE UserId = 5
-- Fast: WHERE UserId = 5 AND CreatedAt > '2024-01-01'
-- Slow: WHERE CreatedAt > '2024-01-01' -- can't skip leading column
```

---

## Transactions (ACID)

**Atomicity:** All operations in a transaction succeed or all fail. No partial commits.

**Consistency:** Transaction brings DB from one valid state to another. Constraints are respected.

**Isolation:** Concurrent transactions don't see each other's uncommitted changes (depends on isolation level).

**Durability:** Committed transaction survives system failure (written to disk).

```sql
BEGIN TRANSACTION;
    UPDATE Accounts SET Balance = Balance - 100 WHERE Id = 1;
    UPDATE Accounts SET Balance = Balance + 100 WHERE Id = 2;
COMMIT;
-- If either UPDATE fails, ROLLBACK undoes both
```

### Isolation Levels (most to least strict):
| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| SERIALIZABLE | No | No | No |
| REPEATABLE READ | No | No | Yes |
| READ COMMITTED | No | Yes | Yes |
| READ UNCOMMITTED | Yes | Yes | Yes |

SQL Server default: `READ COMMITTED`

---

## Normalization

Process of organizing data to reduce redundancy and improve integrity.

### 1NF — First Normal Form
- Each column has atomic (indivisible) values
- No repeating groups

### 2NF — Second Normal Form
- Must be in 1NF
- No partial dependencies (non-key columns depend on the whole primary key)

### 3NF — Third Normal Form
- Must be in 2NF
- No transitive dependencies (non-key columns depend only on the primary key, not on other non-key columns)

**Practical rule:** Every non-key column should depend on the key, the whole key, and nothing but the key.

**Over-normalization trade-off:** Highly normalized = many JOINs = potentially slower reads. Denormalization for read performance is sometimes intentional.

---

## N+1 Query Problem

A performance anti-pattern where you execute 1 query to get N records, then N additional queries to fetch related data — totaling N+1 queries.

**Example:**
```csharp
// N+1 in EF Core — DO NOT DO THIS
var orders = await _context.Orders.ToListAsync(); // 1 query
foreach (var order in orders)
{
    var items = await _context.OrderItems              // N queries
        .Where(i => i.OrderId == order.Id)
        .ToListAsync();
}
```

**Fix:**
```csharp
// 1 query with JOIN (eager loading)
var orders = await _context.Orders
    .Include(o => o.Items)
    .ToListAsync();

// Or explicit projection
var orders = await _context.Orders
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Items = o.Items.Select(i => new ItemDto { ... }).ToList()
    })
    .ToListAsync();
```

---

## Common SQL Patterns

### Pagination
```sql
-- Offset-based (simple but slow for large offsets)
SELECT * FROM Products ORDER BY Id OFFSET 40 ROWS FETCH NEXT 10 ROWS ONLY;

-- Keyset/cursor pagination (fast, use for large datasets)
SELECT * FROM Products WHERE Id > @lastId ORDER BY Id FETCH NEXT 10 ROWS ONLY;
```

### Upsert (SQL Server)
```sql
MERGE Products AS target
USING (SELECT @Id, @Name, @Price) AS source (Id, Name, Price)
ON target.Id = source.Id
WHEN MATCHED THEN UPDATE SET Name = source.Name, Price = source.Price
WHEN NOT MATCHED THEN INSERT (Id, Name, Price) VALUES (source.Id, source.Name, source.Price);
```

---

## Common Interview Questions

1. What is the difference between INNER JOIN and LEFT JOIN?
2. When would you use a subquery vs a JOIN?
3. What is an index and when would you create one?
4. What are the ACID properties?
5. What is the N+1 problem and how do you fix it?
6. What is the difference between WHERE and HAVING?
7. What is normalization and why does it matter?

---

## Common Mistakes

- Selecting `SELECT *` in production code — always specify columns
- Missing index on foreign key columns
- Not using `EXISTS` when checking for presence (vs `COUNT(*) > 0`)
- Updating without a WHERE clause (updates ALL rows)
- Relying on `ORDER BY` in a subquery or view (not guaranteed)
- Using `OFFSET` pagination at high offsets (scans all preceding rows)

---

## How It Connects

- EF Core generates SQL under the hood — understanding SQL helps you write better LINQ
- `AsNoTracking()` → hints EF Core not to change-track, improving read performance
- Database indexes are the SQL equivalent of HashMap lookup in algorithms
- Transactions map to `UnitOfWork.CommitAsync()` in Clean Architecture
- N+1 is solved by EF Core `.Include()` or query projections with `.Select()`

---

## My Confidence Level
- `[c]` SELECT, WHERE, ORDER BY, GROUP BY
- `[c]` JOINs
- `[b]` Indexes (when and why)
- `[b]` Transactions / ACID
- `[b]` Normalization
- `[~]` Window functions
- `[~]` N+1 problem

## My Notes
<!-- Personal notes -->
