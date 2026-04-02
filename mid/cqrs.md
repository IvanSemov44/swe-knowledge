# CQRS + MediatR

## What it is
**CQRS (Command Query Responsibility Segregation):** Separate the model for reading data (Queries) from the model for writing data (Commands). They have different concerns, different performance profiles, and different optimization needs.

**MediatR:** A .NET library implementing the Mediator pattern. Decouples senders from handlers — controllers send requests, handlers process them, with no direct dependency between them.

## Why it matters
- Queries can be optimized for reads (projections, no tracking, read replicas) without touching the write model
- Commands enforce business rules and domain invariants
- MediatR enables cross-cutting concerns (logging, validation, transactions) via pipeline behaviors without cluttering handlers

---

## Commands vs Queries

| | Command | Query |
|---|---|---|
| Purpose | Change state | Read state |
| Returns | `Result<T>` or `Result` | Read model / DTO |
| Side effects | Yes | No |
| Example | `CreateOrderCommand` | `GetOrderByIdQuery` |
| Domain model | Uses aggregates, enforces rules | Can bypass domain, use direct SQL/projections |

**Command naming:** `VerbNounCommand` — `CreateOrderCommand`, `CancelOrderCommand`, `UpdateProductPriceCommand`

**Query naming:** `GetNounQuery` — `GetOrderByIdQuery`, `GetProductListQuery`

---

## MediatR Setup

```csharp
// Program.cs
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly));
```

---

## Command Example

```csharp
// Command (Application layer)
public record CreateOrderCommand(
    Guid UserId,
    Guid ProductId,
    int Quantity,
    Address ShippingAddress
) : IRequest<Result<Guid>>;

// Handler (Application layer)
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    private readonly IUnitOfWork _uow;

    public CreateOrderCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var product = await _uow.Products.GetByIdAsync(cmd.ProductId, ct);
        if (product is null) return Result.Failure<Guid>(ErrorCodes.ProductNotFound);

        if (product.Stock < cmd.Quantity)
            return Result.Failure<Guid>(ErrorCodes.InsufficientStock);

        var order = Order.Create(cmd.UserId, cmd.ShippingAddress);
        order.AddItem(product, cmd.Quantity);

        _uow.Orders.Add(order);
        await _uow.CommitAsync(ct);

        return Result.Success(order.Id);
    }
}
```

---

## Query Example

```csharp
// Query — returns a DTO, not a domain object
public record GetOrderByIdQuery(Guid OrderId) : IRequest<Result<OrderDto>>;

// Handler — can use direct EF projections, bypasses domain model
public class GetOrderByIdQueryHandler : IRequestHandler<GetOrderByIdQuery, Result<OrderDto>>
{
    private readonly AppDbContext _context;

    public GetOrderByIdQueryHandler(AppDbContext context) => _context = context;

    public async Task<Result<OrderDto>> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        var order = await _context.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderDto
            {
                Id = o.Id,
                Status = o.Status.ToString(),
                Items = o.Items.Select(i => new OrderItemDto
                {
                    ProductName = i.Product.Name,
                    Quantity = i.Quantity,
                    UnitPrice = i.UnitPrice
                }).ToList(),
                TotalAmount = o.Items.Sum(i => i.Quantity * i.UnitPrice)
            })
            .FirstOrDefaultAsync(ct);

        return order is null
            ? Result.Failure<OrderDto>(ErrorCodes.OrderNotFound)
            : Result.Success(order);
    }
}
```

**Query handlers inject `AppDbContext` directly** — they're optimized for reads, don't need the repository abstraction. `AsNoTracking()` is mandatory.

---

## Controller (Thin)

```csharp
[ApiController]
[Route("api/v1/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    [HttpPost]
    [ValidationFilter]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderRequest request,
        CancellationToken ct)
    {
        var command = new CreateOrderCommand(
            UserId: CurrentUserId,
            ProductId: request.ProductId,
            Quantity: request.Quantity,
            ShippingAddress: request.ShippingAddress.ToValueObject()
        );
        var result = await _mediator.Send(command, ct);
        return result.ToActionResult(id => CreatedAtAction(nameof(GetById), new { id }, null));
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken ct)
    {
        var result = await _mediator.Send(new GetOrderByIdQuery(id), ct);
        return result.ToActionResult();
    }
}
```

---

## Pipeline Behaviors

Middleware for MediatR. Runs around every command/query handler. Registered in order.

```csharp
// Validation behavior (runs before handler)
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!_validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = _validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Any())
            throw new ValidationException(failures);

        return await next();
    }
}
```

```csharp
// Logging behavior
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        _logger.LogInformation("Handling {RequestName}", typeof(TRequest).Name);
        var response = await next();
        _logger.LogInformation("Handled {RequestName}", typeof(TRequest).Name);
        return response;
    }
}
```

```csharp
// Registration order matters
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TransactionBehavior<,>));
```

---

## Outbox Pattern (Reliable Domain Event Publishing)

Problem: Domain events dispatched after `SaveChanges` can be lost if the process crashes between commit and dispatch.

**Solution:** Save domain events to an `OutboxMessages` table in the **same transaction** as the domain change. A background job reads and dispatches them.

```csharp
// After SaveChanges in UoW, write domain events to OutboxMessages table
// Background job (Hangfire/Worker) polls OutboxMessages, publishes, marks as processed
```

---

## Common Interview Questions

1. What is CQRS and why would you use it?
2. What is the difference between a Command and a Query in CQRS?
3. What is MediatR and what problem does it solve?
4. What are pipeline behaviors and what can you use them for?
5. Why can Query handlers inject DbContext directly instead of using repositories?

---

## Common Mistakes

- Commands returning complex domain objects (should return `Result<T>` with just the ID)
- Query handlers using the domain model (they can project directly from DB)
- Not using `AsNoTracking()` in query handlers
- Putting business logic in the controller instead of the command handler
- Mixing commands and queries in one handler class

---

## How It Connects

- Clean Architecture: CQRS lives in the Application layer
- MediatR decouples Controllers (API layer) from Handlers (Application layer)
- Pipeline behaviors implement cross-cutting concerns without AOP
- Domain Events from DDD are dispatched via MediatR `INotification` / `INotificationHandler`
- Outbox Pattern + MediatR = reliable event-driven architecture

---

## My Confidence Level
- `[b]` Command vs Query separation
- `[b]` MediatR pipeline behaviors
- `[b]` Validation pipeline behavior
- `[~]` Event sourcing basics
- `[ ]` Outbox pattern

## My Notes
<!-- Personal notes -->
