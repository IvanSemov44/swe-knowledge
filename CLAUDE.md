# CLAUDE.md — AI Context File for Ivan Semov

Paste this file at the start of any Claude / Copilot / Cursor session.
The AI will know your full context without re-explaining.

---

## Who I Am

- **Name:** Ivan Semov
- **Age / Location:** 32, Varna, Bulgaria
- **Current situation:** Night-shift casino dealer transitioning full-time into software development
- **Goal:** Junior-to-mid remote SWE role within several months
- **Learning style:** Test-then-teach. Give me a brief problem or question first, score my answer honestly (0–10), identify gaps, then teach. Do not pad scores.

---

## Education & Completed Courses (SoftUni)

| Course | Status |
|---|---|
| C# Fundamentals | ✅ Done |
| C# Advanced | ✅ Done |
| SQL | ✅ Done |
| Entity Framework Core | ✅ Done |
| ASP.NET Core | ✅ Done |
| JS Advanced | ✅ Done |
| React | ✅ Done |

---

## Active Projects

### 1. E-commerce App (Primary Portfolio Piece)

**Stack:**
- Frontend: React 18, TypeScript, Redux Toolkit, RTK Query, Vite
- Backend: .NET 8, ASP.NET Core, EF Core 8, SQL Server
- Architecture: Clean Architecture, DDD, CQRS + MediatR

**Key conventions:**
- Controllers are thin — return `ApiResponse<T>`, use `[ValidationFilter]`
- Services inject `IUnitOfWork`, not individual repositories
- Services return `Result<T>` for expected business outcomes
- Repositories never call `SaveChangesAsync` — UnitOfWork commits
- Use `ErrorCodes` constants (no magic strings) for business failures
- All backend async methods include `CancellationToken cancellationToken = default`
- Frontend API calls use RTK Query (`baseApi.injectEndpoints`), never raw fetch in components
- Frontend server state lives in RTK Query cache; slices manage UI-only state
- Clean Architecture dependency direction: API → Application → Core; Infrastructure → Core/Application

**Directory structure (backend):**
```
src/
  Api/           ← Controllers, Filters, Middleware
  Application/   ← Commands, Queries, DTOs, Interfaces, Services
  Core/          ← Entities, Value Objects, Domain Events, Aggregates
  Infrastructure/← EF Core, Repositories, UnitOfWork, external services
```

**Directory structure (frontend):**
```
src/
  app/           ← store, baseApi, router
  features/      ← feature slices + RTK Query endpoints
  components/    ← shared UI
  pages/         ← route-level components
```

---

### 2. CasinoTrainingApp (Strongest Business Opportunity)

**Stack:** React Native, TypeScript, WatermelonDB (offline-first), React Navigation

**Target:** B2B — casino operators for staff training

**Key focus:** Works fully offline; sync when online; native feel on iOS + Android

---

## My Stack & Versions

| Technology | Version | Level |
|---|---|---|
| C# / .NET | .NET 8 | Mid |
| ASP.NET Core | 8 | Mid |
| EF Core | 8 | Mid |
| SQL Server | 2022 | Mid |
| React | 18 | Mid |
| TypeScript | 5 | Mid |
| Redux Toolkit | 2 | Mid |
| RTK Query | 2 | Mid |
| React Native | 0.73 | Beginner |
| WatermelonDB | latest | Beginner |
| Docker | - | Beginner |
| Git | - | Comfortable |

---

## Architectural Conventions (Do Not Violate)

1. **Clean Architecture** — dependency arrows always point inward toward Core
2. **DDD** — Entities have identity; Value Objects are immutable and compared by value; Aggregates own consistency boundaries; Domain Events decouple side effects
3. **CQRS** — Commands mutate state and return `Result<T>`; Queries return read models and never mutate
4. **Repository pattern** — one repository per Aggregate root; no `SaveChangesAsync` inside repos
5. **Unit of Work** — single transaction per use-case; committed by the service/command handler
6. **Result pattern** — no throwing exceptions for expected business failures; use `Result<T>` with typed errors and `ErrorCodes`
7. **Validation** — input validation via `[ValidationFilter]` + FluentValidation on DTOs at the API boundary; domain invariants enforced inside the domain layer

---

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Commands | `VerbNounCommand` | `CreateOrderCommand` |
| Queries | `GetNounQuery` | `GetOrderByIdQuery` |
| DTOs | `NounDto` / `CreateNounRequest` | `OrderDto`, `CreateProductRequest` |
| Handlers | `VerbNounCommandHandler` | `CreateOrderCommandHandler` |
| Interfaces | `INoun` | `IOrderRepository` |
| EF entities | PascalCase, no prefix | `Order`, `OrderItem` |
| React components | PascalCase | `ProductCard` |
| RTK Query endpoints | camelCase verb | `getProducts`, `createOrder` |
| Redux slices | `nounSlice` | `cartSlice` |
| TS types/interfaces | PascalCase | `ProductDto`, `CartState` |

---

## Learning Goals & Timeline

**Current focus (next 3 months):**
1. Solidify system design fundamentals (caching, queues, scaling)
2. Add AI integration to E-commerce (RAG, LLM calls from .NET + React)
3. Finish CasinoTrainingApp MVP
4. Reach comfortable level in Docker + GitHub Actions CI/CD
5. Grind 2–3 LeetCode mediums per week (arrays, hashmaps, two-pointer, sliding window)

**Longer term (3–6 months):**
- Contribute to open source
- Deploy E-commerce publicly with CI/CD pipeline
- Add agentic features (tool use, planning, multi-agent) to one project
- Achieve mid-level confidence across the full stack

---

## How to Teach Me

1. **Test first** — give me a short problem or question on the topic
2. **Score honestly** — 0–10, no rounding up, identify the gap
3. **Teach the gap** — explain only what I missed; do not re-teach what I got right
4. **Give code examples** in my stack (C# / TypeScript / React) unless otherwise asked
5. **Connect concepts** — how does this relate to what I already know? (DDD → Repository, Clean Architecture → DI, etc.)
6. **Don't pad** — if my answer is 4/10, say 4/10

---

## My Confidence Level Per Topic (Self-Assessment)

Use this to calibrate explanation depth.

| Topic | Level |
|---|---|
| C# OOP | Comfortable |
| C# async/await | Comfortable |
| LINQ | Comfortable |
| ASP.NET Core pipeline | Know the basics |
| EF Core | Know the basics |
| Clean Architecture | Comfortable |
| DDD | Know the basics |
| CQRS + MediatR | Know the basics |
| REST API design | Comfortable |
| JWT / Auth | Know the basics |
| React hooks | Comfortable |
| Redux Toolkit | Comfortable |
| RTK Query | Comfortable |
| TypeScript | Comfortable |
| SQL (joins, indexes) | Know the basics |
| Algorithms / DSA | Studied, mostly forgotten |
| System Design | Never studied deeply |
| Docker | Know the basics |
| CI/CD | Beginner |
| React Native | Beginner |
| AI Integration / RAG | Beginner |
| Testing (unit/integration) | Know the basics |

---

## AI Tools I Use

- **Claude** (primary reasoning, architecture, code review)
- **GitHub Copilot** (in-editor autocomplete)
- **Cursor** (agentic code editing)

I want to learn how to build AI-powered features myself — RAG pipelines, LLM API calls from .NET and React, agentic patterns (Tool Use, Memory, Reflection, Multi-Agent).

---

## Useful Context for This Session

- Today's date: see session context
- Primary project path: `/home/user/E-commerce`
- This knowledge repo path: `/home/user/E-commerce/knowledge-repo`
- GitHub repo: `ivansemov44/e-commerce`
