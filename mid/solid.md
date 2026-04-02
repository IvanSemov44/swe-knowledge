# SOLID Principles

## What it is
Five object-oriented design principles that make software more maintainable, extensible, and testable.

## Why it matters
Violating SOLID leads to tightly coupled, fragile code where changing one thing breaks five others. Following SOLID makes code easy to extend without modifying existing tested code.

---

## S — Single Responsibility Principle (SRP)

**A class should have only one reason to change.**

One class = one responsibility = one actor who could request a change.

```csharp
// WRONG: OrderService does too many things
public class OrderService
{
    public void PlaceOrder(Order order) { }
    public void SendConfirmationEmail(Order order) { } // email concern
    public void GenerateInvoicePdf(Order order) { }   // PDF concern
    public void UpdateInventory(Order order) { }       // inventory concern
}

// RIGHT: Separate concerns
public class OrderService
{
    public Result PlaceOrder(Order order) { }
}
public class OrderEmailService
{
    public Task SendConfirmationAsync(Order order) { }
}
public class InvoiceService
{
    public byte[] GeneratePdf(Order order) { }
}
```

**In your stack:** Controllers are thin (SRP: just HTTP routing). Command handlers have one job. Domain events decouple side effects (email, inventory) from the order placement.

---

## O — Open/Closed Principle (OCP)

**Software entities should be open for extension, but closed for modification.**

Add new behavior by adding new code, not changing existing tested code.

```csharp
// WRONG: adding a new payment method requires modifying PaymentService
public class PaymentService
{
    public void Process(Order order)
    {
        if (order.PaymentMethod == "credit_card") ProcessCard(order);
        else if (order.PaymentMethod == "paypal") ProcessPayPal(order);
        // Adding "crypto" requires modifying this class
    }
}

// RIGHT: Strategy pattern — add new payment without touching existing code
public interface IPaymentStrategy
{
    bool CanHandle(string paymentMethod);
    Task<PaymentResult> ProcessAsync(Order order);
}

public class CreditCardStrategy : IPaymentStrategy { ... }
public class PayPalStrategy : IPaymentStrategy { ... }
public class CryptoStrategy : IPaymentStrategy { ... } // new, no changes needed

public class PaymentService
{
    private readonly IEnumerable<IPaymentStrategy> _strategies;

    public Task<PaymentResult> ProcessAsync(Order order)
    {
        var strategy = _strategies.First(s => s.CanHandle(order.PaymentMethod));
        return strategy.ProcessAsync(order);
    }
}
```

---

## L — Liskov Substitution Principle (LSP)

**Objects of a subtype must be substitutable for objects of their supertype without breaking the program.**

If `S` is a subtype of `T`, you should be able to use `S` wherever `T` is expected without incorrect behavior.

```csharp
// WRONG: Rectangle/Square violates LSP
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width { set { base.Width = base.Height = value; } }
    public override int Height { set { base.Width = base.Height = value; } }
}

// Code that works for Rectangle breaks for Square:
void Resize(Rectangle r) {
    r.Width = 5;
    r.Height = 4;
    Assert(r.Area == 20); // FAILS for Square (area = 16)
}
```

**Signs of LSP violation:**
- Subclass overrides methods and throws `NotImplementedException`
- Subclass adds preconditions stronger than parent
- `if (obj is SpecificSubtype)` checks in consuming code

**Fix:** Prefer composition over inheritance. Use interfaces for contracts.

---

## I — Interface Segregation Principle (ISP)

**Clients should not be forced to depend on interfaces they don't use.**

Many small, specific interfaces are better than one large general-purpose interface.

```csharp
// WRONG: Fat interface forces implementors to implement things they don't need
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id);
    Task<List<T>> GetAllAsync();
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
    Task<int> CountAsync();
    Task<bool> ExistsAsync(Guid id);
    Task<List<T>> SearchAsync(string term);
}

// RIGHT: Segregated — implement only what you need
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<List<T>> GetAllAsync(CancellationToken ct = default);
}

public interface IWriteRepository<T>
{
    void Add(T entity);
    void Update(T entity);
    void Remove(T entity);
}
```

**In your stack:** `IUnitOfWork` exposes only the repositories it manages. `IOrderRepository` has only order-related methods, not generic CRUD for everything.

---

## D — Dependency Inversion Principle (DIP)

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

```csharp
// WRONG: High-level OrderService depends on low-level concrete EmailSender
public class OrderService
{
    private readonly SmtpEmailSender _emailSender = new SmtpEmailSender(); // concrete!

    public async Task PlaceOrderAsync(Order order)
    {
        // ...
        await _emailSender.SendAsync(order.CustomerEmail, "Order confirmed");
    }
}

// RIGHT: Depend on abstraction
public class OrderService
{
    private readonly IEmailService _emailService; // abstraction

    public OrderService(IEmailService emailService) => _emailService = emailService;

    public async Task PlaceOrderAsync(Order order)
    {
        // ...
        await _emailService.SendAsync(order.CustomerEmail, "Order confirmed");
    }
}

// Infrastructure implements the abstraction
public class SmtpEmailService : IEmailService { ... }

// DI container wires it up
services.AddScoped<IEmailService, SmtpEmailService>();
```

**Why it matters for testing:** You can inject `FakeEmailService` in tests without changing `OrderService`.

---

## SOLID in the E-commerce Project

| Principle | Where applied |
|---|---|
| SRP | Thin controllers, single-purpose handlers, domain events for side effects |
| OCP | Strategy pattern for payment methods, new handlers without touching routing |
| LSP | Interfaces over inheritance for repositories; no `NotImplementedException` |
| ISP | `IOrderRepository` vs `IProductRepository` vs `IUnitOfWork` |
| DIP | All services inject interfaces; Infrastructure implements them; wired in DI |

---

## Common Interview Questions

1. Explain each SOLID principle with an example.
2. How does SOLID relate to Clean Architecture?
3. What is the difference between OCP and LSP?
4. How does Dependency Inversion enable unit testing?
5. Give an example of an ISP violation and how to fix it.

---

## Common Mistakes

- Confusing DIP (principle) with Dependency Injection (technique) — DI is a way to achieve DIP
- Making all interfaces have only one method (ISP doesn't mean "one method per interface")
- Applying LSP only to class inheritance, forgetting interface contracts
- SRP scope creep: making classes so small they have no coherent purpose
- Over-engineering: applying all patterns everywhere — SOLID guides judgment, not mandates abstraction everywhere

---

## How It Connects

- DIP + ISP → Clean Architecture layer interfaces
- OCP → Strategy pattern for extensible business rules
- SRP → Command/Query separation in CQRS
- LSP → Why `IOrderRepository` is safer than inheriting from a base repository class
- DIP → How DI container wires Infrastructure to Application interfaces

---

## My Confidence Level
- `[c]` Single Responsibility
- `[b]` Open/Closed
- `[b]` Liskov Substitution
- `[b]` Interface Segregation
- `[b]` Dependency Inversion

## My Notes
<!-- Personal notes -->
