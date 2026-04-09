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
