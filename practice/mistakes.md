# mistakes.md — What I Got Wrong

This file is maintained by the AI during every session.
Each entry = a gap caught in real time (wrong answer, fumbled explanation, bug I caused).

**Why this exists:** Polished notes on things you know have diminishing returns.
This file tracks the exact things that tripped you up — which is what interviews will ask.

---

## Template

```
## YYYY-MM-DD
**Topic:** [topic name]
**What happened:** [what I said or did wrong]
**Root cause:** [why — misconception, forgotten detail, bad habit]
**Fix / key insight:** [correct understanding in 1-3 sentences]
**Revisit:** [link to relevant knowledge file, e.g. mid/ddd.md]
**Revisit by:** YYYY-MM-DD (set 3–5 days out)
```

---

<!-- AI: add entries below after each session -->

## 2026-04-29
**Topic:** Event-Driven — DLQ is a RabbitMQ queue, not a DB table
**What happened:** Described DLQ as a "table where unprocessed events stay with errors."
**Root cause:** Knows the purpose of DLQ but visualized it as a DB concept. DLQ is a broker-side queue, not something in your database.
**Fix / key insight:** DLQ lives in RabbitMQ (`order-placed_error`). After all retries fail, RabbitMQ routes the message there automatically. You inspect and replay via the RabbitMQ Management UI at `:15672`. No DB table involved — unless you build a custom monitoring layer on top.
**Revisit:** mid/event-driven-flow.md
**Revisit by:** 2026-05-04

## 2026-04-29
**Topic:** CQRS/DDD — Projection definition
**What happened:** Described projection as "the table where Inventory sends back data" — conflated a projection with an integration event response.
**Root cause:** Didn't have a clean mental model of what a projection IS. A projection is a read model built from events — not the event itself or the response channel.
**Fix / key insight:** Projection = a denormalized, read-optimized table built by consuming events. Queries ONLY read from projections, never from the domain model. When a `StockReservedEvent` arrives, the consumer writes into a `StockStatusProjection` table. The query later reads from that table fast, with no joins.
**Revisit:** mid/event-driven-flow.md
**Revisit by:** 2026-05-04

## 2026-04-28
**Topic:** DDD — Integration Events vs Domain Events
**What happened:** Answered "outbox pattern" for both parts of the question — named the reliability mechanism but didn't name the cross-BC communication pattern (Integration Event) at all.
**Root cause:** Knows Outbox exists and what it does, but hasn't separated the two distinct concepts: what you send (Integration Event) vs how you guarantee delivery (Outbox).
**Fix / key insight:** Domain Event = in-process, same BC, MediatR. Integration Event = crosses BC boundary, message broker (RabbitMQ/Kafka). Outbox is the delivery guarantee for Integration Events: write event row in same DB transaction as state change, background worker publishes later.
**Revisit:** mid/ddd.md
**Revisit by:** 2026-05-03

## 2026-04-09
**Topic:** React Native — FlatList vs ScrollView
**What happened:** Said FlatList is "more complex and heavy" — correct direction but missed the key reason: virtualization.
**Root cause:** Knows the components exist but not the performance mechanism behind them.
**Fix / key insight:** ScrollView renders ALL children at once. FlatList only renders visible items — unmounts others. For any list that can grow, always use FlatList.
**Revisit:** my-stack/react-native.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** Microservices — when NOT to use them
**What happened:** Got the benefits right but completely ignored the second half of the question — when not to split.
**Root cause:** Knows the sales pitch for microservices, not the tradeoffs. Missing: network failure, distributed transactions, operational overhead, team size requirement.
**Fix / key insight:** Microservices solve org/scale problems, not code problems. Start monolith, extract when you feel pain. Small team + microservices = overhead with no benefit.
**Revisit:** senior/microservices.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** DSA — Two pointers
**What happened:** Only knew brute force O(n²). Correctly identified why two pointers needs sorted array after being taught.
**Root cause:** Never studied optimal patterns — jumped straight to nested loop thinking.
**Fix / key insight:** Sorted → two pointers O(n). Unsorted → hash set O(n). Sort-then-two-pointers is O(n log n) — valid but worse than hash set.
**Revisit:** junior/algorithms.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** CI/CD pipeline order
**What happened:** Listed lint, tests, deploy, restart — right instincts but wrong order and missing build + Docker steps. Confused pre-commit hooks with CI steps.
**Root cause:** Knows pieces of the pipeline but not the full sequence or why the order matters.
**Fix / key insight:** Order is lint → type check → tests → build → docker build/push → deploy. Cheapest first, fail fast. Docker image tagged with commit SHA for traceability. CI runs lint too even if pre-commit hook exists — because hooks can be skipped.
**Revisit:** junior/docker.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** React — re-renders and React.memo
**What happened:** Named useCallback/useMemo but didn't answer "which children re-render." Didn't know when NOT to use React.memo.
**Root cause:** Knows the hooks exist but not the mental model: all children re-render by default, memo + useCallback work as a pair.
**Fix / key insight:** React.memo skips render on shallow prop equality. useCallback stabilizes function props so memo's comparison works. Don't use memo for cheap components or when props always change.
**Revisit:** my-stack/react-performance.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** EF Core — N+1 problem
**What happened:** Described the concept correctly but used pseudocode. Missed that in EF Core N+1 is invisible — it looks like normal property access, not an explicit query call.
**Root cause:** Knows the pattern abstractly, hasn't seen it fail silently in real EF Core code.
**Fix / key insight:** EF lazy loading fires a query on every navigation property access inside a loop. The fix is `.Include()` for commands, `.Select()` projection for queries.
**Revisit:** mid/ef-core-advanced.md
**Revisit by:** 2026-04-27

## 2026-04-09
**Topic:** EF Core — Select vs Include
**What happened:** Said "select is for specific data, include is for whole entity" — correct but too vague.
**Root cause:** Didn't connect it to the CQRS pattern already in use.
**Fix / key insight:** Query handlers always use `.Select()` (read-only, DTO, no tracking). Command handlers use `.Include()` (need full aggregate to mutate). This is already the architecture — just apply it consciously.
**Revisit:** mid/ef-core-advanced.md, mid/cqrs.md
**Revisit by:** 2026-04-27

## 2026-04-20
**Topic:** Caching — cache-aside pattern
**What happened:** Could only say "we use Redis" — no description of hit/miss flow, no invalidation strategy.
**Root cause:** Knows Redis exists as a tool but never internalized the pattern mechanics.
**Fix / key insight:** Cache-aside = check cache first, on miss read DB then write to cache. On update, DELETE the key (never overwrite — avoids race conditions). Thundering herd is the main weakness: many concurrent misses on expiry → use mutex or background refresh.
**Revisit:** senior/caching.md
**Revisit by:** 2026-04-27

## 2026-04-20
**Topic:** Caching — write-through pattern
**What happened:** Described cache-aside on a miss instead of write-through. Also didn't know what TTL is.
**Root cause:** Write-through is about writes, not reads — confused the two. TTL never studied.
**Fix / key insight:** Write-through = on every write, update DB AND cache in the same operation. TTL = expiry timer on a Redis key — automatic deletion, safety net against stale data forever.
**Revisit:** senior/caching.md
**Revisit by:** 2026-04-27

## 2026-04-21
**Topic:** DSA — Two pointers
**What happened:** Started both pointers at the beginning (left=0, right=1) instead of opposite ends.
**Root cause:** Missed the core insight: sorted array means start at opposite ends and move inward — large+small, adjust based on sum.
**Fix / key insight:** left=0, right=arr.Length-1. Sum too small → left++. Sum too big → right--. Both pointers only move toward each other, never past.
**Revisit:** junior/algorithms.md
**Revisit by:** 2026-04-27

## 2026-04-21
**Topic:** Docker — container networking
**What happened:** Said `localhost` works between containers because "they're on the same machine."
**Root cause:** Didn't know each container has its own isolated network namespace. localhost = self, not host or other containers.
**Fix / key insight:** Use the Docker Compose service name as hostname. API connects to `db:1433`, not `localhost:1433`. Compose creates a shared internal network automatically.
**Revisit:** junior/docker.md
**Revisit by:** 2026-04-27

## 2026-04-21
**Topic:** DSA — Recursion
**What happened:** Wrote Fibonacci instead of factorial, and used iteration not recursion.
**Root cause:** Didn't recognise the pattern — recursion requires a function calling itself with a base case to stop.
**Fix / key insight:** Every recursive function needs: (1) base case that returns immediately, (2) recursive call that moves closer to the base case. No base case = stack overflow.
**Revisit:** junior/algorithms.md
**Revisit by:** 2026-04-27

## 2026-04-26
**Topic:** Security — CORS preflight
**What happened:** Knew CORS needs a policy on the server but didn't mention the OPTIONS preflight request or the `Access-Control-Allow-Origin` response header.
**Root cause:** Knows CORS exists as a config step, not the browser mechanism behind it.
**Fix / key insight:** Browser sends OPTIONS preflight first. Server must respond with `Access-Control-Allow-Origin` header. Without it the browser blocks the response — the request still hits the server, the browser just hides the response from JS.
**Revisit:** mid/security-depth.md
**Revisit by:** 2026-05-01

## 2026-04-26
**Topic:** Security — CSRF vs localStorage JWT
**What happened:** Said localStorage is "easy to steal" (XSS risk) instead of explaining why it's immune to CSRF. Got the two attack types mixed up.
**Root cause:** Confused CSRF vulnerability (auto-sent cookies) with XSS vulnerability (JS-readable storage). Two completely different attack vectors.
**Fix / key insight:** CSRF works because browsers auto-attach cookies to every request — even from malicious sites. localStorage is never auto-sent and can't be read cross-origin. Fix for cookie auth: `SameSite=Strict` or `SameSite=Lax`.
**Revisit:** mid/security-depth.md
**Revisit by:** 2026-05-01

## 2026-04-26
**Topic:** Security — Stored XSS
**What happened:** Blank answer — no knowledge of XSS types.
**Root cause:** Never studied XSS. Topic is `[ ]` in PROGRESS.md.
**Fix / key insight:** Stored XSS = malicious script saved to DB, executes in every visitor's browser. Two fixes: (1) escape output on render — React does this automatically, never use `dangerouslySetInnerHTML` with user input; (2) CSP header `Content-Security-Policy: default-src 'self'` blocks inline scripts even if they reach the HTML.
**Revisit:** mid/security-depth.md
**Revisit by:** 2026-05-01

## 2026-04-26
**Topic:** Security — HTTP security headers
**What happened:** Named made-up headers. None of the three answers were real standard security headers.
**Root cause:** Never studied security headers. No mental model at all.
**Fix / key insight:** Three must-know: `Content-Security-Policy: default-src 'self'` (kills XSS), `X-Frame-Options: DENY` (prevents clickjacking), `Strict-Transport-Security: max-age=31536000; includeSubDomains` (forces HTTPS).
**Revisit:** mid/security-depth.md
**Revisit by:** 2026-05-01
