# Observability

## What it is
Observability is the ability to understand what's happening inside your system from the outside. You can't fix what you can't see. Three pillars: **Logs, Metrics, Traces**.

---

## The Three Pillars

### 1. Logs — What happened

A record of discrete events. Something happened at a specific time.

```csharp
// Bad — unstructured, hard to search/filter
_logger.LogInformation("Order 123 placed by user 456 for $99.99");

// Good — structured, every field is queryable
_logger.LogInformation("Order placed {OrderId} {UserId} {Amount}",
    order.Id, order.UserId, order.TotalAmount);
```

Structured logs let you filter: `OrderId = "123"` or `Amount > 100` across millions of entries.

**In .NET:** Serilog is the standard. Outputs to console, files, Seq, Elasticsearch.

```csharp
// Program.cs
builder.Host.UseSerilog((ctx, config) =>
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .WriteTo.Console()
        .WriteTo.Seq("http://localhost:5341"));
```

**Log levels — use them correctly:**
| Level | When |
|---|---|
| `Trace` | Extremely detailed, dev only |
| `Debug` | Useful in development |
| `Information` | Normal business events (order placed, user logged in) |
| `Warning` | Something unexpected but recoverable |
| `Error` | Something failed, needs attention |
| `Critical` | System is down |

---

### 2. Metrics — How is it performing

Numeric measurements over time. Answers: is the system healthy right now?

Key metrics to track:
- **Request rate** — requests per second
- **Error rate** — % of requests that fail (5xx)
- **Latency** — p50, p95, p99 response times (not average — averages hide outliers)
- **CPU / Memory** — server resource usage
- **Queue depth** — how many jobs are waiting

```csharp
// .NET — expose metrics via OpenTelemetry
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics => metrics
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());

app.MapPrometheusScrapingEndpoint(); // /metrics endpoint
```

**Tools:** Prometheus (collects) + Grafana (visualizes dashboards).

---

### 3. Distributed Tracing — Where did it go

Follows a single request as it travels across multiple services. Answers: where is the slowness? Which service is failing?

```
Request → Gateway (2ms) → Products API (45ms) → Database (200ms) → Response
                                                      ↑
                                               Slow query here
```

Each service adds a **trace span** with timing. All spans share a `TraceId`.

```csharp
// .NET — OpenTelemetry auto-instruments ASP.NET Core + HttpClient + EF Core
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation()
        .AddOtlpExporter()); // sends to Jaeger, Zipkin, etc.
```

**Tools:** Jaeger, Zipkin, Azure Application Insights, Grafana Tempo.

---

## Health Checks

Every service should expose a `/health` endpoint. Load balancers and orchestrators (Kubernetes) poll it to know if an instance is alive.

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis(redisConnectionString)
    .AddCheck("disk_space", () =>
        DriveInfo.GetDrives()[0].AvailableFreeSpace > 1_000_000_000
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Degraded("Low disk space"));

app.MapHealthChecks("/health");
app.MapHealthChecks("/health/ready"); // is it ready to serve traffic?
app.MapHealthChecks("/health/live");  // is the process alive?
```

---

## The Three Together — A Real Scenario

**Scenario:** Users report the checkout is slow.

1. **Metrics** → Grafana dashboard shows p99 latency on Orders API spiked from 100ms to 2s at 14:32
2. **Traces** → Jaeger shows the slow span is inside `PlaceOrderCommand` → `DbContext.SaveChangesAsync`
3. **Logs** → Serilog shows `Executing DbCommand` taking 1800ms on a specific query

Result: missing index on `Orders.UserId`. Found and fixed in 15 minutes instead of hours of guessing.

Without observability you'd be blind.

---

## Correlation ID

Every request gets a unique ID. Pass it through every service and log it everywhere. When a user reports "my order failed at 14:32", you search by CorrelationId and see the full trail.

```csharp
// Middleware — generate or forward correlation ID
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"]
        .FirstOrDefault() ?? Guid.NewGuid().ToString();

    context.Response.Headers["X-Correlation-ID"] = correlationId;

    using (LogContext.PushProperty("CorrelationId", correlationId))
    {
        await next();
    }
});
```

---

## Common Interview Questions

1. What are the three pillars of observability?
2. What is the difference between logging and metrics?
3. What is distributed tracing and why do you need it in microservices?
4. Why use p99 latency instead of average latency?
5. What is a health check endpoint and who calls it?
6. What is a Correlation ID?

---

## Common Mistakes

- Using unstructured logs (`string.Format`) instead of structured logging
- Only logging errors — log important business events at Info level too
- Tracking average latency — use percentiles (p95, p99)
- Not adding health checks — load balancer can't detect a dead instance
- Not propagating Correlation IDs across service boundaries

---

## How It Connects

- Load balancer polls `/health/live` — removes unhealthy instances automatically
- Kubernetes uses `/health/ready` to decide when to route traffic to a new pod
- API Gateway adds `X-Correlation-ID` header — all downstream services log it
- OpenTelemetry is the standard — one library, multiple backends (Jaeger, Grafana, Azure)

---

## My Confidence Level
- `[ ]` Structured logging with Serilog
- `[ ]` Log levels — when to use each
- `[ ]` Metrics — request rate, error rate, p99 latency
- `[ ]` Distributed tracing — TraceId, spans
- `[ ]` Health checks — /health/live vs /health/ready
- `[ ]` Correlation ID pattern
- `[ ]` OpenTelemetry setup in .NET

## My Notes
<!-- Personal notes -->
