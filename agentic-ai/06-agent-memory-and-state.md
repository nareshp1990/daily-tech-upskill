# Agentic AI — Day 6: Agent Memory & State Management
**Track:** Agentic AI (Implementation-Focused)
**Time Target:** 1–2 hours

---

## 1. Today's Focus
**Memory Systems for Agents — Short-Term, Long-Term, and Semantic Memory**

Without memory, an agent is a goldfish with tools. It answers your question and forgets everything. Real agents need: (1) **conversation memory** — remember what you said 5 messages ago, (2) **long-term memory** — remember you prefer concise answers across sessions, and (3) **semantic memory** — retrieve relevant knowledge from a large corpus. Today you implement all three in both Java and Python.

---

## 2. Memory Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Agent Memory                       │
│                                                      │
│  ┌──────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Short-Term   │  │ Long-Term│  │   Semantic     │  │
│  │ (Conversation│  │ (Cross-  │  │   (Knowledge   │  │
│  │  Window)     │  │  Session)│  │    Base/RAG)   │  │
│  ├──────────────┤  ├──────────┤  ├───────────────┤  │
│  │ Last N msgs  │  │ User     │  │ Vector store  │  │
│  │ or token     │  │ facts    │  │ embeddings    │  │
│  │ window       │  │ prefs    │  │ retrieval     │  │
│  ├──────────────┤  ├──────────┤  ├───────────────┤  │
│  │ InMemory /   │  │ Redis /  │  │ pgvector /    │  │
│  │ Redis        │  │ Postgres │  │ Pinecone      │  │
│  └──────────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 3. Short-Term Memory (Conversation Window)

### Java (Spring AI)

```java
@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemory chatMemory() {
        return new InMemoryChatMemory();  // Or RedisChatMemory for production
    }

    @Bean
    public ChatClient agentWithMemory(ChatClient.Builder builder, ChatMemory memory) {
        return builder
            .defaultSystem("You are a helpful assistant.")
            .defaultAdvisors(
                new MessageChatMemoryAdvisor(memory, 20)  // Last 20 messages
            )
            .build();
    }
}

// Usage — memory is automatic per conversation
@PostMapping("/chat")
public String chat(@RequestBody ChatRequest req) {
    return agentWithMemory.prompt()
        .advisors(a -> a.param("chat_memory_conversation_id", req.conversationId()))
        .user(req.message())
        .call()
        .content();
}
```

### Python (LangGraph with Checkpointing)

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import create_react_agent

memory = MemorySaver()
agent = create_react_agent(model, tools=tools, checkpointer=memory)

# Conversation 1
config = {"configurable": {"thread_id": "user-123-conv-1"}}
agent.invoke({"messages": [("user", "My name is Naresh")]}, config)
# Later in same conversation...
result = agent.invoke({"messages": [("user", "What's my name?")]}, config)
# → "Your name is Naresh" ✅

# Different conversation
config2 = {"configurable": {"thread_id": "user-123-conv-2"}}
result = agent.invoke({"messages": [("user", "What's my name?")]}, config2)
# → "I don't know your name" ✅ (isolated)
```

### Production: Redis-Backed Memory (Python)

```python
from langgraph.checkpoint.redis import RedisSaver

redis_saver = RedisSaver(url="redis://localhost:6379")
agent = create_react_agent(model, tools=tools, checkpointer=redis_saver)
# Now state persists across process restarts
```

---

## 4. Long-Term Memory (Cross-Session Facts)

### Fact Extraction Agent

```python
from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel

class UserFact(BaseModel):
    fact: str
    category: str  # preference, background, context
    confidence: float

class ExtractedFacts(BaseModel):
    facts: list[UserFact]

model = ChatAnthropic(model="claude-sonnet-4-20250514")

def extract_facts(conversation: str) -> list[UserFact]:
    """Extract memorable facts from a conversation."""
    response = model.with_structured_output(ExtractedFacts).invoke([
        ("system", """Extract user facts worth remembering across sessions.
        Categories: preference (likes/dislikes), background (role, experience),
        context (current project, goals). Only extract non-trivial facts.
        Confidence: 0.0-1.0 based on how certain the fact is."""),
        ("user", f"Conversation:\n{conversation}")
    ])
    return response.facts

# Store facts
import json
def save_user_facts(user_id: str, facts: list[UserFact]):
    """Persist facts to Redis with user namespace."""
    for fact in facts:
        redis.hset(f"user_facts:{user_id}", fact.fact, json.dumps({
            "category": fact.category,
            "confidence": fact.confidence,
            "created_at": datetime.now().isoformat()
        }))

def load_user_facts(user_id: str) -> str:
    """Load user facts as context for the agent."""
    facts = redis.hgetall(f"user_facts:{user_id}")
    if not facts:
        return "No previous context about this user."
    return "Known facts about this user:\n" + "\n".join(
        f"- {fact}" for fact in facts.keys()
    )
```

### Java — Long-Term Memory Service

```java
@Service
@RequiredArgsConstructor
public class LongTermMemoryService {

    private final RedisTemplate<String, String> redis;
    private final ChatClient factExtractor;

    public void extractAndStoreFacts(String userId, List<Message> conversation) {
        String convText = conversation.stream()
            .map(m -> m.getMessageType() + ": " + m.getContent())
            .collect(Collectors.joining("\n"));

        String facts = factExtractor.prompt()
            .system("Extract key facts about the user from this conversation. " +
                    "Return as JSON array of {fact, category, confidence}.")
            .user(convText)
            .call()
            .content();

        // Store each fact
        List<Map<String, String>> parsed = objectMapper.readValue(facts, new TypeReference<>() {});
        for (Map<String, String> fact : parsed) {
            redis.opsForHash().put("user_facts:" + userId,
                fact.get("fact"), objectMapper.writeValueAsString(fact));
        }
    }

    public String getUserContext(String userId) {
        Map<Object, Object> facts = redis.opsForHash().entries("user_facts:" + userId);
        if (facts.isEmpty()) return "No previous context.";
        return "Known facts:\n" + facts.keySet().stream()
            .map(Object::toString)
            .collect(Collectors.joining("\n- ", "- ", ""));
    }
}
```

---

## 5. Semantic Memory (RAG Integration)

```python
from langchain_anthropic import ChatAnthropic
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain_core.tools import tool

# Initialise vector store
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(persist_directory="./chroma_db", embedding_function=embeddings)

@tool
def search_knowledge_base(query: str) -> str:
    """Search the company knowledge base for relevant information.
    Use this when the user asks about company policies, procedures, or documentation.
    Returns the top 3 most relevant documents."""
    results = vectorstore.similarity_search(query, k=3)
    return "\n\n---\n\n".join(
        f"Source: {doc.metadata.get('source', 'unknown')}\n{doc.page_content}"
        for doc in results
    )

# Ingest documents
from langchain_community.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter

loader = DirectoryLoader("./docs", glob="**/*.md")
docs = loader.load()
splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
chunks = splitter.split_documents(docs)
vectorstore.add_documents(chunks)
```

---

## 6. Combining All Memory Types

```python
def create_agent_with_full_memory(user_id: str):
    """Create an agent with short-term, long-term, and semantic memory."""

    # Load long-term user context
    user_context = load_user_facts(user_id)

    system_prompt = f"""You are a helpful assistant.

    {user_context}

    Use the knowledge base tool when the user asks about documentation or policies.
    Remember what the user tells you during this conversation."""

    agent = create_react_agent(
        model=ChatAnthropic(model="claude-sonnet-4-20250514"),
        tools=[search_knowledge_base, get_current_time, calculate],
        prompt=system_prompt,
        checkpointer=redis_saver  # Short-term: conversation persistence
    )

    return agent

# After conversation ends, extract and store new facts
async def on_conversation_end(user_id: str, messages: list):
    new_facts = extract_facts(format_messages(messages))
    save_user_facts(user_id, new_facts)
```

---

## 7. Hands-On Exercise

1. Implement conversation memory in your Spring AI agent (from Day 2) using `MessageChatMemoryAdvisor`.
2. Test multi-turn: "My budget is $2000" → "Am I over budget?" (agent should remember the $2000).
3. Implement the `extract_facts` function in Python and test with sample conversations.
4. Add a vector store (Chroma) and ingest 5 markdown documents as a knowledge base.
5. Build the combined memory agent and test: user context + conversation + RAG in one agent.

---

## 8. Key Takeaways

1. **Short-term memory is table stakes** — every production agent needs conversation persistence.
2. **Long-term memory makes agents personal** — extracted facts create continuity across sessions.
3. **Semantic memory (RAG) gives agents knowledge** — without it, agents only know what the LLM was trained on.
4. **Memory has cost** — more messages = more tokens = more latency + cost; window management is essential.

---

*Day 6. Memory is what separates a tool from a colleague — it remembers, it learns, it adapts.*
