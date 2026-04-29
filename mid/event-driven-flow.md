# Event-Driven Flow — Full Picture

## The Complete Flow

```
1. Handler calls aggregate method
2. Aggregate raises Domain Event
3. UnitOfWork.SaveChangesAsync commits domain changes
4. SaveChangesAsync override dispatches Domain Events via MediatR
5. Domain Event Handler creates Integration Event → writes to Outbox table (same transaction or separate)
6. Background Worker reads Outbox → publishes to RabbitMQ via MassTransit
7. Consumer in another module receives message
8. Consumer checks Inbox table (idempotency) → processes → marks as consumed
```

---

## Step-by-Step in Code

### Step 1–2: Aggregate Raises Domain Event

```csharp
// Orders.Domain
public class Order : AggregateRoot
{
    public Result Place()
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure(ErrorCodes.Orders.InvalidOrderStatus);

        Status = OrderStatus.Placed;

        AddDomainEvent(new OrderPlacedEvent(Id, UserId, CalculateTotal())); // in-process
        return Result.Success();
    }
}
```

### Step 3–4: SaveChangesAsync Dispatches Domain Events

```csharp
// Orders.Infrastructure — OrdersDbContext
public class OrdersDbContext : DbContext
{
    private readonly IMediator _mediator;

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // 1. Collect all domain events from tracked aggregates
        var events = ChangeTracker
            .Entries<AggregateRoot>()
            .SelectMany(e => e.Entity.DomainEvents)
            .ToList();

        // 2. Commit domain changes to DB
        var result = await base.SaveChangesAsync(ct);

        // 3. Dispatch domain events (in-process, MediatR)
        foreach (var domainEvent in events)
            await _mediator.Publish(domainEvent, ct);

        return result;
    }
}
```

### Step 5: Domain Event Handler Writes to Outbox

```csharp
// Orders.Application
public class OrderPlacedEventHandler : INotificationHandler<OrderPlacedEvent>
{
    private readonly OrdersDbContext _context;

    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
    {
        // Create integration event
        var integrationEvent = new OrderPlacedIntegrationEvent(
            notification.OrderId,
            notification.UserId,
            notification.TotalAmount);

        // Write to Outbox table
        _context.OutboxMessages.Add(new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = typeof(OrderPlacedIntegrationEvent).FullName!,
            Content = JsonSerializer.Serialize(integrationEvent),
            OccurredAt = DateTime.UtcNow
        });

        await _context.SaveChangesAsync(ct);
    }
}
```

### Step 6: Background Worker Publishes from Outbox

```csharp
// Orders.Infrastructure
public class OutboxWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessOutboxAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task ProcessOutboxAsync(CancellationToken ct)
    {
        var messages = await _context.OutboxMessages
            .Where(m => m.ProcessedAt == null)
            .OrderBy(m => m.OccurredAt)
            .Take(20)
            .ToListAsync(ct);

        foreach (var message in messages)
        {
            var eventType = Type.GetType(message.Type)!;
            var integrationEvent = JsonSerializer.Deserialize(message.Content, eventType)!;
            await _publisher.Publish(integrationEvent, ct); // MassTransit → RabbitMQ
            message.ProcessedAt = DateTime.UtcNow;
        }

        await _context.SaveChangesAsync(ct);
    }
}
```

### Step 7–8: Consumer With Inbox (Idempotency)

```csharp
// Inventory.Application — different module, consumes the event
public class OrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent>
{
    private readonly InventoryDbContext _context;

    public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> context)
    {
        // Inbox check — was this message already processed?
        var messageId = context.MessageId.ToString()!;
        var alreadyProcessed = await _context.InboxMessages
            .AnyAsync(m => m.MessageId == messageId);

        if (alreadyProcessed) return; // idempotent — safe to skip

        // Process the event
        foreach (var item in context.Message.Items)
            await ReserveStockAsync(item.ProductId, item.Quantity);

        // Mark as processed in Inbox
        _context.InboxMessages.Add(new InboxMessage
        {
            MessageId = messageId,
            ProcessedAt = DateTime.UtcNow
        });

        await _context.SaveChangesAsync(context.CancellationToken);
    }
}
```

---

## The Four Database Tables

```sql
-- 1. OUTBOX — publisher side (your module's DB)
CREATE TABLE OutboxMessages (
    Id          UNIQUEIDENTIFIER PRIMARY KEY,
    Type        NVARCHAR(500) NOT NULL,      -- full type name for deserialization
    Content     NVARCHAR(MAX) NOT NULL,      -- JSON payload
    OccurredAt  DATETIME2 NOT NULL,
    ProcessedAt DATETIME2 NULL               -- NULL = not yet published
);

-- 2. INBOX — consumer side (consuming module's DB)
CREATE TABLE InboxMessages (
    MessageId   NVARCHAR(100) PRIMARY KEY,   -- RabbitMQ message ID
    ProcessedAt DATETIME2 NOT NULL
);

-- 3. SAGA STATE — for long-running workflows (optional)
CREATE TABLE SagaState (
    CorrelationId   UNIQUEIDENTIFIER PRIMARY KEY,
    CurrentState    NVARCHAR(64) NOT NULL,
    OrderId         UNIQUEIDENTIFIER,
    CreatedAt       DATETIME2 NOT NULL,
    UpdatedAt       DATETIME2 NOT NULL
);

-- 4. CONSUMED MESSAGES — MassTransit's built-in deduplication table
-- Created automatically by MassTransit when you use its Outbox feature
```

---

## MassTransit Built-In Outbox (Simpler)

Instead of writing your own OutboxWorker, MassTransit can manage the Outbox for you:

```csharp
// Program.cs
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<OrdersDbContext>(o =>
    {
        o.UseSqlServer();              // uses your existing DbContext
        o.UseBusOutbox();              // replaces manual OutboxWorker
    });

    x.AddConsumer<OrderPlacedConsumer>();

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(config["RabbitMQ:Host"], h =>
        {
            h.Username(config["RabbitMQ:Username"]);
            h.Password(config["RabbitMQ:Password"]);
        });
        cfg.ConfigureEndpoints(ctx);
    });
});
```

MassTransit creates and manages the outbox table, the background worker, and retry logic automatically.

---

## MassTransit is Free — Cost Clarification

| Component | Cost |
|---|---|
| **MassTransit** | Free — open source .NET library |
| **RabbitMQ** (self-hosted via Docker) | Free |
| **CloudAMQP** (hosted RabbitMQ) | Paid — you don't need this |
| **Azure Service Bus** | Paid — you don't need this for local dev |

For your project: run RabbitMQ in Docker, use MassTransit as the library. Zero cost.

```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI (http://localhost:15672)
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
```

---

## Domain Event vs Integration Event — Code Distinction

```csharp
// Domain Event — in-process, MediatR, same module
public record OrderPlacedEvent(Guid OrderId, Guid UserId, decimal Total)
    : DomainEvent; // implements INotification for MediatR

// Integration Event — cross-boundary, MassTransit, RabbitMQ
public record OrderPlacedIntegrationEvent(Guid OrderId, Guid UserId, decimal Total)
    : IntegrationEvent; // what MassTransit publishes
```

**Flow:**
```
Order.Place()
  → raises OrderPlacedEvent (domain event, in-process)
  → MediatR dispatches to OrderPlacedEventHandler
  → Handler creates OrderPlacedIntegrationEvent
  → Writes to Outbox
  → MassTransit reads Outbox → publishes to RabbitMQ
  → Inventory.OrderPlacedConsumer receives it
```

---

## Projections

A **projection** is a read-optimized, denormalized table built by consuming events. It is NOT the event response itself — it is a local copy of data a BC needs for fast reads.

CQRS rule: **queries never read from the domain model**. They read from projections.

```
[Inventory BC]
  StockReservedIntegrationEvent published to RabbitMQ
    ↓
[Ordering BC] StockStatusConsumer
  → writes into StockStatusProjection table
    (OrderId, ProductId, Reserved: true, ReservedAt)
    ↓
GetOrderStockStatusQuery reads from StockStatusProjection (fast, no joins)
```

```csharp
// Projection table — flat, denormalized, query-optimized
public class StockStatusProjection
{
    public Guid OrderId { get; set; }
    public Guid ProductId { get; set; }
    public bool IsReserved { get; set; }
    public DateTime? ReservedAt { get; set; }
}

// Consumer that builds/updates the projection
public class StockReservedConsumer : IConsumer<StockReservedIntegrationEvent>
{
    private readonly OrdersDbContext _context;

    public async Task Consume(ConsumeContext<StockReservedIntegrationEvent> ctx)
    {
        var projection = await _context.StockStatuses
            .FirstOrDefaultAsync(s => s.OrderId == ctx.Message.OrderId);

        if (projection is null)
        {
            projection = new StockStatusProjection { OrderId = ctx.Message.OrderId };
            _context.StockStatuses.Add(projection);
        }

        projection.IsReserved = true;
        projection.ReservedAt = DateTime.UtcNow;
        await _context.SaveChangesAsync(ctx.CancellationToken);
    }
}

// Query handler reads from projection — no domain model, no joins
public class GetOrderStockStatusQueryHandler
    : IRequestHandler<GetOrderStockStatusQuery, StockStatusDto?>
{
    public async Task<StockStatusDto?> Handle(GetOrderStockStatusQuery q, CancellationToken ct)
        => await _context.StockStatuses
            .AsNoTracking()
            .Where(s => s.OrderId == q.OrderId)
            .Select(s => new StockStatusDto(s.OrderId, s.IsReserved, s.ReservedAt))
            .FirstOrDefaultAsync(ct);
}
```

**Key rule:** a projection is always rebuilt from events. If it gets corrupted or out of date, you replay the events to rebuild it.

---

## Retry and Dead Letter Queue

**DLQ is a RabbitMQ queue — not a database table.**

When a consumer fails after all retries, RabbitMQ routes the message to a dead letter exchange, which delivers it to a dead letter queue. You inspect it in the RabbitMQ Management UI at `http://localhost:15672`.

MassTransit handles retries and DLQ automatically:

```csharp
cfg.ReceiveEndpoint("order-placed", e =>
{
    e.ConfigureConsumer<OrderPlacedConsumer>(ctx);

    e.UseMessageRetry(r => r.Intervals(
        TimeSpan.FromSeconds(5),
        TimeSpan.FromSeconds(30),
        TimeSpan.FromMinutes(2),
        TimeSpan.FromMinutes(10)));
});
```

```
Normal queue:   order-placed          ← consumer reads here
DLQ:            order-placed_error    ← auto-routed after all retries fail
```

In the RabbitMQ UI you can see the failed message, its error, and replay it once the bug is fixed. MassTransit also writes error details to a `_skipped` queue for messages that can't be deserialized.

---

## Sagas — Long-Running Workflows

When a business process spans multiple steps across time:

```
Order Placed → Payment Processing → Payment Confirmed → Inventory Reserved → Order Shipped
```

Each step is async. A Saga tracks the state of the whole workflow.

```csharp
public class OrderSaga : MassTransitStateMachine<OrderSagaState>
{
    public State PaymentPending { get; private set; }
    public State InventoryReserving { get; private set; }
    public State Completed { get; private set; }

    public Event<OrderPlacedIntegrationEvent> OrderPlaced { get; private set; }
    public Event<PaymentConfirmedIntegrationEvent> PaymentConfirmed { get; private set; }
    public Event<InventoryReservedIntegrationEvent> InventoryReserved { get; private set; }

    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderPlaced, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => PaymentConfirmed, x => x.CorrelateById(ctx => ctx.Message.OrderId));
        Event(() => InventoryReserved, x => x.CorrelateById(ctx => ctx.Message.OrderId));

        Initially(
            When(OrderPlaced)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .Publish(ctx => new ProcessPaymentCommand(ctx.Saga.OrderId))
                .TransitionTo(PaymentPending));

        During(PaymentPending,
            When(PaymentConfirmed)
                .Publish(ctx => new ReserveInventoryCommand(ctx.Saga.OrderId))
                .TransitionTo(InventoryReserving));

        During(InventoryReserving,
            When(InventoryReserved)
                .TransitionTo(Completed)
                .Finalize());
    }
}
```

Saga state is persisted to the `SagaState` table — survives restarts.

---

## Common Interview Questions

1. What is the Outbox pattern and why do you need it?
2. What is the Inbox pattern and why do you need it?
3. What is the difference between a Domain Event and an Integration Event?
4. What is an idempotent consumer?
5. What happens to a message that keeps failing? (DLQ)
6. What is a Saga and when would you use one?
7. Is MassTransit free?

---

## Common Mistakes

- Publishing to RabbitMQ directly in the handler (no Outbox = unreliable)
- Not implementing Inbox/idempotency (duplicate processing on retry)
- Putting too much data in integration events (use IDs, consumer fetches current state)
- Not handling DLQ (failed messages disappear silently)
- Using Sagas for simple one-step processes (overkill)
- Thinking MassTransit costs money (it doesn't)

---

## How It Connects

- `AggregateRoot.AddDomainEvent()` from Shared.Kernel starts the chain
- `SaveChangesAsync` override dispatches domain events — this is the trigger
- Domain event handlers bridge to integration events → Outbox
- MassTransit manages Outbox + RabbitMQ publishing
- Inbox table is the consumer-side guarantee of idempotency
- Sagas coordinate multi-step async workflows using the same event system

---

## My Confidence Level
- `[b]` Full event flow — domain event → outbox → RabbitMQ → consumer → inbox
- `[b]` Outbox pattern — manual implementation
- `[ ]` MassTransit built-in Outbox setup
- `[b]` Inbox pattern — idempotent consumer
- `[b]` Dead Letter Queue — RabbitMQ queue (not DB table), auto-routed on retry exhaustion
- `[ ]` MassTransit consumer setup
- `[b]` Saga basics — state machine for long-running workflows
- `[ ]` RabbitMQ + Docker setup
- `[b]` Projections — read-optimized tables built from events, queries read from these

## My Notes
<!-- Personal notes -->
