# C# Concurrency & Thread Safety

<!-- level: Mid → Senior | lock, SemaphoreSlim, async pitfalls, CancellationToken, concurrent collections -->

## What it is
Managing multiple operations executing simultaneously. In .NET this covers threads, the thread pool, async/await mechanics, shared state protection, and safe patterns for concurrent code.

## Why it matters
Every ASP.NET Core server handles requests concurrently. Race conditions, deadlocks, and thread starvation cause data corruption and silent failures in production.

---

## Thread vs Task

| | Thread | Task |
|---|---|---|
| Abstraction | OS thread | Logical unit of async work |
| Cost | ~1MB stack, expensive | Lightweight, reuses thread pool |
| Created via | `new Thread()` | `Task.Run()`, `async/await` |
| Control | Manual start/stop | Managed by runtime |
| Use for | Long-running dedicated thread | I/O-bound or CPU-bound offload |

```csharp
// Thread — raw OS thread. Rarely the right choice.
var thread = new Thread(() => DoWork());
thread.IsBackground = true;
thread.Start();

// Task.Run — offloads CPU-bound work to thread pool
await Task.Run(() => ComputeHeavyReport());

// async/await — best for I/O (no thread consumed while waiting)
var data = await _httpClient.GetStringAsync(url); // thread released while waiting
```

**Rule:** Use `async/await` for I/O. Use `Task.Run` for CPU-heavy work. Avoid `new Thread()`.

---

## Race Conditions

Occur when multiple threads read and write shared mutable state without synchronization.

```csharp
// RACE CONDITION — ++ is NOT atomic (read → increment → write = 3 ops)
private int _counter = 0;

public void Increment() => _counter++;
// Thread A reads 5, Thread B reads 5, both write 6. Result: 6 instead of 7.
```

---

## lock (Monitor)

The most common primitive. Guarantees only one thread executes the locked block at a time.

```csharp
private readonly object _lock = new();
private int _counter = 0;

public void Increment()
{
    lock (_lock)
    {
        _counter++; // only one thread at a time
    }
}
```

**Rules:**
- Lock on a **private readonly object** — never `this`, never `typeof(T)`, never a string
- Keep locked sections **as short as possible**
- Never `await` inside a `lock` block — use `SemaphoreSlim` instead

```csharp
// WRONG — can't await inside lock
lock (_lock)
{
    await SomeAsync(); // compile error in some targets, logic error otherwise
}
```

---

## SemaphoreSlim

Like a lock but supports `await` and can allow **N** concurrent entries.

```csharp
// 1 concurrent entry = async mutex
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<string> GetCachedAsync(string key, CancellationToken ct)
{
    await _semaphore.WaitAsync(ct);
    try
    {
        if (_cache.TryGetValue(key, out var cached)) return cached;
        var value = await _db.LoadAsync(key, ct);
        _cache[key] = value;
        return value;
    }
    finally
    {
        _semaphore.Release(); // always release in finally
    }
}

// 3 concurrent entries = limit parallelism
private readonly SemaphoreSlim _dbSemaphore = new(3, 3);

public async Task<Data> QueryAsync(CancellationToken ct)
{
    await _dbSemaphore.WaitAsync(ct);
    try   { return await _db.ExecuteAsync(ct); }
    finally { _dbSemaphore.Release(); }
}
```

**`SemaphoreSlim(1, 1)` = the async equivalent of `lock`.** Use it whenever you need to await inside a critical section.

---

## Interlocked

Atomic operations on primitive types. Fastest option — no lock overhead.

```csharp
private int _requestCount = 0;

public void RecordRequest()
    => Interlocked.Increment(ref _requestCount); // atomic

public int GetAndReset()
    => Interlocked.Exchange(ref _requestCount, 0); // atomic read + reset

// Compare-and-swap
Interlocked.CompareExchange(ref _state, newValue, expectedValue);
```

Use for simple counters and flags. For anything composite, use `lock` or `SemaphoreSlim`.

---

## Thread-Safe Collections

```csharp
// ConcurrentDictionary — most commonly used
private readonly ConcurrentDictionary<string, Product> _cache = new();

_cache.TryAdd("key", product);
_cache.TryGetValue("key", out var value);

// GetOrAdd — atomic check-and-add
var product = _cache.GetOrAdd("key", key => LoadProduct(key));

// AddOrUpdate — atomic update
_cache.AddOrUpdate("key", product, (key, old) => product);

// ConcurrentQueue — thread-safe FIFO
private readonly ConcurrentQueue<WorkItem> _queue = new();
_queue.Enqueue(item);
if (_queue.TryDequeue(out var work)) ProcessWork(work);
```

**Caution:** Individual operations are atomic, but **composite operations are not**:

```csharp
// NOT ATOMIC — race condition between check and add
if (!_cache.ContainsKey(key))
    _cache.TryAdd(key, value); // another thread may add between these two lines

// ATOMIC — use GetOrAdd instead
_cache.GetOrAdd(key, _ => value);
```

---

## async/await Pitfalls

### Deadlock with .Result / .Wait()

The most common deadlock in older ASP.NET code.

```csharp
// DEADLOCK (old ASP.NET with SynchronizationContext)
public string GetData()
{
    return GetDataAsync().Result; // blocks the thread
    // GetDataAsync tries to resume on this same thread → deadlock
}

// SAFE in ASP.NET Core (no SynchronizationContext), but still wrong style.
// CORRECT — async all the way down:
public async Task<string> GetDataAsync()
{
    return await _service.FetchAsync();
}
```

**Rule: Never call `.Result` or `.Wait()` in async code. Go async all the way.**

### async void

```csharp
// DANGER — exceptions escape unhandled and crash the process
async void DoWork() { await Something(); }

// CORRECT — return Task so exceptions can be observed
async Task DoWorkAsync() { await Something(); }

// Fire-and-forget — the right way
_ = Task.Run(async () =>
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
});

// Exception: event handlers are allowed to be async void
submitButton.Click += async (s, e) => await HandleSubmitAsync();
```

### ConfigureAwait(false)

```csharp
// In library / shared code — prevents context capture, avoids potential deadlocks
public async Task<string> GetFromExternalAsync()
{
    var result = await _httpClient.GetStringAsync(url).ConfigureAwait(false);
    // Resumes on any thread pool thread, not the original context
    return result;
}

// In ASP.NET Core — not needed (no SynchronizationContext), but harmless
```

### Task.WhenAll vs sequential await

```csharp
// Sequential — total time = t1 + t2
var product = await _uow.Products.GetByIdAsync(productId, ct);
var user    = await _uow.Users.GetByIdAsync(userId, ct);

// Parallel — total time = max(t1, t2)
var productTask = _uow.Products.GetByIdAsync(productId, ct);
var userTask    = _uow.Users.GetByIdAsync(userId, ct);
await Task.WhenAll(productTask, userTask);
var product = productTask.Result; // safe — task is already completed
var user    = userTask.Result;
```

---

## CancellationToken

Cooperative cancellation — caller requests cancellation, callee checks and stops.

```csharp
public async Task ProcessOrdersAsync(CancellationToken ct = default)
{
    var orders = await _db.Orders
        .Where(o => o.Status == OrderStatus.Pending)
        .ToListAsync(ct);                           // pass to every async call

    foreach (var order in orders)
    {
        ct.ThrowIfCancellationRequested();          // check in loops
        await ProcessOrderAsync(order, ct);
    }
}

// ASP.NET Core automatically passes the request cancellation token
[HttpGet]
public async Task<IActionResult> GetProducts(CancellationToken ct)
{
    var result = await _mediator.Send(new GetProductsQuery(), ct);
    return Ok(result);
}
// If the client disconnects, ct is cancelled → your query is abandoned → no wasted DB work
```

**When to check `ct.ThrowIfCancellationRequested()`:**
- At the top of loops
- Between major stages of a long operation

---

## volatile

Prevents CPU/compiler from caching a field in a register. Only for simple flags — not a substitute for `lock`.

```csharp
private volatile bool _isRunning = true;

// On shutdown thread:
_isRunning = false;

// Worker loop always reads from memory, not a CPU cache:
while (_isRunning) { DoWork(); }
```

---

## Common Interview Questions

1. What is a race condition? Give a concrete example.
2. When would you use `SemaphoreSlim` instead of `lock`?
3. Why can `.Result` cause a deadlock? When is it actually safe?
4. What is `async void` and why should you avoid it?
5. What does `ConfigureAwait(false)` do? When do you need it?
6. What is the difference between `ConcurrentDictionary.TryAdd` and `GetOrAdd`?
7. How does `CancellationToken` work — who creates it and how does it cancel?

---

## Common Mistakes

- Awaiting inside a `lock` block (use `SemaphoreSlim`)
- Calling `.Result` or `.Wait()` from async code
- Using `async void` outside event handlers (exceptions silently crash the process)
- Locking on `this` or a public object (external code can cause deadlocks)
- Not propagating `CancellationToken` through the call chain
- Assuming composite `ConcurrentDictionary` operations are atomic

---

## How It Connects

- Every ASP.NET Core request runs on a thread pool thread — concurrency is automatic
- `CancellationToken` in controllers fires when the HTTP client disconnects
- `SemaphoreSlim` is how Polly implements bulkhead isolation
- `ConcurrentDictionary` powers in-memory caches and rate limiters
- `async void` is why background work uses `IHostedService.ExecuteAsync` not raw threads

---

## My Confidence Level
- `[ ]` Thread vs Task — cost and use cases
- `[ ]` Race conditions — recognizing and fixing
- `[ ]` lock / Monitor — rules and limits
- `[ ]` SemaphoreSlim — async critical sections
- `[ ]` Interlocked — atomic primitives
- `[ ]` Thread-safe collections — ConcurrentDictionary gotchas
- `[ ]` async/await pitfalls — deadlocks, .Result, async void
- `[ ]` ConfigureAwait(false) — when needed
- `[ ]` CancellationToken — propagation and checking

## My Notes
<!-- Personal notes -->
