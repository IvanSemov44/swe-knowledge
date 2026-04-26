# C# Nullable Reference Types & Null Handling

## What it is
C# has two null systems: **nullable value types** (`int?`) which have existed since C# 2, and **nullable reference types** (C# 8+) which make the compiler warn you when you might dereference null.

## Why it matters
`NullReferenceException` is the most common runtime crash in .NET. Nullable reference types move this class of bug to compile time. Every modern .NET 6+ codebase uses them.

---

## Nullable Value Types

Value types (`int`, `bool`, `struct`) can't be null by default. Use `?` to allow null.

```csharp
int a = null;   // compile error
int? b = null;  // fine — Nullable<int>

// Check before use
if (b.HasValue)
    Console.WriteLine(b.Value);

// Shorthand
Console.WriteLine(b ?? 0); // use 0 if null

// Pattern matching (preferred)
if (b is int value)
    Console.WriteLine(value); // value is non-null int here
```

---

## Nullable Reference Types (C# 8+ / .NET 6+)

Enabled project-wide in modern .NET. Makes the compiler track nullability of reference types.

```xml
<!-- .csproj — enabled by default in .NET 6+ projects -->
<Nullable>enable</Nullable>
```

```csharp
// Without nullable context — both are the same, no warnings
string name = null;   // fine (bad practice)
string? name = null;  // fine

// WITH nullable context enabled:
string name = null;   // WARNING: cannot assign null to non-nullable
string? name = null;  // fine — explicitly nullable
string name2 = name;  // fine — name2 is non-nullable, guaranteed non-null

// The compiler tracks flow:
string? input = GetInput();
Console.WriteLine(input.Length); // WARNING: may be null

if (input != null)
    Console.WriteLine(input.Length); // fine — compiler knows it's not null here

// OR use null-forgiving operator (use sparingly — you're lying to the compiler)
Console.WriteLine(input!.Length); // tells compiler "trust me, not null"
```

---

## Null Operators

```csharp
// ?. — null-conditional: returns null instead of throwing
string? name = user?.Name;           // null if user is null
int? length = user?.Name?.Length;    // null if either is null
user?.Address?.City?.ToUpper();      // chained — safe

// ?? — null-coalescing: default if null
string display = name ?? "Anonymous";
int count = nullableCount ?? 0;

// ??= — null-coalescing assignment: assign only if null
_cache ??= new Dictionary<string, string>(); // initialize if not already set
user.Name ??= "Default Name";

// ! — null-forgiving operator: suppresses null warning
// Only use when YOU know it can't be null but the compiler can't prove it
var entity = _context.Find<Order>(id)!; // you KNOW it exists after a check
```

---

## Patterns for Null Checks

```csharp
// is null / is not null — preferred over == null (can't be overloaded)
if (user is null) throw new ArgumentNullException(nameof(user));
if (user is not null) Process(user);

// Pattern matching with type
if (result is Product p)
    Console.WriteLine(p.Name); // p is non-null Product here

// Switch expression with null case
var message = status switch
{
    null       => "Unknown",
    "active"   => "User is active",
    "inactive" => "User is inactive",
    _          => "Unexpected status"
};

// ArgumentNullException.ThrowIfNull (NET 6+) — cleaner than manual check
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);
    // proceed — order is guaranteed non-null
}
```

---

## Required Members (C# 11 / .NET 7+)

```csharp
// required — compiler ensures property is set in object initializer
public class CreateProductRequest
{
    public required string Name { get; init; }
    public required decimal Price { get; init; }
    public string? Description { get; init; } // optional
}

var request = new CreateProductRequest
{
    Name = "Sneakers",
    Price = 89.99m
    // Description is optional — fine to omit
};

var bad = new CreateProductRequest(); // compile error — Name and Price required
```

---

## Nullable in Your Domain

```csharp
// Value Object — use nullable correctly
public record Address(
    string Street,      // required — never null
    string City,        // required
    string? Apartment); // optional floor/unit

// Entity properties
public class Order : Entity
{
    public DateTime? ShippedAt { get; private set; }  // null until shipped
    public string? TrackingNumber { get; private set; } // null until shipped

    public Result Ship(string trackingNumber)
    {
        ArgumentNullException.ThrowIfNull(trackingNumber);
        if (Status != OrderStatus.Confirmed)
            return Result.Failure(ErrorCodes.InvalidOrderStatus);

        ShippedAt = DateTime.UtcNow;
        TrackingNumber = trackingNumber;
        Status = OrderStatus.Shipped;
        return Result.Success();
    }
}

// Repository return types
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default); // null if not found
    Task<IReadOnlyList<Order>> GetByUserAsync(Guid userId, CancellationToken ct = default); // never null, empty list
}
```

---

## Common Interview Questions

1. What is the difference between `int?` and `string?`?
2. What does `#nullable enable` do?
3. What is the null-forgiving operator (`!`) and when should you use it?
4. What is the difference between `??` and `??=`?
5. Why is `is null` preferred over `== null`?
6. What does `required` do on a property?

---

## Common Mistakes

- Using `!` (null-forgiving) frequently — it disables protection; use only when you have proof
- Not enabling `<Nullable>enable</Nullable>` — you lose all compile-time null safety
- Returning `null` from methods instead of empty collections (`return null` vs `return []`)
- Checking `== null` instead of `is null` (can be overloaded, less reliable)
- Making all strings `string?` "to be safe" — defeats the purpose; use `string` when guaranteed non-null

---

## How It Connects

- Repository methods return `T?` when something might not exist, `T` when always present
- Command handlers check for null after repository calls and return `Result.Failure(ErrorCodes.NotFound)`
- DTOs use `required` properties for mandatory fields
- Value Objects use non-nullable properties for required data, `?` for optional
- `ArgumentNullException.ThrowIfNull` at the start of every public method that accepts objects

---

## My Confidence Level
- `[ ]` Nullable value types — `int?`, HasValue, Value
- `[ ]` Nullable reference types — `#nullable enable`, what the compiler tracks
- `[ ]` Null operators — `?.`, `??`, `??=`, `!`
- `[ ]` Null check patterns — `is null`, `is not null`, pattern matching
- `[ ]` required members — compile-time enforcement
- `[ ]` Nullable in domain models — when `T?` vs `T`

## My Notes
<!-- Personal notes -->
