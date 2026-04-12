# EF Core Advanced

## What it is
Entity Framework Core is .NET's ORM. "Advanced" means going beyond basic CRUD — understanding the Change Tracker, performance patterns, and enterprise-grade features.

---

## AsNoTracking — Read-Only Performance

The Change Tracker monitors every loaded entity for mutations. On read-only queries it wastes CPU and memory.

```csharp
// BAD for reads — tracks every entity unnecessarily
var products = await _context.Products.ToListAsync();

// GOOD for reads — no Change Tracker overhead
var products = await _context.Products
    .AsNoTracking()
    .ToListAsync();
```

**Rule:** Any query that feeds a DTO/read model should use `.AsNoTracking()`.

---

## Bulk Operations (EF Core 7+)

Old pattern: fetch entities → loop → mutate → SaveChanges = N+1 SQL statements.

New pattern: single SQL statement, Change Tracker bypassed entirely.

```csharp
// OLD — loads all records into memory, then updates one by one
var products = await _context.Products.Where(p => p.CategoryId == id).ToListAsync();
foreach (var p in products) p.IsActive = false;
await _context.SaveChangesAsync();

// NEW — single UPDATE statement, never loads entities
await _context.Products
    .Where(p => p.CategoryId == id)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.IsActive, false));

// DELETE
await _context.Products
    .Where(p => p.IsActive == false)
    .ExecuteDeleteAsync();
```

**When NOT to use:** when you need domain logic to run (domain events, interceptors, Change Tracker hooks). Bulk operations bypass all of that.

---

## Change Tracker

Monitors entity states so EF knows what SQL to generate on `SaveChanges`.

| State | Meaning |
|---|---|
| `Added` | New entity, will INSERT |
| `Modified` | Existing entity with changes, will UPDATE |
| `Deleted` | Marked for removal, will DELETE |
| `Unchanged` | Loaded but not changed, no SQL |
| `Detached` | Not tracked at all |

```csharp
var entry = _context.Entry(order);
Console.WriteLine(entry.State); // EntityState.Unchanged

order.Ship();
Console.WriteLine(entry.State); // EntityState.Modified
```

---

## Value Converters

Translate C# types ↔ database column types.

```csharp
// Store an enum as a string instead of int (more readable in DB)
builder.Property(p => p.Status)
    .HasConversion<string>();

// Store a custom type as a string
builder.Property(p => p.Color)
    .HasConversion(
        color => color.ToHex(),        // C# → DB
        hex => Color.FromHex(hex)      // DB → C#
    );
```

---

## Interceptors

Hooks that run during EF Core operations. Don't modify entities in your code — let the interceptor handle cross-cutting concerns.

```csharp
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var entries = eventData.Context!.ChangeTracker.Entries<IAuditable>();
        foreach (var entry in entries)
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = DateTime.UtcNow;
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTime.UtcNow;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Register in Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString)
           .AddInterceptors(new AuditInterceptor()));
```

---

## JSON Columns (EF Core 7+)

Store semi-structured data as JSON in a relational column. Avoids the Entity-Attribute-Value (EAV) anti-pattern.

```csharp
// Entity
public class Product
{
    public Guid Id { get; set; }
    public Dictionary<string, string> Attributes { get; set; } = new(); // stored as JSON
}

// Configuration
builder.Property(p => p.Attributes).HasColumnType("nvarchar(max)");

// Or map a full owned type as JSON
builder.OwnsOne(p => p.Metadata, metadata =>
{
    metadata.ToJson(); // entire object stored as JSON column
});
```

**Use when:** data is dynamic/optional per product type (e.g., electronics have Wattage, clothing has Size). Avoids 50 nullable columns or EAV tables.

---

## Vector Search (EF Core 10 + pgvector / SQL Server)

Enables semantic (meaning-based) search inside the database — no separate vector store needed for simpler use cases.

```csharp
public class Product
{
    public Guid Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public Vector? NameEmbedding { get; set; } // stored as vector column
}

// Semantic similarity search
var embedding = await _embeddingService.GetEmbeddingAsync(searchTerm);
var results = await _context.Products
    .OrderBy(p => EF.Functions.VectorDistance("cosine", p.NameEmbedding, embedding))
    .Take(10)
    .ToListAsync();
```

**Use when:** building search features, RAG pipelines, or recommendation systems where exact text match is insufficient.

---

## LINQ Joins (EF Core 10)

```csharp
// Before EF10 — verbose SelectMany pattern
var result = context.Orders
    .GroupJoin(context.Users,
        o => o.UserId, u => u.Id,
        (o, users) => new { Order = o, Users = users })
    .SelectMany(x => x.Users.DefaultIfEmpty(),
        (x, u) => new { x.Order, User = u });

// EF Core 10 — native left join
var result = context.Orders
    .LeftJoin(context.Users,
        o => o.UserId,
        u => u.Id,
        (o, u) => new { Order = o, User = u });
```

---

## Value Objects as ComplexProperty (EF Core 8+)

Replaces the older `OwnsOne` / Owned Types pattern for Value Objects. More predictable, no shadow keys.

```csharp
// Configuration
builder.ComplexProperty(o => o.ShippingAddress, address =>
{
    address.Property(a => a.Street).HasMaxLength(200);
    address.Property(a => a.City).HasMaxLength(100);
});
```

Columns are stored inline in the parent table (same row). No separate table, no foreign key.

---

## DbContext Pooling

Reuses `DbContext` instances instead of creating a new one per request. Major throughput improvement under load.

```csharp
// BAD for high traffic
builder.Services.AddDbContext<AppDbContext>(...);

// GOOD for high traffic
builder.Services.AddDbContextPool<AppDbContext>(..., poolSize: 1024);
```

**Caveat:** don't store request-specific state on the DbContext (e.g., current user). Use scoped services instead.

---

## Optimistic Concurrency

Prevents lost updates when two users edit the same record simultaneously.

```csharp
public class Product
{
    public Guid Id { get; set; }
    [Timestamp]
    public byte[] RowVersion { get; set; } = []; // EF checks this on UPDATE
}
```

If User A and User B both load the same product, and User A saves first, User B's save throws `DbUpdateConcurrencyException` because the `RowVersion` no longer matches.

```csharp
try
{
    await _context.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    // Reload and retry, or return conflict error to client
    return Result.Failure(ErrorCodes.ConcurrencyConflict);
}
```

---

## Multi-Tenancy (Global Query Filters)

For SaaS apps where all tenants share the same database tables. Global Query Filters ensure every query automatically adds a `WHERE TenantId = @current`.

```csharp
public class AppDbContext : DbContext
{
    private readonly ITenantProvider _tenantProvider;

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Automatically appended to every query on this entity
        builder.Entity<Product>()
            .HasQueryFilter(p => p.TenantId == _tenantProvider.CurrentTenantId);
    }
}
```

Pair with an interceptor that stamps `TenantId` on `Added` entities automatically.

---

## Testcontainers (Modern Integration Testing)

`UseInMemoryDatabase` doesn't support real SQL features (transactions, indexes, constraints). Use Testcontainers to spin up a real SQL Server in Docker during tests.

```csharp
public class OrderIntegrationTests : IAsyncLifetime
{
    private readonly MsSqlContainer _sqlContainer = new MsSqlBuilder().Build();

    public async Task InitializeAsync() => await _sqlContainer.StartAsync();
    public async Task DisposeAsync() => await _sqlContainer.DisposeAsync();

    [Fact]
    public async Task CreateOrder_ShouldPersist()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(_sqlContainer.GetConnectionString())
            .Options;

        await using var context = new AppDbContext(options);
        await context.Database.MigrateAsync();
        // ... test against real SQL Server
    }
}
```

---

## Background Workers (IHostedService)

For long-running non-HTTP tasks: processing queues, sending emails, running scheduled jobs.

```csharp
public class OrderProcessingWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Process queued orders
            await ProcessPendingOrdersAsync();
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// Register
builder.Services.AddHostedService<OrderProcessingWorker>();
```

---

## Unit of Work — Dispatching Domain Events

Override `SaveChangesAsync` to dispatch domain events before or after committing.

```csharp
public class AppDbContext : DbContext
{
    private readonly IMediator _mediator;

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Collect domain events from all tracked aggregates
        var events = ChangeTracker.Entries<IHasDomainEvents>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        var result = await base.SaveChangesAsync(ct); // commit first

        foreach (var domainEvent in events)
            await _mediator.Publish(domainEvent, ct); // then dispatch

        return result;
    }
}
```

---

## Common Interview Questions

1. What is the Change Tracker and when would you disable it?
2. What is the difference between `ExecuteUpdateAsync` and loading + mutating entities?
3. When would you use bulk operations vs normal EF update?
4. What is an Interceptor? Give a real use case.
5. What is optimistic concurrency and how does EF Core implement it?
6. Why is `UseInMemoryDatabase` bad for integration tests?
7. What is a Global Query Filter and when would you use it?
8. What is DbContext Pooling and what's the trade-off?

---

## Common Mistakes

- Using `AddDbContext` instead of `AddDbContextPool` in high-traffic APIs
- Forgetting `.AsNoTracking()` on read-only queries
- Using `UseInMemoryDatabase` for integration tests (doesn't test real SQL)
- Storing request state (current user, tenant) directly on the DbContext when using pooling
- Using bulk operations when domain events or interceptors need to fire
- Not handling `DbUpdateConcurrencyException` when using `[Timestamp]`

---

## How It Connects

- Change Tracker + `SaveChangesAsync` override = where Domain Events get dispatched
- Interceptors handle cross-cutting concerns (audit, tenant stamping) so entities stay clean
- Value Objects map as `ComplexProperty` — no separate table, no shadow keys
- Testcontainers + real SQL = tests that actually catch query bugs
- Global Query Filters + ITenantProvider = multi-tenancy without polluting every query
- Bulk operations bypass domain logic — only use for admin/data scripts, not business operations

---

## My Confidence Level
- `[ ]` AsNoTracking — when and why
- `[ ]` Bulk operations (ExecuteUpdateAsync / ExecuteDeleteAsync)
- `[ ]` Change Tracker states
- `[ ]` Value Converters
- `[ ]` Interceptors (SaveChangesInterceptor)
- `[ ]` JSON Columns
- `[ ]` Vector Search
- `[ ]` ComplexProperty for Value Objects
- `[ ]` DbContext Pooling
- `[ ]` Optimistic Concurrency
- `[ ]` Multi-tenancy (Global Query Filters)
- `[ ]` Testcontainers
- `[ ]` Background Workers (IHostedService)
- `[ ]` Dispatching Domain Events in SaveChangesAsync

## My Notes
<!-- Personal notes -->
