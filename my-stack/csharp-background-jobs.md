# Background Jobs in .NET

## What it is
Work that runs outside the HTTP request cycle — scheduled tasks, queue consumers, cleanup jobs. .NET provides `IHostedService` and `BackgroundService`. Hangfire adds a persistent job queue with a dashboard.

## Why it matters
Real applications always have background work: sending emails, processing uploads, running nightly reports, consuming message queues. Interviewers expect you to know the built-in abstractions and when to reach for a library.

---

## IHostedService

The low-level interface. Implement `StartAsync` and `StopAsync`.

```csharp
public class MyHostedService : IHostedService, IDisposable
{
    private readonly ILogger<MyHostedService> _logger;
    private Timer? _timer;

    public MyHostedService(ILogger<MyHostedService> logger)
        => _logger = logger;

    public Task StartAsync(CancellationToken ct)
    {
        _logger.LogInformation("Service starting.");
        _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromMinutes(5));
        return Task.CompletedTask;
    }

    private void DoWork(object? state)
    {
        _logger.LogInformation("Running scheduled work at {Time}", DateTimeOffset.UtcNow);
    }

    public Task StopAsync(CancellationToken ct)
    {
        _timer?.Change(Timeout.Infinite, 0);
        return Task.CompletedTask;
    }

    public void Dispose() => _timer?.Dispose();
}

// Program.cs
builder.Services.AddHostedService<MyHostedService>();
```

---

## BackgroundService (preferred base class)

Abstract base that handles the boilerplate. Override `ExecuteAsync`.

```csharp
public class OrderCleanupService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<OrderCleanupService> _logger;

    public OrderCleanupService(
        IServiceScopeFactory scopeFactory,
        ILogger<OrderCleanupService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Order cleanup service started.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessAsync(stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                _logger.LogError(ex, "Error in order cleanup — will retry.");
            }

            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }

    private async Task ProcessAsync(CancellationToken ct)
    {
        // MUST use IServiceScopeFactory — BackgroundService is Singleton
        // DbContext is Scoped — can't inject directly
        using var scope = _scopeFactory.CreateScope();
        var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();

        var cutoff = DateTime.UtcNow.AddDays(-30);
        var staleOrders = await uow.Orders.GetAbandonedBeforeAsync(cutoff, ct);

        foreach (var order in staleOrders)
            order.Cancel(reason: "Abandoned");

        await uow.CommitAsync(ct);
        _logger.LogInformation("Cancelled {Count} stale orders.", staleOrders.Count);
    }
}

// Program.cs
builder.Services.AddHostedService<OrderCleanupService>();
```

**Key rule:** `BackgroundService` is registered as **Singleton**. Never inject `DbContext` or `IUnitOfWork` directly — always use `IServiceScopeFactory`.

---

## Producer/Consumer with Channel\<T\>

`System.Threading.Channels` — high-performance, in-process async queue. Ideal for decoupling producers from slow consumers within the same process.

```csharp
// The channel — in-memory queue
public class OrderProcessingChannel
{
    private readonly Channel<Guid> _channel =
        Channel.CreateBounded<Guid>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait
        });

    public ChannelReader<Guid> Reader => _channel.Reader;
    public ChannelWriter<Guid> Writer => _channel.Writer;
}

// Producer — writes to channel (e.g., from a controller)
public class OrdersController : ControllerBase
{
    private readonly OrderProcessingChannel _channel;

    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request, CancellationToken ct)
    {
        var orderId = await _mediator.Send(request.ToCommand(), ct);
        await _channel.Writer.WriteAsync(orderId, ct); // fire and continue
        return Accepted(new { orderId });
    }
}

// Consumer — background service reads from channel
public class OrderProcessingWorker : BackgroundService
{
    private readonly OrderProcessingChannel _channel;
    private readonly IServiceScopeFactory _scopeFactory;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var orderId in _channel.Reader.ReadAllAsync(stoppingToken))
        {
            using var scope = _scopeFactory.CreateScope();
            var processor = scope.ServiceProvider.GetRequiredService<IOrderProcessor>();
            await processor.ProcessAsync(orderId, stoppingToken);
        }
    }
}

// Register channel as Singleton (shared between producer and consumer)
builder.Services.AddSingleton<OrderProcessingChannel>();
builder.Services.AddHostedService<OrderProcessingWorker>();
```

---

## Graceful Shutdown

When the app stops (Ctrl+C, deploy restart), .NET sends a `CancellationToken` to `ExecuteAsync`. Respect it.

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await DoUnitOfWorkAsync(stoppingToken); // passes token down
        await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken); // throws on cancel
    }
    // Clean up here — runs after cancellation
    _logger.LogInformation("Service stopped cleanly.");
}
```

Configure shutdown timeout (default 5s — often too short):
```csharp
builder.Services.Configure<HostOptions>(opts =>
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30));
```

---

## Hangfire — Persistent Background Jobs

For jobs that must survive app restarts, need retry logic, or require a visual dashboard.

```bash
dotnet add package Hangfire.AspNetCore
dotnet add package Hangfire.SqlServer
```

```csharp
// Program.cs
builder.Services.AddHangfire(config =>
    config.UseSqlServerStorage(connectionString));
builder.Services.AddHangfireServer();

app.UseHangfireDashboard("/jobs"); // visual dashboard at /jobs

// Enqueue a fire-and-forget job
BackgroundJob.Enqueue<IEmailService>(
    svc => svc.SendOrderConfirmationAsync(orderId, CancellationToken.None));

// Schedule a job for later
BackgroundJob.Schedule<IReportService>(
    svc => svc.GenerateDailyReport(),
    TimeSpan.FromHours(1));

// Recurring job (cron expression)
RecurringJob.AddOrUpdate<IInventoryService>(
    "sync-inventory",
    svc => svc.SyncFromWarehouseAsync(CancellationToken.None),
    Cron.Daily);
```

**Hangfire vs BackgroundService:**

| | BackgroundService | Hangfire |
|---|---|---|
| Persistence | No — lost on restart | Yes — stored in DB |
| Retry on failure | Manual | Automatic (configurable) |
| Visual dashboard | No | Yes |
| Scheduling (cron) | Manual | Built-in |
| Setup complexity | Zero | Needs DB + package |
| Use for | Simple periodic tasks, queue consumers | Reliable delivery, retries, monitoring |

---

## Common Interview Questions

1. What is `IHostedService`? What is `BackgroundService`?
2. Why can't you inject `DbContext` directly into a `BackgroundService`?
3. How do you access Scoped services from a Singleton background service?
4. What is `Channel<T>` and when would you use it?
5. What is graceful shutdown and how do you implement it?
6. When would you choose Hangfire over a plain `BackgroundService`?

---

## Common Mistakes

- Injecting `DbContext` or `IUnitOfWork` directly into `BackgroundService` (captive dependency → shared context across jobs)
- Not catching exceptions in the `ExecuteAsync` loop — one failure stops the entire service forever
- Ignoring `stoppingToken` — app hangs on shutdown waiting for background work to finish
- Using `Thread.Sleep` instead of `await Task.Delay(delay, stoppingToken)`
- Using `BackgroundService` for jobs that need reliable delivery — use Hangfire if restart = lost job is unacceptable

---

## How It Connects

- `BackgroundService` + `IServiceScopeFactory` is the pattern for outbox message relay (from `event-driven-flow.md`)
- `Channel<T>` decouples your API from slow downstream processing — request returns fast, work happens asynchronously
- Graceful shutdown + `CancellationToken` ties into `CancellationToken` propagation from `csharp-concurrency.md`
- Hangfire jobs are idempotent by design — same guarantee as the inbox pattern from `event-driven-flow.md`

---

## My Confidence Level
- `[ ]` IHostedService — StartAsync/StopAsync lifecycle
- `[ ]` BackgroundService — ExecuteAsync loop, error handling
- `[ ]` IServiceScopeFactory in background services — why needed
- `[ ]` Channel<T> — producer/consumer pattern
- `[ ]` Graceful shutdown — CancellationToken + shutdown timeout
- `[ ]` Hangfire — fire-and-forget, scheduled, recurring jobs
- `[ ]` Hangfire vs BackgroundService — when to choose

## My Notes
<!-- Personal notes -->
