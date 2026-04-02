# RAG (Retrieval-Augmented Generation)

## What it is
A pattern that grounds LLM responses in your own data by retrieving relevant context at query time and injecting it into the prompt. The LLM doesn't need to "know" your data in advance — it reads it on demand.

## Why it matters
LLMs have a knowledge cutoff and don't know your private data (your products, docs, customer data). RAG solves this without expensive fine-tuning. It also reduces hallucination by giving the model real facts to reason from.

---

## The Pipeline

```
User Query
    ↓
1. Embed the query (query → vector)
    ↓
2. Vector search against your knowledge base (find top K similar chunks)
    ↓
3. Retrieve the top K text chunks
    ↓
4. Build prompt: "Answer using this context: [chunks]\n\nQuestion: [query]"
    ↓
5. LLM generates answer grounded in retrieved context
    ↓
Answer (with optional source citations)
```

---

## Step 1: Document Processing (Offline)

Before you can retrieve, you need to process and store your documents.

### Chunking Strategies

**Fixed-size chunking:**
```csharp
// Split every N characters with overlap
var chunks = SplitIntoChunks(document, chunkSize: 500, overlap: 50);
```

**Semantic chunking:** Split at natural boundaries (paragraph, heading, sentence).

**Recursive character splitting:** Try to split by paragraph → sentence → word until chunk is small enough.

**Considerations:**
- Chunk too small → loses context
- Chunk too large → dilutes relevance, wastes context window
- Overlap → ensures you don't cut a sentence at the boundary
- For code: split by function/class, not character count

### Embeddings

Embeddings convert text to a vector of numbers that captures semantic meaning. Similar text → similar vectors.

```csharp
// Using OpenAI embeddings API
var embeddingResponse = await _openAi.Embeddings.CreateAsync(new EmbeddingRequest
{
    Model = "text-embedding-3-small",
    Input = chunkText
});
var vector = embeddingResponse.Data[0].Embedding; // float[]
```

**Models:**
- OpenAI: `text-embedding-3-small` (fast, cheap), `text-embedding-3-large` (better quality)
- Anthropic: doesn't provide embeddings — use OpenAI, Cohere, or open-source
- Open-source: `all-MiniLM-L6-v2` (fast, runs locally)

**Important:** Use the **same embedding model** at index time and query time.

### Store in Vector Database

```sql
-- pgvector in PostgreSQL
CREATE EXTENSION vector;
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY,
    document_id UUID,
    content TEXT,
    embedding vector(1536),  -- 1536 dims for text-embedding-3-small
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX ON document_chunks USING ivfflat (embedding vector_cosine_ops);
```

---

## Step 2: Query Time (Online)

### Embed the Query
```csharp
var queryEmbedding = await _embeddingService.EmbedAsync(userQuery);
```

### Similarity Search

Find the K most similar chunks using cosine similarity.

**Cosine similarity** = angle between two vectors. Range: -1 to 1. Higher = more similar.

```sql
-- pgvector: find top 5 most similar chunks
SELECT content, 1 - (embedding <=> @queryVector) AS similarity
FROM document_chunks
ORDER BY embedding <=> @queryVector  -- <=> = cosine distance
LIMIT 5;
```

### Build the Prompt

```csharp
var context = string.Join("\n\n---\n\n", topChunks.Select(c => c.Content));

var prompt = $"""
    You are a helpful assistant. Answer the question using ONLY the context below.
    If the answer is not in the context, say "I don't have that information."
    
    Context:
    {context}
    
    Question: {userQuery}
    
    Answer:
    """;

var answer = await _claude.Messages.CreateAsync(new MessageRequest
{
    Model = "claude-opus-4-6",
    Messages = [new { role = "user", content = prompt }],
    MaxTokens = 1024
});
```

---

## Advanced: Re-Ranking

Initial vector search returns semantically similar chunks. But "similar" doesn't always mean "most relevant for this specific question."

**Re-ranker:** A second model that scores each retrieved chunk's actual relevance to the query. More expensive but more accurate.

```
Vector search: retrieve top 20 candidates
Re-ranker: score each candidate vs query
Return: top 5 by re-rank score
```

Popular re-rankers: Cohere Rerank, cross-encoder models.

---

## Advanced: Hybrid Search

Combine vector search (semantic) with keyword search (BM25/full-text) for better results.

```
Vector score + BM25 score → combined rank (Reciprocal Rank Fusion)
```

Good for: technical docs where exact terms matter ("ProductNotFoundException" should match exactly).

---

## RAG in ASP.NET Core

### Architecture
```
src/
  Application/
    Interfaces/
      IEmbeddingService.cs
      IVectorStore.cs
      IRagService.cs
    Services/
      RagService.cs
  Infrastructure/
    AI/
      OpenAiEmbeddingService.cs
      PgVectorStore.cs
```

### Interfaces
```csharp
public interface IEmbeddingService
{
    Task<float[]> EmbedAsync(string text, CancellationToken ct = default);
}

public interface IVectorStore
{
    Task StoreAsync(DocumentChunk chunk, CancellationToken ct = default);
    Task<List<DocumentChunk>> SearchAsync(float[] queryVector, int topK = 5, CancellationToken ct = default);
}

public interface IRagService
{
    Task<string> AskAsync(string question, CancellationToken ct = default);
}
```

---

## Evaluation (How Do You Know It Works?)

**Retrieval metrics:**
- **Recall@K:** Of all relevant chunks, what % did we retrieve in top K?
- **Precision@K:** Of top K retrieved, what % are actually relevant?

**Generation metrics:**
- **Faithfulness:** Does the answer only use information from retrieved context?
- **Answer Relevance:** Does the answer actually address the question?
- **Context Relevance:** Are the retrieved chunks relevant to the question?

**Tools:** Ragas (Python), custom evaluation scripts.

---

## Common Pitfalls

- **Chunking too large:** Dilutes relevance score, wastes context window
- **Wrong embedding model at query time:** Different model = incompatible vector space
- **No overlap:** Answer split across two chunks, neither has full context
- **Raw text without metadata:** Can't cite sources, can't filter by date/category
- **Hallucination still happens:** LLM can ignore context and make things up — add faithfulness checks
- **No index on the vector column:** `ivfflat` or `hnsw` index required for production scale

---

## Common Interview Questions

1. What problem does RAG solve vs fine-tuning?
2. What is an embedding and how is it used in RAG?
3. What is cosine similarity?
4. What chunking strategy would you use for API documentation?
5. How do you evaluate a RAG pipeline?
6. What is re-ranking and when do you need it?

---

## How It Connects

- pgvector runs in PostgreSQL — same DB you already use for your app data
- Embeddings use the same API call pattern as chat completions
- RAG chunks are effectively domain documents — apply DDD thinking to chunking strategy
- EF Core can query pgvector with raw SQL or the `pgvector-dotnet` package
- Agentic patterns: RAG is often used as a tool within a larger agent

---

## My Confidence Level
- `[ ]` Chunking strategies
- `[ ]` Embeddings (what they are, how to generate)
- `[ ]` Vector stores (pgvector, Qdrant)
- `[ ]` Similarity search
- `[ ]` Retrieval pipeline end-to-end
- `[ ]` Re-ranking

## My Notes
<!-- Personal notes -->
