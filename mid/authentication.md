# Authentication & Authorization

## What it is
**Authentication:** Verifying identity — "who are you?"
**Authorization:** Verifying permissions — "what are you allowed to do?"

## Why it matters
Getting auth wrong is a critical security vulnerability. Understanding JWT, OAuth2, and the cookie vs token tradeoff is essential for any backend developer.

---

## JWT (JSON Web Token)

### Structure
A JWT is three Base64URL-encoded parts separated by dots:

```
header.payload.signature
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Header:**
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (claims):**
```json
{
  "sub": "user-id-123",
  "email": "user@example.com",
  "role": "Customer",
  "iat": 1711929600,
  "exp": 1711933200
}
```

**Signature:** `HMACSHA256(base64(header) + "." + base64(payload), secret)`

### Verification
The server verifies the signature using the secret key. If the signature is valid and `exp` hasn't passed, the token is trusted. **No database lookup needed** — this is why JWTs are stateless.

### Claims
| Claim | Meaning |
|---|---|
| `sub` | Subject (user ID) |
| `iss` | Issuer |
| `aud` | Audience |
| `exp` | Expiration (Unix timestamp) |
| `iat` | Issued at |
| `nbf` | Not before |
| Custom | `role`, `email`, `permissions` |

---

## Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| Lifetime | Short (15 min – 1 hour) | Long (7–30 days) |
| Stored in | Memory (SPA) or HttpOnly cookie | HttpOnly cookie (secure storage) |
| Used for | API calls (Authorization header) | Getting new access tokens |
| Revocable? | Difficult (stateless) | Yes (stored in DB) |
| If stolen | Short window of exposure | Must be revoked in DB |

**Flow:**
1. User logs in → server issues access token + refresh token
2. Client uses access token for API calls
3. Access token expires → client sends refresh token to `/auth/refresh`
4. Server validates refresh token (checks DB), issues new access token
5. On logout → delete refresh token from DB

---

## OAuth2 Flows

OAuth2 is an authorization framework — it lets users grant third-party apps limited access to their resources without giving them their password.

### Authorization Code Flow (most secure, for web apps + SPAs)
1. User clicks "Login with Google"
2. App redirects to Google's auth endpoint with `client_id`, `redirect_uri`, `scope`, `state`
3. User logs in on Google, approves permissions
4. Google redirects back with `code`
5. App exchanges `code` for access + refresh tokens (server-to-server, secret not exposed)

### PKCE — Proof Key for Code Exchange (required for SPAs and mobile)

SPAs can't keep a `client_secret` safe (it would be in the JavaScript bundle). PKCE replaces the client secret with a one-time cryptographic proof.

```
1. Client generates a random code_verifier (e.g., 128 random bytes, base64url encoded)
2. Client computes code_challenge = BASE64URL(SHA256(code_verifier))
3. Client sends code_challenge in the authorization request
4. Server stores code_challenge
5. Client receives authorization code
6. Client exchanges code + code_verifier for tokens
7. Server hashes code_verifier → must match stored code_challenge → proof of identity
```

**Why it prevents attacks:**
- If an attacker intercepts the authorization code, they can't exchange it — they don't have the `code_verifier`
- The `code_verifier` is never sent until the final exchange step
- Even if they intercept the exchange request, the `code_verifier` is single-use

```typescript
// React SPA — PKCE handled by OIDC libraries (oidc-client-ts, Auth0 SDK, etc.)
// You don't implement this manually — but you need to understand it

import { UserManager } from 'oidc-client-ts';

const userManager = new UserManager({
  authority: 'https://auth.myapp.com',
  client_id: 'spa-client',
  redirect_uri: 'https://app.myapp.com/callback',
  response_type: 'code',        // Authorization Code Flow
  scope: 'openid profile email',
  // PKCE is enabled by default in oidc-client-ts
});
```

### Client Credentials Flow (machine-to-machine)
No user involved. Service authenticates with client_id + client_secret to get a token for API calls.

### Implicit Flow (deprecated)
Don't use. Tokens in URL fragment, unsafe. PKCE replaces this for SPAs.

---

## Cookies vs Tokens

| | Cookies (HttpOnly) | Bearer Tokens (Header) |
|---|---|---|
| Storage | Browser handles automatically | localStorage, sessionStorage, or memory |
| XSS protection | HttpOnly prevents JS access | Vulnerable if in localStorage |
| CSRF protection | Need CSRF token or SameSite | Not vulnerable (browser doesn't auto-send headers) |
| Mobile support | Complex | Easy (just set the header) |
| Stateless? | Can be (JWT in cookie) | Yes |
| Best for | Web apps (same domain) | SPAs, mobile, microservices |

**Best practice for SPAs:**
- Store access token in **memory** (not localStorage — XSS risk)
- Store refresh token in **HttpOnly + Secure + SameSite=Strict cookie** (not accessible to JS)

### Cookie Security Attributes

```csharp
// HttpOnly — prevents JavaScript from reading the cookie
// Secure — only sent over HTTPS
// SameSite — controls when the browser sends the cookie cross-site
options.Cookie.HttpOnly = true;
options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
options.Cookie.SameSite = SameSiteMode.Strict;
```

| Attribute | What it prevents |
|---|---|
| `HttpOnly` | XSS stealing the cookie via `document.cookie` |
| `Secure` | Cookie sent over plain HTTP (man-in-the-middle) |
| `SameSite=Strict` | CSRF — cookie not sent on any cross-site request |
| `SameSite=Lax` | CSRF for form POSTs — cookie not sent on cross-site POST (but sent on GET navigations) |
| `SameSite=None` | Required for cross-site cookies (third-party, embedded) — must also set `Secure` |

**Session-based auth vs Token-based auth:**

| | Session | JWT Token |
|---|---|---|
| State stored | Server (DB or Redis) | Client (token contains claims) |
| Revocation | Immediate (delete session) | Hard — must wait for expiry or use token blocklist |
| Scalability | Needs shared session store (Redis) for horizontal scaling | Stateless — any server can validate |
| Size | Small (session ID only) | Larger (claims in token) |
| Best for | Traditional server-rendered apps | APIs, SPAs, microservices |

---

## In ASP.NET Core

```csharp
// Program.cs — JWT setup
services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = config["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = config["Jwt:Audience"],
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(config["Jwt:Secret"]!))
        };
    });

services.AddAuthorization();
```

```csharp
// Generating a JWT
public string GenerateToken(User user)
{
    var claims = new[]
    {
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
        new Claim(ClaimTypes.Email, user.Email),
        new Claim(ClaimTypes.Role, user.Role.ToString())
    };

    var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_config["Jwt:Secret"]!));
    var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: _config["Jwt:Issuer"],
        audience: _config["Jwt:Audience"],
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(15),
        signingCredentials: creds);

    return new JwtSecurityTokenHandler().WriteToken(token);
}
```

```csharp
// Controller
[Authorize]
public class OrdersController : ControllerBase
{
    [Authorize(Roles = "Admin")]
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(Guid id) { }
}

// Get current user ID from claims
private Guid CurrentUserId => Guid.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);
```

---

## Common Security Mistakes

- Storing JWT in localStorage (XSS vulnerable — any injected script can steal it)
- Using a weak or hardcoded JWT secret in production
- Not validating `exp` and `iss` claims
- Putting sensitive data in JWT payload (it's Base64 encoded, not encrypted)
- Long-lived access tokens without refresh token rotation
- Not invalidating refresh tokens on logout
- Storing passwords as plain text or MD5/SHA1 (use bcrypt/Argon2)

---

## OWASP Top 10 (Authentication-related)

- **A01: Broken Access Control** — unauthorized users accessing resources
- **A02: Cryptographic Failures** — weak JWT secrets, HTTP instead of HTTPS, sensitive data in logs
- **A07: Identification and Authentication Failures** — weak passwords, no MFA, session fixation

---

## Common Interview Questions

1. What is the difference between authentication and authorization?
2. What are the three parts of a JWT and what does each contain?
3. Why are access tokens short-lived?
4. What is the OAuth2 Authorization Code flow?
5. Why is storing JWT in localStorage a security risk?
6. What is the difference between cookies and bearer tokens?
7. How do you revoke a JWT?

---

## How It Connects

- ASP.NET Core middleware pipeline: `UseAuthentication()` runs before `UseAuthorization()`
- Claims are available via `HttpContext.User.Claims` in controllers
- Role-based authorization uses claims in the JWT
- Refresh token stored as HttpOnly cookie = secure without JS access
- MediatR handlers can access current user via `ICurrentUserService` injected from `IHttpContextAccessor`

---

## My Confidence Level
- `[b]` JWT structure and verification
- `[b]` Access token vs refresh token
- `[b]` OAuth2 flows
- `[b]` Cookies vs tokens
- `[~]` HTTPS, TLS basics
- `[~]` OWASP top 10

## My Notes
<!-- Personal notes -->
