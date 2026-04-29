# Modular Monolith

## What it is
A single deployable application split into isolated modules that communicate via defined contracts — not via direct code references or shared database tables. Each module is a bounded context with its own domain, application, and infrastructure layers.

**It's the middle ground:**
```
Monolith (everything coupled) → Modular Monolith (isolated modules) → Microservices (separate deployments)
```

You can evolve a modular monolith into microservices by extracting modules — because they're already isolated.

---

## Your Solution Structure

```
ECommerce.sln
│
├── src/
│   ├── Shared.Kernel/                 ← AggregateRoot, Entity, ValueObject, Result<T>
│   │
│   ├── Modules/
│   │   ├── Catalog/
│   │   │   ├── Catalog.Domain/        ← Product, Category, Review aggregates
│   │   │   ├── Catalog.Application/   ← Commands, Queries, Handlers, IProductRepository
│   │   │   ├── Catalog.Infrastructure/← CatalogDbContext, ProductRepository, EF configs
│   │   │   └── Catalog.Tests/
│   │   │
│   │   ├── Orders/
│   │   │   ├── Orders.Domain/
│   │   │   ├── Orders.Application/
│   │   │   ├── Orders.Infrastructure/
│   │   │   └── Orders.Tests/
│   │   │
│   │   ├── Identity/
│   │   ├── Promotions/
│   │   └── Reviews/                   ← if separate from Catalog
│   │
│   └── Api/                           ← Controllers, Startup, Program.cs (thin)
│
└── tests/
    └── Integration.Tests/             ← cross-module integration tests
```

---

## Separate DbContext Per Module

Each module owns its own `DbContext`. They share the same physical database but use different schemas.

```csharp
// Catalog.Infrastructure
public class CatalogDbContext : DbContext
{
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.HasDefaultSchema("catalog"); // all tables: catalog.Products, catalog.Categories

        builder.ApplyConfigurationsFromAssembly(typeof(CatalogDbContext).Assembly);
    }
}

// Orders.Infrastructure
public class OrdersDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.HasDefaultSchema("orders"); // orders.Orders, orders.OrderItems
    }
}
```

**Result in the database:**
```sql
catalog.Products
catalog.Categories
orders.Orders
orders.OrderItems
identity.Users
promotions.PromoCodes
```

Same database, isolated schemas. When you split to microservices: each module gets its own database server.

---

## Repository in a BC

Each BC has its own repository interface (in Application) and implementation (in Infrastructure). The repository only touches that BC's DbContext — never another module's.

```csharp
// Orders.Application — interface lives in Application layer
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<Order>> GetByUserIdAsync(Guid userId, CancellationToken ct = default);
    void Add(Order order);
    void Update(Order order);
}

// Orders.Infrastructure — implementation lives in Infrastructure layer
public class OrderRepository : IOrderRepository
{
    private readonly OrdersDbContext _context;

    public OrderRepository(OrdersDbContext context) => _context = context;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => _context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public void Add(Order order) => _context.Orders.Add(order);
    public void Update(Order order) => _context.Orders.Update(order);
}
```

**Rules:**
- `OrderRepository` touches ONLY `OrdersDbContext` — never `CatalogDbContext`
- No `SaveChangesAsync` inside the repository — that's `IUnitOfWork`'s job
- One repository per aggregate root — `OrderRepository` for `Order`, `ProductRepository` for `Product` in Catalog

---

## BC Configuration (Full Module Wiring)

Each BC self-registers everything it needs. The API project just calls the extension method.

```csharp
// Orders.Infrastructure/OrdersModule.cs
public static class OrdersModule
{
    public static IServiceCollection AddOrdersModule(
        this IServiceCollection services,
        IConfiguration config)
    {
        // 1. DbContext with schema
        services.AddDbContext<OrdersDbContext>(opts =>
            opts.UseSqlServer(config.GetConnectionString("Default")));

        // 2. Repositories + UnitOfWork
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        // 3. MediatR handlers (commands, queries, domain event handlers)
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));

        // 4. MassTransit consumers (integration event handlers for this BC)
        services.AddMassTransit(x =>
        {
            x.AddConsumer<StockReservedConsumer>();
            x.AddConsumer<StockFailedConsumer>();

            x.UsingRabbitMq((ctx, cfg) =>
            {
                cfg.Host(config["RabbitMQ:Host"], h =>
                {
                    h.Username(config["RabbitMQ:Username"]);
                    h.Password(config["RabbitMQ:Password"]);
                });

                // Retry + DLQ routing
                cfg.ReceiveEndpoint("orders.stock-result", e =>
                {
                    e.ConfigureConsumer<StockReservedConsumer>(ctx);
                    e.ConfigureConsumer<StockFailedConsumer>(ctx);
                    e.UseMessageRetry(r => r.Intervals(
                        TimeSpan.FromSeconds(5),
                        TimeSpan.FromSeconds(30),
                        TimeSpan.FromMinutes(5)));
                });
            });
        });

        // 5. Background worker (Outbox dispatcher)
        services.AddHostedService<OutboxWorker>();

        return services;
    }
}

// Api/Program.cs — just wires the modules, contains no logic
builder.Services.AddOrdersModule(builder.Configuration);
builder.Services.AddCatalogModule(builder.Configuration);
builder.Services.AddIdentityModule(builder.Configuration);
```

**What belongs in configuration:**
| Item | Where registered |
|---|---|
| DbContext | `AddDbContext<>` in module |
| Repositories | `AddScoped<IRepo, Impl>` in module |
| MediatR handlers | `AddMediatR(assembly)` in module |
| MassTransit consumers | `AddConsumer<>` + `ReceiveEndpoint` in module |
| Background workers | `AddHostedService<>` in module |
| Outbox/Inbox tables | Part of module's DbContext |

---

## No Cross-Module EF Navigation Properties

The hard rule: **no EF navigation properties crossing module boundaries.**

```csharp
// ❌ WRONG — Orders module directly references Catalog entity
public class OrderItem
{
    public Guid ProductId { get; set; }
    public Product Product { get; set; } // ← EF navigation to Catalog.Product
}

// ✅ CORRECT — Orders only knows the ID + snapshot data
public class OrderItem : Entity
{
    public Guid ProductId { get; private set; }    // ID only — no navigation
    public string ProductName { get; private set; } // snapshot at time of order
    public decimal UnitPrice { get; private set; }  // snapshot — immutable history
    public int Quantity { get; private set; }
}
```

---

## How Modules Get Data From Each Other

**In-process (same deployment) — 3 options:**

**Option 1: Public Module API (recommended)**
Each module exposes a public read interface. Other modules use it.

```csharp
// Catalog.Application — public contract
public interface ICatalogModule
{
    Task<ProductReadModel?> GetProductAsync(Guid productId, CancellationToken ct);
    Task<bool> ProductExistsAsync(Guid productId, CancellationToken ct);
}

// Catalog.Infrastructure — implementation
public class CatalogModule : ICatalogModule
{
    private readonly CatalogDbContext _context;

    public async Task<ProductReadModel?> GetProductAsync(Guid productId, CancellationToken ct)
        => await _context.Products
            .AsNoTracking()
            .Where(p => p.Id == productId)
            .Select(p => new ProductReadModel(p.Id, p.Name, p.Price.Amount))
            .FirstOrDefaultAsync(ct);
}

// Orders.Application handler — uses ICatalogModule
public class PlaceOrderCommandHandler
{
    private readonly ICatalogModule _catalog;

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var product = await _catalog.GetProductAsync(cmd.ProductId, ct);
        if (product is null) return Result.Failure<Guid>(ErrorCodes.Orders.ProductNotFound);

        var order = Order.Create(cmd.UserId);
        order.AddItem(cmd.ProductId, product.Name, product.Price, cmd.Quantity);

        // ...
    }
}
```

**Option 2: MediatR cross-module query**
Orders sends a query through MediatR, Catalog handles it. Looser coupling than direct reference.

**Option 3: Integration Event (async)**
Orders doesn't need data synchronously — it reacts to events from other modules.

---

## Module Registration in Program.cs

Each module registers its own services:

```csharp
// Catalog.Infrastructure
public static class CatalogModule
{
    public static IServiceCollection AddCatalogModule(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddDbContext<CatalogDbContext>(opts =>
            opts.UseSqlServer(config.GetConnectionString("Default")));

        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<ICatalogModule, CatalogModuleService>();
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(CreateProductCommand).Assembly));

        return services;
    }
}

// Orders.Infrastructure
public static class OrdersModule
{
    public static IServiceCollection AddOrdersModule(
        this IServiceCollection services,
        IConfiguration config)
    {
        services.AddDbContext<OrdersDbContext>(opts =>
            opts.UseSqlServer(config.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));

        return services;
    }
}

// Api/Program.cs — thin, just wires modules
builder.Services.AddCatalogModule(builder.Configuration);
builder.Services.AddOrdersModule(builder.Configuration);
builder.Services.AddIdentityModule(builder.Configuration);
builder.Services.AddPromotionsModule(builder.Configuration);
```

---

## Database Migrations Per Module

Each DbContext has its own migrations folder:

```bash
# Catalog migrations
dotnet ef migrations add InitialCatalog \
  --project Catalog.Infrastructure \
  --startup-project Api \
  --context CatalogDbContext \
  --output-dir Migrations

# Orders migrations
dotnet ef migrations add InitialOrders \
  --project Orders.Infrastructure \
  --startup-project Api \
  --context OrdersDbContext \
  --output-dir Migrations
```

Apply all at startup:
```csharp
// Program.cs
app.Services.GetRequiredService<CatalogDbContext>().Database.Migrate();
app.Services.GetRequiredService<OrdersDbContext>().Database.Migrate();
```

---

## Communication Rules

| Situation | Method |
|---|---|
| Module needs data synchronously (e.g., validate product exists) | Public Module API (ICatalogModule) |
| Module reacts to something that happened in another module | Integration Event (RabbitMQ) |
| Module needs to trigger work in another module | Integration Event or direct API call |
| Two aggregates in the SAME module need to coordinate | Domain Event (MediatR in-process) |

---

## Enforcing Module Boundaries

Add a linting rule or code review convention:

```
Catalog.Domain → cannot reference Orders.*, Identity.*, etc.
Catalog.Application → can reference Catalog.Domain + Shared.Kernel only
Catalog.Infrastructure → can reference Catalog.Application + EF Core
Orders.Application → can reference ICatalogModule (interface), NOT CatalogDbContext
```

In C#, enforce this by only adding the right project references in `.csproj` files.

---

## When to Split to Microservices

You extract a module to a separate service when:
- That module needs to scale independently (Catalog gets 10x more traffic than Orders)
- Different deployment cadence (Promotions deploys 10x/day, Orders 1x/week)
- Different team owns it
- Compliance requires data isolation (Identity/PII in separate DB)

The modular monolith makes this extraction safe — the module's boundary is already clean.

---

## Common Interview Questions

1. What is a modular monolith and how does it differ from a regular monolith?
2. Why have separate DbContexts per module if they use the same database?
3. How do modules communicate without creating coupling?
4. What is a schema-per-module approach?
5. How does a modular monolith prepare you for microservices?

---

## Common Mistakes

- Modules referencing each other's DbContexts directly (creates tight coupling)
- EF navigation properties crossing module boundaries
- Sharing a single DbContext across all modules (loses isolation)
- Putting too much in Shared.Kernel (only primitives belong there)
- Cross-module joins in SQL (query one module's data via another module's DbContext)

---

## How It Connects

- Shared.Kernel provides base classes every module uses
- Each module has its own DbContext → separate migrations → separate schema
- MassTransit + RabbitMQ handles cross-module async communication
- CQRS: commands mutate within a module, queries can read cross-module via ICatalogModule
- When splitting to microservices: swap ICatalogModule from in-process to HTTP client

---

## My Confidence Level
- `[b]` Module structure — Domain/Application/Infrastructure/Tests per module
- `[b]` Separate DbContext per module with schema-per-module
- `[b]` Module registration in Program.cs (extension methods)
- `[b]` Public Module API pattern (ICatalogModule)
- `[b]` No cross-module EF navigation properties
- `[b]` Separate migrations per DbContext
- `[b]` When to use in-process call vs integration event
- `[b]` Repository in BC — one per aggregate root, only touches own DbContext
- `[b]` BC configuration — full module wiring (DbContext, repos, MediatR, MassTransit, workers)

## My Notes
<!-- Personal notes -->
