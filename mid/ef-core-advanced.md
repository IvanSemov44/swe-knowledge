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

## Bulk Operations (EF Core 7+)

Single SQL statement, Change Tracker bypassed entirely.

```csharp
// Single UPDATE — never loads entities into memory
await _context.Products
    .Where(p => p.CategoryId == id)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false));

// Single DELETE
await _context.Products
    .Where(p => p.IsActive == false)
    .ExecuteDeleteAsync();
```

**When NOT to use:** when domain events, interceptors, or business rules need to fire. Bulk ops skip all of that.

---

## Change Tracker States

| State | Meaning |
|---|---|
| `Added` | New entity, will INSERT |
| `Modified` | Existing entity with changes, will UPDATE |
| `Deleted` | Marked for removal, will DELETE |
| `Unchanged` | Loaded but not changed, no SQL |
| `Detached` | Not tracked at all |

---

## ComplexProperty for Value Objects (EF Core 8+)

Replaces `OwnsOne` — no shadow keys, more predictable.

```csharp
builder.ComplexProperty(o => o.ShippingAddress, address =>
{
    address.Property(a => a.Street).HasMaxLength(200);
    address.Property(a => a.City).HasMaxLength(100);
});
```

---

## DbContext Pooling

Reuses `DbContext` instances instead of creating a new one per request.

```csharp
// High traffic — use pooling
builder.Services.AddDbContextPool<AppDbContext>(..., poolSize: 1024);
```

**Caveat:** don't store request-specific state on the DbContext when using pooling.

---

## Optimistic Concurrency

Prevents lost updates when two users edit the same record simultaneously.

```csharp
public class Product
{
    [Timestamp]
    public byte[] RowVersion { get; set; } = [];
}

try
{
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    return Result.Failure(ErrorCodes.ConcurrencyConflict);
}
```

---

## Testcontainers (Modern Integration Testing)

`UseInMemoryDatabase` doesn't support real SQL features. Use Testcontainers for a real SQL Server in Docker.

```csharp
public class OrderTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sql = new MsSqlBuilder().Build();

    public async Task InitializeAsync() => await _sql.StartAsync();
    public async Task DisposeAsync() => await _sql.DisposeAsync();

    [Fact]
    public async Task ShouldPersistOrder()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sql.GetConnectionString()).Options;
        await using var ctx = new AppDbContext(options);
        await ctx.Database.MigrateAsync();
        // real SQL tests here
    }
}
```

---

## Background Workers (IHostedService)

For long-running non-HTTP tasks.

```csharp
public class OutboxWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await ProcessOutboxAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(5), ct);
        }
    }
}

builder.Services.AddHostedService<OutboxWorker>();
```

---

## Dispatching Domain Events in SaveChangesAsync

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var events = ChangeTracker.Entries<AggregateRoot>()
        .SelectMany(e => e.Entity.DomainEvents)
        .ToList();

    var result = await base.SaveChangesAsync(ct);

    foreach (var domainEvent in events)
        await _mediator.Publish(domainEvent, ct);

    return result;
}
```

---

## Common Interview Questions

1. What is the N+1 problem and how do you fix it?
2. When would you use `.Include()` vs `.Select()`?
3. What is the Change Tracker and when would you disable it?
4. What is the difference between `ExecuteUpdateAsync` and loading + mutating entities?
5. What is an Interceptor? Give a real use case.
6. What is optimistic concurrency and how does EF Core implement it?
7. Why is `UseInMemoryDatabase` bad for integration tests?
8. What is a Global Query Filter and when would you use it?

---

## Common Mistakes

- Accessing navigation properties in a loop without `.Include()` (N+1)
- Forgetting `.AsNoTracking()` on read-only queries
- Using `UseInMemoryDatabase` for integration tests
- Using bulk operations when domain events or interceptors need to fire
- Not handling `DbUpdateConcurrencyException` when using `[Timestamp]`
- Using `AddDbContext` instead of `AddDbContextPool` in high-traffic APIs

---

## How It Connects

- N+1 → always `.Include()` in command handlers, `.Select()` in query handlers (CQRS)
- Change Tracker + `SaveChangesAsync` override = where Domain Events get dispatched
- Interceptors handle cross-cutting concerns (audit, tenant stamping)
- Testcontainers + real SQL = tests that actually catch query bugs
- Global Query Filters + ITenantProvider = multi-tenancy without polluting every query
- Bulk operations bypass domain logic — only use for admin/data scripts

---

## My Confidence Level
- `[b]` N+1 problem and .Include() fix
- `[b]` Eager loading vs projections — when to use each
- `[b]` AsNoTracking — when and why
- `[b]` Owned entities / ComplexProperty (Value Objects)
- `[b]` Value Converters
- `[c]` Bulk operations (ExecuteUpdateAsync / ExecuteDeleteAsync)
- `[b]` Change Tracker states
- `[~]` Interceptors vs SaveChangesAsync override — when to choose
- `[b]` Global Query Filters — gap: IgnoreQueryFilters()
- `[b]` DbContext Pooling
- `[b]` Optimistic Concurrency (RowVersion)
- `[b]` Testcontainers
- `[b]` Background Workers (IHostedService)
- `[b]` Dispatching Domain Events in SaveChangesAsync
- `[ ]` Vector Search (EF Core 10)
- `[ ]` Compiled queries / Raw SQL / Connection resiliency

## My Notes
<!-- Personal notes -->
