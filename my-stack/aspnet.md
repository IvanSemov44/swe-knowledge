# ASP.NET Core Deep Dive

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: c -->

---

## Dependency Injection Lifetimes

### The three lifetimes

| Lifetime | Instance created | Disposed |
|---|---|---|
| `Singleton` | Once, app startup | App shutdown |
| `Scoped` | Once per HTTP request | End of request |
| `Transient` | Every time it's resolved | When consuming scope ends |

```csharp
builder.Services.AddSingleton<IEmailService, EmailService>();   // one instance ever
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();          // one per request
builder.Services.AddTransient<IOrderValidator, OrderValidator>(); // new every time
```

### Captive Dependency — the classic trap

Injecting a shorter-lived service into a longer-lived one. The short-lived service gets "captured" and lives longer than intended.

```csharp
// WRONG — DbContext (Scoped) injected into Singleton
public class MyCache
{
    private readonly AppDbContext _db; // captured forever
    public MyCache(AppDbContext db) { _db = db; }
}

builder.Services.AddSingleton<MyCache>(); // ← problem here
```

`DbContext` gets created once, never disposed. Accumulates tracked entities across all requests → stale data, entity conflicts, potential data corruption across users.

ASP.NET Core **throws at startup** in Development:
```
InvalidOperationException: Cannot consume scoped service 'AppDbContext'
from singleton 'MyCache'.
```

### The safe rule

```
Singleton  → can only depend on Singleton
Scoped     → can depend on Singleton or Scoped
Transient  → can depend on anything
```

---

## Middleware Pipeline

Middleware runs in the order it's registered. Each piece can short-circuit or pass to the next.

```csharp
app.UseExceptionHandler();   // 1 — outermost, catches everything below
app.UseHttpsRedirection();   // 2
app.UseAuthentication();     // 3 — who are you?
app.UseAuthorization();      // 4 — are you allowed? (must come after Authentication)
app.MapControllers();        // 5 — innermost, route to controller
```

Order matters. Authorization before Authentication = auth never runs.

### Custom middleware

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next) { _next = next; }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine($"→ {context.Request.Method} {context.Request.Path}");
        await _next(context);  // call next middleware
        Console.WriteLine($"← {context.Response.StatusCode}");
    }
}

// Register
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

## Filters

Run at specific points in the MVC pipeline (before/after action execution).

| Filter | When it runs |
|---|---|
| `IActionFilter` | Before / after action method |
| `IExceptionFilter` | When unhandled exception occurs |
| `IAuthorizationFilter` | Before model binding |
| `IResultFilter` | Before / after result execution |

```csharp
// Validation filter — used in your E-commerce project
public class ValidationFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(context.ModelState);
        }
    }

    public void OnActionExecuted(ActionExecutedContext context) { }
}

// Register globally
builder.Services.AddControllers(options =>
    options.Filters.Add<ValidationFilter>());
```

---

## Controllers & Routing

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator) { _mediator = mediator; }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(
        Guid id,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetOrderByIdQuery(id), cancellationToken);
        return result.IsSuccess ? Ok(result.Value) : NotFound();
    }

    [HttpPost]
    public async Task<IActionResult> Create(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new CreateOrderCommand(request), cancellationToken);
        return result.IsSuccess
            ? CreatedAtAction(nameof(GetById), new { id = result.Value }, result.Value)
            : BadRequest(result.Error);
    }
}
```

`[ApiController]` enables: automatic model validation, binding source inference, ProblemDetails responses.

---

## Configuration

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=...;Database=...;"
  },
  "Jwt": {
    "Secret": "...",
    "Issuer": "my-api"
  }
}

// Strongly typed config
public class JwtSettings
{
    public string Secret { get; init; } = string.Empty;
    public string Issuer { get; init; } = string.Empty;
}

builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("Jwt"));

// In service — inject IOptions<JwtSettings>
public class JwtService(IOptions<JwtSettings> options)
{
    private readonly JwtSettings _settings = options.Value;
}
```

Environment overrides: `appsettings.Development.json` → environment variables → command line args (each overrides the previous).

---

## Minimal APIs (vs Controllers)

```csharp
// Controller approach
[HttpGet("{id}")]
public async Task<IActionResult> Get(Guid id) { ... }

// Minimal API approach
app.MapGet("/api/orders/{id}", async (Guid id, IMediator mediator) =>
{
    var result = await mediator.Send(new GetOrderByIdQuery(id));
    return result.IsSuccess ? Results.Ok(result.Value) : Results.NotFound();
});
```

Minimal APIs are lighter — no controller overhead. Good for small services or microservices. Controllers are better for larger APIs with filters, versioning, Swagger.

---

## Gaps to Fill Next

- [ ] SignalR (real-time)
- [ ] Rate limiting (ASP.NET Core 7+ built-in)
- [ ] Output caching middleware
- [ ] Health checks endpoint
- [ ] API versioning strategies

---

## Connected To

- `mid/clean-architecture.md` — controllers are the outermost layer, thin by convention
- `mid/cqrs.md` — controllers dispatch to MediatR, nothing else
- `mid/authentication.md` — UseAuthentication / UseAuthorization middleware order
