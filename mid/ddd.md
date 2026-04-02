# Domain-Driven Design (DDD)

## What it is
A software development approach that centers the design on the business domain. The codebase structure mirrors the business vocabulary and rules. Complex business logic lives in the domain model, not in services or controllers.

## Why it matters
Without DDD, business rules scatter across services, controllers, and database triggers — making them hard to find, understand, and change. DDD makes the code speak the language of the business.

---

## Core Building Blocks

### Entities

Objects with a unique **identity** that persists over time. Two entities are equal if their IDs match, regardless of attribute values.

```csharp
public class Order : Entity
{
    public Guid Id { get; private set; }
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    private Order() { } // EF Core requires private constructor

    public static Order Create(Guid userId, Address shippingAddress)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            UserId = userId,
            ShippingAddress = shippingAddress,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };
        order.AddDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }

    public Result AddItem(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending)
            return Result.Failure(ErrorCodes.InvalidOrderStatus);

        var existingItem = _items.FirstOrDefault(i => i.ProductId == product.Id);
        if (existingItem is not null)
            existingItem.IncreaseQuantity(quantity);
        else
            _items.Add(OrderItem.Create(product, quantity));

        return Result.Success();
    }
}
```

**Key rules:**
- Business logic goes on the entity, not in a service
- Private setters — state changes happen through domain methods
- Factory methods (`Create`) enforce invariants on construction

---

### Value Objects

Objects defined by their **attributes**, not identity. Two value objects are equal if all their properties are equal. Immutable.

```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative");
        if (string.IsNullOrWhiteSpace(currency)) throw new DomainException("Currency is required");
        Amount = amount;
        Currency = currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency) throw new DomainException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        return GetEqualityComponents().SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }

    public override int GetHashCode()
        => GetEqualityComponents().Aggregate(0, HashCode.Combine);
}
```

**Examples:** `Money`, `Address`, `Email`, `PhoneNumber`, `DateRange`, `Coordinates`

**In EF Core:** Map value objects as **owned entities**:
```csharp
modelBuilder.Entity<Order>().OwnsOne(o => o.ShippingAddress);
```

---

### Aggregates & Aggregate Roots

An **Aggregate** is a cluster of related entities and value objects treated as a single unit for data changes. The **Aggregate Root** is the entry point — external code only interacts with the root.

**Rules:**
- Only the Aggregate Root is referenced by outside objects
- Objects outside the aggregate hold only the ID of the root, not direct references to child entities
- All invariants within the aggregate are enforced by the root
- One repository per aggregate root

```
Order (Aggregate Root)
  ├── OrderItem (Entity, only accessed through Order)
  └── ShippingAddress (Value Object)
```

**Why:** Ensures consistency. You can't add an `OrderItem` bypassing `Order.AddItem()`.

---

### Domain Events

Something important that happened in the domain. Published after the state change is committed. Used to trigger side effects in a decoupled way.

```csharp
// Core: define the event
public record OrderPlacedEvent(Guid OrderId, Guid UserId, decimal TotalAmount) : IDomainEvent;

// Entity: raise the event
public void Place()
{
    if (Status != OrderStatus.Pending)
        throw new DomainException("Only pending orders can be placed");
    Status = OrderStatus.Placed;
    AddDomainEvent(new OrderPlacedEvent(Id, UserId, CalculateTotal()));
}

// Application: handle the event (send confirmation email, update inventory, etc.)
public class OrderPlacedEventHandler : INotificationHandler<OrderPlacedEvent>
{
    public async Task Handle(OrderPlacedEvent notification, CancellationToken ct)
    {
        // Send confirmation email, trigger inventory update, etc.
    }
}
```

**Dispatch options:**
- **In-process:** dispatch after `SaveChangesAsync` via MediatR `INotificationHandler`
- **Outbox pattern:** persist events to DB in same transaction, dispatch reliably later

---

### Repositories

One repository per Aggregate Root. The repository is an interface in Core, implementation in Infrastructure.

```csharp
// Core interface
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<Order>> GetByUserIdAsync(Guid userId, CancellationToken ct = default);
    void Add(Order order);
    void Update(Order order);
    void Remove(Order order);
}
```

**What repositories do NOT do:**
- Call `SaveChangesAsync` — that's Unit of Work's job
- Return DTOs — they return domain objects
- Contain business logic

---

### Domain Services

When a business operation involves multiple aggregates and doesn't naturally belong to any one entity, use a Domain Service.

```csharp
// Example: transferring money between accounts
public class TransferService
{
    public Result Transfer(BankAccount from, BankAccount to, Money amount)
    {
        var debitResult = from.Debit(amount);
        if (!debitResult.IsSuccess) return debitResult;
        to.Credit(amount);
        return Result.Success();
    }
}
```

---

### Bounded Contexts

A **Bounded Context** is a boundary within which a particular domain model applies. The same word can mean different things in different contexts.

**Example:**
- "Order" in the Ordering context: a purchase with items, shipping, payment
- "Order" in the Fulfillment context: a shipment with warehouse location and tracking

Each bounded context has its own model, database schema, and ubiquitous language.

In a monolith: separate namespaces/modules. In microservices: separate services.

---

### Ubiquitous Language

Use the same terms as the business domain experts, everywhere:
- In class names, method names, variable names
- In conversations with stakeholders
- In the codebase, documentation, tests

If the business calls it a "reservation", don't name it "booking" in the code.

---

## Common Interview Questions

1. What is the difference between an Entity and a Value Object?
2. What is an Aggregate Root and why does it matter?
3. Why do we have one repository per Aggregate Root?
4. What are Domain Events and when do you use them?
5. What is a Bounded Context?

---

## Common Mistakes

- Anemic domain model: entities are just data bags, all logic in services
- Putting database concerns (SQL, EF) in the Core layer
- Repositories returning DTOs instead of domain objects
- Using `public` setters on all entity properties (bypasses domain logic)
- One repository per DB table instead of per aggregate root
- Aggregate roots that are too large (should protect one consistency boundary)

---

## How It Connects

- Clean Architecture: Core layer IS the domain layer
- CQRS: Commands mutate aggregates; Queries bypass the domain model for reads
- Unit of Work: one transaction per aggregate operation
- Domain Events + Outbox: reliable messaging without distributed transactions
- EF Core Owned Entities: how Value Objects are persisted

---

## My Confidence Level
- `[b]` Entities vs Value Objects
- `[b]` Aggregates & aggregate roots
- `[b]` Domain Events
- `[b]` Repository per aggregate root
- `[b]` Unit of Work
- `[~]` Bounded contexts
- `[~]` Ubiquitous language

## My Notes
<!-- Personal notes -->
