# Agentic Design Patterns

## What it is
Patterns for building AI systems where a language model takes a sequence of actions, uses tools, and makes decisions to accomplish a complex goal — rather than just responding to a single prompt.

## Why it matters
Single-turn LLM calls handle simple Q&A. Real applications require multi-step reasoning: searching, writing to databases, calling APIs, checking results, retrying. Agentic patterns structure this complexity.

---

## The Building Blocks

An AI agent is a loop:
```
Observe → Think → Act → Observe → Think → Act → ...
```

Key components:
- **LLM:** The reasoning engine
- **Tools:** Functions the LLM can call (search, write file, call API)
- **Memory:** What the agent remembers across steps
- **Planning:** How the agent breaks down a goal

---

## Pattern 1: Prompt Chaining

**What:** Break a complex task into a sequence of simpler LLM calls. Output of one step = input of next.

```
Step 1: Extract product requirements from user description
Step 2: Generate SQL schema from requirements
Step 3: Generate seed data from schema
Step 4: Generate API endpoints from schema
```

**When to use:** Task is too complex for one prompt; each step is well-defined; quality gates between steps.

**In code:**
```csharp
public async Task<string> GenerateProductSchemaAsync(string userDescription)
{
    // Step 1
    var requirements = await _llm.CompleteAsync(
        $"Extract structured requirements from: {userDescription}");

    // Step 2 — uses output of step 1
    var schema = await _llm.CompleteAsync(
        $"Generate SQL schema for these requirements: {requirements}");

    return schema;
}
```

---

## Pattern 2: Routing

**What:** A classifier LLM routes the input to specialized sub-agents or prompts based on type.

```
User input
    ↓
Router LLM → "This is a billing question"
    ↓
Billing Agent (specialized prompt + tools)
```

**When to use:** Different types of inputs need very different handling. Keep specialized agents focused.

---

## Pattern 3: Parallelization

**What:** Run multiple LLM calls simultaneously and aggregate results.

**Subtypes:**
- **Sectioning:** Split a task into independent parts, run in parallel
- **Voting:** Run same prompt N times, pick majority answer (for reliability)

```csharp
// Parallel analysis of a document
var tasks = new[]
{
    _llm.CompleteAsync($"Summarize the technical aspects: {document}"),
    _llm.CompleteAsync($"Identify business risks: {document}"),
    _llm.CompleteAsync($"Extract action items: {document}")
};

var results = await Task.WhenAll(tasks);
```

**When to use:** Independent subtasks; need multiple perspectives; confidence through voting.

---

## Pattern 4: Tool Use (Function Calling)

**What:** LLM decides to call a function you've defined. You run the function, feed the result back. The LLM uses the result to continue.

```
LLM: "I need to search for current stock price"
→ Calls: search_stock(symbol="AAPL")
→ Gets: { price: 187.32, change: +1.2% }
→ LLM: "Apple's current stock price is $187.32, up 1.2% today"
```

**Claude API example:**
```csharp
var tools = new[]
{
    new Tool("search_products", "Search the product catalog",
        parameters: new { query = "string", maxResults = "number" })
};

var response = await _claude.Messages.CreateAsync(new MessageRequest
{
    Model = "claude-opus-4-6",
    Messages = [new { role = "user", content = userMessage }],
    Tools = tools
});

// Check if Claude wants to use a tool
if (response.StopReason == "tool_use")
{
    var toolCall = response.Content.OfType<ToolUseBlock>().First();
    var result = await ExecuteTool(toolCall.Name, toolCall.Input);
    // Send result back to Claude
}
```

**When to use:** Agent needs real-time data (search, DB, APIs); needs to take actions (send email, create record).

---

## Pattern 5: Reflection (Self-Critique Loop)

**What:** LLM generates a response, then a second call (same or different LLM) critiques it. Repeat until quality threshold.

```
Generate: "Here's a function to sort an array..."
Critique: "The function has a bug: it doesn't handle empty arrays"
Revise:   "Here's the corrected function..."
Critique: "Looks good. No issues found."
→ Done
```

**When to use:** Code generation, document drafting, any task where quality matters more than speed.

**Risk:** Can loop forever. Always set a max iteration count.

---

## Pattern 6: Planning (ReAct)

**ReAct = Reasoning + Acting**

**What:** LLM alternates between reasoning ("I need to find X") and acting (calling a tool). The trace is visible and auditable.

```
Thought: I need to find the user's order history
Action: get_orders(userId="123")
Observation: [Order #456 - $89.99, Order #789 - $45.00]

Thought: I have the orders. Now I need the product details for order #456
Action: get_order_details(orderId="456")
Observation: { product: "Blue Sneakers", size: "42", status: "Delivered" }

Thought: I have everything I need to answer the question
Answer: Your most recent order is Blue Sneakers (size 42), delivered, for $89.99.
```

**When to use:** Complex multi-step tasks where you need to observe intermediate results.

---

## Pattern 7: Multi-Agent Systems

**What:** Multiple specialized agents collaborate. An orchestrator agent delegates to sub-agents.

```
Orchestrator Agent
├── Research Agent (searches web, reads documents)
├── Writing Agent (drafts content)
├── Review Agent (critiques and suggests improvements)
└── Publishing Agent (formats and publishes)
```

**Communication patterns:**
- **Sequential:** Each agent's output feeds the next
- **Hierarchical:** Orchestrator delegates, collects, combines
- **Parallel:** Multiple agents work simultaneously

**When to use:** Tasks too complex for one agent; need parallel specialization; different tools per domain.

---

## Pattern 8: RAG (Retrieval-Augmented Generation)

Covered in `rag.md`. Summary: retrieve relevant context from a knowledge base and inject it into the prompt.

```
User query → Embed query → Vector search → Top K chunks → Inject into prompt → LLM answer
```

---

## Pattern 9: Memory Management

### Types of Memory

| Type | What | How |
|---|---|---|
| Short-term | Current conversation history | Message array in context |
| Long-term | Facts about user, past sessions | Database, vector store |
| Episodic | Summary of past conversations | Summarization + retrieval |
| Semantic | General knowledge | Embeddings in vector store |

### Context Window Management
- Conversations grow → eventually exceed context limit
- **Summarization:** Periodically summarize old messages, replace with summary
- **Sliding window:** Keep only last N messages
- **Relevance filtering:** Only include messages relevant to current query

```csharp
// Auto-summarize when approaching context limit
if (tokenCount > MAX_CONTEXT_TOKENS * 0.8)
{
    var summary = await _llm.CompleteAsync(
        $"Summarize this conversation in 3-5 sentences: {conversationHistory}");
    conversationHistory = [new SystemMessage($"Previous context: {summary}")];
}
```

---

## Guardrails and Safety

Always add:
1. **Max iterations** — prevent infinite loops
2. **Tool call budget** — max N tool calls per request
3. **Human-in-the-loop** — for irreversible actions (delete, send email, payment)
4. **Output validation** — check LLM output before using it
5. **Timeout** — agents can get stuck

```csharp
public async Task<AgentResult> RunAgentAsync(string goal, int maxIterations = 10)
{
    for (int i = 0; i < maxIterations; i++)
    {
        var response = await _llm.CompleteAsync(BuildPrompt(goal, history));
        if (response.IsFinished) return AgentResult.Success(response.Answer);
        if (response.RequiresHumanApproval) return AgentResult.NeedsApproval(response.Action);
        await ExecuteToolAsync(response.ToolCall);
    }
    return AgentResult.Failure("Max iterations reached");
}
```

---

## Common Interview Questions

1. What is the difference between a chatbot and an AI agent?
2. What is tool use / function calling in LLMs?
3. What is the ReAct pattern?
4. What are the risks of agentic AI systems?
5. How do you manage memory in a long-running agent?

---

## How It Connects

- Prompt Chaining = Functional composition of LLM calls
- Tool Use = Dependency Injection for LLMs (tell the model what's available)
- Multi-Agent = Microservices architecture applied to AI
- Memory = Caching + database persistence for AI context
- Reflection = Integration testing loop applied to AI outputs
- RAG connects to your vector database and embedding pipeline

---

## My Confidence Level
- `[b]` Tool use / function calling
- `[~]` Prompt chaining
- `[ ]` Reflection (self-critique loop)
- `[ ]` Planning (ReAct)
- `[ ]` Multi-agent systems
- `[ ]` Memory management

## My Notes
<!-- Personal notes -->
