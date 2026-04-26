# Behavioral Interview Prep

## What it is
The non-technical half of every interview. Behavioral questions assess how you think, communicate, handle conflict, and grow. They are often what separates candidates with similar technical skills.

## Why it matters
A perfect LeetCode score won't save you if you can't explain your past work clearly. And for remote junior-to-mid roles, communication is weighted even higher.

---

## The STAR Method

The universal framework for answering behavioral questions.

```
S — Situation  What was the context? Set the scene briefly.
T — Task       What were you responsible for? What problem needed solving?
A — Action     What did YOU specifically do? (not "we") Be concrete.
R — Result     What happened? Use numbers where possible. What did you learn?
```

**Keep it tight:** 90 seconds max per answer. Interviewers lose interest after 2 minutes.

**Prepare 5-6 strong stories** that can flex to answer many different questions. One story can answer multiple questions by emphasizing different aspects.

---

## Your Core Stories (based on your projects)

Build these stories before your next interview:

### Story 1: E-commerce Architecture Decision
**When to use it:** "Tell me about a difficult technical decision" / "How do you handle trade-offs"
```
S: Building an e-commerce backend solo — needed to choose architecture
T: Had to decide between simple layered architecture vs Clean Architecture + DDD
A: Researched both, prototype Clean Architecture with one feature to test if complexity was justified.
   Found that DDD aggregates made business invariants explicit and testable.
R: Architecture scaled from 3 to 8 features without debt. Understood the trade-off:
   more upfront complexity, better long-term maintainability. If I were building a 2-week MVP,
   I'd choose simpler architecture.
```

### Story 2: Learning Something Fast
**When to use it:** "Tell me about a time you had to learn quickly" / "How do you approach new tech"
```
S: Transitioning from casino dealer to developer — needed to learn .NET, React, architecture
T: Had to go from beginner to mid-level in months, not years
A: Built a real project instead of tutorials. Identified gaps systematically (kept a learning tracker).
   Used test-then-teach approach — wrote code, hit problems, read, fixed.
R: Completed [specific project/course]. Can now contribute to production .NET/React codebases.
```

### Story 3: Debugging a Hard Problem
**When to use it:** "Describe a challenging bug" / "How do you approach problems"
```
Use a real bug from your E-commerce project.
Template:
S: What feature were you building?
T: What was the unexpected behavior?
A: How did you diagnose it? (what tools, what hypotheses, what ruled out)
R: What was the root cause? What did you change? What did you learn?
```

### Story 4: Handling Feedback or Mistakes
**When to use it:** "Tell me about a mistake" / "How do you handle criticism"
```
S: Describe a concrete mistake (wrong approach, wrong estimate, missed requirement)
T: The impact of the mistake
A: How you discovered it, what you did to fix it, what you communicated
R: The outcome and what you changed to prevent it in the future
Be honest — interviewers can tell when you're spinning. Show self-awareness.
```

### Story 5: Working Under Pressure
**When to use it:** "How do you handle competing priorities" / "Deadlines and pressure"
```
S: Night shift casino dealer while learning to code — genuinely split attention
T: Had to maintain progress on learning goals despite unpredictable schedule
A: Time-blocked learning sessions, built a knowledge tracking system, prioritized ruthlessly
R: Consistent progress across months. Learned that constraints force better prioritization.
```

---

## Common Question Categories

### "Tell me about yourself" (opening)

Not your life story. 60-second pitch:
```
Format: [current role/situation] + [what you've built] + [what you're looking for]

Example:
"I'm a developer transitioning from a background in the casino industry.
Over the past [X months], I've built a full-stack e-commerce application using
.NET 8, ASP.NET Core, and React with TypeScript — end-to-end, including 
Clean Architecture, DDD, and CQRS patterns. I'm looking for a junior-to-mid
remote role where I can contribute immediately and keep growing technically."
```

---

### Technical Growth Questions

**"Why do you want this role?"**
- Be specific about the company/tech stack
- Connect to your learning trajectory
- Never say "to earn money" or vague "growth opportunities"

**"Where do you want to be in 2-3 years?"**
```
"I want to be a solid mid-level engineer who can own features end-to-end,
contribute to architecture decisions, and mentor newer developers.
I'm specifically interested in growing my system design and distributed systems knowledge —
which is one of the reasons this role interests me, since [company thing]."
```

**"What are your weaknesses?"**
- Be honest, but pick something real that doesn't block the job
- Show that you're actively working on it
```
Example: "I'm still building confidence in system design at scale — large distributed 
systems with hundreds of services. I'm working on this systematically: I've read 
[book/resource], and I'm applying the patterns to my current project.
I can already design systems to handle moderate load, but I know I have room to grow here."
```

---

### Conflict and Collaboration

**"Tell me about a disagreement with a colleague/manager"**
```
Key: Show that you disagreed respectfully, presented your reasoning, listened to theirs,
and reached a productive outcome — even if it wasn't your original position.

Avoid: Stories where you were right and they were wrong, period.
Show: Intellectual humility + ability to articulate technical trade-offs calmly.
```

**"How do you handle code review feedback?"**
```
"I view code review as a learning tool, not a judgment. When I get feedback,
I try to understand the reasoning before accepting or questioning it.
If I disagree, I explain my thinking and ask why the reviewer prefers their approach —
usually there's a constraint or context I wasn't aware of.
If I still disagree after understanding the reasoning, I make my case once clearly,
then defer to the team decision."
```

---

### Ownership and Initiative

**"Tell me about a time you went beyond what was asked"**
- Story about adding tests when not required
- Story about proactively documenting something confusing
- Story about identifying a bug before it reached users

**"How do you prioritize when you have multiple tasks?"**
```
"I start by clarifying which task is blocking others — dependencies first.
Then I sort by impact × urgency. I time-box difficult problems (max 1-2 hours before
asking for help) to avoid spiraling. I keep a todo list and update it as priorities shift."
```

---

## Questions to Ask the Interviewer

Asking good questions signals genuine interest. Prepare 3-4 from this list:

**Technical:**
- "What does the tech stack look like day-to-day? Any parts you're actively working to improve?"
- "How do you handle technical debt — is there time set aside for it?"
- "What does the onboarding process look like for a new engineer?"

**Team:**
- "How does the team handle code reviews — what's the culture there?"
- "What does a typical sprint look like from planning to shipping?"

**Growth:**
- "What would a junior engineer need to do to get to mid-level in this company?"
- "What are the biggest technical challenges the team is facing in the next 6 months?"

**Avoid:** "What's the salary?" (wait until they bring it up or it's the right moment). "Is there remote work?" (you should know this before applying).

---

## Talking About Your Projects

Structure for talking about your E-commerce app or CasinoTrainingApp:

```
1. What it does (one sentence)
2. Why you built it / what problem it solves
3. Stack and architecture decisions (and WHY — this is what impresses)
4. Most interesting technical challenge you solved
5. What you'd do differently now
```

**Example:**
```
"I built a full-stack e-commerce platform using .NET 8 and React.
The interesting architectural decision was applying Clean Architecture with DDD —
rather than a standard 3-layer approach. I made this call because the domain
has real business rules: order state transitions, inventory checks, pricing rules.
These belong in domain entities, not in services. The most challenging part was
implementing the Outbox pattern for reliable domain event publishing — ensuring
events reach RabbitMQ exactly once even if the server crashes mid-transaction.
If I were starting over, I'd add more integration tests earlier — I underestimated
how much confidence they give when refactoring."
```

---

## Remote Work Specifics

Companies hiring remote specifically ask about this:

**"How do you communicate in a distributed team?"**
```
"I default to async, written communication — I over-document decisions and reasoning
rather than expecting people to be available synchronously. When something is ambiguous,
I ask in a message with my current understanding and specific question, not just 'can you help?'
I prefer short recorded demos or screenshots over long meetings when I need to show something."
```

**"How do you manage distractions working from home?"**
- Time-blocking
- Dedicated workspace
- Visible availability (calendar, Slack status)

---

## Common Interview Questions Cheat Sheet

| Question | Recommended approach |
|---|---|
| Tell me about yourself | 60s pitch: situation → built → looking for |
| Biggest strength | Pick one real one + concrete example |
| Biggest weakness | Real weakness + active mitigation |
| Challenging bug | STAR: diagnosis process, root cause, fix |
| Disagreement | Respectful, listened, outcome > ego |
| Why this company | Specific, honest, connected to your goals |
| Where in 5 years | Clear direction, shows ambition but realism |
| Questions for us | 2-3 genuine, prepared questions |

---

## Common Mistakes

- Answering "what's your weakness" with a fake positive ("I work too hard")
- Using "we" throughout — interviewers are assessing YOU
- Rambling — keep STAR answers to 60-90 seconds
- Not having specific examples ready — vague answers signal vague thinking
- Not asking any questions at the end (signals low interest)
- Memorizing scripts — sounds robotic; internalize the story, not the words

---

## My Confidence Level
- `[ ]` STAR method — structure and time control
- `[ ]` Core stories prepared (architecture, learning, debugging, mistake, pressure)
- `[ ]` "Tell me about yourself" pitch
- `[ ]` Weakness question — honest + growth framing
- `[ ]` Technical project walkthrough — architecture + decisions + honest reflection
- `[ ]` Remote work answers
- `[ ]` Good questions for the interviewer

## My Notes
<!-- Personal notes — notes from actual interviews, questions you got -->
