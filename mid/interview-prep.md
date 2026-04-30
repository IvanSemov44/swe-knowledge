# Interview Preparation by Level
<!-- level: Junior → Senior | interview formats, live coding strategy, system design framework, question banks, company research, negotiation -->

Everything behavioral (STAR stories, "tell me about yourself", conflict questions) lives in `behavioral.md`.
This file covers the **technical side of interviews** at each level.

---

## What They Actually Test at Each Level

| Level | Core question | Coding | Design | Behavioral weight |
|---|---|---|---|---|
| Junior | "Can you write code and will you learn?" | Easy — arrays, strings, basic DS | None or tiny component | Medium |
| Mid | "Can you own a feature and handle trade-offs?" | Medium — trees, graphs, hash maps | Component / single-service | Equal to technical |
| Senior | "Can you lead, handle ambiguity, design systems?" | Medium-hard + patterns | Full distributed systems | Heavy |

---

## Junior Level Interview

### What to expect
- Recruiter screen (30 min) — background, salary, availability, culture
- Technical phone/video screen (45 min) — basic coding, project walk, maybe one OOP question
- Sometimes a small take-home (CRUD API, one React component)
- Final round: 1-2 technical + culture/HR

### What they're really checking
1. Can you write code that works?
2. Do you understand the tools you say you know (don't lie on your CV)?
3. Are you coachable and collaborative?
4. Do you communicate when stuck, or go silent?

### Junior technical questions (.NET)

| Question | What they're testing |
|---|---|
| What is the difference between `abstract class` and `interface`? | OOP fundamentals |
| What is dependency injection and why use it? | DI understanding |
| What is the difference between `async void` and `async Task`? | Async basics |
| What does `using` do with an `IDisposable`? | Resource management |
| What HTTP status code for validation error? For "not found"? | REST basics |
| What is the difference between `GET` and `POST`? | HTTP verbs |
| What is a SQL `JOIN`? What is a `LEFT JOIN`? | SQL basics |
| What is Git rebasing? When would you use it over merge? | Git basics |
| What is `IEnumerable` vs `IQueryable`? | EF Core / LINQ basics |
| Walk me through your project — what does it do and why did you build it? | Communication |

### Junior coding problems (Easy LeetCode equivalent)
- Reverse a string / array
- Check if a string is a palindrome
- Find the most frequent element in an array
- FizzBuzz with a twist
- Two Sum (hash map approach)
- Valid parentheses (stack approach)

### Junior take-home tips
- A working app beats a perfectly architected incomplete one
- Add a `README.md` — explains how to run, what you'd add next
- Commit frequently with clear messages
- Don't use technologies you can't explain if asked about them

---

## Mid Level Interview

### What to expect
- Recruiter screen
- Technical phone screen — medium coding OR project walkthrough
- Live coding round OR take-home (feature implementation)
- Component/service design round — "design a caching layer for the API"
- Behavioral round (as heavy as technical)

### What they're really checking
1. Can you own a feature end-to-end without hand-holding?
2. Do you understand trade-offs (not just "what works" but "why this over that")?
3. Can you explain your own code and architectural decisions clearly?
4. Do you write tests and handle errors, or only the happy path?

### Mid technical questions (.NET + React)

**Architecture & patterns:**
- Walk me through your project's folder structure and the decisions behind it
- What is Clean Architecture? What problem does it solve?
- What is the difference between a Domain Event and an Integration Event?
- What is the Outbox pattern? Why would you use it instead of publishing directly?
- What is CQRS? Why separate commands from queries?
- What is the Unit of Work pattern? How does it relate to repositories?

**ASP.NET Core:**
- Walk me through what happens when an HTTP request hits your API (middleware pipeline)
- What is the difference between Singleton, Scoped, and Transient lifetimes?
- What is a captive dependency? How do you avoid it?
- How does EF Core's change tracker work? What is `AsNoTracking` for?
- What is the difference between `IOptions`, `IOptionsSnapshot`, and `IOptionsMonitor`?

**C# / async:**
- What is the difference between `Task.Run` and `await`?
- When would you use `ConfigureAwait(false)`?
- What is a `CancellationToken` and when do you propagate it?
- What is `async void` and why is it dangerous?

**React:**
- What triggers a re-render in React?
- When does `useEffect` run? What does the dependency array do?
- What is the difference between controlled and uncontrolled components?
- When would you use `useMemo` vs `useCallback`?
- What is the difference between RTK Query and Zustand? What goes in each?

**Mid component design questions:**
- "Design a simple shopping cart on the backend — data model, API, concurrency"
- "How would you add caching to your API? What would you cache and why?"
- "Design the database schema for an order system"
- "How would you handle a slow endpoint — what steps would you take?"
- "Design a notification service that sends emails when orders are placed"

---

## Senior Level Interview

### What to expect
- Recruiter screen
- Live coding (medium-hard, possibly with design pattern discussion)
- System design × 1-2 (full distributed systems, 45 min each)
- Behavioral/leadership round (conflict, influence, failure, ambiguity)
- Sometimes: architecture review of their existing system or "what's wrong with this code?"

### What they're really checking
1. Can you handle ambiguity and ask the right clarifying questions?
2. Do you make reasonable trade-off decisions under time pressure?
3. Can you lead — technically and interpersonally?
4. Do you have opinions, and can you defend and adjust them?

### Senior technical questions

- How would you design a multi-tenant SaaS database — shared vs separate schemas?
- How would you ensure exactly-once processing in a message-driven system?
- How would you migrate a monolith to microservices without downtime?
- What are the CAP theorem trade-offs for a financial transaction system?
- How would you handle a production incident where the API is returning 500s?
- How do you design for observability from the start of a project?
- What's your strategy for zero-downtime database migrations?
- What is the difference between Hexagonal Architecture and Clean Architecture?
- How would you approach a 10× traffic spike — where do you start?
- How do you handle technical debt on a team with constant feature pressure?

---

## Live Coding: The 6-Step Framework

Use this every time — even when the problem seems easy.

```
1. UNDERSTAND  Read it twice. Repeat the problem in your own words.
               "So I need to find X given Y — is that right?"

2. CLARIFY     Ask 2-3 questions before touching the keyboard.
               - Input constraints? (null? empty? negative numbers?)
               - Return type?
               - Any edge cases I should know about?

3. EXAMPLES    Trace through 2 examples manually — one normal, one edge case.
               Write them in comments or on the whiteboard.

4. APPROACH    Explain your algorithm before writing code.
               "I'll use a hash map to track counts — O(n) time, O(n) space."
               Get buy-in before coding.

5. CODE        Write clean, readable code. Talk as you write.
               Name variables well. Don't optimize prematurely.

6. TEST        Walk through your examples manually with the code.
               Check edge cases: empty input, single element, duplicates.
```

### When you're stuck
- "Let me think out loud for a moment..."
- "I know I need to track [X], what data structure would work well for that?"
- "Can I start with a brute-force O(n²) solution first and then optimize?"
- Hint-fishing is fine — interviewers want to see how you respond to guidance

### Mistakes that fail interviews
- Coding in silence (they can't assess your thinking)
- Jumping to optimize before having anything working
- Not asking ANY clarifying questions
- Changing approach mid-implementation without explaining why
- Skipping the test step
- Writing clever one-liners that you can't explain

---

## System Design Interview Framework

You have ~45 minutes. Use this structure:

### Step 1 — Clarify requirements (5 min)
**Functional:** What does it do? What are the core features? What's out of scope?
**Non-functional:** How many users? Read-heavy or write-heavy? Latency requirements? Availability target?

```
"Before I start, let me clarify requirements.
Functionally, I'll focus on X, Y, Z and leave A out of scope.
For scale — are we talking 1 million DAU? Read-heavy? 
What's our latency target — under 100ms for reads?"
```

### Step 2 — Estimate scale (3 min)
Back-of-envelope math. Shows you can reason about scale.

```
Example (URL shortener):
- 100M DAU, 10:1 read:write ratio
- Writes: 100M × 0.1 = 10M/day = ~120 writes/sec
- Reads: 1200 reads/sec
- Storage: 500 bytes/URL × 10M new URLs/day × 365 × 10 years ≈ 18TB
```

### Step 3 — API design (3 min)
Define the key endpoints. Keep it simple.

```
POST /urls          { originalUrl } → { shortCode }
GET  /{shortCode}   → 301 redirect to originalUrl
```

### Step 4 — Data model (5 min)
Main tables, key fields, SQL vs NoSQL decision with reasoning.

```
urls: id (PK), shortCode (indexed), originalUrl, createdAt, userId
Why SQL: relational data, need ACID, < 1TB fits on one DB
```

### Step 5 — High-level architecture (10 min)
Draw the boxes and explain each one.

```
Client → CDN → API Gateway → App Servers → Cache (Redis) → DB
                                          ↓
                                       Analytics (async queue)
```

### Step 6 — Deep dive (10 min)
Pick the 1-2 hardest parts. For URL shortener: short code generation (hash collision?), redirect performance (cache), expiry.

### Step 7 — Trade-offs (5 min)
"What's the bottleneck? The DB on high read load — so I'd shard by short code or move to a key-value store. The trade-off is losing JOIN capability, which matters less here since lookups are by key."

### Common mid-level system design topics
- URL shortener
- Rate limiter
- Notification service (email/push)
- Shopping cart / order system
- File upload service
- Basic chat system
- News feed (simplified)

---

## Take-Home Assignment Best Practices

Most companies give 3-5 days. Treat it like a real task at a job.

**Do:**
- Write a `README.md`: how to run, architecture decisions, what you'd add next
- Write tests — even a few shows professionalism
- Handle errors at the boundary (invalid input, missing resource → proper status codes)
- Clean git history with meaningful commit messages
- Use `.env.example` for config, never commit secrets

**Don't:**
- Over-engineer — they're testing if you can scope appropriately
- Under-comment tricky decisions — leave a comment explaining why, not what
- Submit after 2 hours of work or after 20 hours — 4-8 hours is the right signal
- Use a technology you can't defend if asked about it

**Winning README template:**
```markdown
## What it does
[One paragraph]

## How to run
[Commands]

## Architecture decisions
- [Decision 1] — because [reason]
- [Decision 2] — because [reason]

## Trade-offs
- [What I simplified and why]

## What I'd add with more time
- [Feature / improvement]
```

---

## Company Research Checklist

Do this before every interview — 30 minutes minimum.

- [ ] What does the company do? Who are their customers?
- [ ] What is the tech stack? (job posting, engineering blog, GitHub, StackShare)
- [ ] Do they have an engineering blog? What problems do they write about?
- [ ] Company stage: seed / Series A-C / public / enterprise? (changes interview style and risk)
- [ ] Team size and structure: remote? hybrid? how many engineers?
- [ ] Recent news: funding, product launches, layoffs, pivots
- [ ] Glassdoor / Blind / LinkedIn: interview experience reviews, salary ranges, culture signals
- [ ] Who is interviewing you? Read their LinkedIn — find common ground or interesting work

---

## Red Flags During the Process

Watch for these. A bad hire is recoverable. A bad job takes months to exit.

**Technical red flags:**
- "We don't have time for tests" or no CI/CD
- Can't explain their own tech stack or why they chose it
- "We're fixing the tech debt soon" (they've been saying this for 3 years)
- All engineers have been there less than a year (churn signal)
- No code review process, or "PR reviews are just approvals"

**Cultural red flags:**
- "We're like a family here" — code for poor work-life balance and emotional manipulation
- Pressure to decide same day (exploding offers) — legitimate companies don't do this
- Negative comments about previous employees during the interview
- Disorganized process: last-minute cancellations, interviewer didn't read your CV
- They can't answer "what does success look like in this role in 6 months?"

**Remote-specific red flags:**
- "Remote but you'll need to be in the office sometimes" with no defined policy
- All communication is synchronous; no documentation culture
- Meetings at your 10pm for their convenience, every week

---

## Salary Negotiation

### The rules
1. **Never give a number first.** "I'm open — what's the budgeted range for this role?" If they press: "I'd need to understand the full scope before giving a number."
2. **Research first.** Use Glassdoor, Levels.fyi, LinkedIn Salary, local job boards, communities (Discord, Slack groups for your area).
3. **Always negotiate.** 80%+ of companies expect a counter. The first offer is rarely the best offer.
4. **Counter with a specific number, not a range.** "I was thinking €X" — a range signals your minimum.
5. **Negotiate the full package:** base + equity/options + bonus + extra PTO + equipment budget + learning/conference budget + remote flexibility.
6. **If base is fixed:** ask for a signing bonus, earlier review for a raise, additional PTO, or a better title.
7. **Get it in writing** before you resign from your current job.

### Framing a counter
```
"I'm genuinely excited about this role and the team.
Based on my research and the scope of the position,
I was hoping we could get to [X]. Is there flexibility there?"
```

### What "no flexibility" usually means
In large companies: HR has a band, the hiring manager can't exceed it — escalate the band or ask for equity/signing.
In startups: often true, but equity and title have more flexibility.

---

## Questions to Ask at Each Stage

### Recruiter screen
- "What is the interview process from here?"
- "What's the timeline for making a decision?"
- "What's the budgeted range for this role?"

### Technical interview
- "What does a typical week look like for someone in this role?"
- "What's the biggest technical challenge the team is facing right now?"
- "How do you handle technical debt — is there dedicated time?"
- "What does the on-call rotation look like?"
- "How much autonomy do engineers have over technical decisions?"

### Final round / hiring manager
- "What would make someone in this role wildly successful at 6 months?"
- "What are the biggest risks or challenges I should know about?"
- "Why is this position open — is it new headcount or backfill?"
- "What's the culture around learning and growth?"

---

## Interview Mindset

The interview is not just them evaluating you. You are also evaluating them.

**What separates good candidates:**
- They think out loud — you can follow their reasoning
- They ask good clarifying questions before diving in
- They say "I don't know, but here's how I'd figure it out" — not silence or bluffing
- They connect their experience to the specific role
- They have genuine curiosity about the team's problems

**What looks bad:**
- Claiming credit for team work ("I built" vs "I was part of the team that built")
- Badmouthing previous employers
- Overselling skills you can't demonstrate under follow-up
- Rambling answers with no structure (use STAR for behavioral, think-out-loud for technical)
- Showing no interest in the product or company

---

## Interview Prep Timeline

### 4 weeks before
- Define your target roles and companies
- Update CV + LinkedIn + GitHub
- Prepare and practice your STAR stories (see `behavioral.md`)
- Start LeetCode — 3-4 problems/week at your target level

### 2 weeks before
- Research target companies thoroughly
- Practice live coding out loud (record yourself or pair with someone)
- Prepare your project walkthrough (architecture decisions, trade-offs, what you'd do differently)
- Research salary ranges for your target role and location

### 1 week before
- Confirm interview format and prepare accordingly
- Do one mock system design (draw it, talk through it out loud)
- Prepare 5-6 questions to ask the interviewer
- Review your own project code — you will be asked about it

### Day before
- Review the company research notes
- Confirm logistics (timezone, link, what to have ready)
- Get a good night's sleep — that's not a cliché

---

## Key Interview Questions

| Type | Questions |
|---|---|
| Opening | "Tell me about yourself" → 90s pitch |
| Strength | One real strength + concrete example |
| Weakness | Real + active mitigation |
| Technical challenge | STAR: what was hard, what you did, what you learned |
| Mistake | Honest + what changed after |
| Disagreement | Respectful, listened, outcome > ego |
| Why this company | Specific, honest, connected to your goals |
| Where in 2-3 years | Clear direction + why this role helps get there |
