# BC Knowledge Map — Everything Inside a Bounded Context

Complete reference of every pattern, library, and technique needed to build and maintain a BC properly. Organized by layer.

---

## Domain Layer

Pure C# — zero framework dependencies. This layer must compile with no NuGet packages other than Shared.Kernel.

| What | Purpose |
|---|---|
| **Entity base class** | Identity, equality by ID, private setters |
| **AggregateRoot base class** | Extends Entity, holds domain events in memory |
| **Value Object base class** | Equality by value, immutable, `GetEqualityComponents()` |
| **Domain Events** (`IDomainEvent`) | Records that something happened — in-process only, dispatched after SaveChanges |
| **Domain Services** | Business logic spanning multiple aggregates (can't live on one entity) |
| **Domain Exceptions** | Thrown only for programmer errors — never for expected business failures |
| **Guard clauses** | Enforce invariants at construction: `if (amount < 0) throw new DomainException(...)` |
| **Backing fields + IReadOnlyCollection** | Prevent external mutation: `_items` private, `Items` exposes `AsReadOnly()` |
| **Factory methods** (`Create(...)`) | Enforce invariants on construction, raise domain events — no public constructors |
| **Enums / domain primitives** | `OrderStatus`, `PaymentMethod` — part of Ubiquitous Language |
| **Repository interfaces** | `IOrderRepository` — defined here, implemented in Infrastructure |

---

## Application Layer

Orchestrates use cases. No business logic. No framework references except MediatR and FluentValidation.

### Commands & Queries (CQRS)
| What | Purpose |
|---|---|
| **Commands** | Mutate state — `PlaceOrderCommand` |
| **Command Handlers** | Fetch aggregate → call domain method → commit via UoW |
| **Queries** | Read state — `GetOrderByIdQuery` |
| **Query Handlers** | Read from projection/DB with `AsNoTracking` → return DTO |

### Data Shapes
| What | Purpose |
|---|---|
| **Request models** | Input shapes — `CreateOrderRequest`, validated at API boundary |
| **DTOs** | Output shapes — `OrderDto`, `OrderSummaryDto` — never expose domain entities |
| **FluentValidation validators** | `AbstractValidator<PlaceOrderCommand>` — business-rule-free input validation |
| **Cross-field validators** | `.Must((cmd, val) => ...)` / `RuleFor(x => x).Custom(...)` — validate two fields together |
| **Async validators** | `MustAsync(async (val, ct) => await _repo.ExistsAsync(val))` — DB checks in validators |
| **`ValidationProblemDetails`** | Standard shape for validation 400: `{ errors: { "field": ["message"] } }` — different from `ProblemDetails` |

### Interfaces (contracts for Infrastructure)
| What | Purpose |
|---|---|
| **IUnitOfWork** | Exposes repositories + `CommitAsync()` |
| **IRepository interfaces** | `IOrderRepository` — one per aggregate root |
| **IEmailService, IBlobStorage, IPaymentGateway** | Any external service interface |

### Pipeline (MediatR Behaviors)
| What | Purpose |
|---|---|
| **ValidationBehavior** | Runs FluentValidation before every command — rejects invalid input |
| **LoggingBehavior** | Logs handler name + duration on every command/query |
| **TransactionBehavior** | Wraps command in a DB transaction — commits on success, rolls back on failure |
| **PerformanceBehavior** | Warns on slow handlers (> N ms) |

```
Request → ValidationBehavior → LoggingBehavior → TransactionBehavior → Handler
```

### Events
| What | Purpose |
|---|---|
| **Domain Event Handlers** | `INotificationHandler<OrderPlacedDomainEvent>` — bridge: creates integration event, writes to Outbox |
| **Integration Event contracts** | `OrderPlacedIntegrationEvent` — cross-BC message shapes, owned by this BC |
| **Integration Event Consumers** | Receive from RabbitMQ, check Inbox (idempotency), update projections or trigger commands |

### Patterns
| What | Purpose |
|---|---|
| **Result<T>** | Expected business outcomes — no exceptions for "order not found" |
| **ErrorCodes** | `public static class ErrorCodes { public const string OrderNotFound = "order.not_found"; }` |

---

## Infrastructure Layer

Framework-dependent. Implements interfaces from Domain/Application.

### EF Core — Persistence
| What | Purpose |
|---|---|
| **DbContext** | Owns this BC's DB tables, dispatches domain events in `SaveChangesAsync` override |
| **`IEntityTypeConfiguration<T>`** | One file per entity — maps columns, relationships, constraints, owned entities |
| **`ApplyConfigurationsFromAssembly`** | Scans and registers all configurations automatically in `OnModelCreating` |
| **Owned entities (`OwnsOne`)** | How Value Objects map to DB — same table, no separate PK |
| **Value Converters** | Map domain types to DB column types (`Money` → `decimal`, `Email` → `string`) |
| **Backing field wiring** | `.Metadata.PrincipalToDependent!.SetField("_items")` — EF writes to private list |
| **`AsNoTracking`** | Every query handler — no change tracking overhead |
| **Global Query Filters** | Soft delete (`WHERE IsDeleted = 0`), multi-tenancy — auto-applied to every query |
| **Interceptors** | `SaveChangesInterceptor` — set `UpdatedAt`, dispatch domain events (alternative to override) |
| **Migrations** | One migration project per DbContext, separate schema per module |
| **Bulk operations** | `ExecuteUpdateAsync` / `ExecuteDeleteAsync` — bypass change tracker, no entity load |
| **Connection resilience** | `EnableRetryOnFailure()` — EF automatically retries transient DB errors (network blip, deadlock) |
| **Raw SQL** | `FromSqlRaw` / `ExecuteSqlRawAsync` — for when LINQ produces poor SQL or for stored procs |
| **Zero-downtime migrations** | Expand-contract: add nullable column first → deploy → backfill → make NOT NULL → deploy again |
| **Multi-tenancy via Global Query Filter** | `modelBuilder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantId)` — auto-scopes every query |

### Repositories & UoW
| What | Purpose |
|---|---|
| **Repository implementation** | Implements `IOrderRepository`, touches only this BC's DbContext |
| **Unit of Work implementation** | Wraps `DbContext.SaveChangesAsync()`, exposes repositories |
| **Specification pattern** | Encapsulates complex query logic as an object — `ISpecification<T>` passed into repository |
| **Dapper alongside EF** | Raw SQL for performance-critical reads where LINQ produces bad SQL — EF for writes, Dapper for hot queries |

### Messaging
| What | Purpose |
|---|---|
| **OutboxMessage entity** | EF entity for the outbox table row |
| **OutboxDispatcher** | `BackgroundService` — polls outbox every N seconds, publishes to RabbitMQ via MassTransit |
| **InboxMessage entity** | EF entity for deduplication — `MessageId` PK, prevents double-processing |
| **MassTransit configuration** | Consumer registration, RabbitMQ host + credentials, retry intervals, DLQ routing |

### Projections
| What | Purpose |
|---|---|
| **Projection entities** | Flat, denormalized EF entities — built by consumers, read by query handlers |

### External services
| What | Purpose |
|---|---|
| **EmailService, BlobService, etc.** | Implement interfaces from Application layer |
| **`IHttpClientFactory`** | Manages `HttpClient` lifetimes — avoids socket exhaustion from creating clients per-request |
| **Named clients** | `services.AddHttpClient("stripe", c => c.BaseAddress = ...)` — named per external service |
| **Typed clients** | `services.AddHttpClient<IStripeClient, StripeClient>()` — strongly typed, inject directly |
| **Polly + `IHttpClientFactory`** | `.AddPolicyHandler(retryPolicy)` on `AddHttpClient` — retry/circuit breaker built into the client |

### Module registration
| What | Purpose |
|---|---|
| **`ModuleName.cs` extension method** | `AddOrdersModule(services, config)` — registers everything above in one call |

---

## API Layer

Thin. No business logic. No domain knowledge.

| What | Purpose |
|---|---|
| **Controllers** | Map HTTP request → MediatR command/query → return `ApiResponse<T>` |
| **`ApiResponse<T>` envelope** | Consistent shape: `{ success, data, errorCode, message }` |
| **`[ValidationFilter]`** | Runs before action — checks ModelState, returns 400 if invalid |
| **Exception handling middleware** | Catches unhandled exceptions → `ProblemDetails` (RFC 7807) |
| **`ProblemDetails`** | Standard HTTP error response — `{ type, title, status, detail }` |
| **Authentication middleware** | JWT validation, claims extraction |
| **Correlation ID middleware** | Generates/reads `X-Correlation-Id` header, attaches to every log entry |
| **Request logging middleware** | Logs method, path, status code, duration |
| **Program.cs** | Calls `AddOrdersModule()`, `AddInventoryModule()` — knows nothing else |

---

## Cross-Cutting Concerns (Shared.Kernel or standalone)

| What | Purpose |
|---|---|
| **Dependency Injection lifetimes** | Singleton / Scoped / Transient — captive dependency trap (Singleton → Scoped = bug) |
| **`IServiceScopeFactory`** | Create a Scoped lifetime inside a Singleton (BackgroundService → DbContext) |
| **Keyed services (.NET 8)** | `services.AddKeyedScoped<IPayment, StripePayment>("stripe")` — multiple implementations of same interface, resolved by key |
| **Decorator pattern in DI** | Wrap a service with cross-cutting behaviour without touching the original class — `services.Decorate<IOrderRepository, CachingOrderRepository>()` (Scrutor) |
| **Cancellation tokens** | Every async method: `CancellationToken ct = default` — propagate, never ignore |
| **`DateTime` vs `DateTimeOffset`** | Always use `DateTimeOffset` for timestamps — includes timezone offset, no ambiguity across regions |
| **Correlation ID** | Trace a request across all log entries — middleware generates once, flows through |
| **Structured logging (Serilog)** | JSON logs with typed properties — not `Console.WriteLine` |
| **Health checks** | `/health/live` (process alive) + `/health/ready` (DB + broker reachable) |
| **Options pattern** | `IOptions<T>` / `IOptionsMonitor<T>` — strongly typed `appsettings.json` sections |
| **Secrets management** | User Secrets (dev), env vars or Key Vault (prod) — never hardcode connection strings |
| **Audit fields** | `CreatedAt`, `UpdatedAt`, `CreatedBy` — set automatically via Interceptor or SaveChanges override |
| **Soft delete** | `IsDeleted` flag on entities + Global Query Filter to hide deleted rows everywhere |

---

## Libraries

| Library | What it does | Where it's used |
|---|---|---|
| **MediatR** | In-process command/query dispatch + pipeline behaviors | Application layer |
| **FluentValidation** | Declarative validators with `AbstractValidator<T>` | Application layer |
| **MassTransit** | Message bus abstraction — RabbitMQ, retry, DLQ, Saga, built-in Outbox | Infrastructure layer |
| **EF Core** | ORM — DbContext, configurations, migrations, LINQ | Infrastructure layer |
| **Polly** | Resilience — retry, circuit breaker, timeout, bulkhead | Infrastructure (HttpClient) |
| **Serilog** | Structured logging with sinks (console, file, Seq) | Cross-cutting |
| **xUnit** | Unit + integration test framework | Tests |
| **Moq** | Mock interfaces in unit tests | Tests |
| **Testcontainers** | Real SQL Server / RabbitMQ in Docker for integration tests | Tests |
| **Bogus** | Generate realistic fake test data | Tests |
| **Mapster / AutoMapper** | Object mapping DTO ↔ domain — optional, many map by hand | Application layer |

---

## EF Core — IEntityTypeConfiguration in Detail

This is the "configuration for entities" pattern. One class per entity, auto-discovered.

```csharp
// Orders.Infrastructure/Persistence/Configurations/OrderConfiguration.cs
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).ValueGeneratedNever();

        // Value Object as Owned Entity (same table, no FK)
        builder.OwnsOne(o => o.ShippingAddress, addr =>
        {
            addr.Property(a => a.Street).HasColumnName("Street").HasMaxLength(200);
            addr.Property(a => a.City).HasColumnName("City").HasMaxLength(100);
        });

        // Value Converter — map Money domain type to decimal column
        builder.Property(o => o.TotalAmount)
            .HasConversion(m => m.Amount, v => new Money(v, "USD"))
            .HasColumnName("TotalAmount")
            .HasPrecision(18, 2);

        // Backing field — EF writes directly to _items
        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("OrderId")
            .Metadata.PrincipalToDependent!.SetField("_items");

        builder.Property(o => o.Status)
            .HasConversion<string>() // store enum as string
            .HasMaxLength(50);
    }
}

// Registered in DbContext automatically:
protected override void OnModelCreating(ModelBuilder builder)
{
    builder.HasDefaultSchema("orders");
    builder.ApplyConfigurationsFromAssembly(typeof(OrdersDbContext).Assembly);
    // ↑ scans this assembly, finds all IEntityTypeConfiguration<T>, applies them
}
```

---

## Value Object — Owned Entity + Value Converter Pattern

```csharp
// Domain/ValueObjects/Money.cs
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative");
        Amount = amount;
        Currency = currency;
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}

// Infrastructure/Configurations/OrderConfiguration.cs
builder.OwnsOne(o => o.Price, price =>
{
    price.Property(p => p.Amount).HasColumnName("Price").HasPrecision(18, 2);
    price.Property(p => p.Currency).HasColumnName("Currency").HasMaxLength(3);
});
// Result in DB: two columns on the same Orders table — Price, Currency
// No separate table, no FK, no join needed
```

---

## Result Pattern

```csharp
// Shared.Kernel/Result.cs
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? ErrorCode { get; }

    private Result(bool isSuccess, T? value, string? errorCode)
    {
        IsSuccess = isSuccess;
        Value = value;
        ErrorCode = errorCode;
    }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string errorCode) => new(false, default, errorCode);
}

public class Result
{
    public bool IsSuccess { get; }
    public string? ErrorCode { get; }

    private Result(bool isSuccess, string? errorCode) { IsSuccess = isSuccess; ErrorCode = errorCode; }

    public static Result Success() => new(true, null);
    public static Result Failure(string errorCode) => new(false, errorCode);
}

// Controller translates to HTTP:
var result = await _mediator.Send(command, ct);
if (!result.IsSuccess)
    return result.ErrorCode switch
    {
        ErrorCodes.Orders.NotFound    => NotFound(ApiResponse.Error(result.ErrorCode)),
        ErrorCodes.Orders.InvalidStatus => Conflict(ApiResponse.Error(result.ErrorCode)),
        _ => BadRequest(ApiResponse.Error(result.ErrorCode))
    };
return Ok(ApiResponse.Success(result.Value));
```

---

## Complete BC Folder Structure

```
Modules/
└── Orders/
    ├── Orders.Domain/
    │   ├── Entities/
    │   │   ├── Order.cs
    │   │   └── OrderItem.cs
    │   ├── ValueObjects/
    │   │   ├── Money.cs
    │   │   └── Address.cs
    │   ├── Events/
    │   │   └── OrderPlacedDomainEvent.cs
    │   ├── Services/
    │   │   └── OrderPricingService.cs        ← domain service (optional)
    │   └── Interfaces/
    │       └── IOrderRepository.cs
    │
    ├── Orders.Application/
    │   ├── Commands/
    │   │   ├── PlaceOrderCommand.cs
    │   │   ├── PlaceOrderCommandHandler.cs
    │   │   ├── PlaceOrderCommandValidator.cs ← FluentValidation
    │   │   ├── CancelOrderCommand.cs
    │   │   └── CancelOrderCommandHandler.cs
    │   ├── Queries/
    │   │   ├── GetOrderByIdQuery.cs
    │   │   ├── GetOrderByIdQueryHandler.cs
    │   │   ├── GetOrdersQuery.cs
    │   │   └── GetOrdersQueryHandler.cs
    │   ├── DTOs/
    │   │   ├── OrderDto.cs
    │   │   └── OrderSummaryDto.cs
    │   ├── Behaviors/
    │   │   ├── ValidationBehavior.cs
    │   │   ├── LoggingBehavior.cs
    │   │   └── TransactionBehavior.cs
    │   ├── DomainEventHandlers/
    │   │   └── OrderPlacedDomainEventHandler.cs
    │   ├── IntegrationEvents/
    │   │   └── OrderPlacedIntegrationEvent.cs
    │   ├── Consumers/
    │   │   ├── StockReservedConsumer.cs
    │   │   └── StockFailedConsumer.cs
    │   └── Interfaces/
    │       ├── IUnitOfWork.cs
    │       └── IEmailService.cs
    │
    ├── Orders.Infrastructure/
    │   ├── Persistence/
    │   │   ├── OrdersDbContext.cs
    │   │   ├── Configurations/
    │   │   │   ├── OrderConfiguration.cs
    │   │   │   └── OrderItemConfiguration.cs
    │   │   ├── Repositories/
    │   │   │   └── OrderRepository.cs
    │   │   ├── UnitOfWork.cs
    │   │   └── Migrations/
    │   ├── Outbox/
    │   │   ├── OutboxMessage.cs
    │   │   └── OutboxDispatcher.cs
    │   ├── Inbox/
    │   │   └── InboxMessage.cs
    │   ├── Projections/
    │   │   └── StockStatusProjection.cs
    │   ├── ExternalServices/
    │   │   └── EmailService.cs
    │   └── OrdersModule.cs
    │
    └── Orders.Tests/
        ├── Unit/
        │   └── OrderTests.cs               ← test domain logic in isolation
        └── Integration/
            └── PlaceOrderTests.cs          ← Testcontainers, real DB
```

---

## What Each Pattern Prevents

| Pattern | Without it |
|---|---|
| **Rich domain model** | Business logic scattered in services, controllers — impossible to track |
| **Value Object** | Primitive obsession — `decimal price` can be negative, wrong currency |
| **Factory method** | Invalid aggregates — construction bypasses invariants |
| **Repository** | EF Core leaks into Application layer — impossible to unit test |
| **Unit of Work** | Multiple `SaveChangesAsync` calls in one use case — partial commits |
| **Pipeline behaviors** | Validation copy-pasted into every handler, transactions forgotten |
| **Result<T>** | Exception used for control flow — catches everywhere, slow, untestable |
| **Outbox** | Event lost if process crashes between business logic and broker publish |
| **Inbox** | Event processed twice on RabbitMQ retry — duplicate side effects |
| **IEntityTypeConfiguration** | All mapping in one giant `OnModelCreating` — unmaintainable |
| **Owned entities** | Value Objects get their own table with a FK — unnecessary join |
| **Value Converters** | Domain type exposed as DB type — EF can't map `Money` without this |
| **Global Query Filters** | Soft delete forgetting `WHERE IsDeleted = 0` on every query |
| **Projection** | Query handlers touching domain model — slow, tracking overhead |
| **Module extension method** | DI registration scattered — impossible to test or extract |

---

## My Confidence Level
- `[ ]` Full BC layer map from memory
- `[b]` Domain layer — entities, value objects, aggregates, events
- `[b]` Application layer — commands, queries, handlers, DTOs
- `[ ]` Pipeline behaviors — all four, where they sit, what each does
- `[b]` IEntityTypeConfiguration — owned entities, value converters, backing fields
- `[b]` Repository + Unit of Work
- `[b]` Result pattern + ErrorCodes
- `[b]` Outbox + Inbox + Projection
- `[ ]` Global Query Filters — soft delete + multi-tenancy
- `[ ]` Interceptors vs SaveChanges override — when to choose
- `[ ]` Polly + HttpClientFactory — resilient outbound HTTP
- `[ ]` Health checks setup
- `[ ]` Options pattern — IOptions vs IOptionsMonitor
