# PROGRESS.md — Master Learning Checklist

Confidence scale:
- `[ ]` Never studied
- `[~]` Studied, mostly forgotten
- `[b]` Know the basics
- `[c]` Comfortable
- `[x]` Can explain and implement from memory

Update this file after every study session. Be honest.

---

## JUNIOR LEVEL

### Data Structures
- `[~]` Arrays & dynamic arrays
- `[~]` Linked lists (singly, doubly)
- `[~]` Stacks & queues
- `[~]` Hash maps / hash tables
- `[b]` Trees (binary tree, BST)
- `[b]` Heaps (min/max)
- `[b]` Tries
- `[ ]` Graphs (adjacency list vs matrix)

### Algorithms
- `[~]` Binary search
- `[~]` Bubble / Selection / Insertion sort
- `[~]` Merge sort
- `[~]` Quick sort
- `[b]` BFS
- `[b]` DFS
- `[b]` Two pointers
- `[b]` Sliding window
- `[b]` Prefix sums
- `[b]` Recursion fundamentals

### Big O
- `[b]` Time complexity analysis
- `[b]` Space complexity analysis
- `[b]` Best / average / worst case
- `[~]` Amortized analysis

### HTTP & REST
- `[c]` HTTP verbs (GET/POST/PUT/PATCH/DELETE)
- `[c]` Status codes
- `[b]` Headers, content negotiation
- `[b]` REST constraints
- `[b]` Idempotency
- `[b]` API versioning strategies
- `[ ]` HTTP/2 — multiplexing, header compression
- `[ ]` HTTP/3 / QUIC — UDP-based, 0-RTT connection, connection migration; why it matters for mobile
- `[ ]` WebSockets vs SSE vs long polling — when each
- `[ ]` WebRTC basics — peer-to-peer media; ICE/STUN/TURN; when you need it (video calls, live collab)

### SQL Basics
- `[c]` SELECT, WHERE, ORDER BY, GROUP BY
- `[c]` JOINs (INNER, LEFT, RIGHT, FULL)
- `[b]` Indexes (when and why)
- `[b]` Transactions (ACID)
- `[b]` Normalization (1NF/2NF/3NF)
- `[b]` N+1 query problem
- `[b]` Window functions (ROW_NUMBER, RANK, LAG, LEAD)
- `[b]` HAVING vs WHERE (execution order)

### Git
- `[c]` Add, commit, push, pull
- `[c]` Branching, merging
- `[b]` Rebase vs merge
- `[b]` PR workflow
- `[b]` Cherry-pick, stash
- `[b]` Advanced branching strategies (GitFlow, trunk-based)
- `[ ]` Interactive rebase — squash, reword, fixup
- `[ ]` reflog — recovering lost commits
- `[ ]` bisect — binary search for bug-introducing commit

### Browser Internals
- `[ ]` DNS → TCP → TLS → HTTP → render — full URL flow
- `[ ]` HTML parsing → DOM, CSS parsing → CSSOM
- `[ ]` Render tree, layout, paint, composite pipeline
- `[ ]` Reflow vs repaint vs composite-only
- `[ ]` Layout thrashing — causes and fix
- `[ ]` Core Web Vitals — LCP, CLS, INP

### Clean Code
- `[ ]` Naming — intent-revealing, no noise words
- `[ ]` Functions — single responsibility, guard clauses
- `[ ]` Comments — when to write them (WHY not WHAT)
- `[ ]` DRY, YAGNI, KISS — applying each correctly
- `[ ]` Code smells — recognising in review

### Agile & Scrum
- `[ ]` Agile values — four core trade-offs
- `[ ]` Scrum roles — PO, SM, Dev Team
- `[ ]` Sprint ceremonies — planning, standup, review, retro
- `[ ]` User stories + acceptance criteria
- `[ ]` Story points, velocity, Definition of Done
- `[ ]` Scrum vs Kanban

### Computer Science Fundamentals
- `[ ]` Processes vs threads — creation cost, context switching, shared memory
- `[ ]` Virtual memory — pages, page faults, address space layout, stack vs heap segments
- `[ ]` File descriptors — everything is a file (socket, pipe, disk); open/close lifecycle
- `[ ]` OSI model — 7 layers; practical focus: L3 IP, L4 TCP/UDP, L7 HTTP/TLS
- `[ ]` TCP — 3-way handshake, reliable delivery, flow control, congestion control (Nagle's algorithm)
- `[ ]` UDP — why faster (no handshake, no retransmit); used in DNS, video streaming, gaming
- `[ ]` DNS — recursive resolver → root → TLD → authoritative; TTL; record types (A, CNAME, MX, TXT)
- `[ ]` CDN — edge PoPs, origin shield, cache-hit ratio, cache invalidation strategies
- `[ ]` Load balancer — L4 (TCP) vs L7 (HTTP); round-robin, least-connections, IP hash, sticky sessions

---

## MID LEVEL

### Design Patterns (GoF)
- `[b]` Factory / Abstract Factory
- `[b]` Builder
- `[b]` Singleton
- `[b]` Repository
- `[b]` Strategy
- `[b]` Observer
- `[b]` Decorator
- `[~]` Adapter
- `[~]` Facade
- `[ ]` Command pattern
- `[ ]` Chain of responsibility

### SOLID Principles
- `[c]` Single Responsibility
- `[b]` Open/Closed
- `[b]` Liskov Substitution
- `[b]` Interface Segregation
- `[b]` Dependency Inversion

### Authentication & Security
- `[b]` JWT structure (header.payload.signature)
- `[b]` Access token vs refresh token
- `[b]` OAuth2 flows
- `[b]` Cookies vs tokens
- `[~]` HTTPS, TLS basics
- `[b]` OWASP top 10
- `[ ]` PKCE — how it works, why SPAs need it
- `[ ]` OpenID Connect (OIDC) — identity layer on top of OAuth2; `id_token` vs `access_token`; `/.well-known/openid-configuration`
- `[ ]` HttpOnly + Secure + SameSite cookie attributes
- `[ ]` Session-based vs token-based auth — trade-offs
- `[ ]` CORS — preflight, headers, ASP.NET Core setup
- `[ ]` CSRF — how the attack works, SameSite defense
- `[ ]` XSS — stored vs reflected vs DOM, CSP, React safety
- `[ ]` SQL injection prevention — parameterized queries
- `[ ]` Security headers (X-Frame-Options, CSP, etc.)
- `[ ]` Secrets management — user secrets, env vars, Key Vault

### EF Core Advanced
- `[b]` Migrations
- `[b]` Relationships (1:1, 1:N, M:N)
- `[b]` AsNoTracking
- `[~]` Query performance, includes vs projections
- `[b]` Owned entities / ComplexProperty (Value Objects)
- `[b]` Value Converters
- `[~]` Interceptors (SaveChangesInterceptor)
- `[c]` Bulk operations (ExecuteUpdateAsync / ExecuteDeleteAsync)
- `[b]` Change Tracker states
- `[b]` JSON Columns
- `[b]` DbContext Pooling
- `[b]` Optimistic Concurrency (RowVersion)
- `[b]` Multi-tenancy (Global Query Filters) — gap: IgnoreQueryFilters()
- `[b]` Testcontainers (real DB integration tests)
- `[b]` Background Workers (IHostedService)
- `[b]` Dispatching Domain Events in SaveChangesAsync
- `[~]` Interceptor vs SaveChangesAsync override — when to choose which
- `[b]` Backing field EF Core wiring (SetField configuration)
- `[ ]` Vector Search (EF Core 10)

### Clean Architecture
- `[c]` Layer responsibilities
- `[c]` Dependency rule
- `[b]` Interfaces at boundaries
- `[b]` Use case pattern
- `[b]` Full BC knowledge map — every pattern, library, layer
- `[ ]` Vertical Slice Architecture — organize by feature not layer; compare trade-offs vs Clean Architecture

### DDD
- `[b]` Entities vs Value Objects
- `[b]` Aggregates & aggregate roots
- `[b]` Domain Events
- `[b]` Repository per aggregate root
- `[b]` Unit of Work
- `[b]` Bounded contexts
- `[b]` Ubiquitous language
- `[b]` Rich vs Anemic domain model
- `[b]` Backing fields (IReadOnlyCollection)
- `[b]` Domain Events vs Integration Events

### Shared Kernel
- `[b]` Entity vs AggregateRoot base class implementation
- `[b]` ValueObject equality implementation
- `[ ]` IDomainEvent vs IIntegrationEvent — what each is for
- `[b]` Result<T> pattern — implement from scratch
- `[b]` IUnitOfWork — interface + implementation
- `[b]` ErrorCodes — structure and usage

### Modular Monolith
- `[b]` Module structure — Domain/Application/Infrastructure/Tests per module
- `[b]` Separate DbContext per module with schema-per-module
- `[b]` Module registration in Program.cs (extension methods)
- `[b]` Public Module API pattern (ICatalogModule)
- `[b]` No cross-module EF navigation properties
- `[b]` Separate migrations per DbContext
- `[b]` When to use in-process call vs integration event
- `[b]` Repository in BC — one per aggregate root, only touches own DbContext
- `[b]` BC configuration — full module wiring (DbContext, repos, MediatR, MassTransit, workers)

### CQRS + MediatR
- `[b]` Command vs Query separation
- `[b]` MediatR pipeline behaviors
- `[b]` Validation pipeline behavior
- `[~]` Event sourcing basics
- `[b]` Outbox pattern

### Event-Driven Flow
- `[b]` Full event flow — domain event → outbox → RabbitMQ → consumer → inbox
- `[b]` Outbox pattern — manual implementation
- `[ ]` MassTransit built-in Outbox setup
- `[b]` Inbox pattern — idempotent consumer
- `[b]` Dead Letter Queue — RabbitMQ queue (not DB table), auto-routed on retry exhaustion
- `[ ]` MassTransit consumer setup
- `[b]` Saga basics — state machine for long-running workflows
- `[ ]` RabbitMQ + Docker setup
- `[b]` Projections — read-optimized tables built from events, queries read from these
- `[b]` Full implementation — folder structure, every file, every role

### React Internals
- `[c]` Components, props, state
- `[c]` useEffect, useMemo, useCallback
- `[b]` Reconciliation algorithm
- `[b]` Virtual DOM
- `[b]` Rendering optimization (React.memo)
- `[b]` Concurrent features (Suspense, transitions)

### RTK Query
- `[c]` injectEndpoints, baseApi
- `[c]` Cache tags, invalidation
- `[b]` Optimistic updates
- `[b]` Polling
- `[~]` Streaming / subscriptions

### Redux Toolkit
- `[b]` Slices (actions, reducers, Immer)
- `[b]` Typed hooks (useAppSelector, useAppDispatch)
- `[b]` createSelector / memoized selectors
- `[b]` createAsyncThunk + extraReducers
- `[b]` Redux DevTools
- `[b]` Redux vs RTK Query — what goes where

### Zustand
- `[b]` create() — basic store with state + actions
- `[b]` Selecting specific state (avoiding full store subscription)
- `[b]` useShallow for multiple fields
- `[b]` Async actions
- `[b]` persist middleware
- `[b]` Zustand vs Redux Toolkit — when to choose which

### State Management — Landscape
- `[ ]` Server state vs client/UI state — the fundamental split (server state = TanStack Query / RTK Query; UI state = Zustand / Context / Redux)
- `[ ]` TanStack Query (React Query v5) — `useQuery`, `useMutation`, `QueryClient`, stale-while-revalidate, `invalidateQueries`
- `[ ]` TanStack Query vs RTK Query — when to choose each (project coupling, backend ownership)
- `[ ]` SWR — Vercel's data fetching; `useSWR`, `mutate`, revalidation strategies; lighter than TanStack Query
- `[ ]` Jotai — atomic state model; `atom()`, `useAtom`; bottom-up composition; no store singleton
- `[ ]` Jotai vs Zustand — atoms vs slices; when atomic model wins
- `[ ]` MobX — observable/computed/action; reactive model; good for complex derived state
- `[ ]` XState (v5) — explicit state machines + statecharts; `createMachine`, `useMachine`; when state has complex transitions
- `[ ]` Context API limits — why Context is not a state manager (no selectors, rerenders on every value change)
- `[ ]` State management decision matrix — server state / shared UI state / local component state / URL state

### React Patterns
- `[b]` Error Boundaries — when to use, what they don't catch
- `[b]` HOC — implement one, explain tradeoffs
- `[b]` Render Props — recognize and implement the pattern
- `[b]` Compound Components — Context-based sub-component API
- `[b]` Portals — when and how to use
- `[b]` Custom hooks as the modern replacement for HOC/render props

### Docker
- `[b]` Dockerfile basics
- `[b]` docker-compose
- `[b]` Multi-stage builds
- `[b]` Networking, volumes
- `[ ]` Container registries

### Resilience Patterns
- `[ ]` Retry with exponential backoff + jitter
- `[ ]` Circuit breaker — three states, thresholds
- `[ ]` Timeout pattern
- `[ ]` Bulkhead isolation
- `[ ]` Fallback pattern
- `[ ]` Polly v8 ResiliencePipeline API
- `[ ]` HttpClientFactory + Polly integration

### API Design (Advanced)
- `[ ]` Offset vs cursor/keyset pagination — trade-offs
- `[ ]` GraphQL — query model, N+1, when over REST
- `[ ]` gRPC — Protobuf, streaming, when over REST
- `[ ]` gRPC in .NET — `Grpc.AspNetCore`, `.proto` file generation, `GrpcChannel` client, streaming RPCs
- `[ ]` GraphQL in .NET — HotChocolate server; schema-first vs code-first; `IQueryable` integration
- `[ ]` Idempotency keys for POST endpoints

### SQL Advanced
- `[ ]` CTEs — basic and recursive
- `[ ]` Clustered vs non-clustered indexes
- `[ ]` Covering indexes
- `[ ]` Composite index column order
- `[ ]` Execution plans — table scan vs index seek
- `[ ]` Deadlocks — causes and prevention
- `[ ]` Connection pooling + DbContext lifetime

### SQL Server (Ivan's Stack)
- `[ ]` Query Store — tracks execution plan history; find plan regressions without a profiler
- `[ ]` Temporal Tables — system-versioned history; `FOR SYSTEM_TIME AS OF` queries; audit + time-travel
- `[ ]` Always Encrypted — column-level encryption; keys never leave client; EF Core provider support
- `[ ]` SQL Server Profiler / Extended Events — lightweight production tracing; replace Profiler with XEvents
- `[ ]` Database snapshots — point-in-time copy for reporting without blocking; no transaction log needed
- `[ ]` NOLOCK hint — risks (dirty reads, phantom reads); when acceptable; better alternatives (Read Committed Snapshot)
- `[ ]` Columnstore indexes — analytical workloads; batch mode execution; 10× compression typical

### Behavioral & Career
- `[ ]` STAR method — Situation, Task, Action, Result; practicing with your actual projects
- `[ ]` Core stories prepared — conflict, failure, leadership, influence without authority, tight deadline
- `[ ]` "Tell me about yourself" pitch — 90-second version, tailored per company type
- `[ ]` Technical project walkthrough — architecture decisions, trade-offs, what you'd do differently
- `[ ]` Salary negotiation — never give number first, counter with specific number, full package (base + equity + PTO + learning budget)
- `[ ]` Asking good interview questions — questions that reveal culture, tech debt, team health, red flags
- `[ ]` Remote work best practices — async communication, visibility, documentation culture
- `[ ]` Engineering communication — writing clear Slack/PR/email messages; async vs sync choices
- `[ ]` Receiving feedback — not getting defensive; separating code critique from personal critique
- `[ ]` Giving feedback — code review tone, "I notice" vs "you should", asking questions vs demanding

### Interview Prep — Junior Level
- `[ ]` Junior interview format — what each round tests, what "coachable and collaborative" looks like to them
- `[ ]` Junior question bank — OOP, DI, async basics, HTTP verbs, SQL joins, Git branching
- `[ ]` Easy LeetCode patterns — Two Sum (hash map), valid parentheses (stack), palindrome, frequency count
- `[ ]` Junior take-home checklist — working app > perfect architecture, README, frequent commits, no unexplained tech

### Interview Prep — Mid Level
- `[ ]` Mid interview format — 4 rounds (recruiter, coding, component design, behavioral); what each tests
- `[ ]` Live coding 6-step framework — Understand → Clarify → Examples → Approach → Code → Test
- `[ ]` When stuck strategies — thinking out loud, brute force first, ask for hints without going silent
- `[ ]` Mid question bank — middleware pipeline, EF change tracker, Domain vs Integration events, Outbox, CancellationToken, captive dependency
- `[ ]` React question bank — re-render triggers, useEffect rules, controlled vs uncontrolled, RTK Query vs Zustand
- `[ ]` Component design questions — shopping cart, caching layer, notification service, file upload
- `[ ]` Take-home best practices — README with decisions, tests, clean git history, 4-8 hour signal, .env.example

### Interview Prep — Senior Level
- `[ ]` Senior interview format — coding + system design ×2 + leadership behavioral; ambiguity handling is core
- `[ ]` System design framework — clarify → estimate → API → data model → high-level → deep dive → trade-offs
- `[ ]` System design topics — URL shortener, rate limiter, notification service, chat, feed, file storage
- `[ ]` Senior question bank — CAP in financial systems, exactly-once processing, zero-downtime migrations, 10× traffic
- `[ ]` Leadership behavioral prep — influence without authority, technical debt under pressure, handling a production incident

### Interview Readiness
- `[ ]` Company research checklist done before each interview (stack, culture, stage, salary range, interviewer LinkedIn)
- `[ ]` Red flags recognition — "no time for tests", "we're like a family", all engineers < 1 year, exploding offers
- `[ ]` Interview timeline — 4-week prep plan (CV → LeetCode → stories → mock → company research)
- `[ ]` Mock interview completed — live coding out loud, recorded or with a partner

### Testing
- `[b]` Unit testing with xUnit
- `[b]` Mocking with Moq
- `[~]` Integration testing with TestContainers
- `[~]` What to test and what not to test
- `[ ]` Frontend testing (React Testing Library)
- `[ ]` Test doubles — Mock vs Stub vs Spy vs Fake vs Dummy
- `[ ]` RTL queries — getBy vs queryBy vs findBy, role-first approach
- `[ ]` Testing async components with RTL + MSW

### Event Sourcing
- `[ ]` Append-only event log — events are immutable facts; state is derived, never stored directly
- `[ ]` Rebuilding current state from event replay
- `[ ]` Snapshots — when to use (high event count); stored alongside events, not instead of them
- `[ ]` Event versioning — upcasters for backwards-compatible schema evolution
- `[ ]` CQRS + Event Sourcing as a natural pair — command writes event, read side projects
- `[ ]` EventStoreDB vs Postgres as event store — trade-offs
- `[ ]` When NOT to use Event Sourcing — simple CRUD, reporting-heavy, no audit requirement

### Architecture Patterns
- `[ ]` Hexagonal Architecture (Ports and Adapters) — primary ports (driving), secondary ports (driven), adapters
- `[ ]` Hexagonal vs Clean Architecture — same idea, different vocabulary; compare trade-offs
- `[ ]` Context Mapping — ACL, Conformist, Partnership, Shared Kernel, Customer/Supplier, Published Language
- `[ ]` Strangler Fig pattern — incrementally replace a monolith feature by feature
- `[ ]` Architecture Decision Records (ADRs) — format (title/status/context/decision/consequences), where to store

### Performance Engineering
- `[ ]` BenchmarkDotNet — `[Benchmark]`, `[MemoryDiagnoser]`; reading Mean, Allocated columns
- `[ ]` dotTrace — CPU Timeline mode vs Sampling; identifying hot paths and lock contention
- `[ ]` dotMemory — heap snapshots, retained vs shallow size, allocation tracking in unit tests
- `[ ]` `System.IO.Pipelines` — `PipeReader`/`PipeWriter`; backpressure, memory reuse, zero-copy parsing
- `[ ]` Allocation-free patterns — struct enumerators, `ArrayPool`, `Span<T>` slicing, avoiding LINQ allocations

---

## SENIOR LEVEL

### System Design
- `[b]` URL shortener
- `[b]` Rate limiter — token bucket, fixed window, sliding window
- `[b]` Rate limiter — Redis as shared state, Lua atomicity
- `[b]` API Gateway vs per-service middleware
- `[b]` Horizontal vs vertical scaling
- `[b]` Stateless design requirement for horizontal scaling
- `[b]` Load balancer role and placement
- `[b]` News feed / timeline
- `[b]` Notification system
- `[b]` File storage system
- `[b]` Chat system
- `[b]` CAP theorem (read, not tested)

### Caching
- `[b]` Cache-aside pattern
- `[b]` Write-through / write-behind
- `[b]` Redis data structures
- `[b]` Cache eviction policies
- `[b]` Distributed cache consistency — cache invalidation via integration events, update DB first then delete Redis key

### Message Queues
- `[b]` RabbitMQ basics (exchange, queue, binding)
- `[b]` Producer/consumer pattern
- `[ ]` Dead letter queues
- `[b]` Outbox pattern + reliable messaging
- `[b]` At-least-once vs exactly-once delivery

### Advanced Algorithms
- `[ ]` Dijkstra's shortest path
- `[ ]` Topological sort
- `[ ]` Union-Find (Disjoint Set)
- `[ ]` Dynamic Programming (top-down, bottom-up)
- `[ ]` Greedy algorithms
- `[ ]` Backtracking

### Microservices
- `[ ]` Service decomposition principles
- `[ ]` API gateway pattern
- `[ ]` Service discovery
- `[ ]` Distributed tracing
- `[ ]` Saga pattern

### CI/CD
- `[b]` GitHub Actions basics
- `[b]` Build, test, deploy pipeline
- `[b]` Docker build in CI
- `[b]` Environment secrets management
- `[b]` Blue/green deployment

### Observability
- `[b]` Structured logging with Serilog
- `[b]` Log levels — when to use each
- `[b]` Metrics — request rate, error rate, p99 latency
- `[b]` Distributed tracing — TraceId, spans
- `[b]` Health checks — /health/live vs /health/ready
- `[b]` Correlation ID pattern
- `[ ]` OpenTelemetry setup in .NET

### API Gateway
- `[ ]` What an API Gateway does vs a Load Balancer
- `[ ]` YARP setup — routes, clusters, destinations
- `[ ]` JWT validation at gateway, forwarding user identity
- `[ ]` Rate limiting at gateway level
- `[ ]` YARP vs Ocelot — when to choose which

### Kubernetes
- `[ ]` Pod vs Deployment vs Service vs Ingress
- `[ ]` liveness vs readiness probes
- `[ ]` Stateless app requirements for K8s
- `[ ]` ConfigMap and Secret for configuration
- `[ ]` Rolling updates and rollbacks
- `[ ]` Basic kubectl commands
- `[ ]` Docker Compose for local dev vs K8s for production

### Distributed Systems Theory
- `[ ]` CAP theorem — deep: Consistency vs Availability under Partition; CP (Zookeeper, HBase) vs AP (Cassandra, DynamoDB) in real systems
- `[ ]` BASE — Basically Available, Soft state, Eventually consistent; contrast with ACID
- `[ ]` Eventual consistency patterns — read-your-writes, monotonic reads, causal consistency, session guarantees
- `[ ]` Two-Phase Commit (2PC) — coordinator/participant protocol; blocking problem; why distributed systems avoid it
- `[ ]` Saga vs 2PC — compensating transactions; choreography (events) vs orchestration (state machine)
- `[ ]` Idempotency at scale — natural keys, deduplication windows, outbox uniqueness, at-least-once delivery guarantee
- `[ ]` Vector clocks — causal ordering of events; detecting concurrent writes in distributed stores
- `[ ]` CRDTs — conflict-free merge (G-Counter, LWW-Register, OR-Set); used in Redis, Riak, collaborative editing
- `[ ]` Raft consensus — leader election, log replication, term numbers; understand the guarantees, don't implement
- `[ ]` Gossip protocol — epidemic dissemination; failure detection; used in Cassandra, Consul
- `[ ]` Consistent hashing — virtual nodes, token ring; minimal rehashing when nodes join/leave

### Database Internals
- `[ ]` B-tree — page structure, internal vs leaf nodes, fill factor, why random inserts cause fragmentation
- `[ ]` LSM tree — write-optimized (memtable → SSTable → compaction); used in RocksDB, Cassandra, InfluxDB
- `[ ]` Write-Ahead Log (WAL) — fsync guarantee, crash recovery, log sequence numbers
- `[ ]` MVCC — multi-version concurrency control; snapshot isolation without read locks; version chain in Postgres
- `[ ]` VACUUM / AUTOVACUUM — dead tuple reclamation, table bloat, aggressive settings for write-heavy tables
- `[ ]` Index deep dive — B-tree, Hash (equality only), GIN (arrays/JSONB/full-text), GiST (ranges/geometry), BRIN (large sorted tables), covering (INCLUDE)
- `[ ]` Partitioning — range, hash, list; partition pruning; local vs global indexes; when sharding vs partitioning
- `[ ]` Replication — physical (WAL shipping) vs logical (row-level); replica lag monitoring; read replicas as cache
- `[ ]` Sharding — hash vs range shard key; cross-shard queries; resharding without downtime (consistent hashing)
- `[ ]` Query optimizer — statistics (pg_stats), cost model, join order, EXPLAIN ANALYZE reading (rows, cost, width)
- `[ ]` Connection pooling at scale — PgBouncer transaction mode; why app-level pooling isn't enough above ~1k conns

### NoSQL & Specialized Stores
- `[ ]` Redis deep — persistence (RDB snapshots vs AOF), Sentinel (HA), Cluster (sharding), Lua scripting, pub/sub
- `[ ]` Redis distributed lock — SET NX EX pattern; Redlock algorithm trade-offs (clock drift problem)
- `[ ]` Redis as cache vs queue vs pub-sub — strengths and failure modes of each use
- `[ ]` Elasticsearch / OpenSearch — inverted index, shards + replicas, query (scored) vs filter (cached), aggregations, mapping types
- `[ ]` Document databases (MongoDB) — embedded vs reference pattern; atomic ops at document level; eventual consistency trade-offs
- `[ ]` Time-series databases (TimescaleDB / InfluxDB) — hypertables, retention policies, continuous aggregates, downsampling
- `[ ]` Graph databases (Neo4j) — property graph model, Cypher query basics; when relationships are the query

### Cloud Platforms
- `[ ]` Azure core — App Service, AKS, Azure SQL, Blob Storage, Service Bus, Key Vault, Managed Identity, Front Door
- `[ ]` AWS core — EC2, ECS/EKS, RDS, S3, SQS/SNS, Secrets Manager, IAM roles/policies
- `[ ]` Infrastructure as Code — Terraform (provider/resource/state/modules) or Bicep (Azure-native HCL-like)
- `[ ]` Serverless — Azure Functions / AWS Lambda; cold start, execution time limits, event triggers, stateless constraints
- `[ ]` Cloud networking — VNet/VPC, subnets, NSG / Security Groups, Private Endpoints, peering
- `[ ]` Managed Identity — no secrets in config; app authenticates as Azure AD service principal; RBAC scope
- `[ ]` CDN + WAF — Azure Front Door / CloudFront; cache rules, custom domains, WAF rule sets, DDoS protection
- `[ ]` Cost optimization — reserved instances, spot/preemptible, autoscaling triggers, right-sizing tooling

### Advanced Security
- `[ ]` Threat modeling — STRIDE: Spoofing, Tampering, Repudiation, Info Disclosure, Denial of Service, Elevation of Privilege
- `[ ]` mTLS — mutual TLS; client certificate authentication; service-to-service in zero-trust networks
- `[ ]` Supply chain security — SBOM (Software Bill of Materials), signed NuGet packages, Dependabot, license scanning
- `[ ]` Zero trust architecture — never trust the network; verify every request; micro-segmentation; least privilege
- `[ ]` Cryptography fundamentals — AES-256 (symmetric), RSA/ECC (asymmetric), SHA-256 (hashing), PBKDF2/Argon2 (passwords), HMAC
- `[ ]` Secrets rotation — automated rotation, zero-downtime key rotation (dual-key pattern, grace period)
- `[ ]` Audit logging — tamper-evident storage, structured fields (actor, action, resource, result, timestamp)
- `[ ]` OWASP Dependency Check / Snyk — automated CVE scanning in CI; severity thresholds as build gates

### Professional Engineering Practices
- `[ ]` SLI / SLO / SLA — Service Level Indicators (what you measure), Objectives (targets), Agreements (contracts); error budget = 1 - SLO
- `[ ]` Error budgets — how much unreliability is allowed; when to ship vs when to focus on reliability
- `[ ]` DORA metrics — Deployment Frequency, Lead Time for Changes, Change Failure Rate, MTTR; measuring team health
- `[ ]` Incident response — severity levels, incident commander role, war room protocol, communication cadence
- `[ ]` Postmortems — blameless format; timeline, root cause, contributing factors, action items; systemic fixes over blame
- `[ ]` Runbooks — step-by-step operational procedures; written when calm, used when panicked
- `[ ]` On-call hygiene — alert fatigue, actionable alerts only, escalation paths, toil reduction
- `[ ]` Technical writing — design docs (problem/proposal/alternatives/trade-offs), RFCs, one-pagers
- `[ ]` Code review culture — review the code not the person; review for correctness + maintainability + learning
- `[ ]` Estimation — T-shirt sizing, Fibonacci story points, why estimates ≠ commitments; cone of uncertainty
- `[ ]` Technical debt — classifying (reckless vs prudent vs deliberate vs inadvertent), tracking, paying it down strategically
- `[ ]` Open source contribution — finding good first issues, PR etiquette, maintaining a contribution, building reputation

### Advanced Testing
- `[ ]` Test pyramid vs test trophy — ratio trade-offs; what each shape optimizes for (speed vs confidence)
- `[ ]` TDD — Red-Green-Refactor cycle; where it shines (domain logic), where it struggles (I/O-heavy code)
- `[ ]` BDD — Given-When-Then; Reqnroll (.NET) for living documentation tied to acceptance criteria
- `[ ]` Property-based testing — FsCheck (.NET) / fast-check (TS); generate arbitrary inputs to find edge cases automatically
- `[ ]` Mutation testing — Stryker.NET; mutants reveal tests that pass without asserting anything meaningful
- `[ ]` Contract testing — Pact; consumer-driven contracts; prevent API breakage between producer/consumer services
- `[ ]` Load testing — k6 / NBomber; VU scripts, ramp-up shape, steady state, interpreting p50/p95/p99/p999
- `[ ]` Chaos engineering — Simmy (Polly chaos extension); deliberately inject failures to verify resilience paths work

---

## AI INTEGRATION

### LLM API Basics
- `[b]` OpenAI / Anthropic API structure
- `[b]` Calling LLM from .NET (HttpClient / SDK)
- `[~]` Calling LLM from React (fetch / SDK)
- `[ ]` Streaming responses
- `[ ]` Token counting, cost estimation

### RAG
- `[ ]` Chunking strategies
- `[b]` Embeddings (what they are, how to generate)
- `[b]` Vector stores (pgvector, Qdrant)
- `[b]` Similarity search (cosine, dot product)
- `[b]` Retrieval pipeline end-to-end
- `[ ]` Re-ranking

### Agentic Patterns
- `[b]` Tool use / function calling
- `[~]` Prompt chaining
- `[ ]` Reflection (self-critique loop)
- `[ ]` Planning (ReAct, chain-of-thought)
- `[ ]` Multi-agent systems
- `[ ]` Memory management (short-term vs long-term)

### Prompt Engineering
- `[b]` Basic prompt structure
- `[b]` Chain-of-thought prompting
- `[~]` RICE / COSTAR frameworks
- `[~]` Few-shot prompting
- `[ ]` System prompt design
- `[ ]` Output format control (JSON mode)

### Semantic Kernel (.NET)
- `[ ]` Kernel setup — plugins, connectors, AI service registration
- `[ ]` Native functions — C# methods exposed as AI-callable tools (`[KernelFunction]`)
- `[ ]` Prompt functions — text templates with `{{$input}}` variables
- `[ ]` Memory + Embeddings — `ISemanticTextMemory`, `IMemoryStore`, similarity search
- `[ ]` Planner — auto-invokes plugins to fulfill a goal (Handlebars planner)
- `[ ]` Filters — `IFunctionInvocationFilter` for logging, safety checks, cost guards

### LLM Evaluation
- `[ ]` RAGAS metrics — context recall, context precision, faithfulness, answer relevancy
- `[ ]` Evals as unit tests — deterministic evals for critical RAG flows; test data sets
- `[ ]` Hallucination detection — self-consistency checks, source attribution, grounding
- `[ ]` Latency vs quality trade-offs — smaller models for routing, larger for generation

### AI Safety & Guardrails
- `[ ]` Prompt injection — direct (user input) vs indirect (retrieved content); defense patterns
- `[ ]` Input/output validation — reject harmful queries before sending to LLM
- `[ ]` Cost management — token counting, prompt caching (Anthropic/OpenAI), model routing by task complexity
- `[ ]` Responsible AI principles — transparency, fairness, accountability, human oversight

---

## MY STACK (Deep Dives)

### C#
- `[c]` OOP (classes, interfaces, inheritance, polymorphism)
- `[c]` async/await, Task
- `[c]` LINQ
- `[b]` Generics
- `[b]` Delegates, events, Func/Action
- `[b]` IDisposable, using pattern
- `[~]` Span<T>, Memory<T>
- `[ ]` Source generators

### C# Modern Syntax (C# 12+ / .NET 8+)
- `[ ]` Primary constructors — `class Foo(IService svc)` — used in services, handlers, repos daily
- `[ ]` Collection expressions — `[1, 2, 3]` for arrays, lists, spans; spread operator `..`
- `[ ]` `using` declarations — `using var x = ...` — scoped disposal without nested block
- `[ ]` `TimeProvider` (.NET 8) — abstraction over `DateTime.UtcNow`; inject in services, use `FakeTimeProvider` in tests
- `[ ]` `FrozenDictionary` / `FrozenSet` (.NET 8) — immutable, faster lookup; use for static lookup data
- `[ ]` `required` members (C# 11) — `required string Name { get; init; }` — compiler-enforced initialization

### C# Nullable Reference Types
- `[ ]` Nullable value types — int?, HasValue
- `[ ]` Nullable reference types — #nullable enable, compiler tracking
- `[ ]` Null operators — ?., ??, ??=, !
- `[ ]` Null check patterns — is null, is not null
- `[ ]` required members
- `[ ]` ArgumentNullException.ThrowIfNull

### C# Exception Handling
- `[ ]` throw vs Result<T> — the decision rule
- `[ ]` try/catch best practices — specific exceptions, throw vs throw ex
- `[ ]` Global exception handler — IExceptionHandler (.NET 8)
- `[ ]` ProblemDetails — RFC 7807

### C# DI Deep Dive
- `[ ]` Captive dependency — what it is, why it breaks
- `[ ]` IServiceScopeFactory — creating scopes in Singletons
- `[ ]` Service Locator anti-pattern
- `[ ]` Options pattern — IOptions vs IOptionsSnapshot vs IOptionsMonitor
- `[ ]` Keyed services (.NET 8)

### C# Background Jobs
- `[ ]` IHostedService / BackgroundService — lifecycle
- `[ ]` IServiceScopeFactory in background services
- `[ ]` Channel<T> — producer/consumer pattern
- `[ ]` Graceful shutdown
- `[ ]` Hangfire — fire-and-forget, scheduled, recurring

### C# Concurrency & Thread Safety
- `[ ]` Thread vs Task — cost and use cases
- `[ ]` Race conditions — recognizing and fixing
- `[ ]` lock / Monitor — rules and limits
- `[ ]` SemaphoreSlim — async critical sections
- `[ ]` Interlocked — atomic operations
- `[ ]` Thread-safe collections — ConcurrentDictionary gotchas
- `[ ]` async/await pitfalls — .Result deadlock, async void
- `[ ]` ConfigureAwait(false) — when needed
- `[ ]` CancellationToken propagation

### C# Memory Management
- `[ ]` Value types vs reference types — stack vs heap
- `[ ]` Boxing and unboxing — where it happens, impact
- `[ ]` GC generations (Gen0/1/2) and LOH
- `[b]` IDisposable — using statement, Dispose pattern
- `[~]` Span<T>, Memory<T> — zero-allocation slices
- `[ ]` ArrayPool<T> — renting buffers
- `[ ]` String interning and StringBuilder
- `[ ]` WeakReference

### C# Performance & Diagnostics
- `[ ]` BenchmarkDotNet — `[Benchmark]`, `[MemoryDiagnoser]`; reading Mean, Allocated, Gen0/1/2 columns
- `[ ]` dotTrace CPU — Timeline vs Sampling mode; identifying hot paths, lock contention, GC pauses
- `[ ]` dotMemory — heap snapshots, retained vs shallow size, finding memory leaks via dominators view
- `[ ]` PerfView — GC events, CPU stacks, allocation sampling; production-safe because it's event-based
- `[ ]` Expression trees — `Expression<Func<T,bool>>`; build + compile lambdas at runtime; how EF Core uses them
- `[ ]` Source generators — `IIncrementalGenerator`, reading syntax trees, emitting code at compile time
- `[ ]` `System.IO.Pipelines` — `PipeReader`/`PipeWriter`, backpressure, memory reuse, zero-copy parsing
- `[ ]` Native AOT (.NET 8+) — no JIT, trim analysis, startup + size wins; what breaks (reflection, dynamic types)
- `[ ]` `IMemoryOwner<T>` / `MemoryPool<T>` — renting memory, returning it; avoiding GC pressure in hot paths

### TypeScript / JavaScript
- `[c]` Types, interfaces, generics
- `[c]` async/await, Promises
- `[b]` Closures
- `[b]` Event loop, call stack, microtask queue
- `[b]` Hoisting
- `[~]` Currying, partial application
- `[~]` Prototype chain
- `[ ]` Promise.all vs allSettled vs race vs any — when each is right
- `[ ]` `using` declarations (TS 5.2 / ES2025) — `await using conn = ...` — explicit resource management

### TypeScript Advanced Types
- `[ ]` Utility types (Partial, Required, Pick, Omit, Record, Exclude, Extract, NonNullable, ReturnType)
- `[ ]` Discriminated unions + exhaustive narrowing with never
- `[ ]` Type guards (typeof, instanceof, in, user-defined is)
- `[ ]` Mapped types
- `[ ]` Conditional types + infer
- `[ ]` Template literal types
- `[ ]` satisfies operator
- `[ ]` Generic constraints

### ASP.NET Core
- `[c]` Controllers, routing, model binding
- `[b]` Middleware pipeline
- `[b]` Middleware pipeline order — exception → HTTPS → CORS → rate → auth → authz → compress → route
- `[b]` Dependency Injection (lifetimes: Singleton/Scoped/Transient)
- `[b]` Filters (action, exception, validation)
- `[b]` Configuration (appsettings, environment)
- `[~]` Minimal APIs
- `[ ]` SignalR
- `[ ]` OpenAPI / Swagger — `Microsoft.AspNetCore.OpenApi` (.NET 9 built-in, no Swashbuckle), document endpoints, describe errors
- `[ ]` Middleware vs Filter vs IEndpointFilter — the decision matrix (scope: all requests vs post-routing vs per-endpoint)
- `[ ]` API versioning (Asp.Versioning.Http)
- `[ ]` Pagination — PagedResult<T>, offset vs keyset
- `[ ]` Rate limiting — built-in .NET 8 + Redis distributed
- `[ ]` Output caching — .NET 8 built-in
- `[ ]` IMemoryCache + IDistributedCache usage pattern
- `[ ]` HybridCache (.NET 9) — one API for local + distributed tier, stampede protection, replaces IMemoryCache + IDistributedCache boilerplate
- `[ ]` ETag + Cache-Control headers
- `[ ]` Idempotency key filter pattern
- `[ ]` IFormFile upload + FileStreamResult download
- `[ ]` WebApplicationFactory — API layer integration tests
- `[ ]` Channel<T> — in-process producer/consumer queue
- `[ ]` Hangfire — persistent scheduled jobs
- `[ ]` Graceful shutdown — HostOptions + CancellationToken
- `[ ]` Response compression — gzip + brotli
- `[ ]` Role-based + policy-based authorization
- `[ ]` Options pattern — IOptions vs IOptionsSnapshot vs IOptionsMonitor
- `[ ]` ValidateOnStart — fail fast on bad config
- `[ ]` IHttpClientFactory — typed + named clients, socket exhaustion
- `[ ]` Polly + IHttpClientFactory wiring
- `[ ]` IAsyncEnumerable<T> streaming large responses
- `[ ]` 202 Accepted + job polling pattern (long-running ops)
- `[ ]` Webhooks — send + receive + HMAC signature verification
- `[ ]` PATCH vs PUT semantics
- `[ ]` System.Text.Json configuration options
- `[ ]` Feature flags — Microsoft.FeatureManagement
- `[ ]` OWASP API Security Top 10 (different from OWASP Web Top 10)
- `[ ]` Minimal APIs + IEndpointFilter
- `[ ]` Content negotiation + model binding sources
- `[ ]` ValidationProblemDetails vs ProblemDetails
- `[ ]` Multi-tenancy strategies + Global Query Filter
- `[ ]` Connection pooling + DbContext pooling
- `[ ]` Keyed services (.NET 8)
- `[ ]` Decorator pattern in DI (Scrutor)
- `[ ]` DateTime vs DateTimeOffset
- `[ ]` Specification pattern (complex query encapsulation)
- `[ ]` Dapper alongside EF (performance-critical reads)
- `[ ]` EF connection resilience — EnableRetryOnFailure
- `[ ]` Zero-downtime migrations — expand-contract pattern
- `[ ]` Cross-field + async FluentValidation
- `[ ]` ValidationProblemDetails shape
- `[ ]` .NET Aspire — cloud-native orchestration: service discovery, telemetry, resource wiring, local dev dashboard

### Blazor
- `[ ]` Blazor Server vs WASM vs Hybrid — rendering model, latency, bundle size trade-offs
- `[ ]` Component model — `@code`, parameters, cascading values, `EventCallback`
- `[ ]` Data binding — one-way (`@value`) vs two-way (`@bind`)
- `[ ]` Lifecycle hooks — `OnInitializedAsync`, `OnParametersSetAsync`, `StateHasChanged`
- `[ ]` JS interop — `IJSRuntime`; calling JS from C# and C# from JS
- `[ ]` When Blazor over React — internal tools, .NET-only teams, sharing domain logic between UI and API

### React
- `[c]` JSX, components, props
- `[c]` useState, useEffect, useRef
- `[c]` Custom hooks
- `[b]` Context API
- `[b]` Forms (controlled vs uncontrolled)
- `[~]` Performance (memo, lazy, Suspense)

### React Router v6/v7
- `[ ]` `createBrowserRouter` — data router API; replaces `<BrowserRouter>` + JSX route config
- `[ ]` Loaders — `loader` function fetches data before render; `useLoaderData()` to consume
- `[ ]` Actions — `action` function handles form POST; `useActionData()` for result
- `[ ]` Nested routes — `<Outlet>` for layout composition; shared layouts with child routes
- `[ ]` Error boundaries per route — `errorElement` prop; `useRouteError()`
- `[ ]` `<Form>` component — progressive enhancement; works without JS
- `[ ]` `useNavigate`, `useParams`, `useSearchParams`, `useLocation`
- `[ ]` Protected routes — wrapper component that checks auth and redirects
- `[ ]` Lazy routes — `lazy: () => import('./Page')` for code splitting per route
- `[ ]` TanStack Router — fully type-safe routing; `createRoute`, `Link` with typed params; alternative to React Router
- `[ ]` React Router vs TanStack Router vs Next.js App Router — trade-offs and when to choose

### React 19
- `[ ]` React Compiler — auto-memoizes; when manual `memo`/`useMemo`/`useCallback` is still needed
- `[ ]` `use()` hook — reads promises and context; replaces some `useEffect` data-fetch patterns
- `[ ]` `useOptimistic` — built-in optimistic updates (was experimental in React 18)
- `[ ]` `useActionState` — form state + async action in one hook; replaces most `useState` form boilerplate
- `[ ]` Server Actions (Next.js / React 19) — form submissions directly to server functions

### React Performance
- `[ ]` When React re-renders — the four triggers
- `[ ]` React.memo — when it helps vs when it fails
- `[ ]` useMemo — expensive values, when NOT to use
- `[ ]` useCallback — stable references, the memo/useCallback/useMemo trio
- `[ ]` Stale closures in useEffect
- `[ ]` React.lazy + Suspense — code splitting
- `[ ]` React Profiler — reading results

### React Forms
- `[ ]` React Hook Form — register, handleSubmit, formState
- `[ ]` Zod schemas — string/number/object, refine, coerce
- `[ ]` z.infer — type from schema
- `[ ]` setError — mapping server errors to fields
- `[ ]` useController — custom input components
- `[ ]` useFieldArray — dynamic lists

### TypeScript with React
- `[ ]` Typing props — interface, optional, literal unions
- `[ ]` Event types — ChangeEvent, FormEvent, MouseEvent
- `[ ]` Typing useState / useRef / useReducer
- `[ ]` Typing Context — undefined default + custom hook
- `[ ]` Generic components
- `[ ]` forwardRef typing

### React Ecosystem Libraries
- `[ ]` Radix UI — unstyled, accessible headless components (Dialog, Popover, Select, Tooltip); bring your own styles
- `[ ]` shadcn/ui — Radix + Tailwind prebuilt; copy-paste components into your project, not a dependency
- `[ ]` Headless UI (Tailwind Labs) — alternative headless library; Listbox, Combobox, Disclosure
- `[ ]` Framer Motion — declarative animation; `motion.div`, `AnimatePresence`, layout animations, gestures
- `[ ]` React Spring — physics-based animations; spring model; better for interactive/gesture-driven animations
- `[ ]` TanStack Table (v8) — headless table; sorting, filtering, pagination, grouping; bring your own UI
- `[ ]` TanStack Virtual — virtual scrolling for large lists/grids without rendering all rows in DOM
- `[ ]` dnd-kit — drag and drop; sortable lists, multiple containers; accessibility-first
- `[ ]` React Hot Toast / Sonner — toast notifications; simple API, accessible
- `[ ]` React Hook Form (already tracked under React Forms)
- `[ ]` Storybook — component workshop; `stories` per component; visual regression with Chromatic
- `[ ]` Vitest — Jest-compatible unit test runner built on Vite; faster than Jest in Vite projects
- `[ ]` Playwright — E2E browser testing; multi-browser, trace viewer, CI screenshots
- `[ ]` Cypress — E2E + component testing; time-travel debugger; good for visual component tests
- `[ ]` MSW (Mock Service Worker) — intercepts fetch/XHR at Service Worker level; same mocks in tests + dev
- `[ ]` date-fns / Day.js — date manipulation; tree-shakable (date-fns) vs chainable (Day.js); alternatives to Moment
- `[ ]` Zod (already tracked under React Forms)

### React Native
- `[b]` Core components (View, Text, ScrollView, FlatList, SectionList)
- `[b]` StyleSheet — style objects, platform-specific values
- `[b]` Navigation (React Navigation) — Stack, Tab, Drawer navigators; deep linking
- `[~]` WatermelonDB (offline-first)
- `[ ]` Expo vs bare workflow — Expo SDK modules vs ejected; when to eject
- `[ ]` Expo Router — file-based routing for RN; same conventions as Next.js App Router
- `[ ]` EAS Build — cloud builds for iOS + Android without local Xcode/Android Studio
- `[ ]` EAS Submit — automated app store submission
- `[ ]` EAS Update (OTA) — over-the-air JS bundle updates without app store review
- `[ ]` Platform-specific code — `Platform.OS`, `Platform.select()`, `.ios.tsx` / `.android.tsx` file suffixes
- `[ ]` React Native Reanimated v3 — JS-thread-free animations; `useSharedValue`, `useAnimatedStyle`, `withSpring`
- `[ ]` React Native Gesture Handler — native gesture recognition; replaces `TouchableOpacity` for complex gestures
- `[ ]` Hermes JS engine — bytecode compilation at build time; faster startup, lower memory
- `[ ]` Metro bundler — RN's bundler; configuration, aliases, custom transformers
- `[ ]` MMKV — fast key-value storage (replaces AsyncStorage for simple data; 30× faster)
- `[ ]` Push notifications — Expo Notifications + Firebase Cloud Messaging (FCM)
- `[ ]` Linking / deep links — `expo-linking`, universal links (iOS), app links (Android)
- `[ ]` React Native Paper / NativeWind / Tamagui — UI libraries for RN; styling trade-offs
- `[ ]` Testing — Jest + React Native Testing Library; mocking native modules
- `[ ]` Performance profiling — Flipper, React DevTools standalone, Hermes profiler

### Next.js
- `[ ]` SSR vs SSG vs ISR vs CSR — trade-offs and use cases; when each is the right default
- `[ ]` App Router — file-system conventions: `layout.tsx`, `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`
- `[ ]` Server Components vs Client Components — rules: no hooks/events in SC; `"use client"` boundary
- `[ ]` Data fetching in Server Components — `fetch()` with Next.js cache extensions (`{ cache: 'force-cache' }`)
- `[ ]` Next.js caching model — 4 layers: Request Memoization, Data Cache, Full Route Cache, Router Cache
- `[ ]` `revalidatePath` and `revalidateTag` — on-demand cache invalidation from Server Actions or Route Handlers
- `[ ]` `unstable_cache` — cache arbitrary async functions with tags; Next.js 14+
- `[ ]` Streaming SSR — `loading.tsx` as Suspense boundary; `<Suspense>` wrapping slow data components
- `[ ]` Parallel routes — `@slot` folders; show multiple pages simultaneously in one layout
- `[ ]` Intercepting routes — `(.)` / `(..)` prefix; modal pattern (show photo in modal, full page on direct nav)
- `[ ]` Route Handlers — `route.ts`; GET/POST handlers replacing `pages/api`; replaces Express for simple APIs
- `[ ]` Server Actions — `"use server"` functions; form POST without writing an API route
- `[ ]` `next/image` — automatic WebP/AVIF conversion, lazy loading, layout shift prevention, remote domains config
- `[ ]` `next/font` — self-hosted Google Fonts + local fonts; zero layout shift, no external request
- `[ ]` Metadata API — `export const metadata` in `layout.tsx` / `page.tsx`; dynamic with `generateMetadata()`
- `[ ]` `generateStaticParams` — pre-render dynamic routes at build time (`[id]` → known IDs)
- `[ ]` Middleware — `middleware.ts` at root; runs on Edge runtime; auth checks, redirects, locale detection
- `[ ]` Next.js 15 changes — async `cookies()` / `headers()` / `params`; PPR (Partial Pre-Rendering); Turbopack stable
- `[ ]` Next.js vs Vite + React — when each is right; Next.js overhead vs Vite simplicity

### Web Platform APIs
- `[ ]` Service Workers — install/activate/fetch lifecycle; cache-first vs network-first vs stale-while-revalidate
- `[ ]` PWA — `manifest.json`, install prompt, push notifications, background sync; what makes a site installable
- `[ ]` IndexedDB — client-side structured storage; Dexie.js as ergonomic wrapper
- `[ ]` WebSockets — full-duplex; when vs SSE (server-push only) vs long-polling
- `[ ]` Web Workers — CPU work off main thread; `Comlink` for structured API over `postMessage`
- `[ ]` Intersection Observer — lazy loading images, infinite scroll, analytics visibility tracking
- `[ ]` Core Web Vitals — LCP (largest paint), CLS (layout shift), INP (interaction delay); what hurts each metric
- `[ ]` Web performance tooling — Lighthouse CI, Chrome DevTools Performance tab, WebPageTest waterfall

### CSS Architecture
- `[ ]` Cascade, specificity, inheritance — three mechanisms; why `!important` is a code smell
- `[ ]` Flexbox — main axis, cross axis, `flex-grow`/`shrink`/`basis`; common one-liner centering patterns
- `[ ]` CSS Grid — template areas, auto-placement, subgrid; when Grid over Flexbox
- `[ ]` CSS custom properties — theming system, scoped variables, JS interop via `getPropertyValue`
- `[ ]` CSS Modules — scoped class names, `composes` keyword, zero runtime
- `[ ]` Tailwind CSS — utility-first, purging, arbitrary values `[]`, responsive `sm:`/`md:` prefixes, dark mode
- `[ ]` BEM naming — block__element--modifier; when useful, when it fights the component model
- `[ ]` Responsive design — media queries, container queries (`@container`), fluid typography (`clamp()`)
- `[ ]` CSS animations — transitions vs `@keyframes` vs Web Animations API vs Framer Motion; choosing the right tool

### Accessibility (a11y)
- `[ ]` WCAG 2.1 / 2.2 — A / AA / AAA levels; AA is the legal minimum in most jurisdictions
- `[ ]` Semantic HTML — landmark regions (`main`, `nav`, `aside`), heading hierarchy, `<button>` vs `<div>`
- `[ ]` ARIA roles and attributes — `aria-label`, `aria-describedby`, `role`; when to add vs when semantic HTML is enough
- `[ ]` Keyboard navigation — focus order (`tabindex`), focus traps in modals, skip-to-content links
- `[ ]` Screen reader testing — NVDA + Chrome (Windows), VoiceOver + Safari (Mac/iOS) — basic smoke testing
- `[ ]` Color contrast — 4.5:1 for normal text, 3:1 for large text; tools: axe DevTools, Colour Contrast Analyser

### Internationalization (i18n)
- `[ ]` react-i18next — `useTranslation`, namespaces, `Trans` component for JSX content with markup
- `[ ]` ICU message format — `{count, plural, one{# item} other{# items}}`; select for gender/variants
- `[ ]` RTL support — `dir="rtl"` on `<html>`; CSS logical properties (`margin-inline-start` vs `margin-left`)
- `[ ]` `Intl` API — `Intl.DateTimeFormat`, `Intl.NumberFormat`, `Intl.RelativeTimeFormat`; browser-native, no library needed
- `[ ]` Locale detection — `Accept-Language` header, URL prefix (`/en/`, `/bg/`), cookie persistence strategy

### Developer Tooling & Workflow
- `[ ]` Monorepos — Turborepo (task graph, remote cache) vs Nx (generators, affected commands); when vs polyrepo
- `[ ]` npm workspaces / pnpm workspaces — shared node_modules, inter-package dependencies
- `[ ]` ESLint config — `flat config` (v9+), rule severity, plugin ecosystem, `@typescript-eslint`, `eslint-plugin-react`
- `[ ]` Prettier — opinionated formatter; `.prettierrc`; integrating with ESLint (no conflicts)
- `[ ]` Husky + lint-staged — pre-commit hooks that only lint staged files; fast CI-equivalent checks locally
- `[ ]` Conventional Commits — `feat:`, `fix:`, `chore:`, `BREAKING CHANGE:`; machine-readable changelog
- `[ ]` Semantic versioning — MAJOR.MINOR.PATCH; when to bump each; `semantic-release` for automation
- `[ ]` `.env` management — `dotenv`, `dotenv-vault`, Zod for runtime env validation (`z.object({ PORT: z.coerce.number() })`)
- `[ ]` Build tools landscape — Vite (dev + build), esbuild (speed), Rollup (libraries), Webpack (legacy); when each
- `[ ]` Debugging in VSCode — launch configs, breakpoints, conditional breakpoints, logpoints, attach to Node/browser
- `[ ]` Debugging .NET in Rider/VSCode — remote debug, hot reload, exception settings, memory dumps
- `[ ]` Chrome DevTools mastery — Network waterfall, Performance flame graph, Memory heap snapshot, Sources breakpoints
- `[ ]` AI-assisted development — Copilot in editor, Cursor agentic edits, when to trust vs when to verify output
