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
- `[ ]` Tries
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
- `[~]` API versioning strategies

### SQL Basics
- `[c]` SELECT, WHERE, ORDER BY, GROUP BY
- `[c]` JOINs (INNER, LEFT, RIGHT, FULL)
- `[b]` Indexes (when and why)
- `[b]` Transactions (ACID)
- `[b]` Normalization (1NF/2NF/3NF)
- `[b]` N+1 query problem

### Git
- `[c]` Add, commit, push, pull
- `[c]` Branching, merging
- `[b]` Rebase vs merge
- `[b]` PR workflow
- `[b]` Cherry-pick, stash
- `[ ]` Advanced branching strategies (GitFlow, trunk-based)

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
- `[~]` OWASP top 10

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

### DDD
- `[b]` Entities vs Value Objects
- `[b]` Aggregates & aggregate roots
- `[b]` Domain Events
- `[b]` Repository per aggregate root
- `[b]` Unit of Work
- `[~]` Bounded contexts
- `[~]` Ubiquitous language
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
- `[b]` Dead Letter Queue — configuration and monitoring
- `[ ]` MassTransit consumer setup
- `[b]` Saga basics — state machine for long-running workflows
- `[ ]` RabbitMQ + Docker setup

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

### Testing
- `[b]` Unit testing with xUnit
- `[b]` Mocking with Moq
- `[~]` Integration testing with TestContainers
- `[~]` What to test and what not to test
- `[ ]` Frontend testing (React Testing Library)

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

### TypeScript / JavaScript
- `[c]` Types, interfaces, generics
- `[c]` async/await, Promises
- `[b]` Closures
- `[b]` Event loop, call stack, microtask queue
- `[b]` Hoisting
- `[~]` Currying, partial application
- `[~]` Prototype chain

### ASP.NET Core
- `[c]` Controllers, routing, model binding
- `[b]` Middleware pipeline
- `[b]` Dependency Injection (lifetimes: Singleton/Scoped/Transient)
- `[b]` Filters (action, exception, validation)
- `[b]` Configuration (appsettings, environment)
- `[~]` Minimal APIs
- `[ ]` SignalR

### React
- `[c]` JSX, components, props
- `[c]` useState, useEffect, useRef
- `[c]` Custom hooks
- `[b]` Context API
- `[b]` React Router
- `[b]` Forms (controlled vs uncontrolled)
- `[~]` Performance (memo, lazy, Suspense)

### React Native
- `[b]` Core components (View, Text, ScrollView, FlatList)
- `[b]` StyleSheet
- `[b]` Navigation (React Navigation)
- `[~]` WatermelonDB (offline-first)
- `[ ]` Native modules
- `[ ]` Performance profiling
