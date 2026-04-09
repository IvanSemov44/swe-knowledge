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
```

---

<!-- AI: add entries below after each session -->

## 2026-04-09
**Topic:** Microservices — when NOT to use them
**What happened:** Got the benefits right but completely ignored the second half of the question — when not to split.
**Root cause:** Knows the sales pitch for microservices, not the tradeoffs. Missing: network failure, distributed transactions, operational overhead, team size requirement.
**Fix / key insight:** Microservices solve org/scale problems, not code problems. Start monolith, extract when you feel pain. Small team + microservices = overhead with no benefit.
**Revisit:** senior/microservices.md

## 2026-04-09
**Topic:** DSA — Two pointers
**What happened:** Only knew brute force O(n²). Correctly identified why two pointers needs sorted array after being taught.
**Root cause:** Never studied optimal patterns — jumped straight to nested loop thinking.
**Fix / key insight:** Sorted → two pointers O(n). Unsorted → hash set O(n). Sort-then-two-pointers is O(n log n) — valid but worse than hash set.
**Revisit:** senior/advanced-algorithms.md

## 2026-04-09
**Topic:** CI/CD pipeline order
**What happened:** Listed lint, tests, deploy, restart — right instincts but wrong order and missing build + Docker steps. Confused pre-commit hooks with CI steps.
**Root cause:** Knows pieces of the pipeline but not the full sequence or why the order matters.
**Fix / key insight:** Order is lint → type check → tests → build → docker build/push → deploy. Cheapest first, fail fast. Docker image tagged with commit SHA for traceability. CI runs lint too even if pre-commit hook exists — because hooks can be skipped.
**Revisit:** senior/ci-cd.md

## 2026-04-09
**Topic:** React — re-renders and React.memo
**What happened:** Named useCallback/useMemo but didn't answer "which children re-render." Didn't know when NOT to use React.memo.
**Root cause:** Knows the hooks exist but not the mental model: all children re-render by default, memo + useCallback work as a pair.
**Fix / key insight:** React.memo skips render on shallow prop equality. useCallback stabilizes function props so memo's comparison works. Don't use memo for cheap components or when props always change.
**Revisit:** mid/react-internals.md

## 2026-04-09
**Topic:** EF Core — N+1 problem
**What happened:** Described the concept correctly but used pseudocode. Missed that in EF Core N+1 is invisible — it looks like normal property access, not an explicit query call.
**Root cause:** Knows the pattern abstractly, hasn't seen it fail silently in real EF Core code.
**Fix / key insight:** EF lazy loading fires a query on every navigation property access inside a loop. The fix is `.Include()` for commands, `.Select()` projection for queries.
**Revisit:** mid/ef-core-advanced.md

## 2026-04-09
**Topic:** EF Core — Select vs Include
**What happened:** Said "select is for specific data, include is for whole entity" — correct but too vague.
**Root cause:** Didn't connect it to the CQRS pattern already in use.
**Fix / key insight:** Query handlers always use `.Select()` (read-only, DTO, no tracking). Command handlers use `.Include()` (need full aggregate to mutate). This is already the architecture — just apply it consciously.
**Revisit:** mid/ef-core-advanced.md, mid/cqrs.md
