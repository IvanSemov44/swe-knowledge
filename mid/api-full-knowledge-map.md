# API Full Knowledge Map

Everything you need beyond the BC internals to build and run a production .NET Web API.

---

## Already Covered (from BC work)

- Clean Architecture, CQRS, MediatR, Pipeline behaviors
- DDD, EF Core, Repository, Unit of Work
- FluentValidation, Result pattern, ErrorCodes
- Outbox, Inbox, MassTransit, Projections
- Exception handling middleware + ProblemDetails
- Structured logging (Serilog), Correlation ID, Health checks
- JWT basics

---

## Authentication & Authorization

| What | Notes |
|---|---|
| **JWT validation middleware** | `AddAuthentication().AddJwtBearer()` — validates signature, issuer, audience, expiry |
| **`[Authorize]` + `[AllowAnonymous]`** | Filter pipeline enforces before action runs |
| **Role-based authorization** | `[Authorize(Roles = "Admin")]` |
| **Policy-based authorization** | `[Authorize(Policy = "CanManageOrders")]` — more flexible than roles |
| **Claims transformation** | `IClaimsTransformation` — enrich claims from DB after token validation |
| **Refresh token rotation** | Issue short-lived access token + long-lived refresh token; rotate on use |
| **PKCE** | SPAs must use it — prevents auth code interception in public clients |
| **HttpOnly + Secure + SameSite cookies** | Cookie auth security attributes — `SameSite=Lax` protects against CSRF |

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = config["Jwt:Issuer"],
            ValidAudience = config["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Secret"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanManageOrders", policy =>
        policy.RequireClaim("permission", "orders.write"));
});
```

---

## Middleware Pipeline — Order Is Critical

Wrong order breaks CORS, auth, rate limiting silently.

```csharp
app.UseExceptionHandler();      // 1. catch everything below this
app.UseHttpsRedirection();      // 2. redirect HTTP → HTTPS
app.UseHsts();                  // 3. HSTS header — tell browsers to remember HTTPS
app.UseCors();                  // 4. CORS headers before auth (browser preflights)
app.UseRateLimiter();           // 5. reject before doing auth work
app.UseAuthentication();        // 6. who are you? (reads JWT, sets ClaimsPrincipal)
app.UseAuthorization();         // 7. are you allowed? (checks [Authorize] policies)
app.UseResponseCompression();   // 8. gzip/brotli responses
app.MapControllers();           // 9. route to action methods
```

**Custom middleware:**
```csharp
public class CorrelationIdMiddleware : IMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var correlationId = context.Request.Headers["X-Correlation-Id"]
            .FirstOrDefault() ?? Guid.NewGuid().ToString();

        context.Response.Headers["X-Correlation-Id"] = correlationId;
        // Add to log context (Serilog enricher)
        using (LogContext.PushProperty("CorrelationId", correlationId))
            await next(context);
    }
}
```

---

## API Versioning

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true; // adds Api-Supported-Versions header
});

[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/orders")]
public class OrdersController : ControllerBase { }

// Deprecate old version — still works, but header warns client
[ApiVersion("1.0", Deprecated = true)]
[ApiVersion("2.0")]
```

---

## Pagination, Filtering, Sorting

Every list endpoint follows this shape:

```csharp
// Consistent query shape
public record GetOrdersQuery(
    int Page = 1,
    int PageSize = 20,
    string? SortBy = null,
    bool Ascending = true,
    OrderStatus? Status = null,
    DateTime? FromDate = null
) : IRequest<PagedResult<OrderSummaryDto>>;

// Consistent response shape
public record PagedResult<T>(
    List<T> Items,
    int TotalCount,
    int Page,
    int PageSize
)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

// Query handler
public async Task<PagedResult<OrderSummaryDto>> Handle(GetOrdersQuery q, CancellationToken ct)
{
    var query = _context.Orders.AsNoTracking();

    if (q.Status.HasValue) query = query.Where(o => o.Status == q.Status.Value);
    if (q.FromDate.HasValue) query = query.Where(o => o.CreatedAt >= q.FromDate.Value);

    var total = await query.CountAsync(ct);

    var items = await query
        .OrderBy(q.SortBy, q.Ascending) // extension method or switch
        .Skip((q.Page - 1) * q.PageSize)
        .Take(q.PageSize)
        .Select(o => new OrderSummaryDto(o.Id, o.Status, o.TotalAmount, o.CreatedAt))
        .ToListAsync(ct);

    return new PagedResult<OrderSummaryDto>(items, total, q.Page, q.PageSize);
}
```

**Offset vs keyset pagination:**
- **Offset** (`SKIP/TAKE`): simple, but slow on page 500 of a 1M row table (DB scans all skipped rows)
- **Keyset/cursor**: `WHERE Id > lastSeenId ORDER BY Id TAKE 20` — always fast, but no random page access

---

## Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Fixed window — 100 requests per minute per IP
    options.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        o.QueueLimit = 10;
    });

    // Sliding window — smoother than fixed
    options.AddSlidingWindowLimiter("sliding", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow = 6; // 10-second segments
    });
});

app.UseRateLimiter();

[EnableRateLimiting("fixed")]
[HttpPost("orders")]
public async Task<IActionResult> PlaceOrder() { }
```

For distributed rate limiting (multiple API instances) → use Redis + `AddRateLimiter` with a custom policy reading from Redis.

---

## Caching

```csharp
// In-memory — single instance only
services.AddMemoryCache();
// inject: IMemoryCache

// Distributed — Redis, works across instances
services.AddStackExchangeRedisCache(options =>
    options.Configuration = config.GetConnectionString("Redis"));
// inject: IDistributedCache

// Output caching — entire response cached, .NET 8 built-in
services.AddOutputCache(options =>
{
    options.AddPolicy("Products", b => b.Tag("products").Expire(TimeSpan.FromMinutes(5)));
});
app.UseOutputCache();

[OutputCache(PolicyName = "Products")]
[HttpGet("products")]
public async Task<IActionResult> GetProducts() { }
```

**ETag — conditional requests:**
```csharp
// Server returns: ETag: "abc123"
// Client next request: If-None-Match: "abc123"
// Server: 304 Not Modified if content unchanged → no body sent
```

**Cache-aside in a handler:**
```csharp
var cached = await _cache.GetStringAsync($"order:{id}", ct);
if (cached is not null) return JsonSerializer.Deserialize<OrderDto>(cached)!;

var order = await _repo.GetByIdAsync(id, ct);
await _cache.SetStringAsync($"order:{id}",
    JsonSerializer.Serialize(order),
    new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) },
    ct);
return order;
```

---

## File Handling

```csharp
// Upload
[HttpPost("products/{id}/image")]
public async Task<IActionResult> UploadImage(Guid id, IFormFile file)
{
    if (file.Length > 5_000_000) return BadRequest("File too large");
    if (!new[] { ".jpg", ".png" }.Contains(Path.GetExtension(file.FileName).ToLower()))
        return BadRequest("Invalid file type");

    await using var stream = file.OpenReadStream();
    var url = await _blobService.UploadAsync($"products/{id}", stream, file.ContentType);
    return Ok(new { url });
}

// Download
[HttpGet("invoices/{id}")]
public async Task<IActionResult> DownloadInvoice(Guid id)
{
    var stream = await _blobService.DownloadAsync($"invoices/{id}.pdf");
    return File(stream, "application/pdf", $"invoice-{id}.pdf");
}
```

---

## Background Jobs

```csharp
// Channel<T> — in-process fire-and-forget queue
// Producer (e.g. controller):
await _channel.Writer.WriteAsync(new SendEmailJob(userId, template), ct);

// Consumer (BackgroundService):
await foreach (var job in _channel.Reader.ReadAllAsync(stoppingToken))
    await _emailService.SendAsync(job, stoppingToken);
```

```csharp
// Hangfire — persistent, survives restarts, has UI dashboard
builder.Services.AddHangfire(config => config.UseSqlServerStorage(connectionString));
builder.Services.AddHangfireServer();

// Fire-and-forget
BackgroundJob.Enqueue(() => emailService.SendAsync(userId, template));

// Scheduled
BackgroundJob.Schedule(() => reportService.GenerateMonthlyReport(), TimeSpan.FromHours(24));

// Recurring
RecurringJob.AddOrUpdate("cleanup", () => cleanupService.Run(), Cron.Daily);
```

---

## Real-Time Communication

```csharp
// SignalR — bi-directional, works over WebSocket / SSE / long polling
builder.Services.AddSignalR();
app.MapHub<OrderHub>("/hubs/orders");

public class OrderHub : Hub
{
    public async Task JoinOrderGroup(string orderId)
        => await Groups.AddToGroupAsync(Context.ConnectionId, orderId);
}

// Push from server (e.g. from a consumer after StockReserved):
await _hubContext.Clients.Group(orderId).SendAsync("StatusUpdated", new { status });
```

---

## API Documentation (Swagger)

```csharp
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo { Title = "ECommerce API", Version = "v1" });

    // Show JWT auth in Swagger UI
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Type = SecuritySchemeType.Http,
        Scheme = "bearer",
        BearerFormat = "JWT"
    });
});

// On actions — tell Swagger what responses to expect
[HttpPost]
[ProducesResponseType(typeof(ApiResponse<Guid>), StatusCodes.Status201Created)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status409Conflict)]
public async Task<IActionResult> PlaceOrder([FromBody] PlaceOrderRequest request) { }
```

---

## Configuration Management

```csharp
// appsettings.json
{
  "Database": { "ConnectionString": "...", "CommandTimeoutSeconds": 30 },
  "Jwt": { "Issuer": "...", "Audience": "...", "Secret": "..." }
}

// Typed options class
public class DatabaseOptions
{
    [Required] public string ConnectionString { get; set; } = string.Empty;
    [Range(1, 300)] public int CommandTimeoutSeconds { get; set; } = 30;
}

// Registration with validation at startup
builder.Services.AddOptions<DatabaseOptions>()
    .BindConfiguration("Database")
    .ValidateDataAnnotations()
    .ValidateOnStart(); // throws at startup if invalid — fail fast

// Three IOptions variants:
// IOptions<T>          — singleton, never reloads
// IOptionsSnapshot<T>  — scoped, reloads per HTTP request
// IOptionsMonitor<T>   — singleton, reloads on file change (hot reload)
```

---

## Idempotency Keys (POST endpoints)

For operations that must not execute twice (payments, order placement):

```csharp
// Client sends: Idempotency-Key: <uuid>
// Middleware/filter checks:
// - If key seen before → return cached response immediately
// - If key not seen → execute, store response against key (Redis with TTL)

public class IdempotencyFilter : IAsyncActionFilter
{
    private readonly IDistributedCache _cache;

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var key = context.HttpContext.Request.Headers["Idempotency-Key"].FirstOrDefault();
        if (key is null) { await next(); return; }

        var cached = await _cache.GetStringAsync(key);
        if (cached is not null)
        {
            context.Result = new ContentResult
            {
                Content = cached,
                ContentType = "application/json",
                StatusCode = 200
            };
            return;
        }

        var result = await next();
        // cache the response for 24h
        await _cache.SetStringAsync(key, SerializeResponse(result), new()
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
        });
    }
}
```

---

## Testing the API Layer

```csharp
// WebApplicationFactory — spins up your entire API in memory
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                // Swap real DbContext for test DB
                services.RemoveAll<DbContextOptions<OrdersDbContext>>();
                services.AddDbContext<OrdersDbContext>(opts =>
                    opts.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task PlaceOrder_ValidRequest_Returns201()
    {
        var request = new { UserId = Guid.NewGuid(), Items = new[] { new { ProductId = Guid.NewGuid(), Quantity = 2 } } };
        var response = await _client.PostAsJsonAsync("/api/v1/orders", request);
        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

---

## Graceful Shutdown

```csharp
// Configure shutdown timeout
builder.Services.Configure<HostOptions>(opts =>
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30));

// BackgroundService respects cancellation
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await DoWorkAsync(stoppingToken);
        await Task.Delay(5000, stoppingToken); // exits cleanly on cancel
    }
    // flush anything in memory here
}
```

---

## Response Compression

```csharp
builder.Services.AddResponseCompression(opts =>
{
    opts.EnableForHttps = true;
    opts.Providers.Add<BrotliCompressionProvider>();  // better ratio
    opts.Providers.Add<GzipCompressionProvider>();    // wider support
    opts.MimeTypes = ResponseCompressionDefaults.MimeTypes
        .Concat(new[] { "application/json" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(opts =>
    opts.Level = CompressionLevel.Fastest);

app.UseResponseCompression(); // in middleware pipeline
```

---

## Full API Knowledge Map

```
API Knowledge
│
├── Security
│   ├── JWT validation setup                    [b]
│   ├── Role-based + policy-based authorization [ ]
│   ├── PKCE + refresh token rotation           [ ]
│   ├── CORS, CSRF, XSS, security headers       [b] (security-depth.md)
│   └── Rate limiting (built-in + Redis)        [ ]
│
├── Middleware Pipeline
│   ├── Correct order (exception → HTTPS → CORS → rate → auth → authz → compress → route)
│   ├── Custom middleware (IMiddleware)          [b]
│   └── Short-circuit patterns                  [ ]
│
├── API Design
│   ├── Versioning (URL / header)               [ ]
│   ├── Pagination + PagedResult<T>             [ ]
│   ├── Filtering + sorting on list endpoints   [ ]
│   ├── Idempotency keys on POST                [ ]
│   └── Swagger / OpenAPI documentation        [ ]
│
├── Caching
│   ├── IMemoryCache (single instance)          [ ]
│   ├── IDistributedCache → Redis               [b] (system design)
│   ├── Output caching (.NET 8)                 [ ]
│   └── ETag + Cache-Control                    [ ]
│
├── File Handling
│   ├── IFormFile upload → IBlobService         [ ]
│   └── FileStreamResult download               [ ]
│
├── Real-Time
│   ├── SignalR (bi-directional)                [ ]
│   └── SSE (server push only)                  [ ]
│
├── Background Work
│   ├── BackgroundService                        [b] (OutboxDispatcher)
│   ├── Channel<T> (in-process queue)           [ ]
│   └── Hangfire (persistent scheduled jobs)    [ ]
│
├── Configuration
│   ├── Options pattern (IOptions variants)     [ ]
│   ├── ValidateOnStart — fail fast             [ ]
│   └── Secrets management                      [ ]
│
├── Testing
│   ├── Unit (xUnit + Moq)                      [b]
│   ├── Integration (Testcontainers)            [b]
│   └── API layer (WebApplicationFactory)       [ ]
│
├── Observability
│   ├── Structured logging (Serilog)             [b]
│   ├── Correlation ID                           [b]
│   ├── Health checks                            [b]
│   ├── OpenTelemetry — distributed tracing      [ ]
│   └── Metrics (p99, error rate, request rate)  [ ]
│
└── Deployment
    ├── Graceful shutdown                        [ ]
    ├── Response compression                     [ ]
    ├── Docker + health check                    [b]
    └── Environment config                       [b]
```

---

## My Confidence Level
- `[ ]` Role-based + policy-based authorization
- `[ ]` PKCE + refresh token rotation
- `[ ]` Middleware pipeline order — all 9 steps from memory
- `[ ]` API versioning setup
- `[ ]` Pagination + PagedResult<T> pattern
- `[ ]` Offset vs keyset pagination — trade-offs
- `[ ]` Rate limiting (.NET 8 built-in)
- `[ ]` IMemoryCache + IDistributedCache usage
- `[ ]` Output caching
- `[ ]` Idempotency key filter pattern
- `[ ]` WebApplicationFactory — API integration tests
- `[ ]` Channel<T> producer/consumer
- `[ ]` Hangfire setup
- `[ ]` SignalR hub setup
- `[ ]` Options pattern — IOptions vs IOptionsSnapshot vs IOptionsMonitor
- `[ ]` ValidateOnStart
- `[ ]` Graceful shutdown + HostOptions
- `[ ]` Response compression
