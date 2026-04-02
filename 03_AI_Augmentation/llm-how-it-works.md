# How LLMs Work — Embeddings, RAG, and Vector Databases

A practical mental model for data engineers. No math required.

---

## 1. How LLMs Work (the short version)

An LLM (Large Language Model) is a neural network trained to predict the next token (word fragment) given everything before it. That's the entire training objective — predict what comes next, billions of times, on trillions of words.

Through this, the model implicitly learns:
- Grammar and syntax
- Facts about the world
- Reasoning patterns
- Code structure

**What it doesn't have:** memory between conversations, access to real-time data, or knowledge of anything after its training cutoff.

**Tokens** are the unit of input/output. One token ≈ 0.75 words. "Data engineering" = 3 tokens. Models have a **context window** — a maximum number of tokens they can see at once (e.g., Claude's is ~200K tokens, GPT-4o is 128K).

---

## 2. Embeddings

### What they are

An embedding is a way to convert text (or any data) into a list of numbers — a **vector** — that captures semantic meaning.

```
"revenue dropped 20%" → [0.12, -0.87, 0.34, 0.91, ...]  (1536 numbers)
"sales declined by a fifth" → [0.11, -0.85, 0.36, 0.89, ...]  (very similar!)
"the cat sat on the mat" → [0.73, 0.21, -0.44, 0.02, ...]  (very different)
```

Semantically similar text produces numerically similar vectors. This is the key property that makes everything else possible.

### How they're generated

You call an embedding model (separate from the LLM) with text, and it returns a vector:

```python
import anthropic  # or openai, etc.

client = openai.OpenAI()
response = client.embeddings.create(
    input="revenue dropped 20% in Q3",
    model="text-embedding-3-small"
)
vector = response.data[0].embedding  # list of 1536 floats
```

### What they're used for

| Use case | How embeddings help |
|---|---|
| Semantic search | Find documents by meaning, not just keyword match |
| RAG | Find relevant context to inject into a prompt |
| Recommendations | "Users who liked X also liked Y" — by vector proximity |
| Clustering | Group similar documents automatically |
| Anomaly detection | Detect text that's semantically far from the norm |

---

## 3. Vector Databases

### The problem they solve

You have 1 million documents, each embedded as a 1536-dimensional vector. When a query comes in, you need to find the top-10 most similar vectors fast. A brute-force comparison would be too slow at scale.

Vector databases solve this with **Approximate Nearest Neighbor (ANN)** indexing — data structures that trade a tiny bit of accuracy for massive speed gains.

### How similarity is measured

The standard metric is **cosine similarity** — measures the angle between two vectors (not distance). Score ranges from -1 to 1; higher = more similar.

```
cosine_similarity([0.12, -0.87, ...], [0.11, -0.85, ...]) → 0.98  ✓ similar
cosine_similarity([0.12, -0.87, ...], [0.73, 0.21, ...])  → 0.11  ✗ different
```

### Popular vector databases

| DB | Type | Best for |
|---|---|---|
| **Pinecone** | Managed cloud | Production, no ops overhead |
| **Weaviate** | Open source / cloud | Hybrid search (vector + keyword) |
| **Qdrant** | Open source / cloud | High performance, Rust-based |
| **Chroma** | Open source | Local dev, prototyping |
| **pgvector** | Postgres extension | Already on Postgres, simpler stack |
| **FAISS** | Library (Meta) | In-memory, research, no persistence |

### What a vector DB stores

Each record has three parts:
- **ID** — unique identifier
- **Vector** — the embedding (e.g., 1536 floats)
- **Metadata** — original text, source, date, any filters you want

```python
# Example record in Pinecone
{
  "id": "doc_123",
  "values": [0.12, -0.87, 0.34, ...],   # the embedding
  "metadata": {
    "text": "Revenue dropped 20% in Q3 due to seasonality",
    "source": "earnings_call_2024_q3.pdf",
    "date": "2024-10-15"
  }
}
```

---

## 4. RAG (Retrieval-Augmented Generation)

### The problem it solves

LLMs are frozen at their training cutoff and don't know your private data. RAG solves this by **fetching relevant context at query time and injecting it into the prompt**.

### How it works — step by step

```
User question
    │
    ▼
[1] Embed the question  →  query vector
    │
    ▼
[2] Search vector DB  →  top-K most similar documents
    │
    ▼
[3] Build prompt:
    "Answer this question using the context below.
     Context: {retrieved docs}
     Question: {user question}"
    │
    ▼
[4] Send to LLM  →  grounded answer
```

### Concrete example

```python
# 1. Embed the user question
query_vector = embed("What caused the Q3 revenue drop?")

# 2. Retrieve relevant docs
results = vector_db.query(query_vector, top_k=5)
context = "\n".join([r.metadata["text"] for r in results])

# 3. Build the prompt
prompt = f"""
Answer the question using only the context below.
Context: {context}
Question: What caused the Q3 revenue drop?
"""

# 4. Call the LLM
response = llm.complete(prompt)
```

### Why RAG beats fine-tuning for most use cases

| | RAG | Fine-tuning |
|---|---|---|
| **When to use** | Dynamic / frequently updated data | Teaching the model a new style or format |
| **Data updates** | Instant — just re-embed new docs | Requires retraining (slow, expensive) |
| **Cost** | Low — embedding + retrieval | High — GPU training runs |
| **Transparency** | You can see exactly what context was used | Black box |
| **Hallucination risk** | Lower — model is grounded in retrieved facts | Higher — model may confabulate |

**Rule of thumb:** Use RAG when your data changes. Use fine-tuning when you need to change *how* the model behaves (tone, output format, domain-specific reasoning patterns).

---

## 5. Chunking — The Detail That Matters Most

Before embedding documents, you have to split them into chunks. Chunk quality directly determines RAG quality.

### Why it matters

- Too large: retrieved chunks contain too much noise, dilute relevance
- Too small: chunks lose context, retrieved text is unintelligible on its own
- Splits at wrong boundaries: a sentence about "revenue" gets cut in half

### Common strategies

| Strategy | How it works | Best for |
|---|---|---|
| **Fixed size** | Split every N characters/tokens, overlap by M | Simple baseline, works ok |
| **Sentence** | Split on sentence boundaries | General prose |
| **Paragraph / section** | Split on `\n\n` or headers | Structured docs |
| **Semantic** | Group sentences by topic shift | High-quality retrieval, slower |
| **Recursive** | Try paragraph → sentence → character until size fits | LangChain default, robust |

**Typical settings:** 512–1024 tokens per chunk, 10–20% overlap between adjacent chunks.

---

## 6. Full Architecture Picture

```
INDEXING (offline, run once or on update)
─────────────────────────────────────────
Documents → Chunker → Embedding Model → Vector DB
                                            │
                                            │ (stored)
                                            ▼
QUERYING (real-time, per user request)  Vector DB
─────────────────────────────────────────────────
User Question → Embedding Model → Vector DB (ANN search)
                                       │
                                       ▼ top-K chunks
                              Prompt Builder
                                       │
                                       ▼ prompt + context
                                      LLM
                                       │
                                       ▼
                                    Answer
```

---

## 7. DE Relevance

As a data engineer, RAG pipelines are data pipelines with different outputs:

| Pipeline concept | RAG equivalent |
|---|---|
| Ingestion | Document loader — pull PDFs, HTML, DBs |
| Transformation | Chunker + metadata enrichment |
| Load | Upsert vectors to vector DB |
| Incremental updates | Re-embed changed docs, delete old vectors |
| Data quality | Chunk quality, embedding coverage, retrieval recall |
| Orchestration | Airflow / Dataswarm DAG triggering re-indexing |
| Observability | Track retrieval latency, top-K relevance scores, LLM answer quality |

The skills you already have (pipeline design, data modeling, DQ checks) transfer directly to building and maintaining RAG systems.
