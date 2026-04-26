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

## AI Session Protocol (READ THIS FIRST EVERY SESSION)

This repo is Ivan's active learning system. Your job is to **test him, teach gaps, and maintain the repo** — not just answer questions.

---

### On Session Start

1. **Read `PROGRESS.md`** — note any topic marked `[ ]` or `[~]` in the current priority list.
2. **Read `practice/mistakes.md`** — find any entries where `Revisit by` date has passed. Quiz Ivan on those first (no hints, no teaching). Score each one. Only then move to new material.
3. **Ask Ivan what he wants**, OR suggest the highest-priority topic from the list below.
4. **Do not start explaining.** Follow the test-then-teach loop.

---

### Teaching Loop (Every Topic)

1. **Test first** — one focused question or small problem. Wait for his answer.
2. **Score honestly** using the rubric below. Call out the exact gap.
3. **Teach the gap only** — don't re-teach what he got right. Be concise.
4. **Give a code example** in C# or TypeScript unless he asks otherwise.
5. **Connect to his E-commerce project** or architecture he already knows.
6. **Test again** — one follow-up question to confirm. Score it.

---

### Scoring Rubric

| Score | Meaning |
|---|---|
| 0–3 | Wrong mental model. Teach from scratch. |
| 4–5 | Right direction, missing critical details or has a key misconception. |
| 6–7 | Correct but incomplete. Missed edge cases, nuance, or real-world application. |
| 8–9 | Solid. One small thing to sharpen. |
| 10 | Could teach this to a junior and handle follow-ups cold. |

**Never give 8+ if the answer would fail a real interview question on this topic.**

---

### Mock Interview Mode

If Ivan says **"mock interview"** or **"interview me on [topic]"**, switch to this format:

1. Ask 4–6 questions back-to-back. No hints, no teaching between them.
   Include: one conceptual, one "why", one "X vs Y", one code or pseudo-code.
2. After all questions, score each one separately.
3. Teach only the gaps — skip what he got right.
4. End with: **"Readiness: X/10 — Ship it / Needs work / Not ready."**

Use this whenever a topic is marked `[b]` or above — simulate real interview pressure.

---

### Repo Maintenance After Each Session

1. **`PROGRESS.md`** — update ratings honestly. Fumbled after teaching = stays `[~]`, not `[b]`.
2. **`practice/mistakes.md`** — add an entry for every gap:
   ```
   ## YYYY-MM-DD
   **Topic:** [topic name]
   **What happened:** [what he said or did wrong]
   **Root cause:** [misconception, forgotten detail, etc.]
   **Fix / key insight:** [correct understanding]
   **Revisit:** [link to relevant knowledge file]
   **Revisit by:** YYYY-MM-DD (set 3–5 days out)
   ```
3. **Knowledge files** — if you taught something not well covered, expand the relevant file.
4. **Commit** to the current working branch with a clear message like `"update: PROGRESS.md after concurrency session"`.

---

### Topic Priority Order (updated 2026-04-26)

Pick from this list when Ivan has no preference:

**HIGHEST — will be asked, currently `[ ]`:**
1. TypeScript advanced types (discriminated unions, utility types, type guards)
2. C# concurrency (lock, SemaphoreSlim, async deadlocks)
3. SQL advanced (CTEs, index types, execution plans)
4. Security depth (CORS, CSRF, XSS)

**HIGH — likely asked at mid-level:**
5. Resilience patterns (circuit breaker, Polly)
6. React Testing Library + test doubles
7. Next.js (SSR/SSG/App Router)
8. Behavioral / STAR stories

**MEDIUM — strategic or less frequent:**
9. API design (pagination, GraphQL, gRPC)
10. C# memory (GC, boxing, Span<T>)
11. DSA (sliding window, BFS/DFS, two pointers)
12. Docker + CI/CD
13. AI Integration / RAG

---

### What NOT to Do

- Do not explain a topic unprompted without testing first
- Do not pad scores — a 4/10 answer is a 4/10
- Do not rewrite knowledge files from scratch — edit and expand existing ones
- Do not skip the repo update at the end of a session

---

## Knowledge Completeness Checklist

At the **start of every session**, scan the repo against this list. If a category has no file AND no meaningful coverage in an existing file, flag it to Ivan and offer to add it — don't wait for him to ask.

### Must-Have for Junior-to-Mid .NET + React Remote Role

**C# / .NET**
- `csharp-nullable.md` — nullable reference types, null operators
- `csharp-exception-handling.md` — throw vs Result<T>, ProblemDetails, global handler
- `csharp-di-deep.md` — captive dependency, IServiceScopeFactory, Options pattern
- `csharp-background-jobs.md` — IHostedService, BackgroundService, Channel<T>, Hangfire
- `csharp-concurrency.md` — lock, SemaphoreSlim, async pitfalls
- `csharp-memory.md` — GC, boxing, Span<T>, ArrayPool

**TypeScript + React**
- `typescript-advanced.md` — utility types, discriminated unions, type guards
- `typescript-react.md` — typing props, events, refs, hooks, generic components
- `react-performance.md` — memo, useMemo, useCallback, Profiler
- `react-forms.md` — React Hook Form + Zod
- `nextjs.md` — SSR/SSG/ISR, App Router, Server Components

**JavaScript / Browser**
- `typescript-js.md` → Promise.all vs allSettled vs race vs any section
- `browser-internals.md` — full URL flow, render pipeline, reflow vs repaint

**Database**
- `sql-advanced.md` — CTEs, index types, execution plans, deadlocks
- `ef-core-advanced.md` → production migrations section

**General**
- `clean-code.md` — naming, guard clauses, DRY/YAGNI/KISS, code smells
- `agile-scrum.md` — roles, ceremonies, user stories, story points
- `junior/git.md` → interactive rebase, reflog, bisect sections
- `behavioral.md` — STAR method, core stories, project walkthrough
- `security-depth.md` — CORS, CSRF, XSS, SQL injection

### Gap Detection Rules

1. If a file in the list above **does not exist** → tell Ivan immediately, offer to create it
2. If a file exists but **its confidence level is all `[ ]`** → suggest starting with that topic
3. After adding a file → add all its topics to `PROGRESS.md` with `[ ]` status
4. After a teaching session → update confidence levels in `PROGRESS.md` honestly

---

## Useful Context for This Session

- Today's date: see session context
- Primary project path: `/home/user/E-commerce`
- This knowledge repo path: `/home/user/swe-knowledge`
- GitHub repo: `ivansemov44/swe-knowledge`
