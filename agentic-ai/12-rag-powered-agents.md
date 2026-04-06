# Agentic AI — Day 12: RAG-Powered Agents
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Combining Retrieval-Augmented Generation with Agentic Reasoning**

RAG alone is passive: query → retrieve → generate. An agentic RAG system is active: the agent decides when to search, what to search for, evaluates results, and iterates if the answer is incomplete. Today you build a RAG-powered agent that can query a knowledge base, decide if the results are sufficient, reformulate queries, and combine information from multiple retrievals into a comprehensive answer.

---

## 2. Agentic RAG vs Naive RAG

```
Naive RAG:
  User question → embed → retrieve top-5 → stuff into prompt → generate answer
  Problem: What if the first retrieval misses relevant context?
           What if the question needs information from multiple documents?

Agentic RAG:
  User question → Agent reasons: "I need to search for X"
                → Retrieves results → evaluates: "Not enough, let me also search Y"
                → Retrieves more → synthesises both results
                → Agent: "Now I have enough to answer fully"
```

---

## 3. Implementation: Python (LangGraph + Chroma)

```python
from langchain_anthropic import ChatAnthropic
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from pathlib import Path

# --- Setup vector store ---
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(persist_directory="./knowledge_base", embedding_function=embeddings)

# Ingest documents
def ingest_documents(docs_dir: str):
    splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
    for filepath in Path(docs_dir).rglob("*.md"):
        text = filepath.read_text()
        chunks = splitter.split_text(text)
        metadatas = [{"source": str(filepath), "chunk": i} for i in range(len(chunks))]
        vectorstore.add_texts(chunks, metadatas=metadatas)

# --- RAG Tools ---
@tool
def search_knowledge_base(query: str) -> str:
    """Search the internal knowledge base for relevant documentation.
    Use this when the user asks about company policies, architecture decisions,
    API documentation, or internal procedures.
    Returns the top 5 most relevant document chunks with source references."""
    results = vectorstore.similarity_search_with_relevance_scores(query, k=5)
    if not results:
        return "No relevant documents found."
    return "\n\n---\n\n".join(
        f"[Source: {doc.metadata['source']}] (relevance: {score:.2f})\n{doc.page_content}"
        for doc, score in results
    )

@tool
def search_by_metadata(source_file: str) -> str:
    """Retrieve all chunks from a specific source document.
    Use this when you need the full context from a document you've partially seen."""
    results = vectorstore.similarity_search("", k=20,
        filter={"source": source_file})
    return "\n\n".join(doc.page_content for doc in results)

@tool
def list_available_documents() -> str:
    """List all documents available in the knowledge base.
    Use this to understand what documentation is available before searching."""
    all_docs = vectorstore.get()
    sources = set(m["source"] for m in all_docs["metadatas"])
    return "Available documents:\n" + "\n".join(f"- {s}" for s in sorted(sources))

# --- Build Agent ---
model = ChatAnthropic(model="claude-sonnet-4-20250514")

rag_agent = create_react_agent(
    model,
    tools=[search_knowledge_base, search_by_metadata, list_available_documents],
    prompt="""You are a knowledgeable assistant with access to a company knowledge base.

    When answering questions:
    1. ALWAYS search the knowledge base first — don't rely on your training data.
    2. If the first search doesn't fully answer the question, search with different terms.
    3. Cite your sources: mention which document the information came from.
    4. If the knowledge base doesn't have the answer, say so clearly.
    5. Combine information from multiple documents when needed."""
)
```

---

## 4. Implementation: Java (Spring AI + pgvector)

```java
@Service
@RequiredArgsConstructor
public class RagAgentService {

    private final ChatClient chatClient;
    private final VectorStore vectorStore;

    public String ask(String question) {
        return chatClient.prompt()
            .system("""You have access to a knowledge base via tools.
                Always search before answering. Cite sources. If multiple
                searches are needed, perform them.""")
            .user(question)
            .tools(new KnowledgeBaseTools(vectorStore))
            .call()
            .content();
    }
}

@RequiredArgsConstructor
class KnowledgeBaseTools {

    private final VectorStore vectorStore;

    @Tool("Search the knowledge base for documents relevant to the query")
    public String searchKnowledgeBase(
            @ToolParam("Search query — be specific for better results") String query) {
        List<Document> results = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );
        return results.stream()
            .map(doc -> "[" + doc.getMetadata().get("source") + "]\n" + doc.getText())
            .collect(Collectors.joining("\n\n---\n\n"));
    }

    @Tool("Search with metadata filter — find documents from a specific source or category")
    public String searchWithFilter(
            @ToolParam("Search query") String query,
            @ToolParam("Source file path or category to filter by") String source) {
        List<Document> results = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(10)
                .filterExpression("source == '" + source + "'")
                .build()
        );
        return results.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
    }
}
```

---

## 5. Self-Reflective RAG (Query Refinement)

```python
from pydantic import BaseModel

class RetrievalEvaluation(BaseModel):
    is_sufficient: bool
    missing_information: str
    suggested_query: str

def evaluate_retrieval(question: str, retrieved_context: str) -> RetrievalEvaluation:
    """LLM evaluates if retrieved context is sufficient to answer the question."""
    model = ChatAnthropic(model="claude-sonnet-4-20250514")
    return model.with_structured_output(RetrievalEvaluation).invoke([
        ("system", """Evaluate if the retrieved context is sufficient to answer the question.
        If not, suggest a refined search query to find the missing information."""),
        ("user", f"Question: {question}\n\nRetrieved context:\n{retrieved_context}")
    ])
```

---

## 6. Hands-On Exercise

1. Create a `knowledge_base/` directory with 5-10 Markdown files (use your existing learning plans!).
2. Ingest them into Chroma vector store.
3. Build the RAG agent with `search_knowledge_base` and `list_available_documents` tools.
4. Test: "What did I learn about circuit breakers?" — verify it cites the correct source.
5. Test multi-hop: "Compare the caching strategies from Day 18 with the CDN caching from Day 28."
6. Implement the self-reflective RAG evaluation and observe query refinement.

---

## 7. Key Takeaways

1. **Agentic RAG > Naive RAG** — the agent decides when and what to search, not just a fixed pipeline.
2. **Multiple retrieval tools give flexibility** — semantic search, metadata filter, document listing.
3. **Self-reflection on retrieval quality catches gaps** — "Is this enough to answer? If not, search again."
4. **Always cite sources** — traceability is critical for trust and debugging.

---

*Day 12. RAG gives the agent knowledge. Agency gives it the judgment to know what it doesn't know.*
