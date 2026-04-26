# API Design — Pagination, GraphQL, gRPC, HTTP/2

## What it is
Practical API design decisions beyond basic REST: how to paginate large datasets, when to use GraphQL or gRPC instead of REST, and how HTTP/2 and WebSockets change the picture.

## Why it matters
Mid-level interviews often ask "how would you design the pagination for this endpoint?" or "when would you use GraphQL over REST?" These are directly applicable to your E-commerce project.

---

## Pagination Strategies

Never return all records. Every collection endpoint needs pagination.

### Offset Pagination

```sql
SELECT * FROM Products ORDER BY Id OFFSET 40 ROWS FETCH NEXT 20 ROWS ONLY;
-- Page 3, page size 20 → skip 40, take 20
```

**API:**
```
GET /products?page=3&pageSize=20
```

**Response:**
```json
{
  "data": [...],
  "page": 3,
  "pageSize": 20,
  "totalCount": 253,
  "totalPages": 13
}
```

**Pros:** Simple to implement. Users can jump to page N. Easy to build "page 3 of 13" UI.

**Cons:**
- Slow at high offsets — DB must scan and discard all preceding rows: `OFFSET 10000` scans 10000 rows
- Inconsistent under concurrent writes — if a row is inserted on page 1 while you're reading page 2, you may see duplicates or skip rows

**Use when:** Small to medium datasets. Admin tables. Where page-jumping matters.

---

### Cursor (Keyset) Pagination

Instead of a page number, use the last-seen record's ID (or timestamp) as a cursor.

```sql
-- First page
SELECT * FROM Products ORDER BY Id FETCH NEXT 20 ROWS ONLY;

-- Next page — cursor is the Id of the last item from previous page
SELECT * FROM Products WHERE Id > @lastId ORDER BY Id FETCH NEXT 20 ROWS ONLY;
```

**API:**
```
GET /products?limit=20
GET /products?limit=20&cursor=eyJpZCI6MTIzfQ==   ← base64 encoded cursor
```

**Response:**
```json
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTQzfQ==",
  "hasMore": true
}
```

**Pros:**
- O(log n) regardless of how deep you are (indexed seek, not scan)
- Consistent under concurrent inserts — cursor anchors to a position

**Cons:**
- Can't jump to page N — only forward/backward
- Cursor must be opaque to the client (encode it, don't expose raw IDs)
- More complex implementation

**Use when:** Infinite scroll, feeds, APIs with large datasets, mobile clients.

---

### Comparison

| | Offset | Cursor/Keyset |
|---|---|---|
| Performance at depth | Degrades O(n) | Constant O(log n) |
| Consistency under inserts | May skip/duplicate | Stable |
| Jump to page N | Yes | No |
| Implementation | Simple | Moderate |
| Best for | Admin tables, small sets | Feeds, large sets, infinite scroll |

---

### In ASP.NET Core

```csharp
// Offset pagination
public record PagedRequest(int Page = 1, int PageSize = 20);

public record PagedResponse<T>(
    IEnumerable<T> Data,
    int Page,
    int PageSize,
    int TotalCount)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
}

// Cursor pagination
public record CursorRequest(int Limit = 20, string? Cursor = null);

// Decode cursor → extract last seen Id
private static Guid? DecodeCursor(string? cursor)
    => cursor is null ? null
     : JsonSerializer.Deserialize<Guid>(
         Encoding.UTF8.GetString(Convert.FromBase64String(cursor)));

private static string EncodeCursor(Guid id)
    => Convert.ToBase64String(Encoding.UTF8.GetBytes(id.ToString()));
```

---

## HTTP/2 vs HTTP/1.1

### HTTP/1.1 Problems

- **Head-of-line blocking:** Requests on the same TCP connection must wait for the one ahead
- **Limited connections:** Browsers open 6 TCP connections per origin to work around this
- **Uncompressed headers:** Repeated headers (Cookie, User-Agent) sent on every request

### HTTP/2 Solutions

```
HTTP/1.1:
  Connection 1: GET /style.css → wait → GET /app.js → wait → GET /data.json
  Connection 2: GET /image.png → wait
  (6 connections × sequential = still slow)

HTTP/2:
  Single TCP connection, multiplexed streams:
  Stream 1: GET /style.css  ──────────────► response
  Stream 3: GET /app.js     ──────────────► response
  Stream 5: GET /data.json  ──────────────► response
  Stream 7: GET /image.png  ──────────────► response
  (all in parallel, one connection)
```

**Key HTTP/2 features:**
- **Multiplexing** — multiple requests over one TCP connection, no blocking
- **Header compression (HPACK)** — repeated headers compressed, huge savings on cookie-heavy requests
- **Server Push** — server can proactively send resources before client requests them (mostly unused in practice)
- **Binary framing** — not human-readable, but more efficient than HTTP/1.1 text

**In ASP.NET Core:** HTTP/2 is enabled by default in Kestrel. No code changes needed — the protocol is negotiated via ALPN in the TLS handshake.

---

## WebSockets vs SSE vs Long Polling

When you need server-to-client push or real-time bidirectional communication.

### Long Polling

Client sends request → server holds it open until data is available → responds → client immediately sends another.

```javascript
async function longPoll() {
  while (true) {
    const response = await fetch('/api/notifications/poll');
    const data = await response.json();
    handleNotification(data);
    // immediately start next poll
  }
}
```

**Pros:** Works everywhere, no special infrastructure.
**Cons:** High overhead (HTTP overhead per "event"), latency equals poll interval.
**Use when:** Simple notifications, fallback for environments that block WebSockets.

---

### Server-Sent Events (SSE)

One-way: server pushes events to client over a persistent HTTP connection.

```typescript
// Client
const events = new EventSource('/api/notifications/stream');
events.onmessage = (event) => handleNotification(JSON.parse(event.data));
events.onerror = () => events.close();
```

```csharp
// ASP.NET Core — SSE endpoint
[HttpGet("stream")]
public async Task Stream(CancellationToken ct)
{
    Response.Headers.Add("Content-Type", "text/event-stream");
    Response.Headers.Add("Cache-Control", "no-cache");

    while (!ct.IsCancellationRequested)
    {
        var notification = await _queue.DequeueAsync(ct);
        await Response.WriteAsync($"data: {JsonSerializer.Serialize(notification)}\n\n", ct);
        await Response.Body.FlushAsync(ct);
    }
}
```

**Pros:** Simple, uses standard HTTP, automatic reconnection, works through proxies.
**Cons:** One-way only (server → client). Limited to text.
**Use when:** Live dashboards, activity feeds, order status updates, notification streams.

---

### WebSockets

Full-duplex: both sides can send at any time after the initial HTTP upgrade.

```typescript
// Client
const ws = new WebSocket('wss://app.com/ws/chat');
ws.onopen = () => ws.send(JSON.stringify({ type: 'join', room: 'general' }));
ws.onmessage = (event) => handleMessage(JSON.parse(event.data));
ws.onclose = () => reconnect();
```

```csharp
// ASP.NET Core — WebSocket endpoint
app.UseWebSockets();

app.Map("/ws/chat", async context =>
{
    if (context.WebSockets.IsWebSocketRequest)
    {
        var ws = await context.WebSockets.AcceptWebSocketAsync();
        await HandleChatConnection(ws);
    }
});

// Or use SignalR — abstracts WebSockets + fallbacks
// services.AddSignalR();
```

**Pros:** Bidirectional, low latency, binary or text.
**Cons:** Stateful connections are harder to scale horizontally (need sticky sessions or pub/sub broker). More complex error handling.
**Use when:** Chat, multiplayer games, collaborative editing, trading platforms.

---

### When to Use What

| Use case | Technology |
|---|---|
| Simple notifications, low frequency | Long polling or SSE |
| Live dashboard (server pushes data) | SSE |
| Chat, real-time collaboration | WebSockets / SignalR |
| Order status updates | SSE |
| Multiplayer game | WebSockets |
| Scale beyond one server | WebSockets + Redis pub/sub (SignalR backplane) |

---

## GraphQL — Conceptual

A query language for APIs where the **client specifies exactly what data it needs**.

```graphql
# Client query — asks for only what it needs
query {
  product(id: "123") {
    name
    price
    category {
      name
    }
    # notice: no 'description', 'stock', 'createdAt' — not fetched
  }
}
```

**Compared to REST:**
```
REST:  GET /products/123 → returns the full Product object (over-fetching)
REST:  need product + category → GET /products/123 then GET /categories/5 (under-fetching)
GraphQL: one request, get exactly what you asked for
```

### N+1 in GraphQL

A common GraphQL problem: resolving a list of products where each one fetches its category triggers N+1 DB queries.

**Solution: DataLoader** — batches and deduplicates queries.

```
Without DataLoader: fetch product[0] category, fetch product[1] category, ... (N queries)
With DataLoader: collect all category IDs, then fetch WHERE Id IN (1, 2, 3...) (1 query)
```

### When REST vs GraphQL

| | REST | GraphQL |
|---|---|---|
| Multiple clients (mobile + web) with different data needs | ❌ Over/under fetching | ✅ Client specifies fields |
| Simple CRUD API | ✅ Simple to implement | ❌ Over-engineered |
| Public API (third-party consumers) | ✅ More familiar | ❌ Steeper learning curve |
| Caching | ✅ URL-based, CDN-friendly | ❌ POST requests, harder to cache |
| Real-time | ❌ Needs SSE/WS | ✅ Subscriptions built-in |

**Use GraphQL when:** Multiple client types (mobile + web) with different data needs, or heavy real-time subscription requirements.

---

## gRPC — Conceptual

A high-performance RPC framework using **Protocol Buffers** (binary serialization) over HTTP/2.

```protobuf
// products.proto — the contract (like an interface)
service ProductService {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc ListProducts (ListProductsRequest) returns (stream ProductResponse); // streaming!
}

message GetProductRequest { string id = 1; }
message ProductResponse { string id = 1; string name = 2; double price = 3; }
```

**Compared to REST:**
- Binary (Protobuf) vs text (JSON) → 3-10x smaller, faster to parse
- Strongly typed contract via `.proto` files → generate client code in any language
- Streaming support built-in (client, server, or bidirectional)
- HTTP/2 required

### When REST vs gRPC

| | REST/JSON | gRPC |
|---|---|---|
| Browser clients | ✅ Native | ❌ Requires grpc-web proxy |
| Internal service-to-service | ✅ Fine | ✅ Better performance |
| Mobile clients | ✅ Fine | ✅ Smaller payloads |
| Streaming | ❌ Needs workarounds | ✅ First-class |
| Human readability | ✅ JSON is readable | ❌ Binary |
| Ecosystem/tooling | ✅ Universal | ✅ Good but narrower |

**Use gRPC when:** Service-to-service communication in microservices, mobile clients where payload size matters, real-time bidirectional streaming.

**In .NET:** `Grpc.AspNetCore` package. Controllers become services implementing generated interfaces.

---

## Idempotency Keys for POST

POST is not idempotent — retrying creates duplicates. Idempotency keys solve this.

```
Client sends:
POST /orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Body: { "productId": "...", "quantity": 2 }

Server:
1. Check if Idempotency-Key has been seen before
2. If yes → return cached response (don't process again)
3. If no → process, save key + response, return response
```

```csharp
// Middleware or endpoint filter to handle idempotency
public class IdempotencyFilter : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var key = context.HttpContext.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrEmpty(key)) return await next(context);

        if (_cache.TryGetValue(key, out var cached)) return cached; // replay

        var result = await next(context);
        _cache.Set(key, result, TimeSpan.FromHours(24)); // store result
        return result;
    }
}
```

---

## Common Interview Questions

1. What is the difference between offset and cursor pagination? When would you choose each?
2. What problems does HTTP/2 solve over HTTP/1.1?
3. What is the difference between SSE and WebSockets?
4. When would you choose GraphQL over REST?
5. What is gRPC and when is it better than REST?
6. What is an idempotency key and why does POST need one?
7. What is the N+1 problem in GraphQL? How does DataLoader solve it?

---

## Common Mistakes

- Using offset pagination for infinite scroll at scale (O(n) becomes very slow)
- Exposing raw IDs as cursors (should be encoded/opaque)
- Not handling WebSocket disconnections with reconnect logic
- Choosing GraphQL for a simple CRUD API (over-engineering)
- Not setting `Access-Control-Max-Age` to cache CORS preflight (per-request preflight = 2× latency)

---

## How It Connects

- Cursor pagination pairs with RTK Query's `merge` strategy for infinite scroll
- WebSockets/SignalR is how your Chat system (from system design) maintains connections
- gRPC is the transport layer in microservices (from `microservices.md`)
- Idempotency keys protect payment operations where duplicate charges must never happen
- HTTP/2 multiplexing is why one CDN connection can serve your JS, CSS, and fonts simultaneously

---

## My Confidence Level
- `[ ]` Offset pagination — implementation and trade-offs
- `[ ]` Cursor/keyset pagination — implementation and trade-offs
- `[ ]` HTTP/2 — what it solves, multiplexing, header compression
- `[ ]` WebSockets — bidirectional, when to use, scaling challenge
- `[ ]` SSE — server-push, when to use vs WebSockets
- `[ ]` Long polling — mechanism and trade-offs
- `[ ]` GraphQL — query model, N+1 problem, when over REST
- `[ ]` gRPC — Protobuf, streaming, when over REST
- `[ ]` Idempotency keys — pattern and implementation

## My Notes
<!-- Personal notes -->
