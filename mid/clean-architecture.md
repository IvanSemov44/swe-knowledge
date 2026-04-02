# Clean Architecture

## What it is
An architectural pattern where software is organized in concentric layers (Core → Application → Infrastructure/API). Dependencies always point inward — outer layers know about inner layers, never the reverse.

## Why it matters
It makes your codebase testable, maintainable, and framework-independent. You can swap your database, web framework, or messaging library without touching business logic.

---

## The Layers

```
┌────────────────────────────────────────────┐
│  API / Presentation                         │
│  (Controllers, Filters, Middleware)          │
├────────────────────────────────────────────┤
│  Application                                │
│  (Use Cases, Commands, Queries, DTOs,       │
│   Service Interfaces, IUnitOfWork)           │
├────────────────────────────────────────────┤
│  Core / Domain                              │
│  (Entities, Value Objects, Aggregates,      │
│   Domain Events, Domain Interfaces)          │
├────────────────────────────────────────────┤
│  Infrastructure                             │
│  (EF Core, Repositories, UoW, Email,        │
│   External APIs, File Storage)               │
└────────────────────────────────────────────┘
```

**Dependency Rule:** Code in an inner layer **never** references code in an outer layer.

```
API       → Application  ✅
API       → Core         ✅
Application → Core       ✅
Core      → Application  ❌
Core      → Infrastructure ❌
Application → Infrastructure ❌ (only through interfaces)
Infrastructure → Core    ✅ (implements Core interfaces)
Infrastructure → Application ✅ (implements Application interfaces)
```

---

## Layer Responsibilities

### Core (Domain)
- **Entities:** objects with identity (Order, Product, User)
- **Value Objects:** immutable, compared by value (Money, Address, Email)
- **Aggregates:** cluster of entities with one Aggregate Root
- **Domain Events:** things that happened (OrderPlaced, PaymentReceived)
- **Domain Interfaces:** `IOrderRepository`, `IProductRepository`
- **Domain Services:** logic that doesn't belong to one entity
- **No framework dependencies** — pure C# classes

### Application
- **Commands & Queries** (CQRS): use cases expressed as request objects
- **Command/Query Handlers:** orchestrate business logic
- **DTOs / Request / Response models**
- **Interfaces for infrastructure:** `IUnitOfWork`, `IEmailService`, `IBlobStorage`
- **Validation:** FluentValidation validators for commands
- **Pipeline behaviors:** logging, validation, transactions (MediatR)
- Depends only on Core

### Infrastructure
- **EF Core DbContext** and entity configurations
- **Repository implementations:** `OrderRepository : IOrderRepository`
- **Unit of Work implementation:** wraps `DbContext.SaveChangesAsync()`
- **External service clients:** email, Stripe, S3
- **Migrations**
- Depends on Core and Application (implements their interfaces)

### API
- **Controllers:** thin, call MediatR, return `ApiResponse<T>`
- **Middleware:** exception handling, request logging
- **Filters:** `[ValidationFilter]`, `[AuthorizeAttribute]`
- **Program.cs / DI registration**
- Depends on Application (sends commands/queries via MediatR)

---

## The Dependency Inversion Principle in Action

The Application layer defines an interface:
```csharp
// Application/Interfaces/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    void Add(Order order);
}
```

Infrastructure implements it:
```csharp
// Infrastructure/Repositories/OrderRepository.cs
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;
    public OrderRepository(AppDbContext context) => _context = context;

    public Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => _context.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);

    public void Add(Order order) => _context.Orders.Add(order);
}
```

Registration in API:
```csharp
// Program.cs
services.AddScoped<IOrderRepository, OrderRepository>();
```

---

## Unit of Work Pattern

Services inject `IUnitOfWork`, not individual repositories. The UoW exposes the repositories and commits the transaction.

```csharp
// Application/Interfaces/IUnitOfWork.cs
public interface IUnitOfWork
{
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    Task<int> CommitAsync(CancellationToken ct = default);
}
```

```csharp
// Command handler
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    private readonly IUnitOfWork _uow;

    public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var product = await _uow.Products.GetByIdAsync(cmd.ProductId, ct);
        if (product is null) return Result.Failure<Guid>(ErrorCodes.ProductNotFound);

        var order = Order.Create(cmd.UserId, product, cmd.Quantity);
        _uow.Orders.Add(order);
        await _uow.CommitAsync(ct);
        return Result.Success(order.Id);
    }
}
```

**Key rule:** Repositories never call `SaveChangesAsync`. Only `UnitOfWork.CommitAsync` commits the transaction.

---

## Result Pattern

Services return `Result<T>` for expected business outcomes — no exceptions for control flow.

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? ErrorCode { get; }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string errorCode) => new(false, default, errorCode);
}
```

Controller translates result to HTTP response:
```csharp
var result = await _mediator.Send(command, ct);
if (!result.IsSuccess)
    return result.ErrorCode switch
    {
        ErrorCodes.ProductNotFound => NotFound(ApiResponse.Error(result.ErrorCode)),
        ErrorCodes.InsufficientStock => Conflict(ApiResponse.Error(result.ErrorCode)),
        _ => BadRequest(ApiResponse.Error(result.ErrorCode))
    };
return Ok(ApiResponse.Success(result.Value));
```

---

## ErrorCodes

No magic strings for business failures:
```csharp
// Application/Common/ErrorCodes.cs
public static class ErrorCodes
{
    public const string ProductNotFound = "product.not_found";
    public const string InsufficientStock = "product.insufficient_stock";
    public const string OrderNotFound = "order.not_found";
    public const string InvalidOrderStatus = "order.invalid_status";
}
```

---

## ApiResponse<T>

Consistent response envelope:
```csharp
public class ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? ErrorCode { get; init; }
    public string? Message { get; init; }
}
```

---

## Common Interview Questions

1. What is the dependency rule in Clean Architecture?
2. Why do repositories not call `SaveChangesAsync`?
3. What belongs in Core vs Application vs Infrastructure?
4. Why return `Result<T>` instead of throwing exceptions?
5. How does Dependency Inversion work between Application and Infrastructure?

---

## Common Mistakes

- Putting business logic in controllers (fat controllers)
- Infrastructure leaking into Application (using `DbContext` directly in handlers)
- Core referencing Application or Infrastructure
- Throwing exceptions for expected failures (e.g., "product not found")
- Calling `SaveChangesAsync` inside a repository

---

## How It Connects

- DDD fits naturally inside Core layer
- CQRS + MediatR implements the Application layer use cases
- EF Core + Repositories live in Infrastructure
- Result pattern + ErrorCodes replace exception-based control flow
- `[ValidationFilter]` + FluentValidation validates at the API boundary

---

## My Confidence Level
- `[c]` Layer responsibilities
- `[c]` Dependency rule
- `[b]` Interfaces at boundaries
- `[b]` Unit of Work pattern
- `[b]` Result pattern

## My Notes
<!-- Personal notes -->
