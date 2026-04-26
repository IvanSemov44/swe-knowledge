# C# Exception Handling & ProblemDetails

## What it is
How .NET handles errors at runtime: the exception hierarchy, when to throw vs return `Result<T>`, and how to expose consistent error responses from an ASP.NET Core API using ProblemDetails (RFC 7807).

## Why it matters
Unhandled exceptions crash requests. Inconsistent error formats frustrate API consumers. Interviewers ask "how do you handle errors in your API?" — you need a complete answer covering domain errors, unexpected exceptions, and the response contract.

---

## Exception Hierarchy

```
Exception
├── SystemException
│   ├── NullReferenceException    — null dereference
│   ├── InvalidOperationException — invalid state (use this for domain violations)
│   ├── ArgumentException
│   │   ├── ArgumentNullException  — null argument
│   │   └── ArgumentOutOfRangeException
│   ├── OverflowException
│   ├── IndexOutOfRangeException
│   └── NotSupportedException
├── IOException
│   ├── FileNotFoundException
│   └── UnauthorizedAccessException
└── ApplicationException          — base for custom app exceptions (rarely used now)
```

---

## When to Throw vs Return Result\<T\>

This is a critical architecture decision. In your Clean Architecture + DDD codebase:

| Situation | Use |
|---|---|
| **Expected business outcome** (not found, invalid state, insufficient stock) | `Result<T>` — predictable, handled flow |
| **Programming error** (null where impossible, invalid argument from internal code) | `throw` — indicates a bug |
| **Unexpected infrastructure failure** (DB down, network timeout) | Let it propagate — global handler catches it |
| **Domain invariant violation** (called from trusted internal code) | `throw DomainException` — should never happen if code is correct |

```csharp
// EXPECTED business outcome — use Result<T>, not exception
public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
{
    var product = await _uow.Products.GetByIdAsync(cmd.ProductId, ct);
    if (product is null)
        return Result.Failure<Guid>(ErrorCodes.ProductNotFound); // expected — caller handles this

    if (product.Stock < cmd.Quantity)
        return Result.Failure<Guid>(ErrorCodes.InsufficientStock); // expected

    var order = Order.Create(cmd.UserId, cmd.ShippingAddress);
    order.AddItem(product, cmd.Quantity);
    _uow.Orders.Add(order);
    await _uow.CommitAsync(ct);
    return Result.Success(order.Id);
}

// PROGRAMMING ERROR — throw
public void SetPrice(decimal price)
{
    if (price < 0)
        throw new ArgumentOutOfRangeException(nameof(price), "Price cannot be negative");
    // This is a bug in the caller, not a business rule
}

// DOMAIN INVARIANT — throw inside the domain
public void Place()
{
    if (Status != OrderStatus.Pending)
        throw new InvalidOperationException($"Cannot place order in {Status} status.");
    // This means the application layer called Place() when it shouldn't — code bug
}
```

---

## Custom Exceptions

```csharp
// Domain exception — signals a violated invariant
public class DomainException : Exception
{
    public string ErrorCode { get; }

    public DomainException(string errorCode, string message)
        : base(message)
    {
        ErrorCode = errorCode;
    }
}

// Infrastructure exception — wraps external failures
public class ExternalServiceException : Exception
{
    public string ServiceName { get; }

    public ExternalServiceException(string serviceName, string message, Exception inner)
        : base(message, inner)
    {
        ServiceName = serviceName;
    }
}
```

---

## try/catch Best Practices

```csharp
// GOOD — catch specific exceptions, handle them meaningfully
try
{
    await _paymentGateway.ChargeAsync(order, ct);
}
catch (PaymentDeclinedException ex)
{
    return Result.Failure(ErrorCodes.PaymentDeclined);
}
catch (ExternalServiceException ex)
{
    _logger.LogError(ex, "Payment gateway failure for order {OrderId}", order.Id);
    return Result.Failure(ErrorCodes.PaymentGatewayUnavailable);
}
// Do NOT catch Exception broadly here — let unexpected errors bubble up

// BAD — swallowing exceptions
try { await DoSomething(); }
catch (Exception) { } // ← never do this — hides bugs

// BAD — catching and rethrowing incorrectly (loses stack trace)
try { await DoSomething(); }
catch (Exception ex) { throw ex; } // ← loses original stack trace

// GOOD — rethrow preserving stack trace
try { await DoSomething(); }
catch (Exception ex) { throw; } // ← preserves stack trace

// finally — always runs, use for cleanup
FileStream? file = null;
try
{
    file = File.Open(path, FileMode.Open);
    Process(file);
}
finally
{
    file?.Dispose(); // prefer 'using' statement, but this is the underlying mechanism
}
```

---

## Global Exception Handling in ASP.NET Core

Don't wrap every controller method in try/catch. Use a global handler.

### Option 1: Exception Handler Middleware

```csharp
// Program.cs
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
        var exception = exceptionFeature?.Error;

        context.Response.ContentType = "application/problem+json";

        var problem = exception switch
        {
            DomainException domainEx => new ProblemDetails
            {
                Status = StatusCodes.Status422UnprocessableEntity,
                Title = "Business rule violation",
                Detail = domainEx.Message,
                Extensions = { ["errorCode"] = domainEx.ErrorCode }
            },
            UnauthorizedAccessException => new ProblemDetails
            {
                Status = StatusCodes.Status403Forbidden,
                Title = "Forbidden"
            },
            _ => new ProblemDetails
            {
                Status = StatusCodes.Status500InternalServerError,
                Title = "An unexpected error occurred"
            }
        };

        context.Response.StatusCode = problem.Status ?? 500;
        await context.Response.WriteAsJsonAsync(problem);
    });
});
```

### Option 2: IExceptionHandler (.NET 8 — preferred)

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken ct)
    {
        _logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        var (status, title) = exception switch
        {
            DomainException e    => (422, e.Message),
            NotFoundException    => (404, "Resource not found"),
            UnauthorizedAccessException => (403, "Forbidden"),
            _                    => (500, "An unexpected error occurred")
        };

        context.Response.StatusCode = status;
        await context.Response.WriteAsJsonAsync(new ProblemDetails
        {
            Status = status,
            Title = title,
            Instance = context.Request.Path
        }, ct);

        return true; // handled — don't rethrow
    }
}

// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
app.UseExceptionHandler();
```

---

## ProblemDetails — RFC 7807

The standard JSON error format for HTTP APIs. ASP.NET Core has built-in support.

```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Insufficient stock",
  "status": 422,
  "detail": "Product 'Blue Sneakers' only has 3 units in stock.",
  "instance": "/api/v1/orders",
  "errorCode": "INSUFFICIENT_STOCK",
  "traceId": "00-abc123-def456-00"
}
```

```csharp
// Controller — return ProblemDetails for business failures
[HttpPost]
public async Task<IActionResult> CreateOrder(
    CreateOrderRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(request.ToCommand(), ct);

    if (!result.IsSuccess)
        return result.ErrorCode switch
        {
            ErrorCodes.ProductNotFound => NotFound(new ProblemDetails
            {
                Title = "Product not found",
                Detail = $"Product {request.ProductId} does not exist."
            }),
            ErrorCodes.InsufficientStock => UnprocessableEntity(new ProblemDetails
            {
                Title = "Insufficient stock",
                Detail = "The requested quantity is not available."
            }),
            _ => Problem("An unexpected error occurred", statusCode: 500)
        };

    return CreatedAtAction(nameof(GetOrder), new { id = result.Value }, null);
}
```

---

## ApiResponse\<T\> Wrapper Pattern

Your E-commerce project wraps responses:

```csharp
public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? ErrorCode { get; init; }
    public string? Message { get; init; }

    public static ApiResponse<T> Ok(T data) =>
        new() { Success = true, Data = data };

    public static ApiResponse<T> Fail(string errorCode, string message) =>
        new() { Success = false, ErrorCode = errorCode, Message = message };
}
```

---

## Common Interview Questions

1. When should you throw an exception vs return `Result<T>`?
2. What is the difference between `throw ex` and `throw`?
3. What is ProblemDetails and why use it?
4. How do you implement global exception handling in ASP.NET Core?
5. What is wrong with catching `Exception` broadly and swallowing it?
6. What is the difference between `DomainException` and using `Result<T>`?

---

## Common Mistakes

- Using exceptions for control flow (expected business outcomes)
- `throw ex` instead of `throw` — loses the original stack trace
- Swallowing exceptions with empty `catch` blocks
- Not having a global exception handler — unhandled exceptions return 500 with stack traces in production
- Returning 500 for business rule violations (use 422)
- Not logging unexpected exceptions before returning generic error to client

---

## How It Connects

- `Result<T>` pattern (from `shared-kernel.md`) handles expected business failures
- Global exception handler catches everything else — unexpected infrastructure failures
- ProblemDetails is the standard response format for all errors — consistent for the frontend (RTK Query)
- Domain exceptions signal programming errors — should never happen in correct code
- MediatR pipeline behavior can wrap all handler calls and convert `Result` failures to ProblemDetails

---

## My Confidence Level
- `[ ]` Exception hierarchy — knowing which built-in exception to throw
- `[ ]` throw vs Result<T> — the decision rule
- `[ ]` try/catch best practices — specific exceptions, rethrow correctly
- `[ ]` Global exception handler — IExceptionHandler (.NET 8)
- `[ ]` ProblemDetails — RFC 7807, ASP.NET Core integration
- `[ ]` Custom exceptions — DomainException, when to use

## My Notes
<!-- Personal notes -->
