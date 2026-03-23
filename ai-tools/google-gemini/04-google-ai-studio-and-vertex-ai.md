# 04 — Google AI Studio and Vertex AI

## Table of Contents

1. [Google AI Studio — Deep Dive](#1-google-ai-studio--deep-dive)
2. [Vertex AI — Enterprise Platform](#2-vertex-ai--enterprise-platform)
3. [Model Tuning](#3-model-tuning)
4. [Evaluation Tools](#4-evaluation-tools)
5. [Vertex AI Agent Builder](#5-vertex-ai-agent-builder)
6. [Grounding with Google Search and Enterprise Data](#6-grounding-with-google-search-and-enterprise-data)
7. [RAG with Vertex AI Search](#7-rag-with-vertex-ai-search)
8. [Practical Workflow: Prototype to Production](#8-practical-workflow-prototype-to-production)
9. [Try This Exercises](#9-try-this-exercises)

---

## 1. Google AI Studio — Deep Dive

**URL:** https://aistudio.google.com

Google AI Studio is the free, web-based IDE for Gemini models. It is where you
prototype prompts before writing any code.

### Interface Walkthrough

```
┌─────────────────────────────────────────────────────────────┐
│  Google AI Studio                                           │
├─────────────┬───────────────────────────────────────────────┤
│             │                                               │
│  Sidebar:   │  Main Panel:                                  │
│  - Prompts  │  ┌─────────────────────────────────────────┐  │
│  - Models   │  │ System Instructions                     │  │
│  - API Keys │  │ [Text area for system prompt]            │  │
│  - Tuning   │  ├─────────────────────────────────────────┤  │
│  - Files    │  │ Chat / Structured / Freeform             │  │
│             │  │                                         │  │
│             │  │ User: [your message]                    │  │
│             │  │ Model: [response]                       │  │
│             │  │                                         │  │
│             │  ├─────────────────────────────────────────┤  │
│             │  │ Settings:                               │  │
│             │  │ Model: [gemini-2.5-pro ▼]              │  │
│             │  │ Temp: [0.7] Top-P: [0.95] Top-K: [40] │  │
│             │  │ Max output: [8192] Safety: [default]    │  │
│             │  │ Tools: [ ] Google Search  [ ] Code exec │  │
│             │  └─────────────────────────────────────────┘  │
│             │                                               │
│             │  [Run] [Get Code ▼] [Save Prompt]            │
└─────────────┴───────────────────────────────────────────────┘
```

### Prompt Modes

| Mode | Use Case |
|------|----------|
| **Chat** | Multi-turn conversation. Best for assistant-style prompts. |
| **Freeform** | Single input/output. Best for completion, generation, transformation. |
| **Structured** | Few-shot with examples. Build input/output pairs visually. |

### Structured Prompts (Few-Shot)

Structured mode lets you build few-shot examples in a table format:

```
System instruction: "You are a Spring Boot error classifier."

| Input (Error Message) | Output (Classification) |
|-----------------------|------------------------|
| NullPointerException in UserService.getUser | BUG: null-safety |
| Connection refused: MySQL port 3306 | INFRA: database-connection |
| 403 Forbidden on /api/admin | SECURITY: authorization |

New input: "OutOfMemoryError: Java heap space in OrderBatchProcessor"
→ Model outputs: "PERFORMANCE: memory-leak"
```

This is extremely useful for building classifiers and extractors that you then
export as code for your Spring Boot service.

### File Management

AI Studio supports uploading files that persist across sessions:

| Type | Max Size | Use |
|------|----------|-----|
| Text | 2 GB | Codebases, documentation |
| Image | 20 MB | Screenshots, diagrams |
| Audio | 2 GB | Meeting recordings |
| Video | 2 GB | Screencasts, demos |
| PDF | 2 GB | Specifications, papers |

**Workflow for codebase analysis:**
1. Upload your concatenated source code as a text file
2. Reference it in prompts: "Analyze the uploaded codebase..."
3. Ask multiple questions without re-uploading

### Token Counter

Use the built-in token counter to estimate costs:
1. Paste your prompt
2. Check "Token count" at the bottom
3. Multiply by pricing rates to estimate cost

### Code Export

Click "Get Code" to export your prompt as:

| Language | Library |
|----------|---------|
| Python | `google-genai` |
| JavaScript | `@google/genai` |
| Kotlin | Vertex AI SDK |
| Swift | `google-generative-ai-swift` |
| Dart | `google_generative_ai` |
| cURL | REST API |
| Java | Not directly available; use cURL and adapt |

> **Java developers:** Export as cURL, then translate to Spring WebClient or
> OkHttp. Or export as Kotlin and adapt — the Vertex AI SDK is the same.

### API Key Management

| Feature | Details |
|---------|---------|
| Create keys | Up to 100 keys per project |
| Restrict keys | Limit to specific APIs and IPs |
| Rotate keys | Create new, migrate, delete old |
| Monitor usage | View usage per key in Cloud Console |

**Security best practices:**
```bash
# Store in environment variable
export GEMINI_API_KEY="your-key-here"

# Spring Boot: use application.yml with env variable
# gemini.api-key: ${GEMINI_API_KEY}

# Never commit to Git — add to .gitignore
echo "*.env" >> .gitignore
echo "application-local.yml" >> .gitignore
```

---

## 2. Vertex AI — Enterprise Platform

**URL:** https://console.cloud.google.com/vertex-ai

Vertex AI is Google Cloud's managed ML/AI platform. For Gemini, it provides
enterprise features on top of the same models available in AI Studio.

### AI Studio vs Vertex AI

| Feature | AI Studio | Vertex AI |
|---------|-----------|-----------|
| Target audience | Developers, prototyping | Enterprise, production |
| Authentication | API key | OAuth, service accounts, IAM |
| Pricing | Free tier + pay-as-you-go | Pay-as-you-go (no free tier) |
| VPC / Private endpoints | No | Yes |
| Data residency | No control | Regional endpoints |
| Model tuning | Limited | Full (supervised, RLHF) |
| Evaluation | Basic | Comprehensive (AutoSxS, custom metrics) |
| Agent Builder | No | Yes |
| RAG Engine | No | Yes |
| Batch prediction | No | Yes |
| Model monitoring | No | Yes |
| SLA | No | Yes (99.9%) |
| CMEK encryption | No | Yes |
| Audit logging | No | Yes (Cloud Audit Logs) |

### Setting Up Vertex AI

```bash
# Enable required APIs
gcloud services enable aiplatform.googleapis.com
gcloud services enable compute.googleapis.com

# Create a service account for your Spring Boot service
gcloud iam service-accounts create gemini-service \
  --display-name="Gemini Service Account"

# Grant necessary roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:gemini-service@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

# Create and download a key (for local dev)
gcloud iam service-accounts keys create gemini-key.json \
  --iam-account=gemini-service@${PROJECT_ID}.iam.gserviceaccount.com

# Set environment variable
export GOOGLE_APPLICATION_CREDENTIALS="gemini-key.json"
```

### Model Garden

The Vertex AI Model Garden provides access to:

| Category | Examples |
|----------|---------|
| Google models | Gemini 2.5 Pro, Flash, Imagen, Chirp |
| Open models | Llama 3.1, Mistral, Gemma |
| Partner models | Anthropic Claude (on Vertex) |
| Task-specific | Code models, embedding models, moderation models |

> **Key insight:** You can access Claude models through Vertex AI, using the same
> infrastructure, billing, and IAM. This avoids managing separate Anthropic accounts.

### Endpoints and Deployment

```
Two ways to use Gemini on Vertex AI:

1. Publisher Endpoints (managed, no deployment needed)
   → Just call the API with your project ID
   → Google manages scaling, availability
   → Pay per token

2. Custom Endpoints (for fine-tuned models)
   → Deploy your tuned model to an endpoint
   → Configure machine type, scaling
   → Pay for compute + tokens
```

### Batch Prediction

For processing large datasets without real-time requirements:

```bash
# Create a JSONL file with requests
cat > batch-input.jsonl << 'EOF'
{"request": {"contents": [{"role": "user", "parts": [{"text": "Classify: NullPointerException in UserService"}]}]}}
{"request": {"contents": [{"role": "user", "parts": [{"text": "Classify: Connection timeout to Redis"}]}]}}
{"request": {"contents": [{"role": "user", "parts": [{"text": "Classify: 401 Unauthorized on /api/admin"}]}]}}
EOF

# Upload to GCS
gsutil cp batch-input.jsonl gs://my-bucket/batch/

# Submit batch job
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/us-central1/batchPredictionJobs" \
  -d '{
    "displayName": "error-classification-batch",
    "model": "publishers/google/models/gemini-2.5-flash",
    "inputConfig": {
      "instancesFormat": "jsonl",
      "gcsSource": {"uris": ["gs://my-bucket/batch/batch-input.jsonl"]}
    },
    "outputConfig": {
      "predictionsFormat": "jsonl",
      "gcsDestination": {"outputUriPrefix": "gs://my-bucket/batch/output/"}
    }
  }'
```

> **Use case:** Nightly batch processing of customer support tickets, log classification,
> code quality analysis across entire repositories.

---

## 3. Model Tuning

### When to Tune vs When to Prompt

| Approach | When | Cost | Effort |
|----------|------|------|--------|
| Prompt engineering | Most cases | $0 | Low |
| Few-shot examples | Need consistent format | $0 | Low |
| Supervised fine-tuning | Domain-specific vocab, style | $$ | Medium |
| RLHF | Align to specific preferences | $$$ | High |

> **Rule of thumb:** Start with prompt engineering. Only tune if you have 100+ examples
> and the base model consistently fails at your specific task format.

### Supervised Fine-Tuning on Vertex AI

**Step 1: Prepare training data (JSONL format):**
```json
{
  "systemInstruction": "You are a code review assistant for Java Spring Boot applications.",
  "contents": [
    {
      "role": "user",
      "parts": [{"text": "Review: @GetMapping public User getUser(@PathVariable Long id) { return repo.findById(id).get(); }"}]
    },
    {
      "role": "model",
      "parts": [{"text": "{\"issues\": [{\"severity\": \"HIGH\", \"description\": \"Using .get() on Optional without check — throws NoSuchElementException if user not found\", \"fix\": \"Use .orElseThrow(() -> new UserNotFoundException(id))\"}]}"}]
    }
  ]
}
```

**Step 2: Upload data to GCS:**
```bash
gsutil cp training-data.jsonl gs://my-bucket/tuning/training-data.jsonl
gsutil cp validation-data.jsonl gs://my-bucket/tuning/validation-data.jsonl
```

**Step 3: Create tuning job:**
```bash
curl -X POST \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type: application/json" \
  "https://us-central1-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/us-central1/tuningJobs" \
  -d '{
    "baseModel": "gemini-2.0-flash",
    "supervisedTuningSpec": {
      "trainingDatasetUri": "gs://my-bucket/tuning/training-data.jsonl",
      "validationDatasetUri": "gs://my-bucket/tuning/validation-data.jsonl",
      "hyperParameters": {
        "epochCount": 3,
        "learningRateMultiplier": 1.0
      }
    },
    "tunedModelDisplayName": "code-review-model-v1"
  }'
```

**Step 4: Use the tuned model:**
```yaml
# application.yml
spring:
  ai:
    vertex-ai:
      gemini:
        project-id: ${GCP_PROJECT_ID}
        location: us-central1
        chat:
          options:
            model: projects/${GCP_PROJECT_ID}/locations/us-central1/endpoints/${ENDPOINT_ID}
```

### Tuning Guidelines

| Guideline | Details |
|-----------|---------|
| Minimum examples | 100 (recommended 500+) |
| Data quality | Consistent format, correct outputs, diverse inputs |
| Epochs | Start with 3, increase if under-fitting |
| Validation split | 10-20% of data |
| Base model | Use Flash for tuning (cheaper, faster) |
| Evaluation | Always compare tuned vs base model |

---

## 4. Evaluation Tools

### Vertex AI Model Evaluation

Vertex provides built-in evaluation for comparing models and measuring quality.

**Evaluation metrics available:**

| Metric | Type | Use |
|--------|------|-----|
| BLEU | Automated | Translation quality |
| ROUGE | Automated | Summarization quality |
| Exact match | Automated | Classification accuracy |
| AutoSxS (Side-by-Side) | LLM-as-judge | Compare two model outputs |
| Coherence | LLM-as-judge | Logical flow of response |
| Fluency | LLM-as-judge | Language quality |
| Groundedness | LLM-as-judge | Factual accuracy |
| Safety | Automated | Harmful content detection |
| Custom | User-defined | Domain-specific criteria |

### AutoSxS Evaluation

AutoSxS lets you compare two models (or a base vs tuned model) side by side
using an LLM judge:

```python
# Python example (adapt concepts for your Java pipeline)
from vertexai.evaluation import AutoSxSEvaluation

eval_task = AutoSxSEvaluation(
    evaluation_dataset="gs://my-bucket/eval/test-data.jsonl",
    model_a="gemini-2.0-flash",
    model_b="projects/my-project/locations/us-central1/endpoints/tuned-model",
    judge_model="gemini-2.5-pro",
    criteria="Which response provides more accurate and actionable code review feedback?",
)

results = eval_task.run()
print(f"Model A wins: {results.model_a_win_rate}")
print(f"Model B wins: {results.model_b_win_rate}")
print(f"Ties: {results.tie_rate}")
```

### Building an Evaluation Pipeline in Java

```java
@Service
public class ModelEvaluationService {

    private final ChatClient baseModelClient;
    private final ChatClient tunedModelClient;
    private final ChatClient judgeClient;

    // Use Gemini 2.5 Pro as the judge
    public EvaluationResult evaluate(List<TestCase> testCases) {
        List<ComparisonResult> results = testCases.stream()
            .map(tc -> {
                String baseResponse = baseModelClient.prompt()
                    .user(tc.input()).call().content();
                String tunedResponse = tunedModelClient.prompt()
                    .user(tc.input()).call().content();

                String judgment = judgeClient.prompt()
                    .system("You are evaluating two code review responses. " +
                            "Return JSON with 'winner' (A/B/TIE) and 'reason'.")
                    .user(String.format("""
                        Input: %s

                        Response A (base model):
                        %s

                        Response B (tuned model):
                        %s

                        Which response is more accurate, actionable, and follows
                        Java/Spring Boot best practices?
                        """, tc.input(), baseResponse, tunedResponse))
                    .call()
                    .content();

                return parseJudgment(judgment);
            })
            .toList();

        return aggregateResults(results);
    }
}
```

---

## 5. Vertex AI Agent Builder

Agent Builder lets you create AI agents that combine Gemini with tools, data stores,
and conversation management — without writing orchestration code.

### Agent Types

| Type | Use Case |
|------|----------|
| **Conversational agent** | Customer support, internal helpdesk |
| **Search agent** | Enterprise search across documents |
| **Code agent** | Code generation and analysis |
| **Custom agent** | Any workflow with tools and reasoning |

### Creating an Agent

```
Step 1: Go to Vertex AI → Agent Builder → Create Agent
Step 2: Configure:
  - Name: "backend-dev-assistant"
  - Model: Gemini 2.5 Flash
  - System instruction: "You are a backend development assistant..."
  - Tools: define data stores, APIs, or code execution
Step 3: Add data stores (for RAG):
  - Upload documentation, runbooks, architecture docs
  - Connect to Cloud Storage, BigQuery, or websites
Step 4: Test in the console
Step 5: Deploy via API or embed in your app
```

### Agent with Custom Tools (API)

```json
{
  "displayName": "deployment-assistant",
  "defaultLanguageCode": "en",
  "generativeAgentSettings": {
    "agent": {
      "model": "gemini-2.5-flash",
      "systemInstruction": "You help engineers with Kubernetes deployments...",
      "tools": [
        {
          "openApiTool": {
            "name": "k8s-api",
            "description": "Interact with the Kubernetes API",
            "schema": {
              "type": "openapi",
              "uri": "gs://my-bucket/openapi/k8s-api.yaml"
            }
          }
        },
        {
          "datastoreTool": {
            "name": "runbooks",
            "description": "Search deployment runbooks",
            "datastoreConnections": [{
              "datastore": "projects/my-project/locations/global/dataStores/runbooks"
            }]
          }
        }
      ]
    }
  }
}
```

> **Azure parallel:** This is similar to Azure AI Agent Service / Azure Bot Service
> combined with Azure AI Search. If you use Azure, the concepts transfer — you are
> building a RAG agent with tools.

---

## 6. Grounding with Google Search and Enterprise Data

### Google Search Grounding

Ground model responses in real-time web search results:

```json
{
  "contents": [{
    "parts": [{"text": "What is the latest LTS version of Spring Boot and its key features?"}]
  }],
  "tools": [{
    "googleSearch": {}
  }]
}
```

**When to use:**
- Checking current library versions
- Finding recent CVEs or security advisories
- Verifying API changes or deprecations
- Getting up-to-date documentation links

### Enterprise Data Grounding (Vertex AI)

Ground responses in your own data:

```
Options:
1. Vertex AI Search data store → upload documents, web pages, or structured data
2. Cloud Storage → point to a bucket with your docs
3. BigQuery → query your data warehouse
4. URL list → crawl and index specific websites
```

**Example: Ground in your team's runbooks:**
```bash
# Create a data store
gcloud ai data-stores create runbooks-store \
  --location=global \
  --display-name="Team Runbooks" \
  --type=UNSTRUCTURED

# Import documents
gcloud ai data-stores import-documents runbooks-store \
  --location=global \
  --source=gs://my-bucket/runbooks/ \
  --reconciliation-mode=FULL
```

Then reference the data store in API calls:
```json
{
  "tools": [{
    "retrieval": {
      "vertexAiSearch": {
        "datastore": "projects/my-project/locations/global/dataStores/runbooks-store"
      }
    }
  }]
}
```

---

## 7. RAG with Vertex AI Search

Vertex AI Search provides a managed RAG pipeline — no need to manage your own
vector database or retrieval logic.

### Architecture

```
Your Documents ──> Vertex AI Search (ingest, chunk, embed, index)
                                    │
User Query ──> Gemini ──> Vertex AI Search (retrieve relevant chunks)
                                    │
                          Gemini (augment + generate response with citations)
```

### Setting Up RAG

**Step 1: Create a search app:**
```bash
gcloud ai search-apps create my-docs-search \
  --location=global \
  --display-name="Internal Documentation Search"
```

**Step 2: Create and populate a data store:**
```bash
# Unstructured (PDFs, docs, HTML)
gcloud ai data-stores create internal-docs \
  --location=global \
  --display-name="Internal Docs" \
  --type=UNSTRUCTURED

# Import from GCS
gcloud ai data-stores import-documents internal-docs \
  --location=global \
  --source=gs://my-bucket/docs/
```

**Step 3: Use in API calls:**
```java
// Spring AI with Vertex AI Search retrieval
@Service
public class DocSearchService {

    private final ChatClient chatClient;

    public DocSearchService(ChatClient.Builder builder) {
        this.chatClient = builder
            .defaultSystem("Answer questions using the provided documentation. " +
                           "Always cite your sources.")
            .build();
    }

    public String searchAndAnswer(String question) {
        // Vertex AI Search retrieval is configured in application.yml
        // or via VertexAiGeminiChatOptions with tools
        return chatClient.prompt()
            .user(question)
            .call()
            .content();
    }
}
```

### RAG vs Direct Context

| Approach | When | Pros | Cons |
|----------|------|------|------|
| Direct context (paste in prompt) | < 100K tokens | Simple, deterministic | Context window limit, expensive |
| Context caching | Repeated queries on same docs | 75% cheaper, simple | Still limited by context window |
| Vertex AI Search RAG | Large document collections | Scalable, managed | Setup overhead, retrieval accuracy |
| Custom RAG (pgvector) | Full control needed | Customizable, portable | You manage infrastructure |

> **For Azure users:** Vertex AI Search is analogous to Azure AI Search with
> semantic ranking. The RAG pattern is identical — ingest, chunk, embed, retrieve,
> augment, generate.

---

## 8. Practical Workflow: Prototype to Production

### Phase 1: Prototype in AI Studio (30 min)

```
1. Open AI Studio → Chat mode
2. Write system instructions:
   "You classify Java exceptions into categories:
   BUG, CONFIG, INFRA, SECURITY, PERFORMANCE.
   Return JSON: {category, confidence, explanation, suggestedAction}"

3. Test with examples:
   - "NullPointerException in UserService.findById" → BUG
   - "Connection refused: Redis port 6379" → INFRA
   - "SQL injection detected in query parameter" → SECURITY

4. Iterate on system prompt until accuracy is good
5. Build structured prompt with 5-10 few-shot examples
6. Verify with edge cases
```

### Phase 2: Export and Build Service (1 hour)

```
1. Click "Get Code" → cURL
2. Translate to Spring Boot:
   - Create Spring AI ChatClient
   - Set system instructions from application.yml
   - Add structured output with entity mapping
   - Add Resilience4j retry/rate limiter

3. Write tests:
   - Unit tests with mock ChatClient
   - Integration tests against real API (use free tier)

4. Configure:
   - Externalize system prompt to config
   - Set up profiles: dev (free tier API key), prod (Vertex AI)
```

### Phase 3: Deploy to Production (1 hour)

```
1. Switch from API key to Vertex AI:
   - Update application-prod.yml with project ID and location
   - Configure service account with aiplatform.user role
   - Use Workload Identity for GKE (no key files)

2. Add observability:
   - Log token usage for cost tracking
   - Add latency metrics (Micrometer)
   - Set up alerts for high error rates

3. Optional: Evaluate and tune
   - Collect production examples → evaluation dataset
   - Run AutoSxS to compare base vs improved prompts
   - Fine-tune if prompt engineering plateaus
```

### Phase 4: Monitor and Optimize (Ongoing)

```
1. Track costs per endpoint (input tokens, output tokens)
2. Monitor quality via user feedback or automated evals
3. Experiment with model routing:
   - Simple classification → Flash Lite (cheapest)
   - Complex analysis → Flash with thinking
   - Critical decisions → 2.5 Pro
4. Use context caching for repeated large-context queries
5. Batch non-urgent tasks for cheaper processing
```

---

## 9. Try This Exercises

### Exercise 1: AI Studio Structured Prompt (20 min)
1. Open AI Studio → Structured Prompt mode
2. Create a "Spring Boot Error Classifier" with 5 few-shot examples
3. Test with 10 new error messages
4. Export to cURL and note the request format

### Exercise 2: Set Up Vertex AI (20 min)
1. Enable the Vertex AI API in your GCP project
2. Create a service account with `aiplatform.user` role
3. Make an API call to Gemini 2.5 Flash via Vertex AI endpoint
4. Compare: same prompt via AI Studio (API key) vs Vertex AI (service account)

### Exercise 3: File Upload Analysis (15 min)
1. Concatenate one of your Spring Boot microservices into a single text file
2. Upload it to AI Studio
3. Ask Gemini to:
   - Generate a README for the project
   - Find potential security issues
   - Suggest performance improvements
4. Evaluate the quality of each response

### Exercise 4: Build a RAG Prototype (30 min)
1. Collect 5-10 internal documents (runbooks, ADRs, onboarding guides)
2. Upload them to a GCS bucket
3. Create a Vertex AI Search data store
4. Import the documents
5. Test natural language queries against your documents

### Exercise 5: Model Evaluation (20 min)
1. Create 20 test cases for a task (e.g., code review quality)
2. Run each test case through Gemini 2.5 Flash and 2.0 Flash
3. Use Gemini 2.5 Pro as the judge to compare outputs
4. Calculate win rates — is the more expensive model worth it for your use case?

---

## Key Takeaways

1. **AI Studio** is your prototyping workbench — always start here before writing code
2. **Vertex AI** adds enterprise features: IAM, VPC, SLA, tuning, evaluation, monitoring
3. **Model tuning** is a last resort — prompt engineering covers 90% of use cases
4. **AutoSxS evaluation** is invaluable for comparing models objectively
5. **Agent Builder** lets you create RAG agents without custom orchestration code
6. **Vertex AI Search** provides managed RAG — same concept as Azure AI Search
7. The prototype-to-production path is: AI Studio → Spring AI → Vertex AI → Monitor
8. **Cost optimization:** Route by task complexity — use the cheapest model that works

---

Next: [05 — Gemini for Cloud and DevOps](05-gemini-for-cloud-and-devops.md)
