# Dependency Injection — Deep Dive

## What it is
.NET's built-in DI container manages object lifetimes and resolves dependencies. Beyond the basics, there are subtle bugs (captive dependency), advanced patterns (named services, IServiceScopeFactory), and the Options pattern for configuration.

## Why it matters
DI lifetime bugs cause data leaks between requests and hard-to-reproduce errors. The captive dependency problem is a classic interview question for .NET roles.

---

## Lifetime Recap

```csharp
// Singleton — one instance for the entire app lifetime
services.AddSingleton<IEmailService, SmtpEmailService>();
// Use for: stateless services, caches, configuration wrappers

// Scoped — one instance per HTTP request (or per scope)
services.AddScoped<AppDbContext>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
// Use for: DbContext, UnitOfWork, anything that must be consistent within a request

// Transient — new instance every time it's requested
services.AddTransient<IPasswordHasher, BCryptPasswordHasher>();
// Use for: lightweight, stateless services where you want isolation
```

---

## The Captive Dependency Problem

**The bug:** A longer-lived service holds a reference to a shorter-lived one. The short-lived service is "captured" and never released — it lives for the lifetime of the parent, which is wrong.

```csharp
// WRONG — Singleton captures a Scoped service
public class OrderProcessor // registered as Singleton
{
    private readonly AppDbContext _context; // Scoped — one per request

    public OrderProcessor(AppDbContext context) // ← context captured at startup
    {
        _context = context;
    }
    // Problem: _context is the SAME instance across ALL requests
    // DbContext is not thread-safe → data corruption, cross-request leaks
}

// The runtime will throw this in development mode:
// InvalidOperationException: Cannot consume scoped service 'AppDbContext'
// from singleton 'OrderProcessor'.
```

**Lifetime compatibility matrix:**
```
Can a longer-lived service depend on a shorter-lived one?

Singleton  → Singleton  ✅
Singleton  → Scoped     ❌ CAPTIVE DEPENDENCY
Singleton  → Transient  ⚠️  Transient becomes effectively Singleton
Scoped     → Scoped     ✅
Scoped     → Transient  ✅ (transient created per scope usage)
Transient  → Scoped     ✅
Transient  → Transient  ✅
```

**Rule:** A service can only depend on services with **equal or longer** lifetime.

---

## IServiceScopeFactory — The Correct Fix

When a Singleton genuinely needs to use a Scoped service (e.g., a background job accessing the DB):

```csharp
// Correct — Singleton uses IServiceScopeFactory to create a scope manually
public class OrderCleanupJob // Singleton or hosted service
{
    private readonly IServiceScopeFactory _scopeFactory;

    public OrderCleanupJob(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope(); // explicit scope
        var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

        var staleOrders = await uow.Orders.GetStaleAsync(ct);
        foreach (var order in staleOrders)
            order.Cancel();

        await uow.CommitAsync(ct);
        // scope disposed here → DbContext disposed → connection returned to pool
    }
}
```

**When you need this:** Background services (`IHostedService`), Singletons that occasionally do DB work, Hangfire jobs.

---

## GetRequiredService vs GetService

```csharp
// GetRequiredService<T> — throws if not registered (preferred)
var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();

// GetService<T> — returns null if not registered
var optional = scope.ServiceProvider.GetService<IFeatureFlag>();
if (optional is not null) optional.IsEnabled("newCheckout");
```

**Avoid injecting `IServiceProvider` directly** into your services — it's the Service Locator anti-pattern. It hides dependencies and makes testing hard. Use constructor injection.

```csharp
// Service Locator anti-pattern — DON'T DO THIS
public class OrderService
{
    private readonly IServiceProvider _sp;
    public OrderService(IServiceProvider sp) => _sp = sp;

    public async Task Process()
    {
        var repo = _sp.GetRequiredService<IOrderRepository>(); // hidden dependency
    }
}

// Constructor injection — DO THIS
public class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo; // explicit, testable
}
```

---

## Keyed Services (.NET 8)

Register multiple implementations of the same interface under different keys.

```csharp
// Register two SMS providers
services.AddKeyedSingleton<INotificationService, SmsNotification>("sms");
services.AddKeyedSingleton<INotificationService, EmailNotification>("email");

// Inject by key
public class AlertService
{
    private readonly INotificationService _sms;
    private readonly INotificationService _email;

    public AlertService(
        [FromKeyedServices("sms")]   INotificationService sms,
        [FromKeyedServices("email")] INotificationService email)
    {
        _sms = sms;
        _email = email;
    }
}
```

Before .NET 8 — common workaround: factory delegate or named factory class.

---

## Options Pattern

The clean way to inject strongly-typed configuration. Avoids `IConfiguration["Key"]` string access everywhere.

```csharp
// appsettings.json
{
  "Jwt": {
    "Secret": "super-secret-key",
    "Issuer": "myapp",
    "ExpiryMinutes": 15
  }
}

// Options class
public class JwtOptions
{
    public const string SectionName = "Jwt";
    public string Secret { get; init; } = string.Empty;
    public string Issuer { get; init; } = string.Empty;
    public int ExpiryMinutes { get; init; } = 15;
}

// Register
services.Configure<JwtOptions>(configuration.GetSection(JwtOptions.SectionName));

// Inject
public class JwtTokenService
{
    private readonly JwtOptions _options;

    public JwtTokenService(IOptions<JwtOptions> options)
    {
        _options = options.Value; // .Value is the resolved options object
    }
}
```

**IOptions vs IOptionsSnapshot vs IOptionsMonitor:**

| | `IOptions<T>` | `IOptionsSnapshot<T>` | `IOptionsMonitor<T>` |
|---|---|---|---|
| Lifetime | Singleton | Scoped | Singleton |
| Reloads config changes? | No | Yes (per-request) | Yes (with OnChange callback) |
| Use for | Startup config, never changes | Per-request dynamic config | Background services needing live reload |

---

## Registering Services — Extension Method Pattern

```csharp
// Clean way — each layer registers its own services
public static class InfrastructureServiceExtensions
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(opts =>
            opts.UseSqlServer(configuration.GetConnectionString("Default")));

        services.AddScoped<IUnitOfWork, UnitOfWork>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();

        services.Configure<JwtOptions>(configuration.GetSection(JwtOptions.SectionName));
        services.AddSingleton<IJwtTokenService, JwtTokenService>();

        return services;
    }
}

// Program.cs stays clean
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddApplication();
builder.Services.AddApi();
```

---

## Common Interview Questions

1. What is the difference between Singleton, Scoped, and Transient?
2. What is the captive dependency problem? How do you fix it?
3. When would you use `IServiceScopeFactory`?
4. What is the Service Locator anti-pattern and why is it bad?
5. What is the Options pattern? What is the difference between `IOptions` and `IOptionsSnapshot`?
6. How do you register multiple implementations of the same interface?

---

## Common Mistakes

- Injecting `AppDbContext` into a Singleton (captive dependency → data corruption)
- Injecting `IServiceProvider` to resolve services at runtime (Service Locator)
- Using `IConfiguration["Key"]` string access in services instead of Options pattern
- Forgetting `.Value` on `IOptions<T>` (`.Value` returns the actual object)
- Registering everything as Transient "to be safe" — wastes resources, breaks things that need shared state within a request

---

## How It Connects

- `IUnitOfWork` is Scoped — same transaction per HTTP request
- `IHostedService` runs as Singleton — must use `IServiceScopeFactory` to access DbContext
- Options pattern replaces all `_config["Jwt:Secret"]` string lookups in your services
- Keyed services are how you'd register multiple payment providers (Stripe, PayPal) and inject the right one based on user choice
- Extension methods keep `Program.cs` clean — each project layer registers itself

---

## My Confidence Level
- `[b]` DI lifetimes — Singleton, Scoped, Transient
- `[ ]` Captive dependency — what it is, why it breaks, the fix
- `[ ]` IServiceScopeFactory — creating scopes in Singletons
- `[ ]` Service Locator anti-pattern — why it's bad
- `[ ]` Options pattern — IOptions vs IOptionsSnapshot vs IOptionsMonitor
- `[ ]` Keyed services (.NET 8)
- `[b]` Extension method registration pattern

## My Notes
<!-- Personal notes -->
