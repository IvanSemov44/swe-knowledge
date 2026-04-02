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
- `[ ]` Trees (binary tree, BST)
- `[ ]` Heaps (min/max)
- `[ ]` Tries
- `[ ]` Graphs (adjacency list vs matrix)

### Algorithms
- `[~]` Binary search
- `[~]` Bubble / Selection / Insertion sort
- `[~]` Merge sort
- `[~]` Quick sort
- `[ ]` BFS
- `[ ]` DFS
- `[ ]` Two pointers
- `[ ]` Sliding window
- `[ ]` Prefix sums
- `[ ]` Recursion fundamentals

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
- `[~]` N+1 query problem

### Git
- `[c]` Add, commit, push, pull
- `[c]` Branching, merging
- `[b]` Rebase vs merge
- `[b]` PR workflow
- `[~]` Cherry-pick, stash
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
- `[~]` Owned entities, value converters
- `[ ]` Interceptors

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

### CQRS + MediatR
- `[b]` Command vs Query separation
- `[b]` MediatR pipeline behaviors
- `[b]` Validation pipeline behavior
- `[~]` Event sourcing basics
- `[ ]` Outbox pattern

### React Internals
- `[c]` Components, props, state
- `[c]` useEffect, useMemo, useCallback
- `[b]` Reconciliation algorithm
- `[b]` Virtual DOM
- `[b]` Rendering optimization (React.memo)
- `[~]` Concurrent features (Suspense, transitions)

### RTK Query
- `[c]` injectEndpoints, baseApi
- `[c]` Cache tags, invalidation
- `[b]` Optimistic updates
- `[b]` Polling
- `[~]` Streaming / subscriptions

### Docker
- `[b]` Dockerfile basics
- `[b]` docker-compose
- `[~]` Multi-stage builds
- `[ ]` Networking, volumes
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
- `[ ]` URL shortener
- `[ ]` Rate limiter
- `[ ]` News feed / timeline
- `[ ]` Notification system
- `[ ]` File storage system
- `[ ]` Chat system

### Caching
- `[~]` Cache-aside pattern
- `[~]` Write-through / write-behind
- `[ ]` Redis data structures
- `[ ]` Cache eviction policies
- `[ ]` Distributed cache consistency

### Message Queues
- `[~]` RabbitMQ basics (exchange, queue, binding)
- `[~]` Producer/consumer pattern
- `[ ]` Dead letter queues
- `[ ]` Outbox pattern + reliable messaging
- `[ ]` At-least-once vs exactly-once delivery

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
- `[~]` Build, test, deploy pipeline
- `[ ]` Docker build in CI
- `[ ]` Environment secrets management
- `[ ]` Blue/green deployment

---

## AI INTEGRATION

### LLM API Basics
- `[b]` OpenAI / Anthropic API structure
- `[~]` Calling LLM from .NET (HttpClient / SDK)
- `[~]` Calling LLM from React (fetch / SDK)
- `[ ]` Streaming responses
- `[ ]` Token counting, cost estimation

### RAG
- `[ ]` Chunking strategies
- `[ ]` Embeddings (what they are, how to generate)
- `[ ]` Vector stores (pgvector, Qdrant)
- `[ ]` Similarity search (cosine, dot product)
- `[ ]` Retrieval pipeline end-to-end
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
