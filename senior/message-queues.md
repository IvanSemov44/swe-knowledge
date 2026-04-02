# Message Queues

## What it is
Middleware that enables asynchronous communication between services. A producer sends a message to a queue; a consumer reads and processes it independently. They don't need to be running at the same time.

## Why it matters
Message queues decouple services, absorb traffic spikes, enable retry on failure, and allow parallel processing. Without them, every service call is synchronous and failures cascade.

---

## Core Concepts

**Producer:** Sends messages
**Queue/Topic:** Stores messages until consumed
**Consumer:** Reads and processes messages
**Exchange (RabbitMQ):** Routes messages to queues based on rules
**Broker:** The message queue server (RabbitMQ, Kafka, Azure Service Bus)

---

## RabbitMQ

### Key Concepts

**Exchange types:**
- **Direct:** Route by exact routing key
- **Topic:** Route by pattern (`order.#`, `*.created`)
- **Fanout:** Broadcast to all bound queues
- **Headers:** Route by message headers

**Message lifecycle:**
```
Producer → Exchange → (routing key match) → Queue → Consumer
```

### Setup in .NET
```bash
dotnet add package MassTransit.RabbitMQ
```

```csharp
// Program.cs
services.AddMassTransit(x =>
{
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

### Publishing a Message
```csharp
public class OrderCommandHandler : IRequestHandler<PlaceOrderCommand, Result<Guid>>
{
    private readonly IPublishEndpoint _publish;
    private readonly IUnitOfWork _uow;

    public async Task<Result<Guid>> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.UserId, cmd.ShippingAddress);
        // ... add items
        _uow.Orders.Add(order);
        await _uow.CommitAsync(ct);

        // Publish event (fire and forget — decoupled)
        await _publish.Publish(new OrderPlacedIntegrationEvent(
            OrderId: order.Id,
            UserId: order.UserId,
            TotalAmount: order.Total), ct);

        return Result.Success(order.Id);
    }
}
```

### Consuming a Message
```csharp
public class OrderPlacedConsumer : IConsumer<OrderPlacedIntegrationEvent>
{
    private readonly IEmailService _emailService;
    private readonly IInventoryService _inventoryService;

    public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> context)
    {
        var evt = context.Message;
        await _emailService.SendOrderConfirmationAsync(evt.UserId, evt.OrderId);
        await _inventoryService.ReserveAsync(evt.OrderId);
    }
}
```

---

## Delivery Guarantees

| Guarantee | Meaning | Trade-off |
|---|---|---|
| **At-most-once** | Message delivered 0 or 1 times. May be lost. | No duplicates, but can lose messages |
| **At-least-once** | Message delivered 1 or more times. Never lost. | May be delivered multiple times |
| **Exactly-once** | Message delivered exactly once. | Complex, expensive, rare |

**Production default:** At-least-once delivery + **idempotent consumers**.

### Idempotent Consumer
A consumer that safely handles duplicate messages:
```csharp
public async Task Consume(ConsumeContext<OrderPlacedIntegrationEvent> context)
{
    // Check if already processed
    if (await _processedMessages.ExistsAsync(context.MessageId.ToString()))
        return; // Already handled, skip

    // Process
    await _emailService.SendConfirmationAsync(context.Message.UserId, context.Message.OrderId);

    // Mark as processed
    await _processedMessages.MarkAsync(context.MessageId.ToString());
}
```

---

## Dead Letter Queue (DLQ)

When a consumer repeatedly fails to process a message, it's moved to a DLQ after N retries.

```csharp
cfg.ReceiveEndpoint("order-placed", e =>
{
    e.ConfigureConsumer<OrderPlacedConsumer>(ctx);
    e.UseMessageRetry(r => r.Intervals(
        TimeSpan.FromSeconds(5),
        TimeSpan.FromSeconds(30),
        TimeSpan.FromMinutes(5)));
});
```

**Why DLQ matters:** Without it, failed messages are lost. DLQ lets you investigate and replay them.

---

## Outbox Pattern

**Problem:** You commit a DB transaction and then publish a message. If the process crashes between those two steps, the message is lost but the DB change is committed → inconsistent state.

**Solution:** In the same DB transaction, write the message to an `OutboxMessages` table. A background worker reads and publishes messages from the outbox. If publishing fails, the outbox still has the message.

```sql
CREATE TABLE OutboxMessages (
    Id UUID PRIMARY KEY,
    Type VARCHAR(200),
    Content TEXT,
    OccurredAt TIMESTAMPTZ,
    ProcessedAt TIMESTAMPTZ NULL
);
```

```csharp
// Same transaction: save domain changes + write to outbox
await _uow.CommitAsync(ct); // saves order + outbox messages atomically

// Background worker (runs every few seconds)
var messages = await _db.OutboxMessages
    .Where(m => m.ProcessedAt == null)
    .OrderBy(m => m.OccurredAt)
    .Take(20)
    .ToListAsync(ct);

foreach (var msg in messages)
{
    await _publisher.PublishAsync(msg.Type, msg.Content, ct);
    msg.ProcessedAt = DateTime.UtcNow;
}
await _db.SaveChangesAsync(ct);
```

**MassTransit has built-in Outbox support.**

---

## RabbitMQ vs Kafka

| | RabbitMQ | Kafka |
|---|---|---|
| Model | Push (broker pushes to consumer) | Pull (consumer pulls from broker) |
| Message retention | Deleted after consumption | Retained for configurable period |
| Ordering | Per-queue, not globally | Per-partition |
| Scale | Thousands of messages/sec | Millions of messages/sec |
| Use case | Task queues, request/response | Event streaming, audit log, replay |
| Complexity | Lower | Higher |

**For your E-commerce app:** RabbitMQ. Simpler, sufficient for the scale.

---

## Async Decoupling Patterns

### Pattern 1: Fire and Forget
Publisher sends and doesn't wait for consumer.
```
Order Service → [queue] → Email Service
```
Use: Side effects (email, SMS, analytics) that don't affect the response.

### Pattern 2: Request/Response via Queue
Publisher sends and waits for response (with correlation ID).
```
API → [request queue] → Pricing Service → [response queue] → API
```
Use: When you need the result but want the benefits of async processing.

### Pattern 3: Competing Consumers
Multiple consumers on the same queue. Message delivered to one consumer.
```
Queue → Consumer 1
      → Consumer 2  (only one gets each message)
      → Consumer 3
```
Use: Scale processing by adding more consumers.

### Pattern 4: Pub/Sub (Fan-out)
One message delivered to all consumers.
```
OrderPlaced → Email Consumer
            → Inventory Consumer
            → Analytics Consumer
```
Use: Broadcasting domain events to multiple interested services.

---

## Common Interview Questions

1. What is the difference between synchronous and asynchronous communication?
2. What are the delivery guarantees (at-most, at-least, exactly-once)?
3. What is a Dead Letter Queue?
4. What is the Outbox pattern and why do you need it?
5. What is the difference between RabbitMQ and Kafka?
6. What does "idempotent consumer" mean?

---

## Common Mistakes

- Not handling poison messages (messages that always fail) → infinite retry loop
- Not implementing idempotency (duplicate processing on retry)
- Putting too much data in the message (use IDs, not full objects — consumer fetches current state)
- Missing DLQ configuration
- Skipping the Outbox pattern (unreliable message publishing)

---

## How It Connects

- Domain Events (DDD) → published via message queue to decouple services
- Outbox Pattern → solves the dual-write problem between DB and queue
- CQRS + Event Sourcing → commands → domain events → message queue → read model updates
- Competing consumers → horizontal scaling of background workers
- DLQ → error handling for async systems (equivalent of try/catch but async)

---

## My Confidence Level
- `[~]` RabbitMQ basics
- `[~]` Producer/consumer pattern
- `[ ]` Dead letter queues
- `[ ]` Outbox pattern + reliable messaging
- `[ ]` At-least-once vs exactly-once delivery

## My Notes
<!-- Personal notes -->
