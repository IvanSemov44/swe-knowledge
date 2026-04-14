# Shared Kernel

## What it is
The Shared Kernel is a small, stable project shared across all bounded contexts. It contains the building blocks every domain object inherits from — no business logic, just infrastructure primitives.

**Rule:** Shared Kernel must have zero dependencies on any bounded context. It only contains abstractions.

---

## Project Structure

```
src/
  Shared.Kernel/
    Domain/
      AggregateRoot.cs
      Entity.cs
      ValueObject.cs
      IDomainEvent.cs
      IIntegrationEvent.cs
    Application/
      Result.cs
      ErrorCodes.cs  (or per-context)
      IUnitOfWork.cs
```

---

## Entity Base Class

```csharp
public abstract class Entity
{
    public Guid Id { get; protected set; }

    protected Entity() { }
    protected Entity(Guid id) => Id = id;

    public override bool Equals(object? obj)
    {
        if (obj is not Entity other) return false;
        if (ReferenceEquals(this, other)) return true;
        if (GetType() != other.GetType()) return false;
        return Id == other.Id;
    }

    public override int GetHashCode() => Id.GetHashCode();

    public static bool operator ==(Entity? left, Entity? right)
        => left?.Equals(right) ?? right is null;

    public static bool operator !=(Entity? left, Entity? right)
        => !(left == right);
}
```

---

## AggregateRoot Base Class

```csharp
public abstract class AggregateRoot : Entity
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected AggregateRoot() { }
    protected AggregateRoot(Guid id) : base(id) { }

    protected void AddDomainEvent(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents()
        => _domainEvents.Clear();
}
```

---

## ValueObject Base Class

```csharp
public abstract class ValueObject
{
    protected abstract IEnumerable<object> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType()) return false;
        return GetEqualityComponents()
            .SequenceEqual(((ValueObject)obj).GetEqualityComponents());
    }

    public override int GetHashCode()
        => GetEqualityComponents().Aggregate(0, HashCode.Combine);

    public static bool operator ==(ValueObject? left, ValueObject? right)
        => left?.Equals(right) ?? right is null;

    public static bool operator !=(ValueObject? left, ValueObject? right)
        => !(left == right);
}
```

**Example Value Object:**
```csharp
public class Money : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new DomainException("Amount cannot be negative");
        Amount = amount;
        Currency = currency.ToUpperInvariant();
    }

    protected override IEnumerable<object> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainException("Cannot add different currencies");
        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount} {Currency}";
}
```

---

## IDomainEvent

In-process events. Raised by aggregates, dispatched by MediatR after SaveChangesAsync.

```csharp
// Shared.Kernel
public interface IDomainEvent : INotification // MediatR
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
}

// Base record for convenience
public abstract record DomainEvent : IDomainEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

// Example in Catalog.Domain
public record ProductPriceChangedEvent(
    Guid ProductId,
    decimal OldPrice,
    decimal NewPrice
) : DomainEvent;
```

---

## IIntegrationEvent

Cross-boundary events. Published to RabbitMQ via MassTransit after domain event is handled.

```csharp
// Shared.Kernel
public interface IIntegrationEvent
{
    Guid EventId { get; }
    DateTime OccurredAt { get; }
    string EventType { get; }
}

public abstract record IntegrationEvent : IIntegrationEvent
{
    public Guid EventId { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
    public string EventType => GetType().Name;
}

// Example in Orders.Application (or Shared.Contracts)
public record OrderPlacedIntegrationEvent(
    Guid OrderId,
    Guid UserId,
    decimal TotalAmount,
    List<OrderItemDto> Items
) : IntegrationEvent;
```

---

## Result<T> Pattern

No throwing exceptions for expected business failures.

```csharp
public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string Error { get; }

    protected Result(bool isSuccess, string error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new(true, string.Empty);
    public static Result Failure(string error) => new(false, error);

    public static Result<T> Success<T>(T value) => new(value, true, string.Empty);
    public static Result<T> Failure<T>(string error) => new(default!, false, error);
}

public class Result<T> : Result
{
    public T Value { get; }

    internal Result(T value, bool isSuccess, string error)
        : base(isSuccess, error)
    {
        Value = value;
    }
}
```

---

## IUnitOfWork

```csharp
// Shared.Kernel or per-module boundary
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

// Implementation wraps DbContext
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context) => _context = context;

    public Task<int> SaveChangesAsync(CancellationToken ct = default)
        => _context.SaveChangesAsync(ct);
}
```

---

## ErrorCodes

Never use magic strings for business errors.

```csharp
public static class ErrorCodes
{
    public static class Catalog
    {
        public const string ProductNotFound = "catalog.product_not_found";
        public const string CategoryNotFound = "catalog.category_not_found";
        public const string DuplicateSku = "catalog.duplicate_sku";
    }

    public static class Orders
    {
        public const string OrderNotFound = "orders.order_not_found";
        public const string InvalidOrderStatus = "orders.invalid_status";
        public const string InsufficientStock = "orders.insufficient_stock";
    }

    public static class Identity
    {
        public const string InvalidCredentials = "identity.invalid_credentials";
        public const string EmailAlreadyExists = "identity.email_exists";
    }
}
```

---

## What Goes in Shared Kernel vs What Doesn't

| Belongs in Shared Kernel | Does NOT belong |
|---|---|
| `AggregateRoot`, `Entity`, `ValueObject` | Business logic |
| `IDomainEvent`, `IIntegrationEvent` | Domain models |
| `Result<T>`, `ErrorCodes` | Infrastructure (EF, RabbitMQ) |
| `IUnitOfWork` interface | Application services |
| Common Value Objects (`Money`, `Email`) | Bounded context DTOs |

---

## Common Interview Questions

1. What is the Shared Kernel in DDD?
2. Why does AggregateRoot hold domain events?
3. What is the difference between IDomainEvent and IIntegrationEvent?
4. Why use Result<T> instead of throwing exceptions?
5. Where should ErrorCodes live?

---

## How It Connects

- Every aggregate inherits `AggregateRoot` → gets `AddDomainEvent` + `DomainEvents`
- `SaveChangesAsync` override reads `DomainEvents` from all tracked aggregates → dispatches via MediatR
- Domain event handlers publish `IntegrationEvent` to the outbox
- MassTransit reads outbox → publishes to RabbitMQ
- `Result<T>` flows from handler → controller → API response

---

## My Confidence Level
- `[ ]` Entity vs AggregateRoot base class implementation
- `[ ]` ValueObject equality implementation
- `[ ]` IDomainEvent vs IIntegrationEvent — what each is for
- `[ ]` Result<T> pattern — implement from scratch
- `[ ]` IUnitOfWork — interface + implementation
- `[ ]` ErrorCodes — structure and usage

## My Notes
<!-- Personal notes -->
