# Design Patterns (GoF)

## What it is
Proven, reusable solutions to commonly occurring software design problems. The "Gang of Four" (GoF) catalog contains 23 patterns split into Creational, Structural, and Behavioral categories.

## Why it matters
Patterns give you a shared vocabulary with other developers and proven blueprints for complex design decisions. Recognizing which pattern solves a problem saves hours of reinventing.

---

## Creational Patterns (object creation)

### Factory Method
Define an interface for creating an object, but let subclasses decide which class to instantiate.

```csharp
// Base creator
public abstract class NotificationFactory
{
    public abstract INotification Create(string type);
}

// Concrete creator
public class EmailNotificationFactory : NotificationFactory
{
    public override INotification Create(string type) => new EmailNotification(type);
}
```

**Use:** When you don't know which subtype to create until runtime.

---

### Abstract Factory
Create families of related objects without specifying their concrete classes.

```csharp
public interface IPaymentFactory
{
    IPaymentProcessor CreateProcessor();
    IPaymentValidator CreateValidator();
}

public class StripePaymentFactory : IPaymentFactory
{
    public IPaymentProcessor CreateProcessor() => new StripeProcessor();
    public IPaymentValidator CreateValidator() => new StripeValidator();
}
```

**Use:** When you need families of related products and want to enforce consistency.

---

### Builder
Construct complex objects step by step. The same construction process can create different representations.

```csharp
var email = new EmailBuilder()
    .To("user@example.com")
    .Subject("Order Confirmation")
    .Body("Your order #123 has been placed.")
    .WithAttachment(invoice)
    .Build();
```

**Use:** Objects with many optional parameters. Avoids telescoping constructors.

**In .NET:** `StringBuilder`, `IHostBuilder`, `WebApplicationBuilder`

---

### Singleton
Ensure a class has only one instance and provide global access to it.

```csharp
// Preferred in .NET: use DI container with Singleton lifetime
services.AddSingleton<IConfiguration>(config);
// NOT manual static Singleton — it's an anti-pattern in modern DI-based code
```

**Use:** Logger, configuration, connection pool.
**Caution:** Makes unit testing hard (global state). Prefer DI Singleton over manual implementation.

---

## Structural Patterns (composing objects)

### Repository
Mediates between the domain and data mapping layers. Provides a collection-like interface for accessing domain objects.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default);
    void Add(Order order);
    void Remove(Order order);
}
```

**Use:** Abstract persistence logic from domain logic. Enables testing with in-memory fakes.

---

### Adapter
Convert the interface of a class into another interface that clients expect. Let incompatible interfaces work together.

```csharp
// Third-party payment gateway has its own interface
public class StripeGateway
{
    public StripeCharge ChargeCard(string cardToken, long amountCents) { ... }
}

// Your domain expects IPaymentGateway
public class StripeAdapter : IPaymentGateway
{
    private readonly StripeGateway _stripe;

    public Task<PaymentResult> ProcessPayment(PaymentRequest request)
    {
        var charge = _stripe.ChargeCard(request.Token, (long)(request.Amount * 100));
        return Task.FromResult(new PaymentResult(charge.Id, charge.Status));
    }
}
```

**Use:** Integrating third-party libraries with your domain interfaces.

---

### Decorator
Attach additional responsibilities to an object dynamically. Wraps another object to add behavior.

```csharp
public class CachingProductRepository : IProductRepository
{
    private readonly IProductRepository _inner;
    private readonly IMemoryCache _cache;

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await _inner.GetByIdAsync(id, ct);
        });
    }
}
```

**Use:** Adding cross-cutting concerns (caching, logging, retry) without modifying the original class.

---

### Facade
Provide a simplified interface to a complex subsystem.

```csharp
public class OrderFacade
{
    // Hides complexity of coordinating inventory, payment, shipping, notifications
    public async Task<OrderResult> PlaceOrder(PlaceOrderRequest request)
    {
        await _inventory.Reserve(request.Items);
        var payment = await _payment.Charge(request.PaymentMethod, request.Total);
        var shipment = await _shipping.Schedule(request.Address);
        await _notifications.SendConfirmation(request.UserId);
        return new OrderResult(payment.Id, shipment.TrackingNumber);
    }
}
```

**Use:** Simplify complex subsystems for client code.

---

## Behavioral Patterns (communication between objects)

### Strategy
Define a family of algorithms, encapsulate each one, and make them interchangeable.

```csharp
public interface IDiscountStrategy
{
    decimal Apply(decimal price, Order order);
}

public class PercentageDiscount : IDiscountStrategy
{
    public decimal Apply(decimal price, Order order) => price * 0.9m; // 10% off
}

public class BulkDiscount : IDiscountStrategy
{
    public decimal Apply(decimal price, Order order)
        => order.Quantity >= 10 ? price * 0.85m : price;
}

// Usage
public class PricingService
{
    private readonly IDiscountStrategy _strategy;
    public decimal Calculate(Order order) => _strategy.Apply(order.BasePrice, order);
}
```

**Use:** When you need to switch algorithms at runtime. Tax calculation, sorting, payment methods.

---

### Observer
Define a one-to-many dependency. When one object changes state, all dependents are notified automatically.

```csharp
// .NET Events are Observer pattern
public class Order
{
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    public void Place()
    {
        // ...
        OrderPlaced?.Invoke(this, new OrderPlacedEventArgs(Id));
    }
}
```

**In modern .NET:** Domain Events + MediatR `INotification`/`INotificationHandler` = Observer pattern.

**Use:** Decoupling side effects from core logic. Event-driven systems.

---

### Command Pattern
Encapsulate a request as an object, allowing parameterization and queuing of requests.

```csharp
// MediatR IRequest<T> IS the Command pattern
public record CreateOrderCommand(Guid UserId, Guid ProductId) : IRequest<Result<Guid>>;
```

**Use:** Undo/redo, queuing operations, audit logs. In ASP.NET: CQRS commands.

---

### Chain of Responsibility
Pass a request along a chain of handlers. Each handler decides to process or pass to the next.

```csharp
// MediatR Pipeline Behaviors ARE Chain of Responsibility
// Logging → Validation → Transaction → Handler
```

**Use:** Middleware pipelines (ASP.NET Core middleware, MediatR behaviors).

---

### Template Method
Define the skeleton of an algorithm in a base class, letting subclasses override specific steps.

```csharp
public abstract class ReportGenerator
{
    public void Generate() // Template method
    {
        var data = FetchData();     // abstract
        var formatted = Format(data); // abstract
        Export(formatted);          // concrete
    }

    protected abstract IEnumerable<object> FetchData();
    protected abstract string Format(IEnumerable<object> data);
    protected virtual void Export(string content) => File.WriteAllText("report.txt", content);
}
```

---

## Pattern Quick Reference

| Pattern | Category | Use When |
|---|---|---|
| Factory Method | Creational | Unknown subtype at compile time |
| Abstract Factory | Creational | Families of related objects |
| Builder | Creational | Complex object construction |
| Singleton | Creational | Single instance needed (prefer DI) |
| Repository | Structural | Abstract persistence |
| Adapter | Structural | Incompatible interfaces |
| Decorator | Structural | Add behavior without subclassing |
| Facade | Structural | Simplify complex subsystem |
| Strategy | Behavioral | Interchangeable algorithms |
| Observer | Behavioral | Event-driven dependencies |
| Command | Behavioral | Encapsulate requests as objects |
| Chain of Responsibility | Behavioral | Pipeline processing |
| Template Method | Behavioral | Algorithm skeleton with variable steps |

---

## Common Interview Questions

1. What is the difference between Factory Method and Abstract Factory?
2. How is the Decorator pattern different from inheritance?
3. Give a real example of the Strategy pattern.
4. What pattern does MediatR implement?
5. What pattern does ASP.NET Core middleware implement?

---

## Common Mistakes

- Overusing patterns where simple code would be clearer
- Using Singleton for objects that should be per-request (use DI lifetimes)
- Confusing Decorator with inheritance (Decorator wraps at runtime, inheritance is compile-time)
- Implementing Observer with `static` events (hard to test, memory leaks)

---

## How It Connects

- Repository + Unit of Work: used in Clean Architecture / DDD
- Strategy: used for payment processing, tax calculation, discount rules
- Decorator: used for caching repositories, retry policies
- Observer: Domain Events via MediatR
- Chain of Responsibility: MediatR pipeline behaviors, ASP.NET middleware
- Command: CQRS commands via MediatR
- Builder: TestDataBuilders in unit tests

---

## My Confidence Level
- `[b]` Factory / Abstract Factory
- `[b]` Builder
- `[b]` Singleton
- `[b]` Repository
- `[b]` Strategy
- `[b]` Observer
- `[b]` Decorator
- `[~]` Adapter
- `[~]` Facade
- `[ ]` Chain of Responsibility (explicit)
- `[ ]` Template Method

## My Notes
<!-- Personal notes -->
