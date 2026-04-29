# Event-Driven Design — Full Implementation Reference

Complete walkthrough of every file, its folder location, and its role in the flow.
Scenario: Order placed → Inventory reserves stock → Ordering updates its projection.

---

## Folder Structure

```
ECommerce.sln
│
├── Shared.Kernel/
│   ├── AggregateRoot.cs          ← base class, holds domain events in memory
│   ├── IDomainEvent.cs           ← marker: in-process events (MediatR)
│   └── IIntegrationEvent.cs      ← marker: cross-BC events (MassTransit)
│
├── Modules/
│   │
│   ├── Orders/
│   │   ├── Orders.Domain/
│   │   │   ├── Entities/
│   │   │   │   └── Order.cs                         ← [STEP 1] raises domain event
│   │   │   └── Events/
│   │   │       └── OrderPlacedDomainEvent.cs        ← [STEP 1] the in-process event
│   │   │
│   │   ├── Orders.Application/
│   │   │   ├── Commands/
│   │   │   │   ├── PlaceOrderCommand.cs
│   │   │   │   └── PlaceOrderCommandHandler.cs      ← [STEP 0] entry point
│   │   │   ├── DomainEventHandlers/
│   │   │   │   └── OrderPlacedDomainEventHandler.cs ← [STEP 3] writes to outbox
│   │   │   ├── IntegrationEvents/
│   │   │   │   └── OrderPlacedIntegrationEvent.cs   ← [STEP 3] cross-BC contract
│   │   │   └── Consumers/
│   │   │       └── StockReservedConsumer.cs          ← [STEP 8] receives Inventory reply
│   │   │
│   │   └── Orders.Infrastructure/
│   │       ├── Persistence/
│   │       │   ├── OrdersDbContext.cs                ← [STEP 2] dispatches domain events
│   │       │   ├── Configurations/
│   │       │   │   └── OrderConfiguration.cs
│   │       │   └── Repositories/
│   │       │       └── OrderRepository.cs
│   │       ├── Outbox/
│   │       │   ├── OutboxMessage.cs                  ← outbox table row entity
│   │       │   └── OutboxDispatcher.cs               ← [STEP 4] BackgroundService → RabbitMQ
│   │       ├── Inbox/
│   │       │   └── InboxMessage.cs                   ← deduplication table row entity
│   │       ├── Projections/
│   │       │   └── StockStatusProjection.cs          ← [STEP 8] read model
│   │       └── OrdersModule.cs                       ← DI wiring
│   │
│   └── Inventory/
│       ├── Inventory.Domain/
│       │   └── Entities/
│       │       └── InventoryItem.cs                  ← [STEP 6] reserves stock
│       │
│       ├── Inventory.Application/
│       │   ├── IntegrationEvents/
│       │   │   ├── OrderPlacedIntegrationEvent.cs    ← Inventory's OWN copy
│       │   │   └── StockReservedIntegrationEvent.cs  ← reply event
│       │   └── Consumers/
│       │       └── OrderPlacedConsumer.cs            ← [STEP 5+6+7]
│       │
│       └── Inventory.Infrastructure/
│           ├── Persistence/
│           │   └── InventoryDbContext.cs
│           ├── Outbox/
│           │   ├── OutboxMessage.cs
│           │   └── OutboxDispatcher.cs               ← [STEP 7] publishes reply
│           ├── Inbox/
│           │   └── InboxMessage.cs
│           └── InventoryModule.cs
│
└── Api/
    └── Program.cs                                    ← calls AddOrdersModule + AddInventoryModule
```

---

## Shared.Kernel

```csharp
// Shared.Kernel/IDomainEvent.cs
public interface IDomainEvent : INotification { }
// INotification from MediatR — in-process publish only

// Shared.Kernel/IIntegrationEvent.cs
public interface IIntegrationEvent
{
    Guid Id { get; }
    DateTime OccurredAt { get; }
}

// Shared.Kernel/AggregateRoot.cs
public abstract class AggregateRoot : Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();
    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents()
        => _domainEvents.Clear();
}
```

---

## STEP 0 — Command Handler (entry point)

```csharp
// Orders.Application/Commands/PlaceOrderCommandHandler.cs
public class PlaceOrderCommandHandler : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly IUnitOfWork _uow;

    public PlaceOrderCommandHandler(IUnitOfWork uow) => _uow = uow;

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.UserId, cmd.ShippingAddress);
        foreach (var item in cmd.Items)
            order.AddItem(item.ProductId, item.ProductName, item.UnitPrice, item.Quantity);

        var result = order.Place(); // STEP 1 happens inside here
        if (!result.IsSuccess) return Result.Failure<Guid>(result.ErrorCode!);

        _uow.Orders.Add(order);
        await _uow.CommitAsync(ct); // STEP 2 happens inside here
        return Result.Success(order.Id);
    }
}
```

---

## STEP 1 — Aggregate Raises Domain Event

```csharp
// Orders.Domain/Entities/Order.cs
public class Order : AggregateRoot
{
    public Guid Id { get; private set; }
    public Guid UserId { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { }

    public static Order Create(Guid userId, Address shippingAddress) => new()
    {
        Id = Guid.NewGuid(),
        UserId = userId,
        Status = OrderStatus.Pending
    };

    public void AddItem(Guid productId, string name, decimal price, int qty)
        => _items.Add(OrderItem.Create(productId, name, price, qty));

    public Result Place()
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure(ErrorCodes.Orders.InvalidStatus);

        Status = OrderStatus.Placed;
        AddDomainEvent(new OrderPlacedDomainEvent(Id, UserId, _items.ToList()));
        // Event stored in memory ONLY — not in DB yet
        return Result.Success();
    }
}

// Orders.Domain/Events/OrderPlacedDomainEvent.cs
public record OrderPlacedDomainEvent(
    Guid OrderId,
    Guid UserId,
    List<OrderItem> Items
) : IDomainEvent;
```

---

## STEP 2 — SaveChanges Dispatches Domain Events

```csharp
// Orders.Infrastructure/Persistence/OrdersDbContext.cs
public class OrdersDbContext : DbContext
{
    private readonly IMediator _mediator;

    public OrdersDbContext(DbContextOptions<OrdersDbContext> options, IMediator mediator)
        : base(options) => _mediator = mediator;

    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OutboxMessage> OutboxMessages => Set<OutboxMessage>();
    public DbSet<InboxMessage> InboxMessages => Set<InboxMessage>();
    public DbSet<StockStatusProjection> StockStatuses => Set<StockStatusProjection>();

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        var aggregates = ChangeTracker
            .Entries<AggregateRoot>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        // 1. Save domain state first
        var result = await base.SaveChangesAsync(ct);

        // 2. Dispatch domain events in-process via MediatR
        foreach (var aggregate in aggregates)
        {
            foreach (var domainEvent in aggregate.DomainEvents)
                await _mediator.Publish(domainEvent, ct);

            aggregate.ClearDomainEvents();
        }

        return result;
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.HasDefaultSchema("orders");
        builder.ApplyConfigurationsFromAssembly(typeof(OrdersDbContext).Assembly);
    }
}
```

---

## STEP 3 — Domain Event Handler Writes to Outbox

```csharp
// Orders.Application/DomainEventHandlers/OrderPlacedDomainEventHandler.cs
public class OrderPlacedDomainEventHandler : INotificationHandler<OrderPlacedDomainEvent>
{
    private readonly OrdersDbContext _context;

    public OrderPlacedDomainEventHandler(OrdersDbContext context) => _context = context;

    public async Task Handle(OrderPlacedDomainEvent notification, CancellationToken ct)
    {
        var integrationEvent = new OrderPlacedIntegrationEvent(
            notification.OrderId,
            notification.Items.Select(i => new OrderItemDto(i.ProductId, i.Quantity)).ToList()
        );

        _context.OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = typeof(OrderPlacedIntegrationEvent).FullName!,
            Payload = JsonSerializer.Serialize(integrationEvent),
            OccurredAt = DateTime.UtcNow,
            ProcessedAt = null
        });

        await _context.SaveChangesAsync(ct);
        // DB now has: Order row + OutboxMessage row — atomic
    }
}

// Orders.Application/IntegrationEvents/OrderPlacedIntegrationEvent.cs
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    List<OrderItemDto> Items
) : IIntegrationEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

public record OrderItemDto(Guid ProductId, int Quantity);

// Orders.Infrastructure/Outbox/OutboxMessage.cs
public class OutboxMessage
{
    public Guid Id { get; set; }
    public string Type { get; set; } = string.Empty;    // full class name for deserialization
    public string Payload { get; set; } = string.Empty; // JSON
    public DateTime OccurredAt { get; set; }
    public DateTime? ProcessedAt { get; set; }          // null = pending publish
    public string? Error { get; set; }
}
```

---

## STEP 4 — OutboxDispatcher Publishes to RabbitMQ

```csharp
// Orders.Infrastructure/Outbox/OutboxDispatcher.cs
public class OutboxDispatcher : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly IPublishEndpoint _publishEndpoint;

    public OutboxDispatcher(IServiceScopeFactory scopeFactory, IPublishEndpoint publishEndpoint)
    {
        _scopeFactory = scopeFactory;
        _publishEndpoint = publishEndpoint;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessPendingAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessPendingAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<OrdersDbContext>();

        var pending = await context.OutboxMessages
            .Where(m => m.ProcessedAt == null)
            .OrderBy(m => m.OccurredAt)
            .Take(20)
            .ToListAsync(ct);

        foreach (var message in pending)
        {
            try
            {
                var eventType = Type.GetType(message.Type)!;
                var integrationEvent = JsonSerializer.Deserialize(message.Payload, eventType)!;
                await _publishEndpoint.Publish(integrationEvent, eventType, ct);
                message.ProcessedAt = DateTime.UtcNow;
            }
            catch (Exception ex)
            {
                message.Error = ex.Message;
            }
        }

        await context.SaveChangesAsync(ct);
    }
}
// Note: uses IServiceScopeFactory because BackgroundService is Singleton,
// but DbContext is Scoped — cannot inject Scoped into Singleton directly.
```

---

## STEP 5+6+7 — Inventory Receives, Processes, Replies

```csharp
// Inventory.Application/IntegrationEvents/OrderPlacedIntegrationEvent.cs
// Inventory's OWN copy — intentional. Each BC owns its contracts.
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    List<OrderItemDto> Items
) : IIntegrationEvent
{
    public Guid Id { get; init; }
    public DateTime OccurredAt { get; init; }
}

// Inventory.Application/Consumers/OrderPlacedConsumer.cs
public class OrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent>
{
    private readonly InventoryDbContext _context;
    private readonly IUnitOfWork _uow;

    public OrderPlacedConsumer(InventoryDbContext context, IUnitOfWork uow)
    {
        _context = context;
        _uow = uow;
    }

    public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> ctx)
    {
        var messageId = ctx.MessageId.ToString()!;

        // STEP 5a: Inbox check — idempotency
        if (await _context.InboxMessages.AnyAsync(m => m.MessageId == messageId))
            return; // already processed — safe to skip

        // STEP 6: Reserve stock
        bool allReserved = true;
        foreach (var item in ctx.Message.Items)
        {
            var inventoryItem = await _uow.InventoryItems.GetByProductIdAsync(item.ProductId);
            if (inventoryItem is null || !inventoryItem.HasStock(item.Quantity))
            {
                allReserved = false;
                break;
            }
            inventoryItem.Reserve(item.Quantity);
        }

        // STEP 7: Write reply to Inventory's own Outbox (NOT directly to RabbitMQ)
        IIntegrationEvent reply = allReserved
            ? new StockReservedIntegrationEvent(ctx.Message.OrderId)
            : new StockFailedIntegrationEvent(ctx.Message.OrderId, "Insufficient stock");

        _context.OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = reply.GetType().FullName!,
            Payload = JsonSerializer.Serialize(reply, reply.GetType()),
            OccurredAt = DateTime.UtcNow
        });

        _context.InboxMessages.Add(new InboxMessage
        {
            MessageId = messageId,
            ProcessedAt = DateTime.UtcNow
        });

        await _uow.CommitAsync(ctx.CancellationToken);
        // One transaction: stock updated + OutboxMessage + InboxMessage
    }
}

// Inventory.Infrastructure/Inbox/InboxMessage.cs
public class InboxMessage
{
    public string MessageId { get; set; } = string.Empty;
    public DateTime ProcessedAt { get; set; }
}
```

---

## STEP 8 — Ordering Receives Reply and Updates Projection

```csharp
// Orders.Application/Consumers/StockReservedConsumer.cs
public class StockReservedConsumer : IConsumer<StockReservedIntegrationEvent>
{
    private readonly OrdersDbContext _context;

    public StockReservedConsumer(OrdersDbContext context) => _context = context;

    public async Task Consume(ConsumeContext<StockReservedIntegrationEvent> ctx)
    {
        var messageId = ctx.MessageId.ToString()!;

        if (await _context.InboxMessages.AnyAsync(m => m.MessageId == messageId))
            return;

        var projection = await _context.StockStatuses
            .FirstOrDefaultAsync(s => s.OrderId == ctx.Message.OrderId);

        if (projection is null)
        {
            projection = new StockStatusProjection { OrderId = ctx.Message.OrderId };
            _context.StockStatuses.Add(projection);
        }

        projection.IsReserved = true;
        projection.ReservedAt = DateTime.UtcNow;

        _context.InboxMessages.Add(new InboxMessage
        {
            MessageId = messageId,
            ProcessedAt = DateTime.UtcNow
        });

        await _context.SaveChangesAsync(ctx.CancellationToken);
    }
}

// Orders.Infrastructure/Projections/StockStatusProjection.cs
public class StockStatusProjection
{
    public Guid OrderId { get; set; }
    public bool IsReserved { get; set; }
    public DateTime? ReservedAt { get; set; }
}
```

---

## Module Wiring

```csharp
// Orders.Infrastructure/OrdersModule.cs
public static class OrdersModule
{
    public static IServiceCollection AddOrdersModule(
        this IServiceCollection services, IConfiguration config)
    {
        services.AddDbContext<OrdersDbContext>(opts =>
            opts.UseSqlServer(config.GetConnectionString("Default")));

        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IUnitOfWork, UnitOfWork>();

        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(PlaceOrderCommand).Assembly));

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

                cfg.ReceiveEndpoint("orders.stock-result", e =>
                {
                    e.ConfigureConsumer<StockReservedConsumer>(ctx);
                    e.ConfigureConsumer<StockFailedConsumer>(ctx);
                    e.UseMessageRetry(r => r.Intervals(
                        TimeSpan.FromSeconds(5),
                        TimeSpan.FromSeconds(30),
                        TimeSpan.FromMinutes(2)));
                });
            });
        });

        services.AddHostedService<OutboxDispatcher>();

        return services;
    }
}

// Api/Program.cs
builder.Services.AddOrdersModule(builder.Configuration);
builder.Services.AddInventoryModule(builder.Configuration);
```

---

## File → Purpose Reference

| File | Layer | What it does |
|---|---|---|
| `IDomainEvent` | Shared.Kernel | Marker: in-process MediatR event |
| `IIntegrationEvent` | Shared.Kernel | Marker: cross-BC RabbitMQ event |
| `AggregateRoot` | Shared.Kernel | Holds domain events in memory until SaveChanges |
| `Order.cs` | Domain | Business logic; calls AddDomainEvent |
| `OrderPlacedDomainEvent` | Domain | Record of what happened in-process |
| `PlaceOrderCommandHandler` | Application | Entry point; calls aggregate, commits |
| `OrderPlacedDomainEventHandler` | Application | Bridge: domain → integration; writes outbox row |
| `OrderPlacedIntegrationEvent` | Application | Cross-BC contract; what travels over RabbitMQ |
| `OrdersDbContext` | Infrastructure | Saves state; dispatches domain events after save |
| `OutboxMessage` | Infrastructure | EF entity for the outbox table row |
| `OutboxDispatcher` | Infrastructure | BackgroundService; reads outbox → publishes to broker |
| `OrderPlacedConsumer` (Inventory) | Application | Receives from RabbitMQ; inbox check; processes; replies |
| `InboxMessage` | Infrastructure | EF entity for deduplication table |
| `StockReservedConsumer` (Orders) | Application | Receives reply; updates projection |
| `StockStatusProjection` | Infrastructure | EF entity for the read model table |
| `OrdersModule` | Infrastructure | Registers all of the above in DI |

---

## The Full Flow

```
PlaceOrderCommandHandler
  → Order.Place()
      → AddDomainEvent(OrderPlacedDomainEvent)   [in memory]
  → UnitOfWork.CommitAsync()
      → OrdersDbContext.SaveChangesAsync()
          → base.SaveChangesAsync()              [Order row saved]
          → MediatR.Publish(OrderPlacedDomainEvent)
              → OrderPlacedDomainEventHandler
                  → writes OutboxMessage row     [atomic with Order row]

[5s later] OutboxDispatcher (BackgroundService)
  → reads pending OutboxMessage rows
  → MassTransit.Publish(OrderPlacedIntegrationEvent) → RabbitMQ

[RabbitMQ] → Inventory.OrderPlacedConsumer
  → inbox check (idempotency)
  → InventoryItem.Reserve()
  → writes StockReservedIntegrationEvent to Inventory's OutboxMessage
  → writes InboxMessage row
  [all in one transaction]

[5s later] Inventory's OutboxDispatcher
  → MassTransit.Publish(StockReservedIntegrationEvent) → RabbitMQ

[RabbitMQ] → Orders.StockReservedConsumer
  → inbox check
  → updates StockStatusProjection
  → writes InboxMessage row

GetOrderStockStatusQuery → reads StockStatusProjection (fast, no joins)
```

---

## Common Confusion Points

**Why does the DomainEventHandler also call SaveChangesAsync?**
Because it needs to write the OutboxMessage to the DB. It uses the same DbContext instance (scoped per request), so no new transaction — it just appends to the same session.

**Why does Inventory write to its own Outbox instead of publishing directly?**
Same reason Ordering does — if the consumer processes the business logic but crashes before publishing the reply, the reply is lost. Outbox guarantees the reply is persisted in the same transaction as the stock reservation.

**Why does each BC have its own copy of `OrderPlacedIntegrationEvent`?**
Each BC owns its contracts. Sharing a single class creates a compile-time dependency between BCs — one change forces a rebuild of both. Separate copies evolve independently.

**Why is `IServiceScopeFactory` used in OutboxDispatcher?**
`BackgroundService` is registered as Singleton. `DbContext` is Scoped. You cannot inject a Scoped service into a Singleton. `IServiceScopeFactory` creates a new scope per iteration — each poll gets its own fresh DbContext.

---

## My Confidence Level
- `[b]` Full event flow with folder structure
- `[b]` Domain event → outbox → dispatcher → RabbitMQ → consumer → inbox → projection
- `[b]` Why each file exists and where it lives
- `[b]` Idempotency via Inbox pattern
- `[b]` Why each BC has its own Outbox and Inbox
- `[b]` Why each BC has its own copy of integration event contracts
- `[ ]` MassTransit built-in Outbox (replaces manual OutboxMessage + OutboxDispatcher)
- `[ ]` Saga implementation with MassTransit StateMachine
