# API Gateway

## What it is
A single entry point that sits in front of all your services. Every client request goes through the gateway before reaching any service. Handles cross-cutting concerns so your services don't have to.

---

## What the Gateway Does

```
Client
  │
  ▼
API Gateway ← does all of this:
  │  - SSL termination (HTTPS → HTTP internally)
  │  - Authentication (validate JWT)
  │  - Rate limiting
  │  - Request routing (/api/products → Products service)
  │  - Load balancing across service instances
  │  - Request/response logging
  │  - Correlation ID injection
  │
  ├── Products API
  ├── Orders API
  └── Users API
```

Your services receive clean, already-authenticated, already-rate-limited requests.

---

## YARP (Yet Another Reverse Proxy)

Microsoft's reverse proxy library for .NET. You create a new ASP.NET Core project as your gateway.

### Setup

```bash
dotnet new web -n ApiGateway
cd ApiGateway
dotnet add package Yarp.ReverseProxy
```

```csharp
// Program.cs
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

builder.Services.AddRateLimiter(opts =>
{
    opts.AddTokenBucketLimiter("global", o =>
    {
        o.TokenLimit = 100;
        o.ReplenishmentPeriod = TimeSpan.FromMinutes(1);
        o.TokensPerPeriod = 100;
        o.AutoReplenishment = true;
    });
    opts.RejectionStatusCode = 429;
});

app.UseRateLimiter();
app.MapReverseProxy();
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "products-route": {
        "ClusterId": "products",
        "Match": { "Path": "/api/products/{**catch-all}" },
        "RateLimiterPolicy": "global"
      },
      "orders-route": {
        "ClusterId": "orders",
        "Match": { "Path": "/api/orders/{**catch-all}" },
        "RateLimiterPolicy": "global"
      }
    },
    "Clusters": {
      "products": {
        "Destinations": {
          "products/instance1": { "Address": "http://products-api:8080" },
          "products/instance2": { "Address": "http://products-api-2:8080" }
        }
      },
      "orders": {
        "Destinations": {
          "orders/instance1": { "Address": "http://orders-api:8081" }
        }
      }
    }
  }
}
```

YARP automatically load balances across all destinations in a cluster (round-robin by default).

---

## JWT Validation at the Gateway

Validate the token once at the gateway — services trust requests that passed through.

```csharp
// Gateway validates JWT
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://your-identity-server";
        options.Audience = "api";
    });

app.UseAuthentication();
app.UseAuthorization();

// Transform — forward user identity to downstream services as a header
app.MapReverseProxy(pipeline =>
{
    pipeline.Use(async (context, next) =>
    {
        if (context.User.Identity?.IsAuthenticated == true)
        {
            var userId = context.User.FindFirst("sub")?.Value;
            context.Request.Headers["X-User-Id"] = userId;
        }
        await next();
    });
});
```

Services read `X-User-Id` header — no JWT validation code duplicated across services.

---

## Ocelot (Alternative)

Older .NET gateway library. More configuration, less flexible than YARP. Still widely used in existing projects.

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/products/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [{ "Host": "products-api", "Port": 8080 }],
      "UpstreamPathTemplate": "/api/products/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ]
}
```

**YARP vs Ocelot:**
| | YARP | Ocelot |
|---|---|---|
| Maintained by | Microsoft | Community |
| Performance | Higher | Lower |
| Flexibility | Code-first, fully extensible | Config-first, limited |
| Use when | New projects | Existing Ocelot projects |

---

## External Gateways (No Code)

For larger teams or multi-language systems:

| Gateway | Best for |
|---|---|
| **Kong** | Self-hosted, plugin ecosystem, any language |
| **AWS API Gateway** | AWS-hosted apps, serverless |
| **Azure API Management** | Azure-hosted apps, enterprise features |
| **Nginx** | Simple routing, battle-tested, free |

These are configured via dashboards or config files — no .NET code needed.

---

## Solution Structure

```
YourSolution/
  src/
    ApiGateway/           ← YARP project (new)
    Products.Api/
    Orders.Api/
    Users.Api/
    Shared.Contracts/     ← shared DTOs, auth constants
  docker-compose.yml      ← wires everything together locally
```

---

## Common Interview Questions

1. What is an API Gateway and what problems does it solve?
2. Why is it better to do auth at the gateway instead of each service?
3. What is the difference between a Load Balancer and an API Gateway?
4. What is YARP and how does it work?
5. How does the gateway pass user identity to downstream services?

---

## Common Mistakes

- Putting business logic in the gateway (it should be infrastructure only)
- Not running multiple gateway instances (single point of failure)
- Services calling each other through the gateway (use direct service-to-service calls internally)
- Forgetting to propagate Correlation IDs through the gateway

---

## How It Connects

- Load Balancer sits in front of the gateway (not instead of it)
- Gateway reads from Redis for rate limiting shared state
- Observability: gateway is the best place to log all requests and inject Correlation IDs
- Services behind the gateway can be completely internal (no public exposure)

---

## My Confidence Level
- `[ ]` What an API Gateway does vs a Load Balancer
- `[ ]` YARP setup — routes, clusters, destinations
- `[ ]` JWT validation at gateway, forwarding user identity
- `[ ]` Rate limiting at gateway level
- `[ ]` YARP vs Ocelot — when to choose which
- `[ ]` External gateways (Kong, AWS, Azure)

## My Notes
<!-- Personal notes -->
