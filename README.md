# Personal Knowledge Repository

A structured collection of software engineering knowledge, organized by level and topic.

---

## Folder Design

```
swe-knowledge/
│
├── junior/          ← Concepts expected at first job or entry interview
├── mid/             ← Concepts expected at mid-level / remote SWE role
├── senior/          ← Concepts for senior roles, system design, ops depth
├── my-stack/        ← Technology-specific deep dives (C#, React, TS, etc.)
│                      Each file covers one technology from basics to advanced.
│                      Level is noted at the top of each file.
├── ai-integration/  ← LLM APIs, RAG, agentic patterns
├── practice/
│   └── mistakes.md  ← Every gap caught in a session, with revisit dates
├── PROGRESS.md      ← Master checklist — update confidence after every session
└── CLAUDE.md        ← AI context file — paste at the start of every AI session
```

---

## Level Guide

| Folder | Who needs it | Examples |
|---|---|---|
| `junior/` | Before or at first job | HTTP/REST, Git, SQL basics, Big-O, algorithms, clean code, agile |
| `mid/` | 1-3 years or targeting remote SWE | DDD, CQRS, Clean Architecture, EF Core, event-driven design, auth, testing |
| `senior/` | 3+ years or targeting senior/lead | System design, microservices, Kubernetes, observability, caching at scale |
| `my-stack/` | Alongside any level | C#, ASP.NET Core, TypeScript, React, Redux, React Native, Next.js |
| `ai-integration/` | When adding AI features | LLM API calls, RAG pipelines, agentic patterns, prompt engineering |

---

## File Index

### junior/
| File | What it covers |
|---|---|
| `algorithms.md` | Sorting, searching, two pointers, sliding window, recursion |
| `big-o.md` | Time/space complexity, best/average/worst case |
| `browser-internals.md` | DNS → TCP → render pipeline, reflow vs repaint |
| `clean-code.md` | Naming, guard clauses, DRY/YAGNI/KISS, code smells |
| `data-structures.md` | Arrays, linked lists, stacks, queues, trees, hash maps |
| `git.md` | Branching, rebase vs merge, PR workflow, reflog, bisect |
| `http-rest.md` | HTTP verbs, status codes, REST constraints, idempotency |
| `sql-basics.md` | SELECT/JOIN/GROUP BY, indexes, transactions, window functions |
| `agile-scrum.md` | Scrum roles, ceremonies, user stories, story points |

### mid/
| File | What it covers |
|---|---|
| `clean-architecture.md` | Layer responsibilities, dependency rule, Result pattern |
| `ddd.md` | Entities, Value Objects, Aggregates, Domain Events, Bounded Contexts |
| `shared-kernel.md` | AggregateRoot, Entity, ValueObject, Result<T>, ErrorCodes base classes |
| `cqrs.md` | Command/Query separation, MediatR, pipeline behaviors |
| `modular-monolith.md` | Module structure, separate DbContexts, module registration |
| `event-driven-flow.md` | Full event flow — domain event → outbox → RabbitMQ → inbox → projection |
| `event-driven-implementation.md` | Every file, folder, and line of code in the full flow |
| `bc-knowledge-map.md` | Complete map of everything inside one Bounded Context |
| `api-full-knowledge-map.md` | Complete map of everything needed to build a production API |
| `ef-core-advanced.md` | Migrations, owned entities, value converters, interceptors, bulk ops |
| `authentication.md` | JWT, OAuth2, refresh tokens, PKCE, cookie attributes |
| `security-depth.md` | CORS, CSRF, XSS, security headers, SQL injection |
| `api-design.md` | Pagination, versioning, idempotency, GraphQL vs REST vs gRPC |
| `design-patterns.md` | GoF patterns: Factory, Strategy, Observer, Decorator, etc. |
| `solid.md` | SRP, OCP, LSP, ISP, DIP with C# examples |
| `testing.md` | Unit, integration, mocking, Testcontainers, WebApplicationFactory |
| `resilience.md` | Polly: retry, circuit breaker, timeout, bulkhead |
| `react-internals.md` | Reconciliation, virtual DOM, Suspense, concurrent features |
| `react-patterns.md` | HOC, render props, compound components, portals, error boundaries |
| `rtk-query.md` | injectEndpoints, cache tags, optimistic updates |
| `sql-advanced.md` | CTEs, indexes, execution plans, deadlocks, connection pooling |
| `docker.md` | Dockerfile, docker-compose, multi-stage builds, networking |
| `behavioral.md` | STAR method, core stories, project walkthrough, "tell me about yourself" |

### senior/
| File | What it covers |
|---|---|
| `system-design.md` | URL shortener, rate limiter, news feed, chat, file storage |
| `caching.md` | Cache-aside, write-through, Redis structures, eviction, consistency |
| `message-queues.md` | RabbitMQ, outbox pattern, at-least-once vs exactly-once |
| `microservices.md` | Decomposition, service discovery, distributed tracing, saga |
| `api-gateway.md` | YARP, JWT validation at gateway, rate limiting, routing |
| `observability.md` | Serilog, OpenTelemetry, health checks, metrics, tracing |
| `ci-cd.md` | GitHub Actions, Docker build, blue/green deployment, secrets |
| `kubernetes.md` | Pod/Deployment/Service, probes, rolling updates, ConfigMap |
| `advanced-algorithms.md` | Dijkstra, topological sort, dynamic programming, backtracking |

### my-stack/
| File | Level | What it covers |
|---|---|---|
| `csharp.md` | Junior → Mid | OOP, async/await, LINQ, generics, delegates |
| `csharp-nullable.md` | Junior → Mid | Nullable types, null operators, required members |
| `csharp-exception-handling.md` | Mid | throw vs Result<T>, global handler, ProblemDetails |
| `csharp-di-deep.md` | Mid | Captive dependency, IServiceScopeFactory, Options pattern |
| `csharp-background-jobs.md` | Mid | IHostedService, BackgroundService, Channel<T>, Hangfire |
| `csharp-concurrency.md` | Mid → Senior | lock, SemaphoreSlim, async pitfalls, CancellationToken |
| `csharp-memory.md` | Senior | GC generations, boxing, Span<T>, ArrayPool |
| `aspnet.md` | Mid | Controllers, middleware, filters, DI lifetimes, minimal APIs |
| `typescript-js.md` | Junior → Mid | Closures, event loop, Promises, prototype chain |
| `typescript-advanced.md` | Mid | Utility types, discriminated unions, mapped types, infer |
| `typescript-react.md` | Mid | Typing props, events, hooks, context, generic components |
| `react.md` | Junior → Mid | Hooks, context, routing, controlled forms |
| `react-performance.md` | Mid | memo, useMemo, useCallback, stale closures, code splitting |
| `react-forms.md` | Mid | React Hook Form, Zod, setError, useFieldArray |
| `redux-toolkit.md` | Mid | Slices, selectors, createAsyncThunk, DevTools |
| `zustand.md` | Mid | create(), useShallow, persist, Zustand vs Redux |
| `react-native.md` | Mid (beginner) | Core components, navigation, WatermelonDB, StyleSheet |
| `nextjs.md` | Mid | SSR/SSG/ISR, App Router, Server Components, middleware |

### ai-integration/
| File | What it covers |
|---|---|
| `llm-api-basics.md` | Anthropic/OpenAI API, calling from .NET, streaming |
| `rag.md` | Chunking, embeddings, vector stores, retrieval pipeline |
| `agentic-patterns.md` | Tool use, prompt chaining, reflection, multi-agent |
| `prompt-engineering.md` | Chain-of-thought, few-shot, COSTAR, output format control |

---

## How to Use

| Goal | Action |
|---|---|
| Start a new AI session | Paste `CLAUDE.md` — the AI gets full context instantly |
| Find what to study | Open `PROGRESS.md`, find `[ ]` or `[~]` topics |
| Study a concept | Open the file in the matching level folder |
| Study a technology | Open the file in `my-stack/` |
| Review mistakes | Open `practice/mistakes.md`, check `Revisit by` dates |

---

## Confidence Scale

| Symbol | Meaning |
|---|---|
| `[ ]` | Never studied |
| `[~]` | Studied, mostly forgotten |
| `[b]` | Know the basics |
| `[c]` | Comfortable |
| `[x]` | Can explain and implement from memory — could teach it |

**Rule:** Never give yourself `[c]` if you couldn't answer an interview question on it cold. Never give `[x]` if you'd need to look up syntax.
