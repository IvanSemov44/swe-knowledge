# HTTP & REST

## What it is
HTTP (HyperText Transfer Protocol) is the foundation of data exchange on the web. REST (Representational State Transfer) is an architectural style for designing networked APIs using HTTP.

## Why it matters
Every web API you build or consume uses HTTP. REST conventions are the default language of web services — knowing them well prevents subtle bugs and makes your APIs predictable.

---

## HTTP Methods

| Method | Purpose | Idempotent? | Safe? | Has Body? |
|---|---|---|---|---|
| GET | Read a resource | Yes | Yes | No |
| POST | Create a resource / trigger action | No | No | Yes |
| PUT | Replace a resource entirely | Yes | No | Yes |
| PATCH | Partially update a resource | No* | No | Yes |
| DELETE | Delete a resource | Yes | No | No |
| HEAD | Like GET, but headers only | Yes | Yes | No |
| OPTIONS | Describe communication options | Yes | Yes | No |

**Idempotent:** Calling N times has the same effect as calling once.
**Safe:** Does not modify server state.

*PATCH idempotency depends on implementation.

---

## HTTP Status Codes

### 2xx — Success
| Code | Meaning | Use when |
|---|---|---|
| 200 OK | Request succeeded | GET, PUT, PATCH success |
| 201 Created | Resource created | POST that creates a resource |
| 204 No Content | Success, no body | DELETE, PUT with no return |

### 3xx — Redirection
| Code | Meaning |
|---|---|
| 301 Moved Permanently | Resource moved, update your bookmarks |
| 302 Found | Temporary redirect |
| 304 Not Modified | Client cache is still valid |

### 4xx — Client Errors
| Code | Meaning | Use when |
|---|---|---|
| 400 Bad Request | Invalid input | Validation failure |
| 401 Unauthorized | Not authenticated | Missing/invalid token |
| 403 Forbidden | Authenticated but not authorized | Role/permission missing |
| 404 Not Found | Resource doesn't exist | Entity not found |
| 409 Conflict | State conflict | Duplicate, optimistic concurrency |
| 422 Unprocessable Entity | Semantic validation failure | Business rule violation |
| 429 Too Many Requests | Rate limit exceeded | Throttling |

### 5xx — Server Errors
| Code | Meaning |
|---|---|
| 500 Internal Server Error | Unexpected crash |
| 502 Bad Gateway | Upstream service failed |
| 503 Service Unavailable | Server overloaded / maintenance |
| 504 Gateway Timeout | Upstream didn't respond in time |

**Common confusion:**
- 401 vs 403: 401 = "who are you?" (not logged in); 403 = "I know who you are, you can't do this"
- 400 vs 422: 400 = malformed request (bad JSON); 422 = valid format but invalid semantics (price < 0)

---

## REST Principles (Richardson Maturity Model)

### Level 0 — HTTP as transport
Single endpoint, action in body. (XML-RPC, SOAP style)

### Level 1 — Resources
Separate URLs per resource: `/orders`, `/products/5`

### Level 2 — HTTP Verbs
Use GET/POST/PUT/DELETE correctly. Return correct status codes.

### Level 3 — Hypermedia (HATEOAS)
Responses include links to related actions. Rarely used in practice.

Most modern REST APIs target Level 2.

---

## REST URL Design

**Rules:**
- Use nouns, not verbs: `/orders` not `/getOrders`
- Plural for collections: `/products`
- Nested for relationships: `/orders/{id}/items`
- Lowercase, hyphenated: `/order-items` not `/orderItems`
- Avoid deep nesting: max 2 levels deep

```
GET    /products              → list all products
POST   /products              → create product
GET    /products/{id}         → get one product
PUT    /products/{id}         → replace product
PATCH  /products/{id}         → partial update
DELETE /products/{id}         → delete product

GET    /orders/{id}/items     → items belonging to an order
```

---

## Idempotency

An operation is idempotent if repeating it produces the same result.

**Why it matters:** Network requests fail. Clients retry. If your endpoint isn't idempotent, retries cause duplicate data.

| Method | Idempotent? | Why |
|---|---|---|
| GET | Yes | Read-only |
| DELETE | Yes | Deleting twice = same end state (resource gone) |
| PUT | Yes | Replacing with same data = same result |
| POST | No | Two POST /orders = two orders |
| PATCH | Depends | `SET price = 10` is idempotent; `INCREMENT price` is not |

**Idempotency key pattern:** For POST operations that must be safe to retry, clients send a unique `Idempotency-Key` header. Server stores it and returns cached response on duplicate.

---

## API Versioning

**Why:** You need to evolve your API without breaking existing clients.

### URL versioning (most common)
```
/api/v1/products
/api/v2/products
```
Pros: Simple, visible. Cons: Pollutes URLs.

### Header versioning
```
Accept: application/vnd.myapi.v2+json
```
Pros: Clean URLs. Cons: Less discoverable.

### Query param versioning
```
/products?version=2
```
Pros: Easy to test. Cons: Messy.

**In ASP.NET Core:** Use `Asp.Versioning.Mvc` package for clean versioning support.

---

## HTTP Headers (Key Ones)

| Header | Direction | Purpose |
|---|---|---|
| `Content-Type` | Request/Response | Body format: `application/json` |
| `Accept` | Request | Formats client accepts |
| `Authorization` | Request | `Bearer <token>` |
| `Cache-Control` | Both | Caching directives |
| `ETag` | Response | Version identifier for caching |
| `If-None-Match` | Request | Conditional GET (sends ETag back) |
| `X-Request-ID` | Request | Correlation ID for distributed tracing |
| `Location` | Response | URL of created resource (with 201) |

---

## Caching with HTTP

**ETags:** Server sends `ETag: "abc123"` with response. Client sends `If-None-Match: "abc123"` on next request. If unchanged, server returns `304 Not Modified` — no body transmitted.

**Cache-Control:**
```
Cache-Control: no-cache          → must revalidate each time
Cache-Control: max-age=3600      → cache for 1 hour
Cache-Control: no-store          → never cache (sensitive data)
Cache-Control: public            → can be cached by CDN
Cache-Control: private           → user-specific, only client cache
```

---

## Content Negotiation

Client tells server what format it wants:
```
Accept: application/json
Accept: application/xml
```

Server responds with matching format and includes:
```
Content-Type: application/json; charset=utf-8
```

---

## In ASP.NET Core

```csharp
[ApiController]
[Route("api/v1/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public async Task<ApiResponse<List<ProductDto>>> GetAll(
        CancellationToken cancellationToken = default) { }

    [HttpGet("{id:guid}")]
    public async Task<ApiResponse<ProductDto>> GetById(
        Guid id, CancellationToken cancellationToken = default) { }

    [HttpPost]
    [ValidationFilter]
    public async Task<ApiResponse<ProductDto>> Create(
        [FromBody] CreateProductRequest request,
        CancellationToken cancellationToken = default) { }
}
```

**Return 201 with Location header on creation:**
```csharp
return CreatedAtAction(nameof(GetById), new { id = product.Id }, response);
```

---

## Common Interview Questions

1. What is the difference between 401 and 403?
2. What does idempotent mean? Which HTTP methods are idempotent?
3. When would you use PUT vs PATCH?
4. How does ETag-based caching work?
5. What are the REST constraints?
6. How do you version a REST API?

---

## Common Mistakes

- Using GET for operations that modify state (side effects)
- Returning 200 for errors (error in body but 200 status)
- Returning 404 when you should return 403 (leaking existence information)
- Designing RPC-style URLs (`/createOrder`) instead of resource URLs (`POST /orders`)
- Not returning `Location` header with 201 responses

---

## How It Connects

- ASP.NET Core controllers map to HTTP endpoints
- RTK Query tags map to GET endpoints; mutations map to POST/PUT/PATCH/DELETE
- Authentication middleware checks `Authorization` header
- CORS policy controls cross-origin HTTP request permissions
- Rate limiting returns 429

---

## My Confidence Level
- `[c]` HTTP methods and status codes
- `[b]` REST URL design
- `[b]` Idempotency
- `[~]` API versioning strategies
- `[~]` HTTP caching headers

## My Notes
<!-- Personal notes -->
