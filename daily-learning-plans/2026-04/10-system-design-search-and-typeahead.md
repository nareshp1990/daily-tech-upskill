# Day 34 — System Design Case Study: Search Engine & Typeahead
**Date:** 2026-04-10
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**System Design Case Study: Search Engine & Typeahead — Inverted Indexes and Sub-100ms Suggestions**

Search is everywhere: e-commerce product search, GitHub code search, Slack message search. The two distinct problems are: (1) **full-text search** — return all documents relevant to a query, ranked by relevance, and (2) **typeahead/autocomplete** — as the user types "jav", suggest "java", "javascript", "javafx" in < 100ms. Both problems require data structures specifically designed for search — inverted indexes and tries.

---

## 2. Full-Text Search — The Inverted Index

### 2.1 How an Inverted Index Works

```
Documents:
  doc1: "Java Spring Boot tutorial"
  doc2: "Spring Cloud microservices"
  doc3: "Java microservices with Kafka"

Inverted Index (token → document list):
  java      → [doc1, doc3]
  spring    → [doc1, doc2]
  boot      → [doc1]
  tutorial  → [doc1]
  cloud     → [doc2]
  microservices → [doc2, doc3]
  kafka     → [doc3]

Query "java microservices":
  java: [doc1, doc3]
  microservices: [doc2, doc3]
  Intersection (AND): [doc3]  ← Result: doc3
  Union (OR): [doc1, doc2, doc3]
```

### 2.2 Text Processing Pipeline

```
Input: "Java Spring-Boot TUTORIAL"
          │
    ┌─────▼─────┐
    │ Tokenise   │  → ["Java", "Spring-Boot", "TUTORIAL"]
    └─────┬─────┘
          │
    ┌─────▼─────┐
    │ Lowercase  │  → ["java", "spring-boot", "tutorial"]
    └─────┬─────┘
          │
    ┌─────▼─────┐
    │ Normalize  │  → ["java", "spring", "boot", "tutorial"]
    │ (split on -)
    └─────┬─────┘
          │
    ┌─────▼─────┐
    │ Stop words │  → ["java", "spring", "boot", "tutorial"]
    │ filter     │  (remove: the, is, at, etc.)
    └─────┬─────┘
          │
    ┌─────▼─────┐
    │ Stemming / │  → ["java", "spring", "boot", "tutori"]
    │ Lemmatize  │  (tutorial → tutori base form)
    └─────┬─────┘
          ▼
    Index tokens
```

---

## 3. Elasticsearch Integration (Spring Boot)

```xml
<dependency>
  <groupId>org.springframework.data</groupId>
  <artifactId>spring-data-elasticsearch</artifactId>
</dependency>
```

```java
@Document(indexName = "products")
public class ProductDocument {
    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "english")
    private String name;

    @Field(type = FieldType.Text, analyzer = "english")
    private String description;

    @Field(type = FieldType.Keyword)   // Exact match (for filtering)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Integer)
    private Integer stockCount;
}
```

```java
@Repository
public interface ProductSearchRepository
    extends ElasticsearchRepository<ProductDocument, String> {
}

@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations esOps;

    public SearchHits<ProductDocument> search(String query, String category,
                                               Double minPrice, Double maxPrice,
                                               int page, int size) {
        Query searchQuery = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    // Full-text search on name and description
                    .must(m -> m.multiMatch(mm -> mm
                        .query(query)
                        .fields("name^2", "description")   // name has 2× boost
                        .type(TextQueryType.BestFields)
                        .fuzziness("AUTO")   // Handle typos
                    ))
                    // Filters (exact, don't affect relevance score)
                    .filter(f -> f.term(t -> t.field("category").value(category)))
                    .filter(f -> f.range(r -> r.field("price").gte(JsonData.of(minPrice)).lte(JsonData.of(maxPrice))))
                ))
            .withPageable(PageRequest.of(page, size))
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(new HighlightField("description"))), null))
            .build();

        return esOps.search(searchQuery, ProductDocument.class);
    }
}
```

### 3.1 Data Sync: MySQL → Elasticsearch

```java
// Event-driven sync via Kafka
@KafkaListener(topics = "product.updated")
public void onProductUpdated(ProductUpdatedEvent event) {
    Product product = productRepository.findById(event.getProductId()).orElseThrow();
    ProductDocument doc = mapper.toDocument(product);
    productSearchRepository.save(doc);
}

// Or use Debezium (CDC) for automatic MySQL → Kafka sync
// Debezium captures MySQL binlog events and publishes to Kafka
// → Zero application code for sync!
```

---

## 4. Typeahead / Autocomplete

### 4.1 Trie Data Structure

```
Trie for ["java", "javascript", "javafx", "spring", "springboot"]:

root
├── j
│   └── a
│       └── v
│           ├── a* (end of "java", count: 50000)
│           │   └── s
│           │       └── c
│           │           └── r
│           │               └── i
│           │                   └── p
│           │                       └── t* (end of "javascript", count: 120000)
│           └── a
│               └── f
│                   └── x* (end of "javafx", count: 5000)
└── s
    └── p
        └── r
            └── i
                └── n
                    └── g* (end of "spring", count: 80000)
                        └── b
                            └── o
                                └── o
                                    └── t* (end of "springboot", count: 30000)

Query "jav" → suggestions: [javascript(120K), java(50K), javafx(5K)]
               (sorted by search frequency, top 5 returned)
```

### 4.2 Redis-Based Typeahead (Production)

A Trie is memory-intensive. For large vocabularies, use Redis sorted sets:

```java
@Service
@RequiredArgsConstructor
public class TypeaheadService {

    private final StringRedisTemplate redis;

    // Index a search term with its frequency score
    public void indexTerm(String term, double score) {
        // Add all prefixes: "java" → index "j", "ja", "jav", "java"
        for (int i = 1; i <= term.length(); i++) {
            String prefix = term.substring(0, i);
            redis.opsForZSet().add("typeahead:" + prefix, term, score);
        }
    }

    // Get top 5 suggestions for prefix
    public List<String> getSuggestions(String prefix, int limit) {
        // Returns terms with highest score (search frequency)
        Set<String> results = redis.opsForZSet()
            .reverseRange("typeahead:" + prefix.toLowerCase(), 0, limit - 1);
        return new ArrayList<>(results);
    }

    // Update frequency when a search is executed
    public void recordSearch(String term) {
        // Increment score for all prefixes of this term
        for (int i = 1; i <= term.length(); i++) {
            String prefix = term.substring(0, i);
            redis.opsForZSet().incrementScore("typeahead:" + prefix, term, 1.0);
        }
        indexTerm(term, 0);   // Ensure term is indexed even if new
    }
}
```

### 4.3 Typeahead Architecture

```
User types "jav"
      │
      ▼ (debounced, fire 300ms after last keystroke)
GET /v1/typeahead?q=jav
      │
      ▼
API Gateway → Typeahead Service
                    │
                    ▼
              Redis ZSet "typeahead:jav"
              → ["javascript", "java", "javafx"] in < 5ms

Response: 200 OK with suggestions (< 100ms total)
```

---

## 5. Relevance Ranking — TF-IDF and BM25

```
TF-IDF (Term Frequency × Inverse Document Frequency):
  TF  = how often term appears in this document
  IDF = log(total docs / docs containing term)
      = rare terms score higher than common terms

  Score("java", doc1) = TF("java", doc1) × IDF("java")
  "java" is common → low IDF
  "JVM bytecode verifier" → rare → high IDF → more specific

BM25 (Elasticsearch default since ES5):
  Improves TF-IDF by:
  - Diminishing returns: 10 occurrences vs 1 = not 10× better
  - Field length normalisation: match in 5-word title > match in 500-word description
```

---

## 6. Key Takeaways

1. **Inverted indexes power full-text search** — never use `LIKE '%query%'` at scale; use Elasticsearch.
2. **Text processing pipeline is critical** — lowercase, stemming, and stop word removal determine match quality.
3. **Tries vs Redis sorted sets for typeahead** — tries for in-memory, Redis ZSets for distributed production.
4. **Separate search index from source of truth** — Elasticsearch is a read-optimised projection; MySQL is the system of record.
5. **Fuzzy matching handles typos** — `fuzziness: "AUTO"` in Elasticsearch matches "jvaa" → "java".

---

## 7. Resources

- **[Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)** — Free online book.
- **[Spring Data Elasticsearch](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/)** — Spring integration reference.
- **[Debezium CDC](https://debezium.io/)** — MySQL → Kafka for search index sync.

---

*Day 34. Search is the window through which users see your data — make it fast, forgiving, and relevant.*
