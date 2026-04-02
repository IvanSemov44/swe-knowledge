# C# Deep Dive

## What it is
My primary backend language. C# is a statically typed, object-oriented language running on the .NET runtime (CLR). It's feature-rich, modern, and has excellent async support.

---

## OOP Fundamentals

### Encapsulation
Private state, public interface. Only expose what's necessary.
```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public void AddItem(OrderItem item) => _items.Add(item); // controlled access
}
```

### Inheritance
Use sparingly. Prefer composition and interfaces.
```csharp
public abstract class Entity
{
    public Guid Id { get; protected set; } = Guid.NewGuid();
    // Shared behavior all entities need
}
public class Order : Entity { }
```

### Polymorphism
One interface, multiple implementations.
```csharp
IPaymentGateway gateway = paymentMethod == "stripe"
    ? new StripeGateway()
    : new PayPalGateway();
await gateway.ProcessAsync(order);
```

### Abstraction
Hide complexity behind an interface. Caller doesn't need to know the implementation.

---

## async/await

### How It Works
`await` suspends the current method without blocking a thread. Control returns to the caller. When the awaited task completes, execution resumes.

```csharp
// Synchronous: blocks thread for 2 seconds
Thread.Sleep(2000);

// Async: releases thread for 2 seconds, resumes after
await Task.Delay(2000);
```

### Task vs Task<T>
- `Task` — async operation with no return value
- `Task<T>` — async operation returning T
- `ValueTask<T>` — lighter weight, use when operation frequently completes synchronously

### ConfigureAwait
```csharp
// In libraries: use ConfigureAwait(false) to avoid deadlocks in non-async contexts
var result = await someTask.ConfigureAwait(false);
// In ASP.NET Core: not needed (no SynchronizationContext)
```

### CancellationToken
Always thread through `CancellationToken`:
```csharp
public async Task<Order?> GetOrderAsync(Guid id, CancellationToken ct = default)
{
    return await _context.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
}
```

### Parallel async
```csharp
// Sequential (total time = t1 + t2)
var product = await _uow.Products.GetByIdAsync(productId, ct);
var user = await _uow.Users.GetByIdAsync(userId, ct);

// Parallel (total time = max(t1, t2))
var productTask = _uow.Products.GetByIdAsync(productId, ct);
var userTask = _uow.Users.GetByIdAsync(userId, ct);
await Task.WhenAll(productTask, userTask);
var product = await productTask;
var user = await userTask;
```

### Anti-patterns
```csharp
// AVOID: async void (unhandled exceptions crash the process)
async void DoSomething() { }

// AVOID: .Result or .Wait() (can deadlock in non-async contexts)
var result = someTask.Result;

// AVOID: unnecessary async wrapping
async Task<int> GetValue() => await Task.FromResult(42); // just return Task.FromResult(42)
```

---

## LINQ

LINQ = Language Integrated Query. Query any IEnumerable<T> with SQL-like syntax.

```csharp
// Method syntax (preferred)
var activeProducts = products
    .Where(p => p.IsActive && p.Price > 10)
    .OrderByDescending(p => p.Price)
    .Select(p => new { p.Name, p.Price })
    .Take(10)
    .ToList();

// Deferred execution — no work done until .ToList(), .First(), etc.
var query = products.Where(p => p.IsActive); // not executed yet
var list = query.ToList(); // executed here

// Immediate execution operators: ToList, ToArray, Count, Sum, First, Any
// Deferred: Where, Select, OrderBy, Skip, Take
```

### Common LINQ Methods
```csharp
.Where(predicate)          // filter
.Select(selector)          // transform/project
.SelectMany(selector)      // flatten nested collections
.OrderBy / OrderByDescending
.ThenBy / ThenByDescending
.GroupBy(key)              // group into IGrouping<K, V>
.First / FirstOrDefault    // first match (throws if empty / returns default)
.Single / SingleOrDefault  // exactly one match
.Any(predicate)            // exists?
.All(predicate)            // all match?
.Count / Sum / Min / Max / Average
.Distinct()                // deduplicate
.Skip(n) / Take(n)         // pagination
.ToList / ToArray / ToDictionary / ToHashSet
.Zip(other, selector)      // pair elements from two sequences
```

### LINQ pitfalls
```csharp
// Multiple enumeration
var query = GetProducts().Where(p => p.IsActive);
var count = query.Count();  // enumerates once
var list = query.ToList();  // enumerates again
// FIX: call .ToList() once and reuse

// Select on EF IQueryable — project before materializing
var names = _context.Products
    .Where(p => p.IsActive)
    .Select(p => p.Name)  // SQL: SELECT Name FROM Products WHERE IsActive = 1
    .ToList();             // NOT: SELECT * then filter in memory
```

---

## Generics

```csharp
// Generic class
public class Result<T>
{
    public T? Value { get; }
    public bool IsSuccess { get; }
    private Result(T? value, bool success) { Value = value; IsSuccess = success; }
    public static Result<T> Success(T value) => new(value, true);
    public static Result<T> Failure() => new(default, false);
}

// Generic constraints
public T GetById<T>(Guid id) where T : Entity, new() { }
public void Process<T>(T item) where T : class, IProcessable { }
public T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) >= 0 ? a : b;
```

---

## Delegates, Func, Action, Events

```csharp
// Func<TInput, TOutput> — delegate that returns a value
Func<int, int, int> add = (a, b) => a + b;
var result = add(3, 4); // 7

// Action<T> — delegate with no return value
Action<string> log = message => Console.WriteLine(message);

// Events (Observer pattern)
public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;
protected virtual void OnOrderPlaced(OrderPlacedEventArgs e)
    => OrderPlaced?.Invoke(this, e);
```

---

## Records

Immutable reference types with value semantics. Perfect for Value Objects, DTOs.

```csharp
// Record: immutable, auto-generated Equals/GetHashCode/ToString
public record Money(decimal Amount, string Currency);

var price = new Money(99.99m, "USD");
var doubled = price with { Amount = price.Amount * 2 }; // non-destructive mutation

// Record struct: value type (stack allocated for small structs)
public record struct Point(int X, int Y);
```

---

## Pattern Matching

```csharp
// switch expression
var message = order.Status switch
{
    OrderStatus.Pending => "Your order is being processed",
    OrderStatus.Shipped => "Your order is on the way",
    OrderStatus.Delivered => "Your order has been delivered",
    _ => "Unknown status"
};

// Type patterns
if (shape is Circle c) { Console.WriteLine(c.Radius); }

// Property patterns
if (order is { Status: OrderStatus.Pending, Total: > 100 })
    ApplyDiscount(order);

// Positional patterns with records
if (point is (0, 0)) Console.WriteLine("Origin");
```

---

## IDisposable and using

```csharp
// Always dispose IDisposable objects
using var connection = new SqlConnection(connectionString);
// connection.Dispose() called automatically at end of scope

// Implement IDisposable for classes holding unmanaged resources
public class FileProcessor : IDisposable
{
    private FileStream? _stream;
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;
        _stream?.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}
```

---

## Common Interview Questions

1. What is the difference between `Task` and `ValueTask`?
2. What happens if you call `.Result` on a Task in an async context?
3. What is deferred execution in LINQ?
4. What is the difference between `First()` and `FirstOrDefault()`?
5. What are generic constraints and why use them?
6. What is the difference between a `class` and a `record`?

---

## My Confidence Level
- `[c]` OOP (classes, interfaces, inheritance, polymorphism)
- `[c]` async/await, Task
- `[c]` LINQ
- `[b]` Generics
- `[b]` Delegates, events, Func/Action
- `[b]` IDisposable, using pattern
- `[~]` Span<T>, Memory<T>
- `[ ]` Source generators

## My Notes
<!-- Personal notes — things you always forget, aha moments -->
