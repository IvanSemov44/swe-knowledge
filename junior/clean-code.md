# Clean Code

## What it is
Principles for writing code that is easy to read, understand, and maintain. Not about correctness — about the human experience of working with the code.

## Why it matters
Interviewers ask about this directly ("what does clean code mean to you?") and assess it indirectly through your code samples. In a team, clean code is the difference between a codebase that stays maintainable and one that becomes a rewrite candidate in 18 months.

---

## Naming

Names should reveal intent. If you need a comment to explain a name, the name is wrong.

```csharp
// BAD — what does d mean? What unit?
int d = 14;
var list = GetList();

// GOOD — self-documenting
int deliveryWindowDays = 14;
var overdueOrders = GetOverdueOrders();

// BAD — meaningless generic names
void ProcessData(List<object> data) { }

// GOOD — specific and intent-revealing
void ApplyDiscounts(List<Order> pendingOrders) { }

// BAD — noise words that add nothing
Product theProduct;
string productNameString;
List<Order> orderList;

// GOOD — concise and meaningful
Product product;
string productName;
List<Order> orders;

// BAD — abbreviations that require context
int cnt;
bool flg;
string addr;

// GOOD
int orderCount;
bool isActive;
string shippingAddress;
```

**Naming conventions by context:**
```csharp
// Booleans — read as a question
bool isActive, hasDiscount, canShip, wasProcessed;

// Methods — verb + noun
void ProcessOrder(), bool ValidatePayment(), List<Order> GetPendingOrders()

// Classes — noun
class OrderProcessor, class PaymentGateway, class EmailNotification

// Constants — UPPER_SNAKE or PascalCase
const int MAX_RETRY_COUNT = 3;
const string DefaultCurrency = "USD";
```

---

## Functions / Methods

**Single Responsibility:** A function should do one thing.

```csharp
// BAD — does three different things
public async Task ProcessOrder(Order order)
{
    // 1. Validate
    if (order.Items.Count == 0) throw new Exception("Empty order");
    if (order.UserId == Guid.Empty) throw new Exception("No user");

    // 2. Calculate price
    decimal total = 0;
    foreach (var item in order.Items)
        total += item.Price * item.Quantity;
    order.Total = total;

    // 3. Save and send email
    await _db.Orders.AddAsync(order);
    await _db.SaveChangesAsync();
    await _emailService.SendConfirmationAsync(order.UserId, order.Id);
}

// GOOD — each function does one thing
public async Task<Result> PlaceOrderAsync(Order order, CancellationToken ct)
{
    var validationResult = ValidateOrder(order);
    if (!validationResult.IsSuccess) return validationResult;

    order.Total = CalculateTotal(order.Items);

    await _uow.Orders.AddAsync(order, ct);
    await _uow.CommitAsync(ct);

    await _emailService.SendConfirmationAsync(order.UserId, order.Id, ct);
    return Result.Success();
}

private static Result ValidateOrder(Order order) { ... }
private static decimal CalculateTotal(IReadOnlyList<OrderItem> items) { ... }
```

**Small functions:** If you can't see the whole function on one screen, it's too long. Aim for < 20 lines. If naming a block is hard, extract it.

**Argument count:** 0–2 parameters = good. 3 = acceptable. 4+ = consider a parameter object.
```csharp
// BAD — 5 parameters
void CreateOrder(Guid userId, string street, string city, string country, List<OrderItem> items)

// GOOD — group related parameters
void CreateOrder(Guid userId, Address shippingAddress, List<OrderItem> items)
```

---

## Comments

**The rule:** Comments should explain WHY, not WHAT. Good code explains what it does through naming. A comment that explains what the code does is a sign the code should be cleaner.

```csharp
// BAD — explains what (the code already shows this)
// Increment counter
count++;

// BAD — lies when code changes but comment isn't updated
// Returns the user's name
public string GetUserEmail() { ... }

// GOOD — explains WHY (non-obvious constraint)
// Stripe requires amount in cents, not dollars
var amountCents = (long)(order.Total * 100);

// GOOD — explains a workaround for a known issue
// EF Core 8 doesn't support JSON_VALUE in SQL Server 2019 — using raw query
var results = await _context.Database.SqlQueryRaw<ProductDto>(...);

// GOOD — explains a non-obvious algorithm choice
// Using exponential backoff here: Stripe's API rate limit resets in 60s windows.
// Linear retry would exhaust all retries before the window resets.
```

**Delete commented-out code.** That's what git is for.

---

## Magic Numbers and Strings

```csharp
// BAD — what is 86400? What is "admin"?
if (tokenAge > 86400) RefreshToken();
if (user.Role == "admin") GrantAccess();

// GOOD — named constants make intent clear
private const int AccessTokenLifetimeSeconds = 86400; // 24 hours

if (tokenAge > AccessTokenLifetimeSeconds) RefreshToken();
if (user.Role == Roles.Admin) GrantAccess();
```

---

## DRY, YAGNI, KISS

**DRY — Don't Repeat Yourself**
```csharp
// BAD — same logic duplicated
public decimal GetOrderDiscount(Order order)
{
    if (order.Items.Count > 10) return order.Total * 0.1m;
    return 0;
}

public bool IsEligibleForFreeShipping(Order order)
{
    if (order.Items.Count > 10) return true; // same rule, duplicated
    return false;
}

// GOOD — extract the rule once
private bool IsLargeOrder(Order order) => order.Items.Count > 10;

public decimal GetOrderDiscount(Order order) => IsLargeOrder(order) ? order.Total * 0.1m : 0;
public bool IsEligibleForFreeShipping(Order order) => IsLargeOrder(order);
```

**YAGNI — You Aren't Gonna Need It**
Don't build features or abstractions for hypothetical future requirements.
```csharp
// BAD — "we might need plugins someday"
public interface IPluginSystem { }
public class PluginManager : IPluginSystem { }
// Nobody asked for this, nobody uses it

// GOOD — build what's needed now, refactor when the need is real
```

**KISS — Keep It Simple, Stupid**
```csharp
// BAD — over-engineered for a simple check
public class OrderValidationStrategyFactory
{
    public IOrderValidationStrategy GetStrategy(string orderType) { ... }
}

// GOOD — for one validation rule, just write the rule
if (order.Items.Count == 0)
    return Result.Failure(ErrorCodes.EmptyOrder);
```

---

## Code Smells

Common patterns that signal problems:

| Smell | Description | Fix |
|---|---|---|
| **Long method** | > 20–30 lines, does multiple things | Extract smaller methods |
| **Large class** | Dozens of methods, multiple responsibilities | Split into focused classes |
| **Duplicate code** | Same logic in multiple places | Extract to shared method/class |
| **Long parameter list** | 4+ parameters | Group into parameter object |
| **Feature envy** | Method uses another class's data more than its own | Move method to that class |
| **Dead code** | Commented-out code, unused methods | Delete it |
| **Magic numbers** | Unexplained literals | Named constants |
| **Inconsistent naming** | `GetUser`, `FetchOrder`, `RetrieveProduct` for same operation | Pick one convention |
| **Nested conditionals** | if inside if inside if | Early return / guard clauses |

---

## Guard Clauses (Early Return)

Reduce nesting by handling invalid cases first.

```csharp
// BAD — deep nesting ("arrow code")
public Result ProcessPayment(Order order, PaymentMethod method)
{
    if (order != null)
    {
        if (order.Total > 0)
        {
            if (method != null)
            {
                // actual work buried here
                return DoPayment(order, method);
            }
            else { return Result.Failure(ErrorCodes.NoPaymentMethod); }
        }
        else { return Result.Failure(ErrorCodes.ZeroTotal); }
    }
    else { return Result.Failure(ErrorCodes.NullOrder); }
}

// GOOD — guard clauses, flat structure
public Result ProcessPayment(Order order, PaymentMethod method)
{
    if (order is null)  return Result.Failure(ErrorCodes.NullOrder);
    if (order.Total <= 0) return Result.Failure(ErrorCodes.ZeroTotal);
    if (method is null) return Result.Failure(ErrorCodes.NoPaymentMethod);

    return DoPayment(order, method); // happy path at the bottom, not buried
}
```

---

## When to Refactor vs Ship

**Refactor when:**
- You're about to add a feature and the existing structure makes it hard
- You've just understood a piece of code well enough to improve it ("make the change easy, then make the easy change")
- The same change is needed in 3+ places

**Don't refactor when:**
- Deadline is immediate and code works
- You don't have tests — refactoring without tests is risky
- "It could be better" — leave working code alone unless there's a reason

**Boy Scout Rule:** Leave the code slightly cleaner than you found it. Small consistent improvements beat periodic big rewrites.

---

## Common Interview Questions

1. What does "clean code" mean to you?
2. What is a code smell? Name three.
3. What is the difference between DRY and YAGNI?
4. When should you add a comment?
5. What are guard clauses and why use them?
6. How do you decide when to refactor vs ship?

---

## Common Mistakes

- Writing comments that explain what the code does instead of why
- Leaving commented-out code "just in case"
- Applying DRY too aggressively — sometimes two similar things are different concepts
- Over-abstracting for hypothetical future use (YAGNI violation)
- Long methods because "it's easier to follow all in one place" — it isn't

---

## How It Connects

- Guard clauses are how command handlers in your Clean Architecture reject invalid inputs fast
- Single Responsibility maps directly to the S in SOLID
- Named constants (ErrorCodes) are in your repo already — that's DRY applied
- "Make the change easy, then make the easy change" — this is why you refactor before adding a feature in your E-commerce project

---

## My Confidence Level
- `[ ]` Naming — intent-revealing names, avoid noise words
- `[ ]` Functions — single responsibility, small, low parameter count
- `[ ]` Comments — when to write them, what to write
- `[ ]` Magic numbers — named constants
- `[ ]` DRY, YAGNI, KISS — knowing when each applies
- `[ ]` Code smells — recognising them in a code review
- `[ ]` Guard clauses — flatten nested conditionals

## My Notes
<!-- Personal notes -->
