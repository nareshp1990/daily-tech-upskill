# Day 4 — Embeddings & Vector Search
**Date:** 2026-03-11
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Embeddings & Vector Search — Generating Embeddings, pgvector, and the Foundation of RAG**

Day 3 gave you a resilient LLM client. Today you learn **how machines understand meaning**. Embeddings are the bridge between human language and math — and vector search is how you find relevant documents at scale. This is the core primitive behind every RAG system, semantic search engine, and recommendation system built on LLMs. Think of it as: embeddings are to semantic search what B-tree indexes are to SQL queries.

---

## 2. Key Concepts

### What Is an Embedding?
- A vector (array of floats) that represents the **semantic meaning** of a piece of text.
- Similar meaning → similar vectors → small distance in vector space.
- Example: `"dog"` and `"puppy"` → vectors very close together. `"dog"` and `"invoice"` → far apart.
- Typical dimensions: 768 (BERT), 1536 (OpenAI ada-002), 1024 (Anthropic voyage-2).
- Java analogy: like a HashMap where the key is the concept, not the string — `equals()` is replaced by cosine similarity.

### Embedding Models
- Separate from chat models — purpose-built for converting text to vectors.
- Spring AI: `EmbeddingModel` interface. Providers: OpenAI, Mistral, Ollama, Voyage (Anthropic's embedding service).
- Input: a string. Output: `float[]`.

### Similarity Search
- **Cosine similarity**: angle between vectors — most common for text. Range: -1 to 1; 1 = identical.
- **Dot product**: fast, used when vectors are normalized.
- **Euclidean distance**: L2 norm — less common for NLP.
- Searching millions of vectors naively is O(n). Vector databases use approximate nearest neighbor (ANN) algorithms (HNSW, IVF) to make this fast.

### pgvector
- PostgreSQL extension that adds a `vector` column type and vector similarity operators.
- Perfect for teams already on Postgres — no new infrastructure.
- Add to schema: `CREATE EXTENSION vector;`
- Column: `embedding vector(1536)`
- Query: `ORDER BY embedding <=> $1 LIMIT 5` (cosine distance)
- Spring Data JPA + pgvector: Spring AI's `PgVectorStore` handles this out of the box.

### Qdrant (standalone vector DB)
- Purpose-built vector database — better for large scale (millions+ vectors) or multi-tenant SaaS.
- Spring AI: `QdrantVectorStore` bean — same interface as `PgVectorStore`.
- Use pgvector when you already have Postgres. Use Qdrant when vector search is a first-class feature.

---

## 3. Hands-On Task

### Goal: Build a document ingestion + semantic search pipeline

**Step 1 — Add dependencies**
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
```

**Step 2 — application.yml**
```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
  datasource:
    url: jdbc:postgresql://localhost:5432/upskill_db
    username: ${DB_USER}
    password: ${DB_PASS}
```

**Step 3 — Init pgvector schema**
```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE TABLE IF NOT EXISTS documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536)
);
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);
```

**Step 4 — Ingest documents**
```java
@Service
public class DocumentIngestionService {

    private final EmbeddingModel embeddingModel;
    private final VectorStore vectorStore;

    public DocumentIngestionService(EmbeddingModel embeddingModel, VectorStore vectorStore) {
        this.embeddingModel = embeddingModel;
        this.vectorStore = vectorStore;
    }

    public void ingest(List<String> texts) {
        List<Document> docs = texts.stream()
            .map(text -> new Document(text, Map.of("source", "manual")))
            .toList();
        vectorStore.add(docs);  // Spring AI handles embedding + storage
    }
}
```

**Step 5 — Semantic search**
```java
@Service
public class SemanticSearchService {

    private final VectorStore vectorStore;

    public SemanticSearchService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public List<Document> search(String query, int topK) {
        SearchRequest request = SearchRequest.query(query).withTopK(topK);
        return vectorStore.similaritySearch(request);
    }
}
```

**Step 6 — Experiments**
1. Ingest 10 tech FAQ entries. Search for "how to restart a service" — verify semantically similar entries surface even without exact keywords.
2. Try searching with a completely unrelated term — note the similarity scores drop.
3. Compare results with `topK=3` vs `topK=10`.
4. Print the raw embedding vector for a word: `embeddingModel.embed("Spring Boot")` — observe it's just a `float[]`.

---

## 4. Resources

- **[Spring AI Vector Stores](https://docs.spring.io/spring-ai/reference/api/vectordbs.html)** — Covers pgvector, Qdrant, Redis, and others with Spring AI config.
- **[pgvector GitHub](https://github.com/pgvector/pgvector)** — Extension docs, index types (HNSW vs IVF), and distance operators.
- **[Voyage AI Embeddings](https://docs.voyageai.com/)** — Anthropic's recommended embedding service, strong on retrieval tasks.

---

## 5. Trending Context

**Embeddings are the most underrated primitive in AI engineering.** In 2025, virtually every production AI feature — semantic search, RAG, duplicate detection, recommendation — is built on embeddings. The shift from keyword search (Elasticsearch BM25) to semantic search (vector similarity) is now mainstream. pgvector has become the go-to for teams already on Postgres, with Supabase and Neon both offering it managed. For new projects, choosing between pgvector and a standalone vector DB is now a standard architecture decision — like choosing between Redis and a dedicated queue.

---

## 6. Community Tip

**Chunk your documents before embedding.** Embedding a 10-page PDF as one vector loses nuance — the embedding model averages everything and specificity drops. Standard practice: split into chunks of ~512 tokens with a 50-token overlap, embed each chunk separately, store the parent document reference in metadata. This is called "chunking strategy" and getting it right is often the difference between a RAG system that works and one that hallucinates. Think of it like partitioning a Kafka topic: finer granularity = better parallelism and precision.

---

## 7. Tomorrow's Preview

**Day 5** will cover **RAG (Retrieval-Augmented Generation)** — combining vector search with LLM generation to answer questions grounded in your own documents. You'll build an end-to-end pipeline: ingest → retrieve → augment → generate.

---

*Built progressively. Each day connects to the next.*
