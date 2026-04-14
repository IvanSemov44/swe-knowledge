# Microservices

<!-- last-reviewed: 2026-04-09 | next-review: 2026-05-09 | confidence: ~ -->

---

## Monolith vs Microservices

### Monolith
Single deployable unit. All modules run in the same process.

```
┌─────────────────────────────────┐
│  E-commerce Monolith            │
│  ┌─────────┐  ┌──────────────┐  │
│  │ Orders  │→ │  Inventory   │  │  ← in-process method call
│  └─────────┘  └──────────────┘  │
│  ┌─────────┐  ┌──────────────┐  │
│  │ Users   │  │  Payments    │  │
│  └─────────┘  └──────────────┘  │
└─────────────────────────────────┘
         ↓
    One Database
```

**Advantages:**
- Simple to develop, test, deploy
- In-process calls — no network latency
- One transaction spans the whole operation
- Easy to debug — one log stream, one codebase

**Disadvantages:**
- Gets complex over time — hard to maintain at scale
- One bug can take down the whole app
- Must deploy everything to change one thing
- Can't scale individual parts independently

---

### Microservices
Each domain is a separate deployable service with its own database.

```
┌──────────┐   HTTP/Queue   ┌───────────────┐
│  Orders  │ ────────────→  │   Inventory   │  ← network call
│  Service │                │   Service     │
└──────────┘                └───────────────┘
     ↓                              ↓
  Orders DB                  Inventory DB
```

**Advantages:**
- Independent scaling — scale only what's under load
- Independent deployment — change one service without touching others
- Smaller codebases — easier to understand and maintain
- Different services can use different tech stacks

**Disadvantages (the real cost):**
- Network calls can fail, timeout, be slow
- No single transaction — distributed consistency is hard
- Operational overhead: API gateway, service discovery, distributed tracing
- Debugging across services is painful
- Needs mature DevOps — separate CI/CD, deployments per service

---

## When NOT to Use Microservices

- **Small team** — microservices are designed for multiple independent teams. One dev managing 8 services is just overhead.
- **Early stage / unclear domain** — split too early and you draw wrong boundaries. Stuck with a distributed monolith (worst of both worlds).
- **Low traffic** — if you don't need independent scaling, you're paying complexity cost for nothing.
- **CRUD-heavy app** — most operations touch multiple domains anyway.

**The senior rule:** Start with a well-structured monolith. Extract services when you actually feel the pain — not before.

---

## Key Patterns

### API Gateway
Single entry point for all clients. Routes requests to the right service. Handles auth, rate limiting, SSL termination.

```
Client → API Gateway → Orders Service
                    → Users Service
                    → Inventory Service
```

### Service Discovery
Services register themselves. Others look them up by name, not IP (IPs change in containerized environments).

```
Orders Service → "Where is Inventory?" → Service Registry → returns address
```

Tools: Consul, Kubernetes DNS, AWS Service Discovery.

### Saga Pattern
Manages distributed transactions across services. Each step publishes an event; failures trigger compensating actions.

```
1. OrderService: Create order (pending)
2. PaymentService: Charge card → success → publish PaymentSucceeded
3. InventoryService: Reserve stock → success → publish StockReserved
4. OrderService: Mark order confirmed

If step 3 fails:
  → InventoryService publishes StockFailed
  → PaymentService refunds card (compensating transaction)
  → OrderService marks order failed
```

Two implementations:
- **Choreography** — services react to each other's events (no central coordinator)
- **Orchestration** — central saga orchestrator tells each service what to do

### Outbox Pattern
Guarantees a message is published even if the service crashes mid-operation.

```csharp
// In one DB transaction:
// 1. Save the order
// 2. Save an OutboxMessage row (not published yet)

// Background job:
// Read unpublished OutboxMessages → publish to queue → mark published
```

Prevents: "order saved but payment event never sent."

---

## Distributed Tracing

One request spans multiple services — how do you trace a bug?

Assign a **Correlation ID** at the API gateway. Pass it in every downstream request header. Log it everywhere.

```
Request → API Gateway (TraceId: abc123)
        → Orders Service    (logs: TraceId abc123)
        → Inventory Service (logs: TraceId abc123)
        → Payment Service   (logs: TraceId abc123)
```

Search logs by `TraceId: abc123` → see the full journey.

Tools: OpenTelemetry, Jaeger, Zipkin, Datadog.

---

## For Your E-commerce App

Your current monolith with Clean Architecture is the right choice now. The layer boundaries (Core / Application / Infrastructure) give you the separation of concerns you'd get from microservices — without the operational cost.

If you later split:
- Orders → its own service (highest load, most critical)
- Notifications → good first extraction (async, low risk)
- Payments → separate for compliance reasons

---

## Connected To

- `senior/message-queues.md` — services communicate via queues; saga pattern relies on this
- `senior/ci-cd.md` — each service needs its own pipeline
- `mid/docker.md` — each service runs in its own container
- `mid/cqrs.md` — CQRS scales naturally into microservices (queries can be separate read services)
