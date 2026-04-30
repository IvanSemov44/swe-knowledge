# C# Memory Management & Garbage Collection

<!-- level: Senior | GC generations, boxing/unboxing, Span<T>, ArrayPool, WeakReference -->

## What it is
How .NET allocates and reclaims memory automatically via the Garbage Collector (GC), and how you write code that works with it rather than against it.

## Why it matters
Poor memory management causes high GC pressure, memory leaks, and latency spikes. Understanding this helps you diagnose performance problems and write allocation-efficient code.

---

## Value Types vs Reference Types

| | Value Types | Reference Types |
|---|---|---|
| Examples | `int`, `bool`, `double`, `struct`, `enum` | `class`, `string`, `array`, `delegate` |
| Stored | On the stack (usually) | On the heap |
| Assignment | Copies the value | Copies the reference |
| Default | `0`, `false`, etc. | `null` |
| Nullable | Must use `int?` (Nullable<int>) | Nullable by default |

```csharp
// Value type — stack allocated, copied on assignment
int a = 5;
int b = a; // b is a copy
b = 10;
Console.WriteLine(a); // still 5

// Reference type — heap allocated, reference copied
var order1 = new Order { Total = 100 };
var order2 = order1; // both point to the same object
order2.Total = 200;
Console.WriteLine(order1.Total); // 200 — same object!

// Struct vs class
public struct Point { public int X, Y; }   // value type — copied
public class Order  { public decimal Total; } // reference type — referenced
```

**When to use struct:**
- Small, simple data (coordinates, color, money amount)
- Immutable or nearly immutable
- No inheritance needed
- Frequently allocated in large arrays (avoids heap pressure)

---

## Boxing and Unboxing

Boxing = converting a value type to `object` (heap allocation).
Unboxing = extracting it back.

```csharp
int value = 42;
object boxed = value;    // BOXING — allocates on heap, copies value into it
int unboxed = (int)boxed; // UNBOXING — copies back to stack

// Where boxing silently happens:
ArrayList list = new ArrayList();
list.Add(42); // BOXING! Use List<int> instead

void Log(object message) { } // Calling Log(42) boxes the int
string.Format("{0}", 42);    // Boxes 42 — use string interpolation or $"{42}"
```

**Impact:** Each box = a heap allocation + GC pressure. Avoid in hot paths.

---

## Garbage Collector — Generations

.NET GC uses a **generational** model based on the observation that most objects die young.

```
Gen 0 — newest, smallest, collected most often (~milliseconds)
Gen 1 — survived one Gen 0 collection, medium-lived objects
Gen 2 — long-lived objects (singletons, caches, static data)
LOH   — Large Object Heap: objects > 85 KB (rarely compacted)
```

**GC lifecycle:**
1. Allocate on Gen 0
2. GC collects Gen 0 — survivors promoted to Gen 1
3. GC collects Gen 1 — survivors promoted to Gen 2
4. Gen 2 collection = full GC — most expensive

```csharp
// Checking GC stats in code (for diagnostics only)
long gen0 = GC.CollectionCount(0);
long gen1 = GC.CollectionCount(1);
long gen2 = GC.CollectionCount(2);
long totalMemory = GC.GetTotalMemory(forceFullCollection: false);

// NEVER call GC.Collect() in production code
// It forces a full GC, causes latency spikes, and the runtime manages it better
```

**LOH (Large Object Heap):**
- Objects > ~85 KB go here
- Not compacted by default → can cause fragmentation
- Avoid allocating large arrays repeatedly (use `ArrayPool<T>`)

---

## IDisposable — Proper Implementation

For objects that hold **unmanaged resources** (file handles, DB connections, HTTP connections).

```csharp
// Simple case — just wrap the managed resource
public class CsvExporter : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public CsvExporter(string path)
    {
        _writer = new StreamWriter(path);
    }

    public void Write(string line) => _writer.WriteLine(line);

    public void Dispose()
    {
        if (_disposed) return;
        _writer.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this); // no need for finalizer if Dispose was called
    }
}

// Usage — using statement guarantees Dispose even if exception thrown
using var exporter = new CsvExporter("output.csv");
exporter.Write("id,name,price");
// Dispose called automatically at end of scope
```

**Full IDisposable pattern with finalizer** (only if you hold actual unmanaged handles):
```csharp
public class NativeResource : IDisposable
{
    private IntPtr _handle;
    private bool _disposed;

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // Free managed resources here
        }
        // Free unmanaged handle regardless
        if (_handle != IntPtr.Zero)
        {
            NativeMethods.CloseHandle(_handle);
            _handle = IntPtr.Zero;
        }
        _disposed = true;
    }

    ~NativeResource() => Dispose(false); // finalizer — last resort if Dispose not called

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
}
```

**Rule:** If you implement `IDisposable`, always call `GC.SuppressFinalize(this)` in `Dispose` — it removes the object from the finalizer queue, avoiding a costly extra GC cycle.

---

## Span\<T\> and Memory\<T\>

Zero-allocation slices over arrays, strings, and memory — no copying.

```csharp
// Reading a substring without allocating a new string
string csv = "Ivan,32,Bulgaria";
ReadOnlySpan<char> name = csv.AsSpan(0, 4); // "Ivan" — no allocation
int comma = csv.AsSpan().IndexOf(',');

// Slicing an array without copying
int[] numbers = { 1, 2, 3, 4, 5 };
Span<int> middle = numbers.AsSpan(1, 3); // [2, 3, 4] — same memory
middle[0] = 99; // modifies the original array

// Processing large byte arrays (e.g., network buffers)
void Process(ReadOnlySpan<byte> data) { }

// Memory<T> — heap-allocated version of Span, can be stored in fields
Memory<byte> buffer = new byte[1024].AsMemory();
await socket.ReceiveAsync(buffer, ct);
```

**Span<T> limitations:**
- Cannot be stored in a class field (stack-only)
- Cannot be used across `await` points
- Use `Memory<T>` when you need to store or pass across await

---

## ArrayPool\<T\>

Rent and return arrays to avoid repeated heap allocations.

```csharp
// Instead of allocating a new buffer every request:
byte[] buffer = new byte[4096]; // heap allocation every time

// Rent from the pool (no allocation if pool has one available):
byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
try
{
    int bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096), ct);
    Process(buffer.AsSpan(0, bytesRead));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer); // return to pool
}
```

**When to use:** Any code in a hot path that repeatedly allocates byte arrays, char arrays, or other large buffers (serialization, network I/O, parsers).

---

## String Interning

```csharp
// String literals are interned — same reference
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // true — same object

// Runtime strings are NOT automatically interned
string c = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(ReferenceEquals(a, c)); // false

// Manual interning
string interned = string.Intern(c);
Console.WriteLine(ReferenceEquals(a, interned)); // true

// String concatenation in loops — don't do this
string result = "";
for (int i = 0; i < 10000; i++)
    result += i; // allocates a NEW string each iteration → 10000 allocations

// Use StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 10000; i++) sb.Append(i);
string result = sb.ToString(); // one allocation
```

---

## WeakReference

A reference that doesn't prevent GC from collecting the object.

```csharp
// Useful for caches where you don't want to keep objects alive indefinitely
var weakRef = new WeakReference<Product>(expensiveProduct);

// Try to get the target
if (weakRef.TryGetTarget(out var product))
{
    // product is still alive — use it
}
else
{
    // product was GC'd — reload it
    product = LoadProduct(id);
    weakRef.SetTarget(product);
}
```

---

## Common Interview Questions

1. What is the difference between a value type and a reference type? Where are each stored?
2. What is boxing? Where does it happen silently?
3. What are GC generations and why does .NET use them?
4. When should you implement `IDisposable`? What is the `using` statement?
5. What is the difference between `Span<T>` and `Memory<T>`?
6. When would you use `ArrayPool<T>`?
7. Why should you never call `GC.Collect()` in production?

---

## Common Mistakes

- Using `ArrayList` or non-generic collections (boxing)
- String concatenation in loops (use `StringBuilder`)
- Implementing `IDisposable` but forgetting `GC.SuppressFinalize(this)`
- Storing `Span<T>` in a class field (not allowed — stack-only)
- Creating large byte arrays per-request instead of using `ArrayPool<T>`
- Calling `GC.Collect()` — you almost never know better than the runtime

---

## How It Connects

- Value types in your domain: `record struct Money(decimal Amount, string Currency)` — stack allocated, no heap pressure
- IDisposable is used by `DbContext`, `HttpClient`, `FileStream` — always `using`
- `Span<T>` is used in high-performance JSON parsing (System.Text.Json internally)
- `ArrayPool<T>` is used in ASP.NET Core's request body reading
- GC generations explain why allocating many short-lived objects (e.g., per-request DTOs) is fine — Gen 0 is cheap

---

## My Confidence Level
- `[ ]` Value types vs reference types — stack vs heap
- `[ ]` Boxing and unboxing — where it happens, impact
- `[ ]` GC generations (Gen0/1/2) and LOH
- `[b]` IDisposable — using statement, Dispose pattern
- `[~]` Span<T>, Memory<T> — zero-allocation slices
- `[ ]` ArrayPool<T> — renting buffers
- `[ ]` String interning and StringBuilder
- `[ ]` WeakReference — cache-friendly references

## My Notes
<!-- Personal notes -->
