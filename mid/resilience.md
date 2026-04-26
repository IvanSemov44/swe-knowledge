# Resilience Patterns & Polly

## What it is
Techniques to make distributed systems survive partial failures — a downstream service being slow, temporarily down, or returning errors.

## Why it matters
In any system calling external services (payment gateways, email providers, other microservices), failures are inevitable. Without resilience, one slow dependency can cascade and take down your entire application.

---

## Why Distributed Systems Fail

Every network call can:
- Fail (connection refused)
- Timeout (server too slow to respond)
- Return errors intermittently (overloaded server, brief outage)
- Succeed but return stale/corrupted data

Naive code retries immediately → thundering herd → makes the downstream worse.

---

## Retry with Exponential Backoff + Jitter

Retry after increasing delays. Add jitter (randomness) to prevent all instances retrying simultaneously (thundering herd problem).

```
Attempt 1 → fails
Wait 2s + jitter
Attempt 2 → fails
Wait 4s + jitter
Attempt 3 → fails
Wait 8s + jitter
Attempt 4 → success or give up
```

**Only retry transient errors** (timeouts, 503). Never retry 400/401/403/404 — those are permanent failures.

```csharp
// Polly v8 — ResiliencePipeline (new API as of Polly 8 / .NET 8)
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(2),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true,  // adds randomness to delay
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => r.StatusCode == HttpStatusCode.ServiceUnavailable)
    })
    .Build();

var response = await pipeline.ExecuteAsync(async ct =>
    await httpClient.GetAsync("/api/products", ct), cancellationToken);
```

---

## Circuit Breaker

Prevents repeated calls to a failing service. Instead of retrying every time and adding load, it "opens" and fails fast.

### States

```
CLOSED (normal)
   │ failures exceed threshold
   ▼
OPEN (failing fast — no calls go through)
   │ after break duration
   ▼
HALF-OPEN (probe — one test call goes through)
   │ test succeeds      │ test fails
   ▼                    ▼
CLOSED               OPEN again
```

| State | Behavior |
|---|---|
| **Closed** | All calls go through. Counts failures. |
| **Open** | All calls immediately throw — no network call made. |
| **Half-Open** | One trial call allowed. Success → Closed. Failure → Open. |

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,                            // open if 50% of calls fail
        SamplingDuration = TimeSpan.FromSeconds(30),   // within a 30s window
        MinimumThroughput = 10,                        // need at least 10 calls to evaluate
        BreakDuration = TimeSpan.FromSeconds(30),      // stay open for 30s
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .Handle<HttpRequestException>()
            .HandleResult(r => !r.IsSuccessStatusCode)
    })
    .Build();
```

**Why it matters:** If payment service is down, circuit breaker prevents 1000 requests/sec hammering the dying service. Instead: fail fast, return error to user, let the service recover.

---

## Timeout

Every external call needs a timeout. Without one, a slow service can exhaust your thread pool.

```csharp
var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddTimeout(TimeSpan.FromSeconds(5)) // throw TimeoutRejectedException after 5s
    .Build();
```

**Configure timeouts at multiple levels:**
- `HttpClient.Timeout` — per-client default
- Polly timeout — per-pipeline (can vary per operation)
- DB command timeout — `_context.Database.SetCommandTimeout(30)`

---

## Bulkhead Isolation

Limit the concurrency for a specific dependency so one slow service can't consume all your threads.

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(
        permitLimit: 10,        // max 10 concurrent calls to this service
        queueLimit: 5)          // queue up to 5 more; beyond that, reject immediately
    .Build();
```

**Analogy:** Bulkheads in ships are watertight compartments — a leak in one doesn't sink the whole ship.

**Without bulkhead:** Slow payment service uses all 500 threads → product catalog API also stops working.
**With bulkhead:** Payment service limited to 10 threads → everything else stays responsive.

---

## Fallback

Return a default value or cached result when all retries fail.

```csharp
var pipeline = new ResiliencePipelineBuilder<IEnumerable<Product>>()
    .AddFallback(new FallbackStrategyOptions<IEnumerable<Product>>
    {
        FallbackAction = _ =>
            ValueTask.FromResult<IEnumerable<Product>>(_cache.GetStaleProducts()),
        ShouldHandle = new PredicateBuilder<IEnumerable<Product>>()
            .Handle<Exception>()
    })
    .Build();
```

**Use when:** You can serve stale or degraded results rather than an error. Don't use when accuracy is critical (e.g., payment processing — fail clearly, don't silently return wrong data).

---

## Combining Policies (Pipeline)

Order matters: **outer policy wraps inner policy**.

```csharp
// Recommended order: Timeout → CircuitBreaker → Retry → (actual call)
// This means: each retry attempt has its own timeout, and the circuit breaker
// counts retry-exhausted attempts as failures.

var pipeline = new ResiliencePipelineBuilder<HttpResponseMessage>()
    .AddTimeout(TimeSpan.FromSeconds(5))          // 5s per attempt
    .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
    {
        MaxRetryAttempts = 3,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Exponential,
        UseJitter = true
    })
    .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
    {
        FailureRatio = 0.5,
        MinimumThroughput = 5,
        BreakDuration = TimeSpan.FromSeconds(30)
    })
    .Build();
```

---

## HttpClientFactory + Polly

The idiomatic way to wire Polly into ASP.NET Core services.

```csharp
// Program.cs
builder.Services
    .AddHttpClient<IPaymentGateway, StripePaymentGateway>(client =>
    {
        client.BaseAddress = new Uri("https://api.stripe.com");
        client.Timeout = TimeSpan.FromSeconds(30);
    })
    .AddResilienceHandler("payment", pipelineBuilder =>
    {
        pipelineBuilder
            .AddTimeout(TimeSpan.FromSeconds(5))
            .AddRetry(new RetryStrategyOptions<HttpResponseMessage>
            {
                MaxRetryAttempts = 3,
                UseJitter = true,
                BackoffType = DelayBackoffType.Exponential,
                Delay = TimeSpan.FromMilliseconds(500)
            })
            .AddCircuitBreaker(new CircuitBreakerStrategyOptions<HttpResponseMessage>
            {
                FailureRatio = 0.5,
                MinimumThroughput = 10,
                BreakDuration = TimeSpan.FromSeconds(30)
            });
    });

// The registered IPaymentGateway automatically uses the resilience pipeline
// No changes needed in the gateway implementation class
```

**Why HttpClientFactory:** Manages `HttpClient` lifetime, prevents socket exhaustion (from creating/disposing clients per request), and integrates cleanly with DI and Polly.

---

## When to Use Each Pattern

| Pattern | Use When |
|---|---|
| Retry | Transient failures expected (timeouts, 503). Don't retry 4xx. |
| Circuit Breaker | Downstream service has extended outages. Fail fast rather than wait. |
| Timeout | Any external call. No call should be unconstrained. |
| Bulkhead | Isolating one slow dependency from starving others. |
| Fallback | Degraded result is acceptable (stale cache, empty list). |

**Pattern combinations for external API calls:**
- Minimum: Timeout + Retry
- Recommended: Timeout + Retry + Circuit Breaker
- Full: Timeout + Retry + Circuit Breaker + Bulkhead + Fallback

---

## Common Interview Questions

1. What is the circuit breaker pattern? Describe its three states.
2. Why do you add jitter to retry delays?
3. What is bulkhead isolation? Give a real example.
4. What is the difference between retry and circuit breaker — when would you use each?
5. What happens if you don't have a timeout on an external call?
6. Why should you not retry 400 or 404 responses?
7. How does HttpClientFactory relate to Polly?

---

## Common Mistakes

- Retrying non-transient errors (400, 401, 404) — wastes resources, hides real bugs
- No jitter on retry → thundering herd makes the downstream worse
- Circuit breaker without minimum throughput → opens on the very first failure
- No timeout → thread pool exhaustion under slow downstream
- Not combining policies → retry without circuit breaker hammers a failing service

---

## How It Connects

- Polly wraps the Infrastructure layer's HTTP clients and external service calls
- Circuit breaker state is checked before calling `IPaymentGateway.ChargeAsync()`
- Bulkhead limits tie into the `SemaphoreSlim` concurrency concepts from `csharp-concurrency.md`
- Fallback can return a cached Redis result (from `caching.md`)
- Observability (from `observability.md`) should log every circuit open/close and retry event

---

## My Confidence Level
- `[ ]` Retry with exponential backoff + jitter
- `[ ]` Circuit breaker — three states, thresholds
- `[ ]` Timeout pattern
- `[ ]` Bulkhead isolation — concurrency limiter
- `[ ]` Fallback pattern
- `[ ]` Polly v8 ResiliencePipeline API
- `[ ]` HttpClientFactory + Polly integration
- `[ ]` Policy combination order and reasoning

## My Notes
<!-- Personal notes -->
