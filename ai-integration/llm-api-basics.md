# LLM API Basics

## What it is
How to call Large Language Model APIs (Claude, OpenAI) from your .NET backend and React frontend to add AI capabilities to your applications.

## Why it matters
LLM APIs are the foundation of every AI feature. Understanding how they work — tokens, context windows, streaming, tool use — lets you build reliable AI-powered features without surprises.

---

## Core Concepts

### Tokens
LLMs don't process words or characters — they process **tokens**. A token ≈ 4 characters in English. "Hello world" = 2 tokens.

- Input tokens = what you send
- Output tokens = what the model generates
- Pricing = cost per 1M input tokens + cost per 1M output tokens
- Context window = max total tokens (input + output) in one call

### Context Window
The "memory" of a conversation. Once you exceed it, you must truncate or summarize older messages.

| Model | Context Window |
|---|---|
| Claude Opus 4.6 | 200K tokens |
| Claude Sonnet 4.6 | 200K tokens |
| GPT-4o | 128K tokens |

### Temperature
Controls randomness. 0 = deterministic (same output every time), 1 = creative/random.
- Code generation: 0–0.2
- Creative writing: 0.7–1.0
- Factual Q&A: 0–0.3

---

## Claude API from .NET

### Setup
```bash
dotnet add package Anthropic.SDK
```

```csharp
// Registration
services.AddSingleton(new AnthropicClient(config["Anthropic:ApiKey"]));
```

### Basic Completion
```csharp
public class ClaudeService
{
    private readonly AnthropicClient _client;

    public async Task<string> CompleteAsync(string userMessage, CancellationToken ct = default)
    {
        var response = await _client.Messages.GetClaudeMessageAsync(
            new MessageParameters
            {
                Model = AnthropicModels.Claude3Opus,
                MaxTokens = 1024,
                Messages = [new Message { Role = RoleType.User, Content = userMessage }],
                System = "You are a helpful e-commerce assistant."
            }, ct);

        return response.Content.OfType<TextContent>().First().Text;
    }
}
```

### Multi-turn Conversation
```csharp
public async Task<string> ChatAsync(List<ConversationMessage> history, string newMessage, CancellationToken ct)
{
    var messages = history.Select(m => new Message
    {
        Role = m.IsUser ? RoleType.User : RoleType.Assistant,
        Content = m.Text
    }).ToList();

    messages.Add(new Message { Role = RoleType.User, Content = newMessage });

    var response = await _client.Messages.GetClaudeMessageAsync(
        new MessageParameters
        {
            Model = AnthropicModels.Claude3Sonnet,
            MaxTokens = 2048,
            Messages = messages
        }, ct);

    return response.Content.OfType<TextContent>().First().Text;
}
```

### Streaming
```csharp
public async IAsyncEnumerable<string> StreamAsync(string prompt, [EnumeratorCancellation] CancellationToken ct)
{
    await foreach (var chunk in _client.Messages.StreamClaudeMessageAsync(
        new MessageParameters
        {
            Model = AnthropicModels.Claude3Sonnet,
            MaxTokens = 1024,
            Messages = [new Message { Role = RoleType.User, Content = prompt }]
        }, ct))
    {
        if (chunk is ContentBlockDeltaEvent { Delta: TextDelta textDelta })
            yield return textDelta.Text;
    }
}
```

### ASP.NET Core Streaming Endpoint
```csharp
[HttpPost("stream")]
public async Task StreamCompletion([FromBody] string prompt, CancellationToken ct)
{
    Response.ContentType = "text/event-stream";
    Response.Headers.CacheControl = "no-cache";

    await foreach (var chunk in _claudeService.StreamAsync(prompt, ct))
    {
        await Response.WriteAsync($"data: {chunk}\n\n", ct);
        await Response.Body.FlushAsync(ct);
    }
}
```

---

## Claude API from React (RTK Query)

### Setup
```bash
npm install @anthropic-ai/sdk
```

**Warning:** Never call LLM APIs directly from the browser with your API key — the key will be exposed. Always proxy through your backend.

### RTK Query Endpoint
```typescript
// features/ai/aiApi.ts
export const aiApi = baseApi.injectEndpoints({
  endpoints: (builder) => ({
    generateProductDescription: builder.mutation<
      { description: string },
      { productName: string; features: string[] }
    >({
      query: (body) => ({
        url: '/ai/product-description',
        method: 'POST',
        body,
      }),
    }),

    // Streaming via EventSource
    streamCompletion: builder.query<string, string>({
      queryFn: () => ({ data: '' }),
      async onCacheEntryAdded(prompt, { updateCachedData, cacheDataLoaded }) {
        await cacheDataLoaded;
        const source = new EventSource(`/api/ai/stream?prompt=${encodeURIComponent(prompt)}`);
        source.onmessage = (e) => {
          updateCachedData((draft) => draft + e.data);
        };
      },
    }),
  }),
});
```

### React Component
```tsx
const ProductDescriptionGenerator = ({ product }: { product: Product }) => {
  const [generate, { data, isLoading }] = useGenerateProductDescriptionMutation();

  return (
    <div>
      <button
        onClick={() => generate({ productName: product.name, features: product.features })}
        disabled={isLoading}
      >
        {isLoading ? 'Generating...' : 'Generate Description'}
      </button>
      {data && <p>{data.description}</p>}
    </div>
  );
};
```

---

## Structured Output

Force the LLM to return JSON matching a schema.

```csharp
var systemPrompt = """
    You are a product data extractor. Always respond with valid JSON matching this schema:
    {
      "name": "string",
      "category": "string",
      "price": number,
      "features": ["string"]
    }
    Never include any text outside the JSON object.
    """;

var response = await _client.Messages.GetClaudeMessageAsync(
    new MessageParameters
    {
        Model = AnthropicModels.Claude3Sonnet,
        MaxTokens = 512,
        System = systemPrompt,
        Messages = [new Message { Role = RoleType.User, Content = rawProductText }]
    });

var json = response.Content.OfType<TextContent>().First().Text;
var product = JsonSerializer.Deserialize<ProductExtraction>(json);
```

---

## Error Handling & Resilience

```csharp
public async Task<Result<string>> SafeCompleteAsync(string prompt, CancellationToken ct)
{
    try
    {
        var response = await _client.Messages.GetClaudeMessageAsync(..., ct);
        return Result.Success(response.Content.OfType<TextContent>().First().Text);
    }
    catch (AnthropicException ex) when (ex.StatusCode == 429)
    {
        // Rate limited — implement exponential backoff
        return Result.Failure<string>(ErrorCodes.AiRateLimited);
    }
    catch (AnthropicException ex) when (ex.StatusCode >= 500)
    {
        return Result.Failure<string>(ErrorCodes.AiServiceUnavailable);
    }
    catch (OperationCanceledException)
    {
        return Result.Failure<string>(ErrorCodes.RequestCancelled);
    }
}
```

**Resilience with Polly:**
```csharp
services.AddHttpClient<ClaudeService>()
    .AddStandardResilienceHandler(); // Retry, circuit breaker, timeout
```

---

## Cost Management

```
Claude Opus 4.6:    ~$15/MTok input, ~$75/MTok output
Claude Sonnet 4.6:  ~$3/MTok input,  ~$15/MTok output
Claude Haiku 4.5:   ~$0.25/MTok input, ~$1.25/MTok output
```

**Strategies:**
- Use Haiku for classification, routing, simple tasks
- Use Sonnet for most generation tasks
- Use Opus only for complex reasoning
- Cache prompts (Anthropic prompt caching = 90% discount on cached input)
- Limit `MaxTokens` to expected output size

---

## Common Interview Questions

1. What is a token in the context of LLMs?
2. How do you prevent API key exposure when calling LLMs from the frontend?
3. What is streaming and why is it important for UX?
4. How do you handle rate limiting from LLM APIs?
5. When would you use a smaller/cheaper model vs a larger one?

---

## Common Mistakes

- Calling LLM API directly from browser (key exposure)
- Not handling rate limits (429) with exponential backoff
- Sending entire database tables as context (use RAG instead)
- Not caching expensive, repetitive LLM calls
- Ignoring `MaxTokens` — unbounded output costs money and takes time
- Storing API keys in code (use environment variables / secrets manager)

---

## How It Connects

- LLM API calls follow the same Result<T> pattern as your other services
- Streaming uses SSE (Server-Sent Events) — HTTP long-lived connection
- RTK Query handles streaming via `onCacheEntryAdded`
- Rate limiting handling mirrors your existing resilience patterns (Polly)
- Structured output + JSON parsing is how you integrate LLM into your domain model

---

## My Confidence Level
- `[b]` OpenAI / Anthropic API structure
- `[~]` Calling LLM from .NET (SDK)
- `[~]` Calling LLM from React
- `[ ]` Streaming responses
- `[ ]` Token counting, cost estimation

## My Notes
<!-- Personal notes -->
