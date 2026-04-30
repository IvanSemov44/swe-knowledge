# Architecture Patterns
<!-- level: Mid → Senior | Event Sourcing, Hexagonal Architecture, Vertical Slice Architecture, Context Mapping, ADRs -->

## Event Sourcing

**Core idea:** Instead of storing current state, store a sequence of immutable events. Current state is derived by replaying those events.

Traditional DB: `Orders` row has `Status = "Shipped"`, `Total = 99.00`. You see the end result; the history is gone.

Event store: `[OrderPlaced, OrderPaid, OrderShipped]` — every state change is recorded. You can reconstruct the order at any point in time.

### Event Store

An append-only log. Two common implementations:
- **EventStoreDB** — purpose-built; native stream concept, server-side projections, catch-up subscriptions
- **Postgres table** — simpler to operate; add columns `StreamId`, `EventType`, `Payload JSONB`, `SequenceNumber`, `OccurredAt`; query with `SELECT ... WHERE StreamId = @id ORDER BY SequenceNumber`

You never UPDATE or DELETE rows. Ever.

### Rebuilding State

Each aggregate exposes private `Apply` methods — one per event type. The repository loads all events for the stream and calls `Apply` for each.

```csharp
public class Order : AggregateRoot
{
    public OrderStatus Status { get; private set; }
    public Money Total { get; private set; }

    // Called when loading from event store
    private void Apply(OrderPlaced e) => (Status, Total) = (OrderStatus.Pending, e.Total);
    private void Apply(OrderShipped e) => Status = OrderStatus.Shipped;
    private void Apply(OrderCancelled e) => Status = OrderStatus.Cancelled;

    public static Order Load(IEnumerable<IDomainEvent> history)
    {
        var order = new Order();
        foreach (var e in history) order.ApplyEvent(e);
        return order;
    }
}
```

`ApplyEvent` on the base `AggregateRoot` dispatches to the correct overload via reflection or a dictionary.

### Snapshots

Replaying 50,000 events on every load is slow. After N events (e.g., 100), save a snapshot of the current state. On next load: fetch snapshot + only events after its sequence number, then replay the delta.

Snapshots are an optimization, not required for correctness.

### Event Versioning

Event schemas change over time. When `OrderPlacedV1` gains a new required field `Currency`:

```csharp
// Upcaster — transforms V1 → V2 before Apply()
public class OrderPlacedV2Upcaster : IUpcaster<OrderPlacedV1>
{
    public IDomainEvent Upcast(OrderPlacedV1 old) =>
        new OrderPlacedV2(old.OrderId, old.CustomerId, old.Total, Currency: "BGN");
}
```

Upcasters run in the repository before events reach `Apply`. Old events stay in the store unchanged; the upcaster is the migration layer.

### CQRS + Event Sourcing as a Natural Pair

Write side: command handler loads aggregate from event store → calls domain method → aggregate raises new events → repository appends events.

Read side: a separate projection listens to the event stream and updates a read-optimized table (e.g., `OrderSummaries`). Queries hit the read table directly — no event replay needed at query time.

### When NOT to Use Event Sourcing

- Simple CRUD (product catalog, config tables) — complexity is not worth it
- Report-heavy systems where temporal queries are rare
- Team unfamiliar with the pattern — operational burden is real
- No audit trail requirement

### When TO Use

- Financial systems (every transaction must be auditable)
- Audit trail required by law or business policy
- Temporal queries: "what was the account balance on March 1?"
- Event-driven integration: events are the natural integration mechanism between bounded contexts

---

## Hexagonal Architecture (Ports and Adapters)

**Author:** Alistair Cockburn. **Core idea:** The application core has zero dependency on infrastructure. All I/O (DB, HTTP, email, message broker) goes through ports — interfaces the core defines.

### Ports

**Primary ports (driving)** — how the outside world talks to your app. Your application *implements* these.
Examples: `IOrderService`, `IRequestHandler<PlaceOrderCommand>`

**Secondary ports (driven)** — how your app talks to the outside world. Your app *declares* these interfaces; adapters implement them.
Examples: `IOrderRepository`, `IEmailGateway`, `IPaymentProvider`

### Adapters

Concrete implementations of secondary ports, living in the Infrastructure layer.

```
  [REST Controller]  ──→  IOrderService (primary port)
  [CLI Command]      ──→        │
                          [Order Application Core]
                                │
        IOrderRepository ←────  │  ────→  IEmailGateway
               │                              │
        [SqlOrderRepository]         [SendGridEmailAdapter]
        (Infrastructure)              (Infrastructure)
```

```csharp
// Secondary port — lives in Application or Core
public interface IEmailGateway
{
    Task SendOrderConfirmationAsync(string to, OrderId orderId);
}

// Adapter — lives in Infrastructure
public class SendGridEmailAdapter(SendGridClient client) : IEmailGateway
{
    public async Task SendOrderConfirmationAsync(string to, OrderId orderId)
    {
        var msg = MailHelper.CreateSingleEmail(
            from: new EmailAddress("no-reply@shop.com"),
            to: new EmailAddress(to),
            subject: $"Order {orderId} confirmed",
            plainTextContent: "Thanks for your order.",
            htmlContent: "<strong>Thanks for your order.</strong>");
        await client.SendEmailAsync(msg);
    }
}
```

### Hexagonal vs Clean Architecture

Same dependency direction — core has no outward dependencies. Different vocabulary:

| | Hexagonal | Clean Architecture |
|---|---|---|
| Center | Application core | Domain + Application layers |
| Inbound boundary | Primary port | Controller → Service interface |
| Outbound boundary | Secondary port | Repository/gateway interface |
| Layer names | None (just ports/adapters) | API, Application, Core, Infrastructure |

Clean Architecture is Hexagonal Architecture with explicit layer naming conventions added. They are not competing patterns.

**Key benefit of Hexagonal thinking:** Swap any adapter without touching the core — replace SendGrid with AWS SES, SQL Server with Cosmos DB, HTTP controller with a CLI or a message consumer. For tests, replace all infrastructure adapters with in-memory fakes.

---

## Vertical Slice Architecture

**Author:** Jimmy Bogard. **Core idea:** Organize code by feature (a vertical slice through all layers), not by technical role (layer).

### The Problem with Horizontal Layers

Traditional structure forces you to touch `Application/`, `Core/`, `Infrastructure/` for every feature. A simple "get orders" query drags through the same interface/repository/service ceremony as a complex domain operation.

### Structure

```
Features/
├── Orders/
│   ├── PlaceOrder/
│   │   ├── PlaceOrderCommand.cs
│   │   ├── PlaceOrderHandler.cs
│   │   ├── PlaceOrderValidator.cs
│   │   └── PlaceOrderResponse.cs
│   └── GetOrders/
│       ├── GetOrdersQuery.cs
│       ├── GetOrdersHandler.cs
│       └── OrderSummaryDto.cs
└── Catalog/
    └── GetProduct/
        ├── GetProductQuery.cs
        └── GetProductHandler.cs
```

Each slice owns everything it needs. No shared abstractions unless multiple slices actually need them — defer to the rule of three.

### Read Slices Can Skip the Repository

A query slice that just reads data doesn't need a repository or domain entity. Query the DB directly.

```csharp
// GetOrdersHandler — no repository, direct Dapper query is fine for a read slice
public class GetOrdersHandler(SqlConnection db)
    : IRequestHandler<GetOrdersQuery, List<OrderSummaryDto>>
{
    public async Task<List<OrderSummaryDto>> Handle(GetOrdersQuery q, CancellationToken ct)
    {
        return (await db.QueryAsync<OrderSummaryDto>(
            "SELECT Id, Status, Total FROM Orders WHERE CustomerId = @CustomerId",
            new { q.CustomerId })).ToList();
    }
}
```

Write slices that mutate domain state still benefit from aggregate + repository — use them there.

### VSA vs Clean Architecture: When to Choose

| Situation | Prefer |
|---|---|
| Greenfield, solo or small team | Vertical Slice |
| Large team with separate layer owners | Clean Architecture |
| Complex domain with rich DDD aggregates | Clean Architecture |
| Mostly CRUD, rapid iteration priority | Vertical Slice |
| Need to enforce dependency rules strictly | Clean Architecture |

They coexist: use VSA directory structure, but keep a shared `Core/` project for domain entities. MediatR works with both.

---

## Context Mapping (DDD Strategic)

When multiple bounded contexts (BCs) need to interact, how they integrate matters as much as what they do. Context Mapping names the relationship patterns.

### Relationship Patterns

| Pattern | Description | Example |
|---|---|---|
| **Partnership** | Two teams coordinate and release together | Shared sprint, joint API contract |
| **Shared Kernel** | Small shared codebase both teams own and change together | `Shared.Kernel` NuGet with base Value Objects |
| **Customer/Supplier** | Upstream provides; downstream is customer with influence | Orders BC (U) publishes events; Shipping BC (D) shapes them |
| **Conformist** | Downstream accepts upstream's model as-is; no influence | Third-party payment API — you conform to their schema |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model into its own domain model | Adapter/translator; upstream types never leak into your domain |
| **Open Host Service** | Upstream publishes a well-defined protocol for any consumer | REST API, gRPC service, event schema |
| **Published Language** | Shared formal language, often paired with Open Host | OpenAPI spec, AsyncAPI schema, Protobuf |
| **Separate Ways** | No integration; each BC solves the problem independently | Duplication is cheaper than coupling |

### ACL in Practice

In Ivan's e-commerce app: the Ordering BC creates its own `ShippingAddress` Value Object. It never lets the Shipping BC's `AddressDto` type bleed into the Ordering domain model. The ACL translates at the boundary.

```csharp
// Anti-Corruption Layer in Ordering BC
// Translates the Shipping BC's DTO into Ordering's own Value Object
public static class ShippingAddressTranslator
{
    public static ShippingAddress ToDomain(ShippingAddressDto dto) =>
        ShippingAddress.Create(dto.Street, dto.City, dto.PostalCode, dto.Country).Value;
}

// Usage in a command handler — dto comes from the Shipping BC integration event
var address = ShippingAddressTranslator.ToDomain(integrationEvent.Address);
order.SetShippingAddress(address);
```

If the Shipping BC renames `PostalCode` to `ZipCode`, only the ACL translator changes — zero impact on the Ordering domain.

### Why This Matters Operationally

Conformist is often the easiest short-term choice but the worst long-term. You inherit every bad decision the upstream team makes. ACL costs more upfront but protects your domain model's integrity. Use Conformist only for third-party APIs you can't influence.

---

## Architecture Decision Records (ADRs)

**What:** A short document capturing an architectural decision, the context that drove it, and its consequences — including trade-offs.

**Why:** Architectural decisions made today are invisible to the team six months from now. New members spend days or weeks reverse-engineering "why is this done this way?" ADRs replace tribal knowledge.

### Format (Nygard's, most common)

```markdown
# ADR-001: Use Outbox Pattern for Integration Event Reliability

## Status
Accepted

## Context
Publishing integration events directly after SaveChangesAsync can fail if the message broker
is down at that moment. This causes inconsistency: DB state is committed but downstream BCs
never receive the event.

## Decision
Use the Transactional Outbox pattern. Write events to an OutboxMessages table in the same
DB transaction as domain state changes. A BackgroundService reads unpublished messages and
publishes them to the broker asynchronously.

## Consequences
- Positive: guaranteed at-least-once delivery; no event loss on broker downtime
- Positive: domain DB transaction is always internally consistent
- Negative: events are delivered asynchronously; slight latency increase
- Negative: OutboxDispatcher adds operational complexity (needs monitoring)
```

### Practical Details

- **Location:** `/docs/adr/` in the repo, committed alongside code — not in a wiki
- **Naming:** `0001-use-outbox-pattern.md`, `0002-choose-vertical-slice.md` (sequential, lowercase, hyphenated)
- **Status values:** `Proposed` → `Accepted` | `Rejected` | `Superseded by ADR-NNN`
- **When to write:** Any decision that is hard to reverse, that future teammates will question, or that involves non-obvious trade-offs. Rule of thumb: if you had to discuss it for more than 30 minutes, write an ADR.
- **Tooling:** `adr-tools` CLI scaffolds new ADRs; or plain Markdown files work fine

---

## Clean Architecture vs Hexagonal vs Vertical Slice — Decision Table

| Criterion | Clean Architecture | Hexagonal | Vertical Slice |
|---|---|---|---|
| Team size | Medium to large | Any | Small to medium |
| Domain complexity | High (DDD) | High (DDD) | Low to medium |
| Strict dependency enforcement | Yes (by layer) | Yes (by port) | No (by convention) |
| Iteration speed | Slower (more ceremony) | Medium | Fast |
| Testability | High (interfaces at all boundaries) | High (swap adapters freely) | High (slice per feature test) |
| Learning curve | High | Medium | Low |
| Best for | Enterprise, large domains | Plugin-heavy, adapter-swap heavy | CRUD services, rapid prototyping |

---

## Key Interview Questions

1. **What problem does Event Sourcing solve that a traditional DB update does not?**
   It preserves full history — you can reconstruct state at any point in time and answer "what happened and when," not just "what is the state now."

2. **What is the difference between a primary port and a secondary port in Hexagonal Architecture?**
   Primary ports are driven by the outside world (REST, CLI call your app); secondary ports are driven by your app to reach external systems (DB, email, payment); your app defines both interfaces, but implements only primary ports.

3. **Why might you choose Vertical Slice Architecture over Clean Architecture for a new project?**
   Smaller team, simpler domain, need for rapid iteration — VSA avoids ceremony overhead; each feature is self-contained and doesn't require touching multiple layers for every change.

4. **What is an Anti-Corruption Layer and when do you need one?**
   A translation layer that converts an upstream bounded context's model into your own domain model, preventing upstream types from polluting your domain; needed any time you integrate with a context whose model you don't control or don't want to conform to.

5. **What goes in an ADR and why commit it to the repo rather than a wiki?**
   Context (why the decision was needed), the decision itself, and consequences (trade-offs); in the repo because it versions alongside the code — when the code changes, the ADR history explains why it changed, and it stays in sync rather than rotting in a separate wiki.
