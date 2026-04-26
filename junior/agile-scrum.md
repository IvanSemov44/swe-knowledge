# Agile & Scrum

## What it is
Agile is a set of values and principles for iterative software development. Scrum is the most popular Agile framework — a structured process with specific roles, ceremonies, and artifacts.

## Why it matters
Every company asks "have you worked in Agile/Scrum?" in behavioral interviews. Even if you haven't, you need to know the vocabulary and concepts. Most dev teams operate some version of Scrum.

---

## Agile Manifesto — The Core Values

```
Individuals and interactions   over   processes and tools
Working software               over   comprehensive documentation
Customer collaboration         over   contract negotiation
Responding to change           over   following a plan
```

The items on the right have value — the items on the LEFT are valued MORE.

**In practice:** Agile means: ship small slices frequently, get feedback, adapt. Don't plan for 6 months, build for 6 months, then show users.

---

## Scrum Roles

| Role | Responsibility |
|---|---|
| **Product Owner (PO)** | Owns the product backlog. Decides WHAT gets built and in what order. Represents the business/users. |
| **Scrum Master (SM)** | Facilitates the process. Removes blockers. Coaches the team on Scrum. NOT a manager. |
| **Development Team** | Builds the product. Cross-functional (dev, QA, design). Self-organizing. Typically 3–9 people. |

---

## Scrum Artifacts

| Artifact | Description |
|---|---|
| **Product Backlog** | Ordered list of everything the product needs. Owned by the PO. Items are user stories, bugs, technical tasks. |
| **Sprint Backlog** | The subset of backlog items the team commits to in the current sprint. |
| **Increment** | The sum of all completed work — must be potentially shippable at every sprint end. |

---

## Sprint

A fixed-length iteration, typically **1–4 weeks** (2 weeks is most common). Each sprint produces a potentially shippable increment.

```
Sprint 1 (2 weeks)
  ├── Sprint Planning    (start: what do we commit to?)
  ├── Daily Standups     (every day: what did I do, what will I do, blockers?)
  ├── Development Work
  ├── Sprint Review      (end: demo to stakeholders)
  └── Sprint Retrospective (end: how do we improve?)

Sprint 2 (2 weeks) → repeat
```

---

## The Five Ceremonies

### 1. Sprint Planning
- **When:** Start of each sprint
- **Who:** Whole team + PO
- **What:** PO presents top backlog items → team discusses and commits to what they can complete → sprint goal agreed
- **Output:** Sprint backlog

### 2. Daily Standup (Daily Scrum)
- **When:** Every day, same time, ≤ 15 minutes
- **Who:** Development team (SM optional, PO optional)
- **Three questions:**
  1. What did I do yesterday?
  2. What will I do today?
  3. Any blockers?
- **Purpose:** Synchronize and surface impediments — NOT a status report to a manager

### 3. Sprint Review (Demo)
- **When:** End of sprint
- **Who:** Team + PO + stakeholders
- **What:** Demo the completed increment to stakeholders. Get feedback. PO accepts or rejects items.
- **Output:** Updated product backlog based on feedback

### 4. Sprint Retrospective
- **When:** After sprint review, before next sprint planning
- **Who:** Team + SM (PO sometimes)
- **Three questions:**
  1. What went well?
  2. What didn't go well?
  3. What do we commit to improving next sprint?
- **Purpose:** Continuous process improvement

### 5. Backlog Refinement (Grooming)
- **When:** Mid-sprint, ongoing
- **Who:** Team + PO
- **What:** Break down large backlog items, add acceptance criteria, estimate upcoming items
- **Not an official Scrum ceremony** but universally practiced

---

## User Stories

The standard format for describing features from the user's perspective.

```
As a [type of user],
I want [some goal],
So that [some reason/value].

Example:
"As a customer,
I want to save products to a wishlist,
So that I can find them easily when I'm ready to buy."
```

**Acceptance Criteria** — defines when the story is "done":
```
Given I am logged in
When I click "Save to Wishlist" on a product
Then the product appears in my wishlist
And I see a confirmation message
And the heart icon on the product turns red
```

---

## Story Points & Velocity

**Story points** estimate relative effort/complexity (not hours).

Common scale: Fibonacci (1, 2, 3, 5, 8, 13, 21) — gaps grow to reflect uncertainty.

```
1 point  = trivial (change a text label)
2 points = small (add a field to a form)
5 points = medium (new CRUD endpoint + UI)
8 points = large (complex feature with edge cases)
13 points = very large (should probably be split)
```

**Velocity** = story points completed per sprint. After 3–4 sprints, velocity stabilizes and can be used to forecast.

```
Average velocity: 40 points/sprint
Product backlog: 200 points
Forecast: ~5 sprints (10 weeks at 2-week sprints) to complete
```

**Estimation technique — Planning Poker:**
All team members reveal their estimate simultaneously (prevent anchoring). Discuss if there's a large spread. Re-estimate.

---

## Definition of Done (DoD)

A shared agreement on what "done" means for every story. Without it, "done" means different things to different people.

Example DoD:
```
✅ Code written and peer-reviewed (PR approved)
✅ Unit tests written and passing
✅ Integration tests passing
✅ No new linting errors
✅ Deployed to staging environment
✅ Acceptance criteria verified by PO
✅ Documentation updated (if relevant)
```

---

## Kanban vs Scrum

| | Scrum | Kanban |
|---|---|---|
| Iterations | Fixed sprints | Continuous flow |
| Roles | PO, SM, Dev Team | No prescribed roles |
| Ceremonies | Sprint planning, standups, review, retro | No required ceremonies |
| WIP limits | Per sprint (sprint backlog) | Explicit WIP limits per column |
| Best for | Product development with regular releases | Support/ops work, maintenance |
| Changes mid-cycle | No new items mid-sprint | Any time |

**Many teams use a hybrid** — Scrum ceremonies with a Kanban-style board.

---

## What Interviewers Actually Ask

**"Have you worked in Agile?"**
```
"Yes — in my personal project work I've applied Scrum principles: breaking work into
small user stories, prioritizing by value, iterating frequently rather than planning
everything upfront. I'm familiar with the Scrum ceremonies and have studied the framework
to prepare for working in a professional team."
```

**"What is a sprint retrospective?"**
```
"It's a ceremony at the end of each sprint where the team reflects on the process —
not the product. The goal is to identify what went well, what didn't, and to commit to
one or two concrete improvements for the next sprint. It's about continuous improvement
of how the team works together."
```

**"What's the difference between a Product Owner and a Scrum Master?"**
```
"The Product Owner is accountable for the product — they prioritize the backlog and
decide what gets built. The Scrum Master is accountable for the process — they facilitate
ceremonies, remove blockers, and coach the team on Scrum. The SM is not a manager;
they serve the team."
```

---

## Common Interview Questions

1. What is the difference between Scrum and Agile?
2. What are the three Scrum roles?
3. What happens in a sprint retrospective?
4. What is a user story? Give an example.
5. What is velocity?
6. What is the Definition of Done?
7. What is the difference between Scrum and Kanban?

---

## Common Mistakes

- Saying Agile and Scrum are the same thing (Agile is the philosophy, Scrum is one framework)
- Confusing Product Owner and Scrum Master responsibilities
- Thinking Daily Standup is a status report to management — it's for the team
- Thinking story points measure hours — they measure relative complexity

---

## How It Connects

- Sprint planning maps to how you'd break down features in your E-commerce project
- User stories + acceptance criteria = the requirements that drive your command/query design
- Definition of Done includes "deployed to staging" — connects to your Docker + CI/CD work
- Backlog refinement is when technical debt items get prioritized alongside new features

---

## My Confidence Level
- `[ ]` Agile values — the four core trade-offs
- `[ ]` Scrum roles — PO vs SM vs Dev Team
- `[ ]` Sprint — what it is, typical length
- `[ ]` The 5 ceremonies — purpose and output of each
- `[ ]` User stories — format and acceptance criteria
- `[ ]` Story points and velocity
- `[ ]` Definition of Done
- `[ ]` Scrum vs Kanban — when each fits

## My Notes
<!-- Personal notes -->
