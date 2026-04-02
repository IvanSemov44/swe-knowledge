# Personal Knowledge Repository

A structured collection of software engineering knowledge, organized by level and topic.

## Purpose

1. Complete SWE learning path from junior to senior
2. Confidence checklist per topic (updated after each study session)
3. `CLAUDE.md` — AI context file to paste at the start of any Claude/Copilot/Cursor session
4. Living document that grows as I learn

## How to Use

- **Starting a new AI session?** Paste `CLAUDE.md` first — the AI will know your full context immediately
- **Checking what to study next?** Open `PROGRESS.md` — find topics marked `[ ]` or `[~]`
- **Deep diving a topic?** Open the relevant file in `/junior`, `/mid`, `/senior`, or `/ai-integration`
- **After a study session?** Update the confidence checkboxes in `PROGRESS.md`

## Structure

```
knowledge-repo/
  CLAUDE.md          ← AI context file (paste this at session start)
  README.md          ← this file
  PROGRESS.md        ← master checklist with confidence levels

  junior/
    algorithms.md
    data-structures.md
    big-o.md
    http-rest.md
    sql-basics.md
    git.md

  mid/
    design-patterns.md
    solid.md
    authentication.md
    ef-core-advanced.md
    clean-architecture.md
    ddd.md
    cqrs.md
    react-internals.md
    rtk-query.md
    docker.md
    testing.md

  senior/
    system-design.md
    caching.md
    message-queues.md
    advanced-algorithms.md
    microservices.md
    ci-cd.md

  ai-integration/
    llm-api-basics.md
    rag.md
    agentic-patterns.md
    prompt-engineering.md

  my-stack/
    csharp.md
    typescript-js.md
    aspnet.md
    react.md
    react-native.md
```

## Confidence Scale

| Symbol | Meaning |
|---|---|
| `[ ]` | Never studied |
| `[~]` | Studied, mostly forgotten |
| `[b]` | Know the basics |
| `[c]` | Comfortable |
| `[x]` | Can explain and implement from memory |
