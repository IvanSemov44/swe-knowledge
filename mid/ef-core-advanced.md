# EF Core Advanced

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: b -->

---

## N+1 Problem

The most common EF Core performance bug. Happens when you access a navigation property inside a loop without eager loading — EF fires a separate SQL query per iteration.

```csharp
// BAD — 1 query for orders + N queries for OrderItems (one per order)
var orders = await _context.Orders.ToListAsync();
foreach (var order in orders)
{
    var count = order.OrderItems.Count; // EF lazy loads here — separate DB hit
}
```

Called **N+1** because: 1 query for the list + N queries for related data = N+1 total.

**Rule:** Any time you access a navigation property inside a loop — did you `.Include()` it?

---

## Eager Loading with .Include()

Loads related entities upfront in a single JOIN query.

```csharp
var orders = await _context.Orders
    .Include(o => o.OrderItems)
    .ThenInclude(i => i.Product)   // nested include
    .ToListAsync();
```

**Use when:** Command handlers — you need the full aggregate to mutate and save back.

**Cost:** Loads entire entities (all columns) into memory, tracked by EF change tracker.

---

## Projections with .Select()

Only fetches the columns you specify. Faster, less memory, no change tracking.

```csharp
var dtos = await _context.Orders
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        TotalItems = o.OrderItems.Count,
        TotalPrice = o.OrderItems.Sum(i => i.Price)
    })
    .ToListAsync();
```

**Use when:** Query handlers — read-only, returning a DTO to the client.

### Include vs Select — the rule

| Scenario | Use |
|---|---|
| Command handler — need to mutate entity | `.Include()` |
| Query handler — returning a DTO | `.Select()` |
| Displaying a list | `.Select()` |
| Loading an aggregate root | `.Include()` |

Maps directly to CQRS: queries always project, commands always include.

---

## AsNoTracking

By default, EF tracks every loaded entity in the change tracker (memory overhead, slower).
For read-only queries, disable it.

```csharp
var orders = await _context.Orders
    .AsNoTracking()
    .Include(o => o.OrderItems)
    .ToListAsync();
```

**Rule:** Any query that won't call `SaveChangesAsync` should use `.AsNoTracking()`.
`.Select()` projections are automatically non-tracked (no entity = nothing to track).

---

## Owned Entities

Value Objects in DDD are mapped as owned entities in EF — they have no separate table by default, stored in the owner's table.

```csharp
// Core — Value Object
public class Address
{
    public string Street { get; }
    public string City { get; }
    // ...
}

// Core — Entity owns Address
public class Order
{
    public Address ShippingAddress { get; private set; }
}

// Infrastructure — EF config
builder.OwnsOne(o => o.ShippingAddress, a =>
{
    a.Property(x => x.Street).HasColumnName("ShippingStreet");
    a.Property(x => x.City).HasColumnName("ShippingCity");
});
```

---

## Value Converters

Map a domain type to a DB column type. Common use: storing enums as strings, Value Objects as single columns.

```csharp
builder.Property(o => o.Status)
    .HasConversion(
        v => v.ToString(),           // to DB
        v => Enum.Parse<OrderStatus>(v)  // from DB
    );
```

---

## Query Filters (Global Filters)

Applied automatically to every query for an entity. Good for soft deletes or multi-tenancy.

```csharp
// In DbContext OnModelCreating
builder.HasQueryFilter(o => !o.IsDeleted);

// To bypass the filter when needed
_context.Orders.IgnoreQueryFilters().ToListAsync();
```

---

## Interceptors

Hook into EF pipeline — runs before/after queries, saves, connections. Use for audit logging, soft-delete automation.

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(
        DbContextEventData eventData, InterceptionResult<int> result)
    {
        // stamp UpdatedAt on every modified entity
        foreach (var entry in eventData.Context.ChangeTracker.Entries())
        {
            if (entry.State == EntityState.Modified)
                entry.Property("UpdatedAt").CurrentValue = DateTime.UtcNow;
        }
        return base.SavingChanges(eventData, result);
    }
}
```

---

## Gaps to Fill Next

- [ ] Compiled queries (for hot paths)
- [ ] Raw SQL with `FromSqlRaw` and when to use it
- [ ] Connection resiliency (EnableRetryOnFailure)
- [ ] Bulk operations (EF Core 8 ExecuteUpdate / ExecuteDelete)

---

## Connected To

- `mid/ddd.md` — Owned entities map Value Objects; Aggregate roots map to repositories
- `mid/cqrs.md` — Query handlers use `.Select()` + `.AsNoTracking()`; Command handlers use `.Include()`
- `junior/sql-basics.md` — N+1 is a SQL problem, not just an EF problem
