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

## IHttpClientFactory + Polly

Never `new HttpClient()` — it exhausts sockets. Use `IHttpClientFactory`.

```csharp
// Typed client — cleanest approach
public class StripeClient : IStripeClient
{
    private readonly HttpClient _client;
    public StripeClient(HttpClient client) => _client = client;
    public Task<ChargeResult> ChargeAsync(ChargeRequest req, CancellationToken ct)
        => _client.PostAsJsonAsync<ChargeResult>("/v1/charges", req, ct);
}

// Registration with Polly resilience pipeline
builder.Services.AddHttpClient<IStripeClient, StripeClient>(client =>
{
    client.BaseAddress = new Uri(config["Stripe:BaseUrl"]!);
    client.DefaultRequestHeaders.Add("Authorization", $"Bearer {config["Stripe:ApiKey"]}");
})
.AddPolicyHandler(Policy<HttpResponseMessage>
    .Handle<HttpRequestException>()
    .OrResult(r => !r.IsSuccessStatusCode)
    .WaitAndRetryAsync(3, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))))
.AddPolicyHandler(Policy<HttpResponseMessage>
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));
```

`IHttpClientFactory` manages a pool of `HttpMessageHandler` instances — clients are reused correctly, DNS changes are respected, no socket exhaustion.

---

## IAsyncEnumerable — Streaming Large Responses

When a query returns thousands of rows, don't buffer all into memory first. Stream directly to the client.

```csharp
// Query handler — yields rows as they come from DB
public async IAsyncEnumerable<OrderSummaryDto> Handle(
    StreamOrdersQuery query,
    [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var order in _context.Orders
        .AsNoTracking()
        .Where(o => o.CreatedAt >= query.From)
        .Select(o => new OrderSummaryDto(o.Id, o.Status, o.TotalAmount))
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return order;
    }
}

// Controller — ASP.NET Core serializes the stream as a JSON array
[HttpGet("orders/stream")]
public IAsyncEnumerable<OrderSummaryDto> StreamOrders([FromQuery] StreamOrdersQuery query, CancellationToken ct)
    => _mediator.CreateStream(query, ct);
// MediatR IStreamRequestHandler + IAsyncEnumerable — same pattern
```

Memory stays flat regardless of result set size. Client receives rows as they arrive.

---

## 202 Accepted — Long-Running Operations

For operations that take more than a few seconds: accept immediately, return a job ID, let client poll.

```csharp
// 1. Controller accepts immediately
[HttpPost("reports/generate")]
public async Task<IActionResult> GenerateReport([FromBody] GenerateReportRequest request)
{
    var jobId = await _mediator.Send(new QueueReportJobCommand(request));
    return Accepted(new { jobId, statusUrl = $"/api/v1/reports/jobs/{jobId}" });
    // HTTP 202 — "I received your request, working on it"
}

// 2. Client polls the status endpoint
[HttpGet("reports/jobs/{jobId}")]
public async Task<IActionResult> GetJobStatus(Guid jobId)
{
    var job = await _mediator.Send(new GetJobStatusQuery(jobId));
    return job.Status switch
    {
        JobStatus.Pending    => Accepted(job),           // 202 — still running
        JobStatus.Completed  => Ok(job),                 // 200 — result ready
        JobStatus.Failed     => UnprocessableEntity(job) // 422 — failed with reason
    };
}
```

---

## Webhooks — Sending and Receiving

**Sending webhooks** (your API notifies external systems):
```csharp
public class WebhookDispatcher : IWebhookDispatcher
{
    private readonly IHttpClientFactory _factory;

    public async Task SendAsync(string url, object payload, string secret, CancellationToken ct)
    {
        var json = JsonSerializer.Serialize(payload);
        var signature = ComputeHmac(json, secret); // HMAC-SHA256

        var client = _factory.CreateClient();
        client.DefaultRequestHeaders.Add("X-Webhook-Signature", signature);
        await client.PostAsJsonAsync(url, payload, ct);
    }

    private static string ComputeHmac(string payload, string secret)
    {
        var key = Encoding.UTF8.GetBytes(secret);
        var data = Encoding.UTF8.GetBytes(payload);
        var hash = HMACSHA256.HashData(key, data);
        return Convert.ToHexString(hash).ToLower();
    }
}
```

**Receiving webhooks** (external systems notify you — e.g. Stripe):
```csharp
[HttpPost("webhooks/stripe")]
public async Task<IActionResult> StripeWebhook()
{
    // 1. Read raw body — must read before model binding
    var json = await new StreamReader(Request.Body).ReadToEndAsync();

    // 2. Verify signature — reject if tampered
    var signature = Request.Headers["Stripe-Signature"].ToString();
    if (!VerifyStripeSignature(json, signature, _config["Stripe:WebhookSecret"]))
        return Unauthorized();

    // 3. Process event
    var stripeEvent = JsonSerializer.Deserialize<StripeEvent>(json);
    await _mediator.Send(new HandleStripeEventCommand(stripeEvent));

    return Ok(); // always return 200 — Stripe retries on non-2xx
}
```

---

## PATCH vs PUT Semantics

| Method | Semantics | Use when |
|---|---|---|
| **PUT** | Replace entire resource | Client sends full object |
| **PATCH** | Partial update — only fields provided | Client sends only changed fields |

```csharp
// PUT — replace whole order
[HttpPut("orders/{id}")]
public Task<IActionResult> UpdateOrder(Guid id, [FromBody] UpdateOrderRequest request)

// PATCH — update only the shipping address
[HttpPatch("orders/{id}/shipping")]
public Task<IActionResult> UpdateShipping(Guid id, [FromBody] UpdateShippingRequest request)
// Only contains the fields allowed to change — mapper ignores null/missing fields
```

**JSON Patch** (`Microsoft.AspNetCore.JsonPatch`) is the RFC standard for PATCH but adds complexity. For most APIs, a simple partial-update DTO is enough.

---

## System.Text.Json Configuration

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
    options.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter()); // enum as string
    options.SerializerOptions.WriteIndented = false; // production: compact JSON
});

// IMPORTANT: Never create JsonSerializerOptions per-request
// ✅ Register once as singleton or use the static options
// ❌ new JsonSerializerOptions() in a loop = memory/perf problem
private static readonly JsonSerializerOptions _options = new()
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase
};
```

---

## Feature Flags

Control feature availability without deployment — gradual rollout, A/B testing, kill switch.

```csharp
// NuGet: Microsoft.FeatureManagement.AspNetCore
builder.Services.AddFeatureManagement();

// appsettings.json
{
  "FeatureManagement": {
    "NewCheckoutFlow": true,
    "ExperimentalSearch": false
  }
}

// In a controller or handler
public class CheckoutController : ControllerBase
{
    private readonly IFeatureManager _features;

    [HttpPost("checkout")]
    public async Task<IActionResult> Checkout([FromBody] CheckoutRequest request)
    {
        if (await _features.IsEnabledAsync("NewCheckoutFlow"))
            return await NewCheckout(request);
        return await LegacyCheckout(request);
    }
}

// Filter — gate entire controller action
[FeatureGate("NewCheckoutFlow")]
[HttpPost("checkout/new")]
public IActionResult NewCheckout() { }
```

---

## OWASP API Security Top 10

Different from OWASP Web Top 10 — specific to APIs.

| # | Vulnerability | Prevention |
|---|---|---|
| **API1** | Broken Object Level Authorization (BOLA) | Always verify `userId == resource.OwnerId` — never trust client-supplied IDs alone |
| **API2** | Broken Authentication | Short-lived tokens, PKCE, rotate refresh tokens |
| **API3** | Broken Object Property Level Auth | Never expose fields user shouldn't see — use DTOs, not entities |
| **API4** | Unrestricted Resource Consumption | Rate limiting, pagination limits, file size limits |
| **API5** | Broken Function Level Authorization | Separate admin endpoints, check roles explicitly |
| **API6** | Unrestricted Access to Sensitive Business Flows | Rate limit checkout, OTP, password reset |
| **API7** | Server-Side Request Forgery (SSRF) | Validate/whitelist URLs before fetching external resources |
| **API8** | Security Misconfiguration | Disable Swagger in prod, check default credentials, CORS policy |
| **API9** | Improper Inventory Management | Version your API, deprecate old versions, track all endpoints |
| **API10** | Unsafe Consumption of APIs | Validate/sanitize responses from third-party APIs before using |

**API1 (BOLA) is the most common API vulnerability.** Always check ownership:
```csharp
// ❌ Wrong — trusts the route ID without checking ownership
var order = await _repo.GetByIdAsync(orderId);
return Ok(order);

// ✅ Correct — verifies the order belongs to the calling user
var userId = User.GetUserId();
var order = await _repo.GetByIdAsync(orderId);
if (order.UserId != userId) return Forbid();
return Ok(order);
```

---

## Minimal APIs

Lighter alternative to controllers — no class required, just route handlers.

```csharp
// Program.cs or a separate file via extension method
app.MapGet("/api/v1/orders/{id}", async (
    Guid id,
    IMediator mediator,
    CancellationToken ct) =>
{
    var result = await mediator.Send(new GetOrderByIdQuery(id), ct);
    return result is null ? Results.NotFound() : Results.Ok(result);
})
.WithName("GetOrder")
.RequireAuthorization()
.Produces<OrderDto>()
.ProducesProblem(StatusCodes.Status404NotFound);

// IEndpointFilter — equivalent of action filters for Minimal APIs
app.MapPost("/api/v1/orders", HandlePlaceOrder)
   .AddEndpointFilter<ValidationFilter<PlaceOrderRequest>>();
```

When to choose Minimal APIs over Controllers:
- Simple CRUD modules with few cross-cutting concerns
- Microservices with small surface area
- Performance-critical paths (slightly less overhead)

---

## Content Negotiation + Model Binding

**Content negotiation** — client tells server what format it wants:
```
Accept: application/json          ← I want JSON back
Accept: application/xml           ← I want XML back
Content-Type: application/json    ← I'm sending JSON
```

ASP.NET Core handles this automatically when you add XML formatters. Default is JSON only.

**Model binding sources** — be explicit:
```csharp
public IActionResult CreateOrder(
    [FromRoute] Guid customerId,     // from /api/customers/{customerId}/orders
    [FromQuery] string? referral,    // from ?referral=abc
    [FromBody] CreateOrderRequest request,  // from request body (JSON)
    [FromHeader(Name = "X-Source")] string? source) // from header
```

Rule: one `[FromBody]` per action. Everything else uses `[FromRoute]`/`[FromQuery]`/`[FromHeader]`.

---

## ValidationProblemDetails vs ProblemDetails

Two different error shapes — know which to return when.

```
ProblemDetails              ← generic error (auth failure, not found, conflict)
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Not Found",
  "status": 404,
  "detail": "Order abc not found"
}

ValidationProblemDetails    ← validation errors (400 Bad Request from FluentValidation)
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Quantity": ["Quantity must be greater than 0"],
    "ProductId": ["ProductId is required"]
  }
}
```

```csharp
// Return ValidationProblemDetails manually
return ValidationProblem(new ValidationProblemDetails(ModelState));

// FluentValidation + ValidationFilter returns this shape automatically
```

---

## Multi-Tenancy Strategies

| Strategy | How tenant is identified | When to use |
|---|---|---|
| **Header** | `X-Tenant-Id: tenant123` | Internal APIs, B2B |
| **JWT claim** | `tenantId` claim in the token | Most common — auth already required |
| **Subdomain** | `tenant123.yourapp.com` | SaaS with white-labelling |
| **Path** | `/tenant123/api/orders` | Least common |

**Implementation with Global Query Filter:**
```csharp
// Infrastructure: resolve current tenant
public interface ITenantContext { Guid TenantId { get; } }

public class JwtTenantContext : ITenantContext
{
    private readonly IHttpContextAccessor _accessor;
    public Guid TenantId =>
        Guid.Parse(_accessor.HttpContext!.User.FindFirstValue("tenantId")!);
}

// DbContext: filter every query
public class OrdersDbContext : DbContext
{
    private readonly ITenantContext _tenant;

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == _tenant.TenantId);
        // all Order queries automatically scoped to current tenant
    }
}
```

---

## Connection Pooling

Two layers — both managed automatically but worth understanding:

**SQL Server connection pool** (ADO.NET level):
- Connections are reused, not closed/opened per request
- Default pool size: 100 connections
- `Min Pool Size`, `Max Pool Size` in connection string
- Exhausted pool → `TimeoutException` after 30s

**DbContext pooling** (EF Core level):
- Reuse `DbContext` instances across requests — expensive to create
- ```csharp
  services.AddDbContextPool<OrdersDbContext>(opts => ..., poolSize: 128);
  ```
- Resets state between uses — don't store request-level data on the context

```csharp
// Connection string options
"Server=localhost;Database=ECommerce;Min Pool Size=5;Max Pool Size=100;Connection Timeout=30"
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

Level key: `[J]` Junior · `[M]` Mid · `[S]` Senior

```
API Knowledge
│
├── ══ JUNIOR ═══════════════════════════════════════════════════
│   ├── HTTP verbs, status codes, REST             [J] http-rest.md
│   ├── Controllers, routing, model binding        [J] aspnet.md
│   ├── [FromBody] / [FromQuery] / [FromRoute]     [J]
│   ├── Swagger / OpenAPI documentation            [J]
│   ├── ProblemDetails — what it looks like        [J]
│   ├── ValidationProblemDetails vs ProblemDetails [J]
│   ├── IFormFile upload + FileStreamResult        [J]
│   ├── PATCH vs PUT semantics                     [J]
│   └── Content negotiation + model binding        [J]
│
├── ══ MID ══════════════════════════════════════════════════════
│   │
│   ├── Security
│   │   ├── JWT validation setup                   [M] authentication.md
│   │   ├── Role-based + policy-based authz        [M]
│   │   ├── PKCE + refresh token rotation          [M]
│   │   ├── CORS, CSRF, XSS, security headers      [M] security-depth.md
│   │   ├── OWASP API Security Top 10 (BOLA etc.)  [M]
│   │   └── Rate limiting — .NET 8 built-in        [M]
│   │
│   ├── Middleware Pipeline
│   │   ├── Correct 9-step order                   [M]
│   │   └── Custom middleware (IMiddleware)         [M] [b]
│   │
│   ├── API Design
│   │   ├── API versioning (URL / header)           [M]
│   │   ├── Pagination + PagedResult<T>             [M]
│   │   ├── Filtering + sorting on list endpoints   [M]
│   │   ├── Idempotency keys on POST                [M]
│   │   ├── 202 Accepted + job polling              [M]
│   │   └── Webhooks — send + receive + HMAC        [M]
│   │
│   ├── Caching
│   │   ├── IMemoryCache (single instance)          [M]
│   │   ├── IDistributedCache → Redis               [M] [b]
│   │   ├── Output caching (.NET 8)                 [M]
│   │   └── ETag + Cache-Control headers            [M]
│   │
│   ├── HTTP Clients
│   │   ├── IHttpClientFactory — typed + named      [M]
│   │   └── Polly + IHttpClientFactory wiring       [M]
│   │
│   ├── Real-Time
│   │   ├── SignalR (bi-directional)                [M]
│   │   └── SSE (server push only)                  [M]
│   │
│   ├── Streaming
│   │   └── IAsyncEnumerable<T> large responses     [M]
│   │
│   ├── Background Work
│   │   ├── BackgroundService                        [M] [b]
│   │   ├── Channel<T> (in-process queue)           [M]
│   │   └── Hangfire (persistent scheduled jobs)    [M]
│   │
│   ├── Configuration
│   │   ├── Options pattern (IOptions variants)     [M]
│   │   ├── ValidateOnStart — fail fast             [M]
│   │   └── Secrets management                      [M]
│   │
│   ├── DI Advanced
│   │   ├── Keyed services (.NET 8)                 [M]
│   │   ├── Decorator pattern (Scrutor)             [M]
│   │   └── DateTime vs DateTimeOffset              [M]
│   │
│   ├── Feature Flags
│   │   └── Microsoft.FeatureManagement             [M]
│   │
│   ├── Minimal APIs
│   │   ├── Route handlers vs controllers           [M]
│   │   └── IEndpointFilter                         [M]
│   │
│   ├── Performance
│   │   ├── System.Text.Json configuration          [M]
│   │   └── Connection + DbContext pooling          [M]
│   │
│   └── Testing
│       ├── Unit (xUnit + Moq)                      [M] [b]
│       ├── Integration (Testcontainers)            [M] [b]
│       └── WebApplicationFactory (API layer)       [M]
│
└── ══ SENIOR ════════════════════════════════════════════════════
    ├── Multi-Tenancy at scale                       [S]
    │   ├── Tenant resolution strategies
    │   └── Global Query Filter per tenant
    ├── Distributed rate limiting (Redis-backed)     [S]
    ├── Observability
    │   ├── Structured logging (Serilog)             [S] [b]
    │   ├── Correlation ID                           [S] [b]
    │   ├── Health checks                            [S] [b]
    │   ├── OpenTelemetry — spans + exporters        [S] observability.md
    │   └── Metrics (p99, error rate, request rate)  [S]
    └── Deployment
        ├── Graceful shutdown + HostOptions          [S]
        ├── Response compression                     [S]
        ├── Docker + health check                    [S] [b]
        └── Zero-downtime deploys (blue/green)       [S] ci-cd.md
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
- `[ ]` IHttpClientFactory — typed + named clients
- `[ ]` Polly + IHttpClientFactory wiring
- `[ ]` IAsyncEnumerable<T> streaming responses
- `[ ]` 202 Accepted + job polling pattern
- `[ ]` Webhooks — send + receive + HMAC signature
- `[ ]` PATCH vs PUT semantics
- `[ ]` System.Text.Json configuration options
- `[ ]` Feature flags — Microsoft.FeatureManagement
- `[ ]` OWASP API Security Top 10
- `[ ]` Minimal APIs + IEndpointFilter
- `[ ]` Content negotiation + model binding sources
- `[ ]` ValidationProblemDetails vs ProblemDetails
- `[ ]` Multi-tenancy strategies + Global Query Filter
- `[ ]` Connection pooling + DbContext pooling
- `[ ]` Keyed services (.NET 8)
- `[ ]` Decorator pattern in DI (Scrutor)
- `[ ]` DateTime vs DateTimeOffset
