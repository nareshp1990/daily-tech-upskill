# Day 5 — RAG (Retrieval-Augmented Generation)
**Date:** 2026-03-12
**Phase:** 1 — Generative AI & Agentic AI
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**RAG — End-to-End Pipeline: Ingest → Retrieve → Augment → Generate**

Day 4 taught you how to store and search meaning with embeddings. Today you close the loop: use retrieved documents to **ground the LLM's answer in your own data**. RAG is the most important production pattern in AI engineering right now — it eliminates hallucinations for domain-specific queries without the cost and complexity of fine-tuning. Think of it as a search engine (Elasticsearch) wired into an LLM: you retrieve the relevant context, inject it into the prompt, and let the model synthesize the answer.

---

## 2. Key Concepts

### Why RAG?
- LLMs have a knowledge cutoff and know nothing about your private data.
- Fine-tuning is expensive and slow — impractical for frequently changing data.
- RAG gives the LLM the exact context it needs at query time, just-in-time.
- Analogy: closed-book exam (base LLM) vs. open-book exam with the right chapter bookmarked (RAG).

### RAG Pipeline — Two Phases

**Ingestion (offline):**
1. Load documents (PDF, DB rows, HTML, etc.)
2. Split into chunks (~512 tokens)
3. Embed each chunk via embedding model
4. Store chunks + embeddings in vector store

**Query (online):**
1. Embed the user's question
2. Similarity search → retrieve top-K relevant chunks
3. Inject chunks into the LLM prompt as context
4. LLM generates a grounded answer

### Prompt Augmentation Pattern
```
System: You are a helpful assistant. Answer ONLY using the provided context.
        If the answer is not in the context, say "I don't know."

Context:
{{chunk_1}}
{{chunk_2}}
{{chunk_3}}

User: {{user_question}}
```

### Naive RAG vs. Advanced RAG
- **Naive RAG**: embed → retrieve → generate. Simple, covers 80% of use cases.
- **HyDE (Hypothetical Document Embeddings)**: generate a hypothetical answer first, embed that for retrieval — improves recall for complex questions.
- **Re-ranking**: after vector search, run a cross-encoder re-ranker to re-score retrieved chunks by relevance before feeding to LLM.
- **Metadata filtering**: filter vector search by document type, date, or tenant ID before similarity search.

### Context Window Management
- Each retrieved chunk costs tokens. With `topK=5` and 512-token chunks, you spend ~2500 tokens on context.
- Budget your context window: leave room for the question, system prompt, and LLM response.
- Rule of thumb: retrieved context ≤ 60% of the model's context window.

---

## 3. Hands-On Task

### Goal: Build a document Q&A service using Spring AI's RAG support

**Step 1 — Spring AI QuestionAnswerAdvisor (built-in RAG)**
```java
@Configuration
public class RagConfig {

    @Bean
    public ChatClient ragChatClient(ChatClient.Builder builder, VectorStore vectorStore) {
        return builder
            .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore,
                SearchRequest.defaults().withTopK(5)))
            .build();
    }
}
```

**Step 2 — Q&A Service**
```java
@Service
public class DocumentQaService {

    private final ChatClient ragChatClient;

    public DocumentQaService(ChatClient ragChatClient) {
        this.ragChatClient = ragChatClient;
    }

    public String ask(String question) {
        return ragChatClient.prompt()
            .system("You are a helpful assistant. Answer ONLY using the provided context. " +
                    "If the answer is not in the context, say: 'I don't have that information.'")
            .user(question)
            .call()
            .content();
    }
}
```

**Step 3 — Document loader (ingest from classpath)**
```java
@Component
public class DocumentLoader implements ApplicationRunner {

    private final VectorStore vectorStore;
    private final ResourceLoader resourceLoader;

    public DocumentLoader(VectorStore vectorStore, ResourceLoader resourceLoader) {
        this.vectorStore = vectorStore;
        this.resourceLoader = resourceLoader;
    }

    @Override
    public void run(ApplicationArguments args) {
        Resource resource = resourceLoader.getResource("classpath:docs/faq.txt");
        var reader = new TokenTextSplitter();
        var loader = new TextReader(resource);
        List<Document> docs = reader.apply(loader.get());
        vectorStore.add(docs);
    }
}
```

**Step 4 — REST endpoint**
```java
@RestController
@RequestMapping("/api/qa")
public class QaController {

    private final DocumentQaService qaService;

    public QaController(DocumentQaService qaService) {
        this.qaService = qaService;
    }

    @PostMapping
    public Map<String, String> ask(@RequestBody Map<String, String> body) {
        String answer = qaService.ask(body.get("question"));
        return Map.of("answer", answer);
    }
}
```

**Step 5 — Experiments**
1. Load 5–10 FAQ entries into `resources/docs/faq.txt`. Ask questions with exact phrasing → verify correct answers.
2. Ask a question whose answer is NOT in the docs — verify the model says "I don't have that information" rather than hallucinating.
3. Log the retrieved chunks before passing to LLM — observe what context the model actually sees.
4. Try `topK=1` vs `topK=5` — observe how answer quality changes with more/less context.
5. Add a metadata filter: `SearchRequest.defaults().withTopK(5).withFilterExpression("source == 'faq'")`.

---

## 4. Resources

- **[Spring AI RAG Docs](https://docs.spring.io/spring-ai/reference/api/retrieval-augmented-generation.html)** — Covers `QuestionAnswerAdvisor`, `VectorStore`, and the full RAG pipeline in Spring AI.
- **[RAG Survey Paper (arxiv 2312.10997)](https://arxiv.org/abs/2312.10997)** — Comprehensive survey: naive RAG, advanced RAG, modular RAG. Read the abstract + section 3.
- **[LangChain4j RAG Guide](https://docs.langchain4j.dev/tutorials/rag)** — Alternative Java RAG library; useful for understanding different chunking and retrieval strategies.

---

## 5. Trending Context

**RAG is now table stakes for enterprise AI.** In 2025, every major enterprise AI deployment — internal knowledge bases, customer support bots, code assistants — uses RAG rather than fine-tuning for domain knowledge. The pattern has matured from research to production: Spring AI, LangChain4j, and LlamaIndex all have production-grade RAG pipelines. The current frontier is "agentic RAG" — where an agent decides *which* retrieval strategy to use, or iteratively refines retrieval based on initial results. You'll see that on Day 6.

---

## 6. Community Tip

**Evaluate your RAG pipeline with a test set.** A common mistake is building a RAG system, testing it manually with 3 questions, and calling it done. In production, use a small golden dataset (20–50 question/answer pairs) and measure retrieval recall (did the right chunk get retrieved?) separately from generation accuracy (did the LLM answer correctly?). Tools like **RAGAS** (Python) or a simple JUnit parameterized test work well. If retrieval recall is low, fix chunking or embedding. If generation accuracy is low despite good retrieval, fix your prompt. Measure both independently — same discipline as measuring P99 latency vs. error rate separately in a microservice.

---

## 7. Tomorrow's Preview

**Day 6** will cover **LLM Tool Use & Function Calling** — giving the LLM the ability to call your code (APIs, databases, calculators) to get real-time information and take actions. This is the foundation of agentic AI: an LLM that can reason about *which tool to call* and *what arguments to pass*.

---

*Built progressively. Each day connects to the next.*
